---
title: "LeetCode - Valid Parentheses (Balanced Brackets)"
date: 2022-01-23
draft: true
tags: ["알고리즘", "Java", "Stack", "LeetCode", "HackerRank"]
categories: ["문제풀이"]
summary: "괄호 균형 확인. Stack으로 여는 괄호를 저장하고 닫는 괄호와 매칭."
---

## 문제

[Valid Parentheses - LeetCode #20](https://leetcode.com/problems/valid-parentheses/)
[Balanced Brackets - HackerRank](https://www.hackerrank.com/challenges/balanced-brackets/problem)

괄호 문자열이 올바르게 균형을 이루는지 확인한다.

## HackerRank 버전

```java
public static String isBalanced(String s) {
    Deque<Character> stack = new ArrayDeque<>();
    Map<Character, Character> pairs = new HashMap<>();
    pairs.put(')', '(');
    pairs.put('}', '{');
    pairs.put(']', '[');

    for (char c : s.toCharArray()) {
        if (pairs.containsValue(c)) {
            stack.push(c);
        } else {
            if (stack.isEmpty() || stack.pop() != pairs.get(c)) {
                return "NO";
            }
        }
    }
    return stack.isEmpty() ? "YES" : "NO";
}
// 시간: O(n), 공간: O(n)
```

## LeetCode 버전

```java
public boolean isValid(String s) {
    Deque<Character> stack = new ArrayDeque<>();
    Map<Character, Character> pairs = Map.of(')', '(', '}', '{', ']', '[');

    for (char c : s.toCharArray()) {
        if (pairs.containsValue(c)) {
            stack.push(c);
        } else {
            if (stack.isEmpty() || stack.pop() != pairs.get(c)) {
                return false;
            }
        }
    }
    return stack.isEmpty();
}
```

## 주의: 마지막 스택 체크 필수!

```java
// ❌ 잘못된 코드
return "YES";

// ✅ 올바른 코드
return stack.isEmpty() ? "YES" : "NO";
```

`"((("` 같은 입력에서 스택에 열린 괄호가 남아있으면 불균형이다.

## Stack vs Deque

`Stack` 대신 `Deque<> = new ArrayDeque<>()` 권장.

| 항목 | Stack | ArrayDeque |
|------|-------|------------|
| 상속 | `extends Vector` | `implements Deque` |
| 동기화 | synchronized | 없음 |
| 설계 | Java 1.0 레거시 | Java 6+ 권장 |
