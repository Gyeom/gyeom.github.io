---
title: "Selectively Opinionated Spring Boot (3) - Mock 남용 없는 통합 테스트"
date: 2024-03-22
draft: false
tags: ["Spring", "Spring Boot", "Testing", "TestContainers", "Integration Test", "Kotest"]
categories: ["Spring"]
summary: "프로덕션과 테스트의 빈 구성을 일치시키면서도 빠른 피드백을 얻는 테스트 전략. TestContainer와 Mock 어댑터 활용"
series: ["Selectively Opinionated Spring Boot"]
series_order: 3
---

## 시리즈

1. [@ComponentScan의 함정](/dev-notes/posts/2024-03-15-spring-component-scan-philosophy-part1/)
2. [멀티앱, 하나의 코드베이스](/dev-notes/posts/2024-03-18-spring-component-scan-philosophy-part2/)
3. **Mock 남용 없는 통합 테스트** (현재 글)

---

## 들어가며

Spring Boot의 opinionated한 테스트 설정은 `@SpringBootTest`로 전체 애플리케이션을 띄우는 것이다. 간단하지만, 프로젝트가 커지면 테스트 시작 시간이 늘어난다. `@SpringBootApplication`이 패키지 전체를 스캔하니 테스트에 불필요한 컴포넌트까지 로드된다.

테스트를 빠르게 만드는 흔한 방법은 Mock을 늘리는 것이다. 하지만 Mock을 남용하면 테스트와 프로덕션의 괴리가 생긴다. 테스트는 통과했는데 프로덕션에서 실패하는 상황, 경험해봤을 것이다.

앞선 글에서 다룬 `@Import` 패턴은 이 딜레마를 해결한다. 프로덕션과 같은 Config를 사용하되, 테스트 범위에 맞는 Config만 Import한다. 외부 API 같은 진짜 의존성만 Mock 어댑터로 교체하고, 나머지는 실제 빈을 쓴다. TestContainer로 실제 DB와 Redis를 띄우면 빈 구성이 프로덕션과 일치한다. 테스트 결과를 신뢰할 수 있고, 범위를 좁혀서 피드백도 빠르다.

---

## Import 패턴의 테스트 이점

`@SpringBootApplication`을 쓰면 모든 테스트가 전체 애플리케이션을 로드한다. Import 패턴을 쓰면 테스트 범위에 맞는 최소한의 컴포넌트만 로드한다.

| 테스트 레벨 | Import 범위 | 컨테이너 | 상대 시간 |
|------------|------------|----------|----------|
| 어댑터 단위 | `PersistenceAdapterConfig` | PostgreSQL만 | 1x (기준) |
| 서비스 통합 | 어댑터 + `UseCaseConfig` | PostgreSQL + Redis | ~2x |
| E2E | 전체 앱 + `TestConfig` | 전체 | ~3x |

---

## 테스트 디렉토리 구조

```
up-core-api-app/
├── src/main/kotlin/...
└── src/integrationTest/
    ├── kotlin/com/example/platform/
    │   ├── TestContainerConfig.kt        # 인프라 컨테이너
    │   ├── HealthCheckIntegrationTest.kt
    │   ├── AccountIntegrationTest.kt
    │   ├── UserIntegrationTest.kt
    │   ├── support/
    │   │   └── BaseTestContainerSpec.kt  # 테스트 베이스 클래스
    │   └── test/
    │       ├── config/
    │       │   ├── TestConfig.kt         # 테스트용 Config
    │       │   └── DatabaseCleanup.kt    # DB 정리
    │       └── mock/
    │           ├── TestAuthServiceAdapter.kt     # Mock 인증 서비스
    │           └── TestGuestAuthServiceAdapter.kt
    └── resources/
        ├── application.yml
        ├── application-datasource.yml
        └── ...
```

`integrationTest` 소스셋으로 통합 테스트를 분리한다.

---

## TestContainer 기반 인프라

[Testcontainers](https://testcontainers.com/)로 실제 인프라를 Docker 컨테이너로 띄운다.

```kotlin
class TestContainerConfig : ApplicationContextInitializer<ConfigurableApplicationContext> {

    companion object {
        private val postgres = PostgreSQLContainer("postgres:15-alpine")
            .withDatabaseName("userplat_int")
            .withInitScript("test-schema.sql")
            .apply { start() }

        private val redis = GenericContainer(DockerImageName.parse("redis:7-alpine"))
            .withExposedPorts(6379)
            .apply { start() }
    }

    override fun initialize(ctx: ConfigurableApplicationContext) {
        TestPropertyValues.of(
            "spring.datasource.url=${postgres.jdbcUrl}",
            "spring.data.redis.host=${redis.host}",
            "spring.data.redis.port=${redis.getMappedPort(6379)}"
        ).applyTo(ctx)
    }
}
```

**핵심:**
- `companion object`: 여러 테스트 클래스가 같은 컨테이너 공유
- `ApplicationContextInitializer`: Spring Context에 동적 프로퍼티 주입
- H2 대신 실제 PostgreSQL 사용 → 프로덕션과 동일한 SQL 문법

---

## 테스트 베이스 클래스

### BaseTestContainerSpec

```kotlin
@SpringBootTest(
    webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT,
    classes = [UserPlatformApiApplication::class, TestConfig::class]
)
@ContextConfiguration(initializers = [TestContainerConfig::class])
@ActiveProfiles("integration")
@AutoConfigureMockMvc
abstract class BaseTestContainerSpec(
    protected val mockMvc: MockMvc,
    private val databaseCleanup: DatabaseCleanup
) : BehaviorSpec() {

    override fun isolationMode(): IsolationMode = IsolationMode.InstancePerLeaf

    override fun extensions() = listOf(SpringExtension)

    init {
        beforeSpec {
            databaseCleanup.execute()
        }
    }
}
```

**어노테이션 설명:**

| 어노테이션 | 설정 | 목적 |
|----------|------|------|
| `@SpringBootTest` | `RANDOM_PORT` | 포트 충돌 방지 |
| `@SpringBootTest` | `classes` | 앱 + TestConfig 로드 |
| `@ContextConfiguration` | `initializers` | TestContainer 적용 |
| `@ActiveProfiles` | `integration` | integration 프로필 |
| `@AutoConfigureMockMvc` | - | MockMvc 자동 설정 |

**Kotest 설정:**
- `BehaviorSpec`: Given/When/Then BDD 스타일
- `IsolationMode.InstancePerLeaf`: 각 테스트 케이스 격리
- `beforeSpec`: 테스트 시작 전 DB 정리

---

## 데이터베이스 정리

테스트 간 데이터 격리를 위해 매 테스트 전 테이블을 TRUNCATE한다.

```kotlin
@Component
class DatabaseCleanup(
    @PersistenceContext private val entityManager: EntityManager
) {
    private val tableNames = mutableSetOf<String>()

    @Transactional
    fun execute() {
        if (tableNames.isEmpty()) extractTableNames()
        tableNames.forEach { table ->
            entityManager.createNativeQuery(
                "TRUNCATE TABLE userplat.$table RESTART IDENTITY CASCADE"
            ).executeUpdate()
        }
    }
}
```

`RESTART IDENTITY CASCADE`로 Auto Increment ID 초기화와 FK 제약 조건을 함께 처리한다.

---

## TestConfig

### 테스트 전용 Config

```kotlin
@Configuration
@ComponentScan(basePackages = ["com.example.platform.test.config", "com.example.platform.test.mock"])
class TestConfig {

    @Bean
    @Primary
    fun objectProducerFactory(
        @Value("\${spring.kafka.bootstrap-servers}") bootstrapServers: String
    ): ProducerFactory<String, Any> {
        val configs = mutableMapOf<String, Any>()
        configs[ProducerConfig.BOOTSTRAP_SERVERS_CONFIG] = bootstrapServers
        configs[ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG] = StringSerializer::class.java
        configs[ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG] = JsonSerializer::class.java
        configs[JsonSerializer.ADD_TYPE_INFO_HEADERS] = false
        return DefaultKafkaProducerFactory(configs)
    }

    @Bean
    @Primary
    fun objectKafkaTemplate(
        objectProducerFactory: ProducerFactory<String, Any>
    ): KafkaTemplate<String, Any> {
        return KafkaTemplate(objectProducerFactory)
    }
}
```

**역할:**
- `com.example.platform.test.mock` 패키지의 Mock 어댑터 스캔
- TestContainer Kafka로 연결되는 KafkaTemplate 제공
- `@Primary`로 프로덕션 빈 오버라이드

---

## Mock 어댑터

외부 API를 호출하는 어댑터만 Mock으로 교체한다.

```kotlin
@Component("authServiceAdapter")
@Primary
class TestAuthServiceAdapter : AuthServiceOut {

    private val accountStore = mutableMapOf<String, AuthAccountInfo>()

    override fun registerAccount(accountSourceId: String, ...): AuthAccountInfo {
        accountStore[accountSourceId]?.let { return it }

        val accountInfo = AuthAccountInfo(accountId = UUID.randomUUID(), isActivated = true)
        accountStore[accountSourceId] = accountInfo
        return accountInfo
    }

    override fun removeAccount(accountId: UUID) {
        accountStore.entries.removeIf { it.value.accountId == accountId }
    }
}
```

**핵심:**
- `@Component("authServiceAdapter")`: 프로덕션과 같은 빈 이름
- `@Primary`: 테스트에서 이 빈이 우선
- 인메모리 저장소로 외부 API 호출 없이 동작

### 빈 선택 메커니즘

```kotlin
// UseCaseConfig (프로덕션/테스트 공용)
@Bean
fun apiAccountService(
    @Qualifier("authServiceAdapter") authOut: AuthServiceOut,  // 테스트: TestAuthServiceAdapter
    // ...
): AccountUseCase
```

프로덕션의 Config가 그대로 동작한다. `@Primary`가 붙은 Mock만 교체된다.

---

## 통합 테스트 예제

```kotlin
class AccountIntegrationTest(
    mockMvc: MockMvc,
    databaseCleanup: DatabaseCleanup,
    private val objectMapper: ObjectMapper,
    private val accountTypeOut: AccountTypeOut,
    private val userOut: UserOut
) : BaseTestContainerSpec(mockMvc, databaseCleanup) {

    init {
        Given("계정 정보가 주어진 상태에서") {
            val accountType = createTestAccountType()
            val user = createTestUser()
            val request = CreateAccountRequest(
                userId = user.id.value,
                accountSourceId = "account-source-001",
                accountTypeId = accountType.id.value
            )

            When("계정 생성 API를 호출하면") {
                Then("계정이 정상적으로 생성되어야 한다") {
                    mockMvc.post("/api/v1/accounts") {
                        contentType = MediaType.APPLICATION_JSON
                        content = objectMapper.writeValueAsString(request)
                    }.andExpect {
                        status { isCreated() }
                        jsonPath("$.id") { exists() }
                    }
                }
            }
        }
    }
}
```

**테스트 패턴:**
- Given: 사전 조건 (DB에 테스트 데이터 삽입)
- When: API 호출
- Then: 응답 검증

Port 인터페이스(`accountTypeOut`, `userOut`)로 테스트 데이터를 직접 삽입한다. Repository가 아닌 Port를 쓰므로 도메인 로직을 거친다.

---

## 어댑터 레벨 테스트

전체 앱을 로드하지 않고 어댑터만 테스트한다.

### PersistenceTestConfig

```kotlin
class PersistenceTestConfig : ApplicationContextInitializer<ConfigurableApplicationContext> {

    companion object {
        private val postgres: PostgreSQLContainer<*> = PostgreSQLContainer("postgres:15-alpine")
            .withDatabaseName("userplat_int")
            .withUsername("test_user")
            .withPassword("test_password")
            .apply { start() }
    }

    override fun initialize(ctx: ConfigurableApplicationContext) {
        TestPropertyValues.of(
            "spring.datasource.url=${postgres.jdbcUrl}",
            "spring.datasource.username=${postgres.username}",
            "spring.datasource.password=${postgres.password}",
            "spring.jpa.hibernate.ddl-auto=create-drop",
            "spring.jpa.show-sql=true"
        ).applyTo(ctx)
    }
}
```

**차이점:**
- `ddl-auto=create-drop`: 테스트 실행 시 스키마 생성, 종료 시 삭제
- PostgreSQL만 사용 (Redis, Kafka 없음)

### 어댑터 테스트 예제

```kotlin
@SpringBootTest(classes = [PersistenceAdapterConfig::class])
@ContextConfiguration(initializers = [PersistenceTestConfig::class])
class AccountPersistenceAdapterTest(
    private val accountOut: AccountOut
) : BehaviorSpec({

    Given("계정이 저장된 상태에서") {
        val account = Account(...)
        val savedAccount = accountOut.save(account)

        When("ID로 조회하면") {
            val found = accountOut.findById(savedAccount.id)

            Then("동일한 계정이 반환된다") {
                found shouldNotBe null
                found?.id shouldBe savedAccount.id
            }
        }
    }
})
```

**로드되는 컴포넌트:**
- `PersistenceAdapterConfig`만 Import
- JPA Repository, Entity, Adapter만 로드
- Controller, UseCase, 다른 어댑터는 로드하지 않음

---

## Consumer App 테스트

Consumer App은 Feign 클라이언트를 MockK로 대체한다.

```kotlin
@TestConfiguration
class MockAdapterConfig {

    @Bean
    @Primary
    fun mockAuthServiceAdapter(): AuthServiceAdapter = mockk<AuthServiceAdapter>().apply {
        every { registerAccount(any(), any(), any(), any(), any()) } returns
            AuthAccountInfo(accountId = UUID.randomUUID(), isActivated = true)
        every { removeAccount(any()) } returns Unit
    }

    @Bean(name = ["authAccountManagementClient"])
    @Primary
    fun mockAuthClient(): AuthAccountManagementClient = mockk<AuthAccountManagementClient>().apply {
        every { activateAccount(any()) } returns AccountDetailCommonFeignResponse(accountId = UUID.randomUUID())
    }
}
```

API App의 Mock 어댑터와 달리 MockK를 사용한다. Feign 클라이언트는 인터페이스만 있고 구현이 없으므로, 인메모리 구현보다 MockK가 간편하다.

---

## 테스트 설정 파일

### integrationTest/resources/application.yml

```yaml
server:
  port: 8080
  shutdown: graceful

spring:
  profiles:
    include: datasource, auth, kafka, web-client, logging, redis, test
  main:
    allow-bean-definition-overriding: true
```

`allow-bean-definition-overriding: true`로 Mock 빈이 프로덕션 빈을 오버라이드한다.

### application-test.yml

```yaml
# 테스트 특화 설정
service:
  auth:
    enabled: false  # 테스트에서 인증 비활성

logging:
  level:
    org.hibernate.SQL: DEBUG
    org.hibernate.type.descriptor.sql: TRACE
```

테스트 환경에서만 필요한 설정을 분리한다.

---

## 테스트 피라미드

```
           /\              E2E (전체 앱)
          /  \             - BaseTestContainerSpec
         /    \            - 모든 컨테이너
        /------\
       /        \          서비스 통합
      /  통합    \         - 어댑터 + UseCase
     /  테스트    \        - 필요한 컨테이너만
    /--------------\
   /                \      어댑터 테스트
  /  어댑터 단위     \     - 단일 Config만 로드
 /--------------------\
```

**Import 패턴의 이점:**
- 테스트 범위에 맞는 Config만 Import
- 불필요한 빈 로드 없이 빠른 피드백
- 테스트 격리 보장

---

## 정리

Import 패턴은 테스트를 쉽게 만든다.

**핵심 원칙:**
1. TestContainer로 실제 인프라 사용 (Mock DB 대신 Real DB)
2. Mock 어댑터로 외부 의존성 제거
3. `@Primary`로 프로덕션 빈 오버라이드
4. 어댑터 레벨 테스트로 빠른 피드백
5. 전체 통합 테스트로 E2E 검증

**테스트 구성 요소:**
- `TestContainerConfig`: 인프라 컨테이너
- `BaseTestContainerSpec`: 테스트 베이스 클래스
- `DatabaseCleanup`: 테스트 간 데이터 격리
- `TestConfig`: Mock 어댑터 스캔
- `Test*Adapter`: 외부 API Mock 구현

프로덕션과 테스트의 빈 구성이 일치하므로 테스트 결과를 신뢰할 수 있다.

---

## 시리즈 마무리

### Hexagonal Architecture와의 연결

[Alistair Cockburn의 원문](https://alistair.cockburn.us/hexagonal-architecture/)에서 Hexagonal Architecture의 목표를 이렇게 정의한다.

> "Allow an application to equally be driven by users, programs, automated test or batch scripts"

사용자, 프로그램, **자동화된 테스트**가 동등하게 애플리케이션을 구동할 수 있어야 한다. 이 시리즈에서 다룬 패턴은 이 목표를 Spring Boot에서 구현하는 방법이다.

- **Part 1**: Port/Adapter 경계를 `@Import`로 명시
- **Part 2**: 같은 Port, 다른 Adapter 조합 (API앱 vs Consumer앱)
- **Part 3**: Driven Adapter를 테스트용으로 교체

Adapter를 쉽게 교체할 수 있으면 테스트도 쉬워진다. Mock Adapter는 Hexagonal Architecture의 핵심 이점이다.

### Opinionated, 하지만 선택적으로

Spring Boot는 스스로를 "opinionated"하다고 말한다. 합리적인 기본값을 제공하고, 개발자가 내려야 할 결정을 줄여준다. 이 시리즈는 그 철학을 거부하는 것이 아니라, **어디까지 받아들일지** 경계를 정하는 이야기였다.

| 영역 | Spring Boot Convention | 우리의 선택 |
|-----|----------------------|----------|
| 인프라 설정 | AutoConfiguration | ✅ 따른다 |
| 빈 등록 | @ComponentScan | ❌ @Import로 대체 |
| 설정 파일 | 단일 application.yml | ❌ profiles.include로 조합 |
| 테스트 | @SpringBootTest 전체 로드 | ❌ 범위별 Config Import |

### 이 패턴이 적합한 상황

- 프로젝트 규모가 커서 "무엇이 등록되는지" 파악이 어려울 때
- 하나의 코드베이스에서 여러 앱을 운영할 때
- 테스트 시작 시간이 문제가 될 때
- Hexagonal Architecture로 어댑터를 명확히 분리할 때

### 적합하지 않은 상황

- 작은 프로젝트에서 빠르게 시작하고 싶을 때
- 팀원 대부분이 Spring Boot Convention에 익숙할 때
- @ComponentScan의 "암묵적 동작"이 문제가 되지 않을 때

Spring Boot의 Convention over Configuration은 좋은 철학이다. 다만 **인프라**에는 Convention을, **비즈니스**에는 Configuration을 적용하는 것이 이 시리즈의 결론이다.

### 참고 자료

- [Hexagonal Architecture - Alistair Cockburn](https://alistair.cockburn.us/hexagonal-architecture/)
- [Hexagonal Architecture - Ports and Adapters Pattern](https://jmgarridopaz.github.io/content/hexagonalarchitecture.html)
- [Spring Boot Reference - Auto-configuration](https://docs.spring.io/spring-boot/reference/using/auto-configuration.html)
- [Spring Boot Reference - Externalized Configuration](https://docs.spring.io/spring-boot/reference/features/external-config.html)
- [Spring Boot Reference - Profiles](https://docs.spring.io/spring-boot/reference/features/profiles.html)
- [Testcontainers](https://testcontainers.com/)
