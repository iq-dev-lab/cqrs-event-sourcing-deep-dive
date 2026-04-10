# 프로젝션 장애 처리

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 프로젝션 처리 실패 시 재처리 전략은 어떻게 설계하는가?
- 멱등 프로젝션을 구현하면 중복 처리 시 왜 안전한가?
- Dead Letter Queue(DLQ)는 어떻게 운영하는가?
- 하나의 프로젝션 장애가 다른 프로젝션에 영향을 주지 않도록 격리하는 방법은?
- 프로젝션이 복구 불가능한 상태에 빠졌을 때 어떻게 처리하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

프로젝션 장애는 피할 수 없다. DB가 일시적으로 연결 불가 상태가 될 수 있고, 이벤트 페이로드의 예상치 못한 값이 NullPointerException을 유발할 수 있다. 중요한 것은 장애가 발생했을 때 얼마나 빠르게 복구하고, 다른 읽기 모델에 영향 없이 격리할 수 있는가다.

---

## 😱 흔한 실수 (Before — 장애 대응 전략 없음)

```
실수 1: 예외 발생 시 전체 프로젝션 중단

  @KafkaListener(topics = "order-events")
  public void handle(String eventJson) {
      DomainEvent event = deserialize(eventJson);
      updateReadModel(event);
      // NullPointerException 발생
      // → Kafka Consumer가 중단됨
      // → 이후 이벤트 모두 처리 중단
      // → 모든 읽기 모델 업데이트 중단
  }

실수 2: 멱등성 없는 프로젝션

  @EventHandler
  public void on(OrderConfirmed event) {
      // 이 이벤트가 두 번 처리되면?
      jdbcTemplate.update(
          "INSERT INTO order_summary (...) VALUES (...)",
          ...
      );
      // UNIQUE 제약 위반 → 예외 → 재처리 루프
  }

실수 3: DLQ 없이 실패 이벤트 영원히 재시도

  while (true) {
      try {
          processEvent(event);
          break;
      } catch (Exception e) {
          retry++;
          Thread.sleep(1000 * retry); // 점점 느려짐
          // 복구 불가 이벤트라면 → 무한 대기
      }
  }
```

---

## ✨ 올바른 접근 (After — 재처리 + 멱등성 + DLQ + 격리)

```
장애 처리 4단계:

1. 재처리 (Retry)
   일시적 오류: DB 연결 실패, 네트워크 타임아웃
   → 지수 백오프로 최대 3~5회 재시도

2. DLQ (Dead Letter Queue)
   재처리 한도 초과 또는 복구 불가 오류
   → DLQ에 저장 후 건너뜀 (다음 이벤트 처리 계속)

3. 멱등성 (Idempotency)
   재처리 시 중복 처리 방지
   → Upsert 패턴으로 동일 이벤트 여러 번 처리해도 동일 결과

4. 격리 (Isolation)
   프로젝션별 독립 Consumer Group
   → 하나 장애가 다른 프로젝션에 영향 없음
```

---

## 🔬 내부 동작 원리

### 1. 재처리 전략 — 오류 유형별 대응

```
오류 유형 분류:

일시적 오류 (Transient Error) — 재시도 가능:
  - DB 연결 타임아웃 (JDBC connection timeout)
  - 네트워크 간헐적 실패
  - DB 데드락 (일시적 경합)
  - 외부 API 일시적 오류 (503)

  → 재시도 전략:
    최대 재시도: 3~5회
    지수 백오프: 1s, 2s, 4s, 8s
    Jitter 추가: 동시 재시도 분산

영구 오류 (Permanent Error) — 재시도 불가:
  - 이벤트 페이로드 파싱 실패 (손상된 JSON)
  - 비즈니스 로직 오류 (예상치 못한 null 값)
  - 스키마 불일치 (이벤트 타입 역직렬화 실패)

  → DLQ로 이동 후 건너뜀

오류 분류 방법:
  try {
      processEvent(event);
  } catch (TransientException e) {
      // DB, 네트워크 오류 → 재시도
      retryWithBackoff(event, e);
  } catch (PermanentException e) {
      // 비즈니스 오류 → DLQ
      sendToDlq(event, e);
  } catch (Exception e) {
      // 알 수 없는 오류 → 보수적으로 DLQ
      sendToDlq(event, e);
  }
```

### 2. 멱등 프로젝션 구현

```
멱등성이 필요한 이유:

  At-Least-Once 처리: Kafka는 이벤트를 최소 1번 전달
  → 네트워크 장애 후 재연결 시 이미 처리한 이벤트 재전달 가능
  → 재전달된 이벤트를 처리해도 읽기 모델이 동일 상태여야 함

멱등 패턴 1 — Upsert (INSERT ON CONFLICT):

  // 동일 order_id로 두 번 INSERT → 두 번째는 UPDATE
  INSERT INTO order_summary (order_id, status, ...)
  VALUES ('ORD-001', 'CONFIRMED', ...)
  ON CONFLICT (order_id)
  DO UPDATE SET
      status = EXCLUDED.status,
      confirmed_at = EXCLUDED.confirmed_at
  WHERE order_summary.last_event_seq < EXCLUDED.last_event_seq;

  // last_event_seq 조건: 더 오래된 이벤트로 덮어쓰기 방지

멱등 패턴 2 — 이벤트 처리 기록:

  processed_events 테이블:
    event_id   UUID PRIMARY KEY
    processed_at TIMESTAMPTZ

  @Transactional
  void handle(StoredEvent event) {
      if (processedEventRepo.existsById(event.eventId())) {
          log.info("중복 이벤트 무시: {}", event.eventId());
          return; // 이미 처리된 이벤트 → 건너뜀
      }
      updateReadModel(event);
      processedEventRepo.save(event.eventId());
      // 두 작업 같은 트랜잭션 → 원자적
  }

  장점: 완전한 중복 방지
  단점: processed_events 테이블이 커짐 (주기적 정리 필요)

멱등 패턴 3 — 상태 비교:

  // 현재 상태를 확인하고 이미 처리됐으면 건너뜀
  void on(OrderConfirmed event) {
      OrderSummary current = repo.findById(event.orderId());
      if (current != null && "CONFIRMED".equals(current.getStatus())) {
          log.info("이미 확인된 주문: {}", event.orderId());
          return;
      }
      updateStatus(event.orderId(), "CONFIRMED");
  }
  // 주의: 이벤트가 순서 외로 도착할 때 잘못 판단할 수 있음
```

### 3. Dead Letter Queue 운영

```
DLQ 구조:

  Kafka DLQ Topic: "order-events-dlq"
  DLQ 메시지:
    original_event: { 원본 이벤트 JSON }
    error_message: "NullPointerException: customerId is null"
    error_class: "java.lang.NullPointerException"
    stack_trace: "..."
    attempt_count: 5
    first_failed_at: 2024-01-15T14:30:00Z
    last_failed_at: 2024-01-15T14:30:50Z
    projection_name: "OrderSummaryProjection"

DLQ 처리 절차:

  Step 1: 모니터링 알람
    DLQ에 메시지가 쌓이면 즉시 알람
    Prometheus alert: kafka_consumer_group_lag{topic="order-events-dlq"} > 0

  Step 2: 원인 분석
    오류 메시지 + 스택 트레이스 확인
    일시적 오류? → 재처리
    코드 버그? → 코드 수정 후 재처리
    데이터 문제? → 이벤트 수정 or 건너뜀

  Step 3: 재처리 or 건너뜀
    재처리: DLQ 메시지를 원본 토픽으로 다시 발행
    건너뜀: DLQ 메시지 수동 확인 후 acknowledged (처리 완료 기록)

DLQ 재처리 API:

  POST /admin/dlq/reprocess
    body: { projection: "OrderSummaryProjection", limit: 100 }
  → DLQ에서 최대 100건 원본 토픽으로 재발행

  POST /admin/dlq/skip
    body: { projection: "OrderSummaryProjection", eventId: "..." }
  → 특정 이벤트 건너뜀 처리
```

### 4. 프로젝션 격리 — 장애 전파 방지

```
격리 설계:

격리 레벨 1 — Consumer Group 분리:
  각 프로젝션 → 독립 Kafka Consumer Group
  → OrderSummaryProjection 장애 → OrderSearchProjection 무영향

격리 레벨 2 — 예외 처리:
  프로젝션 내 예외가 Consumer 스레드를 죽이지 않도록
  try-catch로 감싸고 DLQ로 이동 후 계속 처리

격리 레벨 3 — 트랜잭션 범위:
  각 이벤트 처리를 독립 트랜잭션으로
  → 이벤트 A 처리 실패가 이벤트 B 처리에 영향 없음

격리 레벨 4 — 배포 독립성:
  프로젝션을 별도 서비스/프로세스로 분리 가능
  → OrderSummaryProjection 재배포가 다른 프로젝션에 영향 없음

복구 우선순위:
  Tier 1 (실시간) 프로젝션 → 즉시 알람, 5분 내 대응
  Tier 2 (준실시간) → 30분 내 대응
  Tier 3 (배치) → 다음 배치 실행 전 대응
```

---

## 💻 실전 코드

```java
// ✅ 재처리 + DLQ + 멱등성 통합 구현
@Component
public class ResilientProjectionRunner {

    private static final int MAX_RETRY = 3;

    @KafkaListener(
        topics = "order-events",
        groupId = "order-summary-projection"
    )
    public void handle(ConsumerRecord<String, String> record,
                        Acknowledgment ack) {
        StoredEvent event = deserialize(record.value());

        int attempt = 0;
        while (attempt < MAX_RETRY) {
            try {
                // 멱등 처리 내부
                processIdempotently(event);
                ack.acknowledge(); // 성공 시 오프셋 커밋
                return;

            } catch (TransientException e) {
                attempt++;
                if (attempt >= MAX_RETRY) {
                    sendToDlq(event, e, attempt);
                    ack.acknowledge(); // DLQ로 이동 후 건너뜀
                    return;
                }
                sleepWithBackoff(attempt);

            } catch (Exception e) {
                // 영구 오류 → 즉시 DLQ
                sendToDlq(event, e, 1);
                ack.acknowledge();
                return;
            }
        }
    }

    @Transactional
    private void processIdempotently(StoredEvent event) {
        // 중복 처리 방지
        if (processedEventRepo.existsById(event.eventId())) {
            log.debug("중복 이벤트 무시: {}", event.eventId());
            return;
        }

        DomainEvent domainEvent = upcasterChain.apply(
            event.eventType(), event.payload());

        // 실제 읽기 모델 업데이트
        projectionHandler.handle(domainEvent);

        // 처리 완료 기록 (같은 트랜잭션)
        processedEventRepo.save(new ProcessedEvent(
            event.eventId(), event.projectionName(), Instant.now()));
    }

    private void sendToDlq(StoredEvent event, Exception error, int attempts) {
        DlqMessage dlqMessage = new DlqMessage(
            event,
            error.getMessage(),
            error.getClass().getName(),
            attempts,
            Instant.now()
        );
        kafkaTemplate.send("order-events-dlq",
            event.streamId(), objectMapper.writeValueAsString(dlqMessage));
        metrics.recordDlq(event.eventType(), error.getClass().getSimpleName());
        log.error("DLQ 이동: eventId={} error={}", event.eventId(), error.getMessage());
    }
}

// ✅ 멱등 Upsert 패턴
@Transactional
public void onOrderConfirmed(OrderConfirmed event, long eventSeq) {
    int updated = jdbcTemplate.update("""
        UPDATE order_summary
        SET status = 'CONFIRMED',
            confirmed_at = ?,
            last_event_seq = ?
        WHERE order_id = ?
          AND last_event_seq < ?
        """,
        event.occurredAt(), eventSeq,
        event.orderId(), eventSeq
    );

    if (updated == 0) {
        log.debug("Upsert 무시 (이미 최신): orderId={} seq={}",
            event.orderId(), eventSeq);
        // 더 오래된 이벤트 or 이미 처리됨 → 정상
    }
}

// ✅ DLQ 재처리 어드민 API
@RestController
@RequestMapping("/admin/dlq")
public class DlqAdminController {

    @PostMapping("/reprocess")
    public ReprocessResult reprocess(
            @RequestParam String projectionName,
            @RequestParam(defaultValue = "100") int limit) {

        List<DlqMessage> messages = dlqRepository
            .findByProjection(projectionName, limit);

        int success = 0, failed = 0;
        for (DlqMessage msg : messages) {
            try {
                projectionRunner.processIdempotently(msg.originalEvent());
                dlqRepository.markReprocessed(msg.id());
                success++;
            } catch (Exception e) {
                log.error("DLQ 재처리 실패: {}", e.getMessage());
                failed++;
            }
        }

        return new ReprocessResult(success, failed, messages.size());
    }

    @GetMapping("/status")
    public DlqStatus getDlqStatus() {
        return new DlqStatus(
            dlqRepository.countByProjection("OrderSummaryProjection"),
            dlqRepository.countByProjection("OrderSearchProjection"),
            dlqRepository.getOldestFailedAt()
        );
    }
}
```

---

## 📊 패턴 비교

```
장애 처리 전략 비교:

┌────────────────────┬──────────────┬──────────────┬──────────────────┐
│ 전략               │ 데이터 손실  │ 처리 지연    │ 운영 복잡도      │
├────────────────────┼──────────────┼──────────────┼──────────────────┤
│ 재시도만            │ 없음         │ 높음 (무한)  │ 낮음             │
├────────────────────┼──────────────┼──────────────┼──────────────────┤
│ DLQ + 건너뜀       │ 임시 누락    │ 없음         │ 중간             │
├────────────────────┼──────────────┼──────────────┼──────────────────┤
│ 재시도 + DLQ       │ 없음 (재처리)│ 최소         │ 높음             │
├────────────────────┼──────────────┼──────────────┼──────────────────┤
│ 멱등 + 재시도 + DLQ│ 없음         │ 최소         │ 높음             │
└────────────────────┴──────────────┴──────────────┴──────────────────┘
권장: 멱등 + 재시도 (3회) + DLQ
```

---

## ⚖️ 트레이드오프

```
멱등성 구현 비용:
  processed_events 테이블 → 저장 공간, 조회 비용
  Upsert → 단순 INSERT보다 복잡한 SQL
  편익: 중복 처리 안전 → 재시도 전략 자유롭게 사용

DLQ 운영 비용:
  DLQ 모니터링 + 알람 설정 필요
  DLQ 재처리 프로세스 필요
  편익: 하나의 문제 이벤트가 전체 처리 막지 않음
```

---

## 📌 핵심 정리

```
프로젝션 장애 처리 핵심:

재처리 전략:
  일시적 오류 → 지수 백오프 재시도 (3~5회)
  영구 오류 → 즉시 DLQ

멱등성:
  Upsert (ON CONFLICT DO UPDATE) — 가장 단순
  processed_events 테이블 — 완전한 중복 방지
  last_event_seq 조건 — 순서 역전 방지

DLQ 운영:
  자동 알람 → 원인 분석 → 재처리 or 건너뜀
  재처리 API 구축 필수

격리:
  프로젝션별 독립 Consumer Group
  이벤트별 독립 트랜잭션
  → 하나 장애가 다른 프로젝션에 영향 없음
```

---

## 🤔 생각해볼 문제

**Q1.** 특정 이벤트가 복구 불가능한 이유로 DLQ에 쌓였을 때, 이 이벤트를 영원히 건너뛰면 읽기 모델에 어떤 영향이 있는가?

<details>
<summary>해설 보기</summary>

DLQ에서 건너뛴 이벤트는 해당 읽기 모델에 반영되지 않습니다. 이것이 데이터 불일치를 만들 수 있습니다.

예를 들어 `OrderConfirmed` 이벤트가 DLQ로 이동하고 건너뛴다면, 해당 주문의 `order_summary.status`는 여전히 'PLACED'이지만 실제 비즈니스 상태는 'CONFIRMED'입니다.

이 불일치를 해결하는 방법은 두 가지입니다. 첫째, 이벤트 문제를 수정하고 DLQ에서 재처리합니다. 페이로드 문제라면 이벤트를 수정해 재발행하거나, Upcaster로 변환해 재처리합니다.

둘째, 해당 Aggregate의 이벤트 스트림을 재구축합니다. 해당 stream_id의 체크포인트를 리셋하고 처음부터 재처리합니다. DLQ 이벤트를 수정된 버전으로 교체하거나 건너뛰는 로직을 추가합니다.

건너뜀은 임시 조치이고, 근본적으로는 원인을 수정하고 재처리해야 합니다.

</details>

---

**Q2.** 프로젝션이 처리 중인 이벤트 수백만 건이 모두 DLQ에 쌓였다. 어떻게 대량 재처리를 효율적으로 수행하는가?

<details>
<summary>해설 보기</summary>

대량 DLQ 재처리의 핵심은 원인 파악 → 수정 → 배치 재처리 순서입니다.

먼저 근본 원인을 수정합니다. 수백만 건이 모두 실패했다면 코드 버그가 원인일 가능성이 높습니다. 코드를 수정하지 않고 재처리하면 같은 오류가 반복됩니다.

다음으로 Projection 재구축을 고려합니다. DLQ에서 수백만 건을 하나씩 재처리하는 것보다, 읽기 모델 전체를 처음부터 재구축하는 것이 더 빠를 수 있습니다. Blue/Green Projection으로 무중단 재구축하면 됩니다.

재구축이 적절하지 않다면 배치 재처리합니다. DLQ 이벤트를 원본 Kafka 토픽에 배치로 재발행합니다. 재처리 속도를 조절해 운영 중인 프로덕션 처리에 영향을 최소화합니다. 야간 시간대를 활용합니다.

재처리 중 진행률과 오류율을 모니터링하고, 오류가 0에 가까워지면 완료로 판단합니다.

</details>

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[이전: 여러 읽기 모델의 공존 ⬅️](./05-multiple-read-models.md)** | **[다음: Chapter 5 — 완전한 흐름 ➡️](../integrated-architecture/01-complete-flow.md)**

</div>
