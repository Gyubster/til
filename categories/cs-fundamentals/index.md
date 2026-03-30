---
layout: default
title: "CS Fundamentals"
---

# CS Fundamentals

CS 기초 & 이론

{% assign posts = site.posts | where_exp: "post", "post.categories contains 'cs-fundamentals'" %}
{% for post in posts %}
- [{{ post.title }}]({{ post.url | relative_url }}) — {{ post.date | date: "%Y-%m-%d" }}
{% endfor %}

{% if posts.size == 0 %}
아직 글이 없습니다.
{% endif %}
