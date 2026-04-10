# 여러 읽기 모델의 공존

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 같은 이벤트 스트림에서 어떻게 다양한 읽기 모델을 생성하는가?
- 읽기 모델별 업데이트 시점을 어떻게 조정하는가?
- 읽기 모델 간 일관성의 트레이드오프는 무엇인가?
- 동일 이벤트를 소비하는 여러 프로젝션의 처리 순서를 어떻게 관리하는가?
- 읽기 모델 수가 늘어날 때 이벤트 스토어 부하는 어떻게 관리하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

전자상거래 주문 하나가 완료됐을 때 여러 읽기 모델이 동시에 업데이트돼야 한다. 주문 목록, 주문 상세, 통계 대시보드, 검색 인덱스, 배송 대기 목록, 매출 집계. 이 읽기 모델들이 서로 다른 저장소, 서로 다른 스키마, 서로 다른 실시간성 요구사항을 가진다. 이 다양성을 하나의 이벤트 스트림으로 어떻게 지원하는지 이해하는 것이 핵심이다.

---

## 😱 흔한 실수 (Before — 읽기 모델 난립)

```
실수 1: 읽기 모델마다 독립 이벤트 발행

  OrderConfirmed 이벤트:
    → Projection A가 직접 주문 서비스 DB 폴링
    → Projection B가 직접 메시지 큐 구독
    → Projection C가 REST API 호출

  // 각 Projection이 다른 방법으로 동일 이벤트 소비
  // → 이벤트 소비 방법이 통일되지 않아 관리 어려움

실수 2: 읽기 모델 간 의존성 생성

  OrderSummaryProjection.on(OrderConfirmed) {
      summary.setStatus("CONFIRMED");
      // 다른 읽기 모델을 직접 읽어 데이터 조합
      DeliveryView delivery = deliveryViewRepo.findByOrderId(orderId); // ❌
      summary.setDeliveryCity(delivery.getCity()); // 읽기 모델 간 의존
  }
  // 배송 읽기 모델이 아직 업데이트되지 않으면 null

실수 3: 실시간성 요구사항 무시

  // 통계 대시보드도 50ms 이내로 업데이트
  // 매 이벤트마다 복잡한 집계 재계산
  // → Projection 처리 시간 증가 → 다른 읽기 모델 지연
```

---

## ✨ 올바른 접근 (After — 독립 프로젝션 + 실시간성 계층화)

```
설계 원칙:

1. 각 읽기 모델은 독립 프로젝션
   → 하나 실패가 다른 모델에 영향 없음

2. 각 프로젝션은 이벤트에서 직접 데이터 도출
   → 다른 읽기 모델 의존 없음

3. 실시간성 요구사항별 계층화
   실시간 (< 1초): 주문 목록, 상태 조회
   준실시간 (< 1분): 검색 인덱스, 재고 현황
   배치 (주기적): 통계, 리포트

4. 이벤트 스토어 부하 분산
   실시간 → 이벤트 스토어 직접 폴링
   준실시간 → Kafka (이벤트 스토어 부하 없음)
   배치 → 별도 배치 잡 (업무 시간 외 실행)
```

---

## 🔬 내부 동작 원리

### 1. 같은 이벤트 → 다양한 읽기 모델

```
OrderConfirmed 이벤트 한 개가 처리되는 방식:

이벤트:
  OrderConfirmed {
      orderId: "ORD-001",
      customerId: "USER-42",
      items: [...],
      totalAmount: 150000,
      confirmedBy: "SELLER-07",
      occurredAt: 2024-01-15T14:30:00Z
  }

→ [Projection A] OrderSummaryProjection
  읽기 모델: order_summary (PostgreSQL)
  업데이트: status='CONFIRMED', confirmed_at 저장
  지연: 50ms

→ [Projection B] OrderDetailProjection
  읽기 모델: order_detail (PostgreSQL, 별도 테이블)
  업데이트: 상세 정보 (상품 목록, 결제 정보 포함)
  지연: 80ms

→ [Projection C] SearchProjection
  읽기 모델: Elasticsearch 인덱스
  업데이트: 검색 키워드, 상태 인덱싱
  지연: 200ms

→ [Projection D] RevenueProjection
  읽기 모델: monthly_revenue (PostgreSQL)
  업데이트: 이번 달 확인된 매출 += 150000
  지연: 500ms (집계 계산)

→ [Projection E] DeliveryQueueProjection
  읽기 모델: delivery_queue (PostgreSQL)
  업데이트: 배송 대기 목록에 추가
  지연: 100ms

→ [Projection F] AdminNotificationProjection
  읽기 모델: 없음 (이메일/슬랙 알림 발송)
  지연: 1000ms (외부 API 호출)

→ [Projection G] AnalyticsProjection
  읽기 모델: ClickHouse (OLAP)
  업데이트: 시계열 데이터 삽입
  지연: 5분 (배치 처리)

모든 프로젝션은 동일 이벤트에서 독립적으로 데이터 도출
→ 다른 읽기 모델 참조 없음
→ 이벤트 페이로드에 필요한 모든 정보 포함
```

### 2. 실시간성 계층화 전략

```
실시간성 요구사항별 처리 방식:

Tier 1 — 실시간 (< 1초):
  대상: 주문 목록, 상태 조회, 잔고 현황
  처리: 이벤트 스토어 폴링 (500ms 간격) or Kafka Consumer
  저장소: PostgreSQL, Redis

  Kafka Consumer Group: "order-realtime"
  처리량 최대화, 복잡한 처리 없음

Tier 2 — 준실시간 (< 1분):
  대상: 검색 인덱스, 재고 현황, 추천 시스템
  처리: Kafka Consumer (별도 Consumer Group)
  저장소: Elasticsearch, Redis

  Kafka Consumer Group: "order-nearrealtime"
  Elasticsearch 벌크 인덱싱으로 최적화

Tier 3 — 배치 (주기적):
  대상: 통계 대시보드, 일별/월별 리포트, BI 분석
  처리: 스케줄 잡 (매 시간, 매일)
  저장소: ClickHouse, BigQuery, PostgreSQL

  @Scheduled(cron = "0 0 * * * *") // 매 시간
  void updateHourlyStats() { ... }

계층 간 Kafka Topic 분리:
  order-events-realtime    → Tier 1 Consumer Groups
  order-events-analytics   → Tier 3 Consumer Groups

  또는 동일 토픽 + 다른 Consumer Group으로 처리:
    group: order-summary-projection   → 실시간 처리
    group: analytics-projection       → 배치 처리 (lag 허용)
```

### 3. 읽기 모델 간 일관성 트레이드오프

```
같은 이벤트를 처리하는 5개 읽기 모델의 일관성:

  T=0ms   OrderConfirmed 이벤트 발행

  T=50ms  OrderSummary 업데이트 완료
          GET /orders → 최신 상태 ✅

  T=200ms OrderSearch 업데이트 완료
          GET /orders/search → 최신 상태 ✅

  T=500ms Revenue 업데이트 완료
          GET /dashboard/revenue → 최신 상태 ✅

  T=5분   Analytics 업데이트 완료
          GET /analytics → 최신 상태 ✅

  T=50ms 시점에 "주문 목록"과 "매출 통계"를 같은 화면에 표시:
    주문 목록: 최신 (ORD-001 표시됨) ✅
    매출 통계: 구버전 (ORD-001 금액 미반영) ⚠️

  → 한 화면의 두 위젯이 다른 시점의 데이터 표시
  → 사용자가 불일치를 느낄 수 있음

해결 전략:
  ① 같은 화면의 데이터는 같은 Tier에서:
    주문 목록 + 매출 요약 → 둘 다 Tier 1 (실시간)
    통계 대시보드 → Tier 3 (배치), 별도 화면

  ② "업데이트 기준 시각" 표시:
    "매출 통계: 1시간 전 기준"
    → 사용자가 지연을 인지하고 수용

  ③ 조회 시 일관성 계층 선택:
    GET /dashboard?consistency=eventual  → 준실시간 데이터
    GET /dashboard?consistency=strong    → 쓰기 모델에서 직접 계산
```

### 4. 이벤트 스토어 부하 관리

```
읽기 모델 10개가 모두 이벤트 스토어를 직접 폴링:

  Polling 부하:
    프로젝션 10개 × 500ms 간격 = 초당 20번 조회
    이벤트 1,000건/s 발생 시 → 각 폴링마다 수백 건 조회
    → 이벤트 스토어 DB 부하 급증

해결 1 — Kafka 중간 레이어:
  이벤트 스토어 → Outbox Publisher → Kafka Topic
  프로젝션들 → Kafka Consumer
  → 이벤트 스토어는 단 1회 발행, 나머지는 Kafka가 처리

  이벤트 스토어 DB:
    SELECT * FROM event_store WHERE global_seq > ? (1회/발행)
  Kafka:
    Consumer Group 10개가 독립적으로 소비 (이벤트 스토어 부하 없음)

해결 2 — 이벤트 팬아웃:
  단일 폴링 프로세스가 이벤트 읽어 모든 프로젝션에 분배

  EventFanout:
    events = eventStore.loadBatch(lastSeq, 1000)
    for event in events:
        projections.parallelStream()
            .forEach(p -> p.handle(event))

  → 이벤트 스토어 조회 1회 → 모든 프로젝션 처리
```

---

## 💻 실전 코드

```java
// ✅ 읽기 모델별 독립 Kafka Consumer Group
@Configuration
public class ProjectionKafkaConfig {

    // Tier 1 — 실시간 (빠른 자동 커밋)
    @Bean
    public ConcurrentKafkaListenerContainerFactory<String, String>
            realtimeProjectionFactory() {
        var factory = new ConcurrentKafkaListenerContainerFactory<String, String>();
        factory.setConsumerFactory(consumerFactory("order-realtime-projection"));
        factory.setConcurrency(3); // 파티션 수만큼
        return factory;
    }

    // Tier 2 — 준실시간 (수동 커밋, 재시도 허용)
    @Bean
    public ConcurrentKafkaListenerContainerFactory<String, String>
            nearRealTimeProjectionFactory() {
        var factory = new ConcurrentKafkaListenerContainerFactory<String, String>();
        factory.setConsumerFactory(consumerFactory("order-search-projection"));
        factory.getContainerProperties().setAckMode(AckMode.MANUAL);
        return factory;
    }
}

// ✅ Tier 1 — 실시간 주문 목록 프로젝션
@Component
public class OrderSummaryProjection {

    @KafkaListener(
        topics = "order-events",
        groupId = "order-summary-projection",
        containerFactory = "realtimeProjectionFactory"
    )
    public void handle(String eventJson) {
        DomainEvent event = deserialize(eventJson);
        switch (event) {
            case OrderConfirmed e -> updateOrderStatus(e.orderId(), "CONFIRMED", e.occurredAt());
            case OrderShipped e   -> updateOrderStatus(e.orderId(), "SHIPPED", e.occurredAt());
            default -> { /* 무관심 이벤트 무시 */ }
        }
        // 단순 업데이트 — 빠른 처리
    }
}

// ✅ Tier 2 — 준실시간 검색 인덱스 프로젝션
@Component
public class OrderSearchProjection {

    private final List<Map<String, Object>> buffer = new ArrayList<>();

    @KafkaListener(
        topics = "order-events",
        groupId = "order-search-projection",
        containerFactory = "nearRealTimeProjectionFactory"
    )
    public void handle(ConsumerRecord<String, String> record, Acknowledgment ack) {
        DomainEvent event = deserialize(record.value());
        if (event instanceof OrderPlaced e) {
            buffer.add(buildSearchDoc(e));
            if (buffer.size() >= 100) {
                flushToElasticsearch(); // 100건씩 벌크 인덱싱
                buffer.clear();
            }
        }
        ack.acknowledge();
    }

    @Scheduled(fixedDelay = 5000) // 5초마다 버퍼 플러시
    public void scheduledFlush() {
        if (!buffer.isEmpty()) {
            flushToElasticsearch();
            buffer.clear();
        }
    }
}

// ✅ Tier 3 — 배치 통계 프로젝션
@Component
public class RevenueStatsProjection {

    @Scheduled(cron = "0 0 * * * *") // 매 시간 정각
    @Transactional
    public void updateHourlyRevenue() {
        Instant oneHourAgo = Instant.now().minus(1, HOURS);

        // 이벤트 스토어에서 지난 1시간 OrderConfirmed 이벤트 조회
        List<StoredEvent> events = eventStore.loadByTypeAndPeriod(
            "OrderConfirmed", oneHourAgo, Instant.now());

        Map<YearMonth, BigDecimal> revenueByMonth = events.stream()
            .map(e -> (OrderConfirmed) deserialize(e))
            .collect(groupingBy(
                e -> YearMonth.from(e.occurredAt().atZone(ZoneId.systemDefault())),
                reducing(BigDecimal.ZERO, OrderConfirmed::totalAmount, BigDecimal::add)
            ));

        revenueByMonth.forEach((month, revenue) ->
            revenueStatsRepository.upsert(month, revenue));

        log.info("시간별 매출 통계 업데이트 완료: {}건", events.size());
    }
}
```

---

## 📊 패턴 비교

```
읽기 모델 실시간성 계층별 비교:

┌──────────────────┬─────────────┬─────────────┬─────────────────┐
│ 계층             │ 지연        │ 처리 방식   │ 저장소          │
├──────────────────┼─────────────┼─────────────┼─────────────────┤
│ Tier 1 실시간   │ < 1초       │ Kafka Push  │ RDB, Redis      │
├──────────────────┼─────────────┼─────────────┼─────────────────┤
│ Tier 2 준실시간 │ < 1분       │ Kafka 배치  │ Elasticsearch   │
├──────────────────┼─────────────┼─────────────┼─────────────────┤
│ Tier 3 배치     │ 1시간~1일  │ 스케줄 잡  │ ClickHouse, BQ  │
└──────────────────┴─────────────┴─────────────┴─────────────────┘

읽기 모델 수에 따른 이벤트 스토어 부하:
  읽기 모델 N개, 이벤트 스토어 직접 폴링:
    부하 = N × 폴링 빈도
  Kafka 중간 레이어:
    이벤트 스토어 부하 = 1 (발행 1회)
    Kafka 부하 = N Consumer 처리
```

---

## ⚖️ 트레이드오프

```
읽기 모델 수가 늘어날 때:
  장점: 각 역할에 완벽히 최적화된 뷰
  단점: 이벤트 처리 비용 선형 증가
        프로젝션 관리 복잡도 증가
        저장 공간 증가

읽기 모델 간 일관성:
  모든 읽기 모델 동기 업데이트 → 강한 일관성, 처리 느림
  비동기 독립 업데이트 → 최종 일관성, 처리 빠름
  → 대부분의 경우 최종 일관성 수용
```

---

## 📌 핵심 정리

```
여러 읽기 모델 공존 핵심:

독립 프로젝션 원칙:
  각 읽기 모델 → 독립 프로젝션 → 독립 Consumer Group
  다른 읽기 모델 의존 없음 (이벤트에서 직접)

실시간성 계층화:
  Tier 1 (< 1초): 즉각 반응 필요한 UI
  Tier 2 (< 1분): 검색, 집계
  Tier 3 (배치): 통계, BI

이벤트 스토어 부하:
  프로젝션 수 증가 → Kafka 중간 레이어로 부하 분산

일관성 관리:
  같은 화면의 데이터는 같은 Tier에서
  지연 허용 시 "기준 시각" 표시
```

---

## 🤔 생각해볼 문제

**Q1.** 읽기 모델 10개가 같은 Kafka 토픽을 소비하고 있을 때, 하나의 읽기 모델 처리가 매우 느려지면 다른 읽기 모델에 영향을 주는가?

<details>
<summary>해설 보기</summary>

각 읽기 모델이 독립 Kafka Consumer Group을 사용한다면 영향이 없습니다. Kafka Consumer Group은 각자 독립적으로 오프셋을 관리합니다. 느린 Consumer Group은 자신의 속도로 처리하고, 빠른 Consumer Group은 독립적으로 처리합니다.

단, 같은 Consumer Group 안에 여러 읽기 모델 처리 로직이 있다면 영향이 있습니다. 하나의 처리가 느려지면 같은 Consumer Group의 처리량 전체가 영향받습니다.

Kafka 토픽 자체에는 영향이 없습니다. Consumer Group이 느리면 그 Consumer Group의 lag이 증가할 뿐, 토픽에서 메시지가 사라지거나 다른 Consumer Group에 영향을 주지 않습니다. Kafka의 Consumer Group 격리가 이를 보장합니다.

</details>

---

**Q2.** 새로운 읽기 모델을 추가할 때 과거 이벤트부터 시작해야 하는가, 현재 이벤트부터 시작해도 되는가?

<details>
<summary>해설 보기</summary>

비즈니스 요구사항에 따라 다릅니다.

현재 이벤트부터 시작해도 되는 경우는 새 읽기 모델이 향후 데이터만 필요할 때입니다. 예를 들어 새로운 A/B 테스트 분석용 읽기 모델이나 실험적인 통계는 오늘부터의 데이터로도 충분합니다.

과거 이벤트부터 시작해야 하는 경우는 새 읽기 모델이 역사적 데이터를 포함해야 할 때입니다. 예를 들어 "전체 주문 목록"이나 "고객별 총 구매액" 같은 읽기 모델은 과거 이벤트 없이는 불완전합니다. 이 경우 과거 이벤트부터 재구축(04-projection-rebuild.md)이 필요합니다.

Event Sourcing의 핵심 강점은 과거 이벤트가 보존되어 있어 언제든 새 읽기 모델을 과거부터 구축할 수 있다는 점입니다. 상태 저장 시스템에서는 이것이 불가능합니다.

</details>

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[이전: 프로젝션 재구축 ⬅️](./04-projection-rebuild.md)** | **[다음: 프로젝션 장애 처리 ➡️](./06-projection-failure-handling.md)**

</div>
