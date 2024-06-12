---
title: Huanghe's blog
layout: default
---

## Blog

{% for post in site.posts %}
  {% if post.completed %}
  * [{{ post.title }}]({{ post.url | prepend: site.baseurl | prepend: site.url }})
  {% endif %}
{% endfor %}
