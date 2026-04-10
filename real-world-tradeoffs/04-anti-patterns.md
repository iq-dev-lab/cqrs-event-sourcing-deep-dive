# 안티패턴

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- "command / query 패키지 분리"만 하면 CQRS가 되는가?
- 읽기 모델을 쓰기 모델과 강결합하면 어떤 문제가 발생하는가?
- 모든 도메인에 Event Sourcing을 적용하면 왜 생산성이 떨어지는가?
- 이벤트 스키마를 자주 변경하면 구체적으로 어떤 기술 부채가 생기는가?
- 각 안티패턴을 조기에 감지하는 신호는 무엇인가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

CQRS와 Event Sourcing을 도입하면서 흔히 빠지는 함정들이 있다. 표면적으로 CQRS처럼 보이지만 실제로는 아무 이점도 없는 구조, 혹은 과도한 설계로 오히려 유지보수가 더 어려워지는 경우들이다. 이 패턴들을 미리 알면 팀이 잘못된 방향으로 수개월을 낭비하는 것을 방지할 수 있다.

---

## 😱 흔한 실수 (Before — 안티패턴의 전형)

```
안티패턴 1: 패키지 분리를 CQRS라고 부름

  com.example.order
    ├── command/
    │   └── OrderCommandService (createOrder, confirmOrder, cancelOrder)
    └── query/
        └── OrderQueryService (getOrder, getOrderList)

  OrderCommandService.confirmOrder():
    Order order = orderRepository.findById(id); // 조회
    order.setStatus("CONFIRMED");                // 쓰기 모델 직접 변경
    orderRepository.save(order);
    // 읽기 전용 모델 없음 — OrderQueryService도 orders 테이블 직접 조회
  // → 패키지만 나눴을 뿐 쓰기/읽기 모델 분리 없음

안티패턴 2: 모든 도메인에 Event Sourcing

  UserProfileAggregate:
    @CommandHandler UpdateNicknameCommand → NicknameUpdated 이벤트
    @CommandHandler UpdateBioCommand → BioUpdated 이벤트
    @CommandHandler UpdateAvatarCommand → AvatarUpdated 이벤트
  // 사용자 프로필 = CRUD → ES 복잡도만 추가, 이점 없음

안티패턴 3: 이벤트 스키마를 자주 변경

  Sprint 1: OrderPlaced { orderId, customerId, items }
  Sprint 2: OrderPlaced { orderId, customerId, items, couponId }  ← 필드 추가
  Sprint 3: OrderPlaced { orderId, customerId, lineItems, couponId } ← items→lineItems
  Sprint 4: OrderPlaced { orderId, buyerId, lineItems, promotionId } ← 다른 변경
  // → 매 Sprint마다 Upcaster 작성, Projection 재구축
  // → 이벤트 스키마 변경이 개발 속도보다 빠름
```

---

## ✨ 올바른 접근 (After — 각 안티패턴의 진단과 해결)

```
안티패턴 1 진단: "읽기 모델이 있는가?"
  쓰기 모델과 다른 스키마의 읽기 전용 테이블/뷰 없음 → CQRS 아님
  해결: 읽기 모델 테이블 생성 → 조회 최적화

안티패턴 2 진단: "왜 ES가 필요한가?"
  감사 로그 법적 요구사항? 시간 여행 비즈니스 요구사항? 없음 → ES 불필요
  해결: 상태 저장 + CQRS로도 충분

안티패턴 3 진단: "Upcaster가 몇 개인가?"
  3개월 만에 Upcaster 5개 이상 → 이벤트 스키마 설계 재검토
  해결: 이벤트 설계 원칙 확립 (필드 추가만 허용, 도메인 언어 사용)
```

---

## 🔬 내부 동작 원리

### 1. 안티패턴 1 — 패키지 분리만 한 가짜 CQRS

```
가짜 CQRS의 특징:

  증상:
    OrderCommandService와 OrderQueryService 모두 OrderRepository 사용
    읽기/쓰기가 같은 order 테이블 직접 접근
    OrderQueryService에서 복잡한 JOIN 쿼리 실행 (읽기 최적화 없음)
    "CQRS 도입했음" 이라고 주장

  왜 문제인가:
    CQRS의 이점 없음 (읽기 전용 최적화 없음)
    코드만 더 복잡해짐 (두 Service 관리)
    팀이 "CQRS는 복잡하기만 하다"는 오해

  진짜 CQRS의 기준:
    읽기 모델이 쓰기 모델과 다른 스키마/테이블 사용?
    읽기 쿼리가 쓰기 DB에 JOIN 없이 단일 테이블 조회?
    읽기 DB가 쓰기 DB와 독립적으로 스케일 가능?
    → 하나라도 아니면 패키지 분리에 불과

  올바른 CQRS 최소 기준:
    OrderSummary 읽기 모델 테이블 (비정규화)
    OrderCommandService → orders 테이블 (정규화)
    OrderQueryService → order_summary 테이블 (비정규화)
    → 쓰기와 읽기가 다른 테이블을 사용
```

### 2. 안티패턴 2 — 모든 도메인에 Event Sourcing

```
ES가 맞지 않는 도메인:

단순 CRUD 도메인 → ES 불필요:
  UserPreferences (사용자 설정):
    알림 on/off, 언어 설정, 테마
    → "마지막 상태"만 중요, 변경 이력 불필요
    → ES로: UserPreferenceChanged 이벤트 수십 개 쌓임
    → 이득: 없음 / 비용: 이벤트 스토어, 재구성 코드

  ProductCatalog (상품 카탈로그):
    상품 이름, 설명, 이미지 URL
    → 현재 상태 조회가 99%
    → 이전 상품 설명이 필요한 비즈니스 케이스 없음

  SessionData (세션):
    로그인 상태, 장바구니
    → 매우 짧은 생명주기, 감사 불필요

ES가 맞는 도메인:
  BankAccount: 모든 입출금 이력이 법적 필수
  Order: 주문→확인→배송→완료 상태 전환 감사
  Inventory: 재고 변동 추적 (재고 부족 원인 분석)
  AuditLog: 관리자 행동 추적

판단 기준:
  "이 도메인의 변경 이력이 비즈니스 가치를 제공하는가?"
  → YES: ES 검토
  → NO: 상태 저장 유지

안티패턴 증상:
  팀에서 "이건 ES 안 해도 되는데" 소리가 계속 들림
  ES 관련 코드 작성에 기능 개발 시간의 50% 이상 소비
  감사 로그 요구사항이 없는 도메인에 ES 적용
```

### 3. 안티패턴 3 — 이벤트 스키마 잦은 변경

```
이벤트 스키마 변경의 누적 비용:

변경 1회 비용:
  Upcaster 작성: 2시간
  Upcaster 테스트: 2시간
  Projection 재구축 확인: 1시간
  → 총 5시간

6개월 후 변경 10회:
  Upcaster 10개 관리
  이벤트 로드마다 Upcaster 체인 실행 (성능)
  팀원들이 각 이벤트 버전 이해 어려움
  신규 팀원 온보딩: "왜 이 이벤트는 v3까지 있어요?" 설명 필요

이벤트 스키마 불안정의 근본 원인:
  ① 도메인 언어가 아닌 DB 컬럼 이름을 이벤트 필드에 사용
     customerId → userId (리팩토링 시 변경)
  ② 구현 세부사항이 이벤트에 포함
     internalOrderCode (내부 코드가 바뀌면 이벤트도 변경)
  ③ 읽기 모델 요구사항을 이벤트에 반영
     "읽기 모델에서 필요하다" → 이벤트에 필드 추가
     읽기 모델이 바뀌면 이벤트도 변경

안정적인 이벤트 설계 원칙:
  도메인 언어 사용 (account, not dbColumn)
  의미 있는 행동 표현 (MoneyWithdrawn, not BalanceUpdated)
  최소 필요 데이터만 포함 (읽기 모델 요구사항 제외)
  새 필드는 Optional로 추가 → Upcasting 불필요
```

### 4. 안티패턴 4 — 읽기 모델에서 쓰기 결정

```
읽기 모델로 불변식 검증하는 안티패턴:

  // ❌ 읽기 모델(최종 일관성)로 재고 확인
  @CommandHandler
  public void handle(OrderItemCommand cmd) {
      InventoryView view = inventoryViewRepo.findById(cmd.productId());
      if (view.getStock() < cmd.quantity()) {
          throw new InsufficientStockException(); // 읽기 모델 기준 검증!
      }
      // 주문 생성...
  }
  // 읽기 모델 업데이트 지연 → 실제 재고 = 0인데 view.stock = 5
  // → 재고 없는 상품 주문 가능

  // ✅ 쓰기 모델(Aggregate)에서 불변식 검증
  @CommandHandler
  public void handle(ReserveStockCommand cmd) {
      // Aggregate에서 직접 검증 (이벤트 스토어 기반, 정확한 현재 상태)
      if (availableStock < cmd.quantity())
          throw new InsufficientStockException();
      apply(new StockReserved(...));
  }

읽기 모델은 읽기에만:
  조회 화면 표시 → 읽기 모델
  비즈니스 결정 → 쓰기 모델 (Aggregate)
  예외: 일부 비즈니스 결정이 약한 일관성 수용 가능한 경우
```

---

## 💻 실전 코드

```java
// ✅ 안티패턴 1 — 가짜 CQRS 진단 테스트

// 진단 체크리스트:
// 1. Query Service가 읽기 모델 테이블을 사용하는가?
// 2. 읽기 모델 테이블이 쓰기 모델과 다른 스키마인가?

// ❌ 가짜 CQRS (같은 테이블 사용)
@Service
public class OrderQueryService {
    @Autowired OrderRepository orderRepo; // orders 테이블

    public List<Order> getOrders() {
        return orderRepo.findByStatus("CONFIRMED"); // 쓰기 모델 직접 조회
        // 읽기 최적화 없음, JOIN 여전히 발생
    }
}

// ✅ 진짜 CQRS (분리된 읽기 모델)
@Service
@Transactional(readOnly = true)
public class OrderQueryService {
    @Autowired OrderSummaryRepository summaryRepo; // order_summary 테이블 (읽기 모델)

    public List<OrderSummaryView> getOrders() {
        return summaryRepo.findByStatus("CONFIRMED");
        // 비정규화 테이블 → JOIN 없음, 인덱스 최적화
    }
}

// ✅ 안티패턴 2 — ES 도입 전 체크리스트
public class EventSourcingNecessityChecker {

    public boolean isEventSourcingNecessary(DomainContext ctx) {
        boolean hasAuditRequirement = ctx.requiresAuditLog();
        boolean hasTimeTravelNeed = ctx.requiresHistoricalStateQuery();
        boolean hasComplexInvariants = ctx.hasComplexStateTransitions();
        boolean hasProjectionRebuildNeed = ctx.needsFlexibleReadModels();

        // 4가지 중 하나도 해당 없으면 ES 불필요
        if (!hasAuditRequirement && !hasTimeTravelNeed
                && !hasComplexInvariants && !hasProjectionRebuildNeed) {
            log.warn("도메인 '{}' - Event Sourcing 도입 근거 없음. 상태 저장 권장.",
                ctx.domainName());
            return false;
        }
        return true;
    }
}

// ✅ 안티패턴 3 — 안정적인 이벤트 설계

// ❌ 불안정한 이벤트 (구현 세부사항 포함)
public record OrderPlaced_BAD(
    String orderId,
    String userId,          // customerId → userId로 리팩토링 시 변경 필요
    String dbOrderCode,     // 내부 코드 → 변경 가능성 높음
    List<CartItem> cartItems, // 쇼핑카트 구현체 노출
    String couponDbId       // DB ID → 도메인 언어 아님
) {}

// ✅ 안정적인 이벤트 (도메인 언어 사용)
public record OrderPlaced(
    String orderId,
    String customerId,         // 도메인 언어
    List<OrderLineItem> items, // 도메인 개념
    BigDecimal totalAmount,    // 사전 계산 (계산 로직 변경 시 이벤트 불변)
    String appliedCouponCode,  // 코드(사람이 읽는 값), ID 아님
    Instant occurredAt
) {}

// ✅ 안티패턴 4 — 읽기/쓰기 모델 역할 명확화
@Component
public class ArchitectureGuard {

    // ArchUnit으로 아키텍처 규칙 강제
    @Test
    void commandHandlers_should_not_use_readModel() {
        noClasses()
            .that().areAnnotatedWith(CommandHandler.class)
            .should().dependOnClassesThat()
            .resideInAPackage("..readmodel..")
            .check(importedClasses);
        // Command Handler가 읽기 모델 Repository 사용하면 빌드 실패
    }
}
```

---

## 📊 패턴 비교

```
안티패턴별 감지 지표:

┌─────────────────────────────┬─────────────────────────────────────┐
│ 안티패턴                    │ 조기 감지 신호                       │
├─────────────────────────────┼─────────────────────────────────────┤
│ 가짜 CQRS (패키지만 분리)   │ Query Service가 쓰기 DB 테이블 직접 │
│                             │ 조회, JOIN 쿼리 여전히 존재          │
├─────────────────────────────┼─────────────────────────────────────┤
│ ES 무분별 적용              │ 감사/시간여행 요구사항 없는 도메인   │
│                             │ 에 ES 적용, "왜 ES?" 설명 어려움     │
├─────────────────────────────┼─────────────────────────────────────┤
│ 잦은 이벤트 스키마 변경     │ Upcaster > 3개/월, 이벤트에 DB 컬럼 │
│                             │ 이름 또는 내부 코드 포함             │
├─────────────────────────────┼─────────────────────────────────────┤
│ 읽기 모델로 쓰기 결정       │ CommandHandler에서 View/Summary      │
│                             │ Repository 사용                      │
└─────────────────────────────┴─────────────────────────────────────┘
```

---

## ⚖️ 트레이드오프

```
안티패턴 회피 비용:
  진짜 CQRS: 읽기 모델 테이블 관리 추가 비용
  ES 선택적 적용: 도메인별 다른 패턴 → 일관성 vs 적합성
  안정적 이벤트 설계: 초기 설계 시간 투자 → 장기 유지보수 절약

각 안티패턴이 주는 단기 이점:
  가짜 CQRS: 빠른 구현 (패키지만 나누면 됨)
  ES 전면 적용: "우리는 ES 한다"는 일관성
  잦은 스키마 변경: 즉각적 요구사항 반영
  → 단기 이점 > 장기 비용임을 인지해야 안티패턴 회피 가능
```

---

## 📌 핵심 정리

```
4가지 핵심 안티패턴:

1. 가짜 CQRS:
   패키지 분리 ≠ CQRS
   읽기 모델이 쓰기 모델과 다른 스키마여야 진짜 CQRS

2. ES 무분별 적용:
   감사 로그 / 시간 여행 요구사항 없으면 ES 불필요
   단순 CRUD 도메인 → 상태 저장 유지

3. 잦은 이벤트 스키마 변경:
   도메인 언어 사용, 구현 세부사항 제외
   새 필드는 Optional로 추가 → Upcaster 최소화

4. 읽기 모델로 쓰기 결정:
   불변식 검증은 항상 쓰기 모델(Aggregate)에서
   읽기 모델 = 표시 전용 (최종 일관성 허용)

공통 감지법:
   "왜 이게 필요한가?" 질문에 답 못하면 안티패턴 신호
```

---

## 🤔 생각해볼 문제

**Q1.** "패키지만 분리한 가짜 CQRS"에서 진짜 CQRS로 전환하는 가장 작은 첫 번째 단계는 무엇인가?

<details>
<summary>해설 보기</summary>

가장 느린 조회 쿼리 하나를 위한 읽기 모델 테이블을 만드는 것입니다.

전체 시스템을 한 번에 바꾸려 하지 말고, 구체적인 성능 문제가 있는 조회 하나를 선택합니다. 예를 들어 `getOrderList()`가 5개 테이블 JOIN으로 200ms 걸린다면, `order_summary` 테이블을 만들어 이 조회만 읽기 모델로 전환합니다.

이 과정에서 이중 쓰기(Command Service가 orders 테이블 + order_summary 테이블 동시 업데이트)를 잠시 허용합니다. 조회 성능이 200ms → 5ms로 개선되면 팀이 CQRS의 실제 가치를 경험합니다.

작은 성공 하나가 팀의 신뢰를 만들고, 다음 읽기 모델 분리로 자연스럽게 이어집니다. 처음부터 "전체 CQRS 도입"을 목표로 하면 부담이 너무 크고 포기하기 쉽습니다.

</details>

---

**Q2.** Event Sourcing을 적용하기로 결정했는데, 3개월 후 이벤트 Upcaster가 8개가 됐다. 이것이 설계 문제인지 정상인지 어떻게 판단하는가?

<details>
<summary>해설 보기</summary>

Upcaster 수 자체보다 이유가 중요합니다.

정상적인 경우는 도메인이 실제로 변화하면서 이벤트도 진화한 것입니다. 새로운 비즈니스 요구사항으로 이벤트에 새 필드가 추가됐거나, 법적 요구사항 변경으로 이벤트 구조가 바뀐 경우입니다. 이런 Upcaster는 피할 수 없는 기술 부채입니다.

설계 문제 신호는 다음과 같습니다. 첫째, Upcaster가 DB 컬럼 이름 변경을 반영합니다. 이벤트에 `userId` → `customerId`처럼 내부 리팩토링이 이벤트 변경으로 이어진다면 이벤트에 구현 세부사항이 포함된 것입니다. 둘째, 읽기 모델 요구사항이 바뀔 때마다 이벤트가 변경됩니다. 이벤트는 도메인 사실이어야지, 읽기 모델 스키마가 아닙니다. 셋째, 동일한 이벤트가 두 달 안에 v1 → v2 → v3로 세 번 변경됐습니다. 도메인 개념이 아직 안정화되지 않은 신호입니다.

해결책으로는 이벤트 설계 원칙 문서화, 이벤트 변경 전 팀 리뷰 프로세스 도입, 그리고 도메인이 충분히 안정화될 때까지 ES 도입을 미루는 것을 검토합니다.

</details>

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[이전: 점진적 도입 전략 ⬅️](./03-incremental-adoption.md)** | **[다음: CQRS / ES의 실제 비용과 편익 ➡️](./05-cost-benefit-analysis.md)**

</div>
