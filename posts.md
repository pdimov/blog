---
layout: page
title: Posts
---

{% for post in site.posts %}
{{ post.date | date_to_string }}: [{{ post.title }}]({{ post.url | absolute_url }})
{% endfor %}
