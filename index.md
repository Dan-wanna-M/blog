---
layout: default
---

# My Blog Posts

{% for post in site.posts %}
  {% if post.completed %}
  * [{{ post.title }}]({{ post.url | prepend: site.baseurl | prepend: site.url }})
  {% endif %}
{% endfor %}
