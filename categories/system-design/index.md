---
layout: default
title: "System Design"
---

# System Design

시스템 설계 & 아키텍처 (스탯: SYS)

{% assign posts = site.posts | where_exp: "post", "post.categories contains 'system-design'" %}
{% for post in posts %}
- [{{ post.title }}]({{ post.url | relative_url }}) — {{ post.date | date: "%Y-%m-%d" }}
{% endfor %}

{% if posts.size == 0 %}
아직 글이 없습니다.
{% endif %}
