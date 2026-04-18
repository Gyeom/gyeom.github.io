---
title: "LeetCode & HackerRank - Anagram 문제 비교"
date: 2022-01-28
draft: true
tags: ["알고리즘", "Java", "문자열", "빈도배열", "LeetCode", "HackerRank"]
categories: ["문제풀이"]
summary: "LeetCode는 두 문자열 비교, HackerRank는 반으로 나눠 최소 변경 횟수 계산."
---

## 문제 비교

| 문제 | 플랫폼 | 입력 | 반환 |
|------|--------|------|------|
| Valid Anagram | LeetCode #242 | 두 문자열 s, t | boolean |
| Anagram | HackerRank | 한 문자열 s | int (최소 변경 횟수) |

---

## LeetCode - Valid Anagram

[Valid Anagram - LeetCode #242](https://leetcode.com/problems/valid-anagram/)

두 문자열이 아나그램인지 확인.

```java
public boolean isAnagram(String s, String t) {
    if (s.length() != t.length()) return false;

    int[] count = new int[26];
    for (int i = 0; i < s.length(); i++) {
        count[s.charAt(i) - 'a']++;
        count[t.charAt(i) - 'a']--;
    }

    for (int c : count) {
        if (c != 0) return false;
    }
    return true;
}
// 시간: O(n), 공간: O(1)
```

**핵심**: 한 문자열은 +1, 다른 문자열은 -1 → 최종 0이면 아나그램

---

## HackerRank - Anagram

[Anagram - HackerRank](https://www.hackerrank.com/challenges/anagram/problem)

문자열을 **반으로 나눠** 아나그램 만들기 위한 최소 변경 횟수.

```java
public static int anagram(String s) {
    int n = s.length();
    if (n % 2 != 0) return -1;  // 홀수면 불가

    int half = n / 2;
    String first = s.substring(0, half);
    String second = s.substring(half);

    int[] count = new int[26];
    for (char c : first.toCharArray()) {
        count[c - 'a']++;
    }

    for (char c : second.toCharArray()) {
        if (count[c - 'a'] > 0) {
            count[c - 'a']--;
        }
    }

    int changes = 0;
    for (int c : count) {
        changes += c;
    }
    return changes;
}
// 시간: O(n), 공간: O(1)
```

| 입력 | first | second | 출력 | 이유 |
|------|-------|--------|------|------|
| `aaabbb` | aaa | bbb | 3 | a 3개를 b로 변경 |
| `ab` | a | b | 1 | a를 b로 변경 |
| `abc` | - | - | -1 | 홀수라 분할 불가 |

---

## 흔한 실수

**1. 나누기 vs 나머지 혼동**

```java
// ❌ % = 나머지 (6 % 2 = 0)
String groupA = s.substring(0, s.length() % 2);  // "" 빈 문자열!

// ✅ / = 나누기 (6 / 2 = 3)
String groupA = s.substring(0, s.length() / 2);  // "aaa"
```

**2. 양쪽 차이를 다 더함 (2배 오류)**

```java
// ❌ "aaabbb" → 6 (오답)
count += Math.abs(tempA[i] - tempB[i]);

// ✅ 한쪽만 카운트 → 3 (정답)
```
