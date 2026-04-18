---
title: "LeetCode - Two Sum"
date: 2022-01-20
draft: true
tags: ["알고리즘", "Java", "HashMap", "LeetCode", "HackerRank"]
categories: ["문제풀이"]
summary: "두 수의 합이 target이 되는 인덱스 찾기. HashMap으로 O(n) 최적화."
---

## 문제

[Two Sum - LeetCode #1](https://leetcode.com/problems/two-sum/)
[Ice Cream Parlor - HackerRank](https://www.hackerrank.com/challenges/icecream-parlor/problem)

배열에서 두 수의 합이 `target`이 되는 **인덱스 쌍**을 찾는다.

## 풀이 1: Brute Force

```java
public static List<Integer> icecreamParlor(int m, List<Integer> arr) {
    List<Integer> answers = new ArrayList<>();
    for (int i = 0; i < arr.size(); i++) {
        for (int j = i + 1; j < arr.size(); j++) {
            if (m == arr.get(i) + arr.get(j)) {
                answers.add(i + 1);  // 1-indexed
                answers.add(j + 1);
            }
        }
    }
    return answers;
}
// 시간: O(n²), 공간: O(1)
```

## 풀이 2: HashMap 최적화

```java
public static List<Integer> icecreamParlor(int m, List<Integer> arr) {
    Map<Integer, Integer> numToIndex = new HashMap<>();
    for (int i = 0; i < arr.size(); i++) {
        int complement = m - arr.get(i);
        if (numToIndex.containsKey(complement)) {
            return Arrays.asList(numToIndex.get(complement) + 1, i + 1);
        }
        numToIndex.put(arr.get(i), i);
    }
    return Arrays.asList();
}
// 시간: O(n), 공간: O(n)
```

## 비교

| 풀이 | 시간 | 공간 | 특징 |
|------|------|------|------|
| Brute Force | O(n²) | O(1) | 직관적, 구현 쉬움 |
| HashMap | O(n) | O(n) | 최적화, 면접 권장 |

## 핵심

`target - nums[i]`가 이미 Map에 있는지 확인한다. 한 번의 순회로 해결.
