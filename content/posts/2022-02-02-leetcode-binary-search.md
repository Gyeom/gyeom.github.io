---
title: "LeetCode - Binary Search"
date: 2022-02-02
draft: true
tags: ["알고리즘", "Java", "배열", "이진탐색", "LeetCode"]
categories: ["문제풀이"]
summary: "정렬된 배열에서 타겟 찾기. O(log n)의 기본 이진 탐색."
---

## 문제

[Binary Search - LeetCode #704](https://leetcode.com/problems/binary-search/)

**정렬된 배열**에서 타겟의 인덱스를 찾는다. 없으면 -1 반환.

## 풀이

```java
public int search(int[] nums, int target) {
    int left = 0, right = nums.length - 1;

    while (left <= right) {
        int mid = left + (right - left) / 2;  // 오버플로우 방지

        if (nums[mid] == target) {
            return mid;
        } else if (nums[mid] < target) {
            left = mid + 1;
        } else {
            right = mid - 1;
        }
    }
    return -1;
}
// 시간: O(log n), 공간: O(1)
```

## 동작 예시

```
nums = [-1, 0, 3, 5, 9, 12], target = 9

left=0, right=5
  mid=2, nums[2]=3 < 9 → left=3

left=3, right=5
  mid=4, nums[4]=9 == 9 → return 4
```

## 핵심 포인트

**1. mid 계산 방식**

```java
// ❌ 오버플로우 가능
int mid = (left + right) / 2;

// ✅ 안전
int mid = left + (right - left) / 2;
```

**2. 경계 조건**

| 조건 | 의미 |
|------|------|
| `left <= right` | 검사할 원소가 남아있음 |
| `left = mid + 1` | mid는 이미 확인, 오른쪽 탐색 |
| `right = mid - 1` | mid는 이미 확인, 왼쪽 탐색 |

## 시간 복잡도

매 반복마다 탐색 범위가 **절반**으로 줄어든다.

```
n → n/2 → n/4 → ... → 1
```

반복 횟수: log₂(n) → **O(log n)**

| n | 최대 비교 횟수 |
|---|---------------|
| 1,000 | 10 |
| 1,000,000 | 20 |
| 10억 | 30 |
