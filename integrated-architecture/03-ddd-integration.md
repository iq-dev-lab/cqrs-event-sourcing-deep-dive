# DDD와의 통합

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Aggregate, Domain Event, Repository, Saga가 CQRS + ES에서 어떻게 협력하는가?
- Saga가 여러 Aggregate를 조율하는 방식은 어떻게 구현하는가?
- Bounded Context 간 이벤트 통합에서 내부 이벤트와 외부 이벤트를 왜 분리해야 하는가?
- Anti-Corruption Layer는 CQRS + ES 환경에서 어떤 역할을 하는가?
- DDD 전술 설계 요소들이 CQRS + ES 아키텍처에서 어디에 위치하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

CQRS와 Event Sourcing은 DDD와 자연스럽게 결합된다. Aggregate가 이벤트를 생성하고, 이벤트 스토어가 저장하고, Projection이 읽기 모델을 만들고, Saga가 여러 Aggregate를 조율하는 구조는 DDD의 전술 설계 요소들을 CQRS + ES로 구현하는 방식이다. 이 관계를 명확히 이해해야 복잡한 도메인을 올바른 설계로 표현할 수 있다.

---

## 😱 흔한 실수 (Before — DDD 없이 CQRS + ES)

```
실수 1: Aggregate 경계 없이 이벤트 발행

  // 하나의 Command가 여러 Aggregate 수정 + 이벤트 발행
  @CommandHandler
  @Transactional
  void handle(CompleteOrderCommand cmd) {
      Order order = orderRepo.findById(cmd.orderId());
      order.complete();

      Inventory inventory = inventoryRepo.findById(cmd.productId());
      inventory.decrease(order.getQuantity()); // 다른 Aggregate 수정

      Payment payment = paymentRepo.findById(cmd.paymentId());
      payment.confirm(); // 또 다른 Aggregate 수정

      // 이벤트 3개 발행
      // → Aggregate 경계 위반, 트랜잭션 범위 과다
  }

실수 2: 내부 이벤트를 외부에 그대로 노출

  // OrderService 내부 구현 이벤트
  OrderItemStockChecked { orderId, productId, stockLevel, warehouseId, ... }

  // 외부 서비스(배송, 회계)에 그대로 발행
  → 내부 구현 세부사항이 외부에 노출
  → OrderService 리팩토링 시 외부 서비스도 수정 필요

실수 3: Saga 없이 분산 트랜잭션 시도

  // 주문 + 재고 + 결제를 하나의 DB 트랜잭션으로
  // (마이크로서비스 환경에서 불가능)
  → 2PC, XA 트랜잭션 → 성능 저하, 가용성 감소
```

---

## ✨ 올바른 접근 (After — DDD + CQRS + ES 완전한 통합)

```
DDD 전술 설계 요소들의 위치:

  Aggregate:
    불변식 보호 경계
    Command를 받아 이벤트 생성
    Event Sourcing에서 이벤트 리플레이로 재구성

  Domain Event:
    Aggregate 상태 변화의 사실 표현
    이벤트 스토어에 저장
    Projection과 Saga의 트리거

  Repository:
    Aggregate의 영속화 추상화
    Event Sourcing: 이벤트 스토어에서 로드/저장

  Saga:
    여러 Aggregate에 걸친 프로세스 조율
    이벤트를 듣고 Command 발행
    보상 트랜잭션으로 실패 처리

  Anti-Corruption Layer:
    외부 Bounded Context의 이벤트를 내부 도메인 언어로 변환
```

---

## 🔬 내부 동작 원리

### 1. Aggregate + Domain Event + Event Sourcing

```
DDD Aggregate의 세 가지 역할:

역할 1: 일관성 경계
  Account Aggregate:
    {id, balance, status}
    불변식: balance >= 0, 동결 계좌 출금 불가
    트랜잭션 경계 = Aggregate 하나

역할 2: Command 처리
  account.withdraw(amount) →
    검증 + 이벤트 생성 (MoneyWithdrawn)

역할 3: 이벤트 리플레이로 재구성 (Event Sourcing)
  apply(AccountOpened)  → balance = 0
  apply(MoneyDeposited) → balance += amount
  apply(MoneyWithdrawn) → balance -= amount

Domain Event의 역할:

  내부적 역할 (Event Sourcing):
    Aggregate 상태 변화의 기록
    리플레이의 원료

  외부적 역할 (통합):
    Saga 트리거: "주문이 확인됐다" → 재고 차감 Saga 시작
    Projection 트리거: "잔고 업데이트 해라"
    외부 서비스 통합: "다른 Bounded Context에 알리기"
```

### 2. Saga 패턴 — 여러 Aggregate 조율

```
주문 처리 Saga (주문 + 재고 + 결제):

  단계:
    1. OrderPlaced 이벤트 → Saga 시작
    2. Saga → ReserveInventoryCommand 발행
    3. InventoryReserved 이벤트 → Saga 진행
    4. Saga → ProcessPaymentCommand 발행
    5. PaymentProcessed 이벤트 → Saga 완료
    6. Saga → ConfirmOrderCommand 발행

  실패 시 보상:
    PaymentFailed 이벤트
    → Saga → ReleaseInventoryCommand 발행 (보상)
    → Saga → CancelOrderCommand 발행 (보상)

Saga 구현 (Axon Saga):

  @Saga
  public class OrderProcessingSaga {

      private String orderId;
      private String inventoryReservationId;

      @StartSaga
      @SagaEventHandler(associationProperty = "orderId")
      public void on(OrderPlaced event) {
          this.orderId = event.orderId();
          commandGateway.send(new ReserveInventoryCommand(
              event.orderId(), event.items()
          ));
      }

      @SagaEventHandler(associationProperty = "orderId")
      public void on(InventoryReserved event) {
          this.inventoryReservationId = event.reservationId();
          commandGateway.send(new ProcessPaymentCommand(
              orderId, event.totalAmount()
          ));
      }

      @SagaEventHandler(associationProperty = "orderId")
      public void on(PaymentProcessed event) {
          commandGateway.send(new ConfirmOrderCommand(orderId));
          SagaLifecycle.end(); // Saga 완료
      }

      // 보상 트랜잭션
      @SagaEventHandler(associationProperty = "orderId")
      public void on(PaymentFailed event) {
          // 재고 예약 취소 (보상)
          commandGateway.send(new ReleaseInventoryCommand(
              orderId, inventoryReservationId
          ));
          commandGateway.send(new CancelOrderCommand(orderId, "결제 실패"));
          SagaLifecycle.end();
      }
  }
```

### 3. 내부 이벤트 vs 외부 이벤트 분리

```
왜 분리해야 하는가:

내부 이벤트 (Event Sourcing용):
  Aggregate 재구성에 사용
  내부 구현 세부사항 포함 가능
  변경 시 해당 Bounded Context 내에서만 영향

  MoneyWithdrawn (내부):
    accountId, amount, balanceAfter,
    requestedBy, requestedAt,
    correlationId, causationId

외부 이벤트 (통합용):
  다른 Bounded Context에 발행
  도메인 언어로 표현 (구현 세부사항 없음)
  변경 시 계약 협의 필요

  FundsWithdrawn (외부):
    accountId, amount, timestamp
    // balanceAfter는 내부 정보 → 제외
    // correlationId는 기술 메타데이터 → 별도 헤더

분리 구현:

  Projection이 내부 이벤트 → 외부 이벤트 변환:
    @EventHandler
    void on(MoneyWithdrawn internalEvent) {
        // 내부 이벤트 → 외부 이벤트 변환
        FundsWithdrawn externalEvent = new FundsWithdrawn(
            internalEvent.accountId(),
            internalEvent.amount(),
            internalEvent.occurredAt()
        );
        // 외부 Kafka 토픽에 발행 (다른 Bounded Context용)
        externalEventPublisher.publish("external-account-events", externalEvent);
    }

  이점:
    내부 Event Sourcing 이벤트 구조 자유롭게 변경
    외부 계약은 별도 버전 관리
```

### 4. Anti-Corruption Layer (ACL)

```
외부 Bounded Context 이벤트 수신 시:

  주문 서비스가 배송 서비스의 이벤트 수신:
    배송 서비스 이벤트: DeliveryStatusUpdated {
        trackingNumber, currentLocation, estimatedArrival,
        carrierId, carrierName, warehouseCode, ...
    }

  Anti-Corruption Layer:
    배송 서비스 언어 → 주문 서비스 언어로 변환

  @Component
  public class ShippingACL {
      @KafkaListener(topics = "shipping-events")
      public void on(DeliveryStatusUpdated externalEvent) {
          // 외부 이벤트 → 내부 Command로 변환
          UpdateShippingStatusCommand internalCmd =
              new UpdateShippingStatusCommand(
                  externalEvent.getOrderId(),    // 매핑
                  mapStatus(externalEvent.getStatus()), // 상태 변환
                  externalEvent.getEstimatedArrival()
              );
          commandBus.send(internalCmd);
      }

      private ShippingStatus mapStatus(String carrierStatus) {
          return switch (carrierStatus) {
              case "IN_TRANSIT" -> ShippingStatus.SHIPPING;
              case "OUT_FOR_DELIVERY" -> ShippingStatus.OUT_FOR_DELIVERY;
              case "DELIVERED" -> ShippingStatus.DELIVERED;
              default -> ShippingStatus.UNKNOWN;
          };
      }
  }

  ACL의 이점:
    배송 서비스 이벤트 스키마 변경 → ACL 수정만으로 대응
    주문 서비스 도메인 모델에 배송 서비스 개념 침투 방지
```

---

## 💻 실전 코드

```java
// ✅ DDD Aggregate + Event Sourcing 완전한 구현
public class Order {

    private OrderId orderId;
    private CustomerId customerId;
    private OrderStatus status;
    private List<OrderLine> lines;
    private Money totalAmount;

    // ── Command 처리 ──────────────────────────────────────────

    public static Order place(PlaceOrderCommand cmd) {
        Order order = new Order();
        order.applyAndRecord(new OrderPlaced(
            new OrderId(cmd.orderId()),
            cmd.customerId(), cmd.lines(),
            cmd.calculateTotal(), Instant.now()
        ));
        return order;
    }

    public void confirm(InventoryChecker checker) {
        if (status != OrderStatus.PLACED)
            throw new InvalidOrderStateException(status);
        checker.validateAll(lines);
        applyAndRecord(new OrderConfirmed(orderId, customerId, lines, totalAmount));
    }

    public void cancel(String reason) {
        if (!status.canCancel())
            throw new OrderCannotBeCancelledException(orderId, status);
        applyAndRecord(new OrderCancelled(orderId, reason, Instant.now()));
    }

    // ── 재구성 (Event Sourcing) ──────────────────────────────

    private void apply(DomainEvent event) {
        switch (event) {
            case OrderPlaced e -> {
                this.orderId = e.orderId();
                this.customerId = e.customerId();
                this.lines = new ArrayList<>(e.lines());
                this.totalAmount = e.totalAmount();
                this.status = OrderStatus.PLACED;
            }
            case OrderConfirmed e -> this.status = OrderStatus.CONFIRMED;
            case OrderCancelled e -> this.status = OrderStatus.CANCELLED;
            default -> {}
        }
    }
}

// ✅ Saga — 주문 처리 조율
@Saga
public class OrderFulfillmentSaga {

    @Autowired private transient CommandGateway commandGateway;

    private String orderId;
    private String reservationId;

    @StartSaga
    @SagaEventHandler(associationProperty = "orderId")
    public void on(OrderConfirmed event) {
        this.orderId = event.orderId().value();

        // 재고 예약 Command 발행
        commandGateway.send(new ReserveStockCommand(
            event.orderId().value(),
            event.lines()
        ));
    }

    @SagaEventHandler(associationProperty = "orderId")
    public void on(StockReserved event) {
        this.reservationId = event.reservationId();

        // 결제 처리 Command 발행
        commandGateway.send(new InitiatePaymentCommand(
            orderId, event.totalAmount()
        ));
    }

    @SagaEventHandler(associationProperty = "orderId")
    public void on(PaymentSucceeded event) {
        commandGateway.send(new ScheduleDeliveryCommand(orderId));
        SagaLifecycle.end();
    }

    // 보상 트랜잭션
    @SagaEventHandler(associationProperty = "orderId")
    public void on(StockReservationFailed event) {
        commandGateway.send(new CancelOrderCommand(orderId, "재고 부족"));
        SagaLifecycle.end();
    }

    @SagaEventHandler(associationProperty = "orderId")
    public void on(PaymentFailed event) {
        // 재고 예약 취소 (보상)
        commandGateway.send(new ReleaseStockCommand(orderId, reservationId));
        commandGateway.send(new CancelOrderCommand(orderId, "결제 실패"));
        SagaLifecycle.end();
    }
}
```

---

## 📊 패턴 비교

```
DDD + CQRS + ES 요소 매핑:

┌─────────────────┬────────────────────────────────────────────────────┐
│ DDD 요소         │ CQRS + ES에서의 역할                               │
├─────────────────┼────────────────────────────────────────────────────┤
│ Aggregate       │ Command 처리 + 이벤트 생성 + 불변식 보호            │
├─────────────────┼────────────────────────────────────────────────────┤
│ Domain Event    │ 이벤트 스토어 저장 + Projection/Saga 트리거         │
├─────────────────┼────────────────────────────────────────────────────┤
│ Repository      │ EventStore에서 Aggregate 로드/저장 추상화           │
├─────────────────┼────────────────────────────────────────────────────┤
│ Domain Service  │ 여러 Aggregate 간 불변식 조율                       │
├─────────────────┼────────────────────────────────────────────────────┤
│ Saga            │ 여러 Aggregate에 걸친 Long-Running 프로세스 조율    │
├─────────────────┼────────────────────────────────────────────────────┤
│ Bounded Context │ 독립 이벤트 스토어 + 독립 읽기 모델                 │
├─────────────────┼────────────────────────────────────────────────────┤
│ ACL             │ 외부 이벤트 → 내부 Command/Event 변환 레이어        │
└─────────────────┴────────────────────────────────────────────────────┘
```

---

## ⚖️ 트레이드오프

```
Saga 복잡도:
  장점: Aggregate 경계 유지, 각 단계 독립 실패 처리
  단점: 최종 일관성, 보상 트랜잭션 설계 필요
        디버깅 어려움 (여러 서비스에 걸친 흐름)

내부/외부 이벤트 분리 비용:
  추가 변환 레이어 (Projection or ACL)
  편익: 내부 설계 자유도 + 외부 계약 안정성
```

---

## 📌 핵심 정리

```
DDD + CQRS + ES 통합 핵심:

Aggregate:
  Command → 이벤트 생성 (불변식 보호)
  이벤트 리플레이 → 재구성 (Event Sourcing)

Saga:
  이벤트 → Command 발행 (여러 Aggregate 조율)
  보상 트랜잭션으로 실패 처리

내부 vs 외부 이벤트:
  내부: Event Sourcing용 (구현 세부사항 포함)
  외부: 다른 Bounded Context용 (도메인 언어만)
  분리: ACL이 외부 → 내부 변환

ACL:
  외부 이벤트 스키마 변경 → ACL만 수정
  내부 도메인 모델 보호
```

---

## 🤔 생각해볼 문제

**Q1.** Saga가 이벤트를 기다리는 중에 타임아웃이 발생하면 어떻게 처리하는가?

<details>
<summary>해설 보기</summary>

Saga 타임아웃 처리는 DeadlineManager를 활용합니다. Saga 시작 시 데드라인(예: 30분)을 등록하고, 데드라인 내에 필요한 이벤트가 오지 않으면 타임아웃 처리를 실행합니다.

Axon Framework의 예를 들면 `DeadlineManager.schedule("orderTimeout", Duration.ofMinutes(30))`으로 등록하고, `@DeadlineHandler("orderTimeout")` 메서드에서 타임아웃 처리 로직을 작성합니다. 타임아웃 시 보상 트랜잭션을 발행하거나 Saga를 종료합니다.

타임아웃 처리 시 고려해야 할 것이 있습니다. 이미 일부 단계가 완료된 상태에서 타임아웃이 발생하면 완료된 단계를 보상해야 합니다. 예를 들어 재고는 예약됐지만 결제가 타임아웃되면 재고 예약 취소 Command를 발행해야 합니다.

타임아웃 후에 지연된 이벤트가 도착하는 경우도 처리해야 합니다. Saga가 이미 종료된 후 이벤트가 오면 무시하거나 보정 처리합니다.

</details>

---

**Q2.** 같은 Bounded Context 내에서 두 Aggregate를 한 번에 수정해야 하는 경우(예: 이체에서 출금 계좌와 입금 계좌), 어떤 접근이 더 좋은가?

<details>
<summary>해설 보기</summary>

두 가지 접근이 있고, 상황에 따라 선택이 달라집니다.

첫째, 같은 트랜잭션에서 두 Aggregate 수정하는 방법입니다. 같은 Bounded Context 내이고 두 Aggregate가 같은 DB를 공유한다면, 단일 트랜잭션으로 처리할 수 있습니다. DDD 교과서에서는 "Aggregate당 한 트랜잭션"을 권장하지만, 실용적으로 허용되는 예외입니다. Event Sourcing에서는 두 Aggregate의 이벤트를 각각의 스트림에 저장하되, 같은 DB 트랜잭션 안에서 처리합니다.

둘째, Saga 패턴으로 분리하는 방법입니다. TransferInitiated 이벤트 → Saga → WithdrawFromAccount Command → MoneyWithdrawn 이벤트 → Saga → DepositToAccount Command 순서로 처리합니다. 각 단계가 독립 트랜잭션이므로 Aggregate 경계가 명확합니다. 단, 최종 일관성을 허용해야 하고 실패 시 보상 트랜잭션이 필요합니다.

이체처럼 금전적 일관성이 중요하고 즉각적인 일관성이 필요하다면 첫 번째 방법이 더 안전합니다. 성능과 확장성이 더 중요하거나 Aggregate가 다른 서비스에 있다면 Saga 패턴을 사용합니다.

</details>

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[이전: 이벤트 저장과 발행의 원자성 ⬅️](./02-event-store-kafka-atomicity.md)** | **[다음: 처리 보장 ➡️](./04-processing-guarantee.md)**

</div>
