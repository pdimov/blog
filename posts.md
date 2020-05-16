---
layout: page
title: Posts
---

{% for month in site.posts | group_by_exp: "post", "post.date | date: '%Y %b'" %}
## {{ month.name }}
{% for post in month.items %}
[{{ post.title }}]({{ post.url | absolute_url }}) ({{ post.date | date_to_string }})
{% endfor %}
{% endfor %}
