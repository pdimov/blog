---
layout: page
title: Posts
---

{% assign posts_by_month = site.posts | group_by_exp: "post", "post.date | date: '%Y %b'" %}
{% for month in posts_by_month %}
## {{ month.name }}
{% for post in month.items %}
* [{{ post.title }}]({{ post.url | absolute_url }}) ({{ post.date | date_to_string }})
{% endfor %}
{% endfor %}
