# 쓰기 모델과 읽기 모델의 근본적 차이

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 쓰기 모델이 불변식을 보호한다는 것이 구체적으로 무엇을 의미하는가?
- 읽기 모델이 비정규화되어야 하는 이유는 무엇이고, 어느 수준까지 비정규화해야 하는가?
- 두 모델의 일관성은 어떻게 정의하고, "최종 일관성"은 어디까지 허용 가능한가?
- 단일 모델로 두 요구사항을 억지로 합쳤을 때 각각이 얼마나 타협하는가?
- 쓰기 모델과 읽기 모델이 물리적으로 분리된다는 것이 실제로 무엇을 의미하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

CQRS를 "Command 패키지와 Query 패키지를 나누는 것"으로 이해하면, 결국 같은 Entity를 두 패키지가 함께 참조하는 구조가 된다. 이 구조는 아무것도 해결하지 못한다. CQRS의 핵심은 패키지 분리가 아니라 **쓰기 모델과 읽기 모델이 서로 다른 요구사항을 가진다는 사실을 설계에 명시적으로 반영하는 것**이다.

쓰기 모델은 불변식을 보호하고, 읽기 모델은 조회를 최적화한다. 이 두 목적은 서로 다른 데이터 형태, 서로 다른 저장소, 서로 다른 일관성 수준을 요구한다. 이 차이를 명확히 이해해야 왜 분리가 필요한지, 얼마나 분리해야 하는지 판단할 수 있다.

---

## 😱 흔한 실수 (Before — 단일 모델로 두 요구사항을 억지로 충족)

```
상황: 전자상거래 — 주문 처리 + 주문 현황 대시보드

억지 단일 모델:
  @Entity
  class Order {
      Long id;
      String customerId;
      OrderStatus status;
      List<OrderItem> items;

      // 불변식 보호 (쓰기)
      void confirm() {
          if (status != PENDING) throw new IllegalStateException();
          validateItemsInStock();   // 재고 확인
          this.status = CONFIRMED;
      }

      // 대시보드 조회를 위해 추가된 필드 (읽기)
      String customerName;         // ← user 테이블 복사
      String customerTier;         // ← VIP/일반 구분
      BigDecimal totalAmount;      // ← items 합계 사전 계산
      String deliveryAddress;      // ← address 테이블 복사
      Integer itemCount;           // ← 집계 캐시
  }

쓰기 관점에서의 타협:
  confirm() 불변식 검증 로직이 읽기 필드들과 같은 클래스에 혼재
  → Entity 로드 시 읽기 필드도 항상 함께 로드 (불필요한 데이터)
  → 불변식 변경 시 읽기 필드에 미치는 영향을 항상 고려해야 함

읽기 관점에서의 타협:
  대시보드에서 "고객사별 주문 통계"가 필요
  → Order에는 없는 필드 → 또 추가 or 복잡한 쿼리
  → items 리스트가 N+1의 원인이 되지만 제거 불가 (불변식 검증에 필요)
  → 정규화된 구조라 조회 시 JOIN 불가피
```

---

## ✨ 올바른 접근 (After — 목적에 따라 완전히 다른 두 모델)

```
쓰기 모델 — 불변식에 필요한 데이터만:
  class Order {
      OrderId id;
      CustomerId customerId;
      OrderStatus status;
      List<OrderLine> lines;   // 재고 확인에 필요

      void confirm() {
          if (!status.canTransitionTo(CONFIRMED))
              throw new InvalidStatusTransitionException(status, CONFIRMED);
          lines.forEach(line -> line.validateStock());
          this.status = CONFIRMED;
          registerEvent(new OrderConfirmed(this.id, this.customerId, this.lines));
      }
  }
  // 저장소: orders 테이블 (or 이벤트 스토어)
  // 관심사: 불변식, 상태 전이, 이벤트 발행

읽기 모델 — 대시보드 조회에 맞춘 비정규화 뷰:
  class OrderSummaryView {
      String orderId;
      String customerName;    // 비정규화 — JOIN 없음
      String customerTier;    // 비정규화
      String status;
      BigDecimal totalAmount; // 사전 계산
      Integer itemCount;      // 사전 계산
      LocalDateTime createdAt;
      String deliveryCity;    // 지역별 통계용
  }
  // 저장소: order_summary 테이블 (or Redis, Elasticsearch)
  // 관심사: 빠른 조회, 집계, 필터링

동기화:
  OrderConfirmed 이벤트
    → OrderSummaryProjection.on(OrderConfirmed)
    → order_summary 테이블 업데이트
  → 쓰기 완료 후 수 ms~수 초 내 읽기 모델 반영 (최종 일관성)
```

---

## 🔬 내부 동작 원리

### 1. 쓰기 모델이 보호하는 것 — 불변식(Invariant)

```
불변식이란:
  도메인 규칙 중 "항상 참이어야 하는 조건"
  비즈니스가 절대 허용하지 않는 상태

은행 계좌의 불변식:
  ① 잔고 >= 0  (출금 후 잔고가 음수가 되면 안 됨)
  ② 해지된 계좌는 입출금 불가
  ③ 이체 금액 > 0  (0원 이하 이체 불가)
  ④ 동결된 계좌는 출금 불가

쓰기 모델이 불변식을 보호하는 구조:
  account.withdraw(Money.of(1000000));

  내부 처리:
    1. balance.isLessThan(Money.of(1000000)) → true → 예외
    2. status == FROZEN → 예외
    3. amount.isNegativeOrZero() → 예외
    4. 검증 통과 → 출금 실행

핵심:
  불변식 보호는 트랜잭션 내에서 이루어져야 함
  → 여러 클라이언트가 동시에 같은 계좌에서 출금 시도
  → 각각 독립적으로 잔고 확인 → 둘 다 성공 → 잔고 초과 출금

  해결: Aggregate를 단위로 잠금 (Version 기반 낙관적 잠금)
    account.version = 5
    쓰기 트랜잭션 A: balance 300,000 확인 → 200,000 출금 예정
    쓰기 트랜잭션 B: balance 300,000 확인 → 250,000 출금 예정
    A 커밋: UPDATE account SET balance=100000, version=6 WHERE id=? AND version=5  → 성공
    B 커밋: UPDATE account SET balance=50000, version=6 WHERE id=? AND version=5   → 실패 (version 불일치)
    → B 롤백, 재시도

읽기 모델이 불변식 보호에 불필요한 이유:
  "현재 잔고 > 0인가?" 확인에 필요한 것: balance 필드만
  ownerName, lastLoginAt, totalDepositAmount, monthlyTxCount
  → 불변식 검증에 전혀 불필요
  → 쓰기 모델에 이 필드들이 있으면 불필요한 데이터를 항상 로드해야 함
```

### 2. 읽기 모델이 비정규화되어야 하는 이유

```
정규화 (3NF) — 쓰기에 최적화된 구조:

  orders (id, customer_id, status, created_at)
  order_lines (id, order_id, product_id, quantity, unit_price)
  customers (id, name, email, tier, address_id)
  addresses (id, street, city, postal_code)
  products (id, name, sku, category)

  특징:
    데이터 중복 없음
    갱신 이상 없음 (고객 이름 변경 시 customers 테이블 1곳만 수정)
    INSERT/UPDATE/DELETE 빠름

  대시보드 쿼리의 현실:
    "고객명 + 주문 상태 + 총 금액 + 상품 수 + 배송 도시"

    SELECT
      o.id,
      c.name as customer_name,
      c.tier as customer_tier,
      o.status,
      SUM(ol.quantity * ol.unit_price) as total_amount,
      COUNT(ol.id) as item_count,
      a.city as delivery_city,
      o.created_at
    FROM orders o
    JOIN customers c ON o.customer_id = c.id
    JOIN addresses a ON c.address_id = a.id
    JOIN order_lines ol ON ol.order_id = o.id
    GROUP BY o.id, c.name, c.tier, o.status, a.city, o.created_at
    ORDER BY o.created_at DESC
    LIMIT 50;

    문제:
      4개 테이블 JOIN
      GROUP BY와 집계 함수 (total_amount, item_count)
      페이지네이션 시 정렬 + GROUP BY 조합 → 성능 불안정
      인덱스 설계가 쓰기에 맞춰져 있어 이 쿼리를 최적화하기 어려움

비정규화 읽기 모델 (조회에 최적화):

  order_summary (
    order_id,
    customer_name,    ← customers.name 복사
    customer_tier,    ← customers.tier 복사
    status,
    total_amount,     ← sum(quantity * unit_price) 사전 계산
    item_count,       ← count(order_lines) 사전 계산
    delivery_city,    ← addresses.city 복사
    created_at
  )

  대시보드 쿼리:
    SELECT * FROM order_summary ORDER BY created_at DESC LIMIT 50;
    → JOIN 없음, 집계 없음, 단순 SELECT
    → 커버링 인덱스 가능 (created_at 기준 정렬 + 필요 필드 포함)
    → 쿼리 성능 예측 가능

비정규화의 트레이드오프:
  장점: 조회 빠름, 인덱스 단순, 쿼리 명확
  단점: 데이터 중복 (customer_name이 여러 order_summary에 존재)
        고객 이름 변경 시 order_summary도 업데이트 필요

  → 이 트레이드오프를 명시적으로 선택하는 것이 CQRS
```

### 3. 일관성 수준의 차이

```
쓰기 모델의 일관성:
  강한 일관성 (Strong Consistency)
  트랜잭션 커밋 즉시 데이터 일관성 보장
  "출금 후 잔고가 즉시 차감됨"

읽기 모델의 일관성:
  최종 일관성 (Eventual Consistency)
  쓰기 완료 후 일정 시간 후 읽기 모델 반영
  "출금 완료 → 0ms ~ 수 초 후 → 대시보드 잔고 업데이트"

타임라인:
  T=0ms   이체 Command 수신
  T=5ms   Account Aggregate 잔고 검증
  T=6ms   출금 이벤트 이벤트 스토어 저장 (강한 일관성 완료)
  T=7ms   Kafka에 MoneyWithdrawn 이벤트 발행
  T=50ms  Projection이 이벤트 수신
  T=52ms  account_summary.balance 업데이트 (읽기 모델 반영)

  T=10ms에 대시보드 조회: 아직 구 잔고 표시 (최종 일관성 — 수락 가능)
  T=60ms에 대시보드 조회: 새 잔고 표시 (최종 일관성 도달)

최종 일관성을 허용할 수 없는 케이스:
  "방금 이체한 잔고가 즉시 대시보드에 보여야 한다"
  → 해결책 1: 이체 직후 클라이언트 측 낙관적 UI 업데이트
  → 해결책 2: 이체 결과를 API 응답에 포함 (Query 없이 Command 결과 활용)
  → 해결책 3: 읽기 모델을 쓰기와 동일 DB에 두는 Simple CQRS

  "잔고 부족 여부 확인에 읽기 모델을 사용해도 되는가?"
  → 절대 안 됨! 잔고 확인은 반드시 쓰기 모델(Aggregate)에서 수행
  → 읽기 모델의 잔고는 수십 ms 지연 가능
  → 읽기 모델 기준으로 출금 허용 시 동시 요청에서 초과 출금 발생
```

### 4. 단일 모델에서 각 요구사항이 타협하는 구조

```
쓰기 요구사항이 읽기로 인해 타협하는 경우:

  시나리오: 재고 확인이 필요한 주문 확인(confirm) 불변식

  단일 모델:
    class Order {
        void confirm() {
            items.forEach(item -> {
                Product product = productRepo.findById(item.getProductId());
                if (product.getStock() < item.getQuantity())
                    throw new OutOfStockException(item.getProductId());
            });
            this.status = CONFIRMED;
        }
    }

  조회 요구사항 추가:
    "주문 확인 화면에서 상품 이미지, 카테고리 이름, 브랜드명도 보여주세요"
    → items에 productImageUrl, categoryName, brandName 추가
    → confirm()이 실행될 때 이 필드들도 함께 로드됨
    → 불변식 검증에 불필요한 데이터를 매번 로드
    → Aggregate가 커질수록 confirm() 실행이 느려짐

  분리 후:
    쓰기: Order.confirm() → items의 productId와 quantity만으로 재고 확인
    읽기: OrderDetailView → productImageUrl, categoryName, brandName 포함

읽기 요구사항이 쓰기로 인해 타협하는 경우:

  시나리오: 주문 검색 기능 (상품명으로 검색)

  단일 모델:
    items 테이블: order_id, product_id, quantity, unit_price
    product 테이블: id, name, sku, category
    → 상품명으로 주문 검색: orders JOIN order_items JOIN products WHERE products.name LIKE ?
    → LIKE 검색 + JOIN → 인덱스 활용 불가, 전체 테이블 스캔 위험

  분리 후:
    읽기 모델: Elasticsearch에 order_search 인덱스
      { "orderId": "ORD-001", "productNames": ["아이폰 15", "에어팟"], "customerName": "김철수" }
    → 상품명 전문 검색: GET /order_search?q=아이폰
    → 주 DB 부하 없음, 검색 최적화 독립적으로 가능
```

---

## 💻 실전 코드

### 쓰기 모델 — 불변식 중심 설계

```java
// ✅ 쓰기 모델 — 불변식 보호에 필요한 것만
public class Order {  // 순수 도메인 모델

    private final OrderId id;
    private final CustomerId customerId;
    private OrderStatus status;
    private final List<OrderLine> lines;

    // 불변식: 주문 총액 >= 최소 주문 금액
    private static final Money MINIMUM_ORDER_AMOUNT = Money.of(1000);

    public void confirm(InventoryChecker inventoryChecker) {
        // 불변식 1: 상태 전이 검증
        if (!status.canTransitionTo(CONFIRMED)) {
            throw new InvalidStatusTransitionException(status, CONFIRMED);
        }

        // 불변식 2: 재고 확인
        Map<ProductId, Integer> stockMap = inventoryChecker.checkStock(
            lines.stream()
                .collect(toMap(OrderLine::getProductId, OrderLine::getQuantity))
        );
        lines.forEach(line -> {
            int available = stockMap.getOrDefault(line.getProductId(), 0);
            if (available < line.getQuantity()) {
                throw new OutOfStockException(line.getProductId(), line.getQuantity(), available);
            }
        });

        // 불변식 3: 최소 주문 금액 확인
        Money total = lines.stream()
            .map(OrderLine::subtotal)
            .reduce(Money.ZERO, Money::add);
        if (total.isLessThan(MINIMUM_ORDER_AMOUNT)) {
            throw new MinimumOrderAmountException(total, MINIMUM_ORDER_AMOUNT);
        }

        this.status = CONFIRMED;

        // 이벤트 발행 — 읽기 모델 동기화의 트리거
        registerEvent(new OrderConfirmed(
            this.id,
            this.customerId,
            this.lines,
            total,
            LocalDateTime.now()
        ));
    }

    // 쓰기 모델에는 조회 관련 메서드 없음
    // getCustomerName()? → 없음
    // getTotalAmountFormatted()? → 없음
    // getDeliveryAddress()? → 없음
}
```

### 읽기 모델 — 화면 목적별 완전 분리

```java
// ✅ 읽기 모델 1 — 주문 목록 대시보드
@Entity
@Table(name = "order_summary")
public class OrderSummaryView {

    @Id
    private String orderId;
    private String customerId;
    private String customerName;    // 비정규화 (customers 테이블 JOIN 불필요)
    private String customerTier;    // 비정규화 (VIP 여부 표시)
    private String status;
    private BigDecimal totalAmount; // 사전 계산 (order_lines SUM 불필요)
    private Integer itemCount;      // 사전 계산 (order_lines COUNT 불필요)
    private String deliveryCity;    // 비정규화 (addresses JOIN 불필요)
    private LocalDateTime createdAt;
    private LocalDateTime updatedAt;
}

// ✅ 읽기 모델 2 — 주문 상세 화면
@Entity
@Table(name = "order_detail")
public class OrderDetailView {

    @Id
    private String orderId;
    private String customerId;
    private String customerName;
    private String customerEmail;
    private String customerPhone;
    private String status;

    @Type(JsonType.class)
    @Column(columnDefinition = "jsonb")
    private List<OrderLineDetail> lines; // 상품 이미지 URL, 카테고리명 포함

    private String deliveryStreet;
    private String deliveryCity;
    private String deliveryPostalCode;
    private BigDecimal totalAmount;
    private BigDecimal discountAmount;
    private BigDecimal finalAmount;
    private String paymentMethod;
    private LocalDateTime confirmedAt;
}

// ✅ 읽기 모델 3 — Elasticsearch 검색 인덱스 (별도 저장소)
// document 구조:
// {
//   "orderId": "ORD-001",
//   "customerName": "김철수",
//   "status": "CONFIRMED",
//   "productNames": ["아이폰 15 Pro", "에어팟 Pro 2세대"],
//   "productSkus": ["IPHONE15PRO-256", "AIRPODS-PRO2"],
//   "totalAmount": 2350000,
//   "createdAt": "2024-01-15T09:00:00"
// }

// ✅ 프로젝션 — OrderConfirmed 이벤트로 읽기 모델 업데이트
@Component
public class OrderProjection {

    @EventHandler
    public void on(OrderConfirmed event) {
        // 읽기 모델 1 업데이트: order_summary
        Customer customer = customerQueryRepository.findById(event.getCustomerId());
        Address address = addressQueryRepository.findByCustomerId(event.getCustomerId());

        OrderSummaryView summary = new OrderSummaryView();
        summary.setOrderId(event.getOrderId().value());
        summary.setCustomerId(event.getCustomerId().value());
        summary.setCustomerName(customer.getName());        // 비정규화
        summary.setCustomerTier(customer.getTier().name()); // 비정규화
        summary.setStatus("CONFIRMED");
        summary.setTotalAmount(event.getTotalAmount().amount());
        summary.setItemCount(event.getLines().size());      // 사전 계산
        summary.setDeliveryCity(address.getCity());         // 비정규화
        summary.setCreatedAt(event.getOccurredAt());

        orderSummaryRepository.save(summary);

        // 읽기 모델 2 업데이트: order_detail
        // ...

        // 읽기 모델 3 업데이트: Elasticsearch
        // ...
    }
}
```

---

## 📊 패턴 비교

```
쓰기 모델 vs 읽기 모델 설계 원칙 비교:

┌─────────────────────┬──────────────────────────┬──────────────────────────┐
│ 항목                │ 쓰기 모델                 │ 읽기 모델                 │
├─────────────────────┼──────────────────────────┼──────────────────────────┤
│ 주요 목적           │ 불변식 보호, 상태 변경    │ 빠른 조회, 화면 데이터    │
├─────────────────────┼──────────────────────────┼──────────────────────────┤
│ 데이터 형태         │ 정규화                   │ 비정규화 (화면 기준)      │
├─────────────────────┼──────────────────────────┼──────────────────────────┤
│ 일관성 수준         │ 강한 일관성              │ 최종 일관성 (허용 시)     │
├─────────────────────┼──────────────────────────┼──────────────────────────┤
│ 저장소              │ 관계형 DB (or 이벤트 스토어) │ RDB / Redis / ES / 목적별 │
├─────────────────────┼──────────────────────────┼──────────────────────────┤
│ 변경 주체           │ Command Handler          │ Projection (이벤트 기반) │
├─────────────────────┼──────────────────────────┼──────────────────────────┤
│ 잠금                │ 낙관적 잠금 (Version)    │ 잠금 없음 (읽기 전용)    │
├─────────────────────┼──────────────────────────┼──────────────────────────┤
│ 인덱스 설계         │ 쓰기 성능 최적화         │ 조회 패턴 최적화          │
├─────────────────────┼──────────────────────────┼──────────────────────────┤
│ 크기                │ 최소한의 필드            │ 화면에 필요한 모든 필드  │
├─────────────────────┼──────────────────────────┼──────────────────────────┤
│ 개수                │ 도메인당 1개             │ 화면/목적별 여러 개 가능 │
└─────────────────────┴──────────────────────────┴──────────────────────────┘

단일 모델이 두 요구사항을 수용할 때의 타협점:

  쓰기가 포기하는 것:
    - Aggregate 순수성 (읽기 필드가 혼재)
    - 로드 성능 (불필요한 필드까지 항상 로드)
    - 불변식 명확성 (잡동사니 사이에 묻힘)

  읽기가 포기하는 것:
    - 조회 성능 (정규화로 인한 JOIN 필요)
    - 화면별 최적화 (모든 화면이 같은 데이터 형태 사용)
    - 독립적 스케일 아웃 (쓰기 저장소와 분리 불가)
```

---

## ⚖️ 트레이드오프

```
분리의 이점:
  ① 쓰기 모델 독립성: 불변식 보호 로직이 명확, 테스트 용이
  ② 읽기 성능: 조회 화면에 맞는 비정규화로 JOIN 제거
  ③ 독립적 스케일 아웃: 읽기 부하에만 읽기 저장소 확장
  ④ 화면별 최적화: 관리자/사용자/통계 뷰 독립적으로 설계
  ⑤ 저장소 다양화: 읽기 모델에 Redis, Elasticsearch 등 활용 가능

분리의 비용:
  ① 읽기 모델 동기화 로직 필요 (Projection 구현)
  ② 최종 일관성 허용 필요 (ms~초 단위 지연)
  ③ 운영 복잡도: 읽기 모델 저장소 추가 관리
  ④ 데이터 중복: 비정규화로 인한 저장 용량 증가
  ⑤ 읽기 모델 재구축 전략 필요 (프로젝션 오염 시)

최종 일관성 수락 여부 판단 기준:
  수락 가능:
    - 대시보드/통계 화면 (수 초 지연 허용)
    - 검색 결과 (검색 인덱스 갱신 지연 허용)
    - 이메일/알림 발송 대상 목록

  수락 불가:
    - 다음 Command의 전제 조건이 되는 데이터 (잔고 확인)
    - 결제 전 최종 금액 확인 (쓰기 모델에서 직접 확인)
    - 동시 예약에서 중복 방지 (쓰기 모델 잠금 필요)
```

---

## 📌 핵심 정리

```
쓰기 모델과 읽기 모델의 근본적 차이:

쓰기 모델:
  목적: 불변식 보호, 도메인 규칙 실행
  특징: 정규화, 트랜잭션, 낙관적 잠금, 최소한의 필드
  저장: 관계형 DB (정규화) or 이벤트 스토어 (Event Sourcing)
  일관성: 강한 일관성 (트랜잭션 완료 즉시)

읽기 모델:
  목적: 빠른 조회, 화면 데이터 제공
  특징: 비정규화, 잠금 없음, 화면별 최적화, 목적별 여러 모델
  저장: RDB (비정규화) / Redis (캐시) / Elasticsearch (검색)
  일관성: 최종 일관성 (이벤트 처리 후 반영)

단일 모델의 구조적 한계:
  두 요구사항이 동시에 존재할 때 각각이 서로를 타협하게 만듦
  분리는 각 모델이 본래 목적에만 집중할 수 있게 함

핵심 판단:
  "이 데이터가 불변식 보호에 필요한가?" → 쓰기 모델
  "이 데이터가 조회 화면을 위한 것인가?" → 읽기 모델
```

---

## 🤔 생각해볼 문제

**Q1.** 읽기 모델이 비정규화되어 있을 때, 고객 이름이 변경되면 해당 고객의 모든 주문 summary에서 이름을 업데이트해야 한다. 이 동기화 문제를 어떻게 설계하는가?

<details>
<summary>해설 보기</summary>

CustomerNameChanged 이벤트를 발행하고 Projection이 처리하는 방식이 권장됩니다.

```java
@EventHandler
public void on(CustomerNameChanged event) {
    // 해당 고객의 모든 주문 summary 업데이트
    orderSummaryRepository.updateCustomerName(
        event.getCustomerId(),
        event.getNewName()
    );
    // UPDATE order_summary SET customer_name = ? WHERE customer_id = ?
}
```

이 접근의 중요한 판단이 있습니다. "과거 주문에서 고객 이름이 변경되어야 하는가?" 입니다. 비즈니스 요구사항에 따라 다를 수 있습니다. 주문 당시의 이름을 보존해야 한다면(법적 증거, 계약 등) 오히려 업데이트하지 않는 것이 맞습니다. 현재 고객 이름을 항상 최신으로 유지해야 한다면 이벤트 기반 동기화가 맞습니다.

Event Sourcing을 사용한다면 Projection 재구축 시 최신 이름으로 다시 계산할 수도 있습니다.

</details>

---

**Q2.** 쓰기 모델에서 Command 응답으로 바로 읽기 모델 데이터를 반환하는 패턴이 있다. 이것은 CQRS를 위반하는가?

<details>
<summary>해설 보기</summary>

CQRS의 핵심 원칙은 "상태를 변경하는 작업과 데이터를 반환하는 작업을 분리"하는 것입니다. Command가 완료 후 읽기 모델 데이터를 함께 반환하는 것은 엄밀한 CQRS에서는 위반으로 볼 수 있지만, 실용적 관점에서는 허용되는 경우가 많습니다.

허용되는 패턴의 예로는 생성 Command 후 생성된 리소스의 ID 반환(`return order.getId()`), 이체 Command 후 새 잔고 반환(Eventual Consistency 지연을 피하기 위한 실용적 선택), CQRS Journey(Microsoft)에서도 Command 결과에 최소한의 정보 반환을 허용합니다.

진짜 위반은 Command Handler가 복잡한 읽기 쿼리를 실행하고 비정규화된 뷰 데이터를 직접 조합해 반환하는 경우입니다. 이 경우 쓰기 모델이 읽기 관심사를 흡수하게 됩니다.

</details>

---

**Q3.** 읽기 모델이 여러 개일 때 같은 이벤트를 소비하는 Projection이 여러 개 있다. 하나의 Projection 처리가 실패하면 다른 Projection에 영향을 주는가?

<details>
<summary>해설 보기</summary>

Projection 격리 설계에 따라 달라집니다.

Kafka Consumer Group을 Projection별로 분리한 경우, OrderSummaryProjection Consumer Group과 OrderSearchProjection Consumer Group은 각자 독립적으로 오프셋을 관리합니다. 하나가 실패해도 다른 그룹의 처리에 영향이 없습니다. 실패한 Projection은 재시작 후 실패한 오프셋부터 재처리합니다.

Axon Framework를 사용하는 경우, Tracking Event Processor가 Projection별로 독립적인 Token을 관리합니다. 하나의 Processor 실패가 다른 Processor를 차단하지 않습니다. `@ProcessingGroup` 어노테이션으로 같은 그룹 안의 Processor들은 순서 보장이 되지만, 다른 그룹과는 완전히 독립입니다.

따라서 Projection 격리가 제대로 설계되어 있다면, 하나의 Projection 실패가 다른 읽기 모델에 영향을 주지 않습니다.

</details>

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[이전: 단일 모델의 임피던스 불일치 ⬅️](./01-impedance-mismatch.md)** | **[다음: CQRS의 스펙트럼 ➡️](./03-cqrs-spectrum.md)**

</div>
