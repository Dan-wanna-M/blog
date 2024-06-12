---
layout: default
---

# Huanghe's blogs

{% for post in site.posts %}
  {% if post.completed %}
  * [{{ post.title }}]({{ post.url | prepend: site.baseurl | prepend: site.url }})
  {% endif %}
{% endfor %}
