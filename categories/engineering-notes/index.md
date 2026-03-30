---
layout: default
title: "Engineering Notes"
---

# Engineering Notes

실무에서 겪은 문제 분석, 설계 고민, 고도화 방향 정리

{% assign posts = site.posts | where_exp: "post", "post.categories contains 'engineering-notes'" %}
{% for post in posts %}
- [{{ post.title }}]({{ post.url | relative_url }}) — {{ post.date | date: "%Y-%m-%d" }}
{% endfor %}

{% if posts.size == 0 %}
아직 글이 없습니다.
{% endif %}
