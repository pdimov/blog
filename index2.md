---
layout: page
title: Index
---

{% for post in posts %}
{{ post.date | date_to_string }} [{{ post.title }}]({{ post.url | absolute_url }})
{% endfor %}
