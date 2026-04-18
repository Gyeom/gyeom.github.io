---
title: "LeetCode - Contains Duplicate"
date: 2022-01-21
draft: true
tags: ["알고리즘", "Java", "HashSet", "LeetCode"]
categories: ["문제풀이"]
summary: "배열에 중복이 있는지 확인. HashSet의 add() 반환값 활용."
---

## 문제

[Contains Duplicate - LeetCode #217](https://leetcode.com/problems/contains-duplicate/)

배열에 중복된 값이 있으면 `true`, 없으면 `false` 반환.

## 풀이

```java
public boolean containsDuplicate(int[] nums) {
    Set<Integer> numSet = new HashSet<>();
    for (int num : nums) {
        if (!numSet.add(num)) {
            return true;  // add()가 false면 이미 존재
        }
    }
    return false;
}
// 시간: O(n), 공간: O(n)
```

## 핵심

`Set.add()`는 이미 존재하면 `false`를 반환한다. 이걸 활용하면 `contains()` 호출 없이 한 줄로 체크 가능.

```java
// 이렇게 안 해도 됨
if (set.contains(num)) return true;
set.add(num);

// 한 줄로 가능
if (!set.add(num)) return true;
```
