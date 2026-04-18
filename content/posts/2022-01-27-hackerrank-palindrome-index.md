---
title: "HackerRank - Palindrome Index"
date: 2022-01-27
draft: true
tags: ["알고리즘", "Java", "문자열", "투포인터", "HackerRank"]
categories: ["문제풀이"]
summary: "한 글자만 제거해서 팰린드롬 만들기. 양 끝에서 비교하다 다르면 둘 중 하나 제거 시도."
---

## 문제

[Palindrome Index - HackerRank](https://www.hackerrank.com/challenges/palindrome-index/problem)

문자열에서 **한 글자를 제거**해서 팰린드롬을 만들 수 있는 **인덱스**를 찾는다.
이미 팰린드롬이거나 불가능하면 -1 반환.

## 풀이

```java
public static int palindromeIndex(String s) {
    int left = 0, right = s.length() - 1;

    while (left < right) {
        if (s.charAt(left) != s.charAt(right)) {
            // 왼쪽 제거하면 팰린드롬?
            if (isPalindrome(s, left + 1, right)) return left;
            // 오른쪽 제거하면 팰린드롬?
            if (isPalindrome(s, left, right - 1)) return right;
            return -1;  // 둘 다 안 됨
        }
        left++;
        right--;
    }
    return -1;  // 이미 팰린드롬
}

private static boolean isPalindrome(String s, int left, int right) {
    while (left < right) {
        if (s.charAt(left++) != s.charAt(right--)) return false;
    }
    return true;
}
// 시간: O(n), 공간: O(1)
```

## 동작 예시

```
"aaab"
 ↑  ↑
 0  3   → 'a' != 'b'
         → left+1 제거: "aab" → 팰린드롬 아님
         → right 제거: "aaa" → 팰린드롬!
         → return 3
```

## 테스트 케이스

| 입력 | 출력 | 이유 |
|------|------|------|
| `aaab` | 3 | 'b' 제거 → "aaa" |
| `baa` | 0 | 'b' 제거 → "aa" |
| `aaa` | -1 | 이미 팰린드롬 |

## 복잡도

- **시간 O(n)**: 전체 순회 1번 + 확인용 순회 1번 = 2n
- **공간 O(1)**: 포인터 변수만 사용
