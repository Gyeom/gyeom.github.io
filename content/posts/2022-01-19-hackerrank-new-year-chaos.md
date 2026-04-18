---
title: "HackerRank - New Year Chaos"
date: 2022-01-19
draft: true
tags: ["알고리즘", "Java", "배열", "HackerRank"]
categories: ["문제풀이"]
summary: "놀이공원 줄서기에서 뇌물 횟수 계산하기. 원래 위치와 현재 위치를 비교한다."
---

## 문제

[New Year Chaos - HackerRank](https://www.hackerrank.com/challenges/new-year-chaos/problem)

놀이공원 줄에서 각 사람은 **초기 위치가 적힌 스티커**를 붙이고 있다.

- 초기 상태: `[1, 2, 3, ..., n]` (항상 순서대로)
- 각 사람은 **최대 2번**까지 앞사람에게 뇌물을 줘서 자리를 바꿀 수 있다
- 최종 줄 상태를 보고 **총 뇌물 횟수**를 구한다
- 누군가 2번 초과로 뇌물을 줬으면 `"Too chaotic"` 출력

## 입력 이해

```
2           ← 테스트 케이스 수
5           ← 사람 수
2 1 5 3 4   ← 현재 줄 상태 (초기: 1,2,3,4,5)
5
2 5 1 3 4
```

## 예시 분석

### 케이스 1: `[2, 1, 5, 3, 4]`

```
초기:  [1, 2, 3, 4, 5]
         ↓
현재:  [2, 1, 5, 3, 4]
```

| 사람 | 원래 위치 | 현재 위치 | 이동 | 뇌물 |
|------|----------|----------|------|------|
| 2 | 1 | 0 | 1칸 앞 | OK |
| 5 | 4 | 2 | 2칸 앞 | OK |

뇌물 세기: 나보다 큰 숫자가 내 앞에 있으면 그 사람이 뇌물 준 것

```
위치 0: 2 → 앞에 아무도 없음 (0)
위치 1: 1 → 앞에 2 있음, 2>1 (1)
위치 2: 5 → 앞에 2,1 다 작음 (0)
위치 3: 3 → 앞에서 5>3 (1)
위치 4: 4 → 앞에서 5>4 (1)

총: 3
```

### 케이스 2: `[2, 5, 1, 3, 4]`

```
5번이 인덱스 1에 있음
원래 위치: 4 (0-indexed)
이동: 3칸 앞으로 → 2칸 초과!

답: "Too chaotic"
```

## 풀이

```java
public static void minimumBribes(List<Integer> q) {
    int bribes = 0;

    for (int i = 0; i < q.size(); i++) {
        int person = q.get(i);

        // 3칸 이상 앞으로 왔으면 불가능
        if (person - 1 - i > 2) {
            System.out.println("Too chaotic");
            return;
        }

        // 나보다 앞에 있는 사람 중, 나보다 큰 숫자 세기
        // 최적화: person-2 위치부터만 체크 (그 앞은 절대 못 옴)
        for (int j = Math.max(0, person - 2); j < i; j++) {
            if (q.get(j) > person) {
                bribes++;
            }
        }
    }
    System.out.println(bribes);
}
```

## 핵심 포인트

### Too chaotic 판단

```java
if (person - 1 - i > 2)
```

- `person - 1`: 원래 위치 (0-indexed)
- `i`: 현재 위치
- 차이가 2 초과면 불가능

### 뇌물 수 세기 최적화

```java
for (int j = Math.max(0, person - 2); j < i; j++)
```

왜 `person - 2`부터?
- 5번 사람이 최대 2칸 앞으로 올 수 있음
- 그러면 원래 3번 위치(index 2)까지만 가능
- 그보다 앞(index 0, 1)에서 5를 만날 일은 없음

**전체 탐색 O(n²)** vs **최적화 O(n)**: 실제로는 각 위치에서 최대 2~3개만 체크하므로 거의 O(n)

### void 리턴과 System.out.println

이 문제는 리턴값이 없고 직접 출력한다. HackerRank 문제 중 이런 형태가 종종 있다.

## 시간복잡도

- O(n): 최적화된 버전
- O(n²): 최악의 경우 (실제로는 상수에 가까움)
