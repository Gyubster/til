---
layout: default
title: "Career"
---

# Career

커리어 & 면접 준비

{% assign posts = site.posts | where_exp: "post", "post.categories contains 'career'" %}
{% for post in posts %}
- [{{ post.title }}]({{ post.url | relative_url }}) — {{ post.date | date: "%Y-%m-%d" }}
{% endfor %}

{% if posts.size == 0 %}
아직 글이 없습니다.
{% endif %}
