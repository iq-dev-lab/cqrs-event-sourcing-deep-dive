# Optimistic Locking in CQRS

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 낙관적 잠금이 비관적 잠금보다 CQRS에 더 적합한 이유는 무엇인가?
- Version 필드 기반 낙관적 잠금은 내부적으로 어떻게 동작하는가?
- 동시 Command 처리에서 충돌이 발생했을 때 재시도 전략은 어떻게 설계하는가?
- Event Sourcing에서 낙관적 잠금은 어떻게 구현되는가?
- 낙관적 잠금 충돌을 클라이언트에게 어떻게 전달해야 하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

CQRS에서 쓰기 모델은 불변식을 보호해야 한다. 두 사용자가 동시에 같은 계좌에서 출금하려 한다면, 둘 다 잔고가 충분하다고 판단한 후 각자 출금을 진행하면 잔고가 음수가 될 수 있다. 이 문제를 해결하는 두 가지 방법이 비관적 잠금과 낙관적 잠금이다.

CQRS는 쓰기와 읽기를 분리하므로 읽기 성능이 쓰기 잠금의 영향을 받지 않는다. 이 설계에서 낙관적 잠금이 자연스럽게 맞는 이유와, 충돌 시 재시도 전략을 이해하는 것이 이 문서의 목표다.

---

## 😱 흔한 실수 (Before — 비관적 잠금 과다 사용)

```
실수 1: 모든 쓰기에 비관적 잠금 적용

  // ❌ SELECT FOR UPDATE — 행 수준 잠금
  @Transactional
  public void withdraw(String accountId, BigDecimal amount) {
      Account account = accountRepository
          .findByIdWithLock(accountId);  // SELECT ... FOR UPDATE
      // 다른 트랜잭션이 이 계좌에 접근하면 대기

      account.withdraw(Money.of(amount));
      accountRepository.save(account);
  }
  // 잠금 보유 시간 = 트랜잭션 전체 시간
  // 다른 모든 트랜잭션이 대기 → 처리량 감소

실수 2: 낙관적 잠금 충돌을 처리하지 않음

  @Transactional
  public void withdraw(WithdrawMoneyCommand cmd) {
      Account account = accountRepository.findById(cmd.accountId());
      account.withdraw(cmd.amount());
      accountRepository.save(account);
      // ❌ OptimisticLockingFailureException 발생 시 그냥 500 에러
      // 클라이언트에게 "재시도하세요"를 알리지 않음
  }

실수 3: Version을 클라이언트가 제공하지 않음

  // Command에 version이 없음
  public record WithdrawMoneyCommand(String accountId, BigDecimal amount) {}

  // 서버가 Account 로드 시 현재 version을 사용하면
  // 클라이언트가 "어느 시점의 Account"를 기준으로 출금 요청하는지 알 수 없음
  // → 동시 수정이 있었어도 감지 불가
```

---

## ✨ 올바른 접근 (After — 낙관적 잠금 + 재시도)

```
낙관적 잠금 전략:
  1. Account 로드 시 version 확인 (version = 5)
  2. account.withdraw() 실행 (불변식 검증, 상태 변경)
  3. 저장 시 version 일치 여부 확인:
     UPDATE account SET balance = ?, version = 6
     WHERE id = ? AND version = 5  ← version 조건 추가
  4. 업데이트 영향받은 행 = 0 → 다른 트랜잭션이 먼저 수정
     → OptimisticLockingFailureException
  5. 재시도 or 클라이언트에 409 Conflict 반환

클라이언트가 version을 Command에 포함:
  WithdrawMoneyCommand { accountId, amount, expectedVersion }
  → "version 5인 계좌에서 출금" 의도 명확
  → version이 다르면 충돌 즉시 감지
```

---

## 🔬 내부 동작 원리

### 1. Version 기반 낙관적 잠금 동작 방식

```
타임라인 — 동시 출금 시나리오:

  초기 상태: Account ACC-001, balance=500000, version=5

  T=0ms  트랜잭션 A: account = findById("ACC-001")
              → balance=500000, version=5
         트랜잭션 B: account = findById("ACC-001")
              → balance=500000, version=5

  T=5ms  트랜잭션 A: account.withdraw(300000)
              → balance=200000 (불변식 통과)
  T=5ms  트랜잭션 B: account.withdraw(400000)
              → balance=100000 (불변식 통과 — 아직 A가 커밋 전)

  T=10ms 트랜잭션 A: UPDATE account SET balance=200000, version=6
                     WHERE id='ACC-001' AND version=5
              → affected rows = 1 (성공)
              → COMMIT

  T=10ms 트랜잭션 B: UPDATE account SET balance=100000, version=6
                     WHERE id='ACC-001' AND version=5  ← version=5 이미 없음!
              → affected rows = 0 (실패!)
              → OptimisticLockingFailureException
              → ROLLBACK

  결과:
    트랜잭션 A: 300000 출금 성공, balance=200000, version=6
    트랜잭션 B: 재시도 필요 (잔고 200000에서 400000 출금 시도 → 불변식 위반 → 실패)

비관적 잠금과의 차이:
  비관적 잠금: 트랜잭션 B가 A의 COMMIT까지 대기 (블로킹)
  낙관적 잠금: 두 트랜잭션이 동시에 진행, 커밋 시점에 충돌 감지 (논블로킹)

  낙관적 잠금이 유리한 경우: 충돌 빈도가 낮을 때
  비관적 잠금이 유리한 경우: 충돌이 매우 빈번하여 재시도 비용이 높을 때
```

### 2. JPA @Version 동작 원리

```
JPA @Version 어노테이션:

  @Entity
  public class Account {
      @Id private String id;
      private BigDecimal balance;

      @Version
      private Long version;  // JPA가 자동 관리
  }

  로드:
    SELECT id, balance, version FROM account WHERE id = 'ACC-001'
    → Account { balance=500000, version=5 }

  저장:
    UPDATE account
    SET balance = 200000, version = 6      ← version 자동 증가
    WHERE id = 'ACC-001' AND version = 5   ← 조건에 현재 version
    → affected rows = 0 → JPA가 OptimisticLockException throw

  JPA가 처리하는 것:
    version 자동 증가 (5 → 6)
    UPDATE 조건에 version 추가
    affected rows = 0 시 OptimisticLockException 발생

  개발자가 처리해야 하는 것:
    OptimisticLockException / OptimisticLockingFailureException 캐치
    재시도 또는 클라이언트 응답

Spring Data JPA의 예외 변환:
  OptimisticLockException (JPA) → OptimisticLockingFailureException (Spring)
  → @ControllerAdvice에서 409 Conflict로 매핑
```

### 3. 재시도 전략

```
재시도 설계 원칙:

단순 재시도 (Retry):
  for (int attempt = 0; attempt < MAX_RETRY; attempt++) {
      try {
          account = accountRepository.findById(accountId);
          account.withdraw(amount);
          accountRepository.save(account);
          return; // 성공
      } catch (OptimisticLockingFailureException e) {
          if (attempt == MAX_RETRY - 1) throw e; // 마지막 시도 실패
          Thread.sleep(backoff(attempt)); // 지수 백오프
      }
  }

  MAX_RETRY: 3~5회 (너무 많으면 시스템 부하)
  Backoff: 지수 백오프 + Jitter (동시 재시도 분산)
    1차: 100ms, 2차: 200ms + 랜덤(0~100ms), 3차: 400ms + 랜덤

충돌 많을 때 재시도 vs 포기 결정:
  낙관적 잠금 충돌 빈도 > 20% → 비관적 잠금 검토
  재고나 계좌 같은 핫 레코드 → 파티셔닝 or 비관적 잠금

클라이언트에 409 반환 (재시도 클라이언트 위임):
  서버에서 재시도하지 않고 409 Conflict 반환
  클라이언트가 최신 상태를 다시 로드하고 재시도
  → "현재 처리 중인 요청이 많습니다. 잠시 후 다시 시도하세요"

어느 방식을 선택할까:
  서버 재시도: 클라이언트가 단순, 재시도 로직 중앙화
  클라이언트 재시도: 서버 부하 분산, 클라이언트가 최신 상태로 재시도
  → 트랜잭션 금액이 큰 경우: 클라이언트 재시도 (사용자 확인 절차 재수행)
  → 빈번한 소액 처리: 서버 재시도 (UX 매끄럽게)
```

### 4. Event Sourcing에서의 낙관적 잠금 — Expected Version

```
Event Sourcing에서의 동시성 제어:
  상태 저장: @Version 필드로 행 버전 관리
  Event Sourcing: 이벤트 스트림의 version(sequence)으로 관리

이벤트 스토어 구조:
  event_store:
  ┌──────────────┬─────────┬──────────────┬─────────────────────┐
  │ stream_id    │ version │ event_type   │ payload             │
  ├──────────────┼─────────┼──────────────┼─────────────────────┤
  │ account-001  │ 1       │ AccountOpened│ { owner: USER-42 }  │
  │ account-001  │ 2       │ MoneyDeposited│ { amount: 500000 } │
  │ account-001  │ 3       │ MoneyWithdrawn│ { amount: 100000 } │
  └──────────────┴─────────┴──────────────┴─────────────────────┘
  현재 version = 3

Expected Version 낙관적 잠금:

  트랜잭션 A:
    이벤트 로드: stream "account-001", version 1~3
    Aggregate 재구성: balance = 400000
    withdraw(200000) → MoneyWithdrawn 이벤트 생성
    INSERT INTO event_store (stream_id='account-001', version=4, ...)
    → version 4가 아직 없음 → 성공

  트랜잭션 B (동시에):
    이벤트 로드: stream "account-001", version 1~3
    Aggregate 재구성: balance = 400000
    withdraw(300000) → MoneyWithdrawn 이벤트 생성
    INSERT INTO event_store (stream_id='account-001', version=4, ...)
    → version 4가 이미 존재 (A가 먼저 삽입) → UNIQUE constraint 위반
    → OptimisticConcurrencyException

PostgreSQL UNIQUE 제약으로 구현:
  ALTER TABLE event_store
  ADD CONSTRAINT uq_stream_version UNIQUE (stream_id, version);

  INSERT INTO event_store (stream_id, version, event_type, payload)
  VALUES ('account-001', 4, 'MoneyWithdrawn', '...')
  -- version 4가 이미 있으면 → UNIQUE violation → 재시도
```

---

## 💻 실전 코드

```java
// ==============================
// 낙관적 잠금 완전 구현
// ==============================

// ✅ Account Entity — JPA @Version
@Entity
@Table(name = "account")
public class Account {

    @Id
    private String id;

    @Column(nullable = false)
    private BigDecimal balance;

    @Enumerated(EnumType.STRING)
    private AccountStatus status;

    @Version
    private Long version;  // JPA 낙관적 잠금

    public void withdraw(Money amount, String requestedBy) {
        if (status == AccountStatus.FROZEN)
            throw new AccountFrozenException(id);
        if (balance.compareTo(amount.amount()) < 0)
            throw new InsufficientBalanceException(id, amount, Money.of(balance));

        this.balance = balance.subtract(amount.amount());
        registerEvent(new MoneyWithdrawn(id, amount, Money.of(balance), requestedBy));
    }
}

// ✅ Command에 expectedVersion 포함 (선택적)
public record WithdrawMoneyCommand(
    @NotBlank String accountId,
    @NotNull @Positive BigDecimal amount,
    @NotBlank String requestedBy,
    Long expectedVersion,          // null이면 버전 무시, 있으면 검증
    @NotNull UUID idempotencyKey
) implements Command {}

// ✅ Command Handler — 재시도 포함
@Component
public class AccountCommandHandler {

    private static final int MAX_RETRY = 3;

    @CommandHandler
    public void handle(WithdrawMoneyCommand command) {
        for (int attempt = 0; attempt < MAX_RETRY; attempt++) {
            try {
                executeWithdraw(command);
                return; // 성공
            } catch (OptimisticLockingFailureException e) {
                if (attempt == MAX_RETRY - 1) {
                    throw new ConcurrentModificationException(
                        "동시 요청 충돌. 잠시 후 다시 시도해주세요.", e);
                }
                log.warn("낙관적 잠금 충돌 — 재시도 {}/{}: accountId={}",
                    attempt + 1, MAX_RETRY, command.accountId());
                sleepWithJitter(attempt);
            }
        }
    }

    @Transactional
    private void executeWithdraw(WithdrawMoneyCommand command) {
        Account account = accountRepository.findById(command.accountId())
            .orElseThrow(() -> new AccountNotFoundException(command.accountId()));

        // expectedVersion이 있으면 검증
        if (command.expectedVersion() != null
                && !account.getVersion().equals(command.expectedVersion())) {
            throw new StaleStateException(
                "계좌 상태가 변경됐습니다. 최신 정보를 확인하세요.");
        }

        account.withdraw(Money.of(command.amount()), command.requestedBy());
        accountRepository.save(account); // version 자동 증가
    }

    private void sleepWithJitter(int attempt) {
        long backoff = (long) Math.pow(2, attempt) * 100; // 100, 200, 400ms
        long jitter = (long) (Math.random() * 100);
        try {
            Thread.sleep(backoff + jitter);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }
}

// ✅ Exception Handler — 충돌을 409 Conflict로 변환
@RestControllerAdvice
public class CqrsExceptionHandler {

    @ExceptionHandler(ConcurrentModificationException.class)
    @ResponseStatus(HttpStatus.CONFLICT)
    public ProblemDetail handleConcurrentModification(ConcurrentModificationException e) {
        ProblemDetail problem = ProblemDetail.forStatus(409);
        problem.setTitle("동시 수정 충돌");
        problem.setDetail("요청이 충돌했습니다. 잠시 후 다시 시도해주세요.");
        return problem;
    }

    @ExceptionHandler(StaleStateException.class)
    @ResponseStatus(HttpStatus.CONFLICT)
    public ProblemDetail handleStaleState(StaleStateException e) {
        ProblemDetail problem = ProblemDetail.forStatus(409);
        problem.setTitle("데이터 변경됨");
        problem.setDetail("요청 이후 데이터가 변경됐습니다. 최신 상태를 확인하세요.");
        return problem;
    }
}

// ✅ Event Sourcing — PostgreSQL UNIQUE로 낙관적 잠금
@Repository
public class EventStoreRepository {

    @Transactional
    public void appendEvents(String streamId, long expectedVersion,
                              List<DomainEvent> events) {
        // 현재 최신 version 확인
        long currentVersion = jdbcTemplate.queryForObject(
            "SELECT COALESCE(MAX(version), 0) FROM event_store WHERE stream_id = ?",
            Long.class, streamId);

        if (currentVersion != expectedVersion) {
            throw new OptimisticConcurrencyException(
                String.format("stream=%s expectedVersion=%d currentVersion=%d",
                    streamId, expectedVersion, currentVersion));
        }

        // 이벤트 삽입 — UNIQUE(stream_id, version) 제약으로 동시 삽입 방지
        long nextVersion = expectedVersion;
        for (DomainEvent event : events) {
            nextVersion++;
            try {
                jdbcTemplate.update(
                    "INSERT INTO event_store (stream_id, version, event_type, payload, occurred_at) " +
                    "VALUES (?, ?, ?, ?::jsonb, ?)",
                    streamId, nextVersion,
                    event.getClass().getSimpleName(),
                    objectMapper.writeValueAsString(event),
                    Instant.now()
                );
            } catch (DuplicateKeyException e) {
                // UNIQUE 위반 → 낙관적 잠금 충돌
                throw new OptimisticConcurrencyException(
                    "동시 이벤트 저장 충돌: stream=" + streamId);
            }
        }
    }
}
```

---

## 📊 패턴 비교

```
낙관적 잠금 vs 비관적 잠금:

┌─────────────────────┬──────────────────────┬──────────────────────┐
│ 항목                │ 낙관적 잠금           │ 비관적 잠금           │
├─────────────────────┼──────────────────────┼──────────────────────┤
│ 잠금 시점           │ 커밋 시점             │ 로드 시점             │
├─────────────────────┼──────────────────────┼──────────────────────┤
│ 블로킹              │ 없음                  │ 있음 (대기 발생)      │
├─────────────────────┼──────────────────────┼──────────────────────┤
│ 처리량              │ 높음 (충돌 적을 때)   │ 낮음 (경합 시)        │
├─────────────────────┼──────────────────────┼──────────────────────┤
│ 충돌 처리           │ 재시도 필요           │ 대기 후 자동 처리     │
├─────────────────────┼──────────────────────┼──────────────────────┤
│ 적합한 상황         │ 충돌 빈도 낮음        │ 충돌 빈도 높음        │
├─────────────────────┼──────────────────────┼──────────────────────┤
│ 데드락              │ 없음                  │ 가능                  │
└─────────────────────┴──────────────────────┴──────────────────────┘

Event Sourcing에서의 낙관적 잠금:
  상태 DB: @Version 필드 → UPDATE WHERE version = ?
  이벤트 스토어: UNIQUE(stream_id, version) → INSERT 충돌
  → 두 방식 모두 동일한 의미의 낙관적 잠금
```

---

## ⚖️ 트레이드오프

```
낙관적 잠금의 장점:
  데드락 없음
  읽기를 잠금 없이 처리
  CQRS에서 읽기 모델 조회와 쓰기 모델 잠금 완전 분리

낙관적 잠금의 단점:
  충돌 빈번하면 재시도 오버헤드 증가
  재시도 설계 필요 (최대 시도 횟수, 백오프 전략)
  클라이언트에 409 응답 처리 필요

핫 레코드 문제:
  하나의 계좌에 초당 수백 건의 동시 출금 → 낙관적 잠금 충돌 폭발
  해결: 계좌 파티셔닝, 이벤트 기반 집계, 비관적 잠금으로 전환
```

---

## 📌 핵심 정리

```
CQRS + 낙관적 잠금 핵심:

동작 원리:
  Aggregate 로드 시 version 확인
  저장 시 WHERE version = ? 조건으로 충돌 감지
  충돌: 재시도 or 409 Conflict 반환

JPA 구현:
  @Version Long version; — JPA가 자동으로 version 조건 추가
  OptimisticLockingFailureException — Spring이 변환

Event Sourcing 구현:
  UNIQUE(stream_id, version) 제약
  INSERT 충돌 → OptimisticConcurrencyException

재시도 전략:
  MAX_RETRY: 3~5회
  지수 백오프 + Jitter
  충돌 > 20% → 비관적 잠금 검토

클라이언트 응답:
  충돌 → 409 Conflict
  "최신 상태 확인 후 재시도" 안내
```

---

## 🤔 생각해볼 문제

**Q1.** 낙관적 잠금 충돌이 자주 발생하는 "핫 계좌"(초당 수백 건 동시 접근)가 있다면 어떻게 설계를 변경해야 하는가?

<details>
<summary>해설 보기</summary>

핫 레코드 문제는 낙관적 잠금의 한계입니다. 세 가지 접근이 있습니다.

첫째, 비관적 잠금으로 전환합니다. 충돌이 매우 빈번하다면 재시도 비용이 대기 비용보다 클 수 있습니다. `SELECT FOR UPDATE NOWAIT`(즉시 실패) 또는 `SELECT FOR UPDATE SKIP LOCKED`(잠긴 행 건너뜀) 전략을 활용합니다.

둘째, 집계 계좌(Aggregate Account)를 파티셔닝합니다. 하나의 계좌를 여러 서브 계좌로 분산합니다. 예를 들어 ACC-001 계좌를 ACC-001-0 ~ ACC-001-9로 10개로 분산하고, 출금 요청을 해시 기반으로 분산합니다. 전체 잔고는 10개 서브 계좌의 합으로 계산합니다. 분산 카운터 패턴과 유사합니다.

셋째, 이벤트 기반 비동기 처리로 전환합니다. 출금 요청을 큐에 쌓고 순차적으로 처리합니다. 처리량은 줄지만 충돌이 없습니다. 실시간성보다 정확성이 중요한 배치 이체 등에 적합합니다.

</details>

---

**Q2.** Event Sourcing에서 expectedVersion을 Command에 포함시키면 어떤 문제가 해결되고 어떤 문제가 생기는가?

<details>
<summary>해설 보기</summary>

해결되는 문제로, 클라이언트가 어느 시점의 Aggregate 상태를 기반으로 Command를 보내는지 명확해집니다. "version 5인 계좌에서 출금"이라는 의도가 명확하므로, 그 사이에 다른 변경이 있었다면 즉시 감지됩니다. Aggregate를 다시 로드하지 않고도 충돌 감지가 가능합니다.

생기는 문제로, 클라이언트가 Aggregate의 현재 version을 알아야 합니다. 이는 클라이언트가 Query를 통해 version 정보를 받아야 함을 의미합니다. UI가 읽기 모델을 표시하고 있을 때, 읽기 모델에 version을 포함해야 합니다. 또한 expectedVersion을 포함하지 않는 Command는 어떻게 처리할지 결정해야 합니다(무시 or 필수).

실무에서 expectedVersion을 Command에 포함하는 방식은 금융, 재고 관리처럼 정확성이 중요하고 충돌 시 사용자 확인이 필요한 도메인에 적합합니다. 충돌 빈도가 낮고 재시도가 자연스러운 경우에는 서버에서만 version을 관리하고 클라이언트에 노출하지 않는 방식이 단순합니다.

</details>

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[이전: Command Bus 패턴 ⬅️](./04-command-bus.md)** | **[다음: Command 결과 반환 패턴 ➡️](./06-command-result-patterns.md)**

</div>
