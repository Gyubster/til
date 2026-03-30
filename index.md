---
layout: home
title: "심규보 TIL"
---

4년차 백엔드 엔지니어의 일일 학습 기록입니다.

매일 학습한 내용을 카테고리별로 정리하고, 단순 개념 정리가 아닌 실제로 적용하고 느낀 점을 중심으로 작성합니다.

## 카테고리

{% assign categories = "algorithm,system-design,api-design,optimization,debugging,cs-fundamentals,engineering-notes,career,fitness" | split: "," %}

{% for cat in categories %}
### [{{ cat | replace: "-", " " | capitalize }}](/til/categories/{{ cat }}/)
{% assign cat_posts = site.posts | where_exp: "post", "post.categories contains cat" %}
{{ cat_posts.size }}개의 글
{% endfor %}
