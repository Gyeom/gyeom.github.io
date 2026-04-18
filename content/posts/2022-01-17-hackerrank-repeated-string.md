---
title: "HackerRank - Repeated String"
date: 2022-01-17
draft: true
tags: ["알고리즘", "Java", "HackerRank"]
categories: ["문제풀이"]
summary: "무한 반복 문자열에서 처음 n개 문자 중 'a'의 개수 구하기. 나눗셈과 나머지로 최적화한다."
---

## 문제

[Repeated String - HackerRank](https://www.hackerrank.com/challenges/repeated-string/problem)

문자열 `s`를 무한 반복했을 때, 처음 `n`개 문자 중 `'a'`의 개수를 구한다.

### 예시

```
s = "aba"
n = 10

무한 반복: "abaabaabaa" (10자)
           ^  ^^  ^^  ^
'a' 개수 = 7
```

## 첫 시도 (오답)

```java
public static long repeatedString(String s, long n) {
    char[] cArray = s.toCharArray();
    int lastNumber = cArray[cArray.length - 1];
    long answer = 0;
    for (char c : s.toCharArray()) {
        if (c == lastNumber) {
            answer += 1;
        }
    }
    return answer;
}
```

**문제점:**
1. `'a'`가 아니라 마지막 문자와 비교
2. `n`을 전혀 사용하지 않음

문제를 잘못 읽었다. **항상 `'a'`를 세야 한다.**

## 풀이

```java
public static long repeatedString(String s, long n) {
    // 1. s 한 번에 'a' 몇 개?
    long countInS = 0;
    for (char c : s.toCharArray()) {
        if (c == 'a') countInS++;
    }

    // 2. s가 완전히 반복되는 횟수
    long fullRepeats = n / s.length();

    // 3. 나머지 부분 길이
    int remainder = (int) (n % s.length());

    // 4. 나머지 부분에서 'a' 개수
    long countInRemainder = 0;
    for (int i = 0; i < remainder; i++) {
        if (s.charAt(i) == 'a') countInRemainder++;
    }

    return countInS * fullRepeats + countInRemainder;
}
```

### Stream 버전

```java
public static long repeatedString(String s, long n) {
    long countInS = s.chars().filter(c -> c == 'a').count();
    long fullRepeats = n / s.length();
    int remainder = (int) (n % s.length());

    long countInRemainder = s.substring(0, remainder)
        .chars().filter(c -> c == 'a').count();

    return countInS * fullRepeats + countInRemainder;
}
```

## 핵심 포인트

### n이 매우 클 때

`n = 1,000,000,000,000` (1조)일 수 있다.

```java
// ❌ 시간 초과 - O(n)
for (int i = 0; i < n; i++) { ... }

// ✅ 수학으로 해결 - O(|s|)
long fullRepeats = n / s.length();
int remainder = (int) (n % s.length());
```

반복 횟수를 나눗셈으로 계산하면 O(|s|)로 끝난다.

### long vs int

`n`이 `long`이므로 계산 결과도 `long`으로 처리해야 한다.

```java
long countInS = 0;  // int 아님
return countInS * fullRepeats + countInRemainder;  // long 연산
```

## 시간복잡도

- O(|s|): 문자열 s 길이만큼만 순회
- 공간복잡도 O(1): 추가 자료구조 없음
