---
title: "LeetCode - Maximum Subarray"
date: 2022-02-01
draft: true
tags: ["알고리즘", "Java", "배열", "DP", "Kadane", "LeetCode"]
categories: ["문제풀이"]
summary: "연속 부분 배열의 최대 합. Kadane 알고리즘으로 O(n) 풀이."
---

## 문제

[Maximum Subarray - LeetCode #53](https://leetcode.com/problems/maximum-subarray/)

정수 배열에서 **연속된 부분 배열**의 최대 합을 구한다.

## 풀이: Kadane's Algorithm

```java
public int maxSubArray(int[] nums) {
    int curr = nums[0];  // 현재 위치에서 끝나는 최대 합
    int max = nums[0];   // 전체 최대 합

    for (int i = 1; i < nums.length; i++) {
        curr = Math.max(nums[i], curr + nums[i]);  // 새로 시작 vs 이어가기
        max = Math.max(max, curr);
    }
    return max;
}
// 시간: O(n), 공간: O(1)
```

## 동작 예시

```
nums = [-2, 1, -3, 4, -1, 2, 1, -5, 4]

i=0: curr=-2, max=-2
i=1: curr=1  (-2+1=-1 < 1), max=1
i=2: curr=-2 (1-3=-2), max=1
i=3: curr=4  (-2+4=2 < 4), max=4     ← 새로 시작
i=4: curr=3  (4-1=3), max=4
i=5: curr=5  (3+2=5), max=5
i=6: curr=6  (5+1=6), max=6          ← 최대
i=7: curr=1  (6-5=1), max=6
i=8: curr=5  (1+4=5), max=6

결과: 6 (부분 배열: [4, -1, 2, 1])
```

## 핵심 아이디어

**Kadane's Algorithm**: 각 위치에서 "새로 시작" vs "이전 결과에 이어가기" 중 큰 값 선택.

```
curr = max(nums[i], curr + nums[i])
```

| 선택 | 의미 |
|------|------|
| `nums[i]` | 이전 합이 음수라 새로 시작하는 게 유리 |
| `curr + nums[i]` | 이전 합이 양수라 이어가는 게 유리 |

## DP 관점

```
dp[i] = i에서 끝나는 최대 부분합
dp[i] = max(nums[i], dp[i-1] + nums[i])
```

Kadane은 공간을 O(1)로 최적화한 DP다.
