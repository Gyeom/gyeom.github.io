---
title: "LeetCode - Valid Palindrome"
date: 2022-01-26
draft: true
tags: ["알고리즘", "Java", "문자열", "투포인터", "LeetCode"]
categories: ["문제풀이"]
summary: "문자열이 팰린드롬인지 확인. 전처리 후 뒤집기 비교가 가장 간단."
---

## 문제

[Valid Palindrome - LeetCode #125](https://leetcode.com/problems/valid-palindrome/)

알파벳과 숫자만 고려해서 앞뒤로 읽어도 같은지 확인한다.

## 풀이 1: 전처리 + 뒤집기 (간단)

```java
public boolean isPalindrome(String s) {
    String cleaned = s.replaceAll("[^a-zA-Z0-9]", "").toLowerCase();
    return cleaned.equals(new StringBuilder(cleaned).reverse().toString());
}
// 시간: O(n), 공간: O(n)
```

## 풀이 2: 투 포인터 (공간 최적화)

```java
public boolean isPalindrome(String s) {
    int left = 0, right = s.length() - 1;
    while (left < right) {
        while (left < right && !Character.isLetterOrDigit(s.charAt(left))) left++;
        while (left < right && !Character.isLetterOrDigit(s.charAt(right))) right--;
        if (Character.toLowerCase(s.charAt(left)) != Character.toLowerCase(s.charAt(right))) {
            return false;
        }
        left++;
        right--;
    }
    return true;
}
// 시간: O(n), 공간: O(1)
```

## 비교

| 풀이 | 시간 | 공간 | 언제? |
|------|------|------|-------|
| 전처리 + reverse | O(n) | O(n) | 빠르게 풀 때 |
| 투 포인터 | O(n) | O(1) | 공간 최적화 요구 시 |

## 핵심

면접에서는 **전처리 + reverse로 먼저 풀고**, "공간 O(1)로 할 수 있나요?" 질문에 투 포인터로 개선.
