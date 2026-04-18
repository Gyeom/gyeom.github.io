---
title: "LeetCode - Single Number (Lonely Integer)"
date: 2022-01-22
draft: true
tags: ["알고리즘", "Java", "XOR", "HashMap", "LeetCode", "HackerRank"]
categories: ["문제풀이"]
summary: "혼자만 있는 숫자 찾기. XOR로 O(1) 공간 최적화."
---

## 문제

[Single Number - LeetCode #136](https://leetcode.com/problems/single-number/)
[Lonely Integer - HackerRank](https://www.hackerrank.com/challenges/lonely-integer/problem)

모든 숫자가 두 번씩 나오는데 **하나만 한 번** 나온다. 그 숫자를 찾는다.

## 풀이 1: HashMap (직관적)

```java
public static int lonelyinteger(List<Integer> a) {
    Map<Integer, Integer> counter = new HashMap<>();
    for (int num : a) {
        counter.put(num, counter.getOrDefault(num, 0) + 1);
    }

    for (int num : a) {
        if (counter.get(num) == 1) {
            return num;
        }
    }
    return -1;
}
// 시간: O(n), 공간: O(n)
```

## 풀이 2: XOR (최적)

```java
public int singleNumber(int[] nums) {
    int result = 0;
    for (int num : nums) {
        result ^= num;
    }
    return result;
}
// 시간: O(n), 공간: O(1)
```

## XOR 동작 원리

배열 `[4, 1, 2, 1, 2]`

```
0 ^ 4 = 4     ← 4 등장
4 ^ 1 = 5     ← 1 등장
5 ^ 2 = 7     ← 2 등장
7 ^ 1 = 6     ← 1 상쇄 (1^1=0)
6 ^ 2 = 4     ← 2 상쇄 (2^2=0)
→ 결과: 4 (혼자인 숫자)
```

## 핵심

- `a ^ a = 0` (같은 수는 상쇄)
- `a ^ 0 = a` (0과 XOR하면 그대로)
- 짝은 다 사라지고 혼자만 남음

| 풀이 | 시간 | 공간 | 면접 팁 |
|------|------|------|---------|
| HashMap | O(n) | O(n) | 먼저 제시 (직관적) |
| XOR | O(n) | O(1) | "공간 최적화" 질문 시 |
