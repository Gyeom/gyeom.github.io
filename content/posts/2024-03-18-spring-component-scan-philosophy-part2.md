---
title: "Selectively Opinionated Spring Boot (2) - 멀티앱, 하나의 코드베이스"
date: 2024-03-18
draft: false
tags: ["Spring", "Spring Boot", "Multi-Module", "Configuration", "profiles.include"]
categories: ["Spring"]
summary: "Spring Boot의 profiles.include로 설정을 조합하고, @Import로 앱별 어댑터를 선택하는 멀티앱 구성 전략"
series: ["Selectively Opinionated Spring Boot"]
series_order: 2
---

## 시리즈

1. [@ComponentScan의 함정](/dev-notes/posts/2024-03-15-spring-component-scan-philosophy-part1/)
2. **멀티앱, 하나의 코드베이스** (현재 글)
3. [Mock 남용 없는 통합 테스트](/dev-notes/posts/2024-03-22-spring-component-scan-philosophy-part3/)

---

## 들어가며

API 서버, Kafka 컨슈머, 배치 스케줄러. 세 가지 앱이 같은 도메인 로직을 사용하지만 진입점이 다르다.

Spring Boot의 opinionated한 접근은 "하나의 앱"을 가정한다. `@SpringBootApplication`은 패키지 전체를 스캔하고, `application.yml` 하나로 설정을 관리한다. 하지만 멀티앱 환경에서는 이 Convention이 오히려 걸림돌이 된다. API 서버에 Kafka 리스너가 붙어 있을 이유가 없고, 배치 앱에 HTTP 컨트롤러가 있을 이유도 없다.

여기서 Spring Boot의 다른 Convention이 빛을 발한다. `profiles.include`로 설정 파일을 조합하고, `@Import`로 필요한 Config만 선택한다. Hexagonal Architecture의 "어댑터는 교체 가능해야 한다"는 원칙을 앱 구성에 적용한 결과다.

---

## Spring Boot가 제공하는 조합 도구

Part 1에서 `@EnableAutoConfiguration` + `@Import`로 빈 등록을 명시적으로 관리했다. 멀티앱 환경에서는 설정 파일도 같은 원칙으로 관리해야 한다.

Spring Boot는 [Externalized Configuration](https://docs.spring.io/spring-boot/reference/features/external-config.html) 기능으로 설정을 계층화한다. 그중 [`profiles.include`](https://docs.spring.io/spring-boot/reference/features/profiles.html#features.profiles.including)는 여러 설정 파일을 조합하는 Convention이다.

```yaml
spring:
  profiles:
    include: datasource, kafka, redis
```

이 한 줄로 `application-datasource.yml`, `application-kafka.yml`, `application-redis.yml`이 로드된다. 각 앱은 필요한 설정만 include한다.

| 앱 | 필요한 설정 | 불필요한 설정 |
|-----|----------|------------|
| API | DB, Kafka Producer, Redis | Kafka Consumer |
| Consumer | DB, Kafka Consumer | Redis, HTTP |
| Outbox | DB, Kafka Producer | Redis, HTTP, Consumer |

`@Import`가 빈 등록을 조합하듯, `profiles.include`가 설정 파일을 조합한다. Spring Boot의 Convention을 따르되, "모든 것을 자동으로" 대신 "필요한 것만 명시적으로" 구성하는 원칙이다.

---

## 멀티앱 아키텍처

하나의 코드베이스에서 여러 애플리케이션을 실행한다.

```
user-platform/
├── up-core-api-app/        # API 서버 (HTTP 요청 처리)
├── up-core-consumer-app/   # Kafka 컨슈머 (메시지 수신)
├── up-core-outbox-app/     # 아웃박스 처리 (이벤트 발행)
├── up-adapter/
│   ├── inbound/
│   │   ├── web/            # HTTP 어댑터
│   │   ├── event-consumer/ # Kafka 리스너
│   │   └── outbox-scheduler/  # 스케줄러
│   └── outbound/
│       ├── persistence/    # DB 접근
│       ├── client/         # 외부 API
│       ├── producer/       # Kafka 발행
│       ├── cache/          # Redis
│       └── slack/          # Slack 알림
├── up-application/         # UseCase 구현
└── up-domain/              # 도메인 모델
```

같은 도메인 로직을 공유하지만, 진입점이 다르다.

---

## 앱별 Import 구성

### API App

```kotlin
@EnableAutoConfiguration
@Import(
    WebAdapterConfig::class,           // HTTP 컨트롤러
    PersistenceAdapterConfig::class,   // DB
    ClientAdapterConfig::class,        // 외부 API (Feign)
    CacheAdapterConfig::class,         // Redis
    ProducerAdapterConfig::class,      // Kafka 발행
    UseCaseConfig::class,              // 비즈니스 로직
    KafkaProducerConfig::class,        // Kafka 설정
    ApplicationSupportConfig::class,   // AOP, 메트릭
    MetricsConfig::class               // Prometheus
)
class UserPlatformApiApplication
```

### Consumer App

```kotlin
@EnableAutoConfiguration
@Import(
    EventConsumerAdapterConfig::class,    // Kafka 리스너
    PersistenceAdapterConfig::class,      // DB
    ClientAdapterConfig::class,           // 외부 API
    SlackAdapterConfig::class,            // Slack 알림
    ProducerAdapterConfig::class,         // Kafka 발행 (이벤트 전파)
    UseCaseConfig::class,                 // 비즈니스 로직
    KafkaConsumerConfig::class,           // Kafka Consumer 설정
    ApplicationSupportConfig::class       // AOP
)
class UserPlatformConsumerApplication
```

### Outbox App

```kotlin
@EnableAutoConfiguration
@Import(
    OutboxSchedulerAdapterConfig::class,  // 스케줄러
    PersistenceAdapterConfig::class,      // DB (아웃박스 테이블)
    SlackAdapterConfig::class,            // 실패 알림
    ApplicationSupportConfig::class,      // AOP
    OutboxKafkaConfig::class              // Kafka 발행 설정
)
class OutboxApplication
```

### Import 비교

| Config | API | Consumer | Outbox |
|--------|-----|----------|--------|
| WebAdapterConfig | ✓ | | |
| EventConsumerAdapterConfig | | ✓ | |
| OutboxSchedulerAdapterConfig | | | ✓ |
| PersistenceAdapterConfig | ✓ | ✓ | ✓ |
| ClientAdapterConfig | ✓ | ✓ | |
| CacheAdapterConfig | ✓ | | |
| ProducerAdapterConfig | ✓ | ✓ | |
| SlackAdapterConfig | | ✓ | ✓ |
| UseCaseConfig | ✓ | ✓ | |

Application 클래스만 보면 각 앱이 어떤 어댑터를 사용하는지 파악된다.

---

## profiles.include로 설정 합성

각 앱은 필요한 설정 파일을 `profiles.include`로 조합한다.

### API App의 application.yml

```yaml
spring:
  application:
    name: up-core-api-app
  profiles:
    active: api
    include: datasource, auth, logging, kafka, web-client, redis, client, auth-service, http-client
```

9개의 설정 파일을 포함한다.

### Consumer App의 application.yml

```yaml
spring:
  application:
    name: up-core-consumer-app
  profiles:
    active: consumer
    include: datasource, auth, logging, kafka, web-client, redis, client
```

7개의 설정 파일을 포함한다. `auth-service`, `http-client`가 빠졌다.

### Outbox App의 application.yml

```yaml
spring:
  application:
    name: up-core-outbox-app
  profiles:
    active: outbox
    include: datasource, auth, logging, kafka
```

4개의 설정 파일만 포함한다. 가장 단순한 구성이다.

### 포함 설정 비교

| 설정 파일 | API | Consumer | Outbox | 내용 |
|----------|-----|----------|--------|------|
| datasource | ✓ | ✓ | ✓ | PostgreSQL 연결 |
| auth | ✓ | ✓ | ✓ | 서비스 인증 |
| logging | ✓ | ✓ | ✓ | 로깅 레벨 |
| kafka | ✓ | ✓ | ✓ | Kafka 서버 |
| web-client | ✓ | ✓ | | WebClient 설정 |
| redis | ✓ | ✓ | | Redis 캐시 |
| client | ✓ | ✓ | | 외부 API URL |
| auth-service | ✓ | | | 인증 시스템 설정 |
| http-client | ✓ | | | HTTP 타임아웃 |

---

## 설정 파일 위치 전략

### 앱 전용 설정 vs 공유 설정

설정 파일은 두 곳에 위치한다.

**앱 모듈 (up-core-*-app/src/main/resources/)**
- `application.yml` - 앱별 기본 설정
- `application-datasource.yml` - DB 연결 (풀 사이즈가 앱마다 다름)
- `application-kafka.yml` - Kafka 설정 (Producer/Consumer 차이)

**어댑터 모듈 (up-adapter/*/src/main/resources/)**
- `application-client.yml` - 외부 API URL (모든 앱이 동일)
- `application-logging.yml` - 로깅 설정 (공통)

### 왜 이렇게 나누는가?

**Connection Pool 사이즈**

API 앱과 Consumer 앱의 DB 부하가 다르다.

```yaml
# API App - application-datasource.yml
spring:
  datasource:
    hikari:
      maximum-pool-size: 30  # 동시 HTTP 요청 처리

# Consumer App - application-datasource.yml
spring:
  datasource:
    hikari:
      maximum-pool-size: 10  # Kafka 파티션 수에 맞춤
```

**Kafka 설정**

API 앱은 Producer, Consumer 앱은 Consumer 설정이 필요하다.

```yaml
# API App - application-kafka.yml
spring:
  kafka:
    platform:
      producer:
        acks: all
        retries: 2147483647
        enable-idempotence: true

# Consumer App - application-kafka.yml
spring:
  kafka:
    platform:
      consumer:
        group-id: user-platform
        auto-offset-reset: earliest
        concurrency: 3
        enable-auto-commit: false
```

---

## 환경별 설정 오버라이드

Spring Boot의 [Multi-document Files](https://docs.spring.io/spring-boot/reference/features/external-config.html#features.external-config.files.multi-document) 기능으로 환경별 오버라이드를 같은 파일에서 처리한다.

```yaml
# application-datasource.yml

# Default (local)
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/userplat_int
  jpa:
    hibernate:
      ddl-auto: create  # 로컬: 스키마 자동 생성

---
spring.config.activate.on-profile: int

spring:
  datasource:
    url: jdbc:postgresql://common-int-main.rds.amazonaws.com/userplat_int
    password: ${POSTGRESQL_PASSWORD}
  jpa:
    hibernate:
      ddl-auto: validate  # 운영: 마이그레이션으로만 스키마 변경

---
spring.config.activate.on-profile: real

spring:
  datasource:
    url: ${DATABASE_URL}  # 프로덕션: 모든 민감 정보는 환경변수
    password: ${POSTGRESQL_PASSWORD}
```

**핵심 원칙:**
- **local**: 편의성 우선. 하드코딩, 자동 스키마 생성
- **int/stage**: 운영과 유사하게. 환경변수 + validate
- **real**: 보안 최우선. 모든 민감 정보는 환경변수

---

## 앱별 설정 차이

같은 설정 파일이라도 앱마다 값이 다를 수 있다.

### DB Connection Pool

```yaml
# API App - application-datasource.yml
spring:
  datasource:
    hikari:
      maximum-pool-size: 30  # 동시 HTTP 요청 처리

# Consumer App - application-datasource.yml
spring:
  datasource:
    hikari:
      maximum-pool-size: 10  # Kafka 파티션 수에 맞춤
```

### Kafka 설정

```yaml
# API App - Producer 설정
spring:
  kafka:
    producer:
      acks: all
      enable-idempotence: true

# Consumer App - Consumer 설정
spring:
  kafka:
    consumer:
      group-id: user-platform
      enable-auto-commit: false
```

앱 전용 설정은 각 앱 모듈의 `resources/`에, 공유 설정은 어댑터 모듈의 `resources/`에 위치한다.

---

## 기능 플래그

환경별로 기능을 On/Off한다.

```yaml
# Default (local)
kafka:
  enabled: false  # 로컬: Kafka 없이 개발
service:
  auth:
    enabled: false  # 로컬: 인증 비활성

---
spring.config.activate.on-profile: int

kafka:
  enabled: true
service:
  auth:
    enabled: true

---
spring.config.activate.on-profile: perf

kafka:
  enabled: false  # 성능 테스트: 순수 API 성능만 측정
```

로컬에서는 외부 의존성 없이 빠르게 개발한다. 운영에서는 모든 기능을 활성화한다.

---

## 앱 시작 시간

Import 패턴의 부가 효과로 앱 시작 시간이 줄어든다.

| 앱 | Import 수 | 설정 수 | 상대 시간 |
|----|----------|---------|----------|
| Outbox | 5 | 4 | 1x (기준) |
| Consumer | 8 | 7 | ~1.5x |
| API | 9 | 9 | ~2x |

Import 수가 적을수록 로드할 빈이 줄어든다. Outbox 앱은 스케줄러와 DB만 필요하니 가장 빠르게 뜬다.

---

## 정리

Spring Boot의 opinionated한 접근이 "하나의 앱"을 가정한다면, 우리는 Spring Boot가 제공하는 다른 Convention으로 멀티앱을 구성했다.

| 도구 | 역할 | Spring Boot Convention |
|-----|------|----------------------|
| `@Import` | 빈 등록 조합 | Part 1에서 다룸 |
| `profiles.include` | 설정 파일 조합 | 본 글에서 다룸 |
| `on-profile` | 환경별 오버라이드 | Multi-document Files |

**핵심 원칙:**
1. **빈 조합**: `@Import`로 앱별 어댑터 선택
2. **설정 조합**: `profiles.include`로 앱별 설정 선택
3. **환경 분리**: `on-profile`로 local/int/real 오버라이드
4. **명시성**: Application 클래스와 application.yml만 보면 구성 파악

다음 글에서는 이 구조가 테스트를 어떻게 쉽게 만드는지 다룬다.

---

**다음 글:** [Mock 남용 없는 통합 테스트](/dev-notes/posts/2024-03-22-spring-component-scan-philosophy-part3/)
