---
title: "LeetCode & HackerRank - FizzBuzz"
date: 2022-01-29
draft: true
tags: ["알고리즘", "Java", "조건문", "LeetCode", "HackerRank"]
categories: ["문제풀이"]
summary: "3의 배수는 Fizz, 5의 배수는 Buzz, 둘 다면 FizzBuzz. 조건 순서가 핵심."
---

## 문제 비교

| 문제 | 플랫폼 | 반환 | 특징 |
|------|--------|------|------|
| Fizz Buzz | LeetCode #412 | `List<String>` | n을 입력받음 |
| FizzBuzz | HackerRank | stdout 출력 | 1~100 고정, Code Golf |

---

## LeetCode - Fizz Buzz

[Fizz Buzz - LeetCode #412](https://leetcode.com/problems/fizz-buzz/)

```java
public List<String> fizzBuzz(int n) {
    List<String> result = new ArrayList<>();
    for (int i = 1; i <= n; i++) {
        if (i % 15 == 0) result.add("FizzBuzz");
        else if (i % 3 == 0) result.add("Fizz");
        else if (i % 5 == 0) result.add("Buzz");
        else result.add(String.valueOf(i));
    }
    return result;
}
// 시간: O(n), 공간: O(n)
```

---

## HackerRank - FizzBuzz

[FizzBuzz - HackerRank](https://www.hackerrank.com/challenges/fizzbuzz/problem)

**주의**: 입력 없음! 1~100 고정 출력. (Code Golf 챌린지)

```java
public class Solution {
    public static void main(String[] args) {
        for (int i = 1; i <= 100; i++) {
            if (i % 15 == 0) System.out.println("FizzBuzz");
            else if (i % 3 == 0) System.out.println("Fizz");
            else if (i % 5 == 0) System.out.println("Buzz");
            else System.out.println(i);
        }
    }
}
// 시간: O(1) - 항상 100번 반복
```

---

## 핵심 포인트

**조건 순서가 중요하다!**

```java
// ❌ 잘못된 순서 - 15에서 "Fizz"만 출력됨
if (i % 3 == 0) ...
else if (i % 5 == 0) ...
else if (i % 15 == 0) ...  // 여기 도달 못함

// ✅ 올바른 순서 - 15 먼저 체크
if (i % 15 == 0) ...  // 또는 (i % 3 == 0 && i % 5 == 0)
else if (i % 3 == 0) ...
else if (i % 5 == 0) ...
```

| 조건 | 출력 |
|------|------|
| 3과 5 둘 다 나눠짐 | FizzBuzz |
| 3만 나눠짐 | Fizz |
| 5만 나눠짐 | Buzz |
| 둘 다 안 나눠짐 | 숫자 |
