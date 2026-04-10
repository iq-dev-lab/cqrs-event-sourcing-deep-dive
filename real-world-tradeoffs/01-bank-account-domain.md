# 은행 계좌 도메인 구현

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 은행 계좌 도메인의 핵심 불변식과 Command/Event 설계는 어떻게 되는가?
- 계좌 요약, 거래 내역, 월별 통계 세 가지 읽기 모델이 어떻게 분리되는가?
- Event Sourcing으로 완전한 감사 로그와 시간 여행 기능을 어떻게 구현하는가?
- 이체처럼 두 Aggregate를 조율하는 처리는 어떻게 설계하는가?
- 금융 도메인에서 Eventual Consistency를 어떻게 수용하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

은행 계좌는 CQRS + Event Sourcing이 가장 자연스럽게 맞는 도메인이다. 감사 로그가 법적 요구사항이고, 시간 여행이 비즈니스 요구사항이며, 읽기와 쓰기 비율이 극단적으로 차이난다. 이 도메인을 완전히 구현해보면 CQRS + ES가 실제로 어떻게 작동하는지 명확히 이해할 수 있다.

---

## 😱 흔한 실수 (Before — 은행 계좌를 CRUD로 구현)

```
상태 저장 방식의 한계:

  accounts 테이블: account_id, balance, status, updated_at

  입금:
    UPDATE accounts SET balance = balance + 100000

  이후 발생하는 문제:
    감사팀: "ACC-001 지난 3개월 거래 내역 제출하세요"
    → balance 필드만 있음 → 이전 거래 기록 없음

    고객 분쟁: "2월 15일 오후 2시 잔고가 얼마였나요?"
    → 현재 상태만 있음 → 과거 시점 잔고 불가

    규제: "이 이체를 승인한 직원이 누구인가요?"
    → 승인자 정보 없음

  → 별도 transaction_history 테이블 추가 필요
  → 하지만 원인과 맥락은 여전히 불완전
```

---

## ✨ 올바른 접근 (After — Event Sourcing 기반 완전한 구현)

```
도메인 설계:

Account Aggregate 불변식:
  balance >= 0
  status = FROZEN → 출금, 이체 불가 (입금은 허용)
  status = CLOSED → 모든 거래 불가
  amount <= dailyWithdrawLimit

Command → Event 매핑:
  OpenAccount       → AccountOpened
  DepositMoney      → MoneyDeposited
  WithdrawMoney     → MoneyWithdrawn
  InitiateTransfer  → TransferInitiated
  ReceiveTransfer   → MoneyDeposited (입금 계좌)
  FreezeAccount     → AccountFrozen
  CloseAccount      → AccountClosed

읽기 모델 3개:
  account_summary:    잔고, 상태 (실시간)
  transaction_history: 거래 내역 (감사, 조회)
  monthly_stats:       월별 통계 (리포트)
```

---

## 🔬 내부 동작 원리

### 1. 이벤트 페이로드 설계 — 감사 목적

```
MoneyWithdrawn 이벤트 설계:

  {
    "accountId":    "ACC-001",
    "amount":       100000,
    "balanceAfter": 400000,      ← 절대값 (멱등 처리용)
    "requestedBy":  "USER-42",   ← 누가 요청
    "approvedBy":   "TELLER-07", ← 누가 승인 (고액)
    "description":  "ATM 출금",
    "transferId":   null,        ← 이체 출금인 경우 있음
    "occurredAt":   "...",       ← 언제
    "correlationId": "...",      ← 분산 추적
    "causationId":  "..."        ← 원인 Command/Event ID
  }

감사 기능:
  "누가 언제 얼마를 출금했는가" → MoneyWithdrawn 이벤트 하나로 완결
  별도 audit_log 테이블 불필요

이벤트 → 거래 내역 읽기 모델:
  MoneyDeposited { amount, balanceAfter, depositedBy, occurredAt }
  → transaction_history INSERT:
     { type: "DEPOSIT", amount, balance_after, performed_by, occurred_at }
```

### 2. 세 가지 읽기 모델 분리

```
읽기 모델 1 — account_summary:
  목적: 잔고 화면, 계좌 목록
  스키마: account_id, owner_name, balance, status,
          last_transaction_at, created_at
  인덱스: (account_id), (owner_id, created_at)
  캐시: Redis (입출금마다 갱신)

읽기 모델 2 — transaction_history:
  목적: 거래 내역 조회 (감사, 고객 조회)
  스키마: transaction_id, account_id, type, amount,
          balance_after, description, performed_by,
          counterpart_id, occurred_at
  인덱스: (account_id, occurred_at DESC)
  조회: 날짜 범위 + 페이지네이션

읽기 모델 3 — monthly_stats:
  목적: 월별 통계 리포트
  스키마: account_id, year_month, deposit_count,
          deposit_total, withdraw_count, withdraw_total
  인덱스: (account_id, year_month)
  업데이트: 집계 누적 (event_id ON CONFLICT DO NOTHING으로 멱등)
```

### 3. 시간 여행 구현

```
"ACC-001의 2024-01-15 오후 3시 잔고는?"

구현 흐름:
  1. occurred_at <= '2024-01-15T15:00:00Z' 인 이벤트 로드
  2. Account.reconstitute(events) → 해당 시점 상태 재현
  3. account.getBalance() 반환

SQL:
  SELECT * FROM event_store
  WHERE stream_id = 'account-ACC-001'
    AND occurred_at <= '2024-01-15T15:00:00Z'
  ORDER BY version ASC;

활용 사례:
  고객 분쟁: "그 시점 잔고가 얼마였는지 확인"
  규제 보고: "분기 말 잔고"
  감사: "이 거래 직전 잔고는?"
```

### 4. 이체 — Saga로 두 Aggregate 조율

```
이체 흐름 (ACC-001 → ACC-002, 100,000원):

Step 1: InitiateTransferCommand(ACC-001)
  → TransferInitiated { transferId: "TRF-001",
                         from: ACC-001, to: ACC-002, amount: 100000,
                         balanceAfter: 400000 }
  → ACC-001 잔고 차감 (400,000)

Step 2: Saga가 TransferInitiated 수신
  → ReceiveTransferCommand(ACC-002, "TRF-001", 100000) 발행

Step 3: ReceiveTransferCommand(ACC-002)
  → MoneyDeposited { accountId: ACC-002, amount: 100000,
                     balanceAfter: 600000, transferId: "TRF-001" }
  → ACC-002 잔고 증가 (600,000)

Step 4: Saga가 MoneyDeposited(transferId=TRF-001) 수신
  → CompleteTransferCommand(ACC-001, "TRF-001") 발행

Step 5: TransferCompleted { transferId: "TRF-001" }

실패 보상 (Step 3에서 ACC-002가 해지 계좌):
  AccountClosedException → Saga가 감지
  → CancelTransferCommand(ACC-001, "TRF-001") 발행
  → TransferCancelled { transferId, reason }
  → ACC-001 잔고 복구 (500,000)
```

---

## 💻 실전 코드

```java
// ✅ Account Aggregate — 은행 계좌 완전 구현
@Aggregate
public class AccountAggregate {

    @AggregateIdentifier
    private String accountId;
    private BigDecimal balance;
    private AccountStatus status;
    private BigDecimal dailyWithdrawLimit;
    private BigDecimal todayWithdrawn;
    private LocalDate lastWithdrawDate;

    protected AccountAggregate() {}

    @CommandHandler
    public AccountAggregate(OpenAccountCommand cmd) {
        apply(new AccountOpened(
            cmd.accountId(), cmd.ownerId(), cmd.ownerName(),
            cmd.dailyLimit(), Instant.now()
        ));
    }

    @CommandHandler
    public void handle(DepositMoneyCommand cmd) {
        if (status == AccountStatus.CLOSED)
            throw new AccountClosedException(accountId);
        apply(new MoneyDeposited(
            accountId, cmd.amount(), balance.add(cmd.amount()),
            cmd.depositedBy(), cmd.description(), Instant.now()
        ));
    }

    @CommandHandler
    public void handle(WithdrawMoneyCommand cmd) {
        if (status == AccountStatus.FROZEN) throw new AccountFrozenException(accountId);
        if (status == AccountStatus.CLOSED) throw new AccountClosedException(accountId);
        if (balance.compareTo(cmd.amount()) < 0)
            throw new InsufficientBalanceException(accountId, cmd.amount(), balance);

        LocalDate today = LocalDate.now(ZoneId.systemDefault());
        BigDecimal todayTotal = today.equals(lastWithdrawDate)
            ? todayWithdrawn : BigDecimal.ZERO;
        if (todayTotal.add(cmd.amount()).compareTo(dailyWithdrawLimit) > 0)
            throw new DailyLimitExceededException(accountId);

        apply(new MoneyWithdrawn(
            accountId, cmd.amount(), balance.subtract(cmd.amount()),
            cmd.requestedBy(), cmd.description(), Instant.now()
        ));
    }

    @CommandHandler
    public String handle(InitiateTransferCommand cmd) {
        if (status != AccountStatus.ACTIVE) throw new AccountNotActiveException(accountId, status);
        if (balance.compareTo(cmd.amount()) < 0)
            throw new InsufficientBalanceException(accountId, cmd.amount(), balance);

        String transferId = "TRF-" + UUID.randomUUID().toString().substring(0, 8);
        apply(new TransferInitiated(
            transferId, accountId, cmd.toAccountId(),
            cmd.amount(), balance.subtract(cmd.amount()),
            cmd.requestedBy(), Instant.now()
        ));
        return transferId;
    }

    // @EventSourcingHandler — 순수 상태 업데이트만
    @EventSourcingHandler public void on(AccountOpened e) {
        this.accountId = e.accountId(); this.balance = BigDecimal.ZERO;
        this.status = AccountStatus.ACTIVE; this.dailyWithdrawLimit = e.dailyLimit();
        this.todayWithdrawn = BigDecimal.ZERO;
    }
    @EventSourcingHandler public void on(MoneyDeposited e) { this.balance = e.balanceAfter(); }
    @EventSourcingHandler public void on(MoneyWithdrawn e) {
        this.balance = e.balanceAfter();
        this.todayWithdrawn = todayWithdrawn.add(e.amount());
        this.lastWithdrawDate = e.occurredAt().atZone(ZoneId.systemDefault()).toLocalDate();
    }
    @EventSourcingHandler public void on(TransferInitiated e) {
        this.balance = e.balanceAfter();
        this.todayWithdrawn = todayWithdrawn.add(e.amount());
    }
    @EventSourcingHandler public void on(AccountFrozen e) { this.status = AccountStatus.FROZEN; }
    @EventSourcingHandler public void on(AccountClosed e) { this.status = AccountStatus.CLOSED; }
}

// ✅ 시간 여행 Query
@QueryHandler
@Transactional(readOnly = true)
public BalanceAtPointInTimeView handle(GetBalanceAtQuery query) {
    List<StoredEvent> events = eventStore.loadEventsUntil(
        "account-" + query.accountId(), query.pointInTime());
    if (events.isEmpty()) throw new AccountNotFoundException(query.accountId());

    Account account = Account.reconstitute(
        events.stream().map(serializer::deserialize).collect(toList()));

    return new BalanceAtPointInTimeView(
        query.accountId(), account.getBalance(), query.pointInTime());
}

// ✅ 이체 Saga
@Saga
public class MoneyTransferSaga {

    @Autowired private transient CommandGateway commandGateway;
    private String transferId;
    private String fromAccountId;

    @StartSaga
    @SagaEventHandler(associationProperty = "transferId")
    public void on(TransferInitiated event) {
        this.transferId = event.transferId();
        this.fromAccountId = event.fromAccountId();
        commandGateway.send(new ReceiveTransferCommand(
            event.toAccountId(), event.transferId(), event.amount()));
    }

    @SagaEventHandler(associationProperty = "transferId")
    public void on(TransferReceived event) {
        commandGateway.send(new CompleteTransferCommand(fromAccountId, transferId));
        SagaLifecycle.end();
    }

    @SagaEventHandler(associationProperty = "transferId")
    public void on(TransferFailed event) {
        commandGateway.send(new CancelTransferCommand(fromAccountId, transferId, event.reason()));
        SagaLifecycle.end();
    }
}
```

---

## 📊 패턴 비교

```
은행 계좌 읽기 모델별 최적화:

┌───────────────────┬────────────────┬──────────────┬──────────────┐
│ 읽기 모델         │ 저장소         │ 인덱스       │ 캐시         │
├───────────────────┼────────────────┼──────────────┼──────────────┤
│ account_summary   │ PostgreSQL     │ account_id   │ Redis        │
├───────────────────┼────────────────┼──────────────┼──────────────┤
│ transaction_history│ PostgreSQL    │ (id, date)   │ 불필요       │
├───────────────────┼────────────────┼──────────────┼──────────────┤
│ monthly_stats     │ PostgreSQL     │ (id, month)  │ 완성된 월    │
└───────────────────┴────────────────┴──────────────┴──────────────┘
```

---

## ⚖️ 트레이드오프

```
이체 Saga 복잡도:
  장점: 계좌별 독립 트랜잭션 → 분산 확장 가능
  단점: 최종 일관성 (이체 중간 상태 존재)
        보상 트랜잭션 로직 복잡

동일 트랜잭션 이체:
  장점: 단순, 강한 일관성
  단점: 두 계좌 잠금 가능 → 데드락 위험
        MSA 환경에서 불가
```

---

## 📌 핵심 정리

```
은행 계좌 도메인 핵심:

ES의 비즈니스 가치:
  완전한 감사 로그 (규제 대응) — 내장
  시간 여행 (분쟁 해결) — 이벤트 필터로 구현
  거래 추적 (transferId로 이체 연결)

불변식 설계:
  balance >= 0, 상태별 허용 거래, 일일 한도

읽기 모델 분리:
  account_summary: Redis 캐시 (< 1ms)
  transaction_history: 날짜 인덱스 최적화
  monthly_stats: 사전 집계

이체 처리:
  단일 서비스 → 동일 트랜잭션 (단순)
  분산 서비스 → Saga (확장성)
```

---

## 🤔 생각해볼 문제

**Q1.** 월말 자정을 기준으로 일일 출금 한도가 리셋돼야 한다. Event Sourcing에서 어떻게 구현하는가?

<details>
<summary>해설 보기</summary>

`@EventSourcingHandler`에서 날짜 비교로 자동 처리합니다.

`on(MoneyWithdrawn event)` 내에서 이전 출금 날짜(`lastWithdrawDate`)와 현재 이벤트 날짜를 비교합니다. 날짜가 달라졌으면 `todayWithdrawn`을 0으로 리셋합니다. 이벤트 리플레이 시에도 이벤트의 `occurredAt`을 기준으로 날짜를 비교하므로 정확히 동작합니다.

핵심은 `@CommandHandler`에서도 날짜 리셋을 고려해야 한다는 점입니다. 출금 Command 처리 시 현재 날짜를 기준으로 `todayWithdrawn`을 계산해야 하는데, 이는 `@EventSourcingHandler`에서 계산된 `lastWithdrawDate`를 활용하면 됩니다.

명시적인 `DailyLimitReset` 이벤트를 발행하는 방법도 있습니다. 스케줄러가 자정에 모든 계좌에 리셋 이벤트를 발행합니다. 계좌 수가 많으면 부하가 크지만, 감사 로그에 리셋 기록이 남는 장점이 있습니다.

</details>

---

**Q2.** 금융 규제에서 GDPR처럼 "특정 고객 데이터 삭제" 요청이 오면 Event Sourcing의 Append-Only 원칙과 어떻게 충돌하는가?

<details>
<summary>해설 보기</summary>

이 충돌이 Event Sourcing 도입 시 금융 도메인에서 가장 중요한 법적/설계 문제입니다.

Crypto Shredding으로 해결합니다. 고객 개인정보를 이벤트 페이로드에 직접 저장하지 않습니다. 대신 이벤트에는 `customerId`(식별자)만 저장하고, 실제 이름/이메일/주소는 별도 암호화 저장소에 암호화 키로 보관합니다.

삭제 요청 시 해당 고객의 암호화 키를 삭제합니다. 이벤트 스토어의 이벤트는 남아있지만 개인정보를 복호화할 수 없어 사실상 데이터가 사라진 효과입니다.

이를 위한 이벤트 설계가 필요합니다. `MoneyWithdrawn.requestedBy`에 "USER-42" 대신 암호화된 식별자를 저장합니다. 평문 이름/이메일이 이벤트에 포함되면 나중에 삭제가 불가능합니다. 이벤트 설계 초기부터 개인정보 최소화 원칙을 적용해야 합니다.

</details>

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[이전: Chapter 6 — Axon 없이 직접 구현 ⬅️](../spring-axon-implementation/05-pure-spring-implementation.md)** | **[다음: Event Sourcing 없는 CQRS ➡️](./02-cqrs-without-event-sourcing.md)**

</div>
