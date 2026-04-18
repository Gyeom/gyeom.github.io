---
title: "HackerRank - Counting Valleys"
date: 2022-01-16
draft: true
tags: ["알고리즘", "Java", "HackerRank"]
categories: ["문제풀이"]
summary: "등산 경로에서 골짜기 개수 세기. 해수면 아래로 내려갔다가 복귀하는 순간을 카운트한다."
---

## 문제

[Counting Valleys - HackerRank](https://www.hackerrank.com/challenges/counting-valleys/problem)

등산가가 해수면(sea level)에서 출발해 하이킹을 한다.

- `U` = 한 걸음 올라감 (+1)
- `D` = 한 걸음 내려감 (-1)

**골짜기(Valley)**: 해수면 아래로 내려갔다가 다시 해수면으로 올라오는 구간

```
        /\
해수면 -----/--\----/-----
           \  /
            \/     ← 골짜기 1개
```

### 예시

`UDDDUDUU` 경로를 따라가면:

| Step | U | D | D | D | U | D | U | U |
|------|---|---|---|---|---|---|---|---|
| Level | 1 | 0 | -1 | -2 | -1 | -2 | -1 | **0** |

레벨이 -1 → 0으로 바뀌는 순간이 골짜기 탈출. 답은 1개.

## 풀이

```java
public static int countingValleys(int steps, String path) {
    String[] directions = path.split("");
    int answer = 0;
    int current = 0;
    for (String d : directions) {
        if ("U".equals(d)) {
            if (current == -1) {
                answer += 1;
            }
            current += 1;
        } else {
            current -= 1;
        }
    }
    return answer;
}
```

핵심은 **레벨이 -1일 때 U를 만나면** 해수면 복귀이므로 골짜기 1개 완성.

### 개선 버전

`toCharArray()`를 사용하면 String 배열 생성 오버헤드가 없다.

```java
public static int countingValleys(int steps, String path) {
    int level = 0, valleys = 0;
    for (char step : path.toCharArray()) {
        if (step == 'U') {
            if (level == -1) valleys++;
            level++;
        } else {
            level--;
        }
    }
    return valleys;
}
```

## 핵심 포인트

### split("") vs toCharArray()

| 방식 | 특징 |
|------|------|
| `split("")` | String[] 배열 생성, 각 문자가 String 객체 |
| `toCharArray()` | char[] 배열 생성, primitive 타입으로 가벼움 |

문자 단위 순회가 목적이면 `toCharArray()`가 효율적이다.

### 골짜기 vs 산

- **골짜기**: 해수면 아래(-) → 해수면(0) 복귀
- **산**: 해수면 위(+) → 해수면(0) 복귀

산을 세려면 `level == 1`일 때 `D`를 만나는 순간을 카운트하면 된다.

## 시간복잡도

- O(n): 경로를 한 번 순회
- 공간복잡도 O(n): `toCharArray()`가 새 배열 생성 (O(1)로 하려면 `charAt()` 사용)
