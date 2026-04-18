---
title: "LeetCode - Merge Two Sorted Lists"
date: 2022-02-04
draft: true
tags: ["알고리즘", "Java", "연결리스트", "LeetCode", "HackerRank"]
categories: ["문제풀이"]
summary: "두 정렬된 연결 리스트 병합. 더미 헤드로 시작, 작은 값 연결, 남은 리스트 붙이기."
---

## 문제

- [Merge Two Sorted Lists - LeetCode #21](https://leetcode.com/problems/merge-two-sorted-lists/)
- [Merge two sorted linked lists - HackerRank](https://www.hackerrank.com/challenges/merge-two-sorted-linked-lists/problem)

두 정렬된 연결 리스트를 하나로 병합한다.

## 풀이

```java
public ListNode mergeTwoLists(ListNode list1, ListNode list2) {
    ListNode dummy = new ListNode(0);
    ListNode curr = dummy;

    while (list1 != null && list2 != null) {
        if (list1.val <= list2.val) {
            curr.next = list1;
            list1 = list1.next;
        } else {
            curr.next = list2;
            list2 = list2.next;
        }
        curr = curr.next;
    }

    curr.next = (list1 != null) ? list1 : list2;  // 남은 리스트 연결
    return dummy.next;
}
// 시간: O(n + m), 공간: O(1)
```

## 동작 예시

```
list1: 1 → 3 → 5
list2: 2 → 4

비교 1: 1 vs 2 → 1 선택
비교 2: 3 vs 2 → 2 선택
비교 3: 3 vs 4 → 3 선택
비교 4: 5 vs 4 → 4 선택
남은 것: 5 연결

결과: 1 → 2 → 3 → 4 → 5
```

## 왜 dummy 노드가 필요한가?

dummy 없이 하면 첫 번째 노드를 특별 처리해야 한다.

```java
// ❌ dummy 없이 - 복잡함
ListNode head = null;
ListNode curr = null;

// 첫 번째 노드 결정 (특별 처리!)
if (list1.val <= list2.val) {
    head = list1;
    list1 = list1.next;
} else {
    head = list2;
    list2 = list2.next;
}
curr = head;
// 이후 while 루프...
```

| 방식 | 장점 | 단점 |
|------|------|------|
| dummy 사용 | 코드 깔끔, 엣지케이스 쉬움 | 노드 1개 생성 |
| dummy 없음 | 메모리 절약 | 첫 노드 특별 처리 |

## 마지막 줄이 왜 필요한가?

```java
curr.next = (list1 != null) ? list1 : list2;
```

while 조건이 `list1 != null && list2 != null`이므로 **둘 중 하나라도 null이 되면 종료**된다.

남은 리스트가 있으면 **통째로 연결**. 이미 정렬되어 있으니 개별 비교 불필요.
