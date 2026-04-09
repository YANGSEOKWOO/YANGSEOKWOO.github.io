---
layout: post
title: "Python List Comprehension 정리"
description: "파이썬 리스트 컴프리헨션의 기본 문법과 활용 패턴을 정리했다."
date: 2026-04-09
category: Programming
tags: [Python, TIL]
---

## List Comprehension이란?

리스트를 간결하게 생성하는 파이썬 문법이다.

```python
# 기본 for문
squares = []
for x in range(10):
    squares.append(x ** 2)

# list comprehension
squares = [x ** 2 for x in range(10)]
```

## 조건 필터링

```python
# 짝수만 필터링
evens = [x for x in range(20) if x % 2 == 0]
```

## 중첩 반복

```python
# 2차원 리스트 평탄화
matrix = [[1, 2], [3, 4], [5, 6]]
flat = [x for row in matrix for x in row]
# [1, 2, 3, 4, 5, 6]
```

## 언제 쓰고 언제 안 쓸까?

- **쓸 때**: 단순 변환, 필터링
- **안 쓸 때**: 로직이 복잡해서 가독성이 떨어질 때 → 일반 for문이 낫다
