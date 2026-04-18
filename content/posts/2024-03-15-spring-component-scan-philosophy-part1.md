---
title: "Selectively Opinionated Spring Boot (1) - @ComponentScan의 함정"
date: 2024-03-15
draft: false
tags: ["Spring", "Spring Boot", "Hexagonal Architecture", "Component Scan", "아키텍처"]
categories: ["Spring"]
summary: "Spring Boot의 Convention over Configuration은 어디까지 받아들여야 하는가. @SpringBootApplication 대신 @EnableAutoConfiguration + @Import 패턴을 선택한 이유"
series: ["Selectively Opinionated Spring Boot"]
series_order: 1
---

## 시리즈

1. **@ComponentScan의 함정** (현재 글)
2. [멀티앱, 하나의 코드베이스](/dev-notes/posts/2024-03-18-spring-component-scan-philosophy-part2/)
3. [Mock 남용 없는 통합 테스트](/dev-notes/posts/2024-03-22-spring-component-scan-philosophy-part3/)

---

## 들어가며

Spring Boot는 스스로를 "opinionated"하다고 말한다. 합리적인 기본값을 제공하고, 개발자가 내려야 할 결정을 줄여준다. `@SpringBootApplication` 하나면 애플리케이션이 동작한다. 클래스패스에 라이브러리만 추가하면 자동으로 설정된다.

하지만 이 편리함에는 대가가 있다. "이 서비스가 어떤 빈을 주입받는지 알려면 어디를 봐야 하나요?" 프로젝트가 커지면 자주 듣는 질문이다. IDE가 빈을 찾아줘도, 그 빈이 어떤 Config에서 왔는지는 보이지 않는다.

[Hexagonal Architecture](https://alistair.cockburn.us/hexagonal-architecture/)를 적용하면서 이 문제가 더 뚜렷해졌다. Alistair Cockburn이 정의한 이 아키텍처의 핵심은 Port와 Adapter의 경계가 명확해야 한다는 것이다. 어댑터가 늘어날수록 "이 앱에 어떤 어댑터가 붙어 있는가"를 파악하기 어려워졌다. Spring Boot의 Convention이 인프라 설정에서는 빛을 발하지만, 비즈니스 로직의 의존성까지 숨기면 문제가 된다.

이 시리즈는 Spring Boot의 "opinionated" 철학을 어디까지 받아들이고, 어디서부터 명시적으로 관리할지에 대한 이야기다.

---

## Spring Boot의 Convention over Configuration

### AutoConfiguration의 동작 원리

Spring Boot는 [Convention over Configuration](https://docs.spring.io/spring-framework/reference/overview.html) 철학을 따른다. 개발자가 내려야 할 결정을 줄이고, 합리적인 기본값을 제공한다.

> Spring Boot is opinionated. It provides sensible defaults so you can start quickly.

자동 설정의 핵심은 [`META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`](https://docs.spring.io/spring-boot/reference/using/auto-configuration.html) 파일이다. Spring Boot 2.7부터 도입된 방식이다.

```
# spring-boot-autoconfigure.jar 내부
org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration
org.springframework.boot.autoconfigure.orm.jpa.HibernateJpaAutoConfiguration
org.springframework.boot.autoconfigure.kafka.KafkaAutoConfiguration
...
```

Spring Boot는 이 파일을 읽고, `@Conditional` 어노테이션으로 조건을 평가한다.

| 어노테이션 | 조건 |
|-----------|------|
| `@ConditionalOnClass` | 특정 클래스가 클래스패스에 있을 때 |
| `@ConditionalOnMissingBean` | 해당 타입의 빈이 없을 때 |
| `@ConditionalOnProperty` | 특정 프로퍼티가 설정되었을 때 |

예를 들어 `DataSourceAutoConfiguration`은 `DataSource.class`가 클래스패스에 있고, 개발자가 직접 `DataSource` 빈을 정의하지 않았을 때만 동작한다. [Spring 공식 문서](https://docs.spring.io/spring-boot/reference/using/auto-configuration.html)는 "Auto-configuration is always applied after user-defined beans have been registered"라고 명시한다. 개발자가 정의한 빈이 항상 우선이다.

### 우리가 긋는 경계: 인프라 vs 비즈니스

경계를 **인프라**와 **비즈니스**로 나눈다.

| 구분 | 설정 방식 | 예시 |
|------|----------|------|
| 인프라 | AutoConfiguration (암묵적) | DataSource, JPA, Kafka, Redis |
| 비즈니스 | @Import (명시적) | Controller, Adapter, UseCase |

**인프라**는 Spring Boot의 Convention을 따른다. DataSource, JPA, Kafka 설정은 Spring Boot가 제공하는 기본값이 충분히 좋고, `application.yml`로 튜닝할 수 있다.

**비즈니스**는 명시적으로 관리한다. 어떤 Controller가 등록되는지, 어떤 Adapter가 활성화되는지, UseCase가 어떤 Port를 의존하는지는 코드에서 바로 보여야 한다.

---

## @SpringBootApplication의 문제

대부분의 Spring Boot 프로젝트는 이렇게 시작한다.

```kotlin
@SpringBootApplication
class MyApplication

fun main(args: Array<String>) {
    runApplication<MyApplication>(*args)
}
```

`@SpringBootApplication`은 세 가지 어노테이션의 조합이다.

```kotlin
@SpringBootConfiguration  // @Configuration 포함
@EnableAutoConfiguration  // 자동 설정
@ComponentScan           // 패키지 전체 스캔
```

문제는 `@ComponentScan`이다. 애플리케이션 클래스가 위치한 패키지와 그 하위 패키지를 **전부** 스캔한다.

### 암묵적 의존성의 위험

```
com.example/
├── MyApplication.kt        # @SpringBootApplication
├── controller/
│   └── UserController.kt   # @RestController - 자동 등록
├── service/
│   └── UserService.kt      # @Service - 자동 등록
├── repository/
│   └── UserRepository.kt   # @Repository - 자동 등록
└── config/
    └── SecurityConfig.kt   # @Configuration - 자동 등록
```

모든 빈이 "알아서" 등록된다. 편리하지만 위험하다.

**문제 1: 의존성이 보이지 않는다**

`MyApplication.kt`만 보면 이 애플리케이션이 어떤 컴포넌트로 구성되는지 알 수 없다. 실제로 어떤 빈이 등록되는지 확인하려면 모든 패키지를 뒤져야 한다.

**문제 2: 원치 않는 빈이 등록된다**

테스트용 클래스에 `@Component`를 붙이면 프로덕션에서도 등록된다. 특정 환경에서만 필요한 빈도 항상 등록된다.

**문제 3: 멀티 모듈에서 혼란**

```
project/
├── app-api/           # API 서버
├── app-consumer/      # Kafka 컨슈머
├── adapter-web/       # HTTP 어댑터
├── adapter-persistence/  # DB 어댑터
└── domain/            # 도메인 로직
```

`app-api`와 `app-consumer`가 같은 어댑터 모듈을 의존하면, 어떤 어댑터가 어느 앱에서 활성화되는지 파악하기 어렵다.

---

## 해결: @EnableAutoConfiguration + @Import

```kotlin
@EnableAutoConfiguration
@Import(
    WebAdapterConfig::class,
    PersistenceAdapterConfig::class,
    ClientAdapterConfig::class,
    CacheAdapterConfig::class,
    ProducerAdapterConfig::class,
    UseCaseConfig::class,
)
class UserPlatformApiApplication

fun main(args: Array<String>) {
    TimeZone.setDefault(TimeZone.getTimeZone("UTC"))
    runApplication<UserPlatformApiApplication>(*args)
}
```

### 핵심 차이

| 항목 | @SpringBootApplication | @EnableAutoConfiguration + @Import |
|------|------------------------|-----------------------------------|
| 빈 등록 | 암묵적 (패키지 스캔) | 명시적 (Config 클래스) |
| 의존성 파악 | 전체 코드 탐색 필요 | Application 클래스만 보면 됨 |
| 빈 제어 | 어려움 | Config 단위로 On/Off |
| 멀티 앱 | 복잡 | 앱별 Config 조합 |

`@EnableAutoConfiguration`은 Spring Boot의 인프라 자동 설정을 유지한다. `@Import`로 우리가 만든 Config 클래스만 명시적으로 등록한다.

---

## 어댑터별 Config 클래스

### Inbound Adapter: WebAdapterConfig

HTTP 요청을 처리하는 컴포넌트를 등록한다.

```kotlin
@Configuration
@ComponentScan(
    basePackages = ["com.example.platform.adapter.inbound.web"]
)
@ConfigurationPropertiesScan(
    basePackages = ["com.example.platform.adapter.inbound.web.config"]
)
class WebAdapterConfig
```

이 Config가 Import되면 `com.example.platform.adapter.inbound.web` 패키지의 컴포넌트만 스캔한다.

**등록되는 컴포넌트:**
- `@RestController`: UserController, AccountController, ProfileController
- `@Component`: CorrelationIdFilter, AuthenticationFilter, AuditFilter
- `@ControllerAdvice`: GlobalExceptionHandler

### Outbound Adapter: PersistenceAdapterConfig

데이터베이스 접근을 담당한다.

```kotlin
@Configuration
@ComponentScan(
    basePackages = ["com.example.platform.adapter.outbound.persistence"]
)
@EnableJpaRepositories(
    basePackages = ["com.example.platform.adapter.outbound.persistence"]
)
@EntityScan(
    basePackages = ["com.example.platform.adapter.outbound.persistence"]
)
class PersistenceAdapterConfig
```

**등록되는 컴포넌트:**
- `@Repository`: Spring Data JPA 인터페이스
- `@Entity`: JPA 엔티티
- `@Adapter`: Port 구현체 (UserPersistenceAdapter, AccountPersistenceAdapter)

### Outbound Adapter: ClientAdapterConfig

외부 API 호출을 담당한다.

```kotlin
@Configuration
@EnableFeignClients(
    basePackages = ["com.example.platform.adapter.outbound.client"]
)
@ComponentScan(
    basePackages = ["com.example.platform.adapter.outbound.client"]
)
class ClientAdapterConfig {

    @Bean
    fun feignLoggerLevel(): Logger.Level = Logger.Level.FULL

    @Bean
    fun feignRequestOptions(): Request.Options = Request.Options(
        Duration.ofMillis(5000),   // connect timeout
        Duration.ofMillis(10000),  // read timeout
        true
    )

    @Bean
    fun feignRetryer(): Retryer = Retryer.Default(
        1000L,  // initial interval
        3000L,  // max interval
        3       // max attempts
    )
}
```

Config 클래스에 Feign 공통 설정(타임아웃, 재시도)도 함께 정의한다.

### Outbound Adapter: ProducerAdapterConfig / CacheAdapterConfig

```kotlin
@Configuration
@ComponentScan(basePackages = ["com.example.platform.adapter.outbound.producer"])
class ProducerAdapterConfig

@Configuration
@ComponentScan(basePackages = ["com.example.platform.adapter.outbound.cache"])
class CacheAdapterConfig {
    @Bean
    fun redisTemplate(connectionFactory: RedisConnectionFactory): RedisTemplate<String, Any> {
        // RedisTemplate 설정
    }
}
```

---

## UseCase: Bean 메서드로 명시적 등록

UseCase 클래스는 `@Service` 어노테이션을 붙이지 않는다. Config에서 `@Bean` 메서드로 등록한다.

```kotlin
@Configuration
@EnableAspectJAutoProxy
class UseCaseConfig {

    @Bean
    fun accountService(
        accountOut: AccountOut,
        accountTypeOut: AccountTypeOut,
        @Qualifier("authServiceAdapter") authOut: AuthServiceOut,
        userOut: UserOut,
        eventOut: EventOut
    ): AccountUseCase = AccountService(
        accountOut, accountTypeOut, authOut, userOut, eventOut
    )

    @Bean
    fun userService(
        userGroupOut: UserGroupOut,
        userOut: UserOut,
        accountOut: AccountOut,
        eventOut: EventOut
    ): UserUseCase = UserService(
        userGroupOut, userOut, accountOut, eventOut
    )
}
```

### 왜 @Service를 쓰지 않는가?

**1. 의존성이 명시적으로 드러난다**

`AccountService`가 5개의 Port를 의존한다는 사실이 코드에서 바로 보인다.

**2. 같은 인터페이스의 여러 구현체를 다룰 수 있다**

```kotlin
@Bean
fun accountService(
    @Qualifier("authServiceAdapter") authOut: AuthServiceOut,  // 실제 인증 API
): AccountUseCase = AccountService(...)

@Bean(name = ["guestAccountUseCase"])
fun guestAccountService(
    @Qualifier("guestAuthServiceAdapter") authOut: AuthServiceOut,  // 게스트용
): AccountUseCase = GuestAccountService(...)
```

`@Qualifier`로 어떤 구현체를 주입할지 명시한다.

**3. 앱별로 다른 UseCase를 등록할 수 있다**

API 앱과 Consumer 앱이 같은 UseCase 클래스를 공유하지만, 필요한 것만 등록한다.

---

## 정리

| 구분 | 방식 | 이유 |
|------|------|------|
| 인프라 (DataSource, Kafka) | AutoConfiguration | Spring Boot 기본값이 충분히 좋다 |
| 어댑터 (Web, Persistence) | @Import + ComponentScan | 앱별로 필요한 어댑터만 조립 |
| UseCase | @Bean 메서드 | 의존성을 명시적으로 드러냄 |

**핵심 원칙:**
1. Application 클래스만 보면 전체 구성을 파악할 수 있어야 한다
2. 어댑터는 Config 단위로 On/Off 할 수 있어야 한다
3. UseCase의 의존성은 코드에서 바로 보여야 한다

다음 글에서는 이 패턴이 멀티앱 환경에서 어떻게 활용되는지 다룬다.

---

**다음 글:** [멀티앱, 하나의 코드베이스](/dev-notes/posts/2024-03-18-spring-component-scan-philosophy-part2/)
