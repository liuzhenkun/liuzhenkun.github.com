---
layout: page
title: "Welcome"
description: "技术博客专注与技术分享,包括程序设计,设计模式,缓存策略,技术方案,架构设计,高并发,高可用,高扩展等技术点"
keywords: "缓存,redis,memcache,java,javaee,j2ee,读多写少,mapdb,架构,rpc,分布式,集群"
---
{% include JB/setup %}

## Introduction
- I am a software architect from [_BAIDU-百度_](http://www.baidu.com)(_the biggest Chinese search engine in the world_), China.
- The topics like: the usage and mechanism of famous open source software, designing HA distributed server
  side applications, the common used cache skills, tuning performance of jvm hotspot and other related technology stacks.

## Articles
<ul class="posts">
  {% for post in site.posts %}
    <li><span>{{ post.date | date_to_string }}</span> &raquo; <a href="{{ BASE_PATH }}{{ post.url }}">{{ post.title }}</a>
    {{ post.excerpt | remove: '<p>' | remove: '</p>' }}</li>
  {% endfor %}
</ul>

## Contact Me
- Email : _zhenkum.liu@gmail.com_





