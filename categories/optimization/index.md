---
layout: default
title: "Optimization"
---

# Optimization

성능 최적화 & 튜닝

{% assign posts = site.posts | where_exp: "post", "post.categories contains 'optimization'" %}
{% for post in posts %}
- [{{ post.title }}]({{ post.url | relative_url }}) — {{ post.date | date: "%Y-%m-%d" }}
{% endfor %}

{% if posts.size == 0 %}
아직 글이 없습니다.
{% endif %}
