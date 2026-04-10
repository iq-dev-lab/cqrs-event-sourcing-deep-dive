# Eventual Consistency 처리

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Command 후 읽기 모델 업데이트가 지연되는 원인과 규모는 얼마나 되는가?
- 클라이언트에서 지연을 처리하는 UX 전략에는 어떤 것들이 있는가?
- 낙관적 UI 업데이트의 실패 케이스를 어떻게 안전하게 롤백하는가?
- 읽기 모델 업데이트 지연을 모니터링하는 지표는 어떻게 설계하는가?
- 강한 일관성이 필요한 경우와 최종 일관성이 허용되는 경우를 어떻게 구분하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

Eventual Consistency는 CQRS와 Event Sourcing을 도입할 때 가장 빈번하게 UX 문제로 이어진다. "방금 한 작업이 화면에 반영되지 않는다"는 사용자 경험은 시스템에 대한 신뢰를 떨어뜨린다. 이 문제를 기술 스택의 한계로 받아들이지 않고, UX 전략과 모니터링으로 해결 가능한 설계 문제로 접근하는 시각이 필요하다.

---

## 😱 흔한 실수 (Before — Eventual Consistency를 방치할 때)

```
실수 1: 지연을 무시하고 바로 조회

  // 클라이언트 JavaScript
  await api.post('/orders/ORD-001/confirm');
  // 즉시 목록 재조회 — 아직 읽기 모델 미반영
  const orders = await api.get('/orders?status=CONFIRMED');
  // 방금 확인한 주문이 목록에 없음

실수 2: 서버에서 완료를 기다리는 폴링

  @PostMapping("/orders/{id}/confirm")
  public OrderDetailView confirm(@PathVariable String id) {
      commandBus.send(new ConfirmOrderCommand(id));

      // Projection 완료까지 폴링 (최대 5초)
      for (int i = 0; i < 50; i++) {
          Thread.sleep(100);
          OrderDetailView view = queryService.getDetail(id);
          if ("CONFIRMED".equals(view.getStatus())) return view;
      }
      throw new TimeoutException("Projection 업데이트 타임아웃");
  }
  // 스레드 낭비, 타임아웃 불안정, CQRS 원칙 위반

실수 3: 최종 일관성 지연 모니터링 없음

  // 프로덕션에서 Projection이 30분 지연되고 있음
  // 아무도 모름 — 사용자만 "왜 반영이 안 되냐" 문의
```

---

## ✨ 올바른 접근 (After — UX 전략 + 모니터링)

```
전략 1 — 낙관적 UI 업데이트:
  Command 전송과 동시에 클라이언트 UI 즉시 업데이트
  성공 시 유지, 실패 시 롤백

전략 2 — 명확한 처리 중 상태:
  "처리 중..." 인디케이터 표시
  읽기 모델 업데이트 감지 후 UI 갱신

전략 3 — Command 응답 활용:
  Command 응답에 예상 결과 포함
  클라이언트가 이 값으로 즉시 UI 업데이트

전략 4 — SSE/WebSocket으로 push:
  Projection 완료 시 서버가 클라이언트에 push
  클라이언트가 최신 상태로 자동 갱신

모니터링:
  이벤트 발행 시각 vs 읽기 모델 업데이트 시각 추적
  지연 시간 히스토그램 (p50, p95, p99)
  지연 임계값(예: 5초) 초과 시 알람
```

---

## 🔬 내부 동작 원리

### 1. Eventual Consistency 지연의 발생 원인

```
지연이 발생하는 각 단계:

  T=0ms    Command 수신
  T=5ms    Aggregate 로드 (이벤트 스토어 조회)
  T=8ms    불변식 검증 + 이벤트 생성
  T=12ms   이벤트 스토어 저장 (INSERT)
  T=12ms   Kafka 발행 (또는 Outbox 저장)

  [네트워크 + Kafka 지연]
  T=20ms   Kafka Consumer가 이벤트 수신 (8ms 지연)

  [Projection 처리]
  T=25ms   역직렬화 + 라우팅
  T=30ms   읽기 모델 DB 업데이트

  T=30ms   읽기 모델 최신화 완료

정상 지연: 20~50ms (평균적인 경우)
  → 사용자가 Command 완료 즉시 GET 요청하면 구버전 가능

지연이 길어지는 경우:
  Kafka Consumer 지연: 1~5초 (부하 증가 시)
  읽기 모델 DB 부하: 수백 ms
  Projection 오류 + 재시도: 수 분
  Kafka Consumer 다운: 재시작까지 수 분 ~

실제 p95 지연 범위:
  단순한 이벤트 → Projection: 50~200ms (허용 가능)
  복잡한 Projection (여러 테이블 업데이트): 200~500ms
  부하 증가 시: 1~5초
  장애 상황: 수 분 이상
```

### 2. 낙관적 UI 업데이트 전략

```
낙관적 UI 업데이트 흐름:

  사용자 액션 → [UI 즉시 업데이트] → Command 전송 → [결과 확인]
                                                        ↓
                                          성공: UI 유지 / 실패: UI 롤백

구현 패턴 (React):

  const confirmOrder = async (orderId) => {
      // 1. 현재 상태 스냅샷 (롤백용)
      const previousState = orders.find(o => o.id === orderId);

      // 2. 낙관적 업데이트 — 즉시 UI 변경
      setOrders(orders.map(o =>
          o.id === orderId
              ? { ...o, status: 'CONFIRMED', confirmedAt: new Date() }
              : o
      ));

      try {
          // 3. Command 전송
          await api.post(`/orders/${orderId}/confirm`);
          // 4. 성공 — UI 유지 (이미 업데이트됨)

      } catch (error) {
          // 5. 실패 — UI 롤백
          setOrders(orders.map(o =>
              o.id === orderId ? previousState : o
          ));
          toast.error('주문 확인에 실패했습니다. 다시 시도해주세요.');
      }
  };

React Query 낙관적 업데이트:

  const { mutate: confirmOrder } = useMutation({
      mutationFn: (orderId) => api.post(`/orders/${orderId}/confirm`),
      onMutate: async (orderId) => {
          await queryClient.cancelQueries(['orders']);
          const previousOrders = queryClient.getQueryData(['orders']);

          // 낙관적 업데이트
          queryClient.setQueryData(['orders'], (old) =>
              old.map(o => o.id === orderId
                  ? { ...o, status: 'CONFIRMED' } : o));

          return { previousOrders }; // 컨텍스트로 전달
      },
      onError: (err, orderId, context) => {
          // 실패 시 롤백
          queryClient.setQueryData(['orders'], context.previousOrders);
      },
      onSettled: () => {
          // 완료 후 서버 상태로 동기화 (성공/실패 모두)
          queryClient.invalidateQueries(['orders']);
      }
  });
```

### 3. SSE 기반 실시간 업데이트

```
Projection 완료 후 클라이언트 push:

  // 서버 — SSE 엔드포인트
  @GetMapping(value = "/orders/{id}/updates",
              produces = MediaType.TEXT_EVENT_STREAM_VALUE)
  public SseEmitter subscribeToOrderUpdates(@PathVariable String id) {
      SseEmitter emitter = new SseEmitter(30_000L); // 30초 타임아웃
      sseEmitters.add(id, emitter);
      emitter.onCompletion(() -> sseEmitters.remove(id, emitter));
      return emitter;
  }

  // Projection에서 SSE push
  @EventHandler
  public void on(OrderConfirmed event) {
      // 읽기 모델 업데이트
      orderSummaryRepo.updateStatus(event.orderId(), "CONFIRMED");

      // SSE 구독 클라이언트에 push
      sseEmitters.sendToAll(event.orderId(),
          SseEmitter.event()
              .name("order.confirmed")
              .data(Map.of(
                  "orderId", event.orderId(),
                  "status", "CONFIRMED",
                  "confirmedAt", event.occurredAt()
              ))
      );
  }

  // 클라이언트 JavaScript
  const eventSource = new EventSource(`/orders/${orderId}/updates`);

  eventSource.addEventListener('order.confirmed', (e) => {
      const update = JSON.parse(e.data);
      // 서버가 push한 데이터로 UI 업데이트
      setOrderStatus(update.status);
      setConfirmedAt(update.confirmedAt);
      eventSource.close(); // 업데이트 받았으면 구독 종료
  });
```

### 4. Eventual Consistency 지연 모니터링

```
핵심 지표:

지표 1: Projection 지연 (Event Lag)
  이벤트 발행 시각(occurred_at) vs 읽기 모델 업데이트 시각
  → Projection 지연 = 업데이트 시각 - occurred_at

  읽기 모델 테이블에 last_event_at 컬럼 추가:
    UPDATE order_summary
    SET status = 'CONFIRMED', last_event_at = event.occurred_at
    WHERE order_id = ?

  모니터링 쿼리:
    SELECT AVG(NOW() - last_event_at) as avg_lag,
           MAX(NOW() - last_event_at) as max_lag,
           PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY NOW() - last_event_at) as p95_lag
    FROM order_summary
    WHERE last_event_at > NOW() - INTERVAL '5 minutes'

지표 2: Consumer Group 지연 (Kafka)
  Kafka Consumer Group의 lag (미처리 메시지 수)
  → lag > 임계값 → 알람

  Prometheus + Grafana:
    kafka_consumergroup_lag{group="order-summary-projection"}
    → 임계값: 1000개 이상 → PagerDuty 알람

지표 3: Projection 처리 실패율
  성공 처리 수 / 전체 이벤트 수
  → 실패율 > 1% → 알람

Grafana 대시보드:
  Panel 1: Projection 평균 지연 (p50, p95, p99)
  Panel 2: Kafka Consumer Lag (프로젝션별)
  Panel 3: Projection 처리 실패율
  Panel 4: DLQ 적재 속도
```

---

## 💻 실전 코드

```java
// ✅ 지연 모니터링 — Micrometer 연동
@Component
public class ProjectionMetrics {

    private final MeterRegistry meterRegistry;
    private final Timer projectionLagTimer;
    private final Counter failureCounter;

    public ProjectionMetrics(MeterRegistry meterRegistry) {
        this.meterRegistry = meterRegistry;
        this.projectionLagTimer = Timer.builder("projection.lag")
            .description("이벤트 발행 → 읽기 모델 반영 지연")
            .publishPercentiles(0.5, 0.95, 0.99)
            .register(meterRegistry);
        this.failureCounter = Counter.builder("projection.failures")
            .description("Projection 처리 실패 수")
            .register(meterRegistry);
    }

    public void recordLag(Instant eventOccurredAt) {
        long lagMs = Duration.between(eventOccurredAt, Instant.now()).toMillis();
        projectionLagTimer.record(lagMs, TimeUnit.MILLISECONDS);

        // 임계값 초과 시 로그 경고
        if (lagMs > 5000) {
            log.warn("Projection 지연 임계값 초과: lag={}ms", lagMs);
        }
    }

    public void recordFailure(String projectionName, String eventType) {
        failureCounter.increment();
        meterRegistry.gauge("projection.dlq.size",
            Tags.of("projection", projectionName),
            dlqService.size(projectionName));
    }
}

// ✅ 강한 일관성이 필요한 케이스 — 쓰기 모델에서 직접 확인
@GetMapping("/accounts/{id}/balance")
public BalanceResponse getBalance(@PathVariable String id) {
    // 잔고 확인은 읽기 모델(최종 일관성)이 아닌
    // 쓰기 모델에서 직접 (강한 일관성)
    Account account = accountRepository.findById(new AccountId(id))
        .orElseThrow(() -> new AccountNotFoundException(id));

    return new BalanceResponse(
        id,
        account.getBalance().amount(),
        "STRONG_CONSISTENT" // 항상 최신
    );
    // 단점: 이벤트 리플레이 비용 (스냅샷으로 완화)
}
```

---

## 📊 패턴 비교

```
Eventual Consistency 처리 전략 비교:

┌──────────────────┬──────────────┬──────────────┬──────────────┐
│ 전략             │ UX 품질      │ 구현 복잡도  │ 서버 부하    │
├──────────────────┼──────────────┼──────────────┼──────────────┤
│ 낙관적 UI 업데이트│ 우수         │ 중간         │ 낮음         │
├──────────────────┼──────────────┼──────────────┼──────────────┤
│ SSE push         │ 우수         │ 중간         │ 연결 유지    │
├──────────────────┼──────────────┼──────────────┼──────────────┤
│ 클라이언트 폴링  │ 보통         │ 낮음         │ 반복 요청    │
├──────────────────┼──────────────┼──────────────┼──────────────┤
│ 지연 수용 + 안내 │ 낮음(솔직)  │ 매우 낮음    │ 없음         │
├──────────────────┼──────────────┼──────────────┼──────────────┤
│ 서버 폴링 대기   │ 보통         │ 높음         │ 스레드 낭비  │
└──────────────────┴──────────────┴──────────────┴──────────────┘

강한 일관성 vs 최종 일관성 선택:
  강한 일관성 필요: 잔고 확인 후 거래, 재고 확인 후 주문
  최종 일관성 허용: 목록 조회, 통계 화면, 알림 뱃지
```

---

## ⚖️ 트레이드오프

```
낙관적 UI의 롤백 복잡도:
  단순 상태 변경: 롤백 쉬움 (이전 값 복원)
  복잡한 UI 상태: 롤백 범위 결정 어려움
  여러 동시 작업: 롤백 순서 충돌 가능

SSE 연결 비용:
  연결 유지 = 서버 소켓 비용
  수만 명 동시 접속 시 SSE 스케일 아웃 필요 (Redis pub/sub 활용)
```

---

## 📌 핵심 정리

```
Eventual Consistency 처리 핵심:

지연 발생 단계:
  이벤트 저장 → Kafka → Projection → 읽기 모델 업데이트
  정상: 20~50ms / 부하 시: 수 초

UX 전략:
  낙관적 UI: 즉시 반영, 실패 시 롤백
  SSE: 서버 push, 실시간 업데이트
  Command 응답에 예상 상태 포함

강한 일관성 필요 시:
  읽기 모델이 아닌 쓰기 모델에서 직접 조회

모니터링:
  Projection 지연 (p50, p95, p99)
  Consumer Group lag
  처리 실패율
  → 임계값 알람 설정
```

---

## 🤔 생각해볼 문제

**Q1.** "이체 후 즉시 잔고가 업데이트되어야 한다"는 요구사항에서 CQRS의 Eventual Consistency를 어떻게 수용하는가?

<details>
<summary>해설 보기</summary>

요구사항을 더 세분화하면 해결 방법이 보입니다.

"이체가 완료됐음을 즉시 알아야 한다"는 요구사항이라면, Command 응답에 새 잔고를 포함하고 낙관적 UI 업데이트로 해결합니다. 읽기 모델 업데이트를 기다리지 않고 Command 처리 결과를 즉시 표시합니다.

"이체 후 정확한 잔고를 즉시 조회해야 한다(다른 탭/앱에서도)"는 요구사항이라면, 이것은 진짜 강한 일관성 요구입니다. 이 경우 잔고 조회를 읽기 모델이 아닌 쓰기 모델(Account Aggregate 리플레이)에서 제공합니다. 스냅샷 패턴으로 리플레이 비용을 줄일 수 있습니다.

또는 잔고 조회를 읽기 모델에서 하되, 이체 Command 완료 시 읽기 모델을 동기적으로 업데이트하는 "Simple CQRS" 접근도 있습니다. 단 이 경우 읽기 모델 업데이트 실패가 이체 자체를 실패로 만드는 결합이 생깁니다.

</details>

---

**Q2.** Projection 지연이 갑자기 10분으로 늘어났다는 알람을 받았다. 어떤 순서로 진단하는가?

<details>
<summary>해설 보기</summary>

진단 순서가 있습니다.

먼저 Kafka Consumer Group lag 확인합니다. 특정 Consumer Group의 lag이 급격히 증가했다면 해당 Projection의 처리가 느려진 것입니다. Kafka lag이 정상이면 Projection 처리 자체의 문제가 아닙니다.

다음으로 Projection 처리 실패율을 확인합니다. 실패율이 높다면 특정 이벤트 처리에서 오류가 발생하고 재시도 중인 것입니다. 로그에서 오류 원인을 확인합니다.

읽기 모델 DB 부하를 확인합니다. INSERT/UPDATE가 느린 경우 DB CPU, 락, 인덱스 문제일 수 있습니다. `EXPLAIN ANALYZE`로 슬로우 쿼리를 확인합니다.

이벤트 발생량 급증 여부를 확인합니다. 대량 배치 작업이나 트래픽 폭증으로 처리할 이벤트가 급증했다면 Projection을 수평 확장합니다.

마지막으로 Outbox Publisher 지연을 확인합니다. 이벤트 스토어에는 이벤트가 있지만 Kafka에 발행이 지연되고 있는 경우입니다.

</details>

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[이전: 읽기 모델 설계 원칙 ⬅️](./02-read-model-design.md)** | **[다음: 프로젝션 재구축 ➡️](./04-projection-rebuild.md)**

</div>
