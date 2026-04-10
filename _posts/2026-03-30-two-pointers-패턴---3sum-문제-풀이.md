---
title: "Two Pointers 패턴 - 3Sum 문제 풀이"
date: 2026-03-30
categories: [Algorithm]
tags: [leetcode, two-pointers, medium]
---

## 오늘 배운 것

### Two Pointers 패턴

LeetCode #15 3Sum 문제를 풀면서 Two Pointers 패턴을 학습했다.

### 핵심 아이디어

- 배열을 정렬한 뒤, 하나의 원소를 고정하고 나머지 두 포인터를 양 끝에서 좁혀나간다
- 시간복잡도: O(n2) - 브루트포스 O(n3) 대비 개선
- 중복 제거가 핵심 포인트

### 느낀 점

정렬 후 포인터를 활용하는 패턴은 다양한 문제에 적용할 수 있다.