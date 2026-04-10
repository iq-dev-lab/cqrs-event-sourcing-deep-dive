# 이벤트 스토어 설계

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 이벤트 스토어가 일반 관계형 DB와 구조적으로 무엇이 다른가?
- 이벤트 스트림의 Stream ID, Version, EventType, Payload는 각각 어떤 역할인가?
- PostgreSQL로 이벤트 스토어를 직접 구현할 때 핵심 설계 결정은 무엇인가?
- EventStoreDB는 PostgreSQL 구현과 무엇이 다르고 언제 선택해야 하는가?
- 이벤트 스토어 인덱스 설계는 어떻게 해야 하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

이벤트 스토어는 Event Sourcing의 심장이다. 잘못 설계하면 낙관적 잠금 충돌이 감지되지 않고, 이벤트 로드가 느려지며, 스키마 진화가 불가능해진다. PostgreSQL로 직접 구현하면 모든 결정이 내 책임이고, EventStoreDB를 선택하면 많은 것을 위임하지만 새로운 운영 복잡도가 생긴다.

---

## 😱 흔한 실수 (Before)

```
실수 1: 이벤트를 일반 테이블처럼 설계

  CREATE TABLE events (
      id BIGSERIAL PRIMARY KEY,
      aggregate_id VARCHAR(255),
      event_data JSONB,
      created_at TIMESTAMPTZ
  );
  -- 누락: stream 당 version 관리 없음 → 낙관적 잠금 불가
  -- 누락: event_type 없음 → 타입별 조회 불가
  -- 누락: UNIQUE(aggregate_id, version) → 중복 이벤트 허용

실수 2: 이벤트를 UPDATE로 수정

  UPDATE events SET event_data = '...' WHERE id = 100;
  -- 이벤트는 과거 사실 → 수정 불가 원칙 위반
  -- 이벤트를 수정하면 리플레이 결과가 달라짐

실수 3: stream 단위가 아닌 글로벌 sequence만 사용

  CREATE TABLE events (
      global_seq BIGSERIAL PRIMARY KEY,  -- 글로벌 순서만
      aggregate_id VARCHAR(255),
      event_data JSONB
  );
  -- stream(aggregate) 내 버전을 모름 → 낙관적 잠금 불가
  -- "ACC-001의 3번째 이벤트" 조회 비효율
```

---

## ✨ 올바른 접근 (After)

```
이벤트 스토어 핵심 설계:

CREATE TABLE event_store (
    -- 글로벌 순서 (Projection이 새 이벤트 폴링에 사용)
    global_seq  BIGSERIAL    NOT NULL,

    -- Stream 식별자 (Aggregate 단위)
    stream_id   VARCHAR(255) NOT NULL,   -- "account-ACC-001"

    -- Stream 내 버전 (낙관적 잠금)
    version     BIGINT       NOT NULL,   -- 1, 2, 3, ...

    -- 이벤트 타입 (역직렬화, 라우팅)
    event_type  VARCHAR(255) NOT NULL,   -- "MoneyWithdrawn"

    -- 이벤트 데이터
    payload     JSONB        NOT NULL,   -- 이벤트 내용

    -- 메타데이터 (감사, 추적)
    metadata    JSONB,                   -- correlationId, causationId, userId

    -- 발생 시각
    occurred_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    -- 낙관적 잠금: stream 내 version 유일성
    CONSTRAINT uq_stream_version UNIQUE (stream_id, version)
);

-- 인덱스
CREATE INDEX idx_stream_version ON event_store (stream_id, version);
-- stream 이벤트 로드: WHERE stream_id = ? ORDER BY version
-- Projection 폴링: WHERE global_seq > ? ORDER BY global_seq
```

---

## 🔬 내부 동작 원리

### 1. 이벤트 스트림 구조

```
stream_id 설계:
  "{aggregate_type}-{aggregate_id}"
  "account-ACC-001"
  "order-ORD-001"
  "inventory-PROD-001"

  이점:
    aggregate_type을 stream_id에 포함 → 타입별 스트림 조회 가능
    "account-" prefix로 계좌 이벤트만 선택 가능
    Projection이 특정 타입의 이벤트만 구독 가능

version 관리:
  stream 내 이벤트 순서 보장
  낙관적 잠금의 기반

  stream "account-ACC-001":
  ┌───────────┬─────────────────┬──────────────────────────────────┐
  │ version   │ event_type      │ payload                          │
  ├───────────┼─────────────────┼──────────────────────────────────┤
  │ 1         │ AccountOpened   │ { "owner": "USER-42" }           │
  │ 2         │ MoneyDeposited  │ { "amount": 500000 }             │
  │ 3         │ MoneyWithdrawn  │ { "amount": 100000, "by": "..." }│
  └───────────┴─────────────────┴──────────────────────────────────┘

  낙관적 잠금:
    현재 version = 3
    새 이벤트 저장: version = 4 (UNIQUE 제약 보호)
    동시 저장 시도: 같은 version=4 → UNIQUE 위반 → 충돌 감지

global_seq의 역할:
  Projection이 "마지막으로 처리한 이벤트 이후"를 폴링
  SELECT * FROM event_store WHERE global_seq > {lastProcessed} ORDER BY global_seq
  모든 stream의 이벤트를 발생 순서대로 처리 가능
```

### 2. PostgreSQL 이벤트 스토어 구현

```sql
-- 완전한 이벤트 스토어 스키마
CREATE TABLE event_store (
    global_seq  BIGSERIAL    NOT NULL,
    stream_id   VARCHAR(255) NOT NULL,
    version     BIGINT       NOT NULL,
    event_type  VARCHAR(255) NOT NULL,
    payload     JSONB        NOT NULL,
    metadata    JSONB        DEFAULT '{}',
    occurred_at TIMESTAMPTZ  NOT NULL DEFAULT NOW(),

    CONSTRAINT uq_stream_version UNIQUE (stream_id, version)
);

CREATE INDEX idx_stream_version    ON event_store (stream_id, version);
CREATE INDEX idx_global_seq        ON event_store (global_seq);
CREATE INDEX idx_occurred_at       ON event_store (occurred_at);
CREATE INDEX idx_event_type        ON event_store (event_type);

-- Projection 체크포인트 (어디까지 처리했는지)
CREATE TABLE projection_checkpoints (
    projection_name VARCHAR(255) PRIMARY KEY,
    last_global_seq BIGINT NOT NULL DEFAULT 0,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

### 3. EventStoreDB vs PostgreSQL 비교

```
PostgreSQL 이벤트 스토어:
  장점:
    이미 사용 중인 DB 활용 (추가 인프라 없음)
    JSONB 인덱싱, Full-Text Search 활용
    팀이 SQL에 익숙
    트랜잭션 지원 (이벤트 저장 + 다른 데이터 원자적 처리)

  단점:
    이벤트 스트림 구독 구현 필요 (LISTEN/NOTIFY or 폴링)
    대규모 이벤트 처리 시 별도 파티셔닝 필요
    이벤트 만료/보존 정책 직접 구현

EventStoreDB:
  장점:
    이벤트 스트림 네이티브 지원 (gRPC 기반 구독)
    글로벌 이벤트 스트림 내장
    웹 UI로 스트림 조회
    카테고리별 스트림 ($ce-account: 모든 계좌 이벤트)

  단점:
    추가 인프라 운영 필요
    팀 학습 곡선
    비즈니스 데이터와 트랜잭션 묶기 어려움

선택 기준:
  단일 서비스 + 이미 PostgreSQL 사용 → PostgreSQL 직접 구현
  복잡한 이벤트 구독 + 여러 서비스 → EventStoreDB
  Axon Framework 사용 → Axon Server (이벤트 스토어 내장)
```

---

## 💻 실전 코드

```java
// ✅ PostgreSQL 기반 이벤트 스토어 구현
@Repository
public class PostgreSqlEventStore {

    @Transactional
    public void appendEvents(String streamId, long expectedVersion,
                              List<StoredEvent> events) {
        // 현재 최신 version 확인
        Long currentVersion = jdbcTemplate.queryForObject(
            "SELECT MAX(version) FROM event_store WHERE stream_id = ?",
            Long.class, streamId);

        long current = currentVersion != null ? currentVersion : 0L;

        if (current != expectedVersion) {
            throw new OptimisticConcurrencyException(
                String.format("stream=%s expected=%d actual=%d",
                    streamId, expectedVersion, current));
        }

        // 이벤트 일괄 삽입
        long nextVersion = expectedVersion;
        for (StoredEvent event : events) {
            nextVersion++;
            try {
                jdbcTemplate.update("""
                    INSERT INTO event_store
                        (stream_id, version, event_type, payload, metadata, occurred_at)
                    VALUES (?, ?, ?, ?::jsonb, ?::jsonb, ?)
                    """,
                    streamId,
                    nextVersion,
                    event.eventType(),
                    objectMapper.writeValueAsString(event.payload()),
                    objectMapper.writeValueAsString(event.metadata()),
                    Instant.now()
                );
            } catch (DuplicateKeyException e) {
                throw new OptimisticConcurrencyException(
                    "동시 이벤트 저장 충돌: stream=" + streamId);
            }
        }
    }

    public List<StoredEvent> loadEvents(String streamId) {
        return jdbcTemplate.query("""
            SELECT version, event_type, payload, metadata, occurred_at
            FROM event_store
            WHERE stream_id = ?
            ORDER BY version ASC
            """,
            (rs, rowNum) -> new StoredEvent(
                rs.getString("event_type"),
                parseJson(rs.getString("payload")),
                parseJson(rs.getString("metadata")),
                rs.getTimestamp("occurred_at").toInstant()
            ),
            streamId
        );
    }

    public List<StoredEvent> loadEventsSince(long lastGlobalSeq, int limit) {
        return jdbcTemplate.query("""
            SELECT global_seq, stream_id, event_type, payload, occurred_at
            FROM event_store
            WHERE global_seq > ?
            ORDER BY global_seq ASC
            LIMIT ?
            """,
            (rs, rowNum) -> new StoredEvent(
                rs.getString("event_type"),
                parseJson(rs.getString("payload")),
                Map.of(),
                rs.getTimestamp("occurred_at").toInstant()
            ),
            lastGlobalSeq, limit
        );
    }
}
```

---

## 📊 패턴 비교

```
이벤트 스토어 구현 옵션 비교:

┌──────────────────┬────────────────┬────────────────┬────────────────┐
│ 항목             │ PostgreSQL      │ EventStoreDB   │ Axon Server    │
├──────────────────┼────────────────┼────────────────┼────────────────┤
│ 추가 인프라      │ 없음           │ 필요           │ 필요           │
├──────────────────┼────────────────┼────────────────┼────────────────┤
│ 이벤트 구독      │ 폴링/NOTIFY    │ 네이티브       │ Axon 내장      │
├──────────────────┼────────────────┼────────────────┼────────────────┤
│ 트랜잭션 통합    │ 쉬움           │ 어려움         │ 어려움         │
├──────────────────┼────────────────┼────────────────┼────────────────┤
│ 운영 복잡도      │ 낮음           │ 중간           │ 중간           │
├──────────────────┼────────────────┼────────────────┼────────────────┤
│ Spring 통합      │ 직접 구현      │ SDK 있음       │ 자동           │
└──────────────────┴────────────────┴────────────────┴────────────────┘
```

---

## ⚖️ 트레이드오프

```
이벤트 스토어 크기 관리:
  이벤트는 삭제하지 않음 → 시간이 지날수록 테이블 증가
  해결: 파티셔닝 (occurred_at 기준 월별 파티션)
        오래된 이벤트 콜드 스토리지 이동 (S3 + Parquet)
        스냅샷으로 오래된 이벤트 로드 빈도 감소

JSONB vs Binary 직렬화:
  JSONB: 인덱싱 가능, 가독성, 스키마 유연
         크기 크고 파싱 비용 있음
  Binary (Avro/Protobuf): 작고 빠름
         스키마 레지스트리 필요, 가독성 없음
  → 초기: JSONB (유연성), 대규모: Binary 전환 고려
```

---

## 📌 핵심 정리

```
이벤트 스토어 설계 핵심:

필수 필드:
  stream_id: Aggregate 단위 식별
  version: stream 내 순서 + 낙관적 잠금
  event_type: 역직렬화 타입 식별
  payload: 이벤트 데이터 (JSONB)
  global_seq: Projection 폴링용 전역 순서

핵심 제약:
  UNIQUE(stream_id, version): 낙관적 잠금 + 순서 보장
  Append Only: INSERT만, UPDATE/DELETE 없음

인덱스:
  (stream_id, version): Aggregate 로드
  (global_seq): Projection 폴링
  (event_type): 타입별 조회

구현 선택:
  PostgreSQL: 단순, 트랜잭션 통합 쉬움
  EventStoreDB: 구독 네이티브, 별도 인프라
```

---

## 🤔 생각해볼 문제

**Q1.** 이벤트 스토어가 매우 커져서 특정 stream의 이벤트를 로드하는 데 수 초가 걸린다. 인덱스 외에 어떤 방법으로 해결할 수 있는가?

<details>
<summary>해설 보기</summary>

스냅샷 패턴이 가장 직접적인 해결책입니다. 스냅샷을 저장하면 최신 스냅샷 + 그 이후 이벤트만 로드하면 됩니다. 다음 문서(04-snapshot-pattern.md)에서 자세히 다룹니다.

인덱스 최적화도 중요합니다. `(stream_id, version)` 복합 인덱스가 없으면 순차 스캔이 발생합니다. PostgreSQL 파티셔닝(stream_id 기준 해시 파티셔닝 또는 occurred_at 기준 범위 파티셔닝)으로 각 파티션의 크기를 줄일 수 있습니다.

이벤트 아카이빙도 고려할 수 있습니다. 수년 이상 된 이벤트는 S3 같은 콜드 스토리지로 이동하고, DB에는 최근 이벤트만 유지합니다. 스냅샷이 있다면 아카이빙된 이벤트를 로드할 필요가 없습니다.

</details>

---

**Q2.** `global_seq`가 BIGSERIAL로 자동 증가할 때, 이벤트 삽입 순서와 global_seq 순서가 다를 수 있는가?

<details>
<summary>해설 보기</summary>

예, 다를 수 있습니다. 이것이 `global_seq`를 Projection 폴링에 사용할 때 중요한 함정입니다.

트랜잭션 A가 global_seq=100을 획득하고, 트랜잭션 B가 global_seq=101을 획득했을 때, B가 먼저 커밋하면 DB에는 101만 보이고 100은 아직 없습니다. Projection이 101을 처리했다가 100이 나중에 나타나면 순서가 어긋납니다.

해결 방법이 두 가지입니다. 첫째, 짧은 지연 후 폴링합니다. 트랜잭션 커밋에 걸리는 최대 시간(예: 100ms) 이후의 이벤트만 처리합니다. `WHERE global_seq > ? AND occurred_at < NOW() - INTERVAL '100ms'`처럼 작성합니다.

둘째, PostgreSQL `pg_current_wal_lsn()`을 활용해 WAL 수준에서 정렬하거나, 이벤트 발행 후 커밋 여부를 확인하는 방식을 씁니다.

EventStoreDB는 이 문제를 내부적으로 해결해 제공합니다.

</details>

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[이전: Event Sourcing의 핵심 아이디어 ⬅️](./01-event-sourcing-core.md)** | **[다음: Aggregate 재구성 ➡️](./03-aggregate-reconstitution.md)**

</div>
