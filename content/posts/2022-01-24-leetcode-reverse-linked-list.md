---
title: "LeetCode - Reverse Linked List"
date: 2022-01-24
draft: true
tags: ["알고리즘", "Java", "LinkedList", "LeetCode", "HackerRank"]
categories: ["문제풀이"]
summary: "연결 리스트 뒤집기. 세 개의 포인터로 방향을 역전시킨다."
---

## 문제

[Reverse Linked List - LeetCode #206](https://leetcode.com/problems/reverse-linked-list/)
[Reverse a linked list - HackerRank](https://www.hackerrank.com/challenges/reverse-a-linked-list/problem)

연결 리스트의 순서를 뒤집는다.

## 풀이

```java
public static SinglyLinkedListNode reverse(SinglyLinkedListNode llist) {
    SinglyLinkedListNode curr = llist;
    SinglyLinkedListNode prev = null;

    while (curr != null) {
        SinglyLinkedListNode temp = curr.next;
        curr.next = prev;
        prev = curr;
        curr = temp;
    }
    return prev;
}
// 시간: O(n), 공간: O(1)
```

## 동작 과정

```
Before: 1 → 2 → 3 → null
After:  3 → 2 → 1 → null
```

매 반복마다 4단계 수행:

1. `temp = curr.next` (다음 노드 저장)
2. `curr.next = prev` (방향 역전)
3. `prev = curr` (prev 전진)
4. `curr = temp` (curr 전진)

## 단계별 추적

| 단계 | prev | curr | 연결 상태 |
|------|------|------|----------|
| 초기 | null | 1 | 1→2→3→null |
| 1회 | 1 | 2 | null←1, 2→3→null |
| 2회 | 2 | 3 | null←1←2, 3→null |
| 3회 | 3 | null | null←1←2←3 ✅ |

## 핵심

세 포인터(prev, curr, next)로 한 칸씩 이동하며 링크 방향을 뒤집는다.
- `prev`: 이전 노드 (뒤집힌 부분의 헤드)
- `curr`: 현재 처리 중인 노드
- `temp/next`: 다음에 처리할 노드
