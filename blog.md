---
layout: page
title: 文章
permalink: /blog/
---

记录学习、构建和思考的过程。

{% for post in site.posts %}
- <time>{{ post.date | date: "%Y-%m-%d" }}</time> [{{ post.title }}]({{ post.url | prepend: site.baseurl }})
{% endfor %}
