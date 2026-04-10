# Command 결과 반환 패턴

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Fire-and-Forget, Acknowledgement, Result 패턴의 차이는 무엇인가?
- 비동기 Command 처리 결과를 클라이언트에 전달하는 방법에는 어떤 것이 있는가?
- Polling, WebSocket, SSE 각각이 적합한 시나리오는 무엇인가?
- Eventual Consistency 지연을 UX에서 어떻게 처리해야 하는가?
- Command 결과 반환이 CQRS 원칙과 어떻게 조화를 이루는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

CQRS에서 Command는 상태를 변경하고 Query는 데이터를 반환한다. 이 분리가 UX 설계에 영향을 준다. "출금 버튼을 누르면 즉시 새 잔고를 보여줘야 한다"는 요구사항이 있을 때, 읽기 모델이 아직 업데이트되지 않은 상태에서 어떻게 처리할 것인가? Command 결과 반환 패턴을 이해하면 이 문제를 UX와 아키텍처 모두 만족하는 방식으로 해결할 수 있다.

---

## 😱 흔한 실수 (Before — 잘못된 Command 결과 처리)

```
실수 1: Command Handler가 읽기 모델을 즉시 조회

  @CommandHandler
  @Transactional
  public OrderDetailView handle(ConfirmOrderCommand cmd) {
      Order order = orderRepository.findById(cmd.getOrderId());
      order.confirm();
      orderRepository.save(order);
      // ↓ 이벤트 발행 → Kafka → Projection 처리 중 (아직 안 됨!)
      return orderQueryService.getOrderDetail(cmd.getOrderId());
      // ❌ 읽기 모델이 아직 "PENDING" 상태 → 구버전 반환
  }

실수 2: 클라이언트가 처리 결과를 알 방법이 없음

  @PostMapping("/orders/{id}/confirm")
  @ResponseStatus(HttpStatus.NO_CONTENT)
  public void confirm(@PathVariable String id) {
      commandBus.send(new ConfirmOrderCommand(id));
      // 처리 성공 여부를 클라이언트가 어떻게 알지?
      // 비동기라면 처리 완료 후 어떻게 알림?
  }
  // 클라이언트: 무한 폴링? 페이지 새로고침?

실수 3: 동기 Command를 억지로 비동기 처리

  @PostMapping("/orders/{id}/confirm")
  public ResponseEntity<OrderDetailView> confirm(@PathVariable String id) {
      commandBus.send(new ConfirmOrderCommand(id));
      // Projection 완료를 기다림 (최대 5초)
      for (int i = 0; i < 50; i++) {
          OrderDetailView view = orderQueryService.getDetail(id);
          if ("CONFIRMED".equals(view.getStatus())) return ok(view);
          Thread.sleep(100);
      }
      throw new TimeoutException("처리 시간 초과");
      // ❌ 폴링 로직이 서버에 → 스레드 낭비, 타임아웃 불안정
  }
```

---

## ✨ 올바른 접근 (After — 패턴별 적합한 사용)

```
패턴 선택 기준:

Pattern 1 — Fire-and-Forget:
  사용: 처리 결과가 즉시 필요 없는 경우
  응답: 202 Accepted
  예: 이메일 발송 요청, 비동기 리포트 생성
  UX: "요청이 접수되었습니다. 처리가 완료되면 이메일로 알려드립니다."

Pattern 2 — Acknowledgement (처리 ID 반환):
  사용: 나중에 처리 결과를 확인해야 하는 경우
  응답: 202 + { "requestId": "REQ-001" }
  예: 대용량 파일 처리, 장시간 배치 작업
  UX: "요청 ID REQ-001로 진행 상황을 확인하세요."

Pattern 3 — 낙관적 UI 업데이트:
  사용: Command 성공이 거의 확실하고 즉시 반영이 필요한 경우
  응답: 202 + 예상 결과 데이터
  예: 좋아요 버튼, 장바구니 추가
  UX: 버튼 클릭 즉시 UI 업데이트 → 실패 시 롤백

Pattern 4 — 실시간 업데이트 (WebSocket / SSE):
  사용: 처리 완료를 실시간으로 클라이언트에 push해야 하는 경우
  응답: 202 → WebSocket/SSE로 완료 이벤트 push
  예: 주문 상태 실시간 추적, 결제 처리 결과
```

---

## 🔬 내부 동작 원리

### 1. 세 가지 기본 패턴

```
Pattern 1 — Fire-and-Forget:

  Client                Server
    │                     │
    │── POST /reports/generate ──►│
    │◄── 202 Accepted ────────────│ (즉시 응답)
    │                     │
    │                     │── [비동기] 리포트 생성 중
    │                     │
    │── GET /reports/{id} ────────►│ (완료 후 클라이언트가 체크)
    │◄── 200 OK + 리포트 ──────────│

  적합한 경우:
    처리 시간이 길어 동기 대기 불가
    실패해도 재시도 가능
    클라이언트가 결과를 즉시 필요로 하지 않음

Pattern 2 — Acknowledgement:

  Client                Server
    │                     │
    │── POST /orders/confirm ─────►│
    │◄── 202 + {requestId} ────────│ (처리 ID 즉시 반환)
    │                     │
    │                     │── [비동기] 주문 확인 처리 중
    │                     │
    │── GET /requests/{requestId} ──►│ (처리 상태 폴링)
    │◄── 200 {status: "PROCESSING"} │
    │                     │
    │── GET /requests/{requestId} ──►│
    │◄── 200 {status: "COMPLETED"} │

Pattern 3 — 낙관적 UI 업데이트:

  Client                    Server
    │                           │
    │ [버튼 클릭]                 │
    │ UI 즉시 업데이트 (임시)      │
    │── POST /orders/{id}/confirm ──►│
    │                           │── Command 처리 중
    │◄── 202 Accepted ───────────│
    │                           │
    │ (처리 성공 가정, UI 유지)    │
    │                           │
    │ [만약 실패 응답 오면]        │
    │ UI 롤백 (원래 상태로)        │
    │◄── 409/422 ────────────────│
```

### 2. 실시간 결과 전달 — Polling vs WebSocket vs SSE

```
Polling (클라이언트가 주기적으로 조회):

  장점: 구현 단순, 서버 상태 없음, 네트워크 끊김에 강함
  단점: 불필요한 요청, 지연 발생 (폴링 주기만큼)

  구현:
    클라이언트: setInterval(() => checkStatus(requestId), 2000)
    서버: GET /requests/{id}/status → { status: "PROCESSING" | "COMPLETED" | "FAILED" }

  적합: 처리 시간이 수십 초~수 분, 실시간성보다 안정성 우선

WebSocket (양방향 실시간 통신):

  장점: 진짜 실시간, 서버 push 가능, 양방향
  단점: 연결 유지 비용, 스케일 아웃 시 세션 관리 복잡

  구현:
    클라이언트: ws.on('order.confirmed', handler)
    서버: webSocketSession.sendMessage(new OrderConfirmedEvent(orderId))

  적합: 채팅, 실시간 협업, 빠른 처리 결과 알림

SSE — Server-Sent Events (단방향 서버 push):

  장점: WebSocket보다 단순, HTTP/2 지원, 자동 재연결
  단점: 단방향 (서버 → 클라이언트만), IE 미지원

  구현:
    클라이언트: EventSource('/orders/{id}/events')
    서버: SseEmitter로 이벤트 push

  적합: 주문 상태 추적, 처리 진행률, 알림 스트림

비교:
  처리 시간 5초 이하 + 실시간 필요 → WebSocket or SSE
  처리 시간 5초~1분 → SSE or Polling
  처리 시간 1분 이상 → Polling + 이메일 알림
```

### 3. Eventual Consistency와 낙관적 UI 업데이트

```
문제 상황:
  주문 확인 Command 전송
  → 이벤트 스토어 저장 (즉시)
  → Kafka 발행 (즉시)
  → Projection 처리 (50ms 지연)
  → 읽기 모델 업데이트 (50ms 후)

  클라이언트가 바로 GET /orders/{id} 요청
  → 읽기 모델이 아직 "PENDING" → 구버전 표시

낙관적 UI 업데이트 전략:

  1. Command 전송 시 클라이언트가 UI를 예상 상태로 즉시 업데이트
  2. Command 응답 대기
  3. 성공(202) → UI 유지
  4. 실패(409/422) → UI를 원래 상태로 롤백 + 오류 메시지

  // React 예시
  const confirmOrder = async (orderId) => {
      // 1. 낙관적 업데이트 — 즉시 UI 변경
      setOrderStatus('CONFIRMED');  // 임시 상태

      try {
          await commandApi.confirmOrder(orderId);
          // 2. 성공 → UI 유지 (읽기 모델 업데이트 기다리지 않음)
      } catch (error) {
          // 3. 실패 → UI 롤백
          setOrderStatus('PENDING');  // 원래 상태로 복원
          showError('주문 확인에 실패했습니다.');
      }
  };

Command 응답에 예상 상태 포함:
  @PostMapping("/orders/{id}/confirm")
  public CommandResult confirm(@PathVariable String id) {
      commandBus.send(new ConfirmOrderCommand(id));
      // Command 자체에서 알 수 있는 최소한의 예상 결과
      return new CommandResult(id, "CONFIRMED", Instant.now());
      // ← 읽기 모델 조회 없이, Command 내용에서 도출
  }
```

### 4. Axon Subscription Query — 읽기 모델 실시간 구독

```
Axon Framework의 Subscription Query:
  Command 전송 후 읽기 모델이 업데이트될 때 즉시 받는 방법

  @PostMapping("/orders/{id}/confirm")
  public DeferredResult<OrderDetailView> confirm(@PathVariable String id) {
      DeferredResult<OrderDetailView> deferredResult =
          new DeferredResult<>(5000L); // 5초 타임아웃

      // 읽기 모델 업데이트 구독 시작
      SubscriptionQueryResult<OrderDetailView, OrderDetailView> result =
          queryGateway.subscriptionQuery(
              new GetOrderDetailQuery(id),
              ResponseTypes.instanceOf(OrderDetailView.class),
              ResponseTypes.instanceOf(OrderDetailView.class)
          );

      // Command 전송
      commandGateway.send(new ConfirmOrderCommand(id));

      // 읽기 모델이 CONFIRMED로 업데이트될 때까지 대기
      result.updates()
          .filter(view -> "CONFIRMED".equals(view.getStatus()))
          .take(1)
          .subscribe(view -> {
              deferredResult.setResult(view);
              result.close();
          });

      return deferredResult;
      // 읽기 모델이 업데이트되면 즉시 응답 (최대 5초 대기)
  }
```

---

## 💻 실전 코드

```java
// ==============================
// 패턴별 완전 구현
// ==============================

// ✅ Pattern 1 — Fire-and-Forget
@PostMapping("/reports/generate")
@ResponseStatus(HttpStatus.ACCEPTED)
public void generateReport(@RequestBody ReportRequest request) {
    commandBus.send(new GenerateReportCommand(
        UUID.randomUUID().toString(),
        request.getType(),
        request.getDateRange()
    ));
    // 202 Accepted — 처리 완료 알림 없음
    // 완료 시 이메일 발송 (Projection에서 처리)
}

// ✅ Pattern 2 — Acknowledgement + Polling
@PostMapping("/batch/process")
public ResponseEntity<CommandAcknowledgement> processBatch(
        @RequestBody BatchRequest request) {
    String requestId = UUID.randomUUID().toString();
    commandBus.send(new ProcessBatchCommand(requestId, request.getData()));

    URI statusUri = URI.create("/batch/requests/" + requestId + "/status");
    return ResponseEntity.accepted()
        .location(statusUri)
        .body(new CommandAcknowledgement(requestId, statusUri.toString()));
}

@GetMapping("/batch/requests/{requestId}/status")
public BatchRequestStatus getStatus(@PathVariable String requestId) {
    return batchRequestRepository.findById(requestId)
        .map(r -> new BatchRequestStatus(r.getId(), r.getStatus(), r.getResult()))
        .orElseThrow(() -> new RequestNotFoundException(requestId));
}

// ✅ Pattern 3 — SSE로 실시간 주문 상태 push
@GetMapping(value = "/orders/{id}/status-stream",
            produces = MediaType.TEXT_EVENT_STREAM_VALUE)
public SseEmitter subscribeToOrderStatus(@PathVariable String id) {
    SseEmitter emitter = new SseEmitter(60_000L); // 60초 타임아웃

    // Projection에서 이벤트 발생 시 SSE push
    orderStatusEmitters.register(id, emitter);

    emitter.onCompletion(() -> orderStatusEmitters.remove(id, emitter));
    emitter.onTimeout(() -> orderStatusEmitters.remove(id, emitter));

    return emitter;
}

// Projection에서 SSE 이벤트 전송
@Component
public class OrderStatusProjection {

    @EventHandler
    public void on(OrderConfirmed event) {
        orderSummaryRepository.updateStatus(event.getOrderId(), "CONFIRMED");

        // SSE 구독 중인 클라이언트에 push
        orderStatusEmitters.sendToSubscribers(
            event.getOrderId(),
            SseEmitter.event()
                .name("order.confirmed")
                .data(new OrderStatusUpdate(event.getOrderId(), "CONFIRMED"))
        );
    }
}

// ✅ Pattern 4 — 낙관적 UI + Command 응답에 예상 결과 포함
@PostMapping("/orders/{id}/confirm")
public ResponseEntity<CommandResult> confirmOrder(
        @PathVariable String id,
        @AuthenticationPrincipal UserDetails user) {

    commandBus.send(new ConfirmOrderCommand(id, user.getUsername()));

    // Command 자체에서 알 수 있는 예상 결과만 반환
    // 읽기 모델 추가 조회 없음
    CommandResult result = new CommandResult(
        id,
        "CONFIRMED",        // Command 처리 결과로 예측 가능한 상태
        user.getUsername(),
        Instant.now()
    );
    return ResponseEntity.accepted().body(result);
}

// ✅ CommandResult DTO
public record CommandResult(
    String resourceId,
    String expectedStatus,    // 읽기 모델이 도달할 것으로 예상되는 상태
    String processedBy,
    Instant processedAt
) {}
```

---

## 📊 패턴 비교

```
결과 반환 패턴 선택 가이드:

┌──────────────────────┬────────────────┬──────────────────┬──────────────────┐
│ 패턴                 │ 처리 시간      │ 실시간 필요      │ 구현 복잡도      │
├──────────────────────┼────────────────┼──────────────────┼──────────────────┤
│ Fire-and-Forget      │ 모든 경우      │ 불필요           │ 낮음             │
├──────────────────────┼────────────────┼──────────────────┼──────────────────┤
│ Acknowledgement +    │ 수 초 이상     │ 낮음             │ 낮음~중간        │
│ Polling              │                │                  │                  │
├──────────────────────┼────────────────┼──────────────────┼──────────────────┤
│ 낙관적 UI 업데이트   │ 빠름(< 1초)   │ 높음             │ 중간 (클라이언트)│
├──────────────────────┼────────────────┼──────────────────┼──────────────────┤
│ SSE                  │ 수 초 이내     │ 높음             │ 중간             │
├──────────────────────┼────────────────┼──────────────────┼──────────────────┤
│ WebSocket            │ 수 초 이내     │ 매우 높음        │ 높음             │
└──────────────────────┴────────────────┴──────────────────┴──────────────────┘

도메인별 권장:
  주문 확인: 낙관적 UI + SSE (빠른 처리 + 실시간 알림)
  대용량 파일 처리: Acknowledgement + Polling (장시간 처리)
  실시간 협업: WebSocket (양방향 실시간)
  간단한 CRUD 성공 알림: Fire-and-Forget + 이메일
```

---

## ⚖️ 트레이드오프

```
낙관적 UI 업데이트:
  장점: 즉각적인 사용자 경험, 추가 서버 요청 없음
  단점: 실패 시 롤백 구현 필요, 실패 상황 사용자가 혼란스러울 수 있음

SSE vs WebSocket:
  SSE: 단순, HTTP/2 자동 다중화, 자동 재연결
       단방향 → 클라이언트 → 서버 메시지 불가
  WebSocket: 양방향, 낮은 지연
              연결 유지 비용, 스케일 아웃 시 세션 관리 복잡

Polling:
  장점: 가장 단순, 네트워크 끊김에 강함, 서버 상태 없음
  단점: 불필요한 요청 (처리 중에도 계속 요청), 지연 발생
```

---

## 📌 핵심 정리

```
Command 결과 반환 패턴:

4가지 패턴:
  Fire-and-Forget:       202, 결과 알림 없음
  Acknowledgement+Poll:  202 + requestId, 클라이언트 폴링
  낙관적 UI:             202 + 예상 상태, 클라이언트 즉시 업데이트
  SSE/WebSocket:         202 후 서버 push

CQRS 원칙과의 조화:
  Command 응답: void or 최소 데이터(ID, 예상 상태)
  읽기 모델 조회: 별도 GET 요청으로
  Eventual Consistency: 낙관적 UI or SSE로 보완

핵심 결정:
  "처리 완료를 즉시 알아야 하는가?" → SSE/WebSocket
  "처리 시간이 길어도 되는가?" → Polling
  "UX 매끄럽게 + 빠른 처리" → 낙관적 UI
```

---

## 🤔 생각해볼 문제

**Q1.** 낙관적 UI 업데이트를 적용했을 때 Command가 실패하면 UI를 롤백해야 한다. 여러 동시 Command가 있을 때 롤백 순서를 어떻게 관리하는가?

<details>
<summary>해설 보기</summary>

낙관적 UI에서 동시 Command 롤백은 프론트엔드 상태 관리의 핵심 과제입니다.

각 Command에 고유한 클라이언트 ID를 부여합니다. Command 전송 전 현재 상태를 스냅샷으로 저장합니다. Command 실패 시 해당 Command의 스냅샷으로 롤백합니다.

React Query나 SWR 같은 라이브러리는 낙관적 업데이트 + 롤백 패턴을 내장 지원합니다. Redux Toolkit의 `createAsyncThunk`를 활용하면 pending(낙관적 업데이트) → fulfilled(유지) → rejected(롤백) 상태 전이를 명확히 관리할 수 있습니다.

여러 Command가 동시에 발생하면 최종 성공한 Command의 상태로 수렴해야 합니다. 이 경우 서버에서 처리 완료 후 SSE나 WebSocket으로 최종 상태를 push하고, 클라이언트가 서버 상태를 정답으로 업데이트하는 방식이 안전합니다.

</details>

---

**Q2.** SSE 연결 중 서버가 재시작되면 클라이언트가 놓친 이벤트를 어떻게 복구하는가?

<details>
<summary>해설 보기</summary>

SSE는 자동 재연결 기능이 있습니다. 클라이언트가 재연결 시 `Last-Event-ID` 헤더로 마지막으로 수신한 이벤트 ID를 서버에 전달합니다.

서버는 이 ID 이후의 이벤트를 재전송합니다. 이를 위해 서버는 최근 이벤트를 일정 시간 보관해야 합니다(이벤트 버퍼 또는 이벤트 스토어 활용).

구현 패턴으로는 이벤트 스토어(Kafka 또는 DB)에서 `Last-Event-ID` 이후의 이벤트를 재조회해 전송하는 방식이 가장 신뢰성 있습니다. Redis pub/sub를 백킹 스토어로 사용하면 여러 서버 인스턴스 간 이벤트 공유도 가능합니다.

Kafka를 사용하는 경우 Kafka Consumer의 오프셋 관리를 통해 재연결 시 특정 오프셋부터 재생할 수 있어 가장 강력한 내구성을 제공합니다.

</details>

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[이전: Optimistic Locking in CQRS ⬅️](./05-optimistic-locking.md)** | **[다음: Chapter 3 — Event Sourcing의 핵심 아이디어 ➡️](../event-sourcing/01-event-sourcing-core.md)**

</div>
