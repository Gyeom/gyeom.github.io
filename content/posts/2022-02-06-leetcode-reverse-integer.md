---
title: "LeetCode - Reverse Integer"
date: 2022-02-06
draft: true
tags: ["알고리즘", "Java", "수학", "오버플로우", "LeetCode"]
categories: ["문제풀이"]
summary: "정수 뒤집기. 오버플로우 발생 시 0 반환. % 10으로 추출, / 10으로 제거."
---

## 문제

[Reverse Integer - LeetCode #7](https://leetcode.com/problems/reverse-integer/)

32비트 정수를 뒤집는다. **오버플로우 발생 시 0 반환**.

## 풀이

```java
public int reverse(int x) {
    int result = 0;

    while (x != 0) {
        int digit = x % 10;  // 마지막 자리 추출
        x /= 10;             // 마지막 자리 제거

        // 오버플로우 체크 (곱하기 전에!)
        if (result > Integer.MAX_VALUE / 10 || result < Integer.MIN_VALUE / 10) {
            return 0;
        }
        result = result * 10 + digit;
    }
    return result;
}
// 시간: O(log n), 공간: O(1)
```

## 동작 예시

```
x = 123

반복 1: digit = 3, x = 12, result = 3
반복 2: digit = 2, x = 1,  result = 32
반복 3: digit = 1, x = 0,  result = 321

결과: 321
```

## 오버플로우 체크가 왜 필요한가?

```java
// Integer.MAX_VALUE = 2,147,483,647
// 뒤집으면 7,463,847,412 → 오버플로우!
```

**곱하기 전에** 체크해야 한다.

```java
// ❌ 곱하기 후 체크 - 이미 오버플로우 발생
result = result * 10 + digit;
if (result > Integer.MAX_VALUE) ...  // 늦음!

// ✅ 곱하기 전 체크
if (result > Integer.MAX_VALUE / 10) return 0;
result = result * 10 + digit;
```

## 왜 0을 반환하는가?

문제 조건:
> "If reversing x causes the value to go outside the signed 32-bit integer range, return 0."

오버플로우 처리 방법은 문제마다 다르다. 이 문제는 **0 반환**을 요구한다.

## 음수 처리

`% 10`은 음수에서도 동작한다.

```
x = -123
-123 % 10 = -3
-12 % 10 = -2
-1 % 10 = -1

result = -321 ✅
```

Java의 `%` 연산자는 피연산자의 부호를 유지하므로 별도 처리 불필요.
