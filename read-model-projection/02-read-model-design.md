# 읽기 모델 설계 원칙

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 읽기 모델을 읽기 요구사항 중심으로 설계한다는 것이 구체적으로 무엇인가?
- 조인 없이 한 번에 모든 데이터를 제공하려면 어떤 비정규화가 필요한가?
- 역할별 다른 읽기 모델이 같은 이벤트 스트림에서 어떻게 나오는가?
- 읽기 모델 스키마를 변경해야 할 때 어떻게 처리하는가?
- 화면별 읽기 모델 분리의 비용과 편익은 어떻게 균형을 맞추는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

읽기 모델을 잘못 설계하면 CQRS의 핵심 이점을 얻지 못한다. 읽기 모델에 JOIN이 필요하다면 비정규화가 충분하지 않은 것이고, 읽기 모델 하나가 너무 많은 역할을 담당하면 어느 요구사항도 제대로 최적화할 수 없다. 읽기 모델 설계의 원칙은 단순하다. "이 화면에 필요한 모든 데이터가 단일 SELECT로 제공되는가?"

---

## 😱 흔한 실수 (Before — 쓰기 모델과 비슷한 구조의 읽기 모델)

```
실수 1: 정규화된 읽기 모델

  order_summary (읽기 모델이지만 여전히 정규화):
    order_id, customer_id, status, created_at

  목록 화면에서 customer_name 필요 → JOIN 필요
  → 읽기 모델이 있지만 여전히 JOIN

실수 2: 하나의 읽기 모델로 모든 역할 처리

  order_view:
    order_id, customer_id, customer_name, customer_tier,  // 관리자용
    customer_internal_score,                               // 관리자 전용
    delivery_address, recipient_phone,                     // 배송용
    tax_invoice_no, vat_amount,                            // 회계용
    complaint_history                                      // CS용
  // 모든 역할이 이 테이블 사용 → 인덱스 최적화 불가

실수 3: 이벤트 순서를 고려하지 않은 업데이트

  OrderShipped 이벤트가 OrderConfirmed 이벤트보다 먼저 도착
  (이벤트 파티션이 다를 경우 순서 보장 없음)
  → 읽기 모델 상태가 "SHIPPED" → 나중에 "CONFIRMED"로 덮어씌워짐
```

---

## ✨ 올바른 접근 (After — 화면 중심 비정규화)

```
설계 원칙:
  "이 화면은 무엇을 보여주는가?"에서 시작
  → 그 화면에 필요한 모든 데이터를 하나의 테이블에
  → 단일 SELECT, 조인 없음

주문 목록 화면 → order_summary:
  order_id, customer_name (비정규화), status,
  total_amount (사전 계산), item_count (사전 계산),
  delivery_city (비정규화), created_at

주문 상세 화면 → order_detail:
  order_id, customer_name, customer_email, customer_phone,
  status, items (JSONB 배열), delivery_full_address,
  payment_method, total_amount, confirmed_at

관리자 뷰 → admin_order_view (별도 DB):
  order_id, customer_internal_score, fraud_score,
  ip_address, device_fingerprint, ... 민감 정보 포함

역할별 분리:
  → 같은 이벤트 스트림에서 각자 다른 읽기 모델 생성
  → 각 읽기 모델의 인덱스를 해당 역할에 최적화
```

---

## 🔬 내부 동작 원리

### 1. 화면 중심 설계 — 역방향 접근

```
잘못된 접근 (DB 스키마 → 화면):
  "어떤 데이터가 있는가" → 화면에 표시

올바른 접근 (화면 → DB 스키마):
  "화면에 무엇이 필요한가" → 읽기 모델 설계

주문 관리 목록 화면에 필요한 정보:
  ┌──────────────────────────────────────────────────────┐
  │ 주문번호 │ 고객명 │ 상태 │ 총금액 │ 상품수 │ 배송지 │
  │ ORD-001  │ 김철수  │ 확인됨 │ 50,000 │ 3개   │ 서울   │
  └──────────────────────────────────────────────────────┘

  → order_summary 테이블:
    (order_id, customer_name, status, total_amount,
     item_count, delivery_city, created_at)

  조인 없이 단일 SELECT:
    SELECT * FROM order_summary
    WHERE status = ?
    ORDER BY created_at DESC
    LIMIT 20

  인덱스:
    (status, created_at): 상태별 최신순 조회
    (customer_name): 고객명 검색
    (delivery_city): 지역별 필터

복잡한 필터 화면 (여러 조건 동시 적용):
  "서울, 확인됨, 5만원 이상, 이번 달"
  → 단일 테이블에 모든 필드가 있으면 WHERE 절로 처리 가능
  → JOIN이 없어 인덱스 활용 효율 높음
```

### 2. 비정규화 전략 — 무엇을 얼마나

```
비정규화 대상:

① 외래키 → 이름/값으로 비정규화:
  customer_id → customer_name, customer_tier
  product_id  → product_name, product_sku, product_category
  address_id  → city, district, postal_code

  왜: JOIN 제거, 인덱스 단순화
  비용: 원본 데이터 변경 시 읽기 모델 업데이트 필요

② 집계 → 사전 계산값으로:
  SUM(item.price * item.qty) → total_amount
  COUNT(items)                → item_count
  MAX(item.price)             → max_item_price

  왜: 집계 쿼리 제거, 조회 즉시
  비용: 원본 변경 시 재계산 필요 (이벤트로 처리)

③ 중첩 데이터 → JSONB:
  주문 상품 목록 (1:N 관계)
  → order_items JSONB 배열로 비정규화
  [{"name":"아이폰","qty":1,"price":1200000}, ...]

  왜: 1:N 관계를 단일 행에 포함 → 조인 없음
  비용: JSONB 내부 쿼리 인덱스 제한

④ 복잡한 계산 → 미리 계산:
  배송 예정일 (주문일 + SLA 기준 계산)
  할인 후 최종 금액
  → 이벤트 처리 시 계산 후 저장

비정규화하지 말아야 할 것:
  자주 바뀌는 데이터 (재고 수량, 실시간 가격)
  → 비정규화 시 동기화 비용 높음
  → 별도 API 호출 또는 실시간 조회 유지
```

### 3. 역할별 읽기 모델 — 같은 이벤트, 다른 뷰

```
같은 OrderConfirmed 이벤트 → 역할별 다른 업데이트:

이벤트:
  OrderConfirmed {
    orderId, customerId, items, totalAmount,
    confirmedBy, occurredAt
  }

프로젝션 A → 일반 사용자 읽기 모델:
  UPDATE order_summary SET
    status = 'CONFIRMED',
    confirmed_at = ?
  WHERE order_id = ?
  -- 내부 등급, 사기 점수 없음

프로젝션 B → 관리자 읽기 모델 (별도 DB):
  UPDATE admin_order_view SET
    status = 'CONFIRMED',
    confirmed_by = ?,       -- 누가 확인했는지
    fraud_check_result = ?, -- 사기 감지 결과
    internal_note = ?       -- 내부 메모
  WHERE order_id = ?

프로젝션 C → 배송 읽기 모델:
  INSERT INTO delivery_queue (
    order_id, delivery_address, recipient_name,
    items_weight, special_instruction, priority
  ) VALUES (...)
  -- 결제 정보, 고객 등급 없음

프로젝션 D → 매출 통계:
  UPDATE monthly_revenue SET
    confirmed_revenue = confirmed_revenue + ?
  WHERE year_month = DATE_TRUNC('month', ?)

프로젝션 E → Elasticsearch 검색:
  elasticsearchClient.index("orders",
    { orderId, customerName, productNames, status, totalAmount })

결과:
  OrderConfirmed 이벤트 1개 → 5개의 다른 읽기 모델 업데이트
  각 읽기 모델은 자신의 역할에 최적화
  Order Aggregate는 이 5개의 존재를 모름
```

### 4. 이벤트 순서 문제와 버전 기반 업데이트

```
이벤트 순서가 보장되지 않는 경우:

  Kafka 파티션이 다르면 이벤트 순서 보장 없음:
    Partition 0: OrderPlaced   (seq=100)
    Partition 1: OrderConfirmed (seq=101)
    Partition 0: OrderShipped  (seq=102)

  Consumer가 처리:
    OrderConfirmed (seq=101) 먼저 처리 → status='CONFIRMED'
    OrderPlaced    (seq=100) 나중에 처리 → status='PLACED' (덮어씌움!)

버전 기반 업데이트로 해결:
  읽기 모델에 version 또는 last_event_seq 저장
  → 더 최신 이벤트로만 업데이트

  UPDATE order_summary
  SET status = 'PLACED', last_event_seq = 100
  WHERE order_id = ? AND last_event_seq < 100  ← 더 오래된 이벤트면 무시

  결과:
    OrderConfirmed (seq=101) → last_event_seq=101, status='CONFIRMED'
    OrderPlaced    (seq=100) → 100 < 101 아니므로 업데이트 안 함 (무시)
```

---

## 💻 실전 코드

```java
// ✅ 화면 중심 읽기 모델 — 완전한 스키마

// 주문 목록 읽기 모델
@Entity
@Table(name = "order_summary",
    indexes = {
        @Index(name = "idx_status_created", columnList = "status, created_at DESC"),
        @Index(name = "idx_customer", columnList = "customer_id"),
        @Index(name = "idx_delivery_city", columnList = "delivery_city")
    })
public class OrderSummaryView {
    @Id private String orderId;
    private String customerId;
    private String customerName;     // 비정규화
    private String customerTier;     // 비정규화
    private String status;
    private BigDecimal totalAmount;  // 사전 계산
    private Integer itemCount;       // 사전 계산
    private String deliveryCity;     // 비정규화
    private LocalDateTime createdAt;
    private LocalDateTime updatedAt;
    private Long lastEventSeq;       // 순서 보장용
}

// ✅ 버전 기반 멱등 업데이트
@Transactional
public void onOrderPlaced(OrderPlaced event, long eventSeq) {
    jdbcTemplate.update("""
        INSERT INTO order_summary
            (order_id, customer_id, customer_name, customer_tier,
             status, total_amount, item_count, delivery_city,
             created_at, last_event_seq)
        VALUES (?, ?, ?, ?, 'PLACED', ?, ?, ?, ?, ?)
        ON CONFLICT (order_id) DO UPDATE SET
            status = EXCLUDED.status,
            last_event_seq = EXCLUDED.last_event_seq
        WHERE order_summary.last_event_seq < EXCLUDED.last_event_seq
        """,
        event.orderId(), event.customerId(),
        customerCache.getName(event.customerId()),
        customerCache.getTier(event.customerId()),
        event.totalAmount(), event.itemCount(),
        extractCity(event.deliveryAddress()),
        event.occurredAt(), eventSeq
    );
}
```

---

## 📊 패턴 비교

```
단일 읽기 모델 vs 역할별 분리:

단일 읽기 모델:
  장점: 관리 단순, 테이블 수 적음
  단점: 모든 역할의 인덱스 포함 → 인덱스 비효율
        민감 데이터 분리 불가
        스케일 아웃 불가 (모든 역할이 같은 DB)

역할별 분리:
  장점: 각 역할에 최적화된 인덱스
        민감 데이터 별도 DB에 격리
        읽기 부하를 역할별로 분산
  단점: 프로젝션 수 증가, 테이블 수 증가
        데이터 중복 증가
```

---

## ⚖️ 트레이드오프

```
비정규화 수준 결정:
  적게 비정규화 → JOIN 필요, 조회 느림
  많이 비정규화 → 업데이트 비용, 저장 용량 증가

동기화 이벤트:
  customer_name 변경 → CustomerNameChanged 이벤트
  → OrderSummary의 customer_name 일괄 업데이트
  → 과거 주문에도 현재 이름 반영 여부는 비즈니스 정책
```

---

## 📌 핵심 정리

```
읽기 모델 설계 원칙:

화면 중심 설계:
  "이 화면에 무엇이 필요한가?" → 읽기 모델 스키마
  단일 SELECT, 조인 없음이 목표

비정규화:
  외래키 → 이름/값으로 (customer_id → customer_name)
  집계 → 사전 계산 (total_amount, item_count)
  1:N → JSONB 배열

역할별 분리:
  같은 이벤트 → 역할별 다른 읽기 모델
  각 역할에 최적화된 인덱스, 저장소

이벤트 순서 보장:
  last_event_seq 필드로 오래된 이벤트 무시
```

---

## 🤔 생각해볼 문제

**Q1.** 고객 이름이 바뀌었을 때, 과거 주문 읽기 모델의 customer_name도 업데이트해야 하는가?

<details>
<summary>해설 보기</summary>

비즈니스 정책에 따라 결정해야 할 사항입니다.

과거 주문에 현재 이름을 반영해야 하는 경우는 고객 지원 화면처럼 "이 사람이 주문한 것"이 중요할 때입니다. `CustomerNameChanged` 이벤트 발생 시 `UPDATE order_summary SET customer_name = ? WHERE customer_id = ?`로 일괄 업데이트합니다.

과거 주문에 당시 이름을 보존해야 하는 경우는 주문 당시의 계약이나 법적 문서 성격일 때입니다. 예를 들어 이름 변경 전에 체결된 계약서에는 변경 전 이름이 기재되어야 합니다. 이 경우 order_summary의 customer_name을 업데이트하지 않습니다.

Event Sourcing을 사용한다면, order_summary를 재구축할 때 어느 시점의 고객 이름을 사용할지 규칙을 명확히 해야 합니다. 주문 당시 이름을 사용한다면 `CustomerNameChanged` 이벤트의 순서를 기준으로 판단합니다.

</details>

---

**Q2.** 읽기 모델 스키마를 변경해야 할 때(예: 새 필드 추가) 기존 데이터를 어떻게 처리하는가?

<details>
<summary>해설 보기</summary>

두 가지 접근이 있습니다.

첫째, 마이그레이션 스크립트로 기존 데이터 업데이트합니다. `ALTER TABLE order_summary ADD COLUMN customer_phone VARCHAR(20)`을 실행하고, 기존 행은 NULL 또는 기본값으로 채웁니다. 이 방법은 빠르지만 과거 데이터에 새 필드의 정확한 값이 없을 수 있습니다.

둘째, Projection 재구축으로 새 필드를 포함한 전체 읽기 모델을 다시 만듭니다. Event Sourcing을 사용하면 과거 이벤트에서 new field를 계산할 수 있습니다. 재구축 중 무중단 운영을 위해 Blue/Green Projection 전략을 사용합니다(04-projection-rebuild.md 참고).

일반적으로 새 필드가 과거 이벤트에서 계산 가능하다면 Projection 재구축이 더 정확합니다. 과거 이벤트에 해당 데이터가 없다면 마이그레이션 스크립트로 기본값을 채우는 것이 현실적입니다.

</details>

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[이전: 프로젝션 완전 분해 ⬅️](./01-projection-deep-dive.md)** | **[다음: Eventual Consistency 처리 ➡️](./03-eventual-consistency.md)**

</div>
