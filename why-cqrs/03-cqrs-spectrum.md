# CQRS의 스펙트럼

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- CQRS를 "패키지 분리"로만 적용하면 무엇이 달라지고 무엇이 달라지지 않는가?
- Simple CQRS, 저장소 분리 CQRS, Event-Driven CQRS는 각각 어떤 문제를 해결하는가?
- 세 수준 중 어느 수준에서 진입해야 하는지 어떻게 판단하는가?
- 저장소를 분리하지 않고 읽기 모델을 분리하는 것이 의미가 있는가?
- Event Sourcing은 CQRS의 필수 조건인가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

"CQRS를 도입하겠습니다"라는 결정이 내려지면 팀에서 즉시 물어봐야 할 것이 있다. "어느 수준의 CQRS인가?" CQRS는 단일 수준의 패턴이 아니라 스펙트럼이다. 가장 단순한 형태는 같은 DB, 같은 테이블을 쓰면서 Command Handler와 Query Handler를 분리하는 것이고, 가장 복잡한 형태는 완전히 다른 DB와 Event Sourcing을 결합하는 것이다.

잘못된 수준을 선택하면 두 가지 방향으로 실패한다. 너무 단순한 수준을 선택하면 분리 비용만 지불하고 아무 이점도 얻지 못한다. 너무 복잡한 수준을 선택하면 팀이 운영 복잡도에 압도되어 도메인 로직보다 인프라 관리에 시간을 쏟게 된다.

---

## 😱 흔한 실수 (Before — CQRS를 패키지 분리로만 이해)

```
흔한 "CQRS 도입":

  패키지 구조:
    com.example.order
      ├── command/
      │     ├── CreateOrderCommand.java
      │     ├── ConfirmOrderCommand.java
      │     └── OrderCommandHandler.java
      └── query/
            ├── GetOrderQuery.java
            └── OrderQueryHandler.java

  하지만 내부:
    // OrderCommandHandler
    @Transactional
    public void handle(ConfirmOrderCommand command) {
        Order order = orderRepository.findById(command.getOrderId()); // ← 같은 Repository
        order.confirm();
        orderRepository.save(order); // ← 같은 테이블
    }

    // OrderQueryHandler
    public OrderDetailDto handle(GetOrderQuery query) {
        Order order = orderRepository.findById(query.getOrderId()); // ← 같은 Entity
        return OrderDetailDto.from(order); // ← 같은 테이블에서 읽음
    }

결과:
  패키지만 나뉘고 실제로는 아무것도 달라지지 않음
  N+1 문제: 여전히 발생
  잠금 충돌: 여전히 발생
  읽기 성능 최적화: 불가 (같은 Entity)
  독립적 스케일 아웃: 불가 (같은 DB)

  오히려 부작용:
    코드 경로가 늘어나 복잡도만 증가
    "CQRS 도입했는데 왜 안 좋아졌지?" → 잘못된 수준 선택
```

---

## ✨ 올바른 접근 (After — 목적에 맞는 수준 선택)

```
수준 1: Simple CQRS — 같은 DB, 다른 조회 전략
  쓰기: JPA Entity + Repository
  읽기: JPQL/QueryDSL DTO 프로젝션 or Native Query
  → 같은 테이블, 다른 조회 방식
  → N+1 해결 가능, 조회 쿼리 최적화 가능
  → 잠금 충돌 해결 불가, 저장소 다양화 불가

수준 2: 저장소 분리 CQRS — 다른 DB, 읽기 모델 테이블
  쓰기: PostgreSQL (정규화)
  읽기: PostgreSQL 별도 스키마 or MySQL or Redis
  → 읽기 모델 비정규화 가능
  → 독립적 스케일 아웃 가능
  → 동기화 메커니즘 필요 (이벤트, CDC, 배치)

수준 3: Event-Driven CQRS — 이벤트로 동기화
  쓰기: Event Sourcing (이벤트 스토어)
  읽기: 목적별 다양한 저장소 (RDB + Redis + Elasticsearch)
  동기화: Kafka 이벤트 기반 Projection
  → 완전한 분리, 읽기 모델 재구축 가능
  → 시간 여행, 감사 로그 완벽 지원
  → 팀 학습 비용 높음, 운영 복잡도 높음
```

---

## 🔬 내부 동작 원리

### 1. 수준 1 — Simple CQRS (같은 DB, DTO 프로젝션)

```
구조:
  ┌──────────────────────────────────────────────────────┐
  │                   애플리케이션                         │
  │                                                      │
  │  Command Side            Query Side                  │
  │  ┌───────────────┐       ┌───────────────────────┐   │
  │  │ Command       │       │ Query Handler          │   │
  │  │ Handler       │       │                        │   │
  │  │  ↓            │       │ JPQL DTO 프로젝션       │   │
  │  │ JPA Entity    │       │ or QueryDSL            │   │
  │  │  ↓            │       │ or Native Query        │   │
  │  │ Repository    │       │  ↓                     │   │
  │  └───────┬───────┘       └──────────┬─────────────┘   │
  │          │                          │                 │
  └──────────┼──────────────────────────┼─────────────────┘
             │                          │
             ▼                          ▼
         ┌────────────────────────────────┐
         │       단일 PostgreSQL DB        │
         │  orders, order_lines, customers │
         └────────────────────────────────┘

쓰기: JPA Entity로 상태 변경, 불변식 보호
읽기: JPQL DTO 프로젝션으로 필요한 컬럼만 조회

  // 읽기 — DTO 프로젝션으로 N+1 없는 조회
  @Query("""
      SELECT new com.example.OrderSummaryDto(
          o.id, c.name, o.status,
          SUM(ol.quantity * ol.unitPrice),
          COUNT(ol.id),
          a.city
      )
      FROM Order o
      JOIN Customer c ON o.customerId = c.id
      JOIN Address a ON c.addressId = a.id
      JOIN OrderLine ol ON ol.orderId = o.id
      GROUP BY o.id, c.name, o.status, a.city
      ORDER BY o.createdAt DESC
  """)
  Page<OrderSummaryDto> findOrderSummaries(Pageable pageable);

이 수준의 해결:
  ✅ N+1 문제 — JPQL DTO 프로젝션으로 해결
  ✅ Entity 오염 — 조회 로직이 Query 쪽에만 존재
  ❌ 잠금 충돌 — 같은 테이블 공유
  ❌ 독립적 스케일 아웃 — 같은 DB
  ❌ 저장소 다양화 — RDB 외 사용 불가
  ❌ 복잡한 집계 — GROUP BY가 쿼리에 남음
```

### 2. 수준 2 — 저장소 분리 CQRS

```
구조:
  ┌──────────────────────────────────────────────────────────┐
  │                       애플리케이션                         │
  │                                                          │
  │  Command Side                  Query Side               │
  │  ┌───────────────┐             ┌─────────────────────┐   │
  │  │ Command       │  이벤트/CDC  │ Query Handler        │   │
  │  │ Handler       ├────────────►│ 읽기 모델 Repository │   │
  │  │  ↓            │             │  ↓                  │   │
  │  │ Aggregate     │             │ OrderSummaryView     │   │
  │  │  ↓            │             │ TransactionHistory   │   │
  │  └───────┬───────┘             └──────────┬───────────┘   │
  │          │                               │               │
  └──────────┼───────────────────────────────┼───────────────┘
             │                               │
             ▼                               ▼
     ┌───────────────┐              ┌──────────────────┐
     │ Write DB      │              │ Read DB          │
     │ PostgreSQL    │              │ PostgreSQL       │
     │ (정규화)       │              │ (비정규화 뷰)     │
     └───────────────┘              └──────────────────┘

동기화 방식 1 — 이벤트 기반 (권장):
  쓰기 트랜잭션 완료
    → OrderConfirmed 이벤트 발행 (Kafka)
    → Projection이 이벤트 소비
    → 읽기 DB 업데이트
  지연: 수십 ms ~ 수 초 (최종 일관성)

동기화 방식 2 — Debezium CDC:
  PostgreSQL WAL(Write Ahead Log) 변경 감지
    → Debezium이 변경 이벤트 Kafka 발행
    → 읽기 DB 업데이트
  특징: 애플리케이션 코드 변경 없음, 신뢰성 높음

동기화 방식 3 — 배치:
  주기적으로 Write DB → Read DB 동기화
  지연: 배치 주기만큼 (수 분 ~ 수 시간)
  적합: 통계 대시보드 등 실시간성 불필요한 경우

이 수준의 해결:
  ✅ N+1 문제
  ✅ Entity 오염
  ✅ 잠금 충돌 — 다른 테이블로 분리
  ✅ 독립적 스케일 아웃 — 읽기 DB 독립 확장
  ✅ 저장소 다양화 — 읽기에 Redis, ES 적용 가능
  ✅ 비정규화 읽기 모델
  ⚠️ 최종 일관성 수락 필요
  ⚠️ 동기화 로직 관리 필요
```

### 3. 수준 3 — Event-Driven CQRS + Event Sourcing

```
구조:
  ┌─────────────────────────────────────────────────────────────────┐
  │                          애플리케이션                             │
  │                                                                 │
  │  Command Side                          Query Side              │
  │  ┌──────────────────────┐              ┌────────────────────┐   │
  │  │ Command Handler      │              │ Query Handler       │   │
  │  │  ↓                   │              │  ↓                 │   │
  │  │ Aggregate            │              │ 목적별 읽기 모델     │   │
  │  │  ↓                   │              │ - RDB (목록/상세)  │   │
  │  │ 이벤트 발행           │              │ - Redis (캐시)     │   │
  │  └───────────┬──────────┘              │ - ES (검색)        │   │
  │              │                         └─────────┬──────────┘   │
  └──────────────┼──────────────────────────────────┼───────────────┘
                 │                                  │
                 ▼                                  ▲
     ┌──────────────────┐    ┌──────┐    ┌──────────────────────┐
     │ Event Store      ├───►│Kafka ├───►│ Projection           │
     │ (Axon / PG)      │    │      │    │ (이벤트 소비 + 읽기   │
     │ Append Only      │    └──────┘    │  모델 업데이트)       │
     └──────────────────┘               └──────────────────────┘

이벤트 스토어 내용:
  stream: "account-ACC-001"
  ┌────────────┬─────────┬─────────────────┬─────────────────────────────┐
  │ version    │ type    │ payload          │ occurred_at                 │
  ├────────────┼─────────┼─────────────────┼─────────────────────────────┤
  │ 1          │ AccountOpened  │ {owner, initial} │ 2024-01-01T09:00  │
  │ 2          │ MoneyDeposited │ {amount: 500000}  │ 2024-01-01T10:00  │
  │ 3          │ MoneyWithdrawn │ {amount: 100000}  │ 2024-01-02T14:00  │
  │ 4          │ MoneyTransferred│ {to, amount: 50000}│ 2024-01-03T11:00 │
  └────────────┴─────────┴─────────────────┴─────────────────────────────┘

이 수준의 해결:
  ✅ 위 수준의 모든 해결
  ✅ 완전한 감사 로그 (모든 이벤트 영구 보존)
  ✅ 시간 여행 (과거 시점 상태 재현)
  ✅ 읽기 모델 재구축 (새 뷰 필요 시 과거 이벤트부터 재구축)
  ✅ 디버깅 (이벤트 시퀀스로 이슈 재현)
  ⚠️ 팀 학습 비용 높음
  ⚠️ 운영 복잡도 가장 높음
  ⚠️ 이벤트 스키마 진화 어려움
```

### 4. Event Sourcing은 CQRS의 필수 조건인가

```
답: 아니다. 둘은 독립적인 패턴이다.

CQRS without Event Sourcing (수준 2):
  쓰기: 현재 상태를 관계형 DB에 저장 (UPDATE account SET balance = ?)
  읽기: 별도 읽기 모델 테이블
  동기화: CDC(Debezium) or 배치
  → 충분히 많은 CQRS 이점을 얻음
  → 이벤트 스토어 관리 불필요

Event Sourcing without CQRS (드문 경우):
  이벤트를 저장하되, 읽기도 이벤트 스토어에서 리플레이
  → 모든 조회에 이벤트 리플레이 비용 발생
  → 복잡한 조회 쿼리 불가 (이벤트 스트림은 검색에 취약)
  → 일반적으로 비현실적

CQRS + Event Sourcing이 자연스럽게 결합되는 이유:
  Event Sourcing: "이벤트를 저장"
  CQRS: "읽기를 위해 이벤트를 Projection으로 변환"
  → 이벤트가 쓰기 모델의 변경을 나타냄
  → 그 이벤트를 소비해 읽기 모델을 만드는 것이 CQRS
  → 두 패턴이 서로를 완성하는 구조

결론:
  CQRS 도입 ≠ Event Sourcing 도입 (별개 결정)
  도메인이 CQRS만 필요하면 수준 2로 충분
  완전한 감사 로그, 시간 여행이 필요하면 Event Sourcing 추가
```

---

## 💻 실전 코드

### 수준별 읽기 쿼리 비교

```java
// ==============================
// 수준 1: Simple CQRS
// ==============================

// 쓰기: JPA Entity 그대로
@Transactional
public void handle(ConfirmOrderCommand command) {
    Order order = orderRepository.findById(command.getOrderId())
        .orElseThrow(() -> new OrderNotFoundException(command.getOrderId()));
    order.confirm(inventoryChecker);
    orderRepository.save(order);
}

// 읽기: JPQL DTO 프로젝션 — 같은 DB, N+1 없이
public interface OrderSummaryProjection {
    String getOrderId();
    String getCustomerName();
    String getStatus();
    BigDecimal getTotalAmount();
    Integer getItemCount();
}

// Spring Data JPA Interface Projection
public interface OrderQueryRepository extends JpaRepository<Order, String> {

    @Query("""
        SELECT o.id as orderId, c.name as customerName, o.status,
               SUM(ol.quantity * ol.unitPrice) as totalAmount,
               COUNT(ol.id) as itemCount
        FROM Order o
        JOIN Customer c ON o.customerId = c.id
        JOIN OrderLine ol ON ol.orderId = o.id
        GROUP BY o.id, c.name, o.status
        ORDER BY o.createdAt DESC
    """)
    List<OrderSummaryProjection> findOrderSummaries();
}

// ==============================
// 수준 2: 저장소 분리 CQRS
// ==============================

// 쓰기: 이벤트 발행 포함
@Transactional
public void handle(ConfirmOrderCommand command) {
    Order order = orderRepository.findById(command.getOrderId())
        .orElseThrow();
    order.confirm(inventoryChecker);
    orderRepository.save(order);

    // 이벤트 발행 — 읽기 모델 동기화 트리거
    // (또는 Order.confirm() 내부에서 Domain Event로 발행)
    eventPublisher.publish(new OrderConfirmedEvent(
        order.getId(), order.getCustomerId(),
        order.calculateTotal(), LocalDateTime.now()
    ));
}

// 읽기 모델 저장소 (별도 테이블)
@Repository
public interface OrderSummaryRepository extends JpaRepository<OrderSummaryView, String> {
    Page<OrderSummaryView> findAllByOrderByCreatedAtDesc(Pageable pageable);
    List<OrderSummaryView> findByStatus(String status);
    List<OrderSummaryView> findByDeliveryCity(String city);
}

// Projection — 이벤트 소비, 읽기 모델 업데이트
@Component
public class OrderSummaryProjection {

    @KafkaListener(topics = "order-events", groupId = "order-summary-projection")
    public void on(OrderConfirmedEvent event) {
        Customer customer = customerQueryService.getById(event.getCustomerId());

        OrderSummaryView view = new OrderSummaryView();
        view.setOrderId(event.getOrderId());
        view.setCustomerName(customer.getName());    // 비정규화
        view.setCustomerTier(customer.getTier());    // 비정규화
        view.setStatus("CONFIRMED");
        view.setTotalAmount(event.getTotalAmount());
        view.setCreatedAt(event.getOccurredAt());

        orderSummaryRepository.save(view);
    }
}

// 읽기 서비스 — 읽기 모델만 조회 (JOIN 없음)
@Service
public class OrderQueryService {

    public Page<OrderSummaryView> getOrderSummaries(int page, int size) {
        // 단순 SELECT, 조인 없음, 비정규화 완료
        return orderSummaryRepository.findAllByOrderByCreatedAtDesc(
            PageRequest.of(page, size)
        );
    }
}

// ==============================
// 수준 3: Event-Driven CQRS (Axon 기반)
// ==============================

// 쓰기: Axon Aggregate
@Aggregate
public class Order {

    @AggregateIdentifier
    private OrderId orderId;
    private OrderStatus status;
    private List<OrderLine> lines;

    @CommandHandler
    public void handle(ConfirmOrderCommand command, InventoryChecker checker) {
        validateCanConfirm();
        validateStock(checker);
        // 이벤트 발행 — 이벤트 스토어에 저장됨
        apply(new OrderConfirmed(this.orderId, this.customerId,
                                  this.lines, calculateTotal()));
    }

    @EventSourcingHandler
    public void on(OrderConfirmed event) {
        this.status = OrderStatus.CONFIRMED;
        // 쓰기 모델 상태 갱신 (읽기 모델과 무관)
    }
}

// 읽기: Axon Query Handler
@Component
public class OrderQueryHandler {

    @QueryHandler
    public Page<OrderSummaryView> handle(GetOrderSummariesQuery query) {
        return orderSummaryRepository.findAllByOrderByCreatedAtDesc(
            PageRequest.of(query.getPage(), query.getSize())
        );
    }

    @QueryHandler
    public OrderDetailView handle(GetOrderDetailQuery query) {
        return orderDetailRepository.findById(query.getOrderId())
            .orElseThrow(() -> new OrderNotFoundException(query.getOrderId()));
    }
}
```

---

## 📊 패턴 비교

```
CQRS 수준별 특성 비교:

┌─────────────────────┬────────────────┬────────────────┬────────────────────┐
│ 항목                │ Simple CQRS    │ 저장소 분리     │ Event-Driven CQRS  │
├─────────────────────┼────────────────┼────────────────┼────────────────────┤
│ N+1 해결            │ ✅             │ ✅             │ ✅                 │
├─────────────────────┼────────────────┼────────────────┼────────────────────┤
│ 읽기 모델 비정규화   │ ❌             │ ✅             │ ✅                 │
├─────────────────────┼────────────────┼────────────────┼────────────────────┤
│ 잠금 충돌 해결       │ ❌             │ ✅             │ ✅                 │
├─────────────────────┼────────────────┼────────────────┼────────────────────┤
│ 독립적 스케일 아웃   │ ❌             │ ✅             │ ✅                 │
├─────────────────────┼────────────────┼────────────────┼────────────────────┤
│ 저장소 다양화        │ ❌             │ ✅             │ ✅                 │
├─────────────────────┼────────────────┼────────────────┼────────────────────┤
│ 감사 로그 완전성     │ ❌             │ △ (CDC)       │ ✅ (이벤트 스토어)  │
├─────────────────────┼────────────────┼────────────────┼────────────────────┤
│ 시간 여행            │ ❌             │ ❌             │ ✅                 │
├─────────────────────┼────────────────┼────────────────┼────────────────────┤
│ 읽기 모델 재구축     │ ❌             │ △ (재생성)     │ ✅ (이벤트 리플레이) │
├─────────────────────┼────────────────┼────────────────┼────────────────────┤
│ 최종 일관성 필요     │ ❌             │ ✅             │ ✅                 │
├─────────────────────┼────────────────┼────────────────┼────────────────────┤
│ 팀 학습 비용         │ 낮음           │ 중간           │ 높음               │
├─────────────────────┼────────────────┼────────────────┼────────────────────┤
│ 운영 복잡도          │ 낮음           │ 중간           │ 높음               │
└─────────────────────┴────────────────┴────────────────┴────────────────────┘

적합한 도입 신호:
  Simple CQRS:     N+1 문제가 있고 쿼리 최적화가 필요한 경우
  저장소 분리:     읽기-쓰기 부하가 불균형하거나 저장소를 달리해야 하는 경우
  Event-Driven:    완전한 감사 로그, 시간 여행, 읽기 모델 재구축이 필요한 경우
```

---

## ⚖️ 트레이드오프

```
수준 선택의 핵심 원칙:
  "현재 겪는 구체적인 문제를 해결하는 최소한의 수준을 선택"

Simple CQRS의 함정:
  N+1만 해결하고 싶었는데
  → QueryDSL DTO 프로젝션만으로 충분할 수도 있음
  → CQRS 이름을 붙일 필요도 없음

저장소 분리의 함정:
  읽기 부하를 줄이고 싶어서 도입했는데
  → 동기화 로직이 생각보다 복잡함
  → CDC 구축, Projection 구현, 읽기 모델 재구축 전략 필요
  → 팀이 이 복잡도를 감당할 수 있는가?

Event-Driven의 함정:
  "최신 아키텍처니까 도입하자"
  → 이벤트 스키마 진화 문제
  → 이벤트 스토어 운영
  → 팀 전체가 Eventual Consistency를 이해해야 함
  → 도메인이 단순하면 오버엔지니어링

점진적 접근 권장:
  1단계: Simple CQRS (QueryDSL DTO 프로젝션)
  2단계: 특정 조회에만 읽기 모델 분리 (가장 복잡한 쿼리부터)
  3단계: 도메인이 정말 필요할 때 Event Sourcing 추가
```

---

## 📌 핵심 정리

```
CQRS 스펙트럼:

수준 1 — Simple CQRS:
  같은 DB, 같은 테이블
  Command: JPA Entity + Repository
  Query: JPQL DTO 프로젝션 or QueryDSL
  해결: N+1, Entity 오염
  미해결: 잠금 충돌, 저장소 다양화

수준 2 — 저장소 분리 CQRS:
  다른 DB 또는 다른 테이블 스키마
  Command: 쓰기 DB (정규화)
  Query: 읽기 DB (비정규화 뷰)
  동기화: 이벤트, CDC, 배치
  해결: 위 + 잠금 충돌, 스케일 아웃, 비정규화

수준 3 — Event-Driven CQRS:
  이벤트 스토어 + 다양한 읽기 저장소
  Command: Aggregate → 이벤트 스토어
  Query: Projection → 목적별 읽기 모델
  해결: 위 + 감사 로그, 시간 여행, 재구축

핵심 판단:
  Event Sourcing ≠ CQRS 필수
  현재 문제를 해결하는 최소 수준 선택
  점진적 도입이 안전함
```

---

## 🤔 생각해볼 문제

**Q1.** 수준 2(저장소 분리)에서 동기화 실패로 읽기 모델과 쓰기 모델이 불일치한다면 어떻게 감지하고 복구하는가?

<details>
<summary>해설 보기</summary>

감지 방법으로는 두 가지가 있습니다. 주기적 검증 배치로 Write DB와 Read DB의 특정 필드(잔고, 주문 상태 등)를 샘플링 비교하고, 불일치 발견 시 알람을 발생시킵니다. Projection 지연 모니터링으로 이벤트 발행 시점과 읽기 모델 반영 시점의 차이를 메트릭으로 추적하여 임계값(예: 30초) 초과 시 알람을 발생시킵니다.

복구 방법으로는 이벤트 기반 동기화의 경우 실패한 이벤트를 Dead Letter Queue에서 재처리합니다. Event Sourcing을 사용한다면 이벤트 스토어의 특정 시점부터 읽기 모델을 재구축(Projection Rebuild)합니다. CDC(Debezium) 방식은 Kafka의 WAL 오프셋을 리셋하여 재처리합니다.

장기 불일치 방지를 위해 읽기 모델에 `last_updated_at`과 `version` 필드를 두고, 특정 시간 이상 업데이트되지 않은 레코드를 모니터링하는 것이 좋습니다.

</details>

---

**Q2.** Simple CQRS를 선택했을 때 미래에 저장소 분리(수준 2)로 업그레이드하는 경로가 있는가? 어떤 설계 결정이 이 경로를 열어두는가?

<details>
<summary>해설 보기</summary>

미래 업그레이드 경로를 열어두는 설계 결정들이 있습니다.

Command Handler와 Query Handler를 처음부터 분리하는 것이 중요합니다. 패키지 분리만이라도 Command 처리 경로와 Query 처리 경로를 별도로 유지하면, 나중에 Query Handler가 다른 저장소를 사용하도록 변경하기 쉽습니다.

Repository 인터페이스를 분리하는 것도 도움이 됩니다. `OrderRepository` (쓰기)와 `OrderQueryRepository` (읽기)를 처음부터 분리하면, 나중에 `OrderQueryRepository` 구현체를 읽기 모델 테이블을 바라보도록 교체하기 쉽습니다.

Domain Event 발행 준비를 해두는 것도 좋습니다. Command Handler에서 Domain Event를 발행하는 코드를 처음부터 작성해두면(Spring Application Event로 시작), 나중에 Kafka나 외부 메시지 브로커로 교체하기 쉽습니다.

가장 피해야 할 패턴은 Service 계층에서 Command와 Query가 같은 메서드에서 처리되는 것입니다. 이 경우 분리 비용이 훨씬 커집니다.

</details>

---

**Q3.** 한 팀은 수준 3(Event-Driven CQRS)을 도입했고 다른 팀은 수준 1(Simple CQRS)을 유지했다. 6개월 후 각 팀의 상황이 어떻게 달라질 수 있는가?

<details>
<summary>해설 보기</summary>

도메인 복잡도와 팀 역량에 따라 결과가 달라집니다.

도메인이 복잡하고(금융, 주문 처리) 팀이 충분히 숙련된 경우라면, 수준 3 팀은 6개월 후 감사 로그 요구사항이 생겼을 때 이벤트 스토어가 이미 모든 이력을 보존하고 있어 추가 개발이 없고, 새로운 통계 읽기 모델이 필요할 때 과거 이벤트를 재구축하는 것으로 완성됩니다. 반면 수준 1 팀은 감사 로그를 위해 히스토리 테이블을 별도로 만들어야 하고, 새 통계는 과거 데이터 없이 도입 시점부터만 집계됩니다.

도메인이 단순하거나 팀이 Event Sourcing에 익숙하지 않은 경우라면, 수준 3 팀은 이벤트 스키마 변경 문제, Projection 재구축, 이벤트 스토어 운영 부담으로 실제 도메인 개발이 느려지고, 수준 1 팀은 빠르게 기능을 추가하고 있습니다.

결론적으로 "수준 3이 항상 좋다"는 없습니다. 도메인의 본질적 복잡도와 팀 역량이 선택을 결정합니다.

</details>

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[이전: 쓰기 모델과 읽기 모델의 근본적 차이 ⬅️](./02-write-read-model-difference.md)** | **[다음: CQRS가 해결하는 실제 문제 ➡️](./04-problems-cqrs-solves.md)**

</div>
