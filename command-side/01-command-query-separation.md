# Command와 Query의 명확한 분리

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- CQS(Command-Query Separation) 원칙과 CQRS는 무엇이 다른가?
- Command가 "응답을 최소화"해야 하는 이유는 무엇인가?
- Query가 상태를 변경하면 안 되는 이유는 무엇인가?
- 분리가 만드는 설계 명확성이란 구체적으로 무엇을 의미하는가?
- 분리를 위반했을 때 어떤 문제가 실제로 발생하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

"이 메서드는 데이터를 변경하는가, 아니면 데이터를 읽는가?" 이 질문에 즉시 답할 수 없는 코드는 이해하기 어렵고, 테스트하기 어려우며, 안전하게 호출하기 어렵다. CQS는 이 질문에 코드 구조 자체가 답하도록 강제하는 원칙이다.

CQRS는 CQS를 객체 메서드 수준에서 아키텍처 수준으로 끌어올린다. 단순히 읽기와 쓰기 메서드를 분리하는 것을 넘어, 읽기와 쓰기가 완전히 다른 모델, 다른 저장소, 다른 스택을 가질 수 있다는 설계 결정이다.

---

## 😱 흔한 실수 (Before — CQS를 위반한 코드)

```
실수 1: 상태를 변경하면서 데이터를 반환하는 메서드

  // 주문을 확인하면서 확인된 주문 DTO를 반환
  public OrderDto confirmOrder(String orderId) {
      Order order = orderRepository.findById(orderId);
      order.confirm();                         // ← 상태 변경
      orderRepository.save(order);
      return OrderDto.from(order);             // ← 데이터 반환
  }

  문제:
    이 메서드를 호출하면 항상 부작용이 발생
    테스트에서 검증 목적으로 호출하면 실제 주문이 확인됨
    두 번 호출하면? 두 번 확인 → 예외 or 중복 처리
    "이 메서드 안전하게 재시도해도 되나요?" → 알 수 없음

실수 2: 쿼리가 조회하면서 부수효과를 발생시킴

  public List<Notification> getUnreadNotifications(String userId) {
      List<Notification> notifications = notificationRepository.findUnread(userId);
      notifications.forEach(n -> n.markAsRead()); // ← 상태 변경!
      notificationRepository.saveAll(notifications);
      return notifications;
  }

  문제:
    조회 API인 줄 알고 GET /notifications 를 여러 번 호출
    → 매 호출마다 읽지 않은 알림이 읽음 처리됨
    → 로깅, 모니터링, 재시도 모두 부작용 발생
    클라이언트: "조회만 했는데 왜 알림이 다 읽혔죠?"

실수 3: Command와 Query 경계가 없는 Service

  @Service
  public class OrderService {
      // 이 클래스 안에서 쓰기와 읽기가 뒤섞임
      public void createOrder(...) { ... }         // 쓰기
      public OrderDto getOrder(...) { ... }        // 읽기
      public OrderDto updateAndGetOrder(...) { ... } // ?
      public List<OrderDto> getOrders(...) { ... } // 읽기
      public void cancelOrder(...) { ... }         // 쓰기
  }
  // 어느 메서드가 부작용을 일으키는지 시그니처만 보고 알 수 없음
```

---

## ✨ 올바른 접근 (After — Command와 Query를 명확히 분리)

```
CQS 원칙 적용:
  Command: 상태를 변경한다. 의미 있는 데이터를 반환하지 않는다.
  Query:   상태를 변경하지 않는다. 데이터를 반환한다.

  confirmOrder(orderId): void          ← Command (부작용 있음, 반환 없음)
  getOrder(orderId): OrderDetailView   ← Query (부작용 없음, 데이터 반환)

CQRS 아키텍처 적용:
  Command 처리 경로:
    POST /orders/{id}/confirm
    → ConfirmOrderCommand(orderId)
    → OrderCommandHandler.handle(command)
    → Order.confirm()
    → 이벤트 발행
    → 응답: 202 Accepted (또는 최소한의 ID만)

  Query 처리 경로:
    GET /orders/{id}
    → GetOrderDetailQuery(orderId)
    → OrderQueryHandler.handle(query)
    → 읽기 모델 조회
    → 응답: 200 OK + OrderDetailView

분리가 보장하는 것:
  Command 경로: 항상 부작용 발생 (상태 변경)
  Query 경로:   절대 부작용 없음 (안전한 재시도)
  → 메서드 이름이나 주석 없이 시그니처만으로 의도 명확
```

---

## 🔬 내부 동작 원리

### 1. CQS 원칙 — Bertrand Meyer의 원래 정의

```
CQS (Command-Query Separation) — 1988년 Bertrand Meyer 정의:

  모든 메서드는 다음 둘 중 하나여야 한다:
    Command: 시스템 상태를 변경하는 명령. 반환값 없음(void).
    Query:   시스템 상태를 반환하는 질의. 부수효과 없음.

  핵심: "Asking a question should not change the answer"
  질문하는 행위가 답을 바꿔서는 안 된다.

객체 수준 CQS:
  class Account {
      void deposit(Money amount)   // Command — 잔고 변경, 반환 없음
      void withdraw(Money amount)  // Command — 잔고 변경, 반환 없음
      Money getBalance()           // Query  — 잔고 반환, 상태 불변
      boolean canWithdraw(Money a) // Query  — 가능 여부 반환, 상태 불변
  }

CQS의 예외 (허용되는 경우):
  pop(), dequeue() 같은 연산
  → 값을 꺼내는 동시에 상태 변경
  → 원자성이 필요한 경우 CQS 위반을 허용
  → 하지만 이것도 CQRS에서는 분리 권장

CQS와 멱등성(Idempotency):
  Query: 항상 멱등 (몇 번 호출해도 같은 결과)
  Command: 반드시 멱등은 아님 (한 번 호출과 두 번 호출이 다를 수 있음)
    deposit(1000): 두 번 호출 → 2000 입금됨 (멱등 아님)
    confirm():     두 번 호출 → 두 번째는 예외 (멱등 아님)
```

### 2. CQS → CQRS 확장 — 무엇이 달라지는가

```
CQS는 메서드 수준 원칙:
  같은 클래스, 같은 저장소, 같은 트랜잭션 컨텍스트 안에서
  메서드를 Command와 Query로 구분

CQRS는 아키텍처 수준 패턴:
  Command와 Query가 완전히 다른 스택을 가질 수 있음

  CQS 적용 후:
  ┌─────────────────────────────────┐
  │         OrderService            │
  │  void confirmOrder()   Command  │
  │  OrderDto getOrder()   Query    │
  │  (같은 DB, 같은 Entity)         │
  └─────────────────────────────────┘

  CQRS 적용 후:
  ┌──────────────────────┐    ┌─────────────────────────┐
  │  Command Side        │    │  Query Side             │
  │  ConfirmOrderCommand │    │  GetOrderDetailQuery    │
  │  → OrderAggregate    │    │  → OrderDetailView      │
  │  → Event Store       │    │  → 읽기 DB              │
  │  → Kafka             │    │  (Redis / ES / RDB)     │
  └──────────────────────┘    └─────────────────────────┘
            │                           ▲
            └── 이벤트 기반 동기화 ──────┘

차이:
  CQS: 메서드 분리, 저장소 공유
  CQRS: 모델 분리, 저장소 분리, 스택 분리
  CQRS는 CQS를 포함하지만, CQS가 CQRS는 아님
```

### 3. Command가 응답을 최소화해야 하는 이유

```
Command 후 데이터 반환의 문제점:

  시나리오: 주문 확인 후 확인된 주문 상세 반환

  Command Handler가 읽기 모델을 조회하는 경우:
    void handle(ConfirmOrderCommand cmd) {
        Order order = orderRepository.findById(cmd.getOrderId());
        order.confirm();
        orderRepository.save(order);  // 이벤트 → Kafka → Projection
        // 이 시점에 읽기 모델이 아직 업데이트 안 됨!
        return orderQueryService.getDetail(cmd.getOrderId()); // 구버전 반환
    }

  Event Sourcing 환경에서의 근본적 문제:
    Command 완료 → 이벤트 저장 → Kafka 발행
    읽기 모델 업데이트: 수십 ms 후
    Command Handler에서 즉시 읽기 모델 조회 → 구버전 데이터

  올바른 Command 응답:
    Option 1: void (완전한 Fire-and-Forget)
    Option 2: 생성된 리소스 ID만 반환
      return order.getId(); // 클라이언트가 ID로 나중에 조회
    Option 3: Command 자체에 포함된 데이터만 반환
      return new CommandResult(cmd.getOrderId(), "ACCEPTED");
      // 읽기 모델을 추가 조회하지 않음

  Command 응답에 절대 포함하지 말아야 할 것:
    읽기 모델에서 조회한 비정규화 데이터
    다른 Aggregate의 현재 상태
    집계 결과 (total count, sum 등)
```

### 4. Query가 상태를 변경하면 안 되는 이유

```
Query 부수효과가 만드는 문제들:

문제 1: 안전한 재시도 불가
  GET /notifications → 읽지 않은 알림 반환 + 읽음 처리
  네트워크 오류 → 클라이언트 재시도
  → 두 번째 요청에서 이미 읽음 처리됨
  → 클라이언트가 재시도해야 안전한지 알 수 없음

문제 2: 캐싱 불가
  GET /notifications에 읽기 캐시를 적용하면
  캐시 히트 시: 부수효과(읽음 처리) 미발생
  캐시 미스 시: 부수효과 발생
  → 동일한 API 호출이 캐시 여부에 따라 다른 부작용 발생

문제 3: 모니터링/디버깅 혼란
  API 호출 빈도 로그 → 부수효과 발생 빈도
  디버깅 목적으로 쿼리 반복 실행 → 실제 데이터 변경

문제 4: 부하 테스트 불가
  Query에 부수효과가 있으면 부하 테스트가 실제 데이터 변경
  "읽기 성능 테스트"가 "쓰기 성능 테스트"가 됨

올바른 분리:
  Query: GET /notifications → 읽지 않은 알림 목록만 반환 (상태 불변)
  Command: POST /notifications/mark-as-read → 읽음 처리 (상태 변경, 응답 최소화)

  클라이언트 흐름:
    1. GET /notifications → 목록 표시
    2. 사용자가 알림 확인
    3. POST /notifications/mark-as-read → 읽음 처리
    → Query와 Command가 명확히 분리
```

### 5. 분리가 만드는 설계 명확성

```
코드 수준에서의 명확성:

분리 전:
  @RestController
  public class OrderController {
      @PostMapping("/orders/{id}/confirm")
      public OrderDto confirmOrder(@PathVariable String id) {
          return orderService.confirmOrder(id); // 반환값으로 의도 불명확
      }

      @GetMapping("/orders/{id}")
      public OrderDto getOrder(@PathVariable String id) {
          return orderService.getOrder(id); // 이것도 부작용이 있나?
      }
  }

분리 후:
  @RestController
  public class OrderCommandController {
      @PostMapping("/orders/{id}/confirm")
      @ResponseStatus(HttpStatus.ACCEPTED)
      public void confirmOrder(@PathVariable String id) {
          // void: 상태 변경 발생, 데이터 반환 없음 → Command임이 명확
          commandBus.send(new ConfirmOrderCommand(id));
      }
  }

  @RestController
  public class OrderQueryController {
      @GetMapping("/orders/{id}")
      public OrderDetailView getOrder(@PathVariable String id) {
          // 반환값 있음, 상태 불변 → Query임이 명확
          return queryBus.query(new GetOrderDetailQuery(id));
      }
  }

테스트에서의 명확성:
  Command 테스트: 부작용 발생 여부 검증
    commandBus.send(new ConfirmOrderCommand(orderId));
    verify(orderRepository).save(argThat(o -> o.getStatus() == CONFIRMED));

  Query 테스트: 반환값 검증, 부작용 없음 보장
    OrderDetailView result = queryBus.query(new GetOrderDetailQuery(orderId));
    assertThat(result.getStatus()).isEqualTo("CONFIRMED");
    verifyNoMoreInteractions(orderRepository); // 저장 없음 보장
```

---

## 💻 실전 코드

```java
// ==============================
// CQS + CQRS 분리 구현
// ==============================

// Command 마커 인터페이스
public interface Command {}

// Query 마커 인터페이스 (반환 타입 제네릭)
public interface Query<R> {}

// ✅ Command 정의 — 의도를 명확히 표현
public record ConfirmOrderCommand(
    String orderId,
    String confirmedBy  // 누가 확인했는가 (감사 목적)
) implements Command {}

public record CancelOrderCommand(
    String orderId,
    String reason       // 취소 사유 (불변식 아닌 메타데이터)
) implements Command {}

// ✅ Query 정의 — 반환 타입 명시
public record GetOrderDetailQuery(String orderId)
    implements Query<OrderDetailView> {}

public record GetOrderListQuery(
    String customerId,
    String status,
    int page,
    int size
) implements Query<Page<OrderSummaryView>> {}

// ✅ Command Bus — Command를 Handler로 라우팅
@Component
public class CommandBus {

    private final Map<Class<? extends Command>, CommandHandler<?>> handlers;

    public <C extends Command> void send(C command) {
        @SuppressWarnings("unchecked")
        CommandHandler<C> handler = (CommandHandler<C>)
            handlers.get(command.getClass());

        if (handler == null)
            throw new NoHandlerFoundException(command.getClass());

        handler.handle(command);
        // void: Command는 의미 있는 데이터를 반환하지 않음
    }
}

// ✅ Query Bus — Query를 Handler로 라우팅
@Component
public class QueryBus {

    private final Map<Class<? extends Query<?>>, QueryHandler<?, ?>> handlers;

    public <Q extends Query<R>, R> R query(Q query) {
        @SuppressWarnings("unchecked")
        QueryHandler<Q, R> handler = (QueryHandler<Q, R>)
            handlers.get(query.getClass());

        if (handler == null)
            throw new NoHandlerFoundException(query.getClass());

        return handler.handle(query);
        // R: Query는 데이터를 반환
    }
}

// ✅ Command Handler — 상태 변경, 반환 없음
@Component
public class OrderCommandHandler implements CommandHandler<ConfirmOrderCommand> {

    @Override
    @Transactional
    public void handle(ConfirmOrderCommand command) {
        Order order = orderRepository.findById(command.getOrderId())
            .orElseThrow(() -> new OrderNotFoundException(command.getOrderId()));

        order.confirm();  // 불변식 검증 + 상태 변경 + 이벤트 등록
        orderRepository.save(order);

        // void — 확인된 주문 DTO를 반환하지 않음
        // 클라이언트는 별도 GET 요청으로 최신 상태 조회
    }
}

// ✅ Query Handler — 상태 불변, 데이터 반환
@Component
public class OrderQueryHandler implements QueryHandler<GetOrderDetailQuery, OrderDetailView> {

    @Override
    @Transactional(readOnly = true)  // 읽기 전용 트랜잭션 명시
    public OrderDetailView handle(GetOrderDetailQuery query) {
        return orderDetailRepository.findById(query.getOrderId())
            .orElseThrow(() -> new OrderNotFoundException(query.getOrderId()));
        // 상태 변경 없음 — readOnly = true가 실수로 변경 시 예외 발생
    }
}

// ✅ REST Controller — Command와 Query 명확히 분리
@RestController
@RequestMapping("/orders")
public class OrderController {

    // Command endpoint: POST, 응답 코드 202 Accepted (비동기 처리 암시)
    @PostMapping("/{id}/confirm")
    @ResponseStatus(HttpStatus.ACCEPTED)
    public void confirmOrder(
            @PathVariable String id,
            @AuthenticationPrincipal UserDetails user) {
        commandBus.send(new ConfirmOrderCommand(id, user.getUsername()));
        // void: 클라이언트는 GET으로 최신 상태를 별도 조회
    }

    // Query endpoint: GET, 상태 불변 보장
    @GetMapping("/{id}")
    public ResponseEntity<OrderDetailView> getOrder(@PathVariable String id) {
        OrderDetailView view = queryBus.query(new GetOrderDetailQuery(id));
        return ResponseEntity.ok()
            .cacheControl(CacheControl.maxAge(1, TimeUnit.SECONDS)) // Query는 캐싱 가능
            .body(view);
    }
}
```

---

## 📊 패턴 비교

```
CQS vs CQRS 비교:

┌────────────────────┬────────────────────────┬──────────────────────────────┐
│ 항목               │ CQS                    │ CQRS                         │
├────────────────────┼────────────────────────┼──────────────────────────────┤
│ 적용 수준          │ 메서드/클래스           │ 아키텍처 (시스템 전체)        │
├────────────────────┼────────────────────────┼──────────────────────────────┤
│ 저장소             │ 공유                   │ 분리 가능                    │
├────────────────────┼────────────────────────┼──────────────────────────────┤
│ 모델               │ 공유 (같은 Entity)     │ 분리 (쓰기 모델 / 읽기 모델) │
├────────────────────┼────────────────────────┼──────────────────────────────┤
│ 일관성             │ 강한 일관성             │ 최종 일관성 허용             │
├────────────────────┼────────────────────────┼──────────────────────────────┤
│ Command 반환       │ void 권장              │ void 또는 ID만               │
├────────────────────┼────────────────────────┼──────────────────────────────┤
│ Query 부수효과     │ 없음                   │ 없음 (더 엄격히 강제)        │
└────────────────────┴────────────────────────┴──────────────────────────────┘

Command vs Query HTTP 설계:

  Command endpoints:
    POST   /orders                  → 주문 생성
    POST   /orders/{id}/confirm     → 주문 확인
    DELETE /orders/{id}             → 주문 취소
    PATCH  /orders/{id}/status      → 상태 변경
    응답 코드: 202 Accepted (비동기) or 204 No Content (동기)
    응답 바디: 없음 or 리소스 ID만

  Query endpoints:
    GET    /orders                  → 주문 목록
    GET    /orders/{id}             → 주문 상세
    GET    /orders/{id}/history     → 주문 이력
    응답 코드: 200 OK
    응답 바디: 읽기 모델 DTO
    캐시 헤더: 적용 가능 (부수효과 없으므로)
```

---

## ⚖️ 트레이드오프

```
Command 응답 최소화의 트레이드오프:

장점:
  Command 처리 경로가 읽기 모델에 의존하지 않음
  비동기 Command 처리 가능 (응답 즉시, 처리는 나중에)
  Command와 Query를 독립적으로 스케일 아웃 가능

단점:
  클라이언트가 추가 GET 요청 필요 (라운드트립 증가)
  UI에서 Command 결과를 즉시 반영하기 어려움
    → 해결: 낙관적 UI 업데이트 (Command 전송과 동시에 UI 임시 업데이트)
    → 해결: Command 응답에 처리 결과에 필요한 최소 데이터 포함

분리를 강제하지 않아도 되는 경우:
  도메인이 단순하고 팀이 CQS를 이해하고 있다면
  같은 Service 클래스에서 Command/Query를 분리된 메서드로 관리해도 됨
  중요한 것은 분리 패턴이 아니라 분리의 원칙을 이해하는 것
```

---

## 📌 핵심 정리

```
Command와 Query 분리의 핵심:

CQS 원칙:
  Command: 상태 변경, 반환 없음(void)
  Query: 상태 불변, 데이터 반환
  "질문이 답을 바꿔서는 안 된다"

CQRS 아키텍처:
  CQS를 메서드에서 아키텍처로 확장
  Command 경로: 완전히 다른 모델, 저장소, 스택 가능
  Query 경로: 읽기 최적화 모델, 저장소, 스택 가능

분리가 보장하는 것:
  Query: 안전한 재시도, 캐싱, 부하 테스트 가능
  Command: 명확한 부작용 경계, 멱등성 설계 강제
  코드: 시그니처만으로 의도 파악 가능

HTTP REST에서의 적용:
  Command → POST/PUT/PATCH/DELETE + 202/204
  Query   → GET + 200 + 캐시 가능
```

---

## 🤔 생각해볼 문제

**Q1.** `INCR counter` 같은 Redis 명령어는 값을 증가시키면서 증가된 값을 반환한다. 이것은 CQS 위반인가?

<details>
<summary>해설 보기</summary>

엄밀히 말하면 CQS 위반입니다. `INCR`은 상태를 변경(카운터 증가)하면서 동시에 변경된 값을 반환합니다.

하지만 이것은 허용되는 예외 케이스입니다. 원자성이 필요한 경우입니다. `INCR`을 Command(`INCR`, 반환 없음)와 Query(`GET`, 현재값 반환) 두 단계로 나누면 두 단계 사이에 다른 클라이언트가 값을 변경할 수 있습니다. 원자적으로 "증가 + 현재값 반환"이 필요한 경우 CQS를 위반하는 것이 합리적입니다.

CQRS 맥락에서의 적용: 이런 원자적 연산이 필요하다면 Command가 최소한의 결과(증가된 값)를 반환하는 것을 허용할 수 있습니다. 하지만 이것이 "Command가 읽기 모델 전체를 반환해도 된다"는 의미는 아닙니다.

</details>

---

**Q2.** Command가 void를 반환하면 클라이언트는 처리 성공 여부를 어떻게 알 수 있는가?

<details>
<summary>해설 보기</summary>

HTTP 응답 코드로 처리 결과를 전달합니다.

동기 처리의 경우 `204 No Content`는 Command가 성공적으로 처리됐음을 의미합니다. `400 Bad Request`는 Command 검증 실패, `409 Conflict`는 낙관적 잠금 충돌, `422 Unprocessable Entity`는 불변식 위반을 나타냅니다. 클라이언트는 HTTP 상태 코드만으로 성공/실패를 판단할 수 있습니다.

비동기 처리의 경우 `202 Accepted`는 Command가 수락됐지만 아직 처리 중임을 의미합니다. 응답 바디에 처리 상태를 확인할 URL을 포함할 수 있습니다. 예를 들어 `Location: /orders/{id}/status`처럼 상태 확인 엔드포인트를 제공합니다.

예외의 경우 Command 결과가 다음 화면 렌더링에 필수적이라면 (예: 생성된 리소스 ID), Command 응답에 ID만 포함하는 것이 합리적입니다. `201 Created`와 함께 `{ "id": "ORD-001" }`을 반환하면 됩니다.

</details>

---

**Q3.** GraphQL에서 Mutation과 Query는 CQS와 어떻게 대응되는가? GraphQL Mutation이 데이터를 반환하는 것은 CQS 위반인가?

<details>
<summary>해설 보기</summary>

GraphQL의 Mutation은 CQS의 Command에 해당하고, Query는 CQS의 Query에 해당합니다.

GraphQL Mutation이 데이터를 반환하는 것은 기술적으로 CQS 위반이지만, 실용적으로 허용되는 패턴입니다. GraphQL의 설계 철학상 클라이언트가 필요한 필드를 직접 지정하기 때문에, Mutation 결과로 변경된 리소스의 현재 상태를 반환하는 것이 자연스럽습니다.

CQRS 관점에서 주의할 점이 있습니다. Mutation 응답에서 읽기 모델의 비정규화 데이터를 반환하면, Event Sourcing 환경에서 읽기 모델이 아직 업데이트되지 않은 상태를 반환할 수 있습니다. 이 경우 Mutation 응답에는 변경된 Aggregate의 필드만 포함하고, 읽기 모델의 집계 데이터는 포함하지 않는 것이 안전합니다.

</details>

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[이전: Chapter 1 — CQRS 적용 판단 기준 ⬅️](../why-cqrs/05-when-to-apply-cqrs.md)** | **[다음: Command 객체 설계 ➡️](./02-command-object-design.md)**

</div>
