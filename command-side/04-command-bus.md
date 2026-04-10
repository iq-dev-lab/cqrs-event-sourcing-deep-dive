# Command Bus 패턴

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Command Bus가 없으면 어떤 문제가 생기는가?
- Command Bus의 핵심 역할은 라우팅인가, 미들웨어인가?
- 로깅, 검증, 트랜잭션을 Command Bus에서 처리하면 어떤 이점이 있는가?
- Spring에서 Command Bus를 직접 구현하는 방법은 무엇인가?
- Axon Command Gateway와 직접 구현의 차이는 무엇인가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

Command Bus는 Command와 Handler 사이의 중간 계층이다. "왜 이 계층이 필요한가?"를 이해하지 못하면 Command Bus를 단순 라우터로만 쓰게 된다. Command Bus의 진짜 가치는 횡단 관심사(로깅, 검증, 트랜잭션, 멱등성 체크, 인증)를 Command 처리 파이프라인으로 분리하는 데 있다.

---

## 😱 흔한 실수 (Before — Command Bus 없이 직접 호출)

```
Command Bus 없이 Controller가 Handler를 직접 호출:

@RestController
public class OrderController {

    @Autowired ConfirmOrderHandler confirmHandler;
    @Autowired CancelOrderHandler  cancelHandler;
    @Autowired ShipOrderHandler    shipHandler;

    @PostMapping("/orders/{id}/confirm")
    public void confirm(@PathVariable String id) {
        // ❌ 횡단 관심사가 각 엔드포인트마다 중복
        log.info("Command 시작: ConfirmOrder, orderId={}", id);
        validateToken(request);

        confirmHandler.handle(new ConfirmOrderCommand(id, getCurrentUser()));

        log.info("Command 완료: ConfirmOrder, orderId={}", id);
        metrics.increment("command.confirm_order.success");
    }

    @PostMapping("/orders/{id}/cancel")
    public void cancel(@PathVariable String id, @RequestBody CancelRequest req) {
        // ❌ 동일한 횡단 관심사 코드 반복
        log.info("Command 시작: CancelOrder, orderId={}", id);
        validateToken(request);

        cancelHandler.handle(new CancelOrderCommand(id, req.reason(), getCurrentUser()));

        log.info("Command 완료: CancelOrder, orderId={}", id);
        metrics.increment("command.cancel_order.success");
    }
    // 모든 Command 엔드포인트마다 동일한 패턴 반복
}
```

---

## ✨ 올바른 접근 (After — Command Bus 미들웨어 파이프라인)

```
Command Bus 파이프라인:

  commandBus.send(command)
    → [미들웨어 1] 로깅 (Command 수신 로그)
    → [미들웨어 2] 인증/인가 (Command별 권한 확인)
    → [미들웨어 3] 멱등성 체크 (동일 요청 중복 처리 방지)
    → [미들웨어 4] 트랜잭션 시작
    → [Handler] 실제 Command 처리
    → [미들웨어 4] 트랜잭션 커밋 (or 롤백)
    → [미들웨어 3] 처리 결과 기록
    → [미들웨어 1] 로깅 (처리 완료 로그 + 소요 시간)
    → 완료

이점:
  횡단 관심사가 Command Bus 미들웨어에 한 번만 구현
  모든 Command가 동일한 파이프라인 통과 → 일관성 보장
  새 Command 추가 시 횡단 관심사 구현 불필요
  각 미들웨어를 독립적으로 테스트 가능
```

---

## 🔬 내부 동작 원리

### 1. Command Bus의 핵심 구조

```
Command Bus = 라우터 + 미들웨어 체인

라우터 역할:
  ConfirmOrderCommand   → ConfirmOrderHandler
  CancelOrderCommand    → CancelOrderHandler
  WithdrawMoneyCommand  → AccountCommandHandler
  → Command 타입 → Handler 매핑

미들웨어 체인:
  Pipeline 패턴 (Chain of Responsibility)
  각 미들웨어가 Command를 받아 처리 후 다음 미들웨어에 전달
  마지막에 실제 Handler 호출

  interface CommandMiddleware {
      void handle(Command command, CommandHandler next);
  }

  CommandBus 실행 순서:
    LoggingMiddleware.handle(cmd, next)
      → AuthorizationMiddleware.handle(cmd, next)
        → IdempotencyMiddleware.handle(cmd, next)
          → TransactionMiddleware.handle(cmd, next)
            → ActualHandler.handle(cmd)    ← 실제 처리
          ← (트랜잭션 커밋)
        ← (멱등성 기록)
      ← (인가 완료)
    ← (로그 완료)
```

### 2. 미들웨어별 역할

```
미들웨어 1 — 로깅:

  class LoggingMiddleware implements CommandMiddleware {
      void handle(Command command, CommandHandler next) {
          String commandType = command.getClass().getSimpleName();
          log.info("[CMD START] type={}, id={}", commandType, command.commandId());
          long start = System.currentTimeMillis();
          try {
              next.handle(command);
              long elapsed = System.currentTimeMillis() - start;
              log.info("[CMD SUCCESS] type={}, elapsed={}ms", commandType, elapsed);
              metrics.recordSuccess(commandType, elapsed);
          } catch (Exception e) {
              log.error("[CMD FAILED] type={}, error={}", commandType, e.getMessage());
              metrics.recordFailure(commandType);
              throw e;
          }
      }
  }

미들웨어 2 — 인증/인가:

  class AuthorizationMiddleware implements CommandMiddleware {
      void handle(Command command, CommandHandler next) {
          Authentication auth = SecurityContextHolder.getContext().getAuthentication();
          // Command별 필요 권한 확인
          RequiredRole requiredRole = command.getClass().getAnnotation(RequiredRole.class);
          if (requiredRole != null && !auth.getAuthorities().contains(requiredRole.value()))
              throw new AccessDeniedException("권한 없음: " + requiredRole.value());
          next.handle(command);
      }
  }

미들웨어 3 — 멱등성 체크:

  class IdempotencyMiddleware implements CommandMiddleware {
      void handle(Command command, CommandHandler next) {
          if (command instanceof IdempotentCommand ic) {
              UUID key = ic.idempotencyKey();
              if (processedCommands.contains(key)) {
                  log.info("중복 요청 무시: key={}", key);
                  return; // 실제 처리 없이 종료
              }
              next.handle(command);
              processedCommands.add(key); // 처리 완료 기록
          } else {
              next.handle(command);
          }
      }
  }

미들웨어 4 — 트랜잭션:

  class TransactionMiddleware implements CommandMiddleware {
      void handle(Command command, CommandHandler next) {
          transactionTemplate.execute(status -> {
              try {
                  next.handle(command);
              } catch (DomainException e) {
                  status.setRollbackOnly();
                  throw e;
              }
              return null;
          });
      }
  }
```

### 3. Spring에서 Command Bus 구현

```
Spring ApplicationContext를 활용한 자동 등록:

  // 핸들러 자동 발견 및 등록
  @Component
  public class CommandBus implements ApplicationContextAware {

      private final Map<Class<?>, CommandHandler<?>> handlers = new HashMap<>();
      private final List<CommandMiddleware> middlewares;

      @Override
      public void setApplicationContext(ApplicationContext ctx) {
          // @CommandHandler 어노테이션이 붙은 Bean 자동 수집
          ctx.getBeansOfType(CommandHandler.class).values()
              .forEach(handler -> {
                  Type genericType = /* 제네릭 타입 추출 */;
                  handlers.put((Class<?>) genericType, handler);
              });

          // 미들웨어 순서 정렬
          middlewares.sort(Comparator.comparing(m ->
              m.getClass().getAnnotation(Order.class).value()
          ));
      }

      public <C extends Command> void send(C command) {
          @SuppressWarnings("unchecked")
          CommandHandler<C> handler = (CommandHandler<C>) handlers.get(command.getClass());
          if (handler == null) throw new NoHandlerFoundException(command.getClass());

          // 미들웨어 체인 구성
          CommandHandler<C> chain = buildChain(handler, middlewares);
          chain.handle(command);
      }
  }
```

### 4. Axon Command Gateway vs 직접 구현

```
Axon Framework의 Command Gateway:

  @Autowired CommandGateway commandGateway;

  // 동기 전송
  commandGateway.sendAndWait(new ConfirmOrderCommand(orderId));

  // 비동기 전송 (CompletableFuture)
  CompletableFuture<Void> future = commandGateway.send(new ConfirmOrderCommand(orderId));

  // 타임아웃 지정
  commandGateway.sendAndWait(new ConfirmOrderCommand(orderId), 5, TimeUnit.SECONDS);

  Axon이 제공하는 것:
    Command 라우팅 (로컬 or 원격 핸들러)
    재시도 정책 (@Retryable)
    Command 우선순위 설정
    분산 환경 지원 (Axon Server)
    Command 이력 추적 (Axon Server Dashboard)

직접 구현이 유리한 경우:
  Axon Server 없이 단일 서비스 내 CQRS
  미들웨어 파이프라인을 직접 제어해야 할 때
  Axon 의존성 없이 테스트하고 싶을 때
  팀이 Axon 학습 비용을 감당하기 어려울 때

Axon이 유리한 경우:
  분산 환경에서 여러 서비스 간 Command 라우팅
  Command 이력 추적, 재시도, 모니터링이 필요할 때
  Event Sourcing과 완전한 통합
```

---

## 💻 실전 코드

```java
// ==============================
// Spring 기반 Command Bus 직접 구현
// ==============================

// ✅ Command Bus 인터페이스
public interface CommandBus {
    <C extends Command> void send(C command);
}

// ✅ 미들웨어 인터페이스
@FunctionalInterface
public interface CommandMiddleware {
    void handle(Command command, Runnable next);
}

// ✅ 미들웨어 구현들
@Component
@Order(1)
public class LoggingCommandMiddleware implements CommandMiddleware {

    private static final Logger log = LoggerFactory.getLogger(LoggingCommandMiddleware.class);
    private final MeterRegistry meterRegistry;

    @Override
    public void handle(Command command, Runnable next) {
        String type = command.getClass().getSimpleName();
        String cmdId = command instanceof IdentifiableCommand ic ?
            ic.commandId().toString() : "N/A";

        log.info("[CMD] START type={} id={}", type, cmdId);
        long start = System.nanoTime();

        try {
            next.run();
            long elapsed = TimeUnit.NANOSECONDS.toMillis(System.nanoTime() - start);
            log.info("[CMD] SUCCESS type={} elapsed={}ms", type, elapsed);
            meterRegistry.counter("command.success", "type", type).increment();
            meterRegistry.timer("command.duration", "type", type)
                .record(elapsed, TimeUnit.MILLISECONDS);
        } catch (DomainException e) {
            log.warn("[CMD] DOMAIN_ERROR type={} error={}", type, e.getMessage());
            meterRegistry.counter("command.domain_error", "type", type).increment();
            throw e;
        } catch (Exception e) {
            log.error("[CMD] ERROR type={} error={}", type, e.getMessage(), e);
            meterRegistry.counter("command.error", "type", type).increment();
            throw e;
        }
    }
}

@Component
@Order(2)
public class IdempotencyCommandMiddleware implements CommandMiddleware {

    private final ProcessedCommandRepository processedCommandRepo;

    @Override
    public void handle(Command command, Runnable next) {
        if (!(command instanceof IdempotentCommand ic)) {
            next.run();
            return;
        }

        UUID key = ic.idempotencyKey();
        if (processedCommandRepo.existsByKey(key)) {
            log.info("[CMD] DUPLICATE IGNORED key={}", key);
            return; // 중복 요청 — 실제 처리 없이 성공으로 처리
        }

        next.run();
        processedCommandRepo.save(new ProcessedCommand(key, Instant.now()));
    }
}

// ✅ Command Bus 구현
@Component
public class SimpleCommandBus implements CommandBus {

    private final Map<Class<? extends Command>, CommandHandler<?>> handlers;
    private final List<CommandMiddleware> middlewares;

    public SimpleCommandBus(
            List<CommandHandler<?>> handlerList,
            List<CommandMiddleware> middlewareList) {

        // Handler 등록: Command 타입 → Handler 매핑
        this.handlers = handlerList.stream()
            .collect(toMap(this::extractCommandType, h -> h));

        // 미들웨어: @Order 기준 정렬
        this.middlewares = middlewareList.stream()
            .sorted(Comparator.comparingInt(m ->
                Optional.ofNullable(m.getClass().getAnnotation(Order.class))
                    .map(Order::value).orElse(Integer.MAX_VALUE)))
            .collect(toList());
    }

    @Override
    public <C extends Command> void send(C command) {
        @SuppressWarnings("unchecked")
        CommandHandler<C> handler = (CommandHandler<C>)
            handlers.get(command.getClass());
        if (handler == null)
            throw new NoCommandHandlerException(command.getClass());

        // 미들웨어 체인 + Handler 실행
        executeWithMiddlewares(command, handler, 0);
    }

    private <C extends Command> void executeWithMiddlewares(
            C command, CommandHandler<C> handler, int index) {
        if (index >= middlewares.size()) {
            handler.handle(command);
            return;
        }
        middlewares.get(index).handle(command,
            () -> executeWithMiddlewares(command, handler, index + 1));
    }

    @SuppressWarnings("unchecked")
    private Class<? extends Command> extractCommandType(CommandHandler<?> handler) {
        // 제네릭 타입 파라미터 추출 (리플렉션)
        return (Class<? extends Command>) ((ParameterizedType)
            handler.getClass().getGenericInterfaces()[0])
            .getActualTypeArguments()[0];
    }
}

// ✅ Controller — Command Bus만 알면 됨
@RestController
@RequestMapping("/accounts")
public class AccountCommandController {

    private final CommandBus commandBus;

    @PostMapping("/{id}/withdraw")
    @ResponseStatus(HttpStatus.ACCEPTED)
    public void withdraw(
            @PathVariable String id,
            @RequestBody @Valid WithdrawRequest request,
            @AuthenticationPrincipal UserDetails user) {

        commandBus.send(new WithdrawMoneyCommand(
            id,
            request.amount(),
            user.getUsername(),
            Instant.now(),
            request.idempotencyKey()
        ));
        // Controller는 Handler가 누구인지 모름 — Command Bus에 위임
    }
}
```

---

## 📊 패턴 비교

```
직접 호출 vs Command Bus:

┌──────────────────────┬──────────────────────┬──────────────────────┐
│ 항목                 │ 직접 호출             │ Command Bus          │
├──────────────────────┼──────────────────────┼──────────────────────┤
│ 횡단 관심사          │ 각 Handler에 중복     │ 미들웨어에 한 번만   │
├──────────────────────┼──────────────────────┼──────────────────────┤
│ Command 추가         │ Controller + Handler  │ Handler만 추가       │
├──────────────────────┼──────────────────────┼──────────────────────┤
│ 테스트               │ 횡단 관심사 포함 테스트│ 미들웨어 독립 테스트 │
├──────────────────────┼──────────────────────┼──────────────────────┤
│ 비동기 지원          │ 별도 구현 필요        │ Bus에서 일괄 지원    │
├──────────────────────┼──────────────────────┼──────────────────────┤
│ 복잡도               │ 낮음                  │ 중간                 │
└──────────────────────┴──────────────────────┴──────────────────────┘

Axon Command Gateway vs 직접 구현:
  Axon: 분산 환경, 재시도, 모니터링 내장
  직접: 단순, 의존성 최소, 완전한 제어
```

---

## ⚖️ 트레이드오프

```
Command Bus 도입 비용:
  추가 추상화 레이어 → 디버깅 시 흐름 파악 어려울 수 있음
  미들웨어 순서 실수 → 예상치 못한 동작
  제네릭 타입 처리 → 리플렉션 코드 복잡

Command Bus 편익:
  횡단 관심사 일원화 → 코드 중복 제거
  Command 추가 시 핸들러만 추가 → OCP 준수
  미들웨어 테스트 독립성
  비동기 Command 지원을 Bus에서 일괄 처리 가능
```

---

## 📌 핵심 정리

```
Command Bus 핵심:

역할:
  라우팅: Command 타입 → Handler 매핑
  미들웨어: 횡단 관심사 파이프라인

미들웨어 파이프라인:
  로깅 → 인증/인가 → 멱등성 → 트랜잭션 → Handler

Spring 구현 방법:
  @Component Handler 자동 수집
  미들웨어 @Order로 순서 지정
  체인 패턴으로 파이프라인 구성

Axon vs 직접 구현:
  단일 서비스 CQRS → 직접 구현으로 충분
  분산 환경, 모니터링 → Axon Command Gateway
```

---

## 🤔 생각해볼 문제

**Q1.** Command Bus 미들웨어에서 트랜잭션을 관리하면 Command Handler의 `@Transactional`이 불필요한가?

<details>
<summary>해설 보기</summary>

Command Bus 미들웨어에서 트랜잭션을 관리하면 Handler의 `@Transactional`을 제거할 수 있습니다. 하지만 실무에서는 두 가지 이유로 Handler에도 `@Transactional`을 유지하는 것이 일반적입니다.

첫째, Command Bus 없이 Handler를 직접 호출하는 테스트나 시나리오에서도 트랜잭션이 보장됩니다. 둘째, 특정 Command에 다른 트랜잭션 설정이 필요한 경우(예: `readOnly`, `timeout` 설정)를 Handler 레벨에서 제어할 수 있습니다.

권장 패턴은 Command Bus 미들웨어에서 기본 트랜잭션 시작을 담당하고, 특수한 트랜잭션 설정이 필요한 Handler에서만 `@Transactional`로 재정의하는 것입니다.

</details>

---

**Q2.** Command Bus를 통해 비동기로 Command를 처리하려면 어떻게 설계해야 하는가?

<details>
<summary>해설 보기</summary>

비동기 Command 처리는 두 수준으로 나뉩니다.

단순 비동기(같은 JVM, 별도 스레드)는 Command Bus의 `send()` 메서드에서 `CompletableFuture.runAsync()`로 Handler를 별도 스레드에서 실행합니다. `ExecutorService`를 주입해 스레드 풀을 관리합니다. 이 방식은 빠르지만 JVM 재시작 시 처리 중인 Command가 유실될 수 있습니다.

신뢰성 있는 비동기는 Command를 메시지 큐(Kafka, RabbitMQ, SQS)에 발행하고, 별도 Consumer가 Command를 소비해 Handler를 호출합니다. JVM 재시작, 장애에도 Command가 유실되지 않습니다. 이 경우 Command 직렬화 전략(JSON, Avro)과 스키마 버전 관리가 필요합니다.

Axon Framework는 Command를 Axon Server에 발행하고 구독자가 처리하는 방식으로 이를 기본 지원합니다.

</details>

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[이전: Command Handler와 Aggregate ⬅️](./03-command-handler-aggregate.md)** | **[다음: Optimistic Locking in CQRS ➡️](./05-optimistic-locking.md)**

</div>
