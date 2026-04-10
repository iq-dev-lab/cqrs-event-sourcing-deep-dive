# 마이크로서비스와의 통합

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- CQRS가 서비스 내부 패턴인 경우와 Bounded Context 간 통합 패턴인 경우는 어떻게 다른가?
- MSA 환경에서 읽기 모델을 서비스 간에 공유하는 전략과 위험은 무엇인가?
- 서비스 간 이벤트 계약(Event Contract)을 어떻게 설계하고 버전을 관리하는가?
- Consumer-Driven Contract Testing으로 이벤트 스키마 호환성을 어떻게 검증하는가?
- Choreography(이벤트 기반)와 Orchestration(Saga) 중 어느 통합 방식이 CQRS에 더 적합한가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

CQRS는 처음에는 단일 서비스 내 패턴으로 도입되지만, 조직이 성장하면서 여러 마이크로서비스가 이벤트로 통합되는 상황이 된다. 이때 "어느 서비스의 읽기 모델이 다른 서비스의 이벤트를 소비해야 하는가?", "이벤트 스키마가 바뀌면 누구에게 알려야 하는가?", "서비스 간 이벤트 흐름을 어떻게 추적하는가?" 같은 질문이 생긴다. 서비스 경계와 이벤트 계약 설계를 초기부터 올바르게 하지 않으면 MSA가 분산 모놀리스가 된다.

---

## 😱 흔한 실수 (Before — MSA 통합 설계 실수)

```
실수 1: DB를 직접 공유

  // 주문 서비스가 계좌 서비스 DB에 직접 접근
  // order-service
  @Repository
  public interface AccountRepository extends JpaRepository<Account, String> {
      // ❌ 다른 서비스의 테이블 직접 접근
  }
  // 데이터베이스 강결합 → MSA 독립 배포 불가

실수 2: 내부 이벤트를 외부에 그대로 발행

  // 계좌 서비스 내부 이벤트
  AccountBalanceUpdated {
    accountId, previousBalance, newBalance,
    transactionId, internalLedgerCode,   ← 내부 구현
    auditTrailId, ...                    ← 내부 구현
  }
  // 주문 서비스에 그대로 발행
  // → 계좌 서비스 내부 구현 변경 → 주문 서비스도 수정 필요

실수 3: 이벤트 계약 없이 변경

  // 계좌 서비스 개발자:
  // "MoneyTransferred에 필드 추가해도 되겠지"
  // → 주문 서비스 Projection이 역직렬화 실패
  // → 주문 서비스 팀에 알리지 않아 장애 발생
```

---

## ✨ 올바른 접근 (After — 이벤트 계약 + 서비스 경계 명확화)

```
MSA + CQRS 설계 원칙:

1. DB 공유 금지
   각 서비스는 독립 DB
   데이터 공유 = 이벤트 발행 + Projection으로 읽기 모델 생성

2. 내부/외부 이벤트 분리
   내부: Event Sourcing용 (구현 세부사항 포함)
   외부: 다른 서비스용 (공개 계약, 안정적)

3. 이벤트 계약 명시적 관리
   Schema Registry (Confluent, AWS Glue)
   Consumer-Driven Contract Testing (Pact)
   이벤트 스키마 버전 관리

4. 통합 방식 선택
   Choreography: 이벤트 기반 자율 협력
   Orchestration: Saga로 중앙 조율
```

---

## 🔬 내부 동작 원리

### 1. CQRS 적용 범위 — 서비스 내부 vs 서비스 간

```
서비스 내부 CQRS (대부분의 경우):

  [주문 서비스 내부]
  Command Side: Order Aggregate + Event Store
  Query Side: Order Summary, Order Detail, Order Search 읽기 모델
  이벤트: 내부 Kafka Topic "order-events-internal"

  주문 서비스 경계 안에서 완결

서비스 간 통합:

  주문 서비스 → 배송 서비스
  주문 서비스가 OrderConfirmed 이벤트를 외부 토픽에 발행
  배송 서비스가 소비해 DeliveryRequest 읽기 모델 구성

  [주문 서비스]        [배송 서비스]
  Command Side        ← OrderConfirmed (외부 이벤트)
  Query Side          → Delivery Queue 읽기 모델 구성

서비스 경계 원칙:
  각 서비스는 자신의 이벤트 스토어를 가짐
  다른 서비스의 이벤트 스토어에 직접 접근 불가
  공유 = 이벤트 발행/소비로만
```

### 2. 읽기 모델 공유 전략

```
문제: 주문 서비스가 고객 이름을 표시하려면?

안티패턴 — 고객 서비스 DB 직접 조회:
  order_summary.customer_name JOIN customer.name
  → DB 강결합 → 배포 독립성 상실

전략 1 — 이벤트 소비로 로컬 복사본 생성:
  주문 서비스가 고객 서비스의 이벤트를 소비
    CustomerRegistered { customerId, name, email }
    CustomerNameChanged { customerId, newName }

  주문 서비스 로컬 DB:
    customer_reference (로컬 읽기 모델):
      customer_id, customer_name  ← 최신화된 복사본

  order_summary 생성 시:
    customer_name = customerReferenceRepo.findById(customerId).getName()
    → 로컬 데이터 사용, 고객 서비스 호출 없음

  장점: 고객 서비스 장애 시에도 주문 서비스 정상 작동
  단점: 고객 이름 변경 후 order_summary 업데이트에 지연

전략 2 — 이벤트 발행 시 필요한 데이터 포함:
  고객 서비스가 이벤트에 이름을 포함해 발행:
    OrderPlaced { orderId, customerId, customerName, ... }
    ← 이벤트 발행 시점의 고객 이름 포함

  주문 서비스가 이벤트에서 직접 이름 사용
  → 고객 서비스 데이터 복사 불필요

  단점: 이벤트 발행자가 소비자의 요구사항을 미리 알아야 함
        과거 이름이 이벤트에 고정 (이름 변경 후 조회 불가)

전략 3 — API 직접 조회 (최후 수단):
  읽기 모델 생성 시 고객 서비스 API 호출
  → 동기 의존성, 장애 전파 위험
  → Circuit Breaker 필수

권장: 전략 1 (이벤트로 로컬 복사본) 또는 전략 2 (이벤트에 데이터 포함)
```

### 3. 이벤트 계약(Event Contract) 설계

```
이벤트 계약의 구성 요소:

  이벤트 이름: MoneyTransferred
  버전: 2.0
  발행 서비스: account-service
  소비 서비스: order-service, notification-service, analytics-service
  Kafka Topic: account-events-external

  스키마 (JSON Schema or Avro):
  {
    "$schema": "http://json-schema.org/draft-07/schema",
    "type": "object",
    "required": ["eventId", "fromAccountId", "toAccountId", "amount", "occurredAt"],
    "properties": {
      "eventId":       { "type": "string", "format": "uuid" },
      "fromAccountId": { "type": "string" },
      "toAccountId":   { "type": "string" },
      "amount":        { "type": "number", "minimum": 0 },
      "currency":      { "type": "string", "default": "KRW" },  ← optional (backward compatible)
      "occurredAt":    { "type": "string", "format": "date-time" }
    }
  }

Schema Registry (Confluent Schema Registry):
  이벤트 발행 시 스키마 등록/검증
  역직렬화 시 스키마 ID로 정확한 버전 사용
  호환성 모드 설정:
    BACKWARD: 새 버전으로 구버전 이벤트 읽기 가능
    FORWARD: 구버전으로 새 버전 이벤트 읽기 가능
    FULL: 양방향 호환

이벤트 계약 변경 절차:
  1. 변경 계획 소비 서비스 팀에 사전 공지
  2. 하위 호환 변경 (필드 추가, Optional화)
  3. 비호환 변경 시 새 버전 이벤트 타입 추가 (구버전 유지)
  4. 소비 서비스가 새 버전으로 마이그레이션
  5. 구버전 이벤트 deprecated → 단계적 제거
```

### 4. Consumer-Driven Contract Testing

```
소비자가 주도하는 계약 테스트:

Pact 프레임워크 사용:

  소비 서비스(주문 서비스)가 계약 정의:
    "account-service가 이런 이벤트를 발행해야 한다"

  // order-service 테스트 (Pact 소비자 테스트)
  @Test
  void account_service_should_publish_money_transferred() {
      PactDslJsonBody body = new PactDslJsonBody()
          .uuid("eventId")
          .stringType("fromAccountId")
          .stringType("toAccountId")
          .numberType("amount")
          .datetime("occurredAt");

      // 이 계약이 Pact Broker에 저장됨
  }

  발행 서비스(계좌 서비스)가 계약 검증:
    "주문 서비스가 정의한 계약을 우리가 만족하는가?"

  // account-service 테스트 (Pact 제공자 테스트)
  @Test
  @Provider("account-service")
  @PactBroker(url = "https://pact-broker.example.com")
  void verifyMoneyTransferredEvent() {
      // account-service가 발행하는 이벤트가
      // order-service가 정의한 계약과 일치하는지 검증
  }

  CI/CD 파이프라인:
    account-service 배포 전 → 모든 소비자 계약 검증
    계약 위반 → 배포 차단
    → 소비자가 모르는 사이에 이벤트 스키마 깨지는 것 방지

장점:
  이벤트 스키마 변경이 소비자를 깨지 않음을 자동 검증
  소비자 팀이 자신의 요구사항을 계약으로 표현
  발행자가 실수로 계약을 깨면 CI에서 즉시 감지
```

### 5. Choreography vs Orchestration

```
Choreography (이벤트 기반 협력):

  각 서비스가 이벤트를 듣고 자율적으로 반응
  중앙 조율자 없음

  OrderPlaced 이벤트 발행
    → 재고 서비스: 재고 예약
    → 결제 서비스: 결제 처리
    → 알림 서비스: 이메일 발송
  (각 서비스가 독립적으로 이벤트 소비)

  장점: 느슨한 결합, 서비스 추가 쉬움
  단점: 전체 흐름 파악 어려움, 실패 처리 복잡

Orchestration (Saga 중앙 조율):

  Saga가 전체 흐름을 알고 순서대로 Command 발행
  Saga → ReserveInventory → InventoryReserved 수신
      → ProcessPayment → PaymentProcessed 수신
      → ConfirmOrder

  장점: 전체 흐름 명확, 실패 처리 중앙화
  단점: 오케스트레이터가 모든 서비스를 알아야 함

CQRS + MSA 권장:
  단순한 이벤트 전파 → Choreography
  여러 단계의 비즈니스 프로세스 → Orchestration (Saga)
  두 방식 혼용 가능
```

---

## 💻 실전 코드

```java
// ✅ 서비스 간 이벤트 발행 — 내부/외부 분리
@Component
public class AccountEventPublisher {

    // 내부 이벤트 (Event Sourcing용, 구현 세부사항 포함)
    private static final String INTERNAL_TOPIC = "account-events-internal";

    // 외부 이벤트 (다른 서비스용, 공개 계약)
    private static final String EXTERNAL_TOPIC = "account-events-external";

    @EventHandler  // 내부 이벤트 처리
    public void on(MoneyTransferred internalEvent) {
        // 내부 이벤트 → 내부 토픽 (Event Sourcing용)
        kafkaTemplate.send(INTERNAL_TOPIC,
            internalEvent.accountId(), serialize(internalEvent));

        // 외부 이벤트 변환 (공개 계약, 구현 세부사항 제거)
        FundsTransferred externalEvent = FundsTransferred.from(internalEvent);
        kafkaTemplate.send(EXTERNAL_TOPIC,
            externalEvent.fromAccountId(), serialize(externalEvent));
    }
}

// ✅ 외부 이벤트 스키마 (안정적 공개 계약)
public record FundsTransferred(
    String eventId,
    String eventType,       // "FundsTransferred"
    int schemaVersion,      // 2
    String fromAccountId,
    String toAccountId,
    BigDecimal amount,
    String currency,
    Instant occurredAt
    // 내부 필드 제외: internalLedgerCode, auditTrailId, balanceAfter 등
) {
    public static FundsTransferred from(MoneyTransferred internal) {
        return new FundsTransferred(
            internal.eventId(), "FundsTransferred", 2,
            internal.fromAccountId(), internal.toAccountId(),
            internal.amount(), "KRW", internal.occurredAt()
        );
    }
}

// ✅ 소비 서비스 — ACL + 로컬 읽기 모델
@Component
public class AccountEventACL {  // Anti-Corruption Layer

    @KafkaListener(
        topics = "account-events-external",
        groupId = "order-service-account-acl"
    )
    public void on(FundsTransferred event) {
        // 외부 이벤트 → 내부 도메인 모델로 변환
        // (계좌 서비스의 언어를 주문 서비스 언어로)
        accountReferenceRepo.updateBalance(
            event.fromAccountId(),
            calculateNewBalance(event)
        );
    }
}

// ✅ Consumer-Driven Contract Test (Pact)
@ExtendWith(PactConsumerTestExt.class)
@PactTestFor(providerName = "account-service")
class AccountEventContractTest {

    @Pact(consumer = "order-service")
    public MessagePact fundTransferredPact(MessagePactBuilder builder) {
        return builder
            .expectsToReceive("a funds transferred event")
            .withContent(new PactDslJsonBody()
                .uuid("eventId")
                .stringMatcher("eventType", "FundsTransferred")
                .integerType("schemaVersion")
                .stringType("fromAccountId")
                .stringType("toAccountId")
                .decimalType("amount")
                .stringType("currency")
                .datetime("occurredAt", "yyyy-MM-dd'T'HH:mm:ss'Z'"))
            .toPact();
    }

    @Test
    @PactTestFor(pactMethod = "fundTransferredPact")
    void test_consume_funds_transferred(List<Message> messages) {
        FundsTransferred event = deserialize(messages.get(0).getBody());
        assertThat(event.eventId()).isNotNull();
        assertThat(event.amount()).isPositive();
        // 주문 서비스가 필요한 필드가 계약에 포함됐는지 검증
    }
}
```

---

## 📊 패턴 비교

```
MSA 데이터 공유 방식 비교:

┌──────────────────────┬──────────────┬──────────────┬──────────────┐
│ 방식                 │ 결합도       │ 일관성       │ 복잡도       │
├──────────────────────┼──────────────┼──────────────┼──────────────┤
│ DB 공유              │ 높음 (금지)  │ 강한 일관성  │ 낮음(초기)   │
├──────────────────────┼──────────────┼──────────────┼──────────────┤
│ API 직접 호출        │ 중간         │ 강한 일관성  │ 중간         │
├──────────────────────┼──────────────┼──────────────┼──────────────┤
│ 이벤트 + 로컬 복사본 │ 낮음         │ 최종 일관성  │ 높음         │
└──────────────────────┴──────────────┴──────────────┴──────────────┘

Choreography vs Orchestration:
  단순 알림/집계 → Choreography (느슨한 결합)
  주문 처리 같은 다단계 → Orchestration (Saga)
  실패 처리 명확성 → Orchestration
  서비스 추가 용이성 → Choreography
```

---

## ⚖️ 트레이드오프

```
이벤트 계약 관리 비용:
  Schema Registry 운영
  Consumer-Driven Contract Testing 구축
  이벤트 스키마 변경 시 소비 서비스 팀 협의 프로세스

편익:
  이벤트 스키마 변경으로 인한 서비스 장애 방지
  발행 서비스와 소비 서비스 독립 배포 가능
  이벤트 계약이 공식 문서 역할

이벤트 기반 읽기 모델 복사본:
  데이터 중복 증가
  이벤트 지연만큼 데이터 지연
  편익: 서비스 간 실시간 의존성 없음 → 고가용성
```

---

## 📌 핵심 정리

```
MSA + CQRS 통합 핵심:

서비스 경계:
  DB 공유 금지 → 이벤트로만 데이터 공유
  각 서비스는 독립 이벤트 스토어 + 독립 읽기 모델

이벤트 계약:
  내부 이벤트 (ES용) ≠ 외부 이벤트 (통합용)
  외부 이벤트: 공개 계약, 안정적, 버전 관리
  Schema Registry + Consumer-Driven Contract Testing

읽기 모델 공유:
  다른 서비스 이벤트 소비 → 로컬 읽기 모델 생성
  또는 이벤트에 필요한 데이터 포함
  API 직접 호출은 최후 수단 (Circuit Breaker 필수)

통합 방식:
  Choreography: 단순 이벤트 전파, 느슨한 결합
  Orchestration: 다단계 비즈니스 프로세스, 실패 처리 명확
```

---

## 🤔 생각해볼 문제

**Q1.** 주문 서비스가 계좌 서비스의 이벤트를 소비해 로컬 읽기 모델을 만든다. 이 읽기 모델이 최신이 아닐 때 주문 서비스가 잘못된 판단을 하는 문제가 발생할 수 있는가?

<details>
<summary>해설 보기</summary>

예, 발생할 수 있습니다. 이것이 MSA 환경에서 Eventual Consistency의 실제 위험입니다.

예를 들어 고객이 계좌 잔고 300만원을 보유하고 있고, 다른 탭에서 250만원을 출금한 직후 200만원짜리 주문을 시도하는 상황입니다. 출금 이벤트가 주문 서비스 로컬 읽기 모델에 아직 반영되지 않았다면, 주문 서비스는 잔고가 300만원이라고 판단하고 200만원 주문을 허용할 수 있습니다. 실제 잔고는 50만원인데 말입니다.

이 문제의 해결 방법이 두 가지입니다. 첫째, 불변식 검증은 쓰기 모델에서 합니다. 주문 서비스가 결제를 시도할 때 계좌 서비스에 실시간으로 잔고를 확인하는 API를 호출합니다(Saga 패턴). 로컬 읽기 모델은 표시 목적으로만 사용합니다.

둘째, 도메인을 재설계합니다. "주문 생성"과 "결제 처리"를 분리하면, 주문은 낙관적으로 생성하고 결제 시점에 잔고 확인 + 차감을 원자적으로 처리합니다. 잔고 부족 시 주문을 취소합니다.

결국 MSA에서 정확한 비즈니스 결정은 이벤트 기반 로컬 읽기 모델이 아닌, Saga를 통한 실시간 협력으로 해야 합니다.

</details>

---

**Q2.** 이벤트 계약이 변경될 때 "발행 서비스 먼저 배포 vs 소비 서비스 먼저 배포" 중 어느 것이 더 안전한가?

<details>
<summary>해설 보기</summary>

"소비 서비스 먼저 배포"가 더 안전합니다. 이를 Expand-Contract(또는 Parallel Change) 패턴이라고 합니다.

예를 들어 `MoneyTransferred` 이벤트에 `currency` 필드를 추가하는 경우입니다.

소비 서비스 먼저 배포하면 다음과 같습니다. 먼저 소비 서비스가 구버전(`currency` 없음)과 신버전(`currency` 있음) 이벤트를 모두 처리할 수 있도록 수정합니다. 소비 서비스를 배포합니다. 그 다음 발행 서비스를 배포해 `currency` 포함 이벤트를 발행합니다. 소비 서비스는 이미 신버전을 처리할 수 있으므로 장애 없음입니다.

발행 서비스 먼저 배포하면 다음과 같습니다. 발행 서비스가 `currency` 포함 이벤트를 발행합니다. 소비 서비스는 아직 구버전 코드입니다. `currency` 필드를 처리 못함 → 역직렬화 실패 또는 오작동입니다.

`@JsonIgnoreProperties(ignoreUnknown = true)`로 새 필드를 무시한다면 발행 먼저 배포도 가능합니다. 하지만 필드 제거나 이름 변경 같은 비호환 변경은 반드시 소비 서비스 먼저 배포해야 합니다.

</details>

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[이전: 성능 최적화 ⬅️](./05-performance-optimization.md)** | **[다음: Chapter 6 — Spring & Axon 구현 ➡️](../spring-axon-implementation/01-axon-setup.md)**

</div>
