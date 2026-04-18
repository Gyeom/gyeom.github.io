---
title: "Casbin vs OpenFGA: ReBAC 구현체 심층 비교"
date: 2025-04-15
draft: true
tags: ["Casbin", "OpenFGA", "ReBAC", "Authorization", "Zanzibar"]
summary: "임베디드 라이브러리 Casbin과 Zanzibar 기반 서비스 OpenFGA를 아키텍처, 모델링, 확장성, 일관성 측면에서 비교하고 선택 기준을 제시한다."
---

권한 관리 시스템을 구축할 때 Casbin과 OpenFGA는 자주 비교되는 선택지다. 둘 다 ReBAC(Relationship-Based Access Control)을 지원하지만, 근본적인 아키텍처와 철학이 다르다. 이 글에서는 두 시스템을 심층 비교한다.

## 핵심 차이: 라이브러리 vs 서비스

### Casbin: 임베디드 라이브러리

Casbin은 애플리케이션에 **직접 임베드**하는 라이브러리다.

```go
// Go 애플리케이션에 Casbin 임베드
import "github.com/casbin/casbin/v2"

func main() {
    enforcer, _ := casbin.NewEnforcer("model.conf", "policy.csv")

    // 권한 체크 - 로컬 메모리에서 실행
    allowed, _ := enforcer.Enforce("alice", "data1", "read")
}
```

**특징**:
- 애플리케이션 프로세스 내에서 실행
- 정책을 메모리에 로드
- 네트워크 호출 없음
- 15개 이상 언어 지원 (Go, Java, Python, Node.js, PHP, Rust 등)

### OpenFGA: 중앙화된 서비스

OpenFGA는 **별도의 서비스**로 배포한다.

```go
// OpenFGA 서비스에 HTTP/gRPC 호출
import "github.com/openfga/go-sdk"

func main() {
    client, _ := openfga.NewSdkClient(&openfga.ClientConfiguration{
        ApiUrl: "http://openfga:8080",
    })

    // 권한 체크 - 네트워크 호출
    result, _ := client.Check(context.Background()).Body(openfga.CheckRequest{
        TupleKey: openfga.TupleKey{
            User:     "user:alice",
            Relation: "viewer",
            Object:   "document:doc-1",
        },
    }).Execute()
}
```

**특징**:
- 독립된 서비스로 배포
- 관계 튜플을 전용 데이터베이스에 저장
- Google Zanzibar 논문 기반 설계
- CNCF Incubating 프로젝트

## 아키텍처 비교

```
┌─────────────────────────────────────────────────────────────────────┐
│                         Casbin 아키텍처                              │
│                                                                      │
│   ┌─────────────────────────────────────────────────────────────┐   │
│   │                    Application Process                       │   │
│   │   ┌─────────────┐    ┌─────────────┐    ┌─────────────────┐ │   │
│   │   │ Application │───▶│   Casbin    │◀───│ Policy Storage  │ │   │
│   │   │    Code     │    │  Enforcer   │    │ (DB/File/Redis) │ │   │
│   │   └─────────────┘    └──────┬──────┘    └─────────────────┘ │   │
│   │                             │                                │   │
│   │                    메모리 내 정책 평가                         │   │
│   └─────────────────────────────────────────────────────────────┘   │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│                        OpenFGA 아키텍처                              │
│                                                                      │
│   ┌─────────────┐         ┌─────────────────────────────────────┐   │
│   │ Application │         │           OpenFGA Server            │   │
│   │             │──HTTP──▶│  ┌─────────────────────────────┐    │   │
│   │             │  gRPC   │  │     Authorization Engine    │    │   │
│   └─────────────┘         │  │  (Zanzibar Graph Traversal) │    │   │
│                           │  └──────────────┬──────────────┘    │   │
│                           │                 │                    │   │
│                           │  ┌──────────────▼──────────────┐    │   │
│                           │  │   PostgreSQL / MySQL        │    │   │
│                           │  │   (Relation Tuples)         │    │   │
│                           │  └─────────────────────────────┘    │   │
│                           └─────────────────────────────────────┘   │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

## 모델링 방식 비교

### Casbin: PERM 메타모델

Casbin은 **PERM**(Policy, Effect, Request, Matchers) 메타모델을 사용한다.

```ini
# model.conf - RBAC with domains
[request_definition]
r = sub, dom, obj, act

[policy_definition]
p = sub, dom, obj, act

[role_definition]
g = _, _, _

[policy_effect]
e = some(where (p.eft == allow))

[matchers]
m = g(r.sub, p.sub, r.dom) && r.dom == p.dom && r.obj == p.obj && r.act == p.act
```

```csv
# policy.csv
p, admin, domain1, data1, read
p, admin, domain1, data1, write
g, alice, admin, domain1
```

**특징**:
- 선언적 설정 파일로 모델 정의
- 커스텀 Matcher 표현식 지원
- ACL, RBAC, ABAC, ReBAC 등 다양한 모델 지원
- 정책 변경 시 설정 파일만 수정

### OpenFGA: Zanzibar 튜플 모델

OpenFGA는 **관계 튜플**(Relation Tuple) 기반이다.

```yaml
# model.fga (DSL)
model
  schema 1.1

type user

type document
  relations
    define owner: [user]
    define editor: [user] or owner
    define viewer: [user] or editor

type folder
  relations
    define owner: [user]
    define viewer: [user] or owner
    define parent: [folder]

    # 폴더의 문서는 폴더 권한 상속
    define document_viewer: viewer or viewer from parent
```

```json
// 관계 튜플 작성
{
  "writes": [
    {"user": "user:alice", "relation": "owner", "object": "folder:engineering"},
    {"user": "folder:engineering", "relation": "parent", "object": "document:design-doc"}
  ]
}
```

**특징**:
- 관계를 일급 시민(first-class citizen)으로 취급
- 그래프 순회로 권한 계산
- 상속과 조합을 선언적으로 표현
- 스키마 마이그레이션 지원

## ReBAC 구현 방식 비교

### Casbin의 ReBAC

Casbin에서 ReBAC을 구현하려면 `g` (그룹/역할) 정의를 확장한다.

```ini
# model.conf - ReBAC
[request_definition]
r = sub, obj, act

[policy_definition]
p = role, obj_type, act

[role_definition]
g = _, _, _      # user, object, role 관계
g2 = _, _        # object, type 관계

[policy_effect]
e = some(where (p.eft == allow))

[matchers]
m = g(r.sub, r.obj, p.role) && g2(r.obj, p.obj_type) && r.act == p.act
```

```csv
# policy.csv
# 역할 정의: collaborator는 doc 타입에 read 가능
p, collaborator, doc, read

# 관계: alice는 doc1의 collaborator
g, alice, doc1, collaborator

# 타입: doc1은 doc 타입
g2, doc1, doc
```

**한계**:
- 깊은 상속 관계는 명시적으로 풀어야 한다
- `g` 테이블 크기가 커지면 성능 저하
- 역방향 쿼리 (어떤 문서에 접근 가능한가?)가 어렵다

### OpenFGA의 ReBAC

OpenFGA는 ReBAC을 위해 설계되었다.

```yaml
# 폴더 → 문서 상속이 자동으로 계산됨
type folder
  relations
    define viewer: [user, team#member]
    define parent: [folder]

type document
  relations
    define parent: [folder]
    define viewer: [user] or viewer from parent
```

```
user:alice → viewer → folder:engineering
folder:engineering → parent → document:design-doc

# 자동 계산: alice는 design-doc의 viewer
Check(user:alice, viewer, document:design-doc) = true
```

**장점**:
- 상속 관계를 자동으로 그래프 순회
- `ListObjects`로 접근 가능한 객체 목록 조회
- 조건부 권한 (Contextual Tuples) 지원

## 확장성 비교

### Casbin 확장성

Casbin은 인메모리 정책 평가가 기본이다.

```
문제: 수백만 정책 + 초당 10,000 요청
```

**해결책**:
1. **멀티스레딩**: 여러 Enforcer 인스턴스 생성
2. **클러스터 배포**: Watcher로 정책 동기화
3. **필터링된 정책 로드**: 필요한 정책만 로드

```go
// 필터링된 정책 로드
adapter := gormadapter.NewAdapter("mysql", "root:@tcp(127.0.0.1:3306)/")
enforcer.LoadFilteredPolicy(&gormadapter.Filter{
    V0: []string{"alice"},  // alice 관련 정책만 로드
})
```

**Watcher 패턴**:

```go
// Redis Watcher로 분산 동기화
watcher, _ := rediswatcher.NewWatcher("localhost:6379")
enforcer.SetWatcher(watcher)

// 정책 변경 시 다른 인스턴스에 전파
enforcer.AddPolicy("alice", "data1", "read")
// → Redis Pub/Sub으로 변경 알림
// → 다른 인스턴스가 정책 다시 로드
```

**한계**:
- 개발자가 직접 분산 아키텍처 구현
- 정책 동기화 지연
- 일관성 보장이 어려움

### OpenFGA 확장성

OpenFGA는 분산 환경을 고려해 설계되었다.

```yaml
# 수평 확장
OpenFGA Server (Stateless)
├── Instance 1
├── Instance 2
└── Instance 3
        │
        ▼
   PostgreSQL (Shared)
```

**내장 최적화**:
- 그래프 순회 최적화 (BFS, 캐싱)
- 배치 체크 (`BatchCheck`)
- 스트리밍 응답 (`StreamedListObjects`)
- 동시성 제어 (`maxConcurrentReads`)

**한계**:
- `ListObjects`는 여전히 비용이 높다
- 대규모에서는 [Materialize 패턴](/posts/2025-12-03-rebac-pagination-strategy-guide/) 필요

## 일관성 모델 비교

### Casbin: 수동 관리

Casbin은 일관성을 **개발자가 관리**해야 한다.

```
문제: New Enemy Problem
1. Alice가 Bob을 폴더에서 제거 (Instance A)
2. Charlie가 새 문서 추가 (Instance B)
3. Instance B가 아직 정책 동기화 안 됨
4. Bob이 새 문서에 접근 가능 (Instance B 통해)
```

**완화 방법**:
- Watcher로 동기화 시간 최소화
- 중요한 작업 전 정책 강제 리로드
- 단일 인스턴스 사용 (확장성 포기)

### OpenFGA: Consistency 옵션

OpenFGA는 **일관성 수준을 선택**할 수 있다.

```go
// 높은 일관성 (캐시 무시)
client.Check(ctx).Body(openfga.CheckRequest{
    // ...
    Consistency: openfga.HIGHER_CONSISTENCY,
}).Execute()

// 낮은 레이턴시 (캐시 사용)
client.Check(ctx).Body(openfga.CheckRequest{
    // ...
    Consistency: openfga.MINIMIZE_LATENCY,
}).Execute()
```

**Zookies (로드맵)**:

Google Zanzibar의 Zookies 개념을 도입 예정이다.

```
Write → Zookie 반환 → 다음 Read에 Zookie 전달
→ "이 시점 이후의 권한 상태로 체크"
```

현재 OpenFGA는 완전한 Zookies를 지원하지 않지만, SpiceDB는 지원한다.

## 역방향 쿼리 비교

**"Alice가 접근할 수 있는 문서는?"** - 이 질문에 답하기가 얼마나 쉬운가.

### Casbin

Casbin은 역방향 쿼리가 **어렵다**.

```go
// 불가능: 모든 접근 가능한 객체 조회
// Casbin은 (sub, obj, act) → allow/deny 방향만 지원

// 우회: 모든 정책을 순회
policies := enforcer.GetPolicy()
accessibleDocs := []string{}
for _, p := range policies {
    if p[0] == "alice" && p[2] == "read" {
        accessibleDocs = append(accessibleDocs, p[1])
    }
}
```

**문제**:
- 상속된 권한은 조회 불가
- 정책 수에 비례하는 O(n) 비용
- 암묵적 권한 (`GetImplicitPermissionsForUser`) 계산 필요

### OpenFGA

OpenFGA는 역방향 쿼리를 **기본 지원**한다.

```go
// ListObjects: 접근 가능한 객체 목록
result, _ := client.ListObjects(ctx).Body(openfga.ListObjectsRequest{
    User:     "user:alice",
    Relation: "viewer",
    Type:     "document",
}).Execute()

// 결과: ["document:doc-1", "document:doc-2", ...]
```

```go
// ListUsers: 접근 가능한 사용자 목록
result, _ := client.ListUsers(ctx).Body(openfga.ListUsersRequest{
    Object:   openfga.Object{Type: "document", Id: "doc-1"},
    Relation: "viewer",
}).Execute()

// 결과: ["user:alice", "user:bob", ...]
```

## 성능 특성 비교

| 측면 | Casbin | OpenFGA |
|------|--------|---------|
| **단일 체크 지연** | ~μs (메모리) | ~ms (네트워크 + DB) |
| **처리량** | 높음 (로컬) | 중간 (서비스 호출) |
| **정책 로드** | 시작 시 전체 로드 | 필요 시 조회 |
| **메모리 사용** | 정책 크기에 비례 | 서버 측 관리 |
| **확장 비용** | 개발자 구현 | 수평 확장 지원 |

### Casbin 성능 최적화

```go
// 1. Enforce 캐싱
enforcer.EnableAutoSave(false)  // 자동 저장 비활성화

// 2. 배치 처리
results, _ := enforcer.BatchEnforce([][]interface{}{
    {"alice", "data1", "read"},
    {"bob", "data2", "write"},
})

// 3. 병렬 처리
enforcer.EnableAutoBuildRoleLinks(false)
// 정책 로드 후 수동 빌드
enforcer.BuildRoleLinks()
```

### OpenFGA 성능 최적화

```go
// 1. BatchCheck 사용
result, _ := client.BatchCheck(ctx).Body(openfga.BatchCheckRequest{
    Checks: []openfga.BatchCheckItem{
        {TupleKey: openfga.TupleKey{User: "user:alice", Relation: "viewer", Object: "doc:1"}},
        {TupleKey: openfga.TupleKey{User: "user:alice", Relation: "viewer", Object: "doc:2"}},
    },
}).Execute()

// 2. 캐시 활성화
// 서버 설정: --experimentals enable-check-optimizations

// 3. Contextual Tuples로 실시간 권한 체크
result, _ := client.Check(ctx).Body(openfga.CheckRequest{
    TupleKey: openfga.TupleKey{...},
    ContextualTuples: &openfga.ContextualTupleKeys{
        TupleKeys: []openfga.TupleKey{
            // 임시 권한 (DB에 저장 안 함)
        },
    },
}).Execute()
```

## 언제 무엇을 선택할까?

### Casbin이 적합한 경우

| 상황 | 이유 |
|------|------|
| **단일 애플리케이션** | 네트워크 호출 없이 빠른 권한 체크 |
| **다양한 언어 스택** | 15개 이상 언어 지원 |
| **RBAC/ABAC 중심** | 풍부한 모델 지원과 유연한 Matcher |
| **빠른 MVP** | 외부 인프라 없이 시작 가능 |
| **정적인 권한 구조** | 정책이 자주 변경되지 않음 |
| **내부 도구** | 소규모 사용자, 단순한 권한 |

```go
// Casbin이 잘 맞는 패턴
func middleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        user := r.Context().Value("user").(string)
        path := r.URL.Path
        method := r.Method

        // 빠른 로컬 체크
        if allowed, _ := enforcer.Enforce(user, path, method); !allowed {
            http.Error(w, "Forbidden", 403)
            return
        }
        next.ServeHTTP(w, r)
    })
}
```

### OpenFGA가 적합한 경우

| 상황 | 이유 |
|------|------|
| **마이크로서비스** | 중앙화된 권한 관리 |
| **복잡한 관계** | 그룹/폴더/조직 상속 |
| **대규모 시스템** | 분산 환경에서 일관성 |
| **SaaS 멀티테넌트** | 테넌트별 권한 격리 |
| **ListObjects 필요** | "내 문서 목록" 같은 역방향 쿼리 |
| **권한 감사** | 중앙화된 권한 로그 |

```go
// OpenFGA가 잘 맞는 패턴
func getMyDocuments(ctx context.Context, userID string) ([]Document, error) {
    // 역방향 쿼리: 접근 가능한 문서 목록
    result, _ := fgaClient.ListObjects(ctx).Body(openfga.ListObjectsRequest{
        User:     "user:" + userID,
        Relation: "viewer",
        Type:     "document",
    }).Execute()

    // DB에서 문서 조회
    return documentRepo.FindByIDs(result.Objects)
}
```

## 마이그레이션 전략

### Casbin → OpenFGA

1. **Shadow Mode 운영**
   - 기존 Casbin이 실제 권한 결정
   - OpenFGA를 비동기로 호출해서 결과 비교
   - 불일치 로깅 및 분석

2. **점진적 전환**
   - 신규 기능부터 OpenFGA 적용
   - 기존 기능은 Casbin 유지
   - 검증 완료된 기능부터 전환

3. **데이터 마이그레이션**
   ```python
   # Casbin 정책 → OpenFGA 튜플 변환
   for policy in casbin_policies:
       if policy.is_role_assignment():
           # g, alice, admin → user:alice, member, role:admin
           openfga.write_tuple(...)
       else:
           # p, admin, data1, read → role:admin, viewer, document:data1
           openfga.write_tuple(...)
   ```

### 하이브리드 접근

```
┌─────────────────────────────────────────────────────────────────┐
│                         API Gateway                              │
│                                                                  │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                    Casbin (빠른 체크)                    │   │
│   │   - API 엔드포인트 권한                                  │   │
│   │   - 단순 RBAC                                           │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│                       Application Service                        │
│                                                                  │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                 OpenFGA (복잡한 관계)                    │   │
│   │   - 문서/폴더 권한                                       │   │
│   │   - 그룹 상속                                           │   │
│   │   - ListObjects                                         │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## 정리

| 측면 | Casbin | OpenFGA |
|------|--------|---------|
| **아키텍처** | 임베디드 라이브러리 | 중앙화된 서비스 |
| **모델링** | PERM (선언적 설정) | Zanzibar (관계 튜플) |
| **ReBAC 지원** | 가능하지만 복잡 | 네이티브 지원 |
| **역방향 쿼리** | 어렵다 | ListObjects/ListUsers |
| **확장성** | 개발자 구현 | 내장 지원 |
| **일관성** | 수동 관리 | Consistency 옵션 |
| **언어 지원** | 15개+ | 5개 SDK |
| **시작 복잡도** | 낮음 | 중간 |
| **운영 복잡도** | 분산 시 높음 | 인프라 필요 |

**선택 기준**:
- **단순한 RBAC + 단일 앱** → Casbin
- **복잡한 ReBAC + 마이크로서비스** → OpenFGA
- **빠른 시작 + 점진적 확장** → Casbin → OpenFGA 마이그레이션

### 참고 자료

- [Casbin 공식 문서](https://casbin.org/docs/overview)
- [Casbin ReBAC](https://casbin.org/docs/rebac/)
- [OpenFGA 공식 문서](https://openfga.dev/docs)
- [Casbin vs OpenFGA 비교 (Slashdot)](https://slashdot.org/software/comparison/Casbin-vs-OpenFGA/)
- [Casbin authorization: pros, cons, and alternatives (AuthZed)](https://authzed.com/blog/casbin)
- [OpenFGA Adoption Patterns](https://openfga.dev/docs/best-practices/adoption-patterns)
