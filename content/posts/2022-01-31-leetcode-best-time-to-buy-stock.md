---
title: "LeetCode - Best Time to Buy and Sell Stock"
date: 2022-01-31
draft: true
tags: ["알고리즘", "Java", "배열", "그리디", "LeetCode"]
categories: ["문제풀이"]
summary: "주식 한 번 사고팔아서 최대 이익. 최저점 추적하면서 이익 계산."
---

## 문제

[Best Time to Buy and Sell Stock - LeetCode #121](https://leetcode.com/problems/best-time-to-buy-and-sell-stock/)

주식을 **한 번만** 사고팔 수 있을 때 최대 이익을 구한다.

## 풀이

```java
public int maxProfit(int[] prices) {
    int min = prices[0];
    int maxProfit = 0;

    for (int price : prices) {
        min = Math.min(min, price);           // 지금까지 최저가
        maxProfit = Math.max(maxProfit, price - min);  // 최대 이익
    }
    return maxProfit;
}
// 시간: O(n), 공간: O(1)
```

## 동작 예시

```
prices = [7, 1, 5, 3, 6, 4]

i=0: min=7, profit=0
i=1: min=1, profit=0      ← 최저점 갱신
i=2: min=1, profit=4      ← 5-1=4
i=3: min=1, profit=4
i=4: min=1, profit=5      ← 6-1=5 (최대)
i=5: min=1, profit=5

결과: 5
```

## 핵심 아이디어

**그리디**: 과거 최저점에서 샀다고 가정하고, 현재 가격에서 팔 때 이익 계산.

| 변수 | 의미 |
|------|------|
| `min` | 지금까지 본 최저 가격 |
| `maxProfit` | 지금까지 계산한 최대 이익 |

## 흔한 실수

```java
// ❌ O(n²) - 모든 쌍 비교
for (int i = 0; i < n; i++) {
    for (int j = i + 1; j < n; j++) {
        profit = Math.max(profit, prices[j] - prices[i]);
    }
}

// ✅ O(n) - 최저점만 추적
```
