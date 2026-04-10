# Command Handler와 Aggregate

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Command Handler는 어디서 시작해서 어디서 끝나는가?
- Application Service와 Command Handler는 무엇이 다른가?
- Aggregate를 로드하고 저장하는 책임은 누가 가지는가?
- Command Handler가 여러 도메인 서비스를 호출해야 할 때 어떻게 설계하는가?
- Event Sourcing에서 Command Handler 흐름은 어떻게 달라지는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

Command Handler는 Command와 Aggregate 사이의 조율자다. 너무 많은 책임을 가지면 Transaction Script가 되고, 너무 적은 책임을 가지면 Aggregate가 인프라를 알아야 한다. Command Handler의 적절한 책임 범위를 이해해야 도메인 로직이 Aggregate 안에 응집되고, 인프라 관심사가 Handler 안에 격리된다.

---

## 😱 흔한 실수 (Before — 잘못 분배된 책임)

```
실수 1: Command Handler가 Transaction Script가 됨

  @CommandHandler
  public void handle(TransferMoneyCommand cmd) {
      // ❌ 불변식 검증 로직이 Handler에 있음
      Account from = accountRepository.findById(cmd.getFromAccountId());
      if (from.getBalance().compareTo(cmd.getAmount()) < 0)
          throw new InsufficientBalanceException();
      if (from.getStatus() == FROZEN)
          throw new AccountFrozenException();

      Account to = accountRepository.findById(cmd.getToAccountId());
      if (to.getStatus() == CLOSED)
          throw new AccountClosedException();

      // 상태 변경도 Handler가 직접
      from.setBalance(from.getBalance().subtract(cmd.getAmount()));
      to.setBalance(to.getBalance().add(cmd.getAmount()));

      accountRepository.save(from);
      accountRepository.save(to);
      // → Aggregate에 비즈니스 로직이 없음. 단순 데이터 컨테이너.
  }

실수 2: Aggregate가 Repository를 알아야 하는 구조

  class Account {
      @Autowired AccountRepository accountRepository; // ❌ 도메인이 인프라 의존

      void transfer(AccountId toId, Money amount) {
          Account target = accountRepository.findById(toId); // 도메인이 DB 조회
          this.balance = this.balance.subtract(amount);
          target.balance = target.balance.add(amount);
          accountRepository.save(target); // 도메인이 저장
      }
  }

실수 3: Handler가 읽기 모델에서 쓰기 판단

  @CommandHandler
  public void handle(ConfirmOrderCommand cmd) {
      // ❌ 읽기 모델(비정규화)로 재고 확인 — 지연된 데이터일 수 있음
      OrderSummaryView summary = orderSummaryRepository.findById(cmd.getOrderId());
      if (summary.getItemCount() > 0) {  // 읽기 모델로 불변식 검증
          // ... 위험: 읽기 모델이 최신 상태가 아닐 수 있음
      }
  }
```

---

## ✨ 올바른 접근 (After — 명확한 책임 분배)

```
Command Handler의 책임:
  1. Command에서 식별자 추출
  2. Repository에서 Aggregate 로드 (인프라 접근)
  3. Aggregate 도메인 메서드 호출 (불변식 검증 + 상태 변경)
  4. Repository에 Aggregate 저장 (인프라 접근)
  5. (필요시) 트랜잭션 관리

Aggregate의 책임:
  1. 불변식 검증 (상태 전이 가능한가?)
  2. 상태 변경
  3. 도메인 이벤트 등록 (발행은 인프라가 담당)

Repository의 책임:
  1. Aggregate 로드 (DB or Event Store → Aggregate)
  2. Aggregate 저장 (Aggregate → DB or Event Store)

경계:
  Handler → 인프라(Repository) 알아도 됨
  Aggregate → 인프라 몰라야 함 (순수 도메인 로직만)
```

---

## 🔬 내부 동작 원리

### 1. Command Handler 처리 흐름 — 단계별 분석

```
완전한 흐름:

POST /accounts/ACC-001/withdraw
  Body: { "amount": 100000, "requestedBy": "user-42" }

  ↓ Controller
  commandBus.send(new WithdrawMoneyCommand("ACC-001", 100000, "user-42", ...))

  ↓ Command Bus (라우팅)
  AccountCommandHandler.handle(WithdrawMoneyCommand) 호출

  ↓ Command Handler — 단계 1: Aggregate 로드
  Account account = accountRepository.findById(new AccountId("ACC-001"))
    → DB에서 Account 상태 로드 (또는 이벤트 스토어에서 이벤트 리플레이)
    → 조회 실패 시: AccountNotFoundException throw

  ↓ Command Handler — 단계 2: 도메인 메서드 호출
  account.withdraw(Money.of(100000), "user-42")
    → Account 내부:
       if (balance < amount) throw InsufficientBalanceException  [불변식 1]
       if (status == FROZEN) throw AccountFrozenException        [불변식 2]
       this.balance = balance.subtract(amount)                   [상태 변경]
       registerEvent(new MoneyWithdrawn(id, amount, balance))    [이벤트 등록]

  ↓ Command Handler — 단계 3: 저장
  accountRepository.save(account)
    → Account 상태 DB 저장
    → Account에 등록된 Domain Event 발행 (Spring @TransactionalEventListener)

  ↓ Response
  HTTP 202 Accepted (또는 204 No Content)

  ↓ (비동기) Projection
  MoneyWithdrawn 이벤트 → Kafka → AccountSummaryProjection
  → account_summary.balance 업데이트
```

### 2. Application Service vs Command Handler

```
두 개념의 관계:

전통적 DDD:
  Application Service = 유스케이스 조율자
    Repository 접근
    도메인 객체 로드
    도메인 메서드 호출
    트랜잭션 관리

CQRS:
  Command Handler ≈ Application Service의 역할
  하지만 Command 하나당 Handler 하나로 더 세분화

차이:
  Application Service:
    class OrderApplicationService {
        void confirmOrder(String orderId) { ... }
        void cancelOrder(String orderId, String reason) { ... }
        void shipOrder(String orderId, String trackingNo) { ... }
        // 여러 유스케이스를 하나의 Service에
    }

  Command Handler:
    class ConfirmOrderHandler implements CommandHandler<ConfirmOrderCommand> { ... }
    class CancelOrderHandler  implements CommandHandler<CancelOrderCommand>  { ... }
    class ShipOrderHandler    implements CommandHandler<ShipOrderCommand>    { ... }
    // 유스케이스 하나당 Handler 하나 → 단일 책임

  어느 방식이든 핵심은 같음:
    불변식은 Aggregate 안에
    인프라 접근(Repository)은 Handler/Service 안에
    도메인 로직과 인프라 관심사 분리
```

### 3. 도메인 서비스(Domain Service)가 필요한 경우

```
Command Handler가 단일 Aggregate 이상의 협력이 필요할 때:

시나리오: 이체 처리 — 출금 계좌와 입금 계좌

Option 1: Command Handler가 두 Aggregate를 직접 조율
  @CommandHandler
  @Transactional
  public void handle(TransferMoneyCommand cmd) {
      Account from = accountRepository.findById(cmd.fromAccountId());
      Account to   = accountRepository.findById(cmd.toAccountId());

      from.transferOut(cmd.amount(), cmd.toAccountId()); // 출금 + 이벤트 등록
      to.receiveTransfer(cmd.amount(), cmd.fromAccountId()); // 입금 + 이벤트 등록

      accountRepository.save(from);
      accountRepository.save(to);
      // 동일 트랜잭션 — 두 계좌 모두 성공 or 모두 실패
  }
  // 주의: 두 Aggregate를 같은 트랜잭션으로 처리 → Aggregate 경계 위반 가능성

Option 2: Domain Service로 이체 로직 캡슐화
  // 이체 로직이 두 Account에 걸쳐 있는 불변식을 캡슐화
  @DomainService
  public class MoneyTransferService {
      public void transfer(Account from, Account to, Money amount) {
          from.validateTransferOut(amount);  // 출금 가능 여부
          to.validateTransferIn(amount);     // 입금 가능 여부 (계좌 상태 등)

          from.deduct(amount);
          to.credit(amount);
          // 두 Aggregate 간 불변식이 Domain Service에 응집
      }
  }

  @CommandHandler
  @Transactional
  public void handle(TransferMoneyCommand cmd) {
      Account from = accountRepository.findById(cmd.fromAccountId());
      Account to   = accountRepository.findById(cmd.toAccountId());

      moneyTransferService.transfer(from, to, cmd.amount());

      accountRepository.save(from);
      accountRepository.save(to);
  }

Option 3: Saga 패턴 (Aggregate 경계를 엄격히 유지)
  Step 1: WithdrawFromAccountCommand → from 계좌 출금
  Step 2: MoneyWithdrawn 이벤트 수신 → DepositToAccountCommand → to 계좌 입금
  Step 3: 입금 실패 시 → CompensateWithdrawalCommand → 출금 취소
  → 각 단계가 별도 트랜잭션, 최종 일관성
```

### 4. Event Sourcing에서의 Command Handler

```
상태 저장 CQRS의 Command Handler:

  account = accountRepository.findById(id)  // DB에서 현재 상태 로드
  account.withdraw(amount)                   // 상태 변경
  accountRepository.save(account)            // 변경된 상태 저장

Event Sourcing의 Command Handler:

  이벤트 스토어에서 Aggregate 재구성:
  account = accountRepository.findById(id)
    → 내부: event_store에서 stream "account-ACC-001" 이벤트 로드
    → apply(AccountOpened)    → balance = 0, status = ACTIVE
    → apply(MoneyDeposited)   → balance = 500000
    → apply(MoneyDeposited)   → balance = 800000
    → apply(MoneyWithdrawn)   → balance = 700000
    → 현재 상태: balance = 700000, status = ACTIVE

  account.withdraw(100000)
    → 불변식 검증: 700000 >= 100000 ✓
    → 이벤트 생성: MoneyWithdrawn { amount: 100000, balanceAfter: 600000 }
    → Aggregate 내부 pendingEvents 리스트에 추가

  accountRepository.save(account)
    → 내부: event_store에 MoneyWithdrawn 이벤트 INSERT (Append Only)
    → stream "account-ACC-001", version 4
    → Kafka에 MoneyWithdrawn 발행

핵심 차이:
  상태 저장: "현재 잔고가 600000이다" → UPDATE account SET balance = 600000
  이벤트 저장: "100000원 출금이 발생했다" → INSERT INTO event_store (..., MoneyWithdrawn)
  → 이벤트는 과거 사실의 기록, 현재 상태는 이벤트에서 계산
```

---

## 💻 실전 코드

```java
// ==============================
// 완전한 Command Handler 구현
// ==============================

// ✅ 순수 Command Handler — Aggregate에게 모든 비즈니스 로직 위임
@Component
public class AccountCommandHandler {

    private final AccountRepository accountRepository;
    private final MoneyTransferService moneyTransferService;

    // ── 단일 Aggregate 처리 ──────────────────────────────────

    @CommandHandler
    @Transactional
    public void handle(DepositMoneyCommand command) {
        Account account = loadAccount(command.accountId());

        account.deposit(
            Money.of(command.amount()),
            command.depositedBy()
        );
        // Account.deposit() 내부:
        //   validateStatus() — 해지된 계좌 불가
        //   this.balance = balance.add(amount)
        //   registerEvent(new MoneyDeposited(...))

        accountRepository.save(account);
    }

    @CommandHandler
    @Transactional
    public void handle(WithdrawMoneyCommand command) {
        Account account = loadAccount(command.accountId());

        account.withdraw(
            Money.of(command.amount()),
            command.requestedBy()
        );
        // Account.withdraw() 내부:
        //   if (balance < amount) throw InsufficientBalanceException
        //   if (status == FROZEN) throw AccountFrozenException
        //   this.balance = balance.subtract(amount)
        //   registerEvent(new MoneyWithdrawn(...))

        accountRepository.save(account);
    }

    // ── 두 Aggregate 처리 (Domain Service 활용) ────────────────

    @CommandHandler
    @Transactional
    public void handle(TransferMoneyCommand command) {
        Account fromAccount = loadAccount(command.fromAccountId());
        Account toAccount   = loadAccount(command.toAccountId());

        // 이체 불변식이 두 Aggregate에 걸쳐 있음 → Domain Service
        moneyTransferService.transfer(
            fromAccount,
            toAccount,
            Money.of(command.amount()),
            command.requestedBy()
        );

        // 두 Aggregate 저장
        accountRepository.save(fromAccount);
        accountRepository.save(toAccount);
    }

    // ── 공통 로직 ────────────────────────────────────────────

    private Account loadAccount(String accountId) {
        return accountRepository.findById(new AccountId(accountId))
            .orElseThrow(() -> new AccountNotFoundException(accountId));
    }
}

// ✅ Account Aggregate — 불변식 보호에 집중
public class Account {

    private final AccountId id;
    private Money balance;
    private AccountStatus status;
    private final OwnerId ownerId;

    // 비즈니스 로직은 Aggregate 안에만
    public void withdraw(Money amount, String requestedBy) {
        validateStatus(); // 계좌 상태 검증
        if (balance.isLessThan(amount))
            throw new InsufficientBalanceException(id, amount, balance);

        this.balance = balance.subtract(amount);
        registerEvent(new MoneyWithdrawn(
            id, amount, balance, requestedBy, Instant.now()
        ));
    }

    public void deposit(Money amount, String depositedBy) {
        validateStatus();
        validateAmount(amount);

        this.balance = balance.add(amount);
        registerEvent(new MoneyDeposited(
            id, amount, balance, depositedBy, Instant.now()
        ));
    }

    private void validateStatus() {
        if (status == AccountStatus.CLOSED)
            throw new AccountClosedException(id);
        if (status == AccountStatus.FROZEN)
            throw new AccountFrozenException(id);
    }

    // Domain Event 등록 — 저장/발행은 인프라가 담당
    private final List<DomainEvent> pendingEvents = new ArrayList<>();

    protected void registerEvent(DomainEvent event) {
        pendingEvents.add(event);
    }

    public List<DomainEvent> pullPendingEvents() {
        List<DomainEvent> events = List.copyOf(pendingEvents);
        pendingEvents.clear();
        return events;
    }
}

// ✅ Event Sourcing 방식의 Command Handler (Axon 기반)
@Aggregate
public class AccountAggregate {

    @AggregateIdentifier
    private AccountId accountId;
    private Money balance;
    private AccountStatus status;

    // Command Handler — Aggregate 안에서 Command 처리
    @CommandHandler
    public void handle(WithdrawMoneyCommand command) {
        // 불변식 검증 후 이벤트 apply()
        if (balance.isLessThan(Money.of(command.amount())))
            throw new InsufficientBalanceException(accountId, command.amount(), balance);
        if (status == AccountStatus.FROZEN)
            throw new AccountFrozenException(accountId);

        apply(new MoneyWithdrawn(
            accountId.value(),
            command.amount(),
            balance.subtract(Money.of(command.amount())).amount(),
            command.requestedBy(),
            Instant.now()
        ));
        // apply(): 이벤트 스토어에 저장 + @EventSourcingHandler 호출
    }

    // Event Sourcing Handler — 이벤트로 상태 변경
    @EventSourcingHandler
    public void on(MoneyWithdrawn event) {
        this.balance = Money.of(event.getBalanceAfter());
        // 이벤트 리플레이 시에도 동일하게 호출 → 상태 재현
    }
}
```

---

## 📊 패턴 비교

```
Command Handler 책임 분배 비교:

❌ Transaction Script (Handler가 모든 로직):
  Handler: 불변식 검증 + 상태 변경 + 저장
  Aggregate: 데이터 컨테이너 역할만
  문제: 비즈니스 로직이 Handler에 분산, 테스트 어려움

✅ Domain Model (Aggregate에 비즈니스 로직):
  Handler: Repository 접근 + Aggregate 호출 + 저장
  Aggregate: 불변식 검증 + 상태 변경 + 이벤트 등록
  장점: 비즈니스 로직 응집, 단위 테스트 용이

상태 저장 vs Event Sourcing Command Handler:

  상태 저장:
    Load: SELECT * FROM account WHERE id = ?
    Validate + Mutate: account.withdraw()
    Save: UPDATE account SET balance = ?

  Event Sourcing:
    Load: SELECT * FROM event_store WHERE stream_id = 'account-ACC-001' ORDER BY version
    Replay: apply(event1), apply(event2), ...
    Validate + Emit: account.withdraw() → MoneyWithdrawn 이벤트 생성
    Save: INSERT INTO event_store (MoneyWithdrawn, version = N+1)
```

---

## ⚖️ 트레이드오프

```
Handler 단일 책임의 장단점:

장점:
  각 Handler가 하나의 Command만 처리 → 단위 테스트 간결
  Command 추가 시 기존 Handler 수정 없음 (OCP)
  Handler별 다른 트랜잭션 설정 가능

단점:
  Command 수만큼 Handler 클래스 증가
  공통 로직(loadAccount 등) 중복 → 공통 메서드나 BaseHandler로 해결

Domain Service 사용 시기:
  여러 Aggregate에 걸친 불변식 → Domain Service
  단일 Aggregate 불변식 → Aggregate 메서드
  인프라 의존 로직 → Command Handler
```

---

## 📌 핵심 정리

```
Command Handler 책임 경계:

Handler 책임:
  Repository로 Aggregate 로드
  Aggregate 도메인 메서드 호출 (불변식 위임)
  Repository로 Aggregate 저장
  트랜잭션 경계 관리

Aggregate 책임:
  불변식 검증 (Repository 없이)
  상태 변경
  Domain Event 등록

핵심 원칙:
  비즈니스 로직은 Aggregate 안에 — Handler는 조율자
  Aggregate는 Repository를 모름 — 인프라 독립
  읽기 모델로 쓰기 결정 금지 — 최신성 보장 불가

Event Sourcing 차이:
  상태 저장: Aggregate의 현재 상태를 DB에 저장
  이벤트 저장: Aggregate가 발행한 이벤트를 이벤트 스토어에 Append
  → 현재 상태는 이벤트 리플레이로 재현
```

---

## 🤔 생각해볼 문제

**Q1.** Command Handler가 `@Transactional`을 선언하는데, Aggregate 저장과 Kafka 이벤트 발행을 같은 트랜잭션에서 처리할 수 있는가?

<details>
<summary>해설 보기</summary>

직접 같은 트랜잭션에서 처리할 수 없습니다. 데이터베이스 트랜잭션과 Kafka 발행은 서로 다른 리소스이므로 XA 트랜잭션(2PC)을 쓰지 않는 한 원자성 보장이 안 됩니다.

실무에서 사용하는 패턴이 두 가지입니다. 첫째, Spring의 `@TransactionalEventListener`를 활용합니다. 트랜잭션 커밋 이후 Domain Event를 Kafka로 발행합니다. DB 저장이 완료된 후 발행하므로, 저장은 됐지만 발행 실패하는 경우가 생길 수 있습니다.

둘째, Transactional Outbox 패턴을 활용합니다. 동일 트랜잭션 내에서 이벤트를 outbox 테이블에 함께 저장합니다. 별도 프로세스(Debezium 또는 Polling)가 outbox 테이블을 읽어 Kafka에 발행합니다. DB 저장과 outbox 저장이 같은 트랜잭션이므로 원자성이 보장됩니다.

Event Sourcing에서는 이벤트 스토어 자체가 Kafka 발행의 소스 역할을 합니다. 이벤트 스토어에 저장된 이벤트를 CDC 또는 EventStoreDB의 내장 구독 기능으로 Kafka에 발행합니다.

</details>

---

**Q2.** Command Handler 단위 테스트에서 어떤 부분을 테스트하고, 어떤 부분을 Aggregate 단위 테스트에서 테스트하는가?

<details>
<summary>해설 보기</summary>

책임이 명확히 분리되어 있으므로 테스트 범위도 명확합니다.

Aggregate 단위 테스트에서 검증할 것들입니다. Repository나 외부 의존성 없이 순수 도메인 로직을 검증합니다. 불변식 검증(잔고 부족 시 예외 발생), 상태 전이(확인 후 상태가 CONFIRMED), 이벤트 등록(출금 후 MoneyWithdrawn 이벤트 등록) 등을 검증합니다. Mock이 필요 없어 빠르고 안정적입니다.

Command Handler 단위 테스트에서 검증할 것들입니다. Repository Mock을 이용해 올바른 Aggregate를 로드하는지, 올바른 도메인 메서드를 호출하는지, 저장이 호출되는지를 검증합니다. Aggregate 존재하지 않을 때 예외 처리도 검증합니다.

Handler 통합 테스트에서 검증할 것들입니다. 실제 DB와 연동하여 트랜잭션 경계, 이벤트 발행, 읽기 모델 업데이트까지 엔드투엔드 흐름을 검증합니다.

이 분리 덕분에 대부분의 비즈니스 로직 검증이 빠른 Aggregate 단위 테스트에서 이루어집니다.

</details>

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[이전: Command 객체 설계 ⬅️](./02-command-object-design.md)** | **[다음: Command Bus 패턴 ➡️](./04-command-bus.md)**

</div>
