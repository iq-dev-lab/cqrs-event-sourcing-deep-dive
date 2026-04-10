# CQRS가 해결하는 실제 문제

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 복잡한 조회 쿼리가 Aggregate를 오염시킨다는 것이 구체적으로 어떤 현상인가?
- 읽기 성능을 위해 쓰기 모델이 타협해야 하는 상황은 어떻게 발생하는가?
- 역할별로 다른 뷰가 필요할 때 단일 모델이 왜 한계에 봉착하는가?
- CQRS는 이 세 가지 문제를 각각 어떻게 구조적으로 해결하는가?
- 해결하는 과정에서 생겨나는 새로운 복잡도는 무엇인가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

CQRS를 도입해야 하는 이유를 "좋은 아키텍처니까"로 설명하면 팀을 설득하기 어렵다. 구체적인 고통이 있어야 한다. 이 문서는 단일 모델이 만드는 세 가지 구체적인 고통을 코드 수준에서 추적하고, CQRS가 각각을 어떻게 구조적으로 해결하는지 보여준다. 아키텍처 선택은 이론이 아니라 현재 겪는 고통에서 시작해야 한다.

---

## 😱 흔한 실수 (Before — 단일 모델이 고통을 축적하는 과정)

```
전자상거래 플랫폼, 1년 운영 후 상황:

문제 1: Aggregate 메서드가 조회 로직을 품기 시작함
  "주문 확인 이메일에 상품 이미지가 필요합니다"
  → Order.getItems()에서 각 Item의 Product 정보 필요
  → Order에 Product 연관관계 추가
  → 주문 처리 로직과 이메일 템플릿 로직이 같은 Entity에

  "주문 취소 시 환불 계산이 필요합니다"
  → Order에 환불 정책 로직 추가
  → 쿠폰 할인, VIP 할인, 포인트를 Order가 알아야 함
  → Order가 CouponService, PointService를 의존

  결과: Order 클래스 2,000줄, 의존성 10개
  불변식 로직 찾기: 2,000줄 스크롤

문제 2: 읽기 성능을 위해 도입한 변경이 쓰기를 망침
  "주문 목록 API가 너무 느립니다"
  → 분석: order_lines와 products 테이블 JOIN이 원인
  → 해결책: Order에 totalAmount, itemCount 필드 추가 (사전 계산)
  → 결과: 주문 생성/수정 시 totalAmount, itemCount를 항상 동기화해야 함
  → 주문 생성 트랜잭션이 더 복잡해짐
  → 동기화 누락 버그 발생

문제 3: 역할별 뷰 요구사항이 폭발
  "관리자는 모든 주문 정보와 고객 내부 등급이 필요합니다"
  "배송 담당자는 배송지 + 연락처 + 상품 크기/무게만 필요합니다"
  "회계팀은 세금계산서 정보 + 결제 수단 + 정산 상태가 필요합니다"
  "CS팀은 고객 불만 이력 + 주문 + 반품 내역이 함께 필요합니다"
  → 네 개의 다른 뷰, 네 개의 다른 JOIN, 네 개의 다른 응답 구조
  → 모두 같은 Order Entity에서 파생 → Order는 모든 연관관계를 품어야 함
```

---

## ✨ 올바른 접근 (After — CQRS가 각 문제를 구조적으로 분리)

```
CQRS 도입 후:

문제 1 해결 — Aggregate 순수성 회복:
  쓰기 모델 (Order Aggregate):
    주문 생성, 확인, 취소의 불변식만
    Product? → 재고 확인에 필요한 productId + quantity만
    환불 계산? → OrderCancelled 이벤트 발행 → 환불 서비스가 이벤트 소비
    이메일 템플릿? → OrderConfirmed 이벤트 발행 → 이메일 서비스가 이벤트 소비

  Order 클래스: 300줄로 축소, 의존성 2개

문제 2 해결 — 읽기 모델이 읽기 성능을 독립적으로 최적화:
  order_summary 테이블:
    total_amount, item_count 사전 계산되어 저장
    쓰기 트랜잭션에서 동기화 불필요
    OrderConfirmed 이벤트 → Projection이 summary 업데이트
    쓰기 로직과 읽기 최적화가 완전히 분리

문제 3 해결 — 역할별 읽기 모델 독립 설계:
  admin_order_view: 모든 주문 정보 + 고객 내부 등급 (PostgreSQL)
  delivery_order_view: 배송지 + 연락처 + 상품 크기/무게 (PostgreSQL)
  accounting_order_view: 세금계산서 + 결제 + 정산 (PostgreSQL)
  cs_order_view: 고객 불만 이력 + 주문 + 반품 통합 (Elasticsearch)

  각 뷰: 독립적으로 설계, 독립적으로 인덱스, 독립적으로 스케일
  Order Aggregate: 이 뷰들의 존재를 모름
```

---

## 🔬 내부 동작 원리

### 1. 문제 1 — Aggregate가 조회 로직에 오염되는 메커니즘

```
오염이 시작되는 전형적인 패턴:

Sprint 1: 주문 도메인 설계
  class Order {
      OrderId id;
      CustomerId customerId;
      OrderStatus status;
      List<OrderLine> lines;

      void confirm() { ... }  // 불변식 보호
      void cancel() { ... }
  }

Sprint 3: 주문 확인 이메일 기능
  "이메일에 상품명, 이미지 URL, 배송 예정일이 필요합니다"
  → Product 정보가 필요 → OrderLine에 Product 연관관계 추가
  → Order → OrderLine → Product → ProductImage (4 depth)

Sprint 5: 주문 목록 대시보드
  "고객명, 총금액, 배송지 도시가 필요합니다"
  → Customer, Address 연관관계 추가
  → Order → Customer → Address (3 depth)

Sprint 7: 주문 통계 API
  "월별 카테고리별 매출이 필요합니다"
  → Order에서 items의 category 합산 로직 추가
  → Category 연관관계 추가

Sprint 9: 배송 추적 기능
  "주문 상태 + 배송 상태 + 현재 위치가 필요합니다"
  → Delivery Aggregate와 Order 연결
  → Order → Delivery (양방향 참조)

최종 Order 클래스 의존 관계:
  Order → OrderLine → Product → ProductImage
                               → ProductCategory
               → Customer → Address
               → Coupon
               → Delivery → DeliveryLocation
               → PaymentRecord
               → ReturnRequest

  Aggregate 경계가 무너짐:
    단일 Order 로드 = 10개 테이블 JOIN
    주문 하나 로드에 수십 ms
    불변식 검증에 필요 없는 데이터를 항상 로드
    Order를 수정하면 연결된 모든 테이블에 영향 파악 필요

CQRS 후 Order Aggregate 경계:
  class Order {
      OrderId id;
      CustomerId customerId;  // ID만 (Customer 객체 없음)
      OrderStatus status;
      List<OrderLine> lines;  // productId + quantity만 (재고 확인용)

      void confirm(InventoryChecker checker) {
          // 불변식: 재고 확인, 상태 전이만
          // Product 상세 정보, Customer 이름, 배송지: 없음
      }
  }
  → 연관관계: 0개 (ID 참조만)
  → 로드 = 단일 SELECT
  → 불변식 로직 명확
```

### 2. 문제 2 — 읽기 성능 개선이 쓰기를 복잡하게 만드는 메커니즘

```
단일 모델에서 읽기 성능 개선 시도들:

시도 1: 비정규화 필드 추가
  "주문 목록에서 매번 SUM(order_lines)를 계산하기 싫으니까
   Order에 total_amount 필드를 미리 계산해서 저장하자"

  문제:
    Order 생성 시: total_amount 계산하여 저장
    OrderLine 추가 시: total_amount 재계산 필요
    할인 적용 시: total_amount 재계산 필요
    반품 처리 시: total_amount 재계산 필요
    쿠폰 변경 시: total_amount 재계산 필요

    → total_amount를 변경해야 하는 모든 지점에 갱신 코드 추가
    → 갱신 누락 버그 발생
    → 동기화 실패 시 읽기 모델과 실제 값 불일치

시도 2: 인덱스 최적화를 위한 컬럼 추가
  "주문 목록을 고객ID + 상태 + 날짜로 자주 검색하니까
   (customer_id, status, created_at) 복합 인덱스 추가"

  문제:
    쓰기: 주문 상태 변경마다 인덱스 업데이트 비용
    쓰기: 인덱스 추가로 INSERT/UPDATE 느려짐
    읽기와 쓰기가 같은 인덱스를 공유 → 읽기 최적화가 쓰기 성능에 영향

시도 3: 캐시 도입
  "자주 조회되는 주문 목록을 Redis에 캐시하자"

  문제:
    주문 상태 변경 시 캐시 무효화 필요
    어느 캐시 키를 무효화해야 하는가? (고객별 목록, 상태별 목록, 날짜별 목록...)
    캐시 무효화 누락 → 오래된 데이터 표시
    쓰기 트랜잭션이 Redis 호출을 포함 → 트랜잭션 복잡도 증가

CQRS 후 읽기 성능 최적화:
  읽기 모델 테이블 (order_summary):
    total_amount: 이벤트 처리 시 계산되어 저장 (쓰기 트랜잭션과 무관)
    인덱스: 읽기 패턴에만 최적화 (쓰기 성능과 무관)
    Redis 캐시: 읽기 모델 위에만 적용 (쓰기와 무관)

  쓰기 모델:
    total_amount 필드 없음 → 동기화 책임 없음
    읽기 인덱스 없음 → 쓰기 성능에 영향 없음
    캐시 무효화 코드 없음 → Projection이 읽기 모델 업데이트 담당
```

### 3. 문제 3 — 역할별 뷰가 단일 모델에서 충돌하는 구조

```
전자상거래의 역할별 뷰 요구사항:

관리자 뷰:
  필드: orderId, customerId, customerInternalGrade (내부 등급),
        allItemDetails, paymentMethod, paymentStatus,
        fraudScore (사기 점수), ipAddress, deviceInfo
  필터: 모든 주문, 사기 의심 주문만, 미결제 주문만
  특징: 고객 내부 정보 포함 → 보안 민감

배송 담당자 뷰:
  필드: orderId, deliveryAddress, recipientName, recipientPhone,
        items (상품명 + 수량 + 크기 + 무게), specialInstruction
  필터: 배송 대기, 배송 중
  특징: 개인정보 최소화, 배송 정보 특화

회계팀 뷰:
  필드: orderId, taxInvoiceNumber, taxAmount, vatAmount,
        paymentMethod, settlementStatus, settlementDate, commissionRate
  필터: 정산 대기, 이번 달 주문
  특징: 회계 정보 특화, 고객 정보 불필요

CS팀 뷰:
  필드: customerId, customerName, customerContact,
        [주문 목록], [반품 내역], [문의 내역], [쿠폰 사용 내역]
  특징: 여러 도메인 통합 (주문 + 반품 + CS + 마케팅)
  저장소 요구: 전문 검색 (Elasticsearch)

단일 모델에서 이 뷰들을 처리하는 코드:

  // 하나의 API가 모든 역할을 분기
  @GetMapping("/orders/{id}")
  public OrderResponseDto getOrder(@PathVariable String id,
                                   @AuthenticationPrincipal User user) {
      Order order = orderService.findById(id);
      // Order는 이 모든 연관관계를 로드해야 함

      if (user.hasRole("ADMIN")) {
          return OrderResponseDto.forAdmin(order); // fraudScore, deviceInfo 포함
      } else if (user.hasRole("DELIVERY")) {
          return OrderResponseDto.forDelivery(order); // 배송 정보만
      } else if (user.hasRole("ACCOUNTING")) {
          return OrderResponseDto.forAccounting(order); // 세금 정보만
      } else if (user.hasRole("CS")) {
          // CS팀: 다른 도메인 데이터도 필요
          Customer customer = customerService.findByOrderId(id);
          List<Return> returns = returnService.findByOrderId(id);
          List<Inquiry> inquiries = csService.findByCustomerId(order.getCustomerId());
          return OrderResponseDto.forCS(order, customer, returns, inquiries);
      }
  }
  // 문제:
  //   관리자 조회 한 번에 배송, 회계, CS 데이터 모두 로드됨
  //   역할이 추가될수록 분기 증가, 연관관계 증가
  //   Order가 모든 역할의 데이터를 알아야 함

CQRS 후 역할별 읽기 모델:
  admin_order_summary → PostgreSQL (관리자 보안 DB)
  delivery_order_summary → PostgreSQL (배송팀 DB)
  accounting_order_summary → PostgreSQL (회계팀 DB)
  cs_customer_view → Elasticsearch (전문 검색)

  각 뷰의 API:
    GET /admin/orders/{id} → admin_order_summary 단순 SELECT
    GET /delivery/orders/{id} → delivery_order_summary 단순 SELECT
    GET /accounting/orders/{id} → accounting_order_summary 단순 SELECT
    GET /cs/customers/{id}/overview → Elasticsearch 검색

  Order Aggregate: 네 개의 뷰 존재를 모름
  → 역할이 추가돼도 Order Aggregate 변경 불필요
  → 새 뷰는 이벤트 스트림을 소비하는 새 Projection으로 추가
```

---

## 💻 실전 코드

### 세 가지 문제 해결 패턴

```java
// ===================================
// 문제 1 해결 — Aggregate 순수성 회복
// ===================================

// ✅ 오염 제거된 Order Aggregate
public class Order {

    private final OrderId id;
    private final CustomerId customerId; // ID만, Customer 객체 아님
    private OrderStatus status;
    private final List<OrderLine> lines; // productId + quantity만

    // 불변식에만 집중
    public void confirm(InventoryChecker inventoryChecker) {
        if (!status.canTransitionTo(CONFIRMED))
            throw new InvalidStatusTransitionException(status, CONFIRMED);

        // 재고 확인 — productId와 quantity만 필요 (Product 상세 정보 불필요)
        inventoryChecker.validateAll(lines.stream()
            .collect(toMap(OrderLine::getProductId, OrderLine::getQuantity)));

        this.status = CONFIRMED;

        // 이벤트 발행 — 이메일, 알림 등은 이벤트를 구독하는 서비스가 처리
        registerEvent(new OrderConfirmed(id, customerId, lines, LocalDateTime.now()));
    }

    // 이메일 로직? → 없음 (EmailService가 OrderConfirmed 이벤트 구독)
    // 재고 차감? → 없음 (InventoryService가 OrderConfirmed 이벤트 구독)
    // 포인트 적립? → 없음 (PointService가 OrderConfirmed 이벤트 구독)
}

// 이메일 서비스 — 이벤트 구독
@EventHandler
public void on(OrderConfirmed event) {
    // 이메일에 필요한 상품 상세 정보는 여기서 조회
    List<ProductDetail> products = productQueryService.getByIds(
        event.getLines().stream().map(OrderLine::getProductId).collect(toList())
    );
    emailService.sendOrderConfirmation(event.getCustomerId(), event.getOrderId(), products);
}

// ===================================
// 문제 2 해결 — 읽기/쓰기 성능 독립 최적화
// ===================================

// ✅ 읽기 모델 — total_amount 사전 계산, 인덱스 최적화
@Entity
@Table(name = "order_summary",
    indexes = {
        @Index(name = "idx_status_created", columnList = "status, created_at DESC"),
        @Index(name = "idx_customer_status", columnList = "customer_id, status")
    })
public class OrderSummaryView {

    @Id
    private String orderId;
    private String customerId;
    private String customerName;   // 비정규화
    private String status;
    private BigDecimal totalAmount; // 사전 계산
    private Integer itemCount;      // 사전 계산
    private LocalDateTime createdAt;

    // 읽기 최적화를 위한 인덱스가 쓰기 성능에 영향 없음
    // total_amount 동기화가 쓰기 트랜잭션에 포함되지 않음
}

// ✅ Projection — 이벤트 기반 읽기 모델 업데이트
@Component
public class OrderSummaryProjection {

    @EventHandler
    @Transactional
    public void on(OrderConfirmed event) {
        BigDecimal total = event.getLines().stream()
            .map(l -> l.getUnitPrice().multiply(BigDecimal.valueOf(l.getQuantity())))
            .reduce(BigDecimal.ZERO, BigDecimal::add);

        String customerName = customerQueryRepository.getNameById(event.getCustomerId());

        OrderSummaryView view = orderSummaryRepository
            .findById(event.getOrderId().value())
            .orElse(new OrderSummaryView());

        view.setOrderId(event.getOrderId().value());
        view.setCustomerId(event.getCustomerId().value());
        view.setCustomerName(customerName);  // 비정규화
        view.setStatus("CONFIRMED");
        view.setTotalAmount(total);           // 사전 계산
        view.setItemCount(event.getLines().size()); // 사전 계산
        view.setCreatedAt(event.getOccurredAt());

        orderSummaryRepository.save(view);
    }
}

// ===================================
// 문제 3 해결 — 역할별 읽기 모델 분리
// ===================================

// ✅ 역할별 별도 읽기 모델 + Controller

// 관리자 API
@RestController
@RequestMapping("/admin/orders")
@PreAuthorize("hasRole('ADMIN')")
public class AdminOrderController {

    @GetMapping
    public Page<AdminOrderSummary> getOrders(Pageable pageable) {
        return adminOrderRepository.findAll(pageable);
        // admin_order_summary 테이블 — fraudScore, ipAddress 포함
    }
}

// 배송 API
@RestController
@RequestMapping("/delivery/orders")
@PreAuthorize("hasRole('DELIVERY')")
public class DeliveryOrderController {

    @GetMapping("/pending")
    public List<DeliveryOrderView> getPendingDeliveries() {
        return deliveryOrderRepository.findByStatus("DELIVERY_PENDING");
        // delivery_order_view 테이블 — 배송 정보만, 개인정보 최소화
    }
}

// CS 통합 API (Elasticsearch)
@RestController
@RequestMapping("/cs/customers")
@PreAuthorize("hasRole('CS')")
public class CSCustomerController {

    @GetMapping("/{customerId}/overview")
    public CSCustomerOverview getCustomerOverview(@PathVariable String customerId) {
        // Elasticsearch에서 고객 전체 이력 통합 조회
        return elasticsearchClient.search(SearchRequest.of(s -> s
            .index("cs_customer_view")
            .query(q -> q.term(t -> t.field("customerId").value(customerId)))
        )).hits().hits().stream()
            .map(hit -> hit.source())
            .findFirst()
            .orElseThrow();
    }
}

// ✅ 각 역할별 읽기 모델을 업데이트하는 Projection들
@Component
public class AdminOrderProjection {
    @EventHandler
    public void on(OrderConfirmed event) { /* admin_order_summary 업데이트 */ }
    @EventHandler
    public void on(FraudDetected event) { /* fraudScore 업데이트 */ }
}

@Component
public class DeliveryOrderProjection {
    @EventHandler
    public void on(OrderConfirmed event) { /* delivery_order_view 업데이트 */ }
    @EventHandler
    public void on(DeliveryStarted event) { /* 배송 시작 업데이트 */ }
}

@Component
public class CSCustomerProjection {
    @EventHandler
    public void on(OrderConfirmed event) { /* Elasticsearch 색인 */ }
    @EventHandler
    public void on(ReturnRequested event) { /* 반품 내역 색인 */ }
    @EventHandler
    public void on(InquiryCreated event) { /* 문의 내역 색인 */ }
}
```

---

## 📊 패턴 비교

```
세 가지 문제 해결 전/후 비교:

문제 1: Aggregate 오염

  Before:
    Order 클래스: 2,000줄
    연관관계: Product, Customer, Delivery, Coupon, Payment, Return
    불변식 찾기: 스크롤 다운 300줄
    단위 테스트: 12개 Mock 필요

  After:
    Order 클래스: 250줄
    연관관계: 없음 (ID 참조만)
    불변식 찾기: confirm() 30줄
    단위 테스트: Mock 1개 (InventoryChecker만)

문제 2: 읽기-쓰기 성능 충돌

  Before:
    주문 목록 조회: 4 테이블 JOIN, 200ms
    주문 생성: total_amount 동기화 포함, 50ms
    total_amount 불일치 버그: 월 3건

  After:
    주문 목록 조회: 단순 SELECT, 5ms
    주문 생성: 이벤트 발행만, 15ms
    total_amount 불일치: 없음 (Projection이 단독 업데이트)

문제 3: 역할별 뷰

  Before:
    Order API 1개가 모든 역할 처리
    역할당 분기 1개 추가 = Order 연관관계 1개 추가
    CS팀 조회: 6 테이블 JOIN, 500ms

  After:
    역할별 독립 API, 독립 테이블
    역할 추가: 새 Projection + 새 Controller (Order 변경 없음)
    CS팀 조회: Elasticsearch, 20ms
```

---

## ⚖️ 트레이드오프

```
해결되는 고통:
  ① Aggregate 순수성: 불변식이 명확해지고 테스트 용이해짐
  ② 성능 독립성: 읽기 최적화가 쓰기에 영향 없음
  ③ 뷰 독립성: 역할별 뷰가 서로 영향 없이 독립 설계

생겨나는 새로운 고통:
  ① Projection 구현 및 관리:
     각 읽기 모델마다 Projection 코드 필요
     Projection 실패 시 읽기 모델 불일치 → 재처리 전략 필요

  ② 최종 일관성 허용:
     이벤트 처리 지연 동안 읽기 모델이 구 데이터 표시
     UX 설계에서 이를 고려해야 함

  ③ 읽기 모델 관리:
     역할별 읽기 모델 테이블/인덱스 관리
     스키마 변경 시 Projection 재구축 필요

  ④ 데이터 중복:
     같은 데이터가 여러 읽기 모델에 비정규화
     저장 용량 증가

고통이 교환되는 방향:
  Before: 개발 시 편하고 운영 시 고통
    (단일 모델 → 처음엔 빠름 → 복잡해질수록 모든 것이 엉킴)
  After: 설계 시 고민이 많고 운영 시 명확함
    (각 모델이 자기 역할만 → 문제 발생 위치가 명확)
```

---

## 📌 핵심 정리

```
CQRS가 해결하는 세 가지 실제 문제:

문제 1: Aggregate 오염
  증상: 조회 요구사항이 늘어날수록 도메인 모델에 연관관계와 필드가 추가됨
  원인: 쓰기 불변식 모델이 읽기 데이터 요구사항을 흡수
  해결: 쓰기 모델 = 불변식에 필요한 최소 데이터만
        읽기 요구사항 → 이벤트 + Projection으로 분리

문제 2: 읽기-쓰기 성능 충돌
  증상: 읽기 성능을 개선하려다 쓰기 로직이 복잡해지고 동기화 버그 발생
  원인: 읽기 최적화(비정규화, 캐시)와 쓰기 일관성이 같은 모델에서 충돌
  해결: 읽기 모델은 읽기 목적으로만 설계, 쓰기는 이벤트 발행으로 읽기 갱신

문제 3: 역할별 뷰 충돌
  증상: 역할이 늘어날수록 단일 API가 복잡해지고 Aggregate 연관관계 증가
  원인: 서로 다른 데이터 형태가 필요한 역할들이 단일 모델에 강요됨
  해결: 역할별 독립 읽기 모델, 역할별 독립 API, 같은 이벤트 스트림 소비

공통 해결 구조:
  쓰기: Aggregate → 이벤트 발행 (읽기 관심사 없음)
  읽기: Projection이 이벤트 소비 → 역할별 읽기 모델 업데이트
```

---

## 🤔 생각해볼 문제

**Q1.** Aggregate 오염을 막기 위해 CQRS를 도입하지 않고, 헥사고날 아키텍처와 포트-어댑터 패턴만으로 해결할 수 있는가?

<details>
<summary>해설 보기</summary>

헥사고날 아키텍처는 Aggregate 오염 문제를 일부 완화할 수 있지만, 근본적으로 해결하지는 못합니다.

헥사고날로 해결되는 것은 의존성 방향입니다. 도메인이 인프라에 의존하지 않도록 인터페이스(포트)를 통해 격리할 수 있습니다. 예를 들어 `InventoryChecker`를 포트로 정의하고 실제 구현은 어댑터로 분리하면 됩니다.

하지만 헥사고날로 해결되지 않는 것이 있습니다. 읽기 요구사항이 Aggregate 필드를 추가하는 문제는 그대로입니다. 관리자 뷰에서 fraudScore가 필요하면 헥사고날이든 아니든 Order Aggregate에 fraudScore가 추가되거나, Order에서 FraudService를 호출해야 합니다. 역할별 다른 뷰 요구사항이 단일 모델을 오염시키는 것도 마찬가지입니다.

CQRS는 아키텍처 레이어 분리(헥사고날)가 아니라 "쓰기와 읽기가 근본적으로 다른 데이터 모델을 필요로 한다"는 개념적 분리입니다. 두 패턴은 상호 보완적으로 사용하는 것이 좋습니다.

</details>

---

**Q2.** CS팀 뷰처럼 여러 도메인의 데이터를 통합해야 하는 경우, Projection이 여러 도메인 이벤트를 구독한다. 이때 Projection이 도메인 간 결합도를 높이지 않는가?

<details>
<summary>해설 보기</summary>

이것은 CQRS + Event Sourcing에서 중요한 트레이드오프입니다.

이벤트를 통한 결합과 직접 호출을 통한 결합의 차이를 먼저 이해해야 합니다. 직접 호출 방식에서 CS 서비스가 OrderService, ReturnService, InquiryService를 직접 의존하면 런타임 결합이 발생합니다. 하나가 다운되면 CS 서비스도 다운됩니다. 반면 이벤트 구독 방식에서 CS Projection이 OrderConfirmed, ReturnRequested, InquiryCreated 이벤트를 구독하면 발행자가 다운돼도 Projection은 영향받지 않습니다(이미 발행된 이벤트를 소비). 각 도메인이 이벤트 계약(Event Contract)만 지키면 됩니다.

하지만 새로운 형태의 결합이 생깁니다. Projection은 이벤트 스키마에 결합됩니다. OrderConfirmed 이벤트의 필드가 변경되면 CS Projection도 변경해야 합니다. 이 스키마 결합을 관리하기 위해 이벤트 스키마 버전 관리와 하위 호환성 유지가 중요합니다.

결론적으로 완전한 결합 제거는 불가능합니다. CQRS는 결합의 종류를 "런타임 직접 의존 결합"에서 "이벤트 스키마 결합"으로 바꾸는 것이며, 이것이 더 나은 트레이드오프인 경우가 많습니다.

</details>

---

**Q3.** 읽기 모델이 여러 개일 때 "진실의 원천(Source of Truth)"은 어디인가? 읽기 모델들이 서로 불일치하면 어떤 모델이 올바른 것인가?

<details>
<summary>해설 보기</summary>

CQRS + Event Sourcing에서 진실의 원천은 명확합니다. 이벤트 스토어(또는 Event Sourcing 없이 CQRS만 사용할 때는 쓰기 모델의 DB)가 진실의 원천입니다.

읽기 모델은 이 진실의 원천에서 파생된 뷰입니다. 두 읽기 모델이 불일치하면, 이벤트 스토어의 이벤트를 기준으로 재구축한 값이 정답입니다.

실무적인 판단 기준으로 다음 세 가지를 활용합니다. 첫째, 법적/재무적 결정(출금 가능 여부, 재고 차감 여부)에는 반드시 쓰기 모델을 사용합니다. 읽기 모델의 지연 가능성이 있기 때문입니다. 둘째, 화면 표시에는 읽기 모델을 사용하되, 화면에 "데이터 기준 시간"을 표시하는 것을 고려합니다. 셋째, 읽기 모델 불일치 감지 시 이벤트 스토어에서 특정 Projection을 재구축하여 동기화합니다.

Event Sourcing 없이 CQRS만 사용하는 경우(수준 2)에는 Write DB가 진실의 원천이고, 읽기 모델 테이블은 Write DB에서 파생된 뷰입니다.

</details>

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[이전: CQRS의 스펙트럼 ⬅️](./03-cqrs-spectrum.md)** | **[다음: CQRS 적용 판단 기준 ➡️](./05-when-to-apply-cqrs.md)**

</div>
