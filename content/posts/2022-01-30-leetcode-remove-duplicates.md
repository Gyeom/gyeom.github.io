---
title: "LeetCode - Remove Duplicates from Sorted Array"
date: 2022-01-30
draft: true
tags: ["알고리즘", "Java", "배열", "투포인터", "LeetCode"]
categories: ["문제풀이"]
summary: "정렬된 배열에서 중복 제거. in-place로 배열 수정 필수, k 이후는 검사 안 함."
---

## 문제

[Remove Duplicates from Sorted Array - LeetCode #26](https://leetcode.com/problems/remove-duplicates-from-sorted-array/)

정렬된 배열에서 **중복을 제거**하고 유니크한 원소 개수 `k`를 반환한다.
**in-place**로 배열을 수정해야 한다.

## 풀이

```java
public int removeDuplicates(int[] nums) {
    if (nums.length == 0) return 0;

    int k = 1;  // 첫 번째 원소는 항상 유니크
    for (int i = 1; i < nums.length; i++) {
        if (nums[i] != nums[i - 1]) {  // 이전과 다르면
            nums[k] = nums[i];          // 유니크 위치에 저장
            k++;
        }
    }
    return k;
}
// 시간: O(n), 공간: O(1)
```

## 동작 예시

```
nums = [0, 0, 1, 1, 2]

i=1: 0 == 0 → 스킵
i=2: 1 != 0 → nums[1]=1, k=2
i=3: 1 == 1 → 스킵
i=4: 2 != 1 → nums[2]=2, k=3

결과: nums = [0, 1, 2, 1, 2], k = 3
              -------
              여기만 검사됨
```

## 핵심: k 이후는 검사 안 함

문제 조건:
> "It does not matter what you leave beyond the first k elements."

`nums[k]` 이후가 쓰레기 값이어도 **정답**이다. LeetCode는 `nums[0..k-1]`만 검사한다.

---

## 흔한 실수: 개수만 세고 배열 수정 안 함

```java
// ❌ 개수만 반환하면 틀림
int[] count = new int[201];
int i = 0;
for (int num : nums) {
    count[num + 100]++;
    if (count[num + 100] == 1) i++;
}
return i;  // nums 배열은 그대로!
```

문제는 **in-place 수정**을 요구한다. 개수만 맞아도 배열이 수정되지 않으면 오답.
