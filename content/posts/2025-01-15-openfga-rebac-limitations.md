---
title: "OpenFGA ReBAC 구현의 한계점 심층 분석"
date: 2025-01-15
draft: false
tags: ["OpenFGA", "ReBAC", "Authorization", "Zanzibar"]
summary: "OpenFGA ReBAC 구현의 페이징, 검색 통합, 성능, 일관성 문제와 대규모 데이터셋을 위한 Materialize 패턴, CQRS 아키텍처 등 해결 전략을 정리한다."
---

OpenFGA는 Google Zanzibar에서 영감을 받은 오픈소스 인가(Authorization) 엔진이다. ReBAC(Relationship-Based Access Control)을 구현하기에 좋은 선택지지만, 프로덕션에 적용하기 전에 알아야 할 한계점이 있다.

## 페이징의 근본적 한계

### ListObjects/ListUsers API

OpenFGA의 `ListObjects`와 `ListUsers` API는 실제 페이지네이션을 지원하지 않는다.

```
제한 사항:
- 최대 결과: 1,000개 (기본값)
- 타임아웃: 3초 (기본값)
- 결과 정렬 없음 → 동일 호출이 다른 결과 반환 가능
```

[GitHub Issue #1298](https://github.com/openfga/openfga/issues/1298)에서 페이지네이션 요청이 있었지만, 현재까지 커서 기반 페이지네이션은 미구현 상태다.

REST API에서 "내 문서 목록"을 구현한다고 가정하자. 사용자가 1,000개 이상의 객체에 접근 가능하면 전체 목록을 가져올 수 없다. 타임아웃이 먼저 발생하거나, 최대 결과 제한에 걸린다.

## 검색 + 권한 통합의 딜레마

검색 결과에 권한을 적용하는 건 단순하지 않다. [공식 문서](https://openfga.dev/docs/interacting/search-with-permissions)에서 제시하는 3가지 전략 모두 트레이드오프가 있다.

### 전략 1: Search then Check

DB에서 검색 후, 각 결과에 대해 권한을 체크한다.

```
검색 결과: 1,000개
사용자 접근 가능: 100개
→ 1,000번 체크해서 100개만 표시
```

접근 가능 비율이 낮으면 비효율적이다. 페이지를 채우려면 수천 개를 체크해야 할 수도 있다.

### 전략 2: List Objects then Search

OpenFGA에서 접근 가능한 ID 목록을 가져온 후, 해당 ID로 DB를 검색한다.

```sql
SELECT * FROM documents
WHERE id IN (id1, id2, ..., id10000)
ORDER BY created_at DESC
```

접근 가능 객체가 많으면 그래프 순회 비용이 폭증한다. ID 목록이 커지면 DB 쿼리도 비효율적이다.

### 전략 3: Local Index

Changes API를 소비해서 로컬 인덱스를 구축한다.

인프라 복잡성이 증가한다. 인덱스 동기화, 장애 복구, 스케일링을 고려해야 한다.

### 어떤 전략을 선택할까?

| 시나리오 | 권장 전략 |
|----------|-----------|
| 검색 결과 적음, 접근 비율 높음 | Search then Check |
| 접근 가능 객체 적음 (~1,000) | List Objects then Search |
| 대규모, 복잡한 권한 구조 | Local Index |

실제로는 **Pragmatic Filtering**이 효과적이다. 테넌트 ID 같은 컨텍스트로 먼저 필터링하면 Search then Check도 실용적이 된다.

## 성능 저하 패턴

### Tuple 증가에 따른 성능 저하

[Issue #727](https://github.com/openfga/openfga/issues/727)에서 보고된 실제 벤치마크다.

```
62 tuples   → true: 5ms,   false: 5ms
310K tuples → true: 300ms, false: 2.5~3초
```

`false` check가 특히 느리다. 권한이 없음을 증명하려면 모든 가능한 경로를 탐색해야 하기 때문이다.

### 모델 복잡도의 영향

```
// 비용이 높은 패턴
define viewer: editor and not blocked

// 상대적으로 저렴한 패턴
define viewer: editor or guest
```

`and`, `but not` 연산은 `or`보다 훨씬 비용이 높다. 모델 설계 시 이를 고려해야 한다.

## 일관성(Consistency) 문제

### Eventual Consistency 트레이드오프

캐시를 활성화하면 `Check`, `ListObjects`가 eventually consistent해진다. 캐시 TTL 동안 권한 변경이 반영되지 않는다.

```go
// v1.5.7+에서 HIGHER_CONSISTENCY 옵션 사용 가능
client.Check(ctx, &openfga.CheckRequest{
    // ...
    Consistency: openfga.HIGHER_CONSISTENCY,
})
```

하지만 모든 요청에 `HIGHER_CONSISTENCY`를 사용하면 성능이 크게 저하된다. 런타임에 필요한 경우만 선택적으로 사용해야 한다.

### New Enemy Problem

Google Zanzibar 논문에서 정의한 문제다.

```
1. Alice가 Bob을 폴더에서 제거
2. Charlie가 새 문서를 해당 폴더에 추가
3. ACL 체크가 순서를 무시하면 Bob이 새 문서를 볼 수 있음
```

Zanzibar는 Zookies(일관성 토큰)로 해결한다. OpenFGA는 아직 Zookies를 지원하지 않는다. 로드맵에는 있다.

## Write 작업 제한

| 제한 | 값 | 비고 |
|------|-----|------|
| 요청당 최대 tuple | 100개 | writes + deletes 합산 |
| 트랜잭션 | 전체 성공/실패 | 부분 성공 불가 |
| 대량 배치 (내부) | 2,000개씩 분할 | 각 배치는 별도 트랜잭션 |

v1.10.0부터 `on_duplicate: ignore`, `on_missing: ignore` 옵션이 추가되어 중복/누락 처리가 편해졌다.

```json
{
  "writes": {
    "tuple_keys": [...],
    "on_duplicate": "ignore"
  }
}
```

## 모델링 한계

### 복잡도 폭발

개별 리소스마다 세밀한 권한을 설정하면, 모든 객체를 OpenFGA에 복제해야 한다.

```
원본 DB: document 생성
OpenFGA: tuple 생성
→ 2개 DB에 트랜잭션으로 저장해야 함
```

### 권장 패턴

[마이그레이션 경험담](https://jguer.space/blog/migrating-legacy-access-control-to-openfga)에서 강조하는 원칙이다.

> "최고의 tuple은 작성하지 않아도 되는 tuple이다"

가능하면 프로젝트/폴더 레벨에서 권한을 결정하고, 개별 리소스 권한은 상속으로 해결한다.

## SpiceDB와 비교

| 측면 | OpenFGA | SpiceDB |
|------|---------|---------|
| 일관성 | HIGHER_CONSISTENCY (v1.5.7+) | ZedTokens (완전한 Zanzibar 일관성) |
| 기능 개발 | CNCF Incubating | Caveats 등 선도적 기능 |
| 우선순위 | 개발자 경험 | 스케일 |
| 운영 복잡도 | 중간 | 높음 |

SpiceDB는 일관성 모델을 완전히 구현했다. 대신 운영 복잡도가 높다. 요구사항에 따라 선택하면 된다.

## 실무 적용 가이드

### 적합한 경우

- 중소 규모 (수천~수만 객체)
- 권한 구조가 비교적 단순
- 검색 결과가 작거나 접근 비율이 높음

### 신중해야 하는 경우

- 대규모 데이터셋 + 복잡한 검색/페이징 요구
- 실시간 권한 변경 반영 필수
- 깊은 계층 구조 + 복잡한 조건부 권한

### 마이그레이션 팁

1. **Shadow Mode 운영**: 기존 시스템과 병렬로 운영하며 결과 비교
2. **점진적 적용**: 폴더, 프로젝트 등 대표 리소스부터 시작
3. **멀티테넌시**: 테넌트별 Store 분리로 noisy neighbor 방지

## 대규모 데이터셋 전략

앞서 언급한 한계에도 불구하고, 대규모 환경에서 페이징이 필요한 경우 적용할 수 있는 전략이 있다.

### 전략 선택 가이드

| 규모 | 접근 비율 | 권장 전략 |
|------|----------|----------|
| 소규모 (<1K) | 높음 | Search then Check |
| 소규모 | 낮음 | ListObjects then Search |
| 중규모 (1K-100K) | - | CheckBulkPermissions + 커서 |
| 대규모 (100K+) | - | Materialize 패턴 |
| 대규모 + 검색 필수 | - | 검색 인덱스 + ACL 동기화 |

### 1. Materialize 패턴

가장 확장성이 높은 방식이다. Google Zanzibar 논문의 [Leopard 인덱싱 시스템](https://authzed.com/zanzibar)이 이 개념이다.

권한을 미리 계산해서 비정규화된 인덱스로 저장한다. 검색 시 간단한 JOIN으로 권한 필터링이 가능하다.

**AuthZed Materialize**

SpiceDB를 사용한다면 [AuthZed Materialize](https://authzed.com/docs/authzed/concepts/authzed-materialize)가 이 방식을 구현한다. LookupResources, LookupSubjects 성능을 대폭 개선한다.

**Flowtide (OpenFGA 전용)**

OpenFGA를 사용한다면 [Flowtide](https://koralium.github.io/flowtide/docs/connectors/openfga)가 있다. OpenFGA 권한을 CQRS Read Model로 Materialize한다.

```mermaid
flowchart LR
    OpenFGA[OpenFGA] -->|Watch API| Flowtide[Flowtide]
    Flowtide --> QS["Query Service / ES<br/>+ Materialized ACL"]

    style OpenFGA fill:#e3f2fd
    style Flowtide fill:#fff3e0
    style QS fill:#e8f5e9
```

CQRS 아키텍처에서 Query Service가 Materialized 권한을 받아 검색 쿼리에 JOIN할 수 있다.

### 2. CheckBulkPermissions + 커서 기반 페이지네이션

[SpiceDB 문서](https://authzed.com/docs/spicedb/modeling/protecting-a-list-endpoint)에서 권장하는 중간 규모 전략이다.

```
1. DB에서 커서 기반으로 N개 조회
2. CheckBulkPermissions로 일괄 권한 체크
3. 허용된 결과만 필터링
4. 페이지가 채워질 때까지 반복
5. 다음 커서 반환
```

핵심은 **같은 revision에서 모든 체크를 실행**하는 것이다. 일관성을 보장한다.

```go
// 개념적 예시
func listDocuments(ctx context.Context, userID string, cursor string, pageSize int) (*Page, error) {
    var results []Document
    currentCursor := cursor

    for len(results) < pageSize {
        // 1. DB에서 후보 조회
        candidates, nextCursor := db.Query(currentCursor, pageSize*2)
        if len(candidates) == 0 {
            break
        }

        // 2. 일괄 권한 체크
        checks := buildCheckRequests(candidates, userID)
        permissions := openfga.BatchCheck(ctx, checks)

        // 3. 허용된 것만 추가
        for i, doc := range candidates {
            if permissions[i].Allowed {
                results = append(results, doc)
            }
        }
        currentCursor = nextCursor
    }

    return &Page{Items: results[:pageSize], NextCursor: currentCursor}, nil
}
```

### 3. 검색 인덱스에 ACL 비정규화

Elasticsearch나 OpenSearch를 사용한다면, 문서마다 ACL 필드를 저장한다.

```json
{
  "doc_id": "doc-123",
  "title": "Quarterly Report",
  "content": "...",
  "acl_users": ["user:alice", "user:bob"],
  "acl_groups": ["group:engineering"]
}
```

검색 시 필터로 적용한다.

```json
{
  "query": {
    "bool": {
      "must": { "match": { "content": "report" } },
      "filter": {
        "bool": {
          "should": [
            { "term": { "acl_users": "user:alice" } },
            { "term": { "acl_groups": "group:engineering" } }
          ]
        }
      }
    }
  }
}
```

**단점**: 권한 변경 시 관련된 모든 문서를 업데이트해야 한다. 그룹 멤버십 변경이 수천 개 문서에 영향을 줄 수 있다.

### 4. Local Index from Watch API

OpenFGA의 Watch API로 권한 변경을 스트리밍 수신해서 로컬 인덱스를 구축한다.

```python
# 개념적 예시
async def sync_permissions():
    async for change in openfga.watch(store_id):
        if change.operation == "WRITE":
            await search_index.update_acl(
                object_id=change.tuple.object,
                user_id=change.tuple.user,
                relation=change.tuple.relation
            )
        elif change.operation == "DELETE":
            await search_index.remove_acl(...)
```

인프라 복잡성이 증가하지만, OpenFGA 공식 지원 방식이다.

### 5. Pragmatic Filtering

가장 실용적인 전략이다. **OpenFGA 호출 전에** 테넌트 ID, 조직 ID 등 이미 알고 있는 컨텍스트로 먼저 DB에서 필터링한다.

```sql
-- 핵심: OpenFGA 호출 전에 DB에서 대폭 필터링
SELECT * FROM documents
WHERE tenant_id = :user_tenant_id  -- 100만 건 → 1,000건으로 축소
  AND created_at > :date_filter
ORDER BY created_at DESC
LIMIT 100
-- 이후 100건에 대해서만 BatchCheck 실행
```

전체 데이터에 권한 체크하는 대신, 줄어든 후보에만 체크한다. 많은 실무 사례에서 이 조합이 충분하다.

### CQRS + 권한 Projection 아키텍처

이벤트 소싱을 사용한다면 권한을 별도 Read Model로 Projection할 수 있다.

```mermaid
flowchart LR
    subgraph Command ["Command Side"]
        API[API] --> Domain[Domain]
        Domain --> ES[(Event Store)]
    end

    ES --> Proj

    subgraph Query ["Query Side"]
        subgraph Proj ["Projections"]
            DRM["Domain<br/>Read Model"]
            PRM["Permission<br/>Read Model"]
        end
        Proj --> QS["Query Service<br/>(JOIN)"]
    end

    style Command fill:#e3f2fd
    style Query fill:#e8f5e9
    style ES fill:#fff3e0
```

`UserGrantedPermission`, `RoleAssigned` 같은 이벤트를 Permission Read Model로 Projection하면 검색 쿼리에서 직접 JOIN이 가능하다. OpenFGA 호출 없이 권한 기반 필터링을 할 수 있다.

## 정리

OpenFGA는 ReBAC 구현을 위한 좋은 선택지다. 하지만 페이징, 검색 통합, 대규모 성능, 일관성 측면에서 한계가 있다. 이 한계를 이해하고 아키텍처를 설계해야 프로덕션에서 문제를 피할 수 있다.

### 참고 자료

- [OpenFGA 공식 문서 - Search with Permissions](https://openfga.dev/docs/interacting/search-with-permissions)
- [OpenFGA 공식 문서 - Running in Production](https://openfga.dev/docs/best-practices/running-in-production)
- [Query Consistency Options Announcement](https://openfga.dev/blog/query-consistency-options-announcement)
- [GitHub Issue #727 - Performance Penalty](https://github.com/openfga/openfga/issues/727)
- [Migrating Legacy Access Control to OpenFGA](https://jguer.space/blog/migrating-legacy-access-control-to-openfga)
- [Google Zanzibar Paper (Annotated)](https://authzed.com/zanzibar)
- [AuthZed Materialize](https://authzed.com/docs/authzed/concepts/authzed-materialize)
- [SpiceDB - Protecting a List Endpoint](https://authzed.com/docs/spicedb/modeling/protecting-a-list-endpoint)
- [Flowtide OpenFGA Connector](https://koralium.github.io/flowtide/docs/connectors/openfga)
- [How ReBAC Helps Solve Data Filtering](https://www.aserto.com/blog/how-rebac-helps-solve-data-filtering)
