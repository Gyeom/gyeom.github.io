---
title: "OpenFGA와 ReBAC로 구현하는 관계 기반 권한 제어"
date: 2024-09-10
draft: false
tags: ["OpenFGA", "ReBAC", "권한관리", "RBAC", "Spring Boot", "보안"]
categories: ["백엔드"]
summary: "기존 RBAC의 한계를 넘어 관계 기반 접근 제어(ReBAC)를 구현하는 OpenFGA의 개념과 아키텍처, Spring Boot 연동 방법을 정리했다."
---

Google Drive는 어떻게 폴더 소유자가 그 안의 모든 파일에 자동으로 접근 권한을 가지도록 구현할까? GitHub는 조직 멤버가 자동으로 팀 저장소에 접근할 수 있는 권한을 어떻게 부여할까? 이런 복잡한 권한 관계를 구현하는 방법이 ReBAC(Relationship-Based Access Control)이고, OpenFGA는 이를 쉽게 구현할 수 있는 오픈소스 솔루션이다.

## ReBAC와 RBAC의 차이

### RBAC의 한계

전통적인 RBAC(Role-Based Access Control)은 사용자에게 역할을 부여하고, 역할에 권한을 매핑하는 방식이다.

```
사용자 → 역할(Admin, Editor, Viewer) → 권한(Read, Write, Delete)
```

이 방식은 간단하지만 복잡한 관계를 표현하기 어렵다.

**문제 상황**
- 폴더 소유자는 하위 파일에도 자동으로 접근 권한이 있어야 한다
- 문서를 공유받은 사용자는 편집 권한도 함께 받을 수 있다
- 조직의 관리자는 모든 팀 리소스에 자동으로 접근할 수 있어야 한다

RBAC로는 이런 계층적이고 동적인 권한 관계를 표현하려면 복잡한 규칙을 코드로 구현해야 한다.

### ReBAC의 접근 방식

ReBAC는 **객체 간의 관계**를 기반으로 권한을 결정한다.

```
alice는 folder:docs의 owner다
folder:docs는 file:report.pdf의 parent다
→ alice는 file:report.pdf를 read할 수 있다
```

관계를 선언하면 권한이 자동으로 추론된다. 코드에 복잡한 로직을 작성할 필요가 없다.

## OpenFGA 소개

OpenFGA는 Auth0의 Zanzibar 논문을 기반으로 만든 오픈소스 권한 관리 시스템이다.

### 주요 특징

| 특징 | 설명 |
|------|------|
| **스키마 기반** | DSL로 권한 모델 정의 |
| **관계 튜플** | 주체-관계-객체 형태로 권한 저장 |
| **그래프 탐색** | 관계 그래프를 탐색해 권한 체크 |
| **고성능** | 대규모 서비스에서 검증됨 (Google, Airbnb) |
| **Cloud/Self-hosted** | SaaS 또는 직접 배포 가능 |

### 아키텍처

```mermaid
flowchart TB
    App[애플리케이션] --> API[OpenFGA API]

    subgraph OpenFGA
        API --> Model["Authorization Model<br/>(DSL)"]
        Model --> Tuples["Relationship Tuples<br/>(Storage)"]
    end

    style App fill:#e3f2fd
    style OpenFGA fill:#fff3e0
```

1. **Authorization Model**: 권한 규칙 정의 (DSL)
2. **Relationship Tuples**: 실제 관계 데이터 저장
3. **Check API**: 권한 확인 요청 처리

## 스키마 정의 방법

OpenFGA는 DSL로 권한 모델을 정의한다.

### Google Drive 예시

```fga
model
  schema 1.1

type user

type folder
  relations
    define owner: [user]
    define editor: [user] or owner
    define viewer: [user] or editor
    define parent: [folder]
    define can_read: viewer or can_read from parent
    define can_write: editor or can_write from parent
    define can_delete: owner or can_delete from parent

type file
  relations
    define owner: [user]
    define editor: [user] or owner
    define viewer: [user] or editor
    define parent: [folder]
    define can_read: viewer or can_read from parent
    define can_write: editor or can_write from parent
    define can_delete: owner or can_delete from parent
```

### 스키마 해석

**folder 타입**
- `owner`: 폴더 소유자
- `editor`: 편집자 또는 소유자
- `viewer`: 열람자 또는 편집자
- `parent`: 상위 폴더 관계
- `can_read`: 열람자이거나, 부모 폴더에서 읽기 권한 상속
- `can_write`: 편집자이거나, 부모 폴더에서 쓰기 권한 상속
- `can_delete`: 소유자이거나, 부모 폴더에서 삭제 권한 상속

`from parent` 구문이 계층 구조를 자동으로 처리한다.

## 관계 튜플과 권한 체크

### 관계 튜플 생성

관계 튜플은 `user:alice, owner, folder:docs` 형태로 저장된다.

```json
{
  "writes": [
    {
      "user": "user:alice",
      "relation": "owner",
      "object": "folder:docs"
    },
    {
      "user": "folder:docs",
      "relation": "parent",
      "object": "file:report.pdf"
    }
  ]
}
```

### 권한 체크

```json
{
  "user": "user:alice",
  "relation": "can_read",
  "object": "file:report.pdf"
}
```

OpenFGA는 다음 과정을 거친다.

1. `alice`가 `file:report.pdf`의 `viewer`인가? → 아니오
2. `file:report.pdf`의 `parent`는? → `folder:docs`
3. `alice`가 `folder:docs`의 `can_read` 권한이 있는가? → 예 (owner이므로)
4. **결과: 허용**

## Spring Boot 연동

### 의존성 추가

```gradle
dependencies {
    implementation 'dev.openfga:openfga-sdk:0.3.0'
}
```

### OpenFGA 클라이언트 설정

```java
@Configuration
public class OpenFgaConfig {

    @Bean
    public OpenFgaClient openFgaClient() throws FgaInvalidParameterException {
        var configuration = new ClientConfiguration()
            .apiUrl("http://localhost:8080")
            .storeId("01HXYZ...")
            .authorizationModelId("01ABCD...");

        return new OpenFgaClient(configuration);
    }
}
```

### 권한 체크 서비스

```java
@Service
@RequiredArgsConstructor
public class AuthorizationService {

    private final OpenFgaClient fgaClient;

    public boolean canAccess(String userId, String relation, String objectId)
            throws FgaInvalidParameterException {
        var body = new ClientCheckRequest()
            .user("user:" + userId)
            .relation(relation)
            ._object(objectId);

        try {
            var response = fgaClient.check(body).get();
            return response.getAllowed();
        } catch (Exception e) {
            throw new RuntimeException("Authorization check failed", e);
        }
    }
}
```

### 컨트롤러에서 사용

```java
@RestController
@RequiredArgsConstructor
public class DocumentController {

    private final AuthorizationService authService;
    private final DocumentService documentService;

    @GetMapping("/documents/{id}")
    public ResponseEntity<Document> getDocument(
            @PathVariable String id,
            @AuthenticationPrincipal String userId) {

        if (!authService.canAccess(userId, "can_read", "file:" + id)) {
            return ResponseEntity.status(HttpStatus.FORBIDDEN).build();
        }

        return ResponseEntity.ok(documentService.findById(id));
    }
}
```

### 관계 생성

```java
@Service
@RequiredArgsConstructor
public class DocumentService {

    private final OpenFgaClient fgaClient;

    public void createDocument(String userId, String folderId, Document doc) {
        // 문서 저장
        documentRepository.save(doc);

        // 소유 관계 생성
        var tuple = new ClientTupleKey()
            .user("user:" + userId)
            .relation("owner")
            ._object("file:" + doc.getId());

        fgaClient.write(new ClientWriteRequest().writes(List.of(tuple))).get();

        // 부모 폴더 관계 생성
        var parentTuple = new ClientTupleKey()
            .user("folder:" + folderId)
            .relation("parent")
            ._object("file:" + doc.getId());

        fgaClient.write(new ClientWriteRequest().writes(List.of(parentTuple))).get();
    }
}
```

## 실제 사용 사례

### Google Drive

```
폴더 소유자가 하위 파일 자동 접근
공유 링크로 일시적 권한 부여
조직 드라이브의 계층적 권한
```

### GitHub

```
조직 멤버가 팀 저장소 자동 접근
저장소 관리자가 이슈/PR 관리
팀 계층 구조 반영
```

### Airbnb

```
호스트가 자신의 숙소 관리
게스트가 예약한 숙소 정보 접근
지역 관리자가 해당 지역 숙소 관리
```

## 결과

ReBAC는 복잡한 권한 관계를 선언적으로 정의할 수 있다. OpenFGA는 이를 고성능으로 구현한 오픈소스 솔루션이다.

**ReBAC가 적합한 경우**
- 계층 구조가 있는 리소스 (폴더/파일)
- 조직/팀 단위 권한 관리
- 동적으로 변하는 권한 관계

**RBAC가 적합한 경우**
- 단순한 역할 기반 권한
- 정적인 권한 구조
- 작은 규모의 서비스

복잡한 권한 요구사항이 있다면 OpenFGA를 검토해볼 가치가 있다.
