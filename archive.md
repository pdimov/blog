---
layout: page
title: Archive
---

{% assign posts_by_month = site.posts | group_by_exp: "post", "post.date | date: '%Y %b'" %}
{% for month in posts_by_month %}
## {{ month.name }}
---
{% for post in month.items %}
{% unless post.hidden %}
* [{{ post.title }}]({{ post.url | absolute_url }}) ({{ post.date | date_to_string }})
{% endunless %}
{% endfor %}
{% endfor %}
