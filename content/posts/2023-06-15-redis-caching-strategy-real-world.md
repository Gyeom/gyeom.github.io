---
title: "Redis 캐싱 전략: 실무에서 마주치는 문제와 해결법"
date: 2023-06-15
draft: false
tags: ["Redis", "Cache", "Spring Boot", "성능최적화"]
categories: ["Backend"]
summary: "Cache-Aside, Write-Through, Write-Behind 패턴과 Multi-tier 캐시 구성, Cache Stampede 방지 및 Circuit Breaker 전략을 실무 관점에서 정리한다."
---

## 왜 캐싱인가

데이터베이스 조회는 비용이 크다. 네트워크 왕복, 디스크 I/O, 쿼리 파싱 등 여러 단계를 거친다. 자주 조회되는 데이터를 메모리에 캐싱하면 응답 시간을 수십 ms에서 수 ms로 줄일 수 있다.

하지만 캐싱은 단순히 "Redis에 넣으면 끝"이 아니다. 캐시 일관성, 무효화 전략, 장애 대응까지 고려해야 한다.

---

## 캐시 패턴

### Cache-Aside (Lazy Loading)

가장 널리 사용되는 패턴이다. 애플리케이션이 캐시를 직접 관리한다.

```kotlin
fun getProduct(id: String): Product {
    val cacheKey = "product:$id"

    // 1. 캐시 조회
    redisTemplate.opsForValue().get(cacheKey)?.let { return it }

    // 2. 캐시 미스 → DB 조회
    val product = productRepository.findById(id)
        .orElseThrow { NotFoundException("Product not found: $id") }

    // 3. 캐시에 저장
    redisTemplate.opsForValue().set(cacheKey, product, Duration.ofMinutes(30))

    return product
}
```

**장점**: 필요한 데이터만 캐싱, 구현이 단순함
**단점**: 첫 요청은 항상 느림(Cache Miss), 캐시와 DB 불일치 가능

### Write-Through

쓰기 시 캐시와 DB를 동시에 업데이트한다.

```kotlin
@Transactional
fun updateProduct(id: String, command: UpdateCommand): Product {
    val product = productRepository.findById(id).orElseThrow()
    product.update(command)
    val saved = productRepository.save(product)

    // 캐시도 함께 업데이트
    redisTemplate.opsForValue().set("product:$id", saved, Duration.ofMinutes(30))
    return saved
}
```

**장점**: 캐시와 DB 일관성 유지
**단점**: 쓰기 지연 증가, 읽히지 않을 데이터도 캐싱

### Write-Behind (Write-Back)

쓰기를 캐시에만 하고, 비동기로 DB에 반영한다.

```kotlin
// 캐시에만 쓰기 (빠름)
fun saveEvent(event: Event) {
    redisTemplate.opsForValue().set("event:${event.id}:latest", event)
    redisTemplate.opsForList().rightPush("event:pending", event)  // 배치 대기열
}

// 주기적으로 DB에 반영
@Scheduled(fixedDelay = 5000)
fun flushToDatabase() {
    val pending = mutableListOf<Event>()
    while (true) {
        redisTemplate.opsForList().leftPop("event:pending")?.let { pending.add(it) } ?: break
    }
    if (pending.isNotEmpty()) eventRepository.saveAll(pending)
}
```

**장점**: 쓰기 성능 극대화, 배치 처리로 DB 부하 감소
**단점**: 데이터 유실 위험, 구현 복잡도 증가

---

## Multi-tier 캐시

Redis만으로 부족할 때가 있다. 네트워크 왕복 시간(~1ms)도 아끼고 싶다면 Local Cache를 추가한다.

```
요청 → Local Cache (Caffeine) → Distributed Cache (Redis) → Database
         ~0.01ms                    ~1ms                      ~10ms
```

### 구현

```kotlin
@Configuration
class CacheConfig {

    @Bean
    fun cacheManager(redisConnectionFactory: RedisConnectionFactory): CacheManager {
        val caffeine = CaffeineCacheManager().apply {
            setCaffeine(
                Caffeine.newBuilder()
                    .maximumSize(1000)
                    .expireAfterWrite(Duration.ofMinutes(5))
            )
        }

        val redis = RedisCacheManager.builder(redisConnectionFactory)
            .cacheDefaults(
                RedisCacheConfiguration.defaultCacheConfig()
                    .entryTtl(Duration.ofMinutes(30))
            )
            .build()

        return CompositeCacheManager(caffeine, redis)
    }
}
```

### 계층별 역할

| 계층 | 저장소 | TTL | 용도 |
|------|--------|-----|------|
| L1 | Caffeine | 5분 | Hot data, 인스턴스별 |
| L2 | Redis | 30분 | Warm data, 공유 |
| L3 | Database | - | Cold data, 원본 |

### 주의점: 캐시 일관성

Multi-tier에서 가장 어려운 문제는 일관성이다. 데이터가 업데이트되면 모든 인스턴스의 Local Cache를 무효화해야 한다.

```kotlin
// Redis Pub/Sub으로 캐시 무효화 전파
fun invalidate(key: String) {
    localCache.evict(key)
    redisTemplate.convertAndSend("cache:invalidate", key)  // 다른 인스턴스에 전파
}

@EventListener
fun onCacheInvalidate(message: CacheInvalidateMessage) {
    localCache.evict(message.key)
}
```

---

## 캐시 무효화 전략

"컴퓨터 과학에서 어려운 두 가지: 캐시 무효화와 이름 짓기" - Phil Karlton

### TTL 기반

가장 단순하다. 일정 시간 후 자동 만료.

```kotlin
redisTemplate.opsForValue().set(key, value, Duration.ofMinutes(30))
```

**적합한 경우**: 약간의 지연이 허용되는 데이터 (ex: 통계, 집계)

### 이벤트 기반

데이터 변경 시 즉시 무효화.

```kotlin
@Transactional
fun updateProduct(id: String, command: UpdateCommand): Product {
    val product = productRepository.findById(id).orElseThrow()
    product.update(command)
    val saved = productRepository.save(product)

    eventPublisher.publishEvent(ProductUpdatedEvent(saved))  // 이벤트 발행
    return saved
}

@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
fun onProductUpdated(event: ProductUpdatedEvent) {
    cacheService.invalidate("product:${event.product.id}")
}
```

**적합한 경우**: 실시간 일관성이 중요한 데이터

### 버전 기반

캐시 키에 버전을 포함시킨다.

```kotlin
private var version: Long = System.currentTimeMillis()

fun getConfig(key: String): Config {
    val cacheKey = "config:$key:v$version"
    redisTemplate.opsForValue().get(cacheKey)?.let { return it }

    val config = configRepository.findByKey(key)
    redisTemplate.opsForValue().set(cacheKey, config)
    return config
}

// 버전 변경 → 모든 캐시 자동 무효화
fun refreshAll() {
    version = System.currentTimeMillis()
}
```

**적합한 경우**: 설정 데이터처럼 한 번에 전체 갱신이 필요한 경우

---

## 캐시 장애 대응

### Cache Stampede 방지

캐시가 만료되는 순간 다수의 요청이 DB로 몰리는 현상. 여러 해결책이 있다.

#### 1. 분산 락 (Distributed Lock)

하나의 요청만 DB를 조회하고, 나머지는 대기한다.

```kotlin
fun getProduct(id: String): Product {
    val cacheKey = "product:$id"
    redisTemplate.opsForValue().get(cacheKey)?.let { return it }

    val lock = lockRegistry.obtain(cacheKey)
    return if (lock.tryLock(100, TimeUnit.MILLISECONDS)) {
        try {
            // Double-check: 락 획득 사이에 다른 요청이 캐시를 채웠을 수 있음
            redisTemplate.opsForValue().get(cacheKey)?.let { return it }

            val product = productRepository.findById(id).orElseThrow()
            redisTemplate.opsForValue().set(cacheKey, product, Duration.ofMinutes(30))
            product
        } finally { lock.unlock() }
    } else {
        Thread.sleep(50)
        getProduct(id)  // 재시도
    }
}
```

#### 2. Probabilistic Early Expiration (PER)

TTL 만료 전에 확률적으로 캐시를 미리 갱신한다. Netflix 등에서 사용하는 방식.

```kotlin
fun getProduct(id: String): Product {
    val cacheKey = "product:$id"
    val cached = redisTemplate.opsForValue().get(cacheKey)
    val ttl = redisTemplate.getExpire(cacheKey, TimeUnit.SECONDS)

    // TTL이 20% 이하로 남았을 때, 10% 확률로 미리 갱신
    if (cached != null && ttl > 0) {
        val shouldRefresh = ttl < TTL_SECONDS * 0.2 && Random.nextDouble() < 0.1
        if (!shouldRefresh) return cached
    }

    val product = productRepository.findById(id).orElseThrow()
    redisTemplate.opsForValue().set(cacheKey, product, Duration.ofSeconds(TTL_SECONDS))
    return product
}
```

**장점**: 락 없이 자연스럽게 분산, 구현 단순
**단점**: 확률적이라 완벽하지 않음

#### 3. Singleflight (Request Coalescing)

동일 키에 대한 동시 요청을 하나로 합친다. Caffeine 캐시가 이 패턴을 내장한다.

```kotlin
// Caffeine의 get(key, loader)는 자동으로 Singleflight 적용
private val cache = Caffeine.newBuilder()
    .maximumSize(1000)
    .expireAfterWrite(Duration.ofMinutes(5))
    .build<String, Product>()

fun getProduct(id: String): Product {
    return cache.get(id) { key ->
        productRepository.findById(key).orElseThrow()  // 동시 요청은 하나만 실행
    }
}
```

**장점**: 별도 락 불필요, 라이브러리가 처리
**단점**: Local Cache에만 적용 (Redis에는 분산 락 필요)

#### 4. Stale-While-Revalidate (Soft TTL)

만료된 데이터라도 일단 반환하고, 백그라운드에서 갱신한다.

```kotlin
data class CacheEntry<T>(val data: T, val softExpireAt: Long, val hardExpireAt: Long)

fun getProduct(id: String): Product {
    val cacheKey = "product:$id"
    val entry = redisTemplate.opsForValue().get(cacheKey) as CacheEntry<Product>?

    val now = System.currentTimeMillis()
    if (entry != null) {
        if (now < entry.softExpireAt) return entry.data  // 신선함

        // Soft TTL 지남 → 일단 반환하고 백그라운드 갱신
        if (now < entry.hardExpireAt) {
            refreshAsync(id)  // 비동기 갱신
            return entry.data  // stale 데이터 반환
        }
    }

    // Hard TTL 지남 → 동기 갱신
    return refreshSync(id)
}
```

**장점**: 사용자 대기 시간 최소화
**단점**: 일시적으로 오래된 데이터 반환

#### 해결책 비교

| 방식 | 복잡도 | 지연 | 일관성 | 적합한 경우 |
|------|:------:|:----:|:------:|------------|
| 분산 락 | 중 | 있음 | 높음 | 일관성 중요 |
| PER | 낮 | 없음 | 중 | 읽기 많은 서비스 |
| Singleflight | 낮 | 없음 | 높음 | Local Cache 사용 시 |
| Soft TTL | 중 | 없음 | 낮 | 약간의 지연 허용 |

### Circuit Breaker

Redis 장애 시 DB로 폴백.

```kotlin
fun <T> getWithFallback(key: String, loader: () -> T): T? {
    return try {
        circuitBreaker.executeSupplier { redisTemplate.opsForValue().get(key) as T? }
    } catch (e: Exception) {
        logger.warn("Redis unavailable: ${e.message}")
        null  // 캐시 없이 진행
    }
}
```

---

## 실무 체크리스트

### 캐싱 대상 선정

| 기준 | 캐싱 적합 | 캐싱 부적합 |
|------|----------|------------|
| 읽기/쓰기 비율 | 읽기 >> 쓰기 | 쓰기 빈번 |
| 일관성 요구 | 약간의 지연 허용 | 실시간 필수 |
| 데이터 크기 | 작음 (~1KB) | 큼 (>100KB) |
| 조회 패턴 | 특정 키로 반복 조회 | 랜덤 조회 |

### 모니터링 지표

```kotlin
@Component
class CacheMetrics(
    private val meterRegistry: MeterRegistry
) {
    fun recordHit(cacheName: String) {
        meterRegistry.counter("cache.hits", "name", cacheName).increment()
    }

    fun recordMiss(cacheName: String) {
        meterRegistry.counter("cache.misses", "name", cacheName).increment()
    }
}

// Hit Rate = hits / (hits + misses)
// 목표: 80% 이상
```

---

## 정리

| 전략 | 사용 시점 | 주의점 |
|------|----------|--------|
| Cache-Aside | 범용적, 첫 구현 | 첫 요청 느림, 일관성 |
| Write-Through | 읽기 많고 일관성 중요 | 쓰기 지연 |
| Write-Behind | 쓰기 많고 지연 허용 | 데이터 유실 위험 |
| Multi-tier | 극한의 성능 필요 | 일관성 관리 복잡 |

캐싱은 성능 최적화의 핵심이지만, 복잡도를 높인다. 먼저 캐싱 없이 최적화(인덱스, 쿼리 튜닝)를 시도하고, 그래도 부족할 때 캐싱을 도입하는 것이 좋다.
