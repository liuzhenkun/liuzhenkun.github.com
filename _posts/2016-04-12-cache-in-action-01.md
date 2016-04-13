---
layout: post
title: "Cache in Action"
description: "技术博客专注与技术分享,包括程序设计,设计模式,缓存策略,技术方案,架构设计,高并发,高可用,高扩展等技术点"
keywords: "缓存,redis,memcache,java,javaee,j2ee,读多写少,mapdb,架构,rpc,分布式,集群"
category: cache
tags: [cache, redis, memcache]
---

缓存应用解析.

# 应用缓存（不断完善中）

## 目录
- 为什么使用缓存？
- 缓存技术选型
- 缓存与业务逻辑的时序问题
- 缓存与系统设计
- 缓存的注意事项

## 为什么使用缓存？
-  以读为主且读较写占绝对优势的场景
     - 在 读：写 = 9：1 的场景下,读需求占到了90%以上的比例,如若此时将90%的查询指向DB,则会对DB造成很大压力,而且大部分业务数据在一段时间内是固定不变的,可以考虑将不同业务第一次的查询结果缓存起来。这既减轻了DB压力,也提高了查询效率(在缓存使用得当的情况下,查询缓存的效率至少是查询DB的百倍以上)；

- 并发场景
   - 常见的缓存技术具存活时间TTL的设置,即超时后缓存内容会被清除；
   - 利用TTL,可以用来做抵挡并发获取资源的操作；
   - 比如：
       - 做一个抽奖活动,可以做这样的限制：
       - 同一个用户100ms内只能抽一次奖(以确保来抽奖的是人而不是机器,当然防刷还需要其他的限制才能保证)

- 灵活使用Redis支持的数据结构
    - redis中有很多复杂度o(1)操作的方法,操作简单,性能好,可以用来辅助实现很多业务场景,本文重点介绍缓存技术,以后会开文详细介绍redis数据结构及其应用；

## 缓存技术选型
- 内存缓存
   - 如JAVA内置对象,或者第三方内存缓存mapDB等
   - 优点： 维护简单,效率很高
   - 缺点： 不支持cluster,非分布式缓存,宕机后缓存数据丢失
- redis/memcache
   - 优点：操作简便,效率高,可作为分布式缓存,支持缓存数据持久化（仅redis支持）,支持集群
   - 缺点：需要单独运维一套缓存服务

## 缓存与业务逻辑的时序问题
- 查询数据的时序
- ![redis-select]({{ site.url }}/files/images/2016-04-12-cache-in-action-01/redis-select.png "hello")
- 如上图所示,其实已经非常清楚了。需要提到的一个指标是<font color="red">【缓存命中率】： hit/(hit+miss)</font>
    - hit: 查询数据时,缓存命中的次数(缓存中有结果)；
    - miss: 查询缓存时,缓存未命中的次数(缓存中无结果)；
    - 缓存命中率低于95%,表示可能存在缓存设置不当等问题；

- 插入数据的时序
- ![redis-insert]({{ site.url }}/files/images/2016-04-12-cache-in-action-01/redis-insert.png)

- 缓存的set/delete操作
   - set : 正如软件设计中的所有set逻辑一样,set(key, value)表示如果key存在,则更新为新value;
           如果key不存在,则新增一对,即add(key, value);
   - delete : delete(key) 直接清除key对应的缓存；
   - 其实操作都非常简单,  现在对使用set(key, value)、delete(key)来进行简要的分析：
   - 若场景逻辑较为简单,则建议使用set(key, value)
       例：用户在界面上修改了个人信息之后,点击更新按钮,在获取到更新的用户信息并更新到DB后可以set到缓存中 set(userKey, userInfo)
   - 若场景逻辑较为复杂,则建议使用delete(key)
      - 例： 用户在理财平台购买了一个理财计划,系统会根据规则更新用户的用户账户余额
      - step1 获取用户账户信息
      - step2 获取理财计划余额以及购买条件
      - step3 用户是否具备购买条件
      - step4 满足条件进入购买流程...购买完成后更新用户余额信息至DB
      - step5 set(userKey, userAccount)
      - 第五步,为了获取用户更新后的账户信息要进行一系列操作,代价较大,所以建议直接delete(key), 造成的影响仅仅是一次cache miss.
   - <font color="red">结论：使用delete(key), 虽然会造成一次cache miss</font>

- 更新DB与操作缓存的时序
   - 更新DB和delete(key)孰先孰后呢？
   - 原则： 对业务的影响控制在最小。
   - 先来看一看所有的场景：
   - 场景一
     - ![state1]({{ site.url }}/files/images/2016-04-12-cache-in-action-01/state1.png)
     - 先更新DB失败,后del缓存成功
     - 用户再进行查询的时候仅有一次cache miss, 实际不会发生这样的场景。如果更新数据库失败,则不能操作缓存,redis中还会持有原来的旧数据;   <font color="red">FAILED</font>

   - 场景二
     - ![state2]({{ site.url }}/files/images/2016-04-12-cache-in-action-01/state2.png)
     - 先更新DB成功,后del缓存失败
     - 使得DB与缓存中的数据不一致,再次访问缓存时获取到的是旧数据(DB中的数据较新)  <font color="red">FAILED</font>

   - 场景三
     - ![state3]({{ site.url }}/files/images/2016-04-12-cache-in-action-01/state3.png)
     - 先del缓存失败,后更新DB成功, 同场景二  <font color="red">FAILED</font>

   - 场景四
     - ![state4]({{ site.url }}/files/images/2016-04-12-cache-in-action-01/state4.png)
     - 先del缓存成功,后更新DB失败, 用户再进行查询的时候仅有一次cache miss <font color="purple">PASSED</font>

   - 综上所述
   - 正确的时序： 首先delete(key)缓存,其次更新DB,缺点是此过程完成后的第一次查询会造成cache miss。
   - 此处有坑：delete(key)与update两个操作要连续(两个操作之间不要有其他业务逻辑代码step2/3),否则可能出现如下DB与缓存数据不一致的场景：
     - step1: delete(key)
     - step2: execute other logic
     - step3: 用户查询信息,发现key已被擦除,故从DB中拉去了数据并存入了缓存中(旧数据)
     - step4: update db(新数据)
   - ### <font color="red">正确的时序： 首先delete(key)缓存,其次更新DB,再delete(key)缓存</font> ###

## 缓存与系统设计

### 缓存与standalone系统
 - 业务初阶,无须过于复杂的设计,能够支撑未来一段时间业务的系统就是好系统,总之不要over design;
 属于internet baby born阶段的技术团队,为了balance技术追求和上线时间,往往设计的系统就是一个war,包含mvc, service, dao,以及缓存技术。
- 此阶段可以选择内存缓存,mapdb等比较standalone的缓存方法。缺点上文已经叙述过了,此处不再重复。
- 缓存的操作与业务逻辑耦合在系统中。
- ![cache-architect-01]({{ site.url }}/files/images/2016-04-12-cache-in-action-01/cache-architect-01.png)

### 缓存与拆分后的系统
  - 无论是水平拆分,还是垂直拆分,我们根据业务将系统拆成了若干个。如果系统集群化,这时候standalone的缓存方式根本不能支撑业务正常运行,举个栗子：
    原来的某个系统使用内存mapdb计数,如果这个系统只部署一个节点,那么它记录的是准确的；如果系统被部署了多套,每次将请求负载均衡的打在多个node上,这个时候自有jvm实例维护的内存对象无法共享,无法获得正确的结果。(不要用J2EE早期的HttpSession Copy来反驳我哦,这不是一回事)。
  - 更好的做法是采用集中式缓存(集群)
  - ![cache-architect-02]({{ site.url }}/files/images/2016-04-12-cache-in-action-01/cache-architect-02.png)

### 缓存与拆分后的系统SOA
- 系统服务化治理,拆分rpc-client, rpc-server,注册中心register(具体服务化系统设计不是本文重点,只是粗略一撇,之后开文详述)
![cache-architect-03]({{ site.url }}/files/images/2016-04-12-cache-in-action-01/cache-architect-03.png)
- rpc-server作为距离DB最近的地方,具备所有操作缓存的权限(CRUD)
- rpc-client的职责是校验参数、统一返回结果、统一异常处理等,这里它仅仅具备缓存的读权限
- IF NOT？世界大乱

# <font color="red">结论</font>：

 - <font color="blue">缓存的操作与更新DB之间的顺序问题
     先delete(key), 再更新DB,且两个操作要保持连续；最后再delete(key)</font>
 - <font color="blue">尽量使用集中式缓存,用面向集群的思路去写代码更易于扩展.</font>













