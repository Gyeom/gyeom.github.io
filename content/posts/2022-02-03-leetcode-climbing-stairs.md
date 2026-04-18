---
title: "LeetCode - Climbing Stairs"
date: 2022-02-03
draft: true
tags: ["알고리즘", "Java", "DP", "피보나치", "LeetCode"]
categories: ["문제풀이"]
summary: "n개의 계단을 1칸 또는 2칸씩 오르는 방법 수. 피보나치 수열."
---

## 문제

[Climbing Stairs - LeetCode #70](https://leetcode.com/problems/climbing-stairs/)

계단을 **1칸 또는 2칸**씩 오를 수 있을 때, n개 계단을 오르는 방법의 수.

## 풀이 1: DP 배열

```java
public int climbStairs(int n) {
    if (n <= 2) return n;

    int[] dp = new int[n + 1];
    dp[1] = 1;  // 1칸: 1가지
    dp[2] = 2;  // 2칸: 2가지 (1+1, 2)

    for (int i = 3; i <= n; i++) {
        dp[i] = dp[i - 1] + dp[i - 2];
    }
    return dp[n];
}
// 시간: O(n), 공간: O(n)
```

## 풀이 2: 공간 최적화

```java
public int climbStairs(int n) {
    if (n <= 2) return n;

    int prev2 = 1, prev1 = 2;  // dp[1], dp[2]
    for (int i = 3; i <= n; i++) {
        int curr = prev1 + prev2;
        prev2 = prev1;
        prev1 = curr;
    }
    return prev1;
}
// 시간: O(n), 공간: O(1)
```

## 핵심 아이디어

**점화식**: n번째 계단에 도달하려면?
- (n-1)에서 1칸 올라오기
- (n-2)에서 2칸 올라오기

```
dp[n] = dp[n-1] + dp[n-2]
```

이건 **피보나치 수열**이다.

## 동작 예시

```
n = 5

dp[1] = 1
dp[2] = 2
dp[3] = dp[2] + dp[1] = 3
dp[4] = dp[3] + dp[2] = 5
dp[5] = dp[4] + dp[3] = 8

결과: 8가지 방법
```

| n | 방법 수 | 경우의 수 |
|---|--------|----------|
| 1 | 1 | (1) |
| 2 | 2 | (1,1), (2) |
| 3 | 3 | (1,1,1), (1,2), (2,1) |
| 4 | 5 | ... |
| 5 | 8 | ... |

## DP 문제 인식

- "몇 가지 방법?" → DP
- "이전 상태에서 현재 상태 계산" → 점화식
- 피보나치 변형 문제가 많다
