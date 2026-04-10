# Aggregate 재구성

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `apply()` 패턴으로 이벤트를 리플레이해 Aggregate를 재구성하는 메커니즘은 무엇인가?
- 이벤트 수가 늘어날수록 로딩 시간이 어떻게 변화하는가?
- Command Handler에서 Aggregate 로드 → 검증 → 저장까지의 완전한 흐름은 어떻게 되는가?
- Aggregate 재구성 과정에서 발생할 수 있는 문제와 그 해결 방법은 무엇인가?
- 스냅샷 없이 재구성 성능을 개선할 수 있는 방법은 무엇인가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

Event Sourcing에서 Aggregate는 DB에 현재 상태로 저장되지 않는다. 이벤트 스트림을 처음부터 읽어 순서대로 적용해야 비로소 현재 상태를 얻는다. 이 재구성 과정이 어떻게 동작하는지 이해하지 못하면, 왜 이벤트가 수천 개 쌓이면 느려지는지, 스냅샷이 왜 필요한지, `@EventSourcingHandler`가 어떤 역할인지 알 수 없다.

---

## 😱 흔한 실수 (Before — 재구성 원리를 모를 때)

```
실수 1: Aggregate 재구성에서 비즈니스 로직을 실행

  // ❌ apply() 안에서 불변식 검증
  private void apply(MoneyWithdrawn event) {
      if (this.balance.compareTo(event.getAmount()) < 0)  // 여기서는 하면 안 됨!
          throw new InsufficientBalanceException();
      this.balance = this.balance.subtract(event.getAmount());
  }
  // 문제: 리플레이 중에도 호출됨 → 과거의 이미 검증된 이벤트에 불변식 재검증
  // 이미 저장된 이벤트는 당시에 검증 완료 → 재구성 시 검증 불필요

실수 2: 재구성 메서드에서 외부 의존성 사용

  private void apply(OrderConfirmed event) {
      // ❌ 재구성 중 DB 조회 — 리플레이마다 실행됨
      Product product = productRepository.findById(event.getProductId());
      this.productName = product.getName();
  }
  // 재구성은 순수하게 이벤트 → 상태 변환이어야 함
  // 외부 의존성 → 재구성 비용 폭발

실수 3: 이벤트 타입 누락 처리 없음

  private void apply(DomainEvent event) {
      if (event instanceof MoneyDeposited e) {
          this.balance = this.balance.add(e.getAmount());
      }
      // MoneyTransferred, MoneyWithdrawn 등 처리 누락
      // → 새 이벤트 타입 추가 시 재구성 결과 오류
  }
```

---

## ✨ 올바른 접근 (After — 순수한 상태 전환으로서의 apply())

```
apply() 메서드의 책임:
  ① 이벤트에서 꺼낸 값으로 필드 업데이트
  ② 외부 의존성 없음 (DB 조회, 서비스 호출 금지)
  ③ 불변식 검증 없음 (이미 검증된 이벤트)
  ④ 부수효과 없음 (이메일 발송 등 금지)
  ⑤ 결정론적(Deterministic): 같은 이벤트 → 항상 같은 상태

올바른 흐름:
  Command 수신
    → Aggregate 재구성 (이벤트 리플레이)
    → 불변식 검증 (Command 처리 메서드에서)
    → 새 이벤트 생성 (apply() + 큐에 추가)
    → 이벤트 스토어에 저장
    → Kafka 발행 → Projection 업데이트

재구성만의 흐름 (읽기 전용):
  이벤트 로드 → apply() × N → 현재 상태 완성
  → 불변식 검증 없음 (Command 없으므로)
```

---

## 🔬 내부 동작 원리

### 1. apply() 패턴의 완전한 구조

```
두 가지 컨텍스트에서 apply() 호출:

① Command 처리 중 — 새 이벤트 생성 시:
  account.withdraw(100000)
    → 불변식 검증 (잔고 충분한가?)
    → 새 이벤트 생성: MoneyWithdrawn { amount: 100000 }
    → apply(MoneyWithdrawn) 호출  ← 상태 변경
    → pendingEvents에 추가 → 나중에 이벤트 스토어 저장

② 재구성 중 — 이벤트 스토어에서 로드 시:
  eventStore.load("account-ACC-001")
    → [AccountOpened, MoneyDeposited, MoneyWithdrawn, ...]
    → apply(AccountOpened)    ← 상태 초기화
    → apply(MoneyDeposited)   ← 잔고 += 500000
    → apply(MoneyWithdrawn)   ← 잔고 -= 100000
    → 현재 상태 완성

핵심: 동일한 apply() 메서드가 두 컨텍스트에서 호출
  → 이벤트 저장 시와 재구성 시 결과가 동일 (결정론적)
  → 불변식 검증 / 외부 의존성이 없어야 일관성 보장

apply() 분기 처리:
  private void apply(DomainEvent event) {
      switch (event) {
          case AccountOpened e    -> handleAccountOpened(e);
          case MoneyDeposited e   -> handleMoneyDeposited(e);
          case MoneyWithdrawn e   -> handleMoneyWithdrawn(e);
          case MoneyTransferred e -> handleMoneyTransferred(e);
          case AccountFrozen e    -> handleAccountFrozen(e);
          case AccountClosed e    -> handleAccountClosed(e);
          default -> log.warn("알 수 없는 이벤트 타입: {}", event.getClass());
          // default: 무시 — 미래 이벤트 타입 하위 호환
      }
  }
```

### 2. 이벤트 수와 로딩 시간의 관계

```
이벤트 수에 따른 재구성 비용:

  가정: 이벤트당 처리 시간 0.1ms, DB 조회 왕복 10ms

  이벤트 10개:   10ms(DB) + 1ms(처리)  = 11ms   ← 허용
  이벤트 100개:  10ms(DB) + 10ms(처리) = 20ms   ← 허용
  이벤트 1,000개: 10ms(DB) + 100ms(처리) = 110ms  ← 주의
  이벤트 10,000개: 10ms(DB) + 1000ms(처리) = 1010ms ← 문제

  실제 병목은 두 곳:
    ① DB I/O: 이벤트 수 × 행 크기만큼 읽기
    ② 역직렬화: 이벤트 수 × JSON 파싱 비용
    ③ apply() 실행: 이벤트 수 × 상태 전환 로직

실측 예시 (PostgreSQL, 이벤트 1개당 평균 500B):
  이벤트 100개  = 50KB   → 쿼리 5ms, 역직렬화 5ms  → 총 10ms
  이벤트 1000개 = 500KB  → 쿼리 20ms, 역직렬화 50ms → 총 70ms
  이벤트 5000개 = 2.5MB  → 쿼리 80ms, 역직렬화 250ms→ 총 330ms
  이벤트 10000개= 5MB    → 쿼리 200ms, 역직렬화 500ms→ 총 700ms

한계 임계값:
  일반적으로 이벤트 1,000~2,000개 이상 → 스냅샷 도입 검토
  Command 처리 p99 < 200ms 목표 → 이벤트 최대 허용 수 계산
```

### 3. 재구성 흐름 — Command Handler 관점

```
전체 Command 처리 흐름 (Event Sourcing):

  // 1. Command 수신
  WithdrawMoneyCommand cmd = new WithdrawMoneyCommand("ACC-001", 100000);

  // 2. 이벤트 스트림 로드
  String streamId = "account-" + cmd.accountId();
  List<StoredEvent> storedEvents = eventStore.loadEvents(streamId);
  // SELECT * FROM event_store WHERE stream_id = 'account-ACC-001' ORDER BY version
  // → [{AccountOpened,v1}, {MoneyDeposited,v2}, {MoneyDeposited,v3}]

  // 3. 이벤트 역직렬화
  List<DomainEvent> events = storedEvents.stream()
      .map(se -> eventSerializer.deserialize(se.payload(), se.eventType()))
      .collect(toList());

  // 4. Aggregate 재구성 (이벤트 리플레이)
  Account account = new Account();
  events.forEach(account::apply);
  // apply(AccountOpened)   → id, ownerId, status = ACTIVE, balance = 0
  // apply(MoneyDeposited)  → balance = 500000
  // apply(MoneyDeposited)  → balance = 700000
  // 현재 상태: balance = 700000

  // 5. 현재 version 확인 (낙관적 잠금용)
  long currentVersion = storedEvents.size(); // = 3

  // 6. 불변식 검증 + 새 이벤트 생성
  account.withdraw(Money.of(100000));
  // 내부: balance(700000) >= amount(100000) → OK
  //       MoneyWithdrawn { amount: 100000, balanceAfter: 600000 } 생성
  //       apply(MoneyWithdrawn) → balance = 600000
  //       pendingEvents에 추가

  // 7. 새 이벤트 저장 (낙관적 잠금: expectedVersion = 3)
  eventStore.appendEvents(streamId, currentVersion, account.getPendingEvents());
  // INSERT INTO event_store (stream_id='account-ACC-001', version=4, ...)
  // UNIQUE(stream_id, version) 제약으로 동시 저장 충돌 감지

  // 8. Kafka 발행 → Projection 비동기 처리
```

### 4. 재구성 최적화 — 스냅샷 없이

```
스냅샷 없이 재구성 성능 개선 방법:

방법 1: 이벤트 배치 로드
  대신: 이벤트를 1개씩 로드 (N번 왕복)
  개선: 한 번의 쿼리로 전체 스트림 로드
    SELECT * FROM event_store WHERE stream_id = ? ORDER BY version
  → 이미 이렇게 해야 함 — N+1 쿼리는 치명적

방법 2: 이벤트 페이로드 압축
  대용량 이벤트 페이로드 → GZIP 압축 저장
  → DB I/O 감소, CPU 비용 추가 (압축/해제)

방법 3: 이벤트 정규화 — 불필요한 이벤트 최소화
  "필드가 바뀔 때마다 이벤트" 대신
  "의미 있는 도메인 행동 당 이벤트"
  → 이벤트 수 자체가 줄어듦
  ❌ AccountFieldUpdated (모든 필드 변경마다)
  ✅ AddressChanged, PhoneNumberChanged (의미 있는 변경만)

방법 4: 이벤트 합치기 (Event Folding)
  오래된 이벤트를 합쳐 하나로 만들기 (불가역적)
  → 감사 로그 손실 → 권장하지 않음
  → 스냅샷이 더 나은 대안
```

---

## 💻 실전 코드

```java
// ==============================
// Aggregate 재구성 완전 구현
// ==============================

// ✅ Account Aggregate — apply() 패턴
public class Account {

    // 현재 상태 필드
    private AccountId id;
    private OwnerId ownerId;
    private Money balance;
    private AccountStatus status;

    // 이벤트 스토어 저장 후 새 이벤트 추적
    private final List<DomainEvent> pendingEvents = new ArrayList<>();
    private long version = 0; // 현재 스트림 버전

    // ── 재구성 진입점 ──────────────────────────────────────────

    public static Account reconstitute(List<DomainEvent> events) {
        Account account = new Account();
        events.forEach(event -> {
            account.apply(event);
            account.version++;
        });
        return account;
    }

    // ── Command 처리 메서드 (불변식 검증 + 이벤트 생성) ──────────

    public void withdraw(Money amount, String requestedBy) {
        // 불변식 검증 — 재구성 시에는 호출 안 됨
        if (status != AccountStatus.ACTIVE)
            throw new AccountNotActiveException(id, status);
        if (balance.isLessThan(amount))
            throw new InsufficientBalanceException(id, amount, balance);

        // 새 이벤트 생성 + 즉시 상태 반영
        applyAndRecord(new MoneyWithdrawn(
            id.value(), amount.amount(),
            balance.subtract(amount).amount(),
            requestedBy, Instant.now()
        ));
    }

    public void deposit(Money amount, String depositedBy) {
        if (status == AccountStatus.CLOSED)
            throw new AccountClosedException(id);

        applyAndRecord(new MoneyDeposited(
            id.value(), amount.amount(),
            balance.add(amount).amount(),
            depositedBy, Instant.now()
        ));
    }

    public void freeze(String reason, String frozenBy) {
        if (status != AccountStatus.ACTIVE)
            throw new AccountNotActiveException(id, status);

        applyAndRecord(new AccountFrozen(
            id.value(), reason, frozenBy, Instant.now()
        ));
    }

    // ── apply() — 상태 전환만 (검증 없음, 외부 의존성 없음) ──────

    private void apply(DomainEvent event) {
        switch (event) {
            case AccountOpened e -> {
                this.id = new AccountId(e.accountId());
                this.ownerId = new OwnerId(e.ownerId());
                this.balance = Money.ZERO;
                this.status = AccountStatus.ACTIVE;
            }
            case MoneyDeposited e -> {
                // balanceAfter를 직접 사용 — 재계산 불필요 (결정론적)
                this.balance = Money.of(e.balanceAfter());
            }
            case MoneyWithdrawn e -> {
                this.balance = Money.of(e.balanceAfter());
            }
            case MoneyTransferred e -> {
                this.balance = Money.of(e.balanceAfter());
            }
            case AccountFrozen e -> {
                this.status = AccountStatus.FROZEN;
            }
            case AccountUnfrozen e -> {
                this.status = AccountStatus.ACTIVE;
            }
            case AccountClosed e -> {
                this.status = AccountStatus.CLOSED;
            }
            default ->
                // 알 수 없는 이벤트 타입 — 무시 (미래 이벤트 하위 호환)
                log.warn("알 수 없는 이벤트 타입: {}", event.getClass().getSimpleName());
        }
    }

    // Command 처리: apply() + pendingEvents 기록
    private void applyAndRecord(DomainEvent event) {
        apply(event);
        pendingEvents.add(event);
        this.version++;
    }

    // Command Handler에서 저장 후 초기화
    public List<DomainEvent> pullPendingEvents() {
        List<DomainEvent> events = List.copyOf(pendingEvents);
        pendingEvents.clear();
        return events;
    }

    public long getVersion() { return version; }
}

// ✅ Account Repository — 이벤트 스토어 기반
@Repository
public class AccountRepository {

    private final EventStore eventStore;
    private final EventSerializer serializer;

    public Optional<Account> findById(AccountId accountId) {
        String streamId = "account-" + accountId.value();
        List<StoredEvent> stored = eventStore.loadEvents(streamId);

        if (stored.isEmpty()) return Optional.empty();

        // 역직렬화
        List<DomainEvent> events = stored.stream()
            .map(se -> serializer.deserialize(se.payload(), se.eventType()))
            .collect(toList());

        // 재구성
        return Optional.of(Account.reconstitute(events));
    }

    @Transactional
    public void save(Account account) {
        String streamId = "account-" + account.getId().value();
        long expectedVersion = account.getVersion() - account.getPendingEvents().size();

        List<StoredEvent> toStore = account.pullPendingEvents().stream()
            .map(e -> new StoredEvent(
                e.getClass().getSimpleName(),
                serializer.serialize(e),
                Map.of("correlationId", MDC.get("correlationId"))
            ))
            .collect(toList());

        eventStore.appendEvents(streamId, expectedVersion, toStore);
    }
}

// ✅ Command Handler — 재구성 + 처리 + 저장
@Component
public class AccountCommandHandler {

    @CommandHandler
    @Transactional
    public void handle(WithdrawMoneyCommand command) {
        // 1. 이벤트 스토어에서 재구성
        Account account = accountRepository.findById(
            new AccountId(command.accountId()))
            .orElseThrow(() -> new AccountNotFoundException(command.accountId()));

        // 2. Command 처리 (불변식 검증 + 이벤트 생성)
        account.withdraw(
            Money.of(command.amount()),
            command.requestedBy()
        );

        // 3. 새 이벤트 저장
        accountRepository.save(account);
        // 내부: eventStore.appendEvents("account-ACC-001", expectedVersion, [MoneyWithdrawn])
    }
}

// ✅ Axon Framework 방식 (자동 재구성)
@Aggregate
public class AccountAggregate {

    @AggregateIdentifier
    private String accountId;
    private BigDecimal balance;
    private AccountStatus status;

    // Axon이 자동으로 이벤트 로드 → @EventSourcingHandler 순서대로 호출
    @CommandHandler
    public void handle(WithdrawMoneyCommand command) {
        if (balance.compareTo(command.amount()) < 0)
            throw new InsufficientBalanceException();

        apply(new MoneyWithdrawn(accountId, command.amount(),
            balance.subtract(command.amount()), command.requestedBy()));
    }

    @EventSourcingHandler  // = apply() 패턴
    public void on(MoneyWithdrawn event) {
        this.balance = event.balanceAfter(); // 상태 전환만
        // 검증 없음, 외부 의존성 없음
    }

    @EventSourcingHandler
    public void on(AccountOpened event) {
        this.accountId = event.accountId();
        this.balance = BigDecimal.ZERO;
        this.status = AccountStatus.ACTIVE;
    }
}
```

---

## 📊 패턴 비교

```
상태 저장 vs Event Sourcing — Aggregate 로드 비교:

상태 저장:
  SELECT * FROM account WHERE id = 'ACC-001'
  → 1번 쿼리, 1개 행
  → 즉시 현재 상태
  → 이벤트 수에 무관하게 일정 시간

Event Sourcing:
  SELECT * FROM event_store WHERE stream_id = 'account-ACC-001' ORDER BY version
  → 1번 쿼리, N개 행
  → apply() × N 후 현재 상태
  → 이벤트 수에 선형 비례

이벤트 수별 로딩 시간 (참고치):
  10개  → ~5ms   (상태 저장과 유사)
  100개 → ~15ms  (허용)
  500개 → ~50ms  (주의)
  1000개→ ~100ms (스냅샷 도입 검토)
  5000개→ ~500ms (스냅샷 필수)
```

---

## ⚖️ 트레이드오프

```
재구성 방식의 장점:
  현재 상태가 이벤트에서 자동 계산 → 상태 불일치 불가
  과거 상태 재현: 특정 이벤트까지만 리플레이
  새 필드 추가: Aggregate 코드 수정 + 기존 이벤트 자동 재구성

재구성 방식의 단점:
  이벤트 수에 비례한 로딩 비용
  이벤트가 수천 개면 수백 ms → 스냅샷 필요
  역직렬화 비용 (JSON 파싱 × 이벤트 수)

apply()의 순수성 유지 비용:
  apply()에서 외부 데이터 사용 불가
  → 이벤트 페이로드에 필요한 모든 데이터 포함 필요
  예: 이체 이벤트에 상대 계좌 이름을 저장하고 싶으면
      이벤트 발행 전에 조회해서 페이로드에 포함
```

---

## 📌 핵심 정리

```
Aggregate 재구성 핵심:

apply() 원칙:
  상태 전환만 (balance = balance + amount)
  검증 없음 (이미 저장된 이벤트 = 이미 검증)
  외부 의존성 없음 (결정론적이어야 함)
  부수효과 없음 (이메일 발송 등 금지)

두 컨텍스트:
  ① Command 처리: 새 이벤트 생성 → apply() + pendingEvents 추가
  ② 재구성: 저장된 이벤트 로드 → apply() 순서대로 → 현재 상태

성능 임계값:
  이벤트 ~1000개 이상 → 스냅샷 도입 검토
  Command 처리 p99 목표 기반으로 최대 허용 이벤트 수 결정

완전한 흐름:
  이벤트 로드 → 재구성 → 불변식 검증 → 이벤트 생성 → 저장
```

---

## 🤔 생각해볼 문제

**Q1.** apply() 메서드가 결정론적이어야 한다고 했다. `Instant.now()`를 apply() 안에서 사용하면 왜 문제가 되는가?

<details>
<summary>해설 보기</summary>

결정론적이란 "같은 입력에 대해 항상 같은 출력"을 의미합니다. `Instant.now()`는 호출 시점마다 다른 값을 반환하므로, apply() 안에서 사용하면 동일한 이벤트를 리플레이해도 다른 상태가 만들어집니다.

예를 들어 apply() 안에서 `this.lastUpdatedAt = Instant.now()`를 실행하면, 처음 Command 처리 시의 시각과 나중에 재구성할 때의 시각이 달라집니다. 이는 버그의 원인이 됩니다.

올바른 방법은 시각 정보를 이벤트 페이로드에 포함하는 것입니다. Command 처리 시 `Instant.now()`로 시각을 기록해 이벤트에 담고, apply()에서는 `this.lastUpdatedAt = event.occurredAt()`처럼 이벤트의 값을 사용합니다. 이렇게 하면 재구성할 때도 항상 동일한 시각이 사용됩니다.

</details>

---

**Q2.** Aggregate 재구성에 이벤트 1,000개가 걸린다. 스냅샷 없이 이 숫자를 줄일 수 있는 설계 결정은 무엇인가?

<details>
<summary>해설 보기</summary>

이벤트 수를 줄이는 설계 결정이 두 가지 방향에서 가능합니다.

첫째, 이벤트 세분화 수준을 재검토합니다. "필드가 바뀔 때마다 이벤트"가 아니라 "의미 있는 도메인 행동 당 이벤트"로 설계하면 이벤트 수가 줄어듭니다. 예를 들어 프로필 수정에서 이름, 전화번호, 주소가 한 번에 바뀔 때 세 개의 이벤트 대신 `ProfileUpdated` 하나로 통합할 수 있습니다.

둘째, Aggregate 경계를 재검토합니다. 너무 큰 Aggregate는 이벤트가 빠르게 쌓입니다. 계좌 하나가 입출금 이벤트를 매일 수십 개씩 생성한다면, 일별 집계 같은 설계로 Aggregate를 분리하는 것을 고려합니다.

셋째, 빈번하게 바뀌지 않는 상태만 Aggregate에 포함합니다. 로그성 데이터(접속 이력, 조회 이력)는 별도 이벤트 스트림 또는 별도 저장소로 분리합니다.

하지만 이런 방법에도 한계가 있고, 근본적으로는 스냅샷이 가장 직접적인 해결책입니다.

</details>

---

**Q3.** 이벤트 스트림에 적용할 수 없는 이벤트 타입이 발견됐을 때(알 수 없는 이벤트 타입) apply()에서 어떻게 처리해야 하는가?

<details>
<summary>해설 보기</summary>

알 수 없는 이벤트 타입은 무시하는 것이 권장됩니다. 예외를 던지면 해당 Aggregate 전체를 로드할 수 없게 되어 서비스 장애로 이어질 수 있습니다.

언제 알 수 없는 이벤트가 나타나는가 하면, 과거 버전의 이벤트 타입이 코드에서 제거됐을 때, 또는 미래 버전의 코드가 쓴 이벤트를 구 버전 코드가 읽을 때입니다.

무시 시 주의사항으로, 알 수 없는 이벤트가 Aggregate 상태에 영향을 주는 이벤트라면 현재 상태가 잘못 계산될 수 있습니다. 따라서 무시는 안전하게 처리 가능한 경우에만 적용합니다. 경고 로그를 남겨 모니터링에서 감지할 수 있도록 합니다. 이벤트 타입을 제거하기 전에는 반드시 모든 스트림에서 해당 이벤트가 더 이상 발생하지 않음을 확인해야 합니다.

</details>

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[이전: 이벤트 스토어 설계 ⬅️](./02-event-store-design.md)** | **[다음: 스냅샷 패턴 ➡️](./04-snapshot-pattern.md)**

</div>
