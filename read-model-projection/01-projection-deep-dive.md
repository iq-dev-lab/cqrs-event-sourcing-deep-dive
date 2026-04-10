# 프로젝션(Projection) 완전 분해

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 프로젝션이 이벤트 스트림을 읽기 모델로 변환하는 과정은 내부적으로 어떻게 동작하는가?
- 프로젝션이 Kafka Consumer처럼 동작한다는 것은 무슨 의미인가?
- 오프셋(체크포인트)을 추적하는 이유와 방법은 무엇인가?
- 읽기 모델 저장소를 RDB, Redis, Elasticsearch 중 어떤 기준으로 선택하는가?
- 이벤트 타입별 프로젝션 라우팅은 어떻게 설계하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

프로젝션은 CQRS에서 "쓰기 모델의 변화를 읽기 모델에 반영"하는 유일한 경로다. 프로젝션이 느리면 Eventual Consistency 지연이 길어지고, 프로젝션이 실패하면 읽기 모델이 오래된 데이터를 제공한다. 프로젝션이 중복 처리되면 읽기 모델 데이터가 오염된다. 프로젝션의 내부 동작을 정확히 이해해야 이 모든 문제를 예방하고 대응할 수 있다.

---

## 😱 흔한 실수 (Before — 프로젝션 원리를 모를 때)

```
실수 1: 프로젝션이 이벤트를 어디서 처리했는지 추적하지 않음

  @KafkaListener(topics = "order-events")
  public void handleEvent(String eventJson) {
      DomainEvent event = deserialize(eventJson);
      updateReadModel(event);
      // 오프셋/체크포인트 저장 없음
  }
  // 프로세스 재시작 시 어디서부터 재처리해야 하는지 모름
  // Kafka auto-commit에 의존 → 처리 실패 후 다음 이벤트로 이동 가능

실수 2: 프로젝션과 읽기 모델 업데이트가 비원자적

  @KafkaListener(topics = "order-events")
  public void handleEvent(String eventJson) {
      DomainEvent event = deserialize(eventJson);
      readModelRepository.update(event);      // DB 업데이트
      kafkaConsumer.commitOffset(event);      // 오프셋 커밋
      // 두 단계 사이 프로세스 죽으면:
      // DB 업데이트됨 + 오프셋 미커밋 → 재시작 시 중복 처리
  }

실수 3: 모든 이벤트를 하나의 프로젝션에서 처리

  @EventHandler
  public void handleAll(DomainEvent event) {
      if (event instanceof OrderConfirmed) updateOrderSummary(event);
      else if (event instanceof OrderShipped) updateDeliveryView(event);
      else if (event instanceof OrderCancelled) updateCancelStats(event);
      // 프로젝션 하나가 느려지면 모든 읽기 모델 업데이트 지연
      // 한 처리 실패가 모든 읽기 모델에 영향
  }
```

---

## ✨ 올바른 접근 (After — 프로젝션 설계 원칙)

```
올바른 프로젝션 설계:

1. 체크포인트(오프셋) 추적
   처리한 마지막 이벤트 위치를 DB에 저장
   재시작 시 체크포인트부터 재처리

2. 멱등 처리 (Idempotent)
   같은 이벤트를 두 번 처리해도 동일 결과
   → 중복 처리 시 읽기 모델 오염 방지

3. 프로젝션 격리
   읽기 모델별 독립 프로젝션 + 독립 Consumer Group
   → 하나 실패가 다른 읽기 모델에 영향 없음

4. 읽기 모델 저장소 목적별 선택
   RDB: 복잡한 조건 필터링, 트랜잭션 필요
   Redis: 캐시, 세션, 카운터 (< 1ms 조회)
   Elasticsearch: 전문 검색, 집계 분석
```

---

## 🔬 내부 동작 원리

### 1. 프로젝션 = Kafka Consumer

```
프로젝션이 Kafka Consumer인 이유:

  이벤트 스토어 → (이벤트 발행) → Kafka Topic
  Projection (Consumer Group A) → Order Summary 읽기 모델
  Projection (Consumer Group B) → Order Search 읽기 모델
  Projection (Consumer Group C) → Analytics 읽기 모델

  각 Consumer Group은 독립적으로:
    ① 이벤트 토픽 구독
    ② 자신만의 오프셋 추적 (어디까지 처리했나)
    ③ 자신만의 읽기 모델 업데이트

  Kafka Consumer Group의 핵심 특성:
    ① 각 Consumer Group은 독립적으로 오프셋 관리
       Group A: offset=1050 처리 완료
       Group B: offset=987  처리 완료 (Group A보다 느림)
       → 서로 독립, Group A가 빨라도 Group B가 스킵 못함

    ② 파티션 병렬 처리
       topic: order-events (파티션 3개)
       Consumer Group A: 인스턴스 3개 → 각 인스턴스가 파티션 1개 담당
       → 병렬 처리로 처리량 향상

    ③ At-Least-Once 보장 (기본)
       이벤트는 최소 1번 처리 → 멱등 구현 필요

오프셋 = 체크포인트:
  Kafka: offset (Consumer가 어느 메시지까지 처리했는지)
  직접 이벤트 스토어 폴링: global_seq (어느 이벤트까지 처리했는지)

  projection_checkpoints 테이블:
  ┌──────────────────────────┬──────────────────┬─────────────────┐
  │ projection_name          │ last_processed   │ updated_at      │
  ├──────────────────────────┼──────────────────┼─────────────────┤
  │ OrderSummaryProjection   │ global_seq=10500 │ 2024-01-15 ...  │
  │ OrderSearchProjection    │ global_seq=9870  │ 2024-01-15 ...  │
  │ AnalyticsProjection      │ global_seq=8200  │ 2024-01-14 ...  │
  └──────────────────────────┴──────────────────┴─────────────────┘
```

### 2. 이벤트 타입별 라우팅

```
라우팅 구조:

  이벤트 스트림:
  [OrderPlaced] [OrderConfirmed] [MoneyWithdrawn] [OrderShipped] ...

  Router:
    OrderPlaced    → OrderSummaryProjection, OrderSearchProjection
    OrderConfirmed → OrderSummaryProjection, RevenueProjection
    MoneyWithdrawn → AccountSummaryProjection
    OrderShipped   → DeliveryProjection, OrderSummaryProjection

  프로젝션별 관심 이벤트:
    OrderSummaryProjection:
      @EventHandler OrderPlaced
      @EventHandler OrderConfirmed
      @EventHandler OrderShipped
      @EventHandler OrderCancelled

    RevenueProjection:
      @EventHandler OrderConfirmed (금액 집계)
      @EventHandler OrderCancelled (환불 차감)

    DeliveryProjection:
      @EventHandler OrderShipped
      @EventHandler DeliveryCompleted
      @EventHandler DeliveryFailed

  모르는 이벤트 타입:
    → 무시 (default 처리)
    → 프로젝션이 관심 없는 이벤트를 받아도 오류 없이 넘김
```

### 3. 읽기 모델 저장소 선택 기준

```
RDB (PostgreSQL/MySQL):
  적합한 경우:
    복잡한 WHERE 조건 필터링 (status='CONFIRMED' AND city='서울')
    정렬 + 페이지네이션 (ORDER BY created_at DESC LIMIT 20)
    집계 쿼리 (GROUP BY customer_id)
    트랜잭션이 필요한 읽기 모델 업데이트

  예시: order_summary, account_summary, user_profile

  한계:
    전문 검색 어려움 (LIKE '%아이폰%' → 인덱스 미사용)
    수십억 행 집계 느림

Redis:
  적합한 경우:
    < 1ms 응답 필요 (실시간 잔고 조회)
    카운터 (좋아요 수, 조회수)
    세션 데이터
    자주 읽히고 드물게 바뀌는 데이터

  예시: 계좌 잔고 캐시, 상품 재고, 실시간 랭킹

  한계:
    복잡한 쿼리 불가 (key-value 또는 Hash)
    대용량 데이터 비용

Elasticsearch:
  적합한 경우:
    전문 검색 (상품명, 설명 검색)
    복잡한 집계 (대시보드, 통계)
    다양한 필터 조합 (AND/OR/범위)
    지리 검색

  예시: 상품 검색 인덱스, CS 고객 통합 뷰, 로그 분석

  한계:
    Strong Consistency 불가 (Near Real-Time)
    운영 복잡도 높음
```

### 4. 프로젝션 처리 흐름 — 단계별

```
단계별 처리:

Step 1: 이벤트 소비
  이벤트 스토어 폴링 or Kafka Consumer
  → StoredEvent { global_seq, stream_id, event_type, payload }

Step 2: 역직렬화 + Upcasting
  payload (JSON) → DomainEvent 객체
  Upcaster 체인 적용 (구버전 이벤트 처리)

Step 3: 라우팅
  event.getClass() → 해당 @EventHandler 메서드 호출

Step 4: 읽기 모델 업데이트 (멱등 처리)
  INSERT ... ON CONFLICT DO UPDATE (Upsert)
  → 중복 처리 시 동일 결과 보장

Step 5: 체크포인트 저장
  UPDATE projection_checkpoints SET last_processed = global_seq
  → Step 4와 같은 트랜잭션 (원자성 보장)
  → 재시작 시 이 지점부터 재처리

전체 흐름:
  ┌─────────────────────────────────────────────┐
  │              ProjectionRunner               │
  │                                             │
  │  loop:                                      │
  │    checkpoint = loadCheckpoint(name)        │
  │    events = eventStore.loadSince(checkpoint)│
  │    for event in events:                     │
  │      handler.handle(event)    ← Step 3,4   │
  │      saveCheckpoint(event.seq) ← Step 5    │
  └─────────────────────────────────────────────┘
```

---

## 💻 실전 코드

```java
// ==============================
// 완전한 프로젝션 구현
// ==============================

// ✅ 프로젝션 체크포인트 관리
@Repository
public class ProjectionCheckpointRepository {

    public long loadCheckpoint(String projectionName) {
        try {
            return jdbcTemplate.queryForObject(
                "SELECT last_global_seq FROM projection_checkpoints WHERE projection_name = ?",
                Long.class, projectionName);
        } catch (EmptyResultDataAccessException e) {
            return 0L; // 첫 실행
        }
    }

    @Transactional
    public void saveCheckpoint(String projectionName, long globalSeq) {
        jdbcTemplate.update("""
            INSERT INTO projection_checkpoints (projection_name, last_global_seq, updated_at)
            VALUES (?, ?, NOW())
            ON CONFLICT (projection_name)
            DO UPDATE SET last_global_seq = ?, updated_at = NOW()
            """, projectionName, globalSeq, globalSeq);
    }
}

// ✅ 프로젝션 실행기 — 이벤트 스토어 폴링
@Component
public class ProjectionRunner {

    private final EventStore eventStore;
    private final List<Projection> projections;
    private final ProjectionCheckpointRepository checkpointRepo;

    @Scheduled(fixedDelay = 500) // 500ms마다 폴링
    public void runProjections() {
        projections.forEach(this::processProjection);
    }

    private void processProjection(Projection projection) {
        String name = projection.getName();
        long lastSeq = checkpointRepo.loadCheckpoint(name);

        List<StoredEvent> events = eventStore.loadBatch(lastSeq, 100);
        if (events.isEmpty()) return;

        for (StoredEvent stored : events) {
            try {
                DomainEvent event = deserialize(stored);
                projection.handle(event);
                checkpointRepo.saveCheckpoint(name, stored.globalSeq());
            } catch (Exception e) {
                log.error("[{}] 이벤트 처리 실패: seq={} type={} error={}",
                    name, stored.globalSeq(), stored.eventType(), e.getMessage());
                dlqService.send(stored, e); // DLQ로 이동
                break; // 이 프로젝션 중단, 다음 폴링에서 재시도
            }
        }
    }
}

// ✅ Order Summary 프로젝션 — 멱등 처리 (Upsert)
@Component
public class OrderSummaryProjection implements Projection {

    @Override
    public String getName() { return "OrderSummaryProjection"; }

    @Override
    public void handle(DomainEvent event) {
        switch (event) {
            case OrderPlaced e -> onOrderPlaced(e);
            case OrderConfirmed e -> onOrderConfirmed(e);
            case OrderShipped e -> onOrderShipped(e);
            case OrderCancelled e -> onOrderCancelled(e);
            default -> { /* 무시 */ }
        }
    }

    @Transactional
    private void onOrderPlaced(OrderPlaced event) {
        // UPSERT — 동일 이벤트 두 번 처리 시 안전
        jdbcTemplate.update("""
            INSERT INTO order_summary
                (order_id, customer_id, customer_name, status, total_amount,
                 item_count, created_at)
            VALUES (?, ?, ?, 'PLACED', ?, ?, ?)
            ON CONFLICT (order_id)
            DO UPDATE SET
                status = 'PLACED',
                updated_at = NOW()
            """,
            event.orderId(), event.customerId(),
            customerCache.getName(event.customerId()),
            event.totalAmount(), event.itemCount(),
            event.occurredAt()
        );
    }

    @Transactional
    private void onOrderConfirmed(OrderConfirmed event) {
        jdbcTemplate.update("""
            UPDATE order_summary
            SET status = 'CONFIRMED', confirmed_at = ?, updated_at = NOW()
            WHERE order_id = ?
            """,
            event.occurredAt(), event.orderId()
        );
    }
}

// ✅ Kafka 기반 프로젝션 (Axon 없이)
@Component
public class OrderSearchProjection {

    @KafkaListener(
        topics = "order-events",
        groupId = "order-search-projection",
        containerFactory = "projectionKafkaListenerContainerFactory"
    )
    public void handle(ConsumerRecord<String, String> record,
                        Acknowledgment acknowledgment) {
        try {
            StoredEvent event = objectMapper.readValue(record.value(), StoredEvent.class);
            DomainEvent domainEvent = deserialize(event);

            if (domainEvent instanceof OrderPlaced e) {
                indexOrder(e); // Elasticsearch 인덱싱
            } else if (domainEvent instanceof OrderCancelled e) {
                removeFromIndex(e);
            }

            // 처리 성공 후 수동 오프셋 커밋
            acknowledgment.acknowledge();

        } catch (Exception e) {
            log.error("Elasticsearch 인덱싱 실패: {}", e.getMessage());
            // 커밋 안 함 → 재시도
            // 반복 실패 시 DLQ로 이동 (재시도 횟수 제한)
        }
    }

    private void indexOrder(OrderPlaced event) {
        elasticsearchClient.index(i -> i
            .index("orders")
            .id(event.orderId())
            .document(Map.of(
                "orderId", event.orderId(),
                "customerName", event.customerName(),
                "productNames", event.productNames(),
                "status", "PLACED",
                "totalAmount", event.totalAmount(),
                "createdAt", event.occurredAt().toString()
            ))
        );
    }
}
```

---

## 📊 패턴 비교

```
프로젝션 구동 방식 비교:

┌────────────────────┬───────────────────┬──────────────────────┐
│ 방식               │ 이벤트 스토어 폴링 │ Kafka Consumer       │
├────────────────────┼───────────────────┼──────────────────────┤
│ 체크포인트 관리    │ DB (global_seq)   │ Kafka Consumer Group │
├────────────────────┼───────────────────┼──────────────────────┤
│ 원자성             │ DB 트랜잭션 가능  │ Outbox 패턴 필요     │
├────────────────────┼───────────────────┼──────────────────────┤
│ 재구축 용이성      │ 쉬움 (seq 리셋)  │ 오프셋 리셋 필요     │
├────────────────────┼───────────────────┼──────────────────────┤
│ 처리량             │ 폴링 주기 의존    │ 높음 (Push 기반)     │
├────────────────────┼───────────────────┼──────────────────────┤
│ 추가 인프라        │ 없음              │ Kafka 필요           │
└────────────────────┴───────────────────┴──────────────────────┘

읽기 모델 저장소 선택:
  복잡한 필터 + 정렬 → RDB
  < 1ms 응답 → Redis
  전문 검색 + 집계 → Elasticsearch
  시계열 데이터 → InfluxDB / TimescaleDB
```

---

## ⚖️ 트레이드오프

```
폴링 vs Kafka Push:
  폴링: 구현 단순, 인프라 추가 없음, 재구축 쉬움
        → 폴링 주기 내 최대 지연 발생
  Kafka: 실시간에 가까운 처리, 높은 처리량
        → Kafka 운영 복잡도, 오프셋 관리 필요

프로젝션 격리 비용:
  읽기 모델 N개 → 독립 프로젝션 N개
  → 이벤트 N번 소비, 체크포인트 N개 관리
  편익: 하나 실패가 다른 것에 영향 없음
```

---

## 📌 핵심 정리

```
프로젝션 핵심 개념:

정의:
  이벤트 스트림 → (Projection) → 읽기 모델
  Kafka Consumer처럼 동작 (오프셋 추적)

체크포인트:
  처리한 마지막 이벤트 위치 저장
  재시작 시 체크포인트부터 재처리

멱등 처리:
  같은 이벤트 두 번 처리 → 동일 결과 (Upsert)

격리:
  읽기 모델별 독립 프로젝션 + 독립 Consumer Group
  → 장애 격리

저장소 선택:
  RDB / Redis / ES → 읽기 요구사항 기반 선택
```

---

## 🤔 생각해볼 문제

**Q1.** 프로젝션이 이벤트를 처리하고 읽기 모델을 업데이트한 직후 프로세스가 죽으면 어떤 일이 발생하는가? 체크포인트를 읽기 모델 업데이트와 같은 트랜잭션에 저장해야 하는 이유는 무엇인가?

<details>
<summary>해설 보기</summary>

체크포인트를 별도로 저장하면 두 가지 문제가 발생합니다.

첫째, 읽기 모델 업데이트 성공 + 체크포인트 저장 실패 → 재시작 시 같은 이벤트를 다시 처리합니다. 이 경우 멱등 처리(Upsert)가 되어 있으면 동일한 결과가 나와 문제없습니다.

둘째, 읽기 모델 업데이트 실패 + 체크포인트 저장 성공 → 재시작 시 해당 이벤트를 건너뜁니다. 이 경우 읽기 모델에 해당 이벤트가 반영되지 않아 데이터 누락이 발생합니다. 이것이 진짜 문제입니다.

따라서 "읽기 모델 업데이트"와 "체크포인트 저장"을 같은 DB 트랜잭션 안에서 처리해야 합니다. 두 작업이 원자적으로 성공하거나 실패합니다. 읽기 모델이 RDB라면 자연스럽게 같은 트랜잭션에 묶을 수 있습니다. Elasticsearch나 Redis처럼 다른 저장소라면 Outbox 패턴이 필요합니다.

</details>

---

**Q2.** 같은 이벤트를 처리하는 프로젝션이 3개 있을 때, 각 프로젝션의 처리 속도가 다르면 어떤 문제가 생기는가?

<details>
<summary>해설 보기</summary>

각 프로젝션이 독립 Consumer Group을 가지므로, 처리 속도 차이는 읽기 모델 간 Eventual Consistency 수준의 차이를 만듭니다.

빠른 프로젝션(OrderSummaryProjection, 50ms 지연)의 읽기 모델은 거의 실시간에 가깝고, 느린 프로젝션(AnalyticsProjection, 5분 지연)의 읽기 모델은 5분 전 데이터를 제공합니다.

이것이 문제가 되는 경우는 두 읽기 모델의 데이터를 조합할 때입니다. 예를 들어 "주문 목록(OrderSummary)"과 "매출 통계(Analytics)"가 같은 화면에 있을 때, 방금 주문한 것이 목록에는 보이지만 매출 통계에는 아직 없습니다.

해결 방향은 두 가지입니다. 첫째, 읽기 요구사항에 맞는 지연을 수용합니다. 통계는 5분 지연이 자연스럽고, 목록은 실시간에 가깝습니다. 둘째, 같은 화면의 데이터는 같은 프로젝션 또는 비슷한 처리 속도를 가진 프로젝션에서 가져옵니다.

</details>

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[이전: Chapter 3 — Event Sourcing의 실제 어려움 ⬅️](../event-sourcing/07-event-sourcing-challenges.md)** | **[다음: 읽기 모델 설계 원칙 ➡️](./02-read-model-design.md)**

</div>
