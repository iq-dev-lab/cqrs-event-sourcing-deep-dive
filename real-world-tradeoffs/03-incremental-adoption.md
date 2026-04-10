# 점진적 도입 전략

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 기존 시스템에 CQRS를 점진적으로 적용할 때 어디서부터 시작해야 하는가?
- "가장 복잡한 조회 쿼리부터 분리하는 전략"이 왜 효과적인가?
- 읽기 모델 도입 → Command 분리 → Event Sourcing 순서로 단계별 도입하는 로드맵은 무엇인가?
- 점진적 도입에서 쓰기 모델과 읽기 모델이 일시적으로 공존하는 기간을 어떻게 관리하는가?
- 도입을 중단하거나 되돌려야 할 신호는 무엇인가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

기존 시스템에 CQRS를 처음부터 전면 도입하는 것은 현실적으로 어렵다. 서비스를 중단하거나, 대규모 리팩토링 없이 점진적으로 도입해야 한다. "가장 아픈 곳"에서 시작해 작은 성공을 쌓으며 팀의 경험을 쌓는 것이 현실적이고 안전한 전략이다.

---

## 😱 흔한 실수 (Before — 전면 도입 시도)

```
실수: "CQRS 도입하겠습니다. 전체 리팩토링."

결과:
  6개월 리팩토링 진행
  중간에 요구사항 변경 → 설계 충돌
  팀원 이탈 → 지식 손실
  비즈니스 기능 개발 중단 → 경영진 불만
  최종: CQRS 반만 도입, 기술 부채 증가

올바른 접근:
  "이 화면의 조회 쿼리가 너무 느립니다. 읽기 모델을 분리하겠습니다."
  → 2주 만에 해당 화면 성능 개선
  → 팀이 CQRS 경험
  → 다음 단계로 확장

핵심: 비즈니스 문제를 해결하면서 CQRS를 도입
      "CQRS 도입"이 목표가 되면 안 됨
```

---

## ✨ 올바른 접근 (After — 단계별 점진적 도입)

```
도입 로드맵:

Phase 1: 읽기 모델 분리 (2~4주)
  가장 느린 조회 쿼리 식별
  해당 조회용 읽기 모델 테이블 생성
  기존 쓰기가 읽기 모델도 업데이트 (이중 쓰기)
  조회를 읽기 모델로 전환
  → 즉각적인 조회 성능 개선

Phase 2: Command 분리 (4~8주)
  Service 클래스에서 Command/Query 경로 분리
  Command 객체 도입
  Command Bus 도입 (또는 단순 Service 분리)
  → 코드 명확성 향상

Phase 3: Event 기반 동기화 (4~8주)
  쓰기 이후 Domain Event 발행
  Projection이 Event를 수신해 읽기 모델 업데이트
  이중 쓰기 제거
  → 쓰기와 읽기 모델 결합도 감소

Phase 4: Event Sourcing (선택적, 3~6개월)
  감사 로그가 정말 필요한 Aggregate에만 적용
  이벤트 스토어 도입
  → 완전한 CQRS + ES
```

---

## 🔬 내부 동작 원리

### 1. Phase 1 — 읽기 모델 분리 시작점 찾기

```
시작점 선택 기준:

  ① 가장 느린 조회 쿼리 (p95 응답 시간 기준)
  ② 여러 테이블 JOIN이 많은 쿼리
  ③ 집계 연산이 많은 쿼리
  ④ 읽기 빈도가 압도적으로 높은 기능

  진단 방법:
    SELECT query, mean_exec_time, calls
    FROM pg_stat_statements
    ORDER BY mean_exec_time DESC
    LIMIT 20;
    → 가장 느린 쿼리 Top 20 확인

  첫 번째 읽기 모델 분리 후보:
    주문 목록 조회 (5개 테이블 JOIN, 평균 200ms)
    → order_summary 읽기 모델 생성 (< 5ms)

  이중 쓰기 전략 (과도기):
    쓰기 Service에서 기존 테이블 + 읽기 모델 테이블 동시 업데이트
    @Transactional: 두 업데이트 원자적 처리
    → 기존 코드 최소 변경, 읽기 모델 추가

    void createOrder(CreateOrderCommand cmd) {
        Order order = orderRepo.save(new Order(cmd));  // 기존
        orderSummaryRepo.save(buildSummary(order));     // 추가
    }
```

### 2. Phase 2 — Command 분리

```
기존 Service 분리 전략:

분리 전:
  OrderService {
    createOrder()   쓰기
    confirmOrder()  쓰기
    getOrder()      읽기
    getOrderList()  읽기
    cancelOrder()   쓰기
  }

분리 후:
  OrderCommandService {
    createOrder()    @Transactional (쓰기)
    confirmOrder()   @Transactional (쓰기)
    cancelOrder()    @Transactional (쓰기)
  }

  OrderQueryService {
    getOrder()       @Transactional(readOnly=true) (읽기)
    getOrderList()   @Transactional(readOnly=true) (읽기)
  }

점진적 분리:
  기존 OrderService는 그대로 유지
  새 OrderCommandService, OrderQueryService 생성
  Controller를 하나씩 새 Service로 전환
  기존 Service 호출이 0이 되면 삭제

Controller 이전:
  @PostMapping("/orders")  → OrderCommandService
  @GetMapping("/orders")   → OrderQueryService (읽기 모델 사용)
  @GetMapping("/orders/{id}") → OrderQueryService (읽기 모델 사용)
```

### 3. Phase 3 — Event 기반 동기화

```
이중 쓰기 제거:

이중 쓰기 문제:
  Command Service가 두 저장소에 동시 쓰기
  → 읽기 모델 업데이트 실패 시 Command Service가 롤백?
  → 쓰기 로직이 읽기 모델 구조를 알아야 함 (결합)

Event 기반으로 전환:

  Phase 3a: Spring ApplicationEvent 도입
    Command Service가 Domain Event 발행
    @TransactionalEventListener가 읽기 모델 업데이트
    → Command Service와 읽기 모델 분리

  Phase 3b: Kafka 도입 (필요 시)
    여러 읽기 모델이 같은 이벤트 소비
    다른 서비스도 이벤트 구독
    → 완전한 이벤트 기반 CQRS

코드 변화:
  Phase 2:
    createOrder() → orderRepo.save() + orderSummaryRepo.save()

  Phase 3a:
    createOrder() → orderRepo.save() + publishEvent(OrderCreated)
    @EventListener
    updateOrderSummary(OrderCreated) → orderSummaryRepo.save()

  Phase 3b:
    createOrder() → orderRepo.save() + kafka.send("order-events", OrderCreated)
    @KafkaListener
    updateOrderSummary(OrderCreated) → orderSummaryRepo.save()
```

### 4. 도입 중단 신호

```
계속하지 말아야 할 신호:

  ① 팀이 Eventual Consistency를 받아들이지 못함
     "읽기 모델이 구버전을 보여주면 사용자 불만"
     → 도메인이 강한 일관성 필요 → CQRS 부적합

  ② 이벤트 스키마 변경이 너무 잦음
     매주 이벤트 구조 변경 → Projection 재구축 반복
     → 도메인이 아직 안정화 안 됨 → Event Sourcing 미도입 권장

  ③ 팀의 생산성 급락
     CQRS 관련 디버깅에 시간 소비 > 기능 개발
     → 단계를 낮추거나 도입 속도 늦춤

  ④ 복잡도 대비 이점이 없음
     CQRS 도입 전과 성능/유지보수 차이 없음
     → 해당 도메인이 CQRS 필요 없을 수 있음

롤백 전략:
  Phase 1 (읽기 모델): 이중 쓰기 재활성화 or 읽기 모델 삭제
  Phase 2 (Command 분리): 기존 Service로 Controller 복구
  Phase 3 (Event): 이중 쓰기로 복구 후 Event 제거
  → 각 단계가 독립적으로 롤백 가능하게 설계
```

---

## 💻 실전 코드

```java
// ✅ Phase 1 — 이중 쓰기 (과도기)
@Service
@Transactional
public class OrderService {

    private final OrderRepository orderRepo;
    private final OrderSummaryRepository summaryRepo; // 읽기 모델

    public String createOrder(CreateOrderCommand cmd) {
        Order order = new Order(cmd.customerId(), cmd.items());
        Order saved = orderRepo.save(order);

        // 이중 쓰기 — 읽기 모델도 즉시 업데이트
        summaryRepo.save(OrderSummary.from(saved,
            customerService.getCustomerName(cmd.customerId())));

        return saved.getOrderId();
    }
}

// ✅ Phase 3a — Spring Event 기반 분리
@Service
@Transactional
public class OrderCommandService {

    private final OrderRepository orderRepo;
    private final ApplicationEventPublisher eventPublisher;

    public String createOrder(CreateOrderCommand cmd) {
        Order order = new Order(cmd.customerId(), cmd.items());
        Order saved = orderRepo.save(order);

        // 이벤트 발행 (읽기 모델 업데이트는 Listener에서)
        eventPublisher.publishEvent(new OrderCreated(
            saved.getOrderId(), cmd.customerId(), cmd.items(),
            saved.getTotalAmount(), Instant.now()
        ));

        return saved.getOrderId();
    }
}

@Component
public class OrderSummaryProjection {

    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void on(OrderCreated event) {
        // 쓰기 커밋 후 읽기 모델 업데이트 (별도 트랜잭션)
        String customerName = customerRepository.findNameById(event.customerId());
        summaryRepo.save(new OrderSummary(
            event.orderId(), customerName,
            "PLACED", event.totalAmount(), event.occurredAt()
        ));
    }
}

// ✅ Phase 3b — Kafka 기반 완전 분리
@Service
@Transactional
public class OrderCommandService {

    public String createOrder(CreateOrderCommand cmd) {
        Order saved = orderRepo.save(new Order(cmd));

        // Outbox에 저장 (같은 트랜잭션)
        outboxRepo.save(new OutboxEvent(
            "OrderCreated", OrderCreated.from(saved)));

        return saved.getOrderId();
    }
}

// Kafka 기반 Projection (별도 서비스 또는 동일 서비스 내)
@KafkaListener(topics = "order-events", groupId = "order-summary")
@Transactional
public void on(ConsumerRecord<String, String> record) {
    OrderCreated event = deserialize(record.value(), OrderCreated.class);
    summaryRepo.upsert(buildSummary(event));
    checkpointRepo.save("order-summary", record.offset());
}

// ✅ 단계별 전환 피처 플래그 (안전한 전환)
@Configuration
public class FeatureFlags {
    @Value("${feature.order.use-read-model:false}")
    private boolean useReadModel;
}

@RestController
public class OrderController {

    @GetMapping("/orders")
    public List<OrderView> getOrders() {
        if (featureFlags.isUseReadModel()) {
            return orderQueryService.getFromReadModel(); // 새 경로
        } else {
            return orderService.getOrders(); // 기존 경로
        }
    }
}
```

---

## 📊 패턴 비교

```
도입 단계별 효과와 비용:

┌──────────┬──────────────────┬──────────────┬────────────────┐
│ Phase    │ 주요 효과        │ 소요 시간    │ 위험도         │
├──────────┼──────────────────┼──────────────┼────────────────┤
│ 1. 읽기  │ 조회 성능 개선   │ 2~4주        │ 낮음           │
│ 모델 분리│ (즉각적 효과)    │              │ (롤백 쉬움)    │
├──────────┼──────────────────┼──────────────┼────────────────┤
│ 2. 커맨드│ 코드 명확성 향상 │ 4~8주        │ 낮음           │
│ 분리     │                  │              │                │
├──────────┼──────────────────┼──────────────┼────────────────┤
│ 3. 이벤트│ 느슨한 결합      │ 4~8주        │ 중간           │
│ 기반     │ 다중 읽기 모델   │              │                │
├──────────┼──────────────────┼──────────────┼────────────────┤
│ 4. ES    │ 감사 로그, 시간  │ 3~6개월      │ 높음           │
│          │ 여행, 재구축     │              │                │
└──────────┴──────────────────┴──────────────┴────────────────┘
```

---

## ⚖️ 트레이드오프

```
점진적 도입의 장점:
  각 단계에서 즉각적인 비즈니스 가치 확인
  팀이 학습하며 다음 단계 결정
  롤백 가능 (각 단계가 독립적)

점진적 도입의 단점:
  과도기 동안 이중 코드 (이중 쓰기, 두 Service)
  팀원 간 "어느 방식이 정답인가" 혼란 가능
  → 명확한 전환 일정과 완료 기준 필요
```

---

## 📌 핵심 정리

```
점진적 도입 핵심:

시작점:
  가장 느린 조회 쿼리 or 가장 복잡한 JOIN
  → 읽기 모델 분리 → 즉각적 성능 개선

단계:
  Phase 1: 읽기 모델 분리 (이중 쓰기)
  Phase 2: Command/Query Service 분리
  Phase 3: Event 기반 동기화 (Spring or Kafka)
  Phase 4: Event Sourcing (선택, 필요한 Aggregate만)

안전망:
  피처 플래그로 기존/신규 경로 전환
  각 단계 롤백 계획 수립
  단계별 완료 기준 명확히

중단 신호:
  Eventual Consistency 수용 불가
  이벤트 스키마 변경 빈번 (도메인 불안정)
  복잡도 대비 이점 없음
```

---

## 🤔 생각해볼 문제

**Q1.** Phase 1(이중 쓰기) 단계에서 쓰기 트랜잭션이 성공하고 읽기 모델 업데이트가 실패하면 어떻게 처리하는가?

<details>
<summary>해설 보기</summary>

이것이 이중 쓰기의 핵심 문제입니다. 두 가지 선택이 있습니다.

같은 트랜잭션으로 처리하면(기본 `@Transactional`), 읽기 모델 업데이트 실패 시 쓰기 트랜잭션 전체가 롤백됩니다. 강한 일관성이 보장되지만, 읽기 모델 업데이트 실패가 쓰기를 실패시킨다는 단점이 있습니다.

`@TransactionalEventListener(phase = AFTER_COMMIT)`로 처리하면, 쓰기 커밋 후 읽기 모델을 업데이트합니다. 쓰기는 성공하지만 읽기 모델 업데이트가 실패할 수 있습니다. 이 경우 알람을 발생시키고 수동으로 동기화하거나, 재시도 로직을 추가합니다.

Phase 1의 이중 쓰기는 임시 방편임을 인식하고, Phase 3(Event 기반)으로 빠르게 전환하는 것이 좋습니다. Event 기반으로 전환하면 Outbox 패턴으로 이 문제를 더 우아하게 해결할 수 있습니다.

</details>

---

**Q2.** 점진적 도입 중 팀의 일부는 기존 방식으로, 일부는 새 CQRS 방식으로 개발한다. 코드 혼재를 어떻게 관리하는가?

<details>
<summary>해설 보기</summary>

명확한 가이드라인과 패키지 구조로 관리합니다.

패키지를 분리합니다. 기존 코드는 `com.example.order.service`에, 새 CQRS 코드는 `com.example.order.command`, `com.example.order.query`에 둡니다. 신규 기능은 반드시 새 구조로 개발하고, 기존 기능은 점진적으로 이전합니다.

팀 내 합의된 "완료 기준"을 정합니다. 예를 들어 "분기 내 주문 도메인 전체 CQRS 전환 완료"처럼 명확한 목표를 설정합니다.

ADR(Architecture Decision Record)을 작성합니다. 왜 CQRS를 도입하는지, 현재 어느 단계인지, 다음 단계는 무엇인지를 문서화합니다. 신입 팀원이 합류해도 방향을 이해할 수 있어야 합니다.

코드 리뷰에서 강제합니다. 기존 Service에 새 읽기 로직을 추가하는 PR은 거절하고, 새 Query Service로 분리하도록 요구합니다. 점진적이라도 일관된 방향으로 이동해야 합니다.

</details>

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[이전: Event Sourcing 없는 CQRS ⬅️](./02-cqrs-without-event-sourcing.md)** | **[다음: 안티패턴 ➡️](./04-anti-patterns.md)**

</div>
