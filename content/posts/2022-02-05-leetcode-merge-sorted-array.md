---
title: "LeetCode - Merge Sorted Array"
date: 2022-02-05
draft: true
tags: ["알고리즘", "Java", "배열", "투포인터", "LeetCode"]
categories: ["문제풀이"]
summary: "두 정렬된 배열 병합. 뒤에서부터 채우면 덮어쓰기 걱정 없음."
---

## 문제

[Merge Sorted Array - LeetCode #88](https://leetcode.com/problems/merge-sorted-array/)

`nums1`에 `nums2`를 **in-place**로 병합한다. `nums1`은 뒤에 0으로 채워진 공간이 있다.

## 풀이

```java
public void merge(int[] nums1, int m, int[] nums2, int n) {
    int i = m - 1;      // nums1의 마지막 원소
    int j = n - 1;      // nums2의 마지막 원소
    int k = m + n - 1;  // 병합 결과의 마지막 위치

    while (j >= 0) {
        if (i >= 0 && nums1[i] > nums2[j]) {
            nums1[k--] = nums1[i--];
        } else {
            nums1[k--] = nums2[j--];
        }
    }
}
// 시간: O(m + n), 공간: O(1)
```

## 동작 예시

```
nums1 = [1, 2, 3, 0, 0, 0], m = 3
nums2 = [2, 5, 6], n = 3

i=2, j=2, k=5: nums1[2]=3 < nums2[2]=6 → nums1[5]=6, j=1
i=2, j=1, k=4: nums1[2]=3 < nums2[1]=5 → nums1[4]=5, j=0
i=2, j=0, k=3: nums1[2]=3 > nums2[0]=2 → nums1[3]=3, i=1
i=1, j=0, k=2: nums1[1]=2 == nums2[0]=2 → nums1[2]=2, j=-1

결과: [1, 2, 2, 3, 5, 6]
```

## 왜 뒤에서부터 채우는가?

```
❌ 앞에서부터 채우면:
nums1 = [1, 2, 3, 0, 0, 0]
          ↑
          여기에 새 값 넣으면 기존 값 덮어씀!

✅ 뒤에서부터 채우면:
nums1 = [1, 2, 3, 0, 0, 0]
                        ↑
                        여기는 빈 공간!
```

뒤에서부터 채우면 아직 비교하지 않은 원소를 덮어쓸 걱정이 없다.

## for문 버전

```java
public void merge(int[] nums1, int m, int[] nums2, int n) {
    int i = m - 1, j = n - 1;

    for (int k = m + n - 1; k >= 0 && j >= 0; k--) {
        if (i >= 0 && nums1[i] > nums2[j]) {
            nums1[k] = nums1[i--];
        } else {
            nums1[k] = nums2[j--];
        }
    }
}
```

while문과 동일한 로직. 취향에 따라 선택.
