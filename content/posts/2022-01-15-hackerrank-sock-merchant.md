---
title: "HackerRank - Sock Merchant"
date: 2022-01-15
draft: true
tags: ["알고리즘", "Java", "HashMap", "HackerRank"]
categories: ["문제풀이"]
summary: "양말 짝 맞추기 문제. HashMap으로 개수를 세면서 짝이 완성될 때마다 카운트한다."
---

## 문제

[Sock Merchant - HackerRank](https://www.hackerrank.com/challenges/sock-merchant/problem)

양말 더미에서 색깔별로 짝을 맞춰 판매할 수 있는 쌍의 개수를 구한다.

## 풀이

HashMap으로 각 색깔의 개수를 세면서, 짝이 완성될 때마다 카운트한다.

```java
public static int sockMerchant(int n, List<Integer> ar) {
    Map<Integer, Integer> socksMap = new HashMap<Integer, Integer>();
    int sum = 0;
    for (int item : ar) {
        int value = socksMap.getOrDefault(item, 0);
        if ((value + 1) % 2 == 0) {
            sum += 1;
        }
        socksMap.put(item, value + 1);
    }
    return sum;
}
```

### 개선 버전

`merge()`를 사용하면 더 간결해진다.

```java
public static int sockMerchant(int n, List<Integer> ar) {
    Map<Integer, Integer> count = new HashMap<>();
    int pairs = 0;
    for (int sock : ar) {
        int newCount = count.merge(sock, 1, Integer::sum);
        if (newCount % 2 == 0) pairs++;
    }
    return pairs;
}
```

## 핵심 포인트

### getOrDefault vs merge

처음에는 `getOrDefault`로 풀었다.

```java
int value = count.getOrDefault(sock, 0);
count.put(sock, value + 1);
```

`merge()`를 쓰면 한 줄로 줄어든다.

```java
count.merge(sock, 1, Integer::sum);  // 없으면 1, 있으면 +1
```

### Map vs HashMap 선언

```java
Map<Integer, Integer> count = new HashMap<>();  // 권장
HashMap<Integer, Integer> count = new HashMap<>();  // 동작하지만 비권장
```

인터페이스로 선언하면 나중에 TreeMap, LinkedHashMap으로 교체가 쉽다.

### NullPointerException 주의

```java
int value = map.get(key);  // 키 없으면 null → NPE 발생
int value = map.getOrDefault(key, 0);  // 안전
```

## 시간복잡도

- O(n): 리스트를 한 번 순회
- 공간복잡도 O(k): k는 양말 색깔 종류 수
