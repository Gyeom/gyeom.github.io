---
title: "LeetCode - Linked List Cycle"
date: 2022-01-25
draft: true
tags: ["알고리즘", "Java", "LinkedList", "투포인터", "LeetCode", "HackerRank"]
categories: ["문제풀이"]
summary: "연결 리스트에 사이클이 있는지 확인. Floyd의 토끼와 거북이 알고리즘."
---

## 문제

[Linked List Cycle - LeetCode #141](https://leetcode.com/problems/linked-list-cycle/)
[Cycle Detection - HackerRank](https://www.hackerrank.com/challenges/detect-whether-a-linked-list-contains-a-cycle/problem)

연결 리스트에 **사이클(순환)**이 있는지 확인한다.

## 풀이: Floyd's Tortoise and Hare

```java
static boolean hasCycle(SinglyLinkedListNode head) {
    if (head == null) return false;

    SinglyLinkedListNode slow = head;
    SinglyLinkedListNode fast = head;

    while (fast != null && fast.next != null) {
        slow = slow.next;        // 1칸 이동
        fast = fast.next.next;   // 2칸 이동

        if (slow == fast) {
            return true;  // 만나면 사이클 존재
        }
    }
    return false;  // fast가 끝에 도달하면 사이클 없음
}
// 시간: O(n), 공간: O(1)
```

## 동작 원리

```
사이클 있는 경우:
1 → 2 → 3 → 4
        ↑   ↓
        ← ←

slow: 1 → 2 → 3 → 4 → 3 → 4 ...
fast: 1 → 3 → 3 → 3 ...
               ↑ 만남!
```

- slow는 1칸씩, fast는 2칸씩 이동
- 사이클이 있으면 fast가 slow를 **반드시 따라잡음**
- 사이클이 없으면 fast가 `null`에 도달

## 흔한 실수: 데이터 값 비교

```java
// ❌ 잘못된 접근
if (curr.data >= curr.next.data) {
    return true;  // 값이 정렬되어 있다고 가정?
}
```

**문제점**: 노드의 데이터 값은 **정렬되어 있지 않다!**

사이클 판정은 **노드 포인터**가 같은지 비교해야 한다.

```java
// ✅ 올바른 접근
if (slow == fast) {  // 포인터 비교
    return true;
}
```

## 핵심

| 항목 | 설명 |
|------|------|
| 알고리즘 | Floyd's Cycle Detection |
| 비교 대상 | 노드 포인터 (값 X) |
| 시간 복잡도 | O(n) |
| 공간 복잡도 | O(1) - HashSet 불필요 |
