# Event Sourcing 없는 CQRS

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Event Sourcing 없이 CQRS만 적용하면 어떤 이점을 얻을 수 있는가?
- DB 트리거/CDC로 읽기 모델을 동기화하는 방법은 무엇인가?
- 단순 CQRS(같은 DB, 다른 스키마)의 실용적 구현은 어떻게 되는가?
- Event Sourcing 도입 없이 CQRS 이점을 얻는 수준별 전략은 무엇인가?
- "CQRS를 하려면 Event Sourcing이 필수"라는 오해는 왜 생기는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

CQRS와 Event Sourcing은 함께 자주 소개되지만 독립적인 패턴이다. Event Sourcing 없이도 CQRS의 핵심 이점(읽기/쓰기 모델 분리, 각각 독립 최적화)을 얻을 수 있다. 오히려 Event Sourcing의 복잡도 없이 CQRS만 적용하면 더 넓은 팀에서 더 빠르게 도입할 수 있다.

---

## 😱 흔한 실수 (Before — CQRS = Event Sourcing이라고 착각)

```
오해: "CQRS를 하려면 이벤트 스토어와 Projection이 필수다"

현실:
  CQRS의 본질은 Command와 Query를 다른 모델로 처리하는 것
  Event Sourcing은 쓰기 모델의 저장 방식 선택
  → 이벤트 스토어 없이도 CQRS 가능

잘못된 접근:
  팀이 CQRS 도입하려다 "Event Sourcing까지 필요"로 판단
  → 학습 비용 과다 → 포기
  → 결과: 읽기-쓰기 혼합된 Service 계속 사용

올바른 접근:
  단계 1: Service 내 읽기/쓰기 메서드 분리 (CQS)
  단계 2: 같은 DB, 읽기 전용 Query 객체 도입
  단계 3: 읽기 모델 DB 분리 (CDC or 동기 업데이트)
  단계 4: (필요시) Event Sourcing 도입
```

---

## ✨ 올바른 접근 (After — 수준별 CQRS)

```
Level 0: CQS (메서드 분리)
  같은 Service, 읽기와 쓰기 메서드 분리
  → 코드 명확성 향상

Level 1: Command/Query 객체 도입
  같은 DB, 별도 Repository/DAO
  → 읽기 최적화 쿼리 별도 관리

Level 2: 읽기 모델 DB 뷰
  같은 DB, 읽기 전용 View 또는 Materialized View
  → 복잡한 조인을 사전 계산

Level 3: 분리된 읽기 모델 DB
  별도 DB/스키마, CDC or 트리거로 동기화
  → 각 DB를 목적에 맞게 최적화

Level 4: CQRS + Event Sourcing (완전한 CQRS)
  이벤트 스토어 기반 쓰기, 이벤트 기반 읽기 모델 구축
```

---

## 🔬 내부 동작 원리

### 1. Level 2 — PostgreSQL Materialized View

```
Materialized View로 CQRS 효과:

  쓰기 모델 (정규화된 테이블):
    orders (order_id, customer_id, status, created_at)
    order_items (order_id, product_id, quantity, price)
    customers (customer_id, name, email, tier)

  읽기 모델 (Materialized View):
    CREATE MATERIALIZED VIEW order_summary_view AS
    SELECT
        o.order_id,
        c.name AS customer_name,
        c.tier AS customer_tier,
        COUNT(oi.product_id) AS item_count,
        SUM(oi.quantity * oi.price) AS total_amount,
        o.status,
        o.created_at
    FROM orders o
    JOIN customers c ON o.customer_id = c.customer_id
    JOIN order_items oi ON o.order_id = oi.order_id
    GROUP BY o.order_id, c.name, c.tier, o.status, o.created_at;

    CREATE UNIQUE INDEX ON order_summary_view (order_id);
    CREATE INDEX ON order_summary_view (status, created_at DESC);

  조회:
    SELECT * FROM order_summary_view
    WHERE status = 'CONFIRMED'
    ORDER BY created_at DESC
    LIMIT 20;
    → JOIN 없이 단일 테이블 조회

  갱신:
    REFRESH MATERIALIZED VIEW CONCURRENTLY order_summary_view;
    → 변경이 있을 때마다 또는 주기적으로

  장점:
    DB 변경 없이 읽기 최적화
    JOIN 제거 → 조회 빠름
    추가 인프라 없음

  단점:
    갱신 주기만큼 지연 (REFRESH 필요)
    실시간 반영 어려움
```

### 2. Level 3 — CDC (Change Data Capture)로 읽기 모델 동기화

```
CDC + 읽기 전용 DB:

  쓰기 DB (PostgreSQL — 정규화, ACID):
    orders, order_items, customers 테이블

  CDC (Debezium):
    PostgreSQL WAL 감시
    변경 발생 → Kafka 발행
    "orders 테이블에 UPDATE 발생" → Kafka 메시지

  읽기 DB 업데이트 Consumer:
    Kafka에서 변경 수신
    order_summary 읽기 테이블에 반영

  읽기 DB (PostgreSQL or Elasticsearch — 비정규화):
    order_summary: 모든 필요 데이터 한 테이블에

  장점:
    쓰기 DB와 읽기 DB 완전 분리
    읽기 DB를 읽기에 최적화 (다른 DB 엔진도 가능)
    쓰기 코드 변경 없이 읽기 모델 추가

  단점:
    Debezium, Kafka 운영 복잡도
    Eventual Consistency 지연
```

### 3. Level 3 — DB 트리거로 읽기 모델 업데이트

```
PostgreSQL 트리거 기반 동기화:

  -- orders 테이블 변경 시 order_summary 자동 업데이트
  CREATE OR REPLACE FUNCTION sync_order_summary()
  RETURNS TRIGGER AS $$
  BEGIN
      IF TG_OP = 'INSERT' OR TG_OP = 'UPDATE' THEN
          INSERT INTO order_summary (
              order_id, customer_name, status, total_amount, created_at
          )
          SELECT
              NEW.order_id,
              c.name,
              NEW.status,
              COALESCE(SUM(oi.quantity * oi.price), 0),
              NEW.created_at
          FROM customers c
          JOIN order_items oi ON NEW.order_id = oi.order_id
          WHERE c.customer_id = NEW.customer_id
          ON CONFLICT (order_id) DO UPDATE SET
              status = EXCLUDED.status,
              total_amount = EXCLUDED.total_amount;
      END IF;
      RETURN NEW;
  END;
  $$ LANGUAGE plpgsql;

  CREATE TRIGGER sync_order_summary_trigger
  AFTER INSERT OR UPDATE ON orders
  FOR EACH ROW EXECUTE FUNCTION sync_order_summary();

  장점:
    쓰기 트랜잭션과 동시에 읽기 모델 업데이트 (강한 일관성!)
    애플리케이션 코드 변경 없음

  단점:
    DB 로직이 트리거에 분산 → 유지보수 어려움
    트리거 오류가 쓰기 트랜잭션 실패로 이어짐
    트리거 내 복잡한 로직 → 성능 저하
```

### 4. 코드 레벨 CQRS — 같은 DB에서 Command/Query 분리

```
Command Side (쓰기):
  OrderCommandService → OrderRepository (JPA, 정규화 모델)
  → orders, order_items 테이블 (쓰기 최적화)

Query Side (읽기):
  OrderQueryService → OrderQueryRepository (JDBC, 비정규화 조회)
  → 복잡한 JOIN 쿼리 or 읽기 전용 View

  @Repository
  public class OrderQueryRepository {

      @Transactional(readOnly = true)
      public Page<OrderSummaryView> findConfirmedOrders(Pageable pageable) {
          // 복잡한 JOIN 쿼리를 여기에 캡슐화
          return jdbcTemplate.query("""
              SELECT o.order_id, c.name, SUM(oi.price) as total
              FROM orders o
              JOIN customers c ON o.customer_id = c.customer_id
              JOIN order_items oi ON o.order_id = oi.order_id
              WHERE o.status = 'CONFIRMED'
              GROUP BY o.order_id, c.name
              ORDER BY o.created_at DESC
              LIMIT ? OFFSET ?
              """, ...
          );
      }
  }

  장점: 구현 단순, 추가 인프라 없음
  단점: 읽기 모델이 쓰기 DB에 의존 → 쓰기 부하가 읽기에 영향
```

---

## 💻 실전 코드

```java
// ✅ Level 1 — 단순 CQRS (같은 DB, Command/Query 분리)
@Service
public class OrderCommandService {
    private final OrderRepository orderRepo; // JPA

    @Transactional
    public String placeOrder(PlaceOrderCommand cmd) {
        Order order = new Order(cmd.customerId(), cmd.items());
        return orderRepo.save(order).getOrderId();
    }

    @Transactional
    public void confirmOrder(String orderId) {
        Order order = orderRepo.findById(orderId).orElseThrow();
        order.confirm();
        orderRepo.save(order);
    }
}

@Service
@Transactional(readOnly = true)
public class OrderQueryService {
    private final OrderSummaryRepository summaryRepo; // JDBC, 비정규화

    public Page<OrderSummaryView> getConfirmedOrders(Pageable pageable) {
        return summaryRepo.findByStatus("CONFIRMED", pageable);
    }

    public OrderDetailView getOrderDetail(String orderId) {
        return summaryRepo.findDetailById(orderId).orElseThrow();
    }
}

// ✅ Level 3 — Spring 이벤트로 읽기 모델 동기화 (Kafka 없이)
@Service
@Transactional
public class OrderCommandServiceWithEvents {
    private final OrderRepository orderRepo;
    private final ApplicationEventPublisher eventPublisher;

    public String placeOrder(PlaceOrderCommand cmd) {
        Order order = new Order(cmd.customerId(), cmd.items());
        String orderId = orderRepo.save(order).getOrderId();

        // Spring ApplicationEvent로 읽기 모델 업데이트 트리거
        eventPublisher.publishEvent(new OrderPlacedEvent(orderId, cmd));
        return orderId;
    }
}

// 읽기 모델 업데이트 (같은 트랜잭션 or 별도 트랜잭션 선택)
@Component
public class OrderSummaryUpdater {

    // @TransactionalEventListener(phase = AFTER_COMMIT): 커밋 후 처리
    // 장점: 쓰기 성공 후에만 읽기 업데이트 → 일관성
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void on(OrderPlacedEvent event) {
        // 별도 트랜잭션 (쓰기 커밋 후)
        orderSummaryRepo.insert(buildSummary(event));
    }
}
```

---

## 📊 패턴 비교

```
CQRS 수준별 비교:

┌──────────┬──────────────┬──────────────┬──────────────┬──────────┐
│ Level    │ 분리 수준    │ 읽기 최적화  │ 복잡도       │ 추가 인프│
├──────────┼──────────────┼──────────────┼──────────────┼──────────┤
│ 0: CQS   │ 메서드       │ 없음         │ 매우 낮음    │ 없음     │
├──────────┼──────────────┼──────────────┼──────────────┼──────────┤
│ 1: Query │ Repository   │ 쿼리 최적화  │ 낮음         │ 없음     │
│ 객체     │              │              │              │          │
├──────────┼──────────────┼──────────────┼──────────────┼──────────┤
│ 2: MView │ View/Schema  │ 사전 계산    │ 낮음         │ 없음     │
├──────────┼──────────────┼──────────────┼──────────────┼──────────┤
│ 3: CDC   │ 별도 DB      │ 완전 독립    │ 높음         │ Kafka    │
├──────────┼──────────────┼──────────────┼──────────────┼──────────┤
│ 4: ES    │ 완전 분리    │ 완전 독립    │ 매우 높음    │ 이벤트스│
└──────────┴──────────────┴──────────────┴──────────────┴──────────┘
```

---

## ⚖️ 트레이드오프

```
Level 2 (Materialized View):
  장점: 추가 인프라 없음, 구현 단순
  단점: 갱신 시 잠금 (CONCURRENTLY 옵션으로 완화 가능)
        실시간 반영 어려움 (갱신 주기만큼 지연)

Level 3 (CDC/트리거):
  장점: 읽기/쓰기 완전 분리
  단점: Debezium/Kafka 운영 복잡도 or 트리거 유지보수 부담

트리거 vs CDC:
  트리거: 강한 일관성 (쓰기와 같은 트랜잭션) — 단순한 경우 적합
  CDC: 약한 결합 (Kafka 비동기) — 복잡하고 확장성 필요한 경우
```

---

## 📌 핵심 정리

```
Event Sourcing 없는 CQRS 핵심:

CQRS ≠ Event Sourcing:
  CQRS: Command와 Query를 다른 모델로 처리
  Event Sourcing: 쓰기 저장 방식 선택 (독립적)

수준별 도입:
  Level 1: Query 객체 분리 (즉시 도입 가능)
  Level 2: Materialized View (DB만으로 가능)
  Level 3: CDC + 분리된 읽기 DB (Debezium/트리거)
  Level 4: Event Sourcing (필요할 때만)

적합한 경우:
  감사 로그 불필요 → Level 1~3으로 충분
  팀이 ES 학습 비용 감당 어려움 → Level 3까지
  조회 최적화만 필요 → Level 2 Materialized View
```

---

## 🤔 생각해볼 문제

**Q1.** PostgreSQL Materialized View를 `REFRESH MATERIALIZED VIEW CONCURRENTLY`로 갱신할 때 동시성 문제는 없는가?

<details>
<summary>해설 보기</summary>

`CONCURRENTLY` 옵션은 갱신 중에도 읽기가 가능하게 합니다. 갱신 중 새 버전을 별도로 구성하고 완료 후 원자적으로 교체합니다. 읽기 쿼리는 갱신 중에도 구버전 데이터를 조회하므로 차단되지 않습니다.

단, 갱신 자체는 직렬화됩니다. 두 REFRESH가 동시에 실행될 수 없어 한쪽이 대기합니다. 또한 `CONCURRENTLY`는 UNIQUE 인덱스가 필요합니다.

갱신 빈도와 데이터 지연을 균형 있게 설정하는 것이 중요합니다. 초당 갱신은 과부하, 1시간마다 갱신은 지연이 길다는 단점이 있습니다. 쓰기 빈도와 읽기 일관성 요구사항에 맞게 갱신 주기를 설정합니다.

실시간성이 중요하다면 트리거 방식이 더 적합합니다. 각 쓰기마다 즉시 Materialized View를 갱신하지 않고, 별도 테이블로 Materialized View를 대체하는 방식을 쓰면 실시간 업데이트가 가능합니다.

</details>

---

**Q2.** DB 트리거로 읽기 모델을 업데이트할 때 트리거가 실패하면 쓰기 트랜잭션도 실패하는가?

<details>
<summary>해설 보기</summary>

기본적으로 예, 트리거가 예외를 던지면 쓰기 트랜잭션 전체가 롤백됩니다. 트리거와 메인 트랜잭션이 같은 트랜잭션 컨텍스트를 공유하기 때문입니다.

이것이 트리거 방식의 큰 단점입니다. 읽기 모델 업데이트 로직의 버그(예: 참조 테이블이 없어 JOIN 실패)가 쓰기 자체를 실패시킵니다.

완화 방법이 있습니다. 트리거 내에서 `EXCEPTION` 블록으로 오류를 잡아 로그만 남기고 계속 진행할 수 있습니다. 단, 이렇게 하면 읽기 모델이 실패해도 쓰기는 성공하므로 읽기 모델이 불일치 상태가 될 수 있습니다.

트리거보다는 Spring의 `@TransactionalEventListener(phase = AFTER_COMMIT)`를 사용하는 것이 더 좋습니다. 쓰기 커밋 후 읽기 모델을 업데이트하므로, 읽기 모델 업데이트 실패가 쓰기에 영향을 주지 않습니다. 단, Eventual Consistency(짧은 지연)가 발생합니다.

</details>

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[이전: 은행 계좌 도메인 구현 ⬅️](./01-bank-account-domain.md)** | **[다음: 점진적 도입 전략 ➡️](./03-incremental-adoption.md)**

</div>
