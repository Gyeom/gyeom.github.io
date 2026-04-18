---
title: "Casbin K8s 멀티 인스턴스 정책 동기화 전략"
date: 2025-05-20
draft: true
tags: ["Casbin", "Kubernetes", "Spring-Boot", "Authorization", "Redis"]
categories: ["Architecture"]
summary: "K8s 환경에서 Casbin 인스턴스 간 정책 동기화 전략(Redis/PostgreSQL Watcher)과 대규모 리소스 목록 페이징 전략을 정리한다."
---

## 문제 상황

Casbin은 **임베디드 라이브러리**다. 정책을 앱 시작 시 메모리에 로드하고, 이후 메모리에서 권한을 체크한다.

```
Pod A: [App + Casbin] ─── 메모리에 정책 로드
Pod B: [App + Casbin] ─── 메모리에 정책 로드 (별도 복사본)
Pod C: [App + Casbin] ─── 메모리에 정책 로드 (별도 복사본)
```

**문제**: Pod A에서 정책을 변경해도 Pod B, C는 모른다.

```
1. Admin이 Pod A에 "user:alice에게 admin 권한 부여" 요청
2. Pod A의 Casbin이 정책 추가 → Pod A 메모리에만 반영
3. Alice가 Pod B로 라우팅됨 → 권한 없음!
```

---

## 전략 비교

| 전략 | 실시간성 | 복잡도 | 적합한 상황 |
|------|----------|--------|-------------|
| Watcher (Redis) | ✅ 즉시 | 중간 | 정책 변경이 잦은 경우 |
| Watcher (PostgreSQL) | ✅ 즉시 | 중간 | 이미 PostgreSQL 사용 중일 때 |
| ConfigMap + Rolling | ❌ 재배포 필요 | 낮음 | 정책 변경이 드문 경우 |
| 중앙화 서비스 | ✅ 즉시 | 높음 | 대규모 멀티 서비스 |

---

## 전략 1: Redis Watcher (권장)

가장 일반적인 방법. Redis Pub/Sub으로 정책 변경을 브로드캐스트한다.

```
┌─────────────────────────────────────────────────────┐
│                    Redis Pub/Sub                     │
│                  (casbin-policy-topic)               │
└──────────┬──────────────┬──────────────┬────────────┘
           │              │              │
      subscribe      subscribe      subscribe
           │              │              │
    ┌──────▼──────┐ ┌─────▼──────┐ ┌─────▼──────┐
    │   Pod A     │ │   Pod B    │ │   Pod C    │
    │  [Casbin]   │ │  [Casbin]  │ │  [Casbin]  │
    │  [Watcher]  │ │  [Watcher] │ │  [Watcher] │
    └─────────────┘ └────────────┘ └────────────┘
```

### 동작 원리

1. 각 Pod의 Watcher가 Redis 토픽을 구독
2. Pod A에서 정책 변경 시 → Redis에 메시지 발행
3. Pod B, C가 메시지 수신 → `LoadPolicy()` 실행
4. 모든 Pod의 정책이 동기화됨

### Spring Boot 구현

```gradle
dependencies {
    implementation 'org.casbin:jcasbin:1.55.0'
    implementation 'org.casbin:jdbc-adapter:2.7.0'
    implementation 'org.casbin:jcasbin-redis-watcher:1.4.1'
}
```

```java
@Configuration
public class CasbinConfig {

    @Bean
    public Enforcer enforcer(DataSource dataSource, RedisWatcher watcher) {
        // 1. DB Adapter: 정책 저장소
        JDBCAdapter adapter = new JDBCAdapter(dataSource);

        // 2. Enforcer 생성
        Enforcer enforcer = new Enforcer("model.conf", adapter);

        // 3. Watcher 연결: 변경 알림
        enforcer.setWatcher(watcher);

        // 4. 변경 수신 시 정책 리로드
        watcher.setUpdateCallback(() -> {
            enforcer.loadPolicy();
            log.info("Policy reloaded from watcher notification");
        });

        return enforcer;
    }

    @Bean
    public RedisWatcher redisWatcher(
            @Value("${redis.host}") String host,
            @Value("${redis.port}") int port) {
        return new RedisWatcher(host, port);
    }
}
```

```java
@Service
@RequiredArgsConstructor
public class PolicyService {

    private final Enforcer enforcer;

    public void addPolicy(String sub, String obj, String act) {
        // 정책 추가 → DB 저장 + Redis 브로드캐스트 자동 실행
        enforcer.addPolicy(sub, obj, act);
        // 다른 Pod들이 자동으로 리로드
    }

    public boolean check(String sub, String obj, String act) {
        return enforcer.enforce(sub, obj, act);
    }
}
```

### K8s 배포

```yaml
# redis.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis
spec:
  replicas: 1
  template:
    spec:
      containers:
      - name: redis
        image: redis:7-alpine
        ports:
        - containerPort: 6379
---
apiVersion: v1
kind: Service
metadata:
  name: redis
spec:
  ports:
  - port: 6379
  selector:
    app: redis
```

```yaml
# application.yaml (Spring)
redis:
  host: redis.default.svc.cluster.local
  port: 6379
```

---

## 전략 2: PostgreSQL Watcher

이미 PostgreSQL을 사용 중이라면 별도 Redis 없이 가능하다.

```gradle
implementation 'org.casbin:jcasbin-postgres-watcher:1.0.0'
```

```java
@Bean
public PostgresWatcher postgresWatcher(DataSource dataSource) {
    return new PostgresWatcher(dataSource);
}
```

PostgreSQL의 `LISTEN/NOTIFY` 기능을 사용해 동기화한다.

---

## 전략 3: ConfigMap + Rolling Update

정책 변경이 드물고, 재배포가 허용되는 경우 가장 단순한 방법.

```yaml
# configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: casbin-policy
data:
  policy.csv: |
    p, admin, /api/*, *
    p, user, /api/public/*, GET
    g, alice, admin
```

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  annotations:
    # ConfigMap 변경 시 자동 재배포 트리거
    checksum/policy: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
spec:
  template:
    spec:
      containers:
      - name: app
        volumeMounts:
        - name: policy
          mountPath: /app/policy
      volumes:
      - name: policy
        configMap:
          name: casbin-policy
```

```java
@Bean
public Enforcer enforcer() {
    return new Enforcer("model.conf", "/app/policy/policy.csv");
}
```

**장점**: 인프라 추가 없음
**단점**: 정책 변경 시 모든 Pod 재시작 필요

---

## 전략 4: 중앙화 서비스 (대규모용)

마이크로서비스가 많다면 Casbin을 별도 서비스로 분리한다.

```
┌─────────────────────────────────────────────┐
│            Casbin Auth Service              │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐     │
│  │ Pod A   │  │ Pod B   │  │ Pod C   │     │
│  │[Casbin] │  │[Casbin] │  │[Casbin] │     │
│  └────┬────┘  └────┬────┘  └────┬────┘     │
│       └───────┬────┴───────────┘           │
│               ▼                             │
│        [Shared Redis Watcher]               │
└─────────────────────────────────────────────┘
          ▲           ▲           ▲
          │ gRPC      │ gRPC      │ gRPC
    ┌─────┴──┐  ┌─────┴──┐  ┌─────┴──┐
    │Service │  │Service │  │Service │
    │   A    │  │   B    │  │   C    │
    └────────┘  └────────┘  └────────┘
```

이 정도 규모라면 [OpenFGA](https://openfga.dev)나 [SpiceDB](https://authzed.com/spicedb) 같은 전용 서비스를 고려한다.

---

## 실전 팁

### 1. Adapter + Watcher 조합이 핵심

```java
// ❌ 파일 기반 + Watcher만
new Enforcer("model.conf", "policy.csv");  // 리로드해도 파일이 안 바뀜

// ✅ DB Adapter + Watcher
JDBCAdapter adapter = new JDBCAdapter(dataSource);
Enforcer enforcer = new Enforcer("model.conf", adapter);
enforcer.setWatcher(watcher);  // 알림 받으면 DB에서 리로드
```

### 2. AutoSave 활성화

```java
enforcer.enableAutoSave(true);  // 정책 변경 시 자동으로 DB 저장
```

### 3. 헬스체크에 Watcher 상태 포함

```java
@Component
public class CasbinHealthIndicator implements HealthIndicator {

    private final RedisWatcher watcher;

    @Override
    public Health health() {
        if (watcher.isConnected()) {
            return Health.up().build();
        }
        return Health.down().withDetail("reason", "Redis watcher disconnected").build();
    }
}
```

### 4. 정책 변경 감사 로그

```java
watcher.setUpdateCallback(() -> {
    log.info("Policy update received at {}", Instant.now());
    enforcer.loadPolicy();
    auditService.record("POLICY_RELOAD", getCurrentPodName());
});
```

---

## 페이징과 리소스 목록 조회

멀티 인스턴스 동기화와 별개로, Casbin의 또 다른 고려사항이 있다. **"사용자가 접근 가능한 리소스 목록"**을 어떻게 조회할 것인가?

### OpenFGA와의 차이

| 측면 | OpenFGA | Casbin |
|------|---------|--------|
| **리소스 목록 API** | `ListObjects` (한계 있음) | ❌ 없음 |
| **최대 결과** | 1,000개, 3초 타임아웃 | 제한 없음 (메모리) |
| **권한 체크 방식** | API 호출 | 메모리 접근 |

OpenFGA는 `ListObjects` API가 있지만 1,000개 제한과 페이지네이션 미지원 문제가 있다. Casbin은 **그런 API 자체가 없다**.

```java
// OpenFGA: 한 번에 목록 요청 (1,000개 제한)
client.listObjects(user, "viewer", "document");

// Casbin: 리소스마다 개별 체크
for (Document doc : allDocuments) {
    if (enforcer.enforce(user, doc.getId(), "read")) {
        results.add(doc);
    }
}
```

### Casbin의 장점

임베디드라서 **배치 체크가 빠르다**.

```java
// 1,000개 체크해도 수십ms (메모리 접근)
List<Document> filtered = documents.stream()
    .filter(doc -> enforcer.enforce("alice", doc.getId(), "read"))
    .toList();
```

OpenFGA는 같은 작업에 네트워크 호출이 필요하다.

### 대규모 페이징 전략

100만 개 문서 중 접근 가능한 것만 10개씩 페이징해야 한다면?

**방법 1: DB 페이징 + 권한 필터 (소규모)**

```java
public Page<Document> getMyDocuments(String userId, Pageable pageable) {
    // 1. DB에서 페이지 조회
    Page<Document> page = documentRepo.findAll(pageable);

    // 2. 권한 필터링 (메모리에서 빠르게)
    List<Document> filtered = page.stream()
        .filter(doc -> enforcer.enforce(userId, doc.getId(), "read"))
        .toList();

    // 문제: 10개 요청했는데 3개만 통과하면?
    return new PageImpl<>(filtered, pageable, page.getTotalElements());
}
```

**방법 2: Filtered Adapter (테넌트별 정책)**

```java
// 테넌트별로 정책을 분리 로드
Filter filter = new Filter();
filter.setP(new String[]{"", "tenant_123"});
adapter.loadFilteredPolicy(enforcer.getModel(), filter);
```

정책 자체를 테넌트/조직 단위로 분리하면 메모리 사용량과 검색 범위를 줄일 수 있다.

**방법 3: 권한 비정규화 (대규모)**

OpenFGA 포스트에서 다룬 것과 같은 전략이 필요하다.

```
┌──────────────┐                  ┌──────────────────────┐
│   Casbin     │  Watcher 이벤트  │  Search Index / DB   │
│  (정책 관리) │ ───────────────▶ │  + ACL 비정규화      │
└──────────────┘                  └──────────────────────┘
```

```sql
-- 문서 테이블에 ACL 컬럼 추가
SELECT * FROM documents
WHERE tenant_id = :tenantId
  AND (owner_id = :userId OR :userId = ANY(acl_users))
ORDER BY created_at DESC
LIMIT 10 OFFSET 0
```

### 전략 선택 가이드

| 규모 | 접근 비율 | 권장 전략 |
|------|----------|----------|
| 소규모 (~1K) | 높음 | DB 페이징 + `enforce()` 필터 |
| 소규모 | 낮음 | `getImplicitPermissionsForUser()` 후 IN 쿼리 |
| 중규모 (~100K) | - | Filtered Adapter + 테넌트 분리 |
| 대규모 (100K+) | - | ACL 비정규화 또는 검색 인덱스 동기화 |

Casbin은 **권한 체크는 빠르지만**, 대규모 리소스 목록 조회는 추가 설계가 필요하다. 이 점은 OpenFGA나 다른 인가 시스템과 동일하다.

---

## 정리

| 상황 | 권장 전략 |
|------|-----------|
| 일반적인 K8s 배포 | Redis Watcher |
| PostgreSQL만 사용 | PostgreSQL Watcher |
| 정책 변경 거의 없음 | ConfigMap + Rolling |
| 10개+ 마이크로서비스 | 중앙화 서비스 또는 OpenFGA |

Casbin은 임베디드 라이브러리지만, Watcher를 붙이면 분산 환경에서도 충분히 사용할 수 있다. Redis Pub/Sub의 지연 시간은 밀리초 단위라서 실시간 동기화와 다름없다.

---

## 참고 자료

- [Casbin Watchers 공식 문서](https://casbin.org/docs/watchers/)
- [jcasbin-redis-watcher GitHub](https://github.com/jcasbin/redis-watcher)
- [jcasbin-postgres-watcher GitHub](https://github.com/jcasbin/postgres-watcher)
- [Casbin Performance Optimization](https://casbin.org/docs/performance/)
