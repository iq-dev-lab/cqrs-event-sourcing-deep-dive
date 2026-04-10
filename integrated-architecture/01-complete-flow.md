# 완전한 흐름

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Command 수신부터 Query 응답까지 전체 사이클의 각 단계 책임은 무엇인가?
- 각 단계에서 실패가 발생했을 때 어떻게 처리되는가?
- 성공 경로와 실패 경로의 차이는 어디에서 갈리는가?
- 전체 사이클에서 데이터 일관성은 어느 단계에서 어떻게 보장되는가?
- 분산 환경에서 이 사이클이 어떻게 달라지는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

CQRS + Event Sourcing의 각 요소(Command, Aggregate, Event Store, Kafka, Projection, Read Model, Query)를 개별적으로 이해해도, 이들이 연결된 전체 흐름을 모르면 장애 발생 시 원인 파악이 어렵다. "이체 후 잔고가 업데이트되지 않는다"는 문제가 어느 단계에서 발생했는지 파악하려면 전체 사이클을 머릿속에 명확히 그릴 수 있어야 한다.

---

## 😱 흔한 실수 (Before — 전체 흐름을 파악하지 못할 때)

```
장애 상황: "이체 후 잔고가 안 바뀜"

전체 흐름을 모르는 개발자의 디버깅:
  → 이체 API 응답 코드 확인: 200 OK (정상)
  → "API는 성공했는데 잔고가 왜 안 바뀌지?"
  → Account 테이블 확인: 이벤트 스토어에는 MoneyTransferred 이벤트 있음
  → "DB에는 있는데 왜 안 보이지?"
  → 30분 후 발견: AccountSummaryProjection의 Kafka Consumer가
    5시간 전부터 다운 상태
  → 읽기 모델이 5시간치 이벤트 미반영

사이클을 아는 개발자의 디버깅 (5분):
  1. Kafka Consumer Group lag 확인 → 급증
  2. Projection 상태 확인 → 다운
  3. 원인: Elasticsearch 연결 오류로 Projection 스레드 죽음
  4. 해결: Projection 재시작 + 5시간치 이벤트 순서대로 재처리
```

---

## ✨ 올바른 접근 (After — 전체 사이클 이해)

```
완전한 CQRS + ES 사이클:

[Command 경로]
  Client → HTTP → Controller → CommandBus → CommandHandler
  → Aggregate (재구성 + 검증 + 이벤트 생성) → EventStore (저장)
  → OutboxPublisher → Kafka

[이벤트 전파]
  Kafka → Projection (Consumer) → ReadModel DB

[Query 경로]
  Client → HTTP → Controller → QueryBus → QueryHandler
  → ReadModel DB → Response

각 단계의 실패와 처리:
  Command 검증 실패 → 400/422 (클라이언트 수정 필요)
  Aggregate 낙관적 잠금 충돌 → 409 (재시도)
  이벤트 스토어 저장 실패 → 500 (롤백, 재시도)
  Kafka 발행 실패 → Outbox 테이블에 보관 (나중에 재발행)
  Projection 실패 → DLQ (읽기 모델 지연, 서비스 계속)
  Query 실패 → 500 (읽기 모델 장애)
```

---

## 🔬 내부 동작 원리

### 1. 성공 경로 — 단계별 타임라인

```
이체 Command 전체 사이클:

T=0ms    [1] Client → POST /accounts/ACC-001/transfer
              Body: { toAccountId: "ACC-002", amount: 100000 }

T=1ms    [2] Controller 수신
              TransferMoneyCommand 생성 + 구문 검증
              → idempotencyKey, amount > 0, accountId 형식 확인

T=2ms    [3] CommandBus 미들웨어 체인
              LoggingMiddleware → IdempotencyMiddleware → CommandHandler

T=3ms    [4] CommandHandler 시작
              Aggregate 로드: "account-ACC-001" 스트림 조회

T=8ms    [5] 이벤트 스토어에서 이벤트 로드
              SELECT * FROM event_store WHERE stream_id='account-ACC-001'
              → 15개 이벤트 반환

T=12ms   [6] Aggregate 재구성
              apply(AccountOpened) ... apply(MoneyDeposited) × 14
              현재 balance = 500000, status = ACTIVE

T=13ms   [7] 불변식 검증
              500000 >= 100000 ✓
              status = ACTIVE ✓

T=14ms   [8] 이벤트 생성
              MoneyTransferred { fromAccount: ACC-001, toAccount: ACC-002,
                                 amount: 100000, balanceAfter: 400000 }
              apply(MoneyTransferred) → balance = 400000

T=15ms   [9] 이벤트 스토어 저장
              INSERT INTO event_store (stream_id='account-ACC-001', version=16, ...)
              INSERT INTO outbox (event_id, payload, ...)  ← 같은 트랜잭션
              COMMIT

T=15ms   [10] HTTP 응답
              202 Accepted (처리 완료 = 이벤트 저장 완료)

[비동기 경로]

T=100ms  [11] OutboxPublisher 폴링
              SELECT * FROM outbox WHERE published = false
              → MoneyTransferred 이벤트 발견

T=105ms  [12] Kafka 발행
              kafka.send("account-events", MoneyTransferred)
              → 발행 성공 후 outbox.published = true

T=150ms  [13] AccountSummaryProjection 이벤트 수신
              Kafka Consumer: "account-events" 토픽

T=155ms  [14] 읽기 모델 업데이트
              UPDATE account_summary SET balance = 400000
              WHERE account_id = 'ACC-001'
              (FROM 계좌 업데이트)

              UPDATE account_summary SET balance = (현재 + 100000)
              WHERE account_id = 'ACC-002'
              (TO 계좌 업데이트)

T=200ms  [15] Client → GET /accounts/ACC-001/balance
              → account_summary 조회
              → 400000 반환 (최신 상태)
```

### 2. 실패 경로 — 단계별 처리

```
실패 시나리오별 처리:

실패 1: Aggregate 로드 실패 (DB 연결 오류)
  [5]에서 SQLException 발생
  → CommandHandler 예외
  → HTTP 500 Internal Server Error
  → 이벤트 스토어에 저장 없음 → 데이터 변경 없음
  → 클라이언트: 재시도 가능

실패 2: 낙관적 잠금 충돌
  [9]에서 UNIQUE(stream_id, version) 위반
  → OptimisticConcurrencyException
  → CommandHandler 재시도 (최대 3회)
  → 재시도 후에도 실패 → HTTP 409 Conflict
  → 클라이언트: 최신 상태 확인 후 재시도

실패 3: 이벤트 스토어 저장 성공 + Kafka 발행 실패
  [9] COMMIT 성공 (이벤트 스토어 + Outbox)
  [12] Kafka 발행 실패
  → Outbox에 published=false인 이벤트 남음
  → OutboxPublisher가 주기적으로 재시도
  → 최종적으로 발행 성공 (Eventual Consistency)
  → 클라이언트는 이미 202 받음 (처리 중임을 인지)

실패 4: Projection 처리 실패
  [14]에서 읽기 모델 업데이트 실패
  → DLQ로 이동 후 건너뜀
  → 읽기 모델은 구버전 상태 유지
  → 클라이언트가 조회하면 이전 잔고 표시
  → Projection 복구 후 재처리 → 최신화

실패 5: Query 중 읽기 모델 DB 장애
  [15] account_summary DB 다운
  → HTTP 503 Service Unavailable
  → 쓰기 경로는 영향 없음 (이벤트 스토어 정상)
  → 읽기 모델 DB 복구 후 Projection이 밀린 이벤트 처리
  → 서비스 재개
```

### 3. 관찰 가능성 — 전체 사이클 추적

```
분산 추적 (Distributed Tracing):

모든 단계에서 correlationId 전파:
  Command에 포함: correlationId = UUID
  이벤트 메타데이터: correlationId 포함
  Kafka 헤더: correlationId 전달
  Projection: correlationId로 로깅

  → 하나의 correlationId로 전체 흐름 추적 가능

Jaeger / Zipkin 트레이스 예시:
  POST /transfer                        15ms
  ├── CommandBus.send                   13ms
  │   ├── EventStore.load              8ms
  │   ├── Aggregate.reconstitute       2ms
  │   ├── Account.transfer             1ms
  │   └── EventStore.append            2ms
  └── HTTP 202 Accepted

  [비동기]
  Kafka.consume (AccountSummaryProjection)  5ms
  └── ReadModel.update                     3ms

핵심 메트릭:
  command_processing_duration_ms (p50, p95, p99)
  event_store_load_events_count (이벤트 수별 분포)
  projection_lag_ms (이벤트 발행 ~ 읽기 모델 반영)
  query_duration_ms (읽기 모델 조회 성능)
  dlq_size (DLQ 적재 건수)
```

---

## 💻 실전 코드

```java
// ✅ 완전한 사이클 — 각 단계 코드 요약

// [2] Controller
@PostMapping("/accounts/{id}/transfer")
@ResponseStatus(HttpStatus.ACCEPTED)
public void transfer(@PathVariable String id,
                      @RequestBody @Valid TransferRequest req,
                      @AuthenticationPrincipal UserDetails user) {
    commandBus.send(new TransferMoneyCommand(
        id, req.toAccountId(), req.amount(),
        user.getUsername(), Instant.now(), req.idempotencyKey()
    ));
}

// [4~9] CommandHandler
@CommandHandler
@Transactional
public void handle(TransferMoneyCommand cmd) {
    // [5] 이벤트 로드
    Account from = accountRepository.findById(new AccountId(cmd.fromAccountId())).orElseThrow();
    Account to   = accountRepository.findById(new AccountId(cmd.toAccountId())).orElseThrow();

    // [7] 불변식 검증 + [8] 이벤트 생성
    moneyTransferService.transfer(from, to, Money.of(cmd.amount()));

    // [9] 이벤트 스토어 + Outbox 저장 (트랜잭션)
    accountRepository.save(from);
    accountRepository.save(to);
    // 내부: EventStore INSERT + Outbox INSERT (같은 트랜잭션)
}

// [11~12] OutboxPublisher
@Scheduled(fixedDelay = 500)
@Transactional
public void publishPendingEvents() {
    List<OutboxEvent> pending = outboxRepo.findUnpublished(100);
    pending.forEach(event -> {
        try {
            kafkaTemplate.send("account-events", event.streamId(), event.payload()).get();
            outboxRepo.markPublished(event.id());
        } catch (Exception e) {
            log.warn("Kafka 발행 실패 — 다음 폴링에서 재시도: {}", e.getMessage());
        }
    });
}

// [13~14] Projection
@KafkaListener(topics = "account-events", groupId = "account-summary-projection")
@Transactional
public void handle(ConsumerRecord<String, String> record, Acknowledgment ack) {
    try {
        StoredEvent event = deserialize(record.value());
        if (event.eventType().equals("MoneyTransferred")) {
            MoneyTransferred e = deserialize(event, MoneyTransferred.class);
            updateAccountSummary(e.fromAccountId(), e.balanceAfter());
            updateAccountSummary(e.toAccountId(), e.toBalanceAfter());
        }
        ack.acknowledge();
    } catch (Exception e) {
        sendToDlq(record, e);
        ack.acknowledge(); // 건너뜀
    }
}

// [15] QueryHandler
@QueryHandler
@Transactional(readOnly = true)
public AccountSummaryView handle(GetAccountSummaryQuery query) {
    return accountSummaryRepository.findById(query.accountId())
        .orElseThrow(() -> new AccountNotFoundException(query.accountId()));
}
```

---

## 📊 패턴 비교

```
전체 사이클 단계별 책임:

┌────────────────┬────────────────────────────────────────────────┐
│ 단계           │ 책임                                           │
├────────────────┼────────────────────────────────────────────────┤
│ Controller     │ HTTP 파싱, Command 생성, 권한 확인              │
├────────────────┼────────────────────────────────────────────────┤
│ CommandBus     │ 라우팅, 로깅, 멱등성 체크, 트랜잭션 시작        │
├────────────────┼────────────────────────────────────────────────┤
│ CommandHandler │ Aggregate 로드, 메서드 호출, 저장              │
├────────────────┼────────────────────────────────────────────────┤
│ Aggregate      │ 불변식 검증, 상태 변경, 이벤트 생성             │
├────────────────┼────────────────────────────────────────────────┤
│ EventStore     │ Append Only 저장, 낙관적 잠금 보장             │
├────────────────┼────────────────────────────────────────────────┤
│ Outbox         │ 이벤트 발행 보장 (저장과 원자적)                │
├────────────────┼────────────────────────────────────────────────┤
│ Kafka          │ 이벤트 전파, Consumer Group 격리               │
├────────────────┼────────────────────────────────────────────────┤
│ Projection     │ 이벤트 → 읽기 모델 변환, 멱등 처리             │
├────────────────┼────────────────────────────────────────────────┤
│ QueryHandler   │ 읽기 모델 조회, 응답 매핑                       │
└────────────────┴────────────────────────────────────────────────┘
```

---

## ⚖️ 트레이드오프

```
완전한 사이클의 복잡도 비용:
  9개 이상의 단계
  각 단계 독립 모니터링 필요
  장애 발생 위치 파악에 시간 필요

복잡도를 감수하는 이유:
  각 단계가 단일 책임 → 독립 장애 처리
  쓰기와 읽기 경로 완전 분리 → 독립 스케일 아웃
  이벤트가 모든 단계의 계약 → 느슨한 결합
```

---

## 📌 핵심 정리

```
완전한 흐름 핵심:

성공 경로:
  Command → Bus → Handler → Aggregate → EventStore → Outbox
  → Kafka → Projection → ReadModel
  → Query → ReadModel → Response

장애 격리:
  이벤트 스토어 저장 = 진실의 원천 (여기까지 성공 = 완료)
  Kafka 발행 실패 → Outbox 재시도 (Eventually)
  Projection 실패 → DLQ (읽기 모델 지연, 서비스 계속)
  읽기 모델 장애 → 쓰기 경로 무영향

관찰 가능성:
  correlationId로 전체 사이클 추적
  각 단계 메트릭 수집 (지연, 에러율)
```

---

## 🤔 생각해볼 문제

**Q1.** 이벤트 스토어에 이벤트가 저장된 후 Kafka 발행 전에 서버가 재시작됐다. 데이터 손실이 발생하는가?

<details>
<summary>해설 보기</summary>

Outbox 패턴을 사용한다면 데이터 손실이 없습니다. 이벤트 스토어 저장과 Outbox 저장이 같은 트랜잭션에서 이루어지므로, 서버 재시작 후 OutboxPublisher가 다시 시작되면 published=false인 Outbox 레코드를 발견하고 Kafka에 재발행합니다.

Outbox 패턴 없이 이벤트 발행한다면 데이터 손실이 발생할 수 있습니다. 이벤트 스토어에는 있지만 Kafka에는 없는 상태가 됩니다. 이 경우 이벤트 스토어의 global_seq와 Kafka 마지막 오프셋을 비교해 누락된 이벤트를 찾고 재발행해야 합니다.

또 다른 방법으로 Debezium CDC를 사용하면, 이벤트 스토어의 WAL(Write Ahead Log)을 직접 읽어 Kafka에 발행합니다. 이 경우 Outbox 테이블 없이도 이벤트 스토어와 Kafka 발행의 원자성을 보장합니다.

</details>

---

**Q2.** Query 경로에서 읽기 모델 DB가 응답하지 않을 때 쓰기 경로(Command)는 영향을 받는가? 이 격리가 어떻게 구현되는가?

<details>
<summary>해설 보기</summary>

읽기 모델 DB 장애는 쓰기 경로에 영향을 주지 않습니다. CQRS의 핵심 이점 중 하나입니다.

물리적 격리로 읽기 모델 DB와 이벤트 스토어 DB가 다른 서버입니다. 읽기 모델 DB가 다운돼도 이벤트 스토어 DB는 정상이므로 Command 처리가 계속됩니다.

코드 경로 격리로 Command Handler는 이벤트 스토어에만 접근합니다. 읽기 모델 Repository를 사용하지 않습니다. Query Handler만 읽기 모델 DB에 접근합니다.

네트워크 격리로 Command 경로와 Query 경로가 다른 서비스 인스턴스를 사용한다면 (마이크로서비스), 읽기 서비스 장애가 쓰기 서비스에 네트워크 영향도 없습니다.

결과적으로 읽기 모델 DB가 다운되면 조회 기능은 503이 되지만, 이체, 주문, 등록 등의 쓰기 기능은 정상 작동합니다. 다만 Projection은 이벤트를 계속 수신하므로, 읽기 DB 복구 후 Projection이 밀린 이벤트를 순서대로 처리합니다.

</details>

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[이전: Chapter 4 — 프로젝션 장애 처리 ⬅️](../read-model-projection/06-projection-failure-handling.md)** | **[다음: 이벤트 저장과 발행의 원자성 ➡️](./02-event-store-kafka-atomicity.md)**

</div>
