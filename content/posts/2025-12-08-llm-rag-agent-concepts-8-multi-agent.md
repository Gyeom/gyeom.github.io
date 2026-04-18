---
title: "LLM 개념 정리 (8) - Multi-Agent 패턴"
date: 2025-12-08
draft: true
tags: ["AI-Agent", "Multi-Agent", "LLM", "개념정리"]
categories: ["AI/ML"]
series: ["LLM/RAG/Agent 개념 정리"]
summary: "여러 Agent가 협력하는 Multi-Agent 패턴을 정리한다. Supervisor, Worker, Reflection, Debate 아키텍처까지."
---

## 개요

Multi-Agent는 여러 Agent가 협력하여 복잡한 태스크를 수행하는 아키텍처다. 각 Agent가 전문 역할을 맡아 분업한다.

```mermaid
flowchart LR
    U["사용자"] --> S["Supervisor"]
    S --> W1["Worker 1"]
    S --> W2["Worker 2"]
    S --> W3["Worker 3"]
    W1 --> S
    W2 --> S
    W3 --> S
    S --> R["결과"]
```

---

## Single Agent vs Multi-Agent

| 구분 | Single Agent | Multi-Agent |
|------|--------------|-------------|
| 구조 | 하나의 LLM | 여러 LLM 협력 |
| 복잡도 | 낮음 | 높음 |
| 전문성 | 범용 | 역할별 특화 |
| 확장성 | 제한적 | 유연함 |
| 비용 | 낮음 | 높음 |

**Multi-Agent가 적합한 경우**:
- 여러 전문 영역이 필요한 태스크
- 검토/개선이 필요한 출력물
- 병렬 처리로 속도 향상 가능

---

## 기본 패턴

### Supervisor-Worker

Supervisor가 작업을 분배하고 Worker들이 실행한다.

```mermaid
flowchart TB
    S["Supervisor<br/>작업 분배, 결과 통합"]
    S --> W1["Worker 1<br/>검색"]
    S --> W2["Worker 2<br/>분석"]
    S --> W3["Worker 3<br/>작성"]
    W1 --> S
    W2 --> S
    W3 --> S
```

**Supervisor 역할**:
- 태스크 분해
- 적절한 Worker 선택
- 결과 통합 및 품질 확인

**Worker 역할**:
- 특정 태스크 전문 수행
- 결과 반환

### Hierarchical (계층적)

여러 레벨의 Supervisor가 존재한다.

```mermaid
flowchart TB
    S["최상위 Supervisor"]
    S --> M1["중간 Supervisor 1"]
    S --> M2["중간 Supervisor 2"]
    M1 --> W1["Worker"]
    M1 --> W2["Worker"]
    M2 --> W3["Worker"]
    M2 --> W4["Worker"]
```

복잡한 조직 구조나 대규모 태스크에 적합하다.

### Peer-to-Peer

Agent들이 동등하게 협력한다.

```mermaid
flowchart LR
    A1["Agent 1"] <--> A2["Agent 2"]
    A2 <--> A3["Agent 3"]
    A3 <--> A1
```

---

## 품질 향상 패턴

### Reflection

Agent가 자신의 출력을 검토하고 개선한다.

```mermaid
flowchart LR
    G["Generator"] --> O["출력"]
    O --> R["Reflector"]
    R --> F["피드백"]
    F --> G
```

```
Generator: 코드 작성
Reflector: "이 코드에 에러 처리가 없다. 예외 상황을 처리해야 한다."
Generator: 에러 처리 추가하여 재작성
```

### Self-Critique

같은 Agent가 생성과 비평을 번갈아 수행한다.

```mermaid
flowchart LR
    A["Agent"] --> G["생성 모드"]
    G --> C["비평 모드"]
    C --> G
    C --> F["최종 출력"]
```

Reflection과 유사하지만 단일 Agent 내에서 수행한다.

### Debate

여러 Agent가 토론하여 최선의 답을 도출한다.

```mermaid
flowchart LR
    Q["질문"] --> A1["Agent 1<br/>의견 A"]
    Q --> A2["Agent 2<br/>의견 B"]
    A1 --> D["토론"]
    A2 --> D
    D --> J["Judge<br/>최종 결정"]
```

| 단계 | 설명 |
|------|------|
| 1. 개별 응답 | 각 Agent가 독립적으로 답변 |
| 2. 토론 | 서로의 의견 비평 |
| 3. 판정 | Judge가 최종 결론 |

복잡한 판단이 필요한 경우 다양한 관점을 반영한다.

---

## Role-based Agent

각 Agent에 특정 역할(페르소나)을 부여한다.

### 역할 예시

| 역할 | 전문 영역 |
|------|-----------|
| Researcher | 정보 검색, 조사 |
| Coder | 코드 작성 |
| Reviewer | 코드 리뷰, 품질 검토 |
| Writer | 문서 작성 |
| Critic | 비평, 개선점 제안 |

### 소프트웨어 개발 예시

```mermaid
flowchart LR
    U["요구사항"] --> PM["PM Agent"]
    PM --> DEV["Developer Agent"]
    DEV --> C["코드"]
    C --> REV["Reviewer Agent"]
    REV -->|피드백| DEV
    REV -->|승인| T["Tester Agent"]
    T --> R["결과"]
```

---

## Agent Orchestration

여러 Agent의 실행 흐름을 관리한다.

### 순차 실행

```mermaid
flowchart LR
    A1["Agent 1"] --> A2["Agent 2"] --> A3["Agent 3"]
```

이전 Agent 결과가 다음 Agent 입력이 된다.

### 병렬 실행

```mermaid
flowchart LR
    S["시작"] --> A1["Agent 1"]
    S --> A2["Agent 2"]
    S --> A3["Agent 3"]
    A1 --> E["통합"]
    A2 --> E
    A3 --> E
```

독립적인 태스크를 동시 처리하여 속도를 높인다.

### 조건부 분기

```mermaid
flowchart LR
    I["입력"] --> C{"분류"}
    C -->|기술 질문| T["Tech Agent"]
    C -->|일반 질문| G["General Agent"]
    C -->|창작 요청| W["Writer Agent"]
```

입력에 따라 적절한 Agent로 라우팅한다.

---

## 통신 방식

### 직접 통신

Agent끼리 메시지를 직접 주고받는다.

### 공유 메모리

중앙 메모리에 상태를 저장하고 읽는다.

```mermaid
flowchart TB
    M["공유 메모리"]
    A1["Agent 1"] --> M
    A2["Agent 2"] --> M
    A3["Agent 3"] --> M
    M --> A1
    M --> A2
    M --> A3
```

### 메시지 큐

비동기 메시지로 통신한다.

| 방식 | 장점 | 단점 |
|------|------|------|
| 직접 통신 | 단순 | 결합도 높음 |
| 공유 메모리 | 상태 공유 용이 | 동시성 관리 |
| 메시지 큐 | 비동기, 확장성 | 복잡도 |

---

## 실무 고려사항

### 비용 관리

| 전략 | 설명 |
|------|------|
| Agent 수 최소화 | 필요한 역할만 |
| 작은 모델 활용 | 간단한 판단은 저렴한 모델 |
| 캐싱 | 반복 작업 결과 재사용 |

### 오류 처리

| 상황 | 대응 |
|------|------|
| Agent 실패 | 재시도 또는 대체 Agent |
| 무한 루프 | 최대 반복 제한 |
| 불일치 | Supervisor 중재 |

---

## 정리

| 패턴 | 특징 |
|------|------|
| Supervisor-Worker | 중앙 조정, 분업 |
| Hierarchical | 다단계 관리 |
| Reflection | 자기 검토 |
| Debate | 다관점 토론 |
| Role-based | 역할 전문화 |

```mermaid
flowchart LR
    U["복잡한 태스크"] --> O["Orchestrator"]
    O --> A1["전문 Agent 1"]
    O --> A2["전문 Agent 2"]
    A1 --> R["Reflection"]
    A2 --> R
    R --> F["최종 결과"]
```

**다음 편**: Tool Calling - Agent가 외부 도구를 호출하는 메커니즘을 다룬다.
