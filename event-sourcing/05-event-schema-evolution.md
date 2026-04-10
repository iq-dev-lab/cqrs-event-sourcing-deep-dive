# 이벤트 스키마 진화

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 이벤트는 왜 영구적으로 변경할 수 없고, 이것이 스키마 변경을 어렵게 만드는 이유는 무엇인가?
- Upcasting이란 무엇이며, 구버전 이벤트를 신버전으로 어떻게 변환하는가?
- 필드 추가, 필드 제거, 필드 이름 변경, 타입 변경 각각의 안전성 수준은 어떻게 다른가?
- 이벤트 버전을 관리하는 실전 패턴은 무엇인가?
- 스키마 변경이 Projection에 미치는 영향은 어떻게 처리하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

이벤트는 영원히 저장된다. 1년 전에 발행된 이벤트 페이로드는 지금도 이벤트 스토어에 있고, 새 코드로 역직렬화해야 한다. 도메인이 성장하면서 이벤트 스키마도 변해야 하는데, 과거 이벤트를 수정할 수 없다면 어떻게 할 것인가? 이 문제를 이해하지 못하면 첫 번째 스키마 변경에서 시스템이 멈추거나, 변경을 두려워해 도메인 모델이 경직된다.

---

## 😱 흔한 실수 (Before — 스키마 변경을 두려워하거나 무분별하게 변경)

```
실수 1: 과거 이벤트 페이로드를 직접 수정

  // 개발자: "필드 이름이 잘못됐으니 수정하자"
  UPDATE event_store
  SET payload = jsonb_set(payload, '{fromAccountId}',
                          payload->'sourceAccountId')
  WHERE event_type = 'MoneyTransferred';

  // 문제:
  //   이벤트는 과거 사실의 기록 — 수정은 사실 조작
  //   수정 후 감사 로그 무결성 깨짐
  //   다른 서비스의 Consumer가 원본 스키마를 기대하면 깨짐
  //   백업에서 복구하면 수정 내용 날아감

실수 2: 필드 제거 후 바로 코드 배포

  // 이전 이벤트
  { "accountId": "ACC-001", "amount": 100000, "currency": "KRW" }

  // 새 코드에서 currency 제거 후 역직렬화
  @JsonIgnoreUnknownProperties(false) // 알 수 없는 필드 오류 처리
  public record MoneyDeposited(String accountId, BigDecimal amount) {}
  // currency 필드가 이벤트에 있는데 Java 클래스에 없음 → 예외

실수 3: 이벤트 타입 이름 변경

  // 이전: MoneyTransferred → 새: FundsTransferred (이름 변경)
  // 이벤트 스토어에는 여전히 event_type = 'MoneyTransferred'
  // 새 코드에서 FundsTransferred만 처리
  // → 모든 과거 MoneyTransferred 이벤트 역직렬화 실패
```

---

## ✨ 올바른 접근 (After — Upcasting으로 하위 호환 유지)

```
안전한 스키마 변경 원칙:

✅ 안전한 변경 (Backward Compatible):
  필드 추가 (기본값 있음)
  필드를 Optional/nullable로 변경
  새 이벤트 타입 추가

⚠️ 주의 필요한 변경:
  필드 이름 변경 → Upcasting 레이어 필요
  필드 제거 → 읽기 코드에서 무시 처리 후 제거
  필드 타입 변경 → Upcasting 레이어 필요

❌ 절대 금지:
  이벤트 페이로드 직접 수정 (DB UPDATE)
  이벤트 타입 이름 변경 (마이그레이션 없이)
  이벤트 삭제

Upcasting 전략:
  구버전 이벤트를 역직렬화한 직후
  신버전 형태로 변환하는 레이어
  이벤트 스토어의 원본은 그대로 유지
```

---

## 🔬 내부 동작 원리

### 1. 변경 유형별 안전성 분석

```
변경 1 — 필드 추가 (항상 안전):

  v1 이벤트: { "accountId": "ACC-001", "amount": 100000 }
  v2 이벤트: { "accountId": "ACC-001", "amount": 100000, "currency": "KRW" }

  v1 이벤트 역직렬화 시:
    currency 필드 없음 → null or 기본값 처리
    → @JsonSetter(nulls = Nulls.AS_EMPTY) or 기본값 지정

  코드:
    public record MoneyDeposited(
        String accountId,
        BigDecimal amount,
        String currency = "KRW"  // 기본값 — v1 이벤트 호환
    ) {}

변경 2 — 필드 제거 (단계적 처리 필요):

  v1 이벤트: { "accountId": "ACC-001", "amount": 100000, "note": "ATM 입금" }
  v2 이벤트: { "accountId": "ACC-001", "amount": 100000 }  (note 제거)

  올바른 절차:
    Step 1: note 필드를 코드에서 사용하지 않도록 변경 (@Deprecated)
            역직렬화 시 무시 처리: @JsonIgnoreProperties(ignoreUnknown = true)
    Step 2: 새 이벤트부터 note 없이 발행
    Step 3: 구버전 이벤트의 note 필드는 역직렬화 시 무시
    Step 4: 모든 소비자(Projection)가 note를 안 쓴다면 스키마에서 제거

  주의: 절대 v1 이벤트의 note 필드를 DB에서 삭제하지 말 것

변경 3 — 필드 이름 변경 (Upcasting 필요):

  v1: { "sourceId": "ACC-001", "targetId": "ACC-002", "amount": 100000 }
  v2: { "fromAccountId": "ACC-001", "toAccountId": "ACC-002", "amount": 100000 }

  Upcaster:
    MoneyTransferredV1 { sourceId, targetId, amount }
    → MoneyTransferredV2 { fromAccountId, toAccountId, amount }

    변환 코드:
      if (event instanceof MoneyTransferredV1 v1) {
          return new MoneyTransferredV2(
              v1.sourceId(),   // → fromAccountId
              v1.targetId(),   // → toAccountId
              v1.amount()
          );
      }

변경 4 — 이벤트 타입 이름 변경:

  이전: MoneyTransferred → 새: FundsTransferred

  방법 1: 별칭(Alias) 등록
    deserialize("MoneyTransferred") → FundsTransferred.class 로 매핑

  방법 2: Upcaster 레이어에서 처리
    event_type = 'MoneyTransferred' → FundsTransferred로 변환

  방법 3: 두 이름 모두 처리
    if (rawType.equals("MoneyTransferred") || rawType.equals("FundsTransferred"))
        return deserializeAs(FundsTransferred.class, payload);
```

### 2. Upcasting 패턴 구현

```
Upcasting 레이어:

  이벤트 로드 흐름:
    eventStore → [raw event: type="MoneyTransferred", payload=v1]
    → UpcasterChain
    → [MoneyTransferred v1 → MoneyTransferred v2 변환]
    → apply() 메서드 호출 (항상 최신 버전 이벤트)

  Upcaster 체인:
    UpcasterV1toV2 → UpcasterV2toV3 → ... → 최신 버전
    → 여러 버전을 건너뛸 때도 순차 변환

  구조:
    interface Upcaster<Old, New> {
        boolean canUpcast(String eventType, int version);
        New upcast(Old oldEvent);
    }

  // 이벤트 페이로드 버전 관리:
  { "schemaVersion": 1,  ← 버전 명시
    "sourceId": "ACC-001",
    "targetId": "ACC-002",
    "amount": 100000 }

  { "schemaVersion": 2,  ← 업그레이드된 버전
    "fromAccountId": "ACC-001",
    "toAccountId": "ACC-002",
    "amount": 100000 }
```

### 3. Projection과 스키마 변경

```
Projection이 이벤트 스키마에 의존하는 구조:

  OrderSummaryProjection.on(OrderConfirmed event) {
      summary.setCustomerName(event.getCustomerName()); // 특정 필드 의존
  }

스키마 변경 → Projection 영향:

  OrderConfirmed에서 customerName → customer.name (중첩 객체)로 변경:
    v1: { "customerName": "김철수" }
    v2: { "customer": { "name": "김철수", "tier": "VIP" } }

  Projection 변경 필요:
    v1: event.getCustomerName()
    v2: event.getCustomer().getName()

  Upcasting 없이 대응하면:
    v1 이벤트 처리: event.getCustomerName()
    v2 이벤트 처리: event.getCustomer().getName()
    → Projection에 버전 분기 코드 증가

  Upcasting으로 대응하면:
    OrderConfirmedV1 → OrderConfirmedV2 변환 (Upcaster에서)
    Projection: 항상 v2 이벤트를 받음 → 분기 없음

Projection 재구축 시 스키마 변경 이점:
  "새 필드 customer.tier가 추가됐다 → 기존 주문 통계에 적용하고 싶다"
  Event Sourcing: Upcaster에 tier 기본값 추가 → 전체 이벤트 재구축
  상태 저장: 과거 주문에는 tier 없음 → 과거 데이터 없음
```

---

## 💻 실전 코드

```java
// ==============================
// 이벤트 스키마 버전 관리 구현
// ==============================

// ✅ 이벤트에 schemaVersion 포함
public record MoneyTransferred(
    int schemaVersion,      // 이벤트 스키마 버전
    String fromAccountId,   // v2에서 변경된 필드 이름
    String toAccountId,
    BigDecimal amount,
    Instant occurredAt
) implements DomainEvent {}

// ✅ Upcaster 인터페이스
public interface Upcaster {
    boolean canUpcast(String eventType, Map<String, Object> rawPayload);
    Map<String, Object> upcast(Map<String, Object> rawPayload);
}

// ✅ MoneyTransferred v1 → v2 Upcaster
@Component
public class MoneyTransferredV1ToV2Upcaster implements Upcaster {

    @Override
    public boolean canUpcast(String eventType, Map<String, Object> payload) {
        return "MoneyTransferred".equals(eventType)
            && (payload.get("schemaVersion") == null
                || (int) payload.get("schemaVersion") < 2);
    }

    @Override
    public Map<String, Object> upcast(Map<String, Object> payload) {
        Map<String, Object> upgraded = new HashMap<>(payload);

        // sourceId → fromAccountId
        upgraded.put("fromAccountId", payload.get("sourceId"));
        upgraded.remove("sourceId");

        // targetId → toAccountId
        upgraded.put("toAccountId", payload.get("targetId"));
        upgraded.remove("targetId");

        // schemaVersion 업데이트
        upgraded.put("schemaVersion", 2);

        return upgraded;
    }
}

// ✅ Upcaster 체인 — 역직렬화 전에 적용
@Component
public class UpcasterChain {

    private final List<Upcaster> upcasters;

    public Map<String, Object> applyAll(String eventType, String rawPayload) {
        Map<String, Object> payload = objectMapper.readValue(rawPayload, Map.class);

        // 순서대로 적용: v1→v2, v2→v3, ...
        for (Upcaster upcaster : upcasters) {
            if (upcaster.canUpcast(eventType, payload)) {
                payload = upcaster.upcast(payload);
            }
        }

        return payload; // 항상 최신 버전 반환
    }
}

// ✅ Event Serializer — Upcaster 통합
@Component
public class EventSerializer {

    public DomainEvent deserialize(String rawPayload, String eventType) {
        // 1. Upcasting 적용 (구버전 → 최신 버전)
        Map<String, Object> upgradedPayload =
            upcasterChain.applyAll(eventType, rawPayload);

        // 2. 최신 버전으로 역직렬화
        Class<? extends DomainEvent> eventClass =
            eventTypeRegistry.resolve(eventType);

        return objectMapper.convertValue(upgradedPayload, eventClass);
    }
}

// ✅ 안전한 역직렬화 — 알 수 없는 필드 무시
@JsonIgnoreProperties(ignoreUnknown = true)  // 구버전 필드 무시
public record MoneyDeposited(
    String accountId,
    BigDecimal amount,
    BigDecimal balanceAfter,
    String depositedBy,
    @JsonProperty("currency")
    @JsonSetter(nulls = Nulls.AS_EMPTY)
    String currency,          // 신규 필드 — null이면 기본값 "KRW"
    Instant occurredAt
) implements DomainEvent {

    // 기본값 처리
    public String currency() {
        return currency != null ? currency : "KRW";
    }
}
```

---

## 📊 패턴 비교

```
스키마 변경 유형별 대응 전략:

┌───────────────────┬─────────────────┬───────────────────────────┐
│ 변경 유형         │ 안전성          │ 대응 전략                  │
├───────────────────┼─────────────────┼───────────────────────────┤
│ 필드 추가         │ ✅ 안전         │ 기본값 지정, @JsonSetter   │
├───────────────────┼─────────────────┼───────────────────────────┤
│ 필드 Optional화   │ ✅ 안전         │ nullable 타입 사용         │
├───────────────────┼─────────────────┼───────────────────────────┤
│ 새 이벤트 타입 추가│ ✅ 안전        │ apply() default 처리 추가  │
├───────────────────┼─────────────────┼───────────────────────────┤
│ 필드 이름 변경    │ ⚠️ 주의        │ Upcaster 구현              │
├───────────────────┼─────────────────┼───────────────────────────┤
│ 필드 타입 변경    │ ⚠️ 주의        │ Upcaster 구현              │
├───────────────────┼─────────────────┼───────────────────────────┤
│ 이벤트 타입 이름  │ ⚠️ 주의        │ 별칭 또는 Upcaster         │
│ 변경              │                 │                           │
├───────────────────┼─────────────────┼───────────────────────────┤
│ 필드 제거         │ ⚠️ 단계적       │ 먼저 무시 처리, 후에 제거 │
├───────────────────┼─────────────────┼───────────────────────────┤
│ 이벤트 직접 수정  │ ❌ 금지         │ 절대 금지                  │
└───────────────────┴─────────────────┴───────────────────────────┘
```

---

## ⚖️ 트레이드오프

```
Upcasting 비용:
  역직렬화마다 Upcaster 체인 실행 (성능 영향 미미)
  Upcaster 코드 유지 필요 (버전 수만큼 증가)
  테스트: 각 Upcaster + 체인 통합 테스트

Upcasting의 대안 — Weak Schema:
  JSON에서 알 수 없는 필드 무시 (@JsonIgnoreProperties)
  필드 추가/제거는 자동 처리
  필드 이름 변경은 여전히 Upcasting 필요

이벤트 버전의 장기 관리:
  버전이 쌓이면 Upcaster 체인이 길어짐
  오래된 Upcaster 제거 기준:
    "v1 이벤트가 더 이상 이벤트 스토어에 없다면" → Upcaster 제거 가능
    → 스냅샷으로 모든 v1 이벤트를 v2 이후로 변환했다면 안전
```

---

## 📌 핵심 정리

```
이벤트 스키마 진화 핵심:

기본 원칙:
  이벤트는 절대 수정/삭제 불가 — 과거 사실
  스키마 변경은 새 코드가 구 이벤트를 읽는 방식으로 처리
  @JsonIgnoreProperties(ignoreUnknown=true) — 항상 설정

안전한 변경:
  필드 추가: 기본값 처리, 하위 호환 자동
  필드 제거: 단계적 (사용 중단 → 무시 → 제거)

Upcasting:
  이벤트 로드 직후 적용 → 항상 최신 버전 이벤트로 변환
  Projection은 항상 최신 버전만 처리
  체인으로 여러 버전 건너뛰기 가능

버전 관리:
  이벤트에 schemaVersion 필드 포함 권장
  이벤트 타입 이름 변경 → 별칭 또는 Upcaster로 처리
```

---

## 🤔 생각해볼 문제

**Q1.** 이벤트에 `schemaVersion` 필드를 포함하는 방식과 이벤트 타입 이름에 버전을 포함하는 방식(`MoneyTransferredV1`, `MoneyTransferredV2`) 중 어느 것이 더 나은가?

<details>
<summary>해설 보기</summary>

두 방식 모두 장단점이 있고, 팀의 규모와 변경 빈도에 따라 선택이 달라집니다.

이벤트 타입 이름에 버전 포함 방식(`MoneyTransferredV2`)의 장점은 코드에서 어느 버전인지 명확하고, 타입 시스템이 버전을 강제합니다. 단점은 새 버전마다 새 클래스가 필요하고, Projection에서 여러 버전을 분기해야 할 수 있습니다.

`schemaVersion` 필드 방식의 장점은 하나의 클래스로 여러 버전을 Upcasting으로 처리할 수 있고, Projection 코드가 단순합니다. 단점은 schemaVersion 값이 런타임에서만 확인되어 컴파일 타임 안전성이 없습니다.

실무에서는 두 방식의 혼합이 많이 사용됩니다. 소규모 변경은 `schemaVersion`으로 처리하고, 의미가 완전히 달라진 이벤트는 새 이름으로 추가합니다. Axon Framework는 `@Revision("2.0")` 어노테이션으로 이벤트 버전을 관리합니다.

</details>

---

**Q2.** Upcaster가 필요하지 않은 "이벤트 스키마 설계 원칙"이 있다면 무엇인가?

<details>
<summary>해설 보기</summary>

스키마 변경을 최소화하는 이벤트 설계 원칙들이 있습니다.

첫째, 이벤트 페이로드에 필요한 모든 데이터를 포함합니다. "나중에 필요하면 추가하자"보다 처음부터 충분한 데이터를 포함하면 나중에 필드를 추가할 필요가 줄어듭니다. 단, 과도한 데이터 포함은 페이로드를 무겁게 합니다.

둘째, 비즈니스 의미가 명확한 이름을 처음부터 사용합니다. `sourceId` 대신 `fromAccountId`처럼 의미가 분명한 이름을 쓰면 나중에 이름을 바꿀 이유가 줄어듭니다.

셋째, 계산 가능한 값은 이벤트에 포함하지 않습니다. `totalAmount = quantity × price`처럼 계산 가능한 값은 이벤트에 넣지 않고 Projection에서 계산합니다. 계산 로직이 바뀌면 이벤트 스키마를 바꾸지 않아도 됩니다.

넷째, 참조 데이터(고객 이름 등)를 이벤트에 포함하지 않고 ID만 포함합니다. 이름이 바뀌어도 이벤트 스키마는 변하지 않습니다.

</details>

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[이전: 스냅샷 패턴 ⬅️](./04-snapshot-pattern.md)** | **[다음: Event Sourcing의 장점 완전 분해 ➡️](./06-event-sourcing-benefits.md)**

</div>
