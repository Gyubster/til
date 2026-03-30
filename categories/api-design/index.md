---
layout: default
title: "API Design"
---

# API Design

API 설계 & 마이크로서비스

{% assign posts = site.posts | where_exp: "post", "post.categories contains 'api-design'" %}
{% for post in posts %}
- [{{ post.title }}]({{ post.url | relative_url }}) — {{ post.date | date: "%Y-%m-%d" }}
{% endfor %}

{% if posts.size == 0 %}
아직 글이 없습니다.
{% endif %}
