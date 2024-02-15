---
layout: home
title: "심규보 TIL"
---

백엔드 엔지니어 심규보의 일일 학습 기록입니다.

## 카테고리

{% assign categories = "algorithm,system-design,api-design,optimization,debugging,cs-fundamentals,career,fitness" | split: "," %}

{% for cat in categories %}
### [{{ cat | replace: "-", " " | capitalize }}](/til/categories/{{ cat }}/)
{% assign cat_posts = site.posts | where_exp: "post", "post.categories contains cat" %}
{{ cat_posts.size }}개의 글
{% endfor %}
