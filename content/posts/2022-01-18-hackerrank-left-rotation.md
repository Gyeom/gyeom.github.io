---
title: "HackerRank - Left Rotation"
date: 2022-01-18
draft: true
tags: ["알고리즘", "Java", "배열", "HackerRank"]
categories: ["문제풀이"]
summary: "배열을 왼쪽으로 d칸 회전하기. 앞부분을 뒤로 보내면 된다."
---

## 문제

[Left Rotation - HackerRank](https://www.hackerrank.com/challenges/array-left-rotation/problem)

배열을 왼쪽으로 `d`칸 회전한다.

### 예시

```
arr = [1, 2, 3, 4, 5]
d = 2

결과: [3, 4, 5, 1, 2]
       ↑------↑ ↑--↑
       뒷부분   앞부분
```

## 풀이

```java
public static List<Integer> rotateLeft(int d, List<Integer> arr) {
    List<Integer> answers = new ArrayList<>();

    for (int i = d; i < arr.size(); i++) {
        answers.add(arr.get(i));
    }

    for (int i = 0; i < d; i++) {
        answers.add(arr.get(i));
    }

    return answers;
}
```

d번째부터 끝까지 먼저 넣고, 0번째부터 d-1번째까지 뒤에 붙인다.

### subList 버전

```java
public static List<Integer> rotateLeft(int d, List<Integer> arr) {
    List<Integer> result = new ArrayList<>();
    result.addAll(arr.subList(d, arr.size()));
    result.addAll(arr.subList(0, d));
    return result;
}
```

`subList()`로 범위를 잘라서 붙이면 더 간결하다.

### 한 줄 버전

```java
public static List<Integer> rotateLeft(int d, List<Integer> arr) {
    return Stream.concat(
        arr.subList(d, arr.size()).stream(),
        arr.subList(0, d).stream()
    ).collect(Collectors.toList());
}
```

## 핵심 포인트

### d가 배열 크기보다 클 때

```java
d = d % arr.size();  // 실제 회전 횟수
```

`d = 7`, `arr.size() = 5`면 실제로는 2칸 회전과 같다.

### In-place 회전 (추가 공간 없이)

면접에서 "O(1) 공간으로 할 수 있나요?" 물어볼 수 있다.

```java
// 1. 전체 뒤집기: [5,4,3,2,1]
// 2. 앞 (n-d)개 뒤집기: [3,4,5,2,1]
// 3. 뒤 d개 뒤집기: [3,4,5,1,2]
```

세 번 뒤집기로 O(1) 공간에 해결 가능하다.

## 시간복잡도

- O(n): 배열 한 번 순회
- 공간복잡도 O(n): 결과 배열 생성
