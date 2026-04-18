---
title: "차량 텔레매틱스 데이터 파이프라인에서 MongoDB 활용과 한계"
date: 2023-09-20
draft: true
tags: ["MongoDB", "Time-Series", "Vehicle", "ClickHouse"]
categories: ["Architecture"]
summary: "차량 엣지에서 올라오는 텔레매틱스 데이터를 MongoDB로 처리할 때의 활용법, 성능 한계, 그리고 대안 아키텍처를 분석한다."
---

## 차량 텔레매틱스 데이터 특성

차량에서 올라오는 데이터는 일반적인 CRUD와 다르다.

```
특성:
- 빈도: 차량당 0.5~2초 간격
- 볼륨: 10,000대 × 2Hz = 20,000 writes/sec
- 구조: GPS, 속도, 엔진 상태, 센서 값 등 다양한 필드
- 카디널리티: 차량 수 × 센서 종류
- 패턴: Write 집중, 시간 범위 쿼리
```

---

## MongoDB Time Series Collections

MongoDB 5.0+에서 도입된 Time Series Collections를 활용할 수 있다.

### 기본 설정

```javascript
db.createCollection("vehicle_telemetry", {
  timeseries: {
    timeField: "timestamp",
    metaField: "vehicleId",
    granularity: "seconds"
  },
  expireAfterSeconds: 86400 * 30  // 30일 후 자동 삭제
})
```

### 데이터 구조

```javascript
// 단일 텔레매틱스 이벤트
{
  timestamp: ISODate("2024-01-15T10:30:00Z"),
  vehicleId: "v-12345",
  location: {
    type: "Point",
    coordinates: [127.0276, 37.4979]
  },
  speed: 65.5,
  rpm: 2500,
  fuelLevel: 0.45,
  engineTemp: 92,
  diagnostics: {
    batteryVoltage: 12.6,
    tirePressure: [32, 32, 31, 32]
  }
}
```

### 장점

- **자동 버킷팅**: 시간 + metaField 기준으로 문서 그룹화
- **압축 효율**: 유사 데이터 압축으로 저장 공간 절약
- **쿼리 최적화**: 클러스터 인덱스로 시간 범위 쿼리 가속
- **자동 TTL**: expireAfterSeconds로 오래된 데이터 자동 삭제

---

## MongoDB의 한계점

### 1. Write 성능 한계

| DB | 초당 Insert (노드당) |
|----|---------------------|
| MongoDB | 100,000 ~ 300,000 |
| ClickHouse | 500,000 ~ 1,000,000 |
| TimescaleDB | 고카디널리티에서 3.5배 빠름 |

```
문제 시나리오:
- 10,000대 × 2 센서 × 초당 2회 = 40,000 writes/sec → OK
- 100,000대 확장 시 = 400,000 writes/sec → 한계 도달
```

단일 Primary 노드의 한계다. Sharding으로 확장 가능하지만 복잡도가 증가한다.

### 2. 버킷팅 문제

Time Series Collections는 내부적으로 버킷 단위로 저장한다.

```
granularity: "seconds" → 버킷 크기 1시간
granularity: "minutes" → 버킷 크기 24시간
granularity: "hours" → 버킷 크기 30일
```

**문제 상황:**

```
센서가 시간당 2-3개만 기록하는 경우:
→ 희소 버킷 생성 (1시간 버킷에 3개만)
→ 버킷 수 폭발
→ 저장 효율 급감, 쿼리 성능 저하
```

**metaField 폭발:**

```javascript
// ❌ 너무 세분화된 metaField
{ vehicleId: "v1", sensorId: "s1", location: "seoul" }
// → 조합 폭발 → 버킷 과다 생성

// ✅ 적절한 metaField
{ vehicleId: "v1" }
// → 차량당 하나의 버킷 스트림
```

### 3. Sharding 제약

Time Series Collections의 Shard Key 제약이 있다.

```yaml
# 허용되는 Shard Key
- metaField만
- timeField만 (권장하지 않음)
- metaField + timeField 조합

# 불가능
- _id 필드 포함
- 기타 필드
```

```
문제:
- timeField만 사용 → 모든 write가 최신 청크로 집중 (핫스팟)
- metaField만 사용 → 특정 차량 데이터가 많으면 불균형

권장:
- { vehicleId: 1, timestamp: 1 } 복합 shard key
- MongoDB 8.0부터 timeField 단독 사용 deprecated
```

### 4. Update/Delete 제약

Time Series Collections는 수정에 제한이 있다.

```javascript
// ✅ 허용 (MongoDB 5.1+)
db.telemetry.updateMany(
  { vehicleId: "v1" },  // metaField만 매칭 가능
  { $set: { "metadata.processed": true } }  // metaField만 수정 가능
)

// ❌ 불가능
db.telemetry.updateMany(
  { timestamp: { $lt: ISODate("2024-01-01") } },  // timeField 매칭
  { $set: { speed: 0 } }  // 일반 필드 수정
)
```

잘못된 데이터 보정이 어렵다.

### 5. 분석 쿼리 성능

벤치마크 결과 (IIoT 시나리오):

| 쿼리 유형 | MongoDB vs TimescaleDB |
|----------|----------------------|
| 단순 조회 | 비슷 |
| 복잡한 집계 | 3.4x ~ 71x 느림 |
| JOIN 연산 | 지원 제한 |
| Window 함수 | 제한적 |

```javascript
// MongoDB에서 느린 쿼리 예시
db.telemetry.aggregate([
  { $match: { timestamp: { $gte: startDate, $lt: endDate } } },
  { $group: {
      _id: {
        vehicleId: "$vehicleId",
        hour: { $hour: "$timestamp" }
      },
      avgSpeed: { $avg: "$speed" },
      maxRpm: { $max: "$rpm" }
  }},
  { $sort: { "_id.hour": 1 } }
])
// 차량 10,000대, 30일 데이터 → 수십 초
```

---

## 전용 시계열 DB 비교

| 항목 | MongoDB | TimescaleDB | ClickHouse |
|------|---------|-------------|------------|
| **쿼리 언어** | MQL | SQL | SQL |
| **Write 성능** | 10~30만/s | 높음 | 50~100만/s |
| **압축률** | 보통 | 우수 | 매우 우수 |
| **복잡한 쿼리** | 느림 | 빠름 | 매우 빠름 |
| **JOIN** | 제한적 | 완전 지원 | 제한적 |
| **유연한 스키마** | ✅ | ❌ | ❌ |
| **운영 복잡도** | 보통 | 낮음 | 높음 |

---

## 아키텍처 패턴

### 패턴 1: MongoDB 단독 (소규모)

```
[Vehicle Edge] → [Kafka] → [MongoDB Time Series]
                                   ↓
                              [Application]
```

**적합한 경우:**
- 차량 1,000대 이하
- 유연한 스키마 필요 (차량 모델별 다른 센서)
- 분석 요구 적음
- 빠른 개발 필요

### 패턴 2: MongoDB + ClickHouse (중규모)

```
[Vehicle Edge] → [Kafka] ─┬→ [MongoDB] ← Operational Queries
                          │      │
                          │   [Debezium CDC]
                          │      ↓
                          └→ [ClickHouse] ← Analytics
```

**역할 분리:**
- MongoDB: 원시 이벤트, 유연한 스키마, 최근 데이터 조회
- ClickHouse: 집계, 대시보드, 장기 분석

```kotlin
// Operational Query (MongoDB)
fun getRecentTelemetry(vehicleId: String, minutes: Int): List<Telemetry> {
    return mongoTemplate.find(
        Query.query(
            Criteria.where("vehicleId").`is`(vehicleId)
                .and("timestamp").gte(Instant.now().minusSeconds(minutes * 60L))
        ).with(Sort.by(Sort.Direction.DESC, "timestamp")).limit(100),
        Telemetry::class.java
    )
}

// Analytics Query (ClickHouse)
fun getDailyStats(startDate: LocalDate, endDate: LocalDate): List<DailyStats> {
    return clickHouseJdbcTemplate.query("""
        SELECT
            toDate(timestamp) as date,
            vehicleId,
            avg(speed) as avgSpeed,
            max(speed) as maxSpeed,
            count() as dataPoints
        FROM telemetry
        WHERE timestamp >= ? AND timestamp < ?
        GROUP BY date, vehicleId
        ORDER BY date, vehicleId
    """, startDate, endDate)
}
```

### 패턴 3: TimescaleDB 단독 (분석 중심)

```
[Vehicle Edge] → [Kafka] → [TimescaleDB]
                                 ↓
                           [Grafana / BI Tools]
```

**적합한 경우:**
- SQL 기반 분석 필요
- JOIN이 필수 (차량 마스터 + 텔레매틱스)
- PostgreSQL 생태계 활용
- 고카디널리티 (차량 수 많음)

```sql
-- TimescaleDB: Continuous Aggregates로 실시간 집계
CREATE MATERIALIZED VIEW hourly_vehicle_stats
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 hour', timestamp) AS hour,
    vehicle_id,
    avg(speed) AS avg_speed,
    max(speed) AS max_speed,
    avg(fuel_level) AS avg_fuel
FROM telemetry
GROUP BY hour, vehicle_id;

-- 자동 새로고침 정책
SELECT add_continuous_aggregate_policy('hourly_vehicle_stats',
    start_offset => INTERVAL '3 hours',
    end_offset => INTERVAL '1 hour',
    schedule_interval => INTERVAL '1 hour'
);
```

### 패턴 4: 하이브리드 (대규모)

```
[Vehicle Edge]
      ↓
[Kafka] ────┬──────────────┬──────────────┐
            ↓              ↓              ↓
     [MongoDB Atlas]  [TimescaleDB]  [ClickHouse]
     (Hot: 7일)       (Warm: 90일)   (Cold: 분석)
            │              │              │
            └─────[Data Catalog]──────────┘
```

**데이터 계층화:**
- Hot (MongoDB): 최근 7일, 빠른 조회, 유연한 스키마
- Warm (TimescaleDB): 90일, SQL 분석, 압축
- Cold (ClickHouse/S3): 장기 보관, 대규모 분석

---

## MongoDB 사용 시 Best Practices

### 1. Granularity 최적화

```javascript
// 데이터 빈도에 따라 선택
db.createCollection("telemetry", {
  timeseries: {
    timeField: "ts",
    metaField: "vehicleId",
    // 초당 1회 이상 → "seconds" (버킷 1시간)
    // 분당 1회 정도 → "minutes" (버킷 24시간)
    // 시간당 1회 → "hours" (버킷 30일)
    granularity: "seconds"
  }
})
```

### 2. 배치 Insert + 정렬

```kotlin
// ❌ 개별 Insert
events.forEach { collection.insertOne(it) }

// ✅ 배치 Insert + metaField 정렬
val sortedEvents = events.sortedBy { it.vehicleId }
collection.insertMany(sortedEvents, InsertManyOptions().ordered(false))
```

metaField 순서로 정렬하면 같은 버킷에 연속 insert되어 성능이 향상된다.

### 3. 인덱스 전략

```javascript
// 기본 생성되는 인덱스
// { timestamp: 1, vehicleId: 1 } - 클러스터 인덱스

// 추가 인덱스 (필요시)
db.telemetry.createIndex({ "location": "2dsphere" })  // 위치 쿼리
db.telemetry.createIndex({ "vehicleId": 1, "timestamp": -1 })  // 차량별 최신 조회
```

### 4. TTL 활용

```javascript
// 생성 시 설정
db.createCollection("telemetry", {
  timeseries: { ... },
  expireAfterSeconds: 86400 * 30  // 30일
})

// 또는 나중에 수정
db.runCommand({
  collMod: "telemetry",
  expireAfterSeconds: 86400 * 7  // 7일로 변경
})
```

### 5. 압축 활성화

```javascript
// zstd 압축 (가장 효율적)
db.runCommand({
  collMod: "telemetry",
  storageEngine: {
    wiredTiger: { configString: "block_compressor=zstd" }
  }
})
```

---

## 선택 가이드

| 상황 | 권장 |
|------|------|
| 차량 1,000대 이하 | MongoDB 단독 |
| 유연한 스키마 필수 | MongoDB |
| 차량 10,000대+ | MongoDB + ClickHouse |
| SQL 기반 분석 | TimescaleDB |
| 초당 100만+ writes | ClickHouse |
| PostgreSQL 생태계 | TimescaleDB |
| 복잡한 JOIN 필요 | TimescaleDB |

---

## 정리

MongoDB Time Series Collections는 **소중규모 차량 텔레매틱스**에 적합하다.

**장점:**
- 유연한 스키마 (차량 모델별 다른 센서 구조)
- 빠른 개발 (스키마 마이그레이션 불필요)
- Atlas 통합 (관리형 서비스)

**한계:**
- Write 성능 상한 (30만/sec)
- 분석 쿼리 느림
- Update/Delete 제약
- Sharding 복잡도

대규모나 분석 요구가 높다면 **MongoDB(Hot) + ClickHouse(Analytics)** 또는 **TimescaleDB 단독**을 고려한다.

---

## 참고 자료

- [MongoDB Time Series Collections](https://www.mongodb.com/docs/manual/core/timeseries/)
- [Time Series Collection Limitations](https://www.mongodb.com/docs/manual/core/timeseries/timeseries-limitations/)
- [Connected Vehicles with MongoDB Atlas](https://www.mongodb.com/company/blog/innovationconnected-vehicles-accelerate-automotive-innovation-mongodb-atlas-aws)
- [ClickHouse vs MongoDB](https://www.tinybird.co/blog/clickhouse-vs-mongodb)
- [CrateDB IIoT Benchmark](https://cratedb.com/blog/comparing-databases-industrial-iot-use-case)
- [TimescaleDB vs MongoDB](https://stackoverflow.com/questions/63357181/mongodb-vs-timeseries-database-for-timeseries-data)
