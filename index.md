---
layout: page
title: Welcome
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
- GitHub: [My Github](https://github.com/liuzhenkun)
- Email : _zhenkum.liu@gmail.com_





