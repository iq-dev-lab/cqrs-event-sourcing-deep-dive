# Query 처리

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `@QueryHandler`는 어떻게 읽기 모델을 조회해 반환하는가?
- Subscription Query로 읽기 모델 업데이트를 실시간으로 클라이언트에 push하는 원리는 무엇인가?
- Point Query, Collection Query, Scatter-Gather Query의 차이는 무엇인가?
- Query 처리에서 읽기 모델과 쓰기 모델의 분리가 어떻게 적용되는가?
- Command 후 Subscription Query로 Eventual Consistency 지연 문제를 어떻게 해결하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

CQRS에서 Query는 Command와 완전히 독립된 경로다. `@QueryHandler`는 읽기 모델만 조회하고 상태를 변경하지 않는다. Subscription Query는 CQRS의 Eventual Consistency 문제를 해결하는 강력한 도구다. Command 후 읽기 모델이 업데이트될 때까지 클라이언트가 구독해 실시간으로 변경을 받는 방식은 폴링 없이 정확한 타이밍에 업데이트를 받을 수 있다.

---

## 😱 흔한 실수 (Before)

```
실수 1: @QueryHandler에서 상태 변경

  @QueryHandler
  public AccountSummaryView handle(GetAccountSummaryQuery query) {
      AccountSummary summary = repo.findById(query.accountId()).orElseThrow();
      summary.setLastAccessedAt(Instant.now()); // ❌ 상태 변경!
      repo.save(summary);
      return AccountSummaryView.from(summary);
  }
  // Query가 부수효과를 가짐 → CQS 위반

실수 2: @QueryHandler에서 Aggregate(쓰기 모델) 직접 로드

  @QueryHandler
  public AccountBalanceView handle(GetBalanceQuery query) {
      // ❌ 이벤트 스토어에서 Aggregate 로드 (비용 높음)
      Account account = accountRepository.findById(query.accountId());
      return new AccountBalanceView(account.getBalance());
      // 읽기 모델이 있는데 굳이 이벤트 리플레이
  }

실수 3: Subscription Query 정리 안 함

  SubscriptionQueryResult<?, ?> result = queryGateway.subscriptionQuery(...);
  result.updates().subscribe(...);
  // result.close() 없음 → 메모리 누수, 연결 유지
```

---

## ✨ 올바른 접근 (After — Query/Subscription Query 패턴)

```
Query 처리 3가지 패턴:

1. Point Query (단건 조회):
   특정 ID의 읽기 모델 조회
   @QueryHandler → 즉시 반환

2. Collection Query (목록 조회):
   조건에 맞는 여러 읽기 모델 조회
   @QueryHandler → List/Page 반환

3. Subscription Query (실시간 구독):
   현재 값 반환 + 이후 변경을 실시간 push
   Command 후 읽기 모델 업데이트를 기다리는 패턴에 활용
```

---

## 🔬 내부 동작 원리

### 1. @QueryHandler 기본 동작

```
Query 처리 흐름:

  queryGateway.query(
      new GetAccountSummaryQuery("ACC-001"),
      ResponseTypes.instanceOf(AccountSummaryView.class)
  )

  Step 1: QueryGateway → QueryBus
  Step 2: QueryBus가 GetAccountSummaryQuery 타입에 맞는 @QueryHandler 찾기
  Step 3: @QueryHandler 메서드 호출
  Step 4: 읽기 모델 조회 → 결과 반환
  Step 5: CompletableFuture로 호출자에게 반환

  @QueryHandler의 역할:
    읽기 모델(DB, Redis, Elasticsearch) 조회
    DTO/View 객체로 변환
    상태 변경 없음 (readOnly = true)

Query 타입과 반환 타입:

  Point Query (단건):
    queryGateway.query(query, ResponseTypes.instanceOf(T.class))
    → CompletableFuture<T>

  Collection Query (목록):
    queryGateway.query(query, ResponseTypes.multipleInstancesOf(T.class))
    → CompletableFuture<List<T>>

  Optional Query (없을 수도 있는 단건):
    queryGateway.query(query, ResponseTypes.optionalInstanceOf(T.class))
    → CompletableFuture<Optional<T>>
```

### 2. Subscription Query — 실시간 읽기 모델 구독

```
Subscription Query 동작 원리:

  subscriptionQuery = queryGateway.subscriptionQuery(
      new GetOrderStatusQuery(orderId),
      ResponseTypes.instanceOf(OrderStatusView.class),  // 초기값 타입
      ResponseTypes.instanceOf(OrderStatusUpdate.class) // 업데이트 타입
  )

  Step 1: 초기 @QueryHandler 호출 → 현재 읽기 모델 반환
  Step 2: 읽기 모델이 변경될 때마다 클라이언트에 push
  Step 3: 클라이언트가 구독 해제 시 연결 종료

Projection에서 업데이트 push:

  @EventHandler
  public void on(OrderConfirmed event) {
      // 1. 읽기 모델 업데이트
      orderSummaryRepo.updateStatus(event.orderId(), "CONFIRMED");

      // 2. 구독 중인 클라이언트에게 push
      QueryUpdateEmitter emitter;
      emitter.emit(
          GetOrderStatusQuery.class,            // 어떤 Query의 구독자에게
          q -> q.orderId().equals(event.orderId()), // 필터: 이 orderId 구독자만
          new OrderStatusUpdate(event.orderId(), "CONFIRMED", event.occurredAt())
      );
  }

활용 시나리오:

  // Command 후 읽기 모델 업데이트를 기다리는 패턴
  @PostMapping("/orders/{id}/confirm")
  public DeferredResult<OrderStatusView> confirm(@PathVariable String id) {
      DeferredResult<OrderStatusView> result = new DeferredResult<>(10_000L);

      // 구독 시작 (Command 전에!)
      SubscriptionQueryResult<OrderStatusView, OrderStatusUpdate> sub =
          queryGateway.subscriptionQuery(
              new GetOrderStatusQuery(id),
              ResponseTypes.instanceOf(OrderStatusView.class),
              ResponseTypes.instanceOf(OrderStatusUpdate.class)
          );

      // Command 전송
      commandGateway.send(new ConfirmOrderCommand(id));

      // 읽기 모델이 CONFIRMED로 업데이트될 때까지 대기
      sub.updates()
          .filter(update -> "CONFIRMED".equals(update.status()))
          .take(1)
          .subscribe(update -> {
              result.setResult(new OrderStatusView(id, update.status()));
              sub.close();
          });

      return result; // 최대 10초 대기 후 응답
  }
```

### 3. Scatter-Gather Query

```
Scatter-Gather Query:
  여러 서비스의 @QueryHandler에 동시 쿼리
  모든 응답을 수집해 병합

  사용 사례:
    여러 마이크로서비스에 분산된 읽기 모델 조합
    "모든 서비스의 주문 상태를 합쳐서 보여줘"

  queryGateway.scatterGather(
      new GetOrderDetailsQuery(orderId),
      ResponseTypes.instanceOf(OrderPartialView.class),
      5, SECONDS  // 타임아웃
  ).forEach(result -> orderDetails.merge(result));

  동작:
    → QueryBus가 GetOrderDetailsQuery를 처리하는 모든 Handler에 발송
    → 각 Handler가 독립적으로 응답
    → 5초 내 응답한 결과 모두 수집
    → 타임아웃 시 수집된 것만 반환

  Axon Server 없으면:
    같은 JVM의 모든 @QueryHandler에 발송
    → 분산 환경에서는 Axon Server 필요
```

### 4. @QueryHandler 설계 원칙

```
Query는 부수효과 없이 읽기 모델만 조회:

  @QueryHandler
  @Transactional(readOnly = true)  // 읽기 전용 트랜잭션
  public AccountSummaryView handle(GetAccountSummaryQuery query) {
      return accountSummaryRepo.findById(query.accountId())
          .map(AccountSummaryView::from)
          .orElseThrow(() -> new AccountNotFoundException(query.accountId()));
      // 상태 변경 없음, 읽기 모델에서만 조회
  }

강한 일관성이 필요한 경우:
  읽기 모델(최종 일관성)이 아닌 이벤트 스토어에서 직접 로드

  @QueryHandler
  public BigDecimal handle(GetCurrentBalanceQuery query) {
      // 이벤트 스토어에서 Aggregate 로드 → 정확한 현재 잔고
      Account account = accountRepository.findById(
          new AccountId(query.accountId())).orElseThrow();
      return account.getBalance().amount();
      // 단점: 이벤트 리플레이 비용 (스냅샷으로 완화)
  }
```

---

## 💻 실전 코드

```java
// ✅ 완전한 Query Handler 구현
@Component
public class AccountQueryHandler {

    private final AccountSummaryRepository summaryRepo;
    private final TransactionHistoryRepository historyRepo;
    private final QueryUpdateEmitter queryUpdateEmitter;

    // ── Point Query ────────────────────────────────────────────

    @QueryHandler
    @Transactional(readOnly = true)
    public AccountSummaryView handle(GetAccountSummaryQuery query) {
        return summaryRepo.findById(query.accountId())
            .map(AccountSummaryView::from)
            .orElseThrow(() -> new AccountNotFoundException(query.accountId()));
    }

    // ── Collection Query ───────────────────────────────────────

    @QueryHandler
    @Transactional(readOnly = true)
    public Page<TransactionHistoryView> handle(GetTransactionHistoryQuery query) {
        Pageable pageable = PageRequest.of(
            query.page(), query.size(),
            Sort.by("occurredAt").descending()
        );
        return historyRepo
            .findByAccountIdAndOccurredAtBetween(
                query.accountId(),
                query.from(),
                query.to(),
                pageable
            )
            .map(TransactionHistoryView::from);
    }

    // ── Subscription Query 지원 — 읽기 모델 업데이트 push ─────────

    // AccountSummaryProjection에서 호출
    public void emitBalanceUpdate(String accountId, BigDecimal newBalance) {
        queryUpdateEmitter.emit(
            GetAccountSummaryQuery.class,
            q -> q.accountId().equals(accountId),  // 이 계좌 구독자만
            new AccountBalanceUpdate(accountId, newBalance, Instant.now())
        );
    }
}

// ✅ Subscription Query를 활용한 Controller
@RestController
@RequestMapping("/accounts")
public class AccountQueryController {

    private final QueryGateway queryGateway;
    private final CommandGateway commandGateway;

    // 일반 Query (Eventual Consistency 수용)
    @GetMapping("/{id}/summary")
    public CompletableFuture<AccountSummaryView> getSummary(@PathVariable String id) {
        return queryGateway.query(
            new GetAccountSummaryQuery(id),
            ResponseTypes.instanceOf(AccountSummaryView.class)
        );
    }

    // Subscription Query로 Command 후 즉시 최신 상태 반환
    @PostMapping("/{id}/deposit")
    public DeferredResult<AccountSummaryView> deposit(
            @PathVariable String id,
            @RequestBody DepositRequest request) {

        DeferredResult<AccountSummaryView> deferredResult =
            new DeferredResult<>(5_000L); // 5초 타임아웃

        // 1. 현재 잔고 조회 + 업데이트 구독 (Command 전에 설정!)
        SubscriptionQueryResult<AccountSummaryView, AccountBalanceUpdate> subscription =
            queryGateway.subscriptionQuery(
                new GetAccountSummaryQuery(id),
                ResponseTypes.instanceOf(AccountSummaryView.class),
                ResponseTypes.instanceOf(AccountBalanceUpdate.class)
            );

        // 2. 초기값 (현재 상태)
        subscription.initialResult().subscribe(current ->
            log.debug("현재 잔고: {}", current.balance()));

        // 3. Command 전송
        commandGateway.send(new DepositMoneyCommand(id, request.amount(),
            request.depositedBy()));

        // 4. 잔고 업데이트 수신 → 결과 반환
        subscription.updates()
            .take(1) // 첫 번째 업데이트만
            .subscribe(
                update -> {
                    deferredResult.setResult(
                        new AccountSummaryView(id, update.newBalance(), "ACTIVE"));
                    subscription.close();
                },
                error -> {
                    deferredResult.setErrorResult(error);
                    subscription.close();
                }
            );

        // 타임아웃 시 현재 읽기 모델로 응답
        deferredResult.onTimeout(() -> {
            subscription.close();
            queryGateway.query(
                new GetAccountSummaryQuery(id),
                ResponseTypes.instanceOf(AccountSummaryView.class)
            ).thenAccept(deferredResult::setResult);
        });

        return deferredResult;
    }

    // SSE 기반 실시간 잔고 스트림
    @GetMapping(value = "/{id}/balance-stream",
                produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public Flux<AccountBalanceUpdate> balanceStream(@PathVariable String id) {
        SubscriptionQueryResult<AccountSummaryView, AccountBalanceUpdate> subscription =
            queryGateway.subscriptionQuery(
                new GetAccountSummaryQuery(id),
                ResponseTypes.instanceOf(AccountSummaryView.class),
                ResponseTypes.instanceOf(AccountBalanceUpdate.class)
            );

        return subscription.updates()
            .doOnCancel(subscription::close)
            .doOnError(e -> subscription.close());
    }
}

// ✅ Projection에서 QueryUpdateEmitter 호출
@ProcessingGroup("account-summary")
@Component
public class AccountSummaryProjection {

    private final AccountSummaryRepository repo;
    private final QueryUpdateEmitter queryUpdateEmitter;

    @EventHandler
    public void on(MoneyDeposited event, @SequenceNumber long seq) {
        repo.updateBalanceIfNewer(event.accountId(), event.balanceAfter(), seq);

        // Subscription Query 구독자에게 push
        queryUpdateEmitter.emit(
            GetAccountSummaryQuery.class,
            q -> q.accountId().equals(event.accountId()),
            new AccountBalanceUpdate(event.accountId(), event.balanceAfter(), event.occurredAt())
        );
    }
}
```

---

## 📊 패턴 비교

```
Query 처리 방식 비교:

┌──────────────────┬────────────────┬──────────────┬──────────────┐
│ 방식             │ 일관성         │ 실시간성     │ 복잡도       │
├──────────────────┼────────────────┼──────────────┼──────────────┤
│ 읽기 모델 조회   │ 최종 일관성    │ 낮음(지연)  │ 낮음         │
├──────────────────┼────────────────┼──────────────┼──────────────┤
│ Subscription     │ 최종 일관성    │ 높음(push)  │ 중간         │
│ Query            │                │              │              │
├──────────────────┼────────────────┼──────────────┼──────────────┤
│ Aggregate 직접   │ 강한 일관성    │ 즉시         │ 높음(비용)   │
│ 조회             │                │              │              │
├──────────────────┼────────────────┼──────────────┼──────────────┤
│ Scatter-Gather   │ 최종 일관성    │ 낮음         │ 높음         │
└──────────────────┴────────────────┴──────────────┴──────────────┘

언제 무엇을:
  목록 조회, 통계 → 읽기 모델 조회 (빠름, Eventual Consistency 수용)
  Command 후 즉시 결과 → Subscription Query
  잔고 확인 후 즉각 처리 → Aggregate 직접 조회 (강한 일관성)
  분산 서비스 조합 → Scatter-Gather
```

---

## ⚖️ 트레이드오프

```
Subscription Query 비용:
  서버에서 구독 연결 유지 (메모리, 소켓)
  QueryUpdateEmitter 호출 비용 (구독자 수만큼 push)
  편익: 폴링 없이 정확한 타이밍에 업데이트

Scatter-Gather 주의사항:
  일부 서비스가 응답 없으면 타임아웃까지 대기
  응답 병합 로직 복잡할 수 있음
  → 타임아웃을 짧게, 응답 없는 서비스는 기본값으로 처리
```

---

## 📌 핵심 정리

```
Axon Query 처리 핵심:

@QueryHandler:
  읽기 모델만 조회 (부수효과 없음)
  @Transactional(readOnly = true) 필수
  QueryBus → Handler 자동 라우팅

Subscription Query:
  현재값 + 이후 업데이트 push
  Command 후 Eventual Consistency 지연 해결
  Projection의 QueryUpdateEmitter.emit()으로 push

Scatter-Gather Query:
  여러 @QueryHandler에 동시 발송
  분산 환경에서 여러 읽기 모델 병합

강한 일관성 필요:
  Aggregate 직접 로드 (이벤트 스토어, 비용 높음)
  스냅샷으로 리플레이 비용 완화
```

---

## 🤔 생각해볼 문제

**Q1.** Subscription Query를 사용할 때 Command보다 구독을 먼저 설정해야 하는 이유는 무엇인가?

<details>
<summary>해설 보기</summary>

구독보다 Command가 먼저 처리되면 이벤트를 놓칩니다.

타임라인이 다음과 같습니다. Command를 먼저 보내면 이벤트가 발행되고 Projection이 읽기 모델을 업데이트합니다. 이 시점에 `queryUpdateEmitter.emit()`이 호출됩니다. 그 후에 구독을 설정하면 이미 발행된 이벤트는 다시 오지 않으므로 구독자가 업데이트를 받지 못합니다.

구독을 먼저 설정하면 Command 처리 결과로 발행되는 업데이트를 놓치지 않습니다. Axon의 Subscription Query는 구독 시작 후 발생하는 모든 업데이트를 버퍼링해 구독자에게 전달합니다.

실무에서는 `subscriptionQuery()` 호출 → Command 전송 → `updates()` 구독 순서를 엄수합니다. Spring의 DeferredResult나 WebFlux의 Mono를 활용해 비동기로 처리합니다.

</details>

---

**Q2.** Scatter-Gather Query에서 하나의 서비스가 타임아웃되면 전체 응답이 지연된다. 어떻게 처리해야 하는가?

<details>
<summary>해설 보기</summary>

타임아웃을 짧게 설정하고 타임아웃 시 수집된 응답만 사용하는 방식이 권장됩니다.

Axon의 `scatterGather(query, responseType, timeout, unit)`는 타임아웃 시 수집된 응답만 반환합니다. 응답이 없는 서비스는 결과에 포함되지 않습니다. 이를 처리하는 코드에서 응답이 없는 서비스에 대한 기본값을 제공하면 됩니다.

Circuit Breaker를 함께 사용하면 반복적으로 타임아웃이 발생하는 서비스를 일시적으로 제외할 수 있습니다. Resilience4j의 `@CircuitBreaker`를 `@QueryHandler`에 적용합니다.

설계 단계에서 Scatter-Gather가 정말 필요한지 재검토합니다. 각 서비스가 독립 읽기 모델을 가지고 있다면, 클라이언트에서 여러 API를 병렬 호출하는 방식이 더 단순하고 타임아웃 처리도 명확합니다.

</details>

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[이전: 프로젝션 구현 ⬅️](./03-projection-implementation.md)** | **[다음: Axon 없이 직접 구현 ➡️](./05-pure-spring-implementation.md)**

</div>
