# 이벤트 저장과 발행의 원자성

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 이벤트 스토어 저장과 Kafka 발행을 왜 원자적으로 처리해야 하는가?
- 저장은 됐지만 발행이 안 된 경우와 발행은 됐지만 저장이 안 된 경우 각각 무슨 문제가 생기는가?
- Transactional Outbox 패턴은 어떻게 구현하는가?
- CDC(Change Data Capture) 방식과 Outbox 방식의 차이와 선택 기준은 무엇인가?
- Exactly-Once 발행을 보장할 수 있는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

이벤트 스토어와 Kafka는 서로 다른 리소스다. DB 트랜잭션은 이벤트 스토어에서만 작동한다. 이 두 저장소에 동시에 원자적으로 쓰는 것은 XA 트랜잭션(2PC) 없이는 불가능하다. XA 트랜잭션은 성능이 나쁘고 복잡하다. Transactional Outbox 패턴이 XA 없이 원자성에 가까운 보장을 제공하는 이유를 이해해야 한다.

---

## 😱 흔한 실수 (Before — 비원자적 발행)

```
실수 1: 이벤트 스토어 저장 후 Kafka 발행 (비원자적)

  @Transactional
  public void handle(WithdrawMoneyCommand cmd) {
      account.withdraw(amount);
      eventStore.save(account.getPendingEvents());
      // COMMIT ← 여기서 성공
      kafkaTemplate.send("account-events", event); // 여기서 실패?
      // 저장은 됐지만 Kafka에 없음 → Projection 누락
  }

실수 2: Kafka 발행 후 이벤트 스토어 저장 (더 나쁨)

  kafkaTemplate.send("account-events", event); // 발행 성공
  eventStore.save(events);                      // 저장 실패
  // Kafka에는 이벤트 있음 → Projection이 읽기 모델 업데이트
  // 이벤트 스토어에는 없음 → Aggregate 재구성 시 이 이벤트 없음
  // → 불일치: 읽기 모델과 쓰기 모델이 다른 상태
```

---

## ✨ 올바른 접근 (After — Transactional Outbox)

```
Transactional Outbox 패턴:

  이벤트 스토어 저장
    + Outbox 테이블에 이벤트 저장
    = 단일 DB 트랜잭션 (원자적)

  별도 OutboxPublisher 프로세스:
    Outbox 테이블 폴링 → Kafka 발행 → published=true

  이점:
    이벤트 스토어 저장 실패 → Outbox도 실패 (트랜잭션 롤백)
    이벤트 스토어 저장 성공 → Outbox도 성공
    Kafka 발행 실패 → Outbox에 published=false 남음 → 재시도
    → 결국 이벤트 스토어와 Kafka 일치 보장
```

---

## 🔬 내부 동작 원리

### 1. 원자성 문제의 두 가지 시나리오

```
시나리오 A — 저장 성공, 발행 실패:

  이벤트 스토어: MoneyWithdrawn 이벤트 저장됨 ✅
  Kafka: 발행 실패 (Kafka 브로커 다운) ❌

  결과:
    다음 Command 처리: Aggregate 재구성 시 MoneyWithdrawn 포함
    → 정확한 현재 상태 계산 (쓰기 모델 정상)

    Projection: MoneyWithdrawn 이벤트 받지 못함
    → account_summary.balance 업데이트 안 됨
    → 읽기 모델 불일치 (구버전 잔고 표시)

  시간이 지나면:
    Kafka가 복구되어도 이 이벤트는 이미 누락됨
    → 읽기 모델은 영구적으로 이 이벤트 반영 못함
    → 데이터 불일치 지속

시나리오 B — 발행 성공, 저장 실패:

  Kafka: MoneyWithdrawn 발행됨 ✅
  이벤트 스토어: 저장 실패 (DB 오류, 롤백) ❌

  결과:
    Projection: MoneyWithdrawn 처리 → account_summary.balance 차감 ✅
    다음 Aggregate 로드: MoneyWithdrawn 없음 (이벤트 스토어에 없음)
    → balance = 이전 값 (감소 안 됨)

    쓰기 모델 balance ≠ 읽기 모델 balance
    → 이후 출금 시 실제보다 많은 잔고로 판단
    → 초과 출금 가능성!

  → 시나리오 B가 더 심각 (데이터 불일치 + 금전 오류)
  → Kafka 발행은 항상 이벤트 스토어 저장 후에
```

### 2. Transactional Outbox 구현

```
Outbox 테이블 구조:

CREATE TABLE outbox (
    id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    event_id     UUID NOT NULL,
    stream_id    VARCHAR(255) NOT NULL,
    event_type   VARCHAR(255) NOT NULL,
    payload      JSONB NOT NULL,
    created_at   TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    published_at TIMESTAMPTZ,          -- null이면 미발행
    published    BOOLEAN DEFAULT FALSE,
    retry_count  INT DEFAULT 0
);

CREATE INDEX idx_outbox_unpublished ON outbox (published, created_at)
WHERE published = false;

처리 흐름:

  CommandHandler (같은 트랜잭션):
    BEGIN;
      INSERT INTO event_store (...)        -- 이벤트 저장
      INSERT INTO outbox (...)             -- 발행 예약
    COMMIT;

  OutboxPublisher (별도 프로세스, 500ms 간격):
    BEGIN;
      SELECT * FROM outbox WHERE published = false
        ORDER BY created_at LIMIT 100
        FOR UPDATE SKIP LOCKED;             -- 동시 발행 방지

      for each event:
        kafkaTemplate.send(...).get();      -- 동기 발행
        UPDATE outbox SET published = true, published_at = NOW()
          WHERE id = ?;
    COMMIT;
```

### 3. CDC 방식 — Debezium

```
CDC (Change Data Capture):
  DB의 WAL(Write Ahead Log)을 직접 읽어 변경 이벤트를 Kafka로 발행
  Outbox 테이블 없이도 원자성 보장

Debezium + PostgreSQL:
  PostgreSQL WAL → Debezium Connector → Kafka

  설정:
    {"connector.class": "io.debezium.connector.postgresql.PostgresConnector",
     "database.server.name": "account-db",
     "table.include.list": "public.event_store",  -- 이벤트 스토어 감시
     "publication.name": "dbz_publication"}

  이벤트 스토어에 INSERT 발생
  → WAL에 변경 기록
  → Debezium이 WAL 읽어 Kafka 발행
  → Kafka Topic: "account-db.public.event_store"

  장점:
    애플리케이션 코드 변경 없음
    Outbox 테이블 불필요
    DB 트랜잭션과 자동으로 동기화

  단점:
    Debezium 커넥터 운영 복잡도
    Kafka Connect 클러스터 필요
    WAL 보존 기간 설정 필요 (pg_wal_level = logical)
    데이터 증가 (WAL 크기 증가)

CDC vs Outbox 선택 기준:

  Outbox 선택:
    팀이 Kafka Connect 운영 경험 없음
    단순한 이벤트 발행 (대용량 아님)
    이미 DB 폴링이 수용 가능한 부하

  CDC 선택:
    애플리케이션 코드를 건드리지 않고 이벤트 발행
    기존 DB에 이미 Debezium 환경 있음
    높은 처리량 (폴링 오버헤드 없음)
    이벤트 스토어 외 다른 테이블도 CDC 대상
```

### 4. Exactly-Once 발행 가능한가

```
현실:

Kafka는 기본적으로 At-Least-Once 전달:
  네트워크 오류 → 재시도 → 중복 발행 가능

Exactly-Once (Kafka Transactions):
  Kafka Producer 트랜잭션 + 이벤트 스토어 저장을 결합
  → 이벤트 스토어 + Kafka에 정확히 1번 저장

  제약: Kafka Streams 환경에서 더 쉬움
        일반 Producer에서 Exactly-Once는 복잡

실무 권장 접근:
  At-Least-Once 발행 + 멱등 Projection (Idempotent Consumer)
  → Kafka가 이벤트를 2번 발행해도 Projection이 멱등하게 처리
  → 결과적으로 Exactly-Once 효과

  멱등 Projection:
    processed_events 테이블로 중복 감지
    Upsert로 중복 업데이트 방지

결론:
  Exactly-Once는 필요 없음
  At-Least-Once + 멱등 Projection = 동등한 효과
```

---

## 💻 실전 코드

```java
// ✅ Transactional Outbox 구현
@Repository
public class OutboxRepository {

    @Transactional  // CommandHandler의 트랜잭션에 참여
    public void save(List<DomainEvent> events, String streamId) {
        events.forEach(event -> jdbcTemplate.update("""
            INSERT INTO outbox (event_id, stream_id, event_type, payload)
            VALUES (?, ?, ?, ?::jsonb)
            """,
            event.eventId(), streamId,
            event.getClass().getSimpleName(),
            objectMapper.writeValueAsString(event)
        ));
    }
}

// ✅ CommandHandler — EventStore + Outbox 같은 트랜잭션
@CommandHandler
@Transactional
public void handle(WithdrawMoneyCommand cmd) {
    Account account = accountRepository.findById(cmd.accountId()).orElseThrow();
    account.withdraw(Money.of(cmd.amount()), cmd.requestedBy());

    // 이벤트 스토어 저장
    eventStore.appendEvents("account-" + cmd.accountId(),
        account.getVersion() - account.getPendingEvents().size(),
        account.getPendingEvents());

    // Outbox 저장 (같은 트랜잭션)
    outboxRepository.save(account.pullPendingEvents(), "account-" + cmd.accountId());
}

// ✅ OutboxPublisher — 주기적 폴링 + Kafka 발행
@Component
public class OutboxPublisher {

    @Scheduled(fixedDelay = 500)
    @Transactional
    public void publishPendingEvents() {
        // FOR UPDATE SKIP LOCKED: 다중 인스턴스에서 중복 발행 방지
        List<OutboxEvent> pending = jdbcTemplate.query("""
            SELECT id, event_id, stream_id, event_type, payload
            FROM outbox
            WHERE published = false
            ORDER BY created_at ASC
            LIMIT 100
            FOR UPDATE SKIP LOCKED
            """, outboxEventRowMapper);

        if (pending.isEmpty()) return;

        for (OutboxEvent event : pending) {
            try {
                // 동기 발행 (발행 확인 후 published 업데이트)
                kafkaTemplate.send("account-events", event.streamId(),
                    event.payload()).get(5, SECONDS);

                jdbcTemplate.update(
                    "UPDATE outbox SET published = true, published_at = NOW() WHERE id = ?",
                    event.id()
                );
                metrics.recordPublished(event.eventType());

            } catch (Exception e) {
                log.warn("Kafka 발행 실패 (재시도 예정): eventId={} error={}",
                    event.eventId(), e.getMessage());
                jdbcTemplate.update(
                    "UPDATE outbox SET retry_count = retry_count + 1 WHERE id = ?",
                    event.id()
                );
                // 트랜잭션 롤백하지 않음 → 다음 폴링에서 재시도
            }
        }
    }
}

// ✅ Debezium CDC 설정 (대안)
// application.yml에 Kafka Connect REST API 호출
// 또는 Debezium Embedded 방식
@Configuration
public class DebeziumConfig {

    @Bean
    public EmbeddedEngineChangeConsumer eventStoreChangeConsumer() {
        return records -> records.forEach(record -> {
            Struct value = (Struct) record.value();
            if ("c".equals(value.getString("op"))) { // CREATE only
                String eventType = value.getStruct("after").getString("event_type");
                String payload   = value.getStruct("after").getString("payload");
                kafkaTemplate.send("account-events", eventType, payload);
            }
        });
    }
}
```

---

## 📊 패턴 비교

```
원자성 보장 방식 비교:

┌──────────────────┬───────────────┬──────────────┬──────────────────┐
│ 방식             │ 원자성        │ 운영 복잡도  │ 처리량           │
├──────────────────┼───────────────┼──────────────┼──────────────────┤
│ 직접 Kafka 발행  │ 없음          │ 낮음         │ 높음             │
├──────────────────┼───────────────┼──────────────┼──────────────────┤
│ Outbox 패턴      │ 보장(DB 트랜)  │ 중간         │ 중간(폴링 지연)  │
├──────────────────┼───────────────┼──────────────┼──────────────────┤
│ CDC (Debezium)   │ 보장(WAL)     │ 높음         │ 높음             │
├──────────────────┼───────────────┼──────────────┼──────────────────┤
│ XA 트랜잭션      │ 완전 보장     │ 매우 높음    │ 낮음             │
└──────────────────┴───────────────┴──────────────┴──────────────────┘
권장: 소규모 → Outbox / 대규모 → CDC
```

---

## ⚖️ 트레이드오프

```
Outbox 패턴 비용:
  추가 테이블 관리
  폴링으로 인한 발행 지연 (최대 500ms)
  대용량 이벤트 시 폴링 부하

편익:
  구현 단순 (DB 트랜잭션만으로 원자성)
  Kafka 없이도 테스트 가능
  재시도 로직 직접 제어
```

---

## 📌 핵심 정리

```
이벤트 저장-발행 원자성 핵심:

문제:
  이벤트 스토어(DB) + Kafka = 다른 리소스
  → 하나 성공, 하나 실패 = 불일치

해결 — Transactional Outbox:
  이벤트 스토어 + Outbox = 같은 DB 트랜잭션
  OutboxPublisher = 별도 프로세스 (폴링 → Kafka 발행)
  결과: 이벤트 스토어에 있으면 결국 Kafka에도 발행됨

대안 — Debezium CDC:
  이벤트 스토어 WAL → Debezium → Kafka
  코드 변경 없음, 높은 처리량

At-Least-Once + 멱등 Projection:
  Exactly-Once 불필요
  중복 발행 → 멱등 처리로 동일 결과 보장
```

---

## 🤔 생각해볼 문제

**Q1.** Outbox Publisher가 여러 인스턴스로 실행될 때 같은 이벤트를 중복으로 Kafka에 발행하는 것을 어떻게 방지하는가?

<details>
<summary>해설 보기</summary>

`SELECT FOR UPDATE SKIP LOCKED`가 핵심입니다. 한 인스턴스가 Outbox 레코드를 SELECT FOR UPDATE로 잠그면, 다른 인스턴스는 SKIP LOCKED 덕분에 잠긴 레코드를 건너뛰고 다른 레코드를 처리합니다. 같은 레코드를 두 인스턴스가 동시에 처리하지 않습니다.

처리 완료 후 published=true로 업데이트하고 커밋하면 잠금이 해제됩니다. 이 레코드는 이후 폴링에서 published=false 조건으로 조회되지 않습니다.

단, SKIP LOCKED는 PostgreSQL 9.5+, MySQL 8.0+에서 지원합니다. 지원하지 않는 DB라면 인스턴스 단위로 처리 범위를 분할하거나(stream_id 해시 기반), 단일 인스턴스로만 실행하는 방법을 씁니다.

</details>

---

**Q2.** Outbox 테이블이 무한히 커지지 않도록 어떻게 관리하는가?

<details>
<summary>해설 보기</summary>

발행 완료된 레코드를 주기적으로 삭제하거나 아카이빙합니다.

주기적 삭제 방법으로는 스케줄 잡으로 `DELETE FROM outbox WHERE published = true AND published_at < NOW() - INTERVAL '7 days'`를 실행합니다. 7일 이상 된 발행 완료 레코드는 삭제합니다.

대량 삭제 시 성능 이슈가 있다면 파티셔닝을 활용합니다. `outbox` 테이블을 created_at 기준으로 월별 파티셔닝하면, 오래된 파티션을 DROP으로 즉시 삭제할 수 있어 DELETE보다 훨씬 빠릅니다.

재처리나 감사 목적으로 발행 완료 이벤트를 보존해야 한다면, S3 같은 저렴한 저장소에 아카이빙 후 DB에서 삭제합니다.

발행 완료 레코드 보존 기간은 비즈니스 요구사항(감사, 재처리 필요 기간)을 기준으로 결정합니다.

</details>

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[이전: 완전한 흐름 ⬅️](./01-complete-flow.md)** | **[다음: DDD와의 통합 ➡️](./03-ddd-integration.md)**

</div>
