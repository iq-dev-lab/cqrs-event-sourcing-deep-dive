# Aggregate 구현

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `@Aggregate`, `@AggregateIdentifier`, `@CommandHandler`, `@EventSourcingHandler`의 역할은 무엇인가?
- `AggregateLifecycle.apply()`가 내부적으로 하는 일은 무엇인가?
- Aggregate 생성(Constructor `@CommandHandler`)과 메서드 `@CommandHandler`의 차이는 무엇인가?
- Expected Version으로 낙관적 잠금은 어떻게 구현하는가?
- Aggregate 테스트는 어떻게 작성하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

Axon Aggregate 구현은 단순해 보이지만 세부 규칙을 모르면 미묘한 버그가 발생한다. `@EventSourcingHandler`에서 부수효과를 실행하면 재구축 시마다 이메일이 발송된다. `apply()` 없이 상태를 직접 변경하면 이벤트 스토어에 기록이 없어 재구축 시 해당 변경이 사라진다. 이 규칙들을 명확히 이해하는 것이 올바른 Aggregate 구현의 출발점이다.

---

## 😱 흔한 실수 (Before)

```
실수 1: @EventSourcingHandler에서 검증 로직 실행

  @EventSourcingHandler
  public void on(MoneyWithdrawn event) {
      if (this.balance.compareTo(event.amount()) < 0)  // ❌ 검증을 여기서
          throw new InsufficientBalanceException();
      this.balance = balance.subtract(event.amount());
  }
  // 재구성 시: 과거에 이미 성공한 이벤트인데 검증 재실행 → 예외 가능

실수 2: apply() 없이 상태 직접 변경

  @CommandHandler
  public void handle(WithdrawMoneyCommand cmd) {
      this.balance = balance.subtract(cmd.amount()); // ❌ 직접 변경
      // apply() 호출 없음 → 이벤트 스토어 기록 없음
      // 재구성 시 이 변경 사라짐
  }

실수 3: Aggregate 생성자에 @CommandHandler 누락

  // Axon은 Aggregate 생성 Command를 생성자 @CommandHandler로 처리
  public AccountAggregate() {}  // ← 기본 생성자만 있으면 생성 Command 처리 불가
  // → "No handler found for CreateAccountCommand"
```

---

## ✨ 올바른 접근 (After — 올바른 Axon Aggregate 구조)

```
Aggregate 구조:

@Aggregate
public class AccountAggregate {

    // Aggregate 식별자
    @AggregateIdentifier
    private String accountId;

    // 현재 상태 필드
    private BigDecimal balance;
    private AccountStatus status;

    // ── 생성 Command (static factory or constructor) ──────────
    // @CommandHandler가 붙은 생성자 또는 static 메서드
    // → 새 Aggregate 인스턴스 생성

    // ── 행동 Command ──────────────────────────────────────────
    // @CommandHandler 인스턴스 메서드
    // 불변식 검증 + AggregateLifecycle.apply() 호출

    // ── 상태 전환 ─────────────────────────────────────────────
    // @EventSourcingHandler 메서드
    // apply() 호출 시 + 재구성 시 호출
    // 순수 상태 업데이트만 (부수효과 없음)
}
```

---

## 🔬 내부 동작 원리

### 1. Aggregate 생명주기 — 생성부터 저장까지

```
새 Aggregate 생성:

  commandGateway.send(new CreateAccountCommand("ACC-001", "USER-42"))

  Axon이 하는 일:
    1. CreateAccountCommand → @CommandHandler 생성자 찾기
    2. 기본 생성자로 AccountAggregate() 인스턴스 생성
    3. @CommandHandler 생성자 호출
       → 불변식 검증
       → AggregateLifecycle.apply(new AccountCreated(...))
          → @EventSourcingHandler on(AccountCreated) 즉시 호출
          → accountId, balance 필드 초기화
          → 이벤트 큐에 추가
    4. Unit of Work 커밋 시
       → 이벤트 스토어에 AccountCreated 저장
       → EventBus에 AccountCreated 발행

기존 Aggregate 로드:

  commandGateway.send(new WithdrawMoneyCommand("ACC-001", 100000))

  Axon이 하는 일:
    1. WithdrawMoneyCommand → @CommandHandler 찾기 (AccountAggregate)
    2. AggregateIdentifier: "ACC-001"
    3. EventStore에서 "AccountAggregate-ACC-001" 스트림 로드
    4. 기본 생성자 AccountAggregate() 생성
    5. 이벤트 순서대로 @EventSourcingHandler 호출 (재구성)
       on(AccountCreated) → accountId="ACC-001", balance=0
       on(MoneyDeposited) → balance=500000
       on(MoneyDeposited) → balance=700000
    6. @CommandHandler handle(WithdrawMoneyCommand) 호출
       → 불변식 검증 (700000 >= 100000 ✓)
       → apply(new MoneyWithdrawn(...))
          → @EventSourcingHandler on(MoneyWithdrawn) 호출 (balance=600000)
    7. 커밋 → MoneyWithdrawn 이벤트 스토어 저장 + 발행
```

### 2. AggregateLifecycle.apply() 동작

```
apply() 호출 시 발생하는 일:

  AggregateLifecycle.apply(new MoneyWithdrawn(accountId, amount, 600000))

  Step 1: 이벤트를 현재 Unit of Work에 등록
          (아직 이벤트 스토어에 저장되지 않음)

  Step 2: 즉시 해당 @EventSourcingHandler 호출
          on(MoneyWithdrawn event)
          → this.balance = Money.of(event.balanceAfter())
          → Aggregate 상태가 즉시 변경됨

  Step 3: CommandHandler가 완료되면 Unit of Work 커밋
          → 이벤트 스토어에 저장 (Axon Server or JPA)
          → EventBus에 발행 → Projection, Saga 처리

apply() 체인 (한 Command에서 여러 이벤트):

  void handle(TransferMoneyCommand cmd) {
      apply(new MoneyTransferInitiated(cmd.fromId, cmd.toId, cmd.amount));
      // @EventSourcingHandler: from 잔고 차감
      apply(new MoneyTransferCompleted(cmd.fromId, cmd.toId, cmd.amount));
      // @EventSourcingHandler: 추가 상태 업데이트
  }
  // 두 이벤트 모두 같은 트랜잭션에 저장
  // @EventSourcingHandler 순서대로 즉시 호출
  // → 두 번째 apply() 시점에 첫 번째 apply()로 변경된 상태 반영됨
```

### 3. Expected Version — 낙관적 잠금

```
Axon에서 낙관적 잠금:

  기본적으로 Aggregate 로드 시 현재 버전 확인
  → 저장 시 버전이 맞지 않으면 ConcurrencyException

Command에 Expected Version 포함:

  @TargetAggregateVersion 어노테이션:
    public record WithdrawMoneyCommand(
        @TargetAggregateIdentifier String accountId,
        BigDecimal amount,
        @TargetAggregateVersion Long expectedVersion  // 클라이언트가 아는 버전
    ) {}

  Axon이 하는 일:
    1. Aggregate 로드 (현재 버전 = 5)
    2. Command의 expectedVersion = 5와 비교
    3. 불일치 시 ConcurrencyException → HTTP 409

직접 버전 확인:

  @CommandHandler
  public void handle(WithdrawMoneyCommand cmd) {
      if (cmd.expectedVersion() != null) {
          long currentVersion = AggregateLifecycle.getVersion();
          if (!cmd.expectedVersion().equals(currentVersion))
              throw new ConcurrencyException(
                  "버전 불일치: expected=" + cmd.expectedVersion() +
                  " actual=" + currentVersion);
      }
      // ... 처리
  }
```

### 4. Aggregate 테스트 — AggregateTestFixture

```
Axon이 제공하는 테스트 도구:

  AggregateTestFixture<AccountAggregate> fixture;

  @BeforeEach
  void setUp() {
      fixture = new AggregateTestFixture<>(AccountAggregate.class);
  }

  테스트 패턴:
    fixture
      .given(...)   // 이미 발생한 이벤트 (재구성)
      .when(...)    // 실행할 Command
      .expectEvents(...)  // 기대하는 이벤트
      .expectException(...)  // 예상 예외

  장점:
    실제 Axon 동작 그대로 테스트
    DB, Kafka 없이 단위 테스트
    이벤트 재구성 → 불변식 검증 전체 흐름 테스트
```

---

## 💻 실전 코드

```java
// ✅ 완전한 Axon Aggregate 구현 — 은행 계좌

@Aggregate
public class AccountAggregate {

    @AggregateIdentifier
    private String accountId;
    private BigDecimal balance;
    private AccountStatus status;
    private String ownerId;

    // 기본 생성자 (Axon이 재구성 시 사용)
    protected AccountAggregate() {}

    // ── 생성 Command ──────────────────────────────────────────

    @CommandHandler
    public AccountAggregate(CreateAccountCommand cmd) {
        // 구문 검증
        if (cmd.initialDeposit().compareTo(BigDecimal.ZERO) < 0)
            throw new IllegalArgumentException("초기 입금액은 0 이상이어야 합니다");

        apply(new AccountCreated(
            cmd.accountId(),
            cmd.ownerId(),
            cmd.initialDeposit(),
            Instant.now()
        ));
    }

    // ── 행동 Command ──────────────────────────────────────────

    @CommandHandler
    public void handle(DepositMoneyCommand cmd) {
        if (status == AccountStatus.CLOSED)
            throw new AccountClosedException(accountId);
        if (cmd.amount().compareTo(BigDecimal.ZERO) <= 0)
            throw new IllegalArgumentException("입금액은 0보다 커야 합니다");

        apply(new MoneyDeposited(
            accountId, cmd.amount(),
            balance.add(cmd.amount()),
            cmd.depositedBy(), Instant.now()
        ));
    }

    @CommandHandler
    public void handle(WithdrawMoneyCommand cmd) {
        // 의미 검증 (불변식)
        if (status != AccountStatus.ACTIVE)
            throw new AccountNotActiveException(accountId, status);
        if (balance.compareTo(cmd.amount()) < 0)
            throw new InsufficientBalanceException(accountId, cmd.amount(), balance);

        apply(new MoneyWithdrawn(
            accountId, cmd.amount(),
            balance.subtract(cmd.amount()),
            cmd.requestedBy(), Instant.now()
        ));
    }

    @CommandHandler
    public void handle(FreezeAccountCommand cmd) {
        if (status != AccountStatus.ACTIVE)
            throw new AccountNotActiveException(accountId, status);

        apply(new AccountFrozen(accountId, cmd.reason(), cmd.frozenBy(), Instant.now()));
    }

    // ── 상태 전환 (@EventSourcingHandler) ─────────────────────
    // 규칙: 순수 상태 업데이트만. 검증 없음. 부수효과 없음.

    @EventSourcingHandler
    public void on(AccountCreated event) {
        this.accountId = event.accountId();
        this.ownerId = event.ownerId();
        this.balance = event.initialDeposit();
        this.status = AccountStatus.ACTIVE;
    }

    @EventSourcingHandler
    public void on(MoneyDeposited event) {
        this.balance = event.balanceAfter(); // 절대값 사용
    }

    @EventSourcingHandler
    public void on(MoneyWithdrawn event) {
        this.balance = event.balanceAfter(); // 절대값 사용
    }

    @EventSourcingHandler
    public void on(AccountFrozen event) {
        this.status = AccountStatus.FROZEN;
    }

    @EventSourcingHandler
    public void on(AccountUnfrozen event) {
        this.status = AccountStatus.ACTIVE;
    }
}

// ✅ Command 정의
public record CreateAccountCommand(
    @TargetAggregateIdentifier String accountId,
    String ownerId,
    BigDecimal initialDeposit
) {}

public record WithdrawMoneyCommand(
    @TargetAggregateIdentifier String accountId,
    BigDecimal amount,
    String requestedBy,
    @TargetAggregateVersion Long expectedVersion  // 낙관적 잠금
) {}

// ✅ AggregateTestFixture 기반 단위 테스트
class AccountAggregateTest {

    private AggregateTestFixture<AccountAggregate> fixture;

    @BeforeEach
    void setUp() {
        fixture = new AggregateTestFixture<>(AccountAggregate.class);
    }

    @Test
    void 계좌_생성_성공() {
        fixture
            .givenNoPriorActivity()
            .when(new CreateAccountCommand("ACC-001", "USER-42", new BigDecimal("500000")))
            .expectSuccessfulHandlerExecution()
            .expectEvents(new AccountCreated("ACC-001", "USER-42",
                new BigDecimal("500000"), /* any */ null));
    }

    @Test
    void 출금_성공() {
        fixture
            .given(
                new AccountCreated("ACC-001", "USER-42", new BigDecimal("500000"), Instant.now()),
                new MoneyDeposited("ACC-001", new BigDecimal("200000"),
                    new BigDecimal("700000"), "ATM", Instant.now())
            )
            .when(new WithdrawMoneyCommand("ACC-001",
                new BigDecimal("100000"), "USER-42", null))
            .expectEvents(new MoneyWithdrawn("ACC-001",
                new BigDecimal("100000"), new BigDecimal("600000"),
                "USER-42", null));
    }

    @Test
    void 잔고_부족_시_예외() {
        fixture
            .given(new AccountCreated("ACC-001", "USER-42",
                new BigDecimal("50000"), Instant.now()))
            .when(new WithdrawMoneyCommand("ACC-001",
                new BigDecimal("100000"), "USER-42", null))
            .expectException(InsufficientBalanceException.class);
    }

    @Test
    void 동결_계좌_출금_불가() {
        fixture
            .given(
                new AccountCreated("ACC-001", "USER-42", new BigDecimal("500000"), Instant.now()),
                new AccountFrozen("ACC-001", "사기 의심", "ADMIN-01", Instant.now())
            )
            .when(new WithdrawMoneyCommand("ACC-001",
                new BigDecimal("100000"), "USER-42", null))
            .expectException(AccountNotActiveException.class);
    }
}
```

---

## 📊 패턴 비교

```
@CommandHandler 위치별 비교:

생성자 @CommandHandler:
  → 새 Aggregate 인스턴스 생성
  → @AggregateIdentifier 필드 초기화 (apply() 통해서)
  → 처음으로 apply() 호출

메서드 @CommandHandler:
  → 기존 Aggregate 로드 후 호출
  → 불변식 검증 + apply() 호출
  → @AggregateIdentifier는 이미 설정됨

Static Factory @CommandHandler (대안):
  → 생성자 대신 static 메서드로 구현
  → new AccountAggregate(cmd) 대신 AccountAggregate.open(cmd)
  → 도메인 언어 표현에 유리
```

---

## ⚖️ 트레이드오프

```
@EventSourcingHandler에서 절대값 사용:
  balance = event.balanceAfter()  // 절대값
  vs
  balance = balance.subtract(event.amount())  // 누적 계산

  절대값 권장:
    멱등 처리 가능 (동일 이벤트 두 번 적용 시 같은 결과)
    이벤트 페이로드에 사후 상태 포함 → 명확한 감사 로그
    누적 계산: 이벤트 순서 역전 시 잘못된 상태 가능
```

---

## 📌 핵심 정리

```
Axon Aggregate 핵심 규칙:

@CommandHandler:
  불변식 검증 (검증 실패 → 예외)
  apply() 호출 (상태 변경 X, 이벤트 생성 O)

AggregateLifecycle.apply():
  이벤트 큐에 추가
  즉시 @EventSourcingHandler 호출 (상태 업데이트)
  커밋 시 이벤트 스토어 저장 + EventBus 발행

@EventSourcingHandler:
  순수 상태 업데이트만 (balance = event.balanceAfter())
  검증 없음 (이미 검증된 이벤트)
  부수효과 없음 (재구성 시마다 호출됨)

테스트:
  AggregateTestFixture.given(...).when(...).expectEvents(...)
  DB, Kafka 없이 단위 테스트
```

---

## 🤔 생각해볼 문제

**Q1.** 하나의 Command 처리에서 `apply()`를 3번 호출했다. 이 3개 이벤트는 언제 이벤트 스토어에 저장되는가?

<details>
<summary>해설 보기</summary>

3개 이벤트 모두 Unit of Work 커밋 시 한 번에 이벤트 스토어에 저장됩니다.

`apply()` 호출 순서마다 다음이 발생합니다. 첫째, 해당 `@EventSourcingHandler`가 즉시 호출되어 Aggregate 상태가 업데이트됩니다. 둘째, 이벤트가 현재 Unit of Work의 이벤트 큐에 추가됩니다.

CommandHandler가 성공적으로 완료되고 Unit of Work이 커밋되는 시점에 큐의 3개 이벤트가 모두 이벤트 스토어에 저장됩니다. 저장 후 EventBus를 통해 `@EventHandler`(Projection, Saga)에게 순서대로 발행됩니다.

중간에 예외가 발생하면 Unit of Work이 롤백되어 이벤트 스토어에 아무것도 저장되지 않습니다. `@EventSourcingHandler`로 변경된 Aggregate 상태도 Unit of Work 롤백으로 되돌아갑니다.

</details>

---

**Q2.** 잔고 필드를 `balance = event.balanceAfter()`처럼 절대값으로 업데이트하는 방식과 `balance = balance.subtract(event.amount())`처럼 누적 계산하는 방식이 있다. 실제로 차이가 생기는 시나리오는 무엇인가?

<details>
<summary>해설 보기</summary>

두 방식이 차이를 만드는 시나리오가 두 가지 있습니다.

첫째, 중복 이벤트 처리 시 차이가 납니다. 동일한 `MoneyWithdrawn` 이벤트가 두 번 재생되는 경우입니다. 절대값 방식은 `balance = 400000` (두 번 모두 동일). 누적 계산 방식은 `balance = 500000 - 100000 = 400000`, 두 번째 `= 400000 - 100000 = 300000` (잘못됨). 잘못된 잔고가 계산됩니다.

둘째, 이벤트 페이로드의 신뢰성 문제입니다. 누적 계산 방식은 이전 이벤트들이 올바르게 적용됐다는 가정을 합니다. 만약 이벤트 스트림에 버그로 인한 이벤트가 하나 잘못 저장됐다면, 이후 모든 잔고 계산이 틀려집니다. 절대값 방식은 `balanceAfter`가 이미 이벤트 발행 시점에 계산됐으므로, 해당 이벤트만 올바르면 그 시점의 잔고가 정확합니다.

실무에서는 절대값 방식을 권장합니다. 이벤트 페이로드에 `balanceAfter`를 포함해 명시적으로 사후 상태를 기록하고, `@EventSourcingHandler`는 이 값을 그대로 사용합니다.

</details>

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[이전: Axon Framework 아키텍처 ⬅️](./01-axon-architecture.md)** | **[다음: 프로젝션 구현 ➡️](./03-projection-implementation.md)**

</div>
