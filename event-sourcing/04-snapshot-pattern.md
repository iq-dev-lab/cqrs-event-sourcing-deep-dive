# 스냅샷(Snapshot) 패턴

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 스냅샷이 이벤트 리플레이 비용을 어떻게 줄이는가?
- 스냅샷을 언제 찍어야 하는가? 기준은 무엇인가?
- 스냅샷 + 이후 이벤트를 함께 로드하는 전략은 어떻게 구현하는가?
- 스냅샷 스키마가 Aggregate 구조 변경으로 오래됐을 때 어떻게 처리하는가?
- 스냅샷과 이벤트 스토어를 동기화하는 방법은 무엇인가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

활발한 계좌는 하루에 수십 개의 이벤트를 쌓는다. 1년이 지나면 수천 개. 매 Command 처리마다 수천 개의 이벤트를 리플레이하면 수백 ms가 걸린다. 스냅샷은 이 문제를 "특정 시점의 현재 상태를 저장해두고, 그 이후 이벤트만 리플레이"하는 방식으로 해결한다. 이 문서는 스냅샷의 구조, 저장 시점 결정, 로딩 전략, 스키마 마이그레이션까지 다룬다.

---

## 😱 흔한 실수 (Before — 스냅샷 없이 무한 성장)

```
실수 1: 이벤트만 믿고 스냅샷 없이 운영

  계좌 개설 1년 후:
    event_store에 3,650개 이벤트 (하루 10개 × 365일)
    Account 로드: SELECT 3650행 + 역직렬화 + 3650회 apply()
    → 평균 로딩 시간: 2~3초
    → Command 처리 SLA 500ms → 위반

  처음엔 빠르고 나중에 느려지는 패턴:
    이벤트 < 100개: 문제 없음 → "ES 잘 작동하네"
    이벤트 1000개: 조금 느려짐 → "인프라 문제?"
    이벤트 5000개: 매우 느림 → "ES가 문제였어!"

실수 2: 스냅샷을 모든 이벤트마다 찍음

  void handle(WithdrawMoneyCommand cmd) {
      account.withdraw(...);
      eventStore.appendEvents(...);
      snapshotStore.save(account); // 매 Command마다 스냅샷
  }
  // 이벤트 저장과 스냅샷 저장이 같은 빈도 → 스냅샷 의미 없음
  // 스냅샷 저장 비용만 추가됨

실수 3: 스냅샷 역직렬화 실패 시 폴백 없음

  Account account = snapshotStore.loadLatest("account-ACC-001");
  // Aggregate 구조 변경 후 구버전 스냅샷 로드
  // → ClassCastException → 서비스 중단
  // 스냅샷 실패 시 이벤트 전체 리플레이로 폴백해야 함
```

---

## ✨ 올바른 접근 (After — N개 이벤트마다 스냅샷)

```
스냅샷 로딩 전략:

스냅샷 없을 때:
  전체 이벤트 로드 → 리플레이 → 현재 상태

스냅샷 있을 때:
  최신 스냅샷 로드 (version N까지의 상태)
  + N+1 이후 이벤트만 로드 → 리플레이
  = 현재 상태

  예시: 이벤트 5000개, 스냅샷(version=4990)
    5000개 리플레이 → 스냅샷 도입 후 → 10개 리플레이
    로딩 시간: 500ms → 5ms

스냅샷 저장 시점:
  전략 1: N개 이벤트마다 (권장)
    이벤트 100개 추가될 때마다 스냅샷 저장
    → 최악의 경우 99개 리플레이

  전략 2: 주기적으로 (배치)
    1시간마다 스냅샷 저장
    → 이벤트 생성 빈도에 따라 리플레이 수 다양

  전략 3: Command 처리 시간 기반
    로딩 시간 > 임계값 → 스냅샷 저장
    → 실측 기반이지만 측정 복잡
```

---

## 🔬 내부 동작 원리

### 1. 스냅샷 저장소 구조

```
스냅샷 테이블:

CREATE TABLE snapshots (
    stream_id    VARCHAR(255) NOT NULL,
    version      BIGINT NOT NULL,         -- 스냅샷 시점의 이벤트 버전
    aggregate_type VARCHAR(255) NOT NULL, -- "Account"
    state        JSONB NOT NULL,          -- 직렬화된 Aggregate 상태
    created_at   TIMESTAMPTZ DEFAULT NOW(),

    PRIMARY KEY (stream_id, version)
);

-- 최신 스냅샷 조회에 사용
CREATE INDEX idx_snapshot_latest ON snapshots (stream_id, version DESC);

예시 스냅샷 데이터:
  stream_id: "account-ACC-001"
  version:   4990  ← 이벤트 4990개 처리 후의 상태
  state: {
    "id": "ACC-001",
    "ownerId": "USER-42",
    "balance": 1250000,
    "status": "ACTIVE"
  }

스냅샷 로드 + 이후 이벤트 로드:
  1. SELECT * FROM snapshots
     WHERE stream_id = 'account-ACC-001'
     ORDER BY version DESC LIMIT 1
     → 스냅샷: version=4990, balance=1250000

  2. SELECT * FROM event_store
     WHERE stream_id = 'account-ACC-001'
       AND version > 4990
     ORDER BY version ASC
     → 이벤트: version 4991, 4992, ..., 5000 (10개)

  3. 스냅샷 역직렬화 → 상태 복원
  4. 10개 이벤트 리플레이 → 최종 현재 상태
  → 5000개 리플레이 → 10개 리플레이 (99.8% 감소)
```

### 2. 스냅샷 저장 시점 결정

```
전략 1: N개 이벤트마다 (가장 일반적)

  스냅샷 저장 판단 코드:
    @CommandHandler
    @Transactional
    void handle(WithdrawMoneyCommand cmd) {
        Account account = accountRepository.findById(cmd.accountId());
        account.withdraw(cmd.amount());
        accountRepository.save(account);

        // 100개 이벤트마다 스냅샷
        if (account.getVersion() % 100 == 0) {
            snapshotStore.save(account);
        }
    }

  N 결정 기준:
    목표 최대 리플레이 수: M (예: 100개)
    → N = M (100개마다 스냅샷)
    이벤트 생성 빈도: F개/일
    → 스냅샷 저장 빈도: F/N 개/일

  예시: 하루 100개 이벤트, N=50
    → 하루 2번 스냅샷 저장
    → 최악: 49개 이벤트 리플레이
    → 로딩 시간: ~5ms (허용)

전략 2: 비동기 배치 스냅샷 (Command 처리 경로에서 제외)

  @Scheduled(fixedDelay = 3600000) // 1시간마다
  void takeSnapshots() {
      // 최근 1시간 동안 이벤트가 많이 쌓인 stream 식별
      List<String> hotStreams = eventStore.findStreamsWithManyEvents(
          threshold: 50, since: Instant.now().minus(1, HOURS));

      hotStreams.forEach(streamId -> {
          Account account = accountRepository.findById(streamId);
          snapshotStore.save(account);
      });
  }
  // 장점: Command 처리 경로에 스냅샷 저장 비용 없음
  // 단점: 최대 1시간 동안 스냅샷 없을 수 있음
```

### 3. 스냅샷 + 이후 이벤트 로딩 전략

```
완전한 로딩 알고리즘:

  Account account = loadWithSnapshot("account-ACC-001");

  내부 동작:
    1. 최신 스냅샷 조회
       Snapshot snapshot = snapshotStore.findLatest(streamId);

    2a. 스냅샷 있음:
        Account account = deserialize(snapshot.state()); // 스냅샷 → 상태 복원
        long fromVersion = snapshot.version();

        // 스냅샷 이후 이벤트만 로드
        List<DomainEvent> recentEvents = eventStore
            .loadEventsAfterVersion(streamId, fromVersion);

        recentEvents.forEach(account::apply);
        account.setVersion(fromVersion + recentEvents.size());
        return account;

    2b. 스냅샷 없음 (또는 스냅샷 역직렬화 실패):
        // 전체 이벤트 리플레이 (폴백)
        List<DomainEvent> allEvents = eventStore.loadAllEvents(streamId);
        return Account.reconstitute(allEvents);

폴백의 중요성:
  스냅샷 역직렬화 실패 시 (Aggregate 구조 변경, 스키마 불일치)
  → 예외를 잡아 전체 이벤트 리플레이로 폴백
  → 서비스 중단 방지

  try {
      account = deserialize(snapshot.state());
  } catch (Exception e) {
      log.warn("스냅샷 역직렬화 실패, 전체 리플레이: stream={}", streamId);
      account = new Account(); // 초기 상태
      fromVersion = 0;
  }
```

### 4. 스냅샷 스키마 마이그레이션

```
문제: Aggregate 구조 변경 후 구버전 스냅샷

  변경 전 Account:
    { "id": "ACC-001", "balance": 1000000, "status": "ACTIVE" }

  변경 후 Account (새 필드 추가):
    { "id": "ACC-001", "balance": 1000000, "status": "ACTIVE",
      "currency": "KRW",    ← 새 필드
      "tier": "PREMIUM" }   ← 새 필드

  구버전 스냅샷 로드 시:
    currency, tier 필드 없음
    → 역직렬화 후 null
    → NullPointerException 또는 의도치 않은 동작

해결 전략:

전략 1: 기본값 처리 (신규 필드가 nullable or 기본값 있음)
  @JsonSetter(nulls = Nulls.AS_EMPTY)
  private String currency = "KRW"; // null이면 기본값 사용
  → 구버전 스냅샷 자동 호환

전략 2: 스냅샷 버전 관리
  snapshots 테이블에 schema_version 컬럼 추가
  역직렬화 시 schema_version 확인 → 버전별 역직렬화 로직
  → 복잡하지만 완전한 제어

전략 3: 스냅샷 무효화 (가장 단순)
  Aggregate 구조 변경 시 기존 스냅샷을 모두 삭제
  → 다음 로드 시 전체 이벤트 리플레이 (폴백)
  → 새 구조로 새 스냅샷 자동 생성
  → 단점: 변경 직후 일시적 성능 저하
```

---

## 💻 실전 코드

```java
// ==============================
// 스냅샷 패턴 완전 구현
// ==============================

// ✅ Snapshot 저장소
@Repository
public class SnapshotRepository {

    public Optional<AccountSnapshot> findLatest(String streamId) {
        try {
            return Optional.ofNullable(jdbcTemplate.queryForObject("""
                SELECT version, state, created_at
                FROM snapshots
                WHERE stream_id = ?
                ORDER BY version DESC
                LIMIT 1
                """,
                (rs, row) -> new AccountSnapshot(
                    streamId,
                    rs.getLong("version"),
                    rs.getString("state"),
                    rs.getTimestamp("created_at").toInstant()
                ),
                streamId
            ));
        } catch (EmptyResultDataAccessException e) {
            return Optional.empty();
        }
    }

    @Transactional
    public void save(Account account) {
        String state = objectMapper.writeValueAsString(
            AccountSnapshotState.from(account));

        jdbcTemplate.update("""
            INSERT INTO snapshots (stream_id, version, aggregate_type, state)
            VALUES (?, ?, ?, ?::jsonb)
            ON CONFLICT (stream_id, version) DO NOTHING
            """,
            "account-" + account.getId().value(),
            account.getVersion(),
            "Account",
            state
        );
    }
}

// ✅ 스냅샷 상태 DTO — Aggregate 내부와 분리
public record AccountSnapshotState(
    String id,
    String ownerId,
    BigDecimal balance,
    String status,
    int schemaVersion   // 스키마 버전 추적
) {
    static final int CURRENT_SCHEMA_VERSION = 2;

    public static AccountSnapshotState from(Account account) {
        return new AccountSnapshotState(
            account.getId().value(),
            account.getOwnerId().value(),
            account.getBalance().amount(),
            account.getStatus().name(),
            CURRENT_SCHEMA_VERSION
        );
    }
}

// ✅ Account Repository — 스냅샷 통합 로딩
@Repository
public class AccountRepository {

    private static final int SNAPSHOT_THRESHOLD = 100; // 100개마다 스냅샷

    public Optional<Account> findById(AccountId accountId) {
        String streamId = "account-" + accountId.value();

        // 1. 최신 스냅샷 시도
        Optional<AccountSnapshot> snapshot = snapshotRepo.findLatest(streamId);

        Account account;
        long fromVersion;

        if (snapshot.isPresent()) {
            try {
                // 2a. 스냅샷 역직렬화 → 상태 복원
                AccountSnapshotState state = objectMapper.readValue(
                    snapshot.get().state(), AccountSnapshotState.class);

                account = Account.restoreFromSnapshot(state);
                fromVersion = snapshot.get().version();

                log.debug("스냅샷 로드: stream={} version={}", streamId, fromVersion);

            } catch (Exception e) {
                // 역직렬화 실패 → 전체 리플레이 폴백
                log.warn("스냅샷 역직렬화 실패, 전체 리플레이: stream={} error={}",
                    streamId, e.getMessage());
                account = new Account();
                fromVersion = 0;
            }
        } else {
            account = new Account();
            fromVersion = 0;
        }

        // 3. 스냅샷 이후 이벤트 로드 + 리플레이
        List<DomainEvent> events = eventStore
            .loadEventsAfterVersion(streamId, fromVersion).stream()
            .map(se -> serializer.deserialize(se.payload(), se.eventType()))
            .collect(toList());

        if (fromVersion == 0 && events.isEmpty()) return Optional.empty();

        events.forEach(account::apply);
        account.setVersion(fromVersion + events.size());

        return Optional.of(account);
    }

    @Transactional
    public void save(Account account) {
        String streamId = "account-" + account.getId().value();
        List<DomainEvent> pending = account.pullPendingEvents();

        eventStore.appendEvents(streamId,
            account.getVersion() - pending.size(),
            toStoredEvents(pending));

        // N개 이벤트마다 스냅샷 저장
        if (account.getVersion() % SNAPSHOT_THRESHOLD == 0) {
            snapshotRepo.save(account);
            log.info("스냅샷 저장: stream={} version={}", streamId, account.getVersion());
        }
    }
}

// ✅ Account Aggregate — 스냅샷 복원 메서드
public class Account {

    // 스냅샷에서 복원 (전체 리플레이 없이 직접 상태 설정)
    public static Account restoreFromSnapshot(AccountSnapshotState state) {
        Account account = new Account();
        account.id = new AccountId(state.id());
        account.ownerId = new OwnerId(state.ownerId());
        account.balance = Money.of(state.balance());
        account.status = AccountStatus.valueOf(state.status());
        // version은 Repository에서 설정
        return account;
    }

    // ... 나머지 Command 처리, apply() 메서드
}
```

---

## 📊 패턴 비교

```
스냅샷 없음 vs 스냅샷 100개마다 vs 스냅샷 매번:

이벤트 5,000개 스트림, 새 Command 처리 시:

┌─────────────────────┬──────────────┬──────────────┬───────────────┐
│ 항목                │ 스냅샷 없음  │ 100개마다    │ 매 Command    │
├─────────────────────┼──────────────┼──────────────┼───────────────┤
│ 로드할 이벤트 수     │ 5,000개      │ 최대 99개    │ 0개           │
├─────────────────────┼──────────────┼──────────────┼───────────────┤
│ 로딩 시간 (참고)    │ ~500ms       │ ~10ms        │ ~5ms          │
├─────────────────────┼──────────────┼──────────────┼───────────────┤
│ 스냅샷 저장 빈도    │ 없음         │ 100회당 1회  │ 매 Command    │
├─────────────────────┼──────────────┼──────────────┼───────────────┤
│ 스냅샷 저장 비용    │ 없음         │ 낮음         │ 매우 높음     │
├─────────────────────┼──────────────┼──────────────┼───────────────┤
│ 구조 변경 영향      │ 없음         │ 스냅샷 무효화│ 스냅샷 무효화  │
└─────────────────────┴──────────────┴──────────────┴───────────────┘

권장: 100개마다 — 로딩 성능과 저장 비용의 균형
```

---

## ⚖️ 트레이드오프

```
스냅샷의 이점:
  이벤트 로딩 시간 O(N) → O(M) (M: 스냅샷 이후 이벤트 수)
  Command 처리 지연 감소
  DB I/O 감소

스냅샷의 비용:
  스냅샷 저장소 추가 관리
  Aggregate 구조 변경 시 스냅샷 무효화 또는 마이그레이션
  폴백 로직 구현 필요 (역직렬화 실패 처리)
  스냅샷과 이벤트 스토어 동기화 고려

스냅샷 스키마 문제:
  이벤트는 Upcasting으로 버전 관리 (다음 문서)
  스냅샷은 현재 Aggregate 구조와 강하게 결합
  → Aggregate 구조 변경 시 스냅샷 마이그레이션 또는 무효화 필요
  → 이벤트보다 스냅샷 스키마 관리가 더 어렵다는 주장도 있음
```

---

## 📌 핵심 정리

```
스냅샷 패턴 핵심:

목적:
  이벤트 리플레이 비용 O(전체 이벤트 수) → O(스냅샷 이후 이벤트 수)
  수천 개 이벤트 → 최악 N-1개 리플레이

로딩 전략:
  1. 최신 스냅샷 로드 (없으면 초기 상태)
  2. 스냅샷 이후 이벤트만 로드
  3. 이벤트 리플레이 → 현재 상태
  4. 역직렬화 실패 → 전체 리플레이 폴백

저장 시점:
  N개 이벤트마다 (권장: N=50~200)
  배치로 비동기 저장 (Command 처리 경로 제외)

스키마 변경 대응:
  신규 필드: 기본값 처리
  대규모 변경: 스냅샷 무효화 + 전체 리플레이 폴백
  schema_version 필드로 버전 추적

임계값 결정:
  목표 로딩 시간 → 최대 허용 이벤트 수 → N 결정
```

---

## 🤔 생각해볼 문제

**Q1.** 스냅샷을 Command 처리 경로에서 동기적으로 저장하면 어떤 문제가 생기는가?

<details>
<summary>해설 보기</summary>

Command 처리 지연이 증가합니다. 스냅샷 저장은 직렬화 + DB INSERT 비용이 있어, 스냅샷을 찍는 Command는 그렇지 않은 Command보다 수십 ms 더 걸립니다. N개마다 1번 이 비용이 발생하면 주기적인 지연 스파이크가 생깁니다.

트랜잭션 범위 문제도 있습니다. 이벤트 저장과 스냅샷 저장을 같은 트랜잭션에 묶으면 이벤트 저장이 빠른데 스냅샷 저장이 느려 트랜잭션 시간이 늘어납니다. 다른 트랜잭션에 묶으면 이벤트는 저장됐지만 스냅샷이 실패할 수 있습니다.

비동기 배치 스냅샷이 더 좋은 이유입니다. Command 처리 경로에서 스냅샷을 분리하면 Command 처리 지연에 영향이 없습니다. 스냅샷이 약간 오래됐어도(최대 1시간) 이후 이벤트를 리플레이하면 되므로 정확성에 문제 없습니다.

</details>

---

**Q2.** 같은 Aggregate에 대해 동시에 두 Command가 처리되는 중에 하나의 Command 처리에서 스냅샷이 저장된다. 다른 Command 처리에서 이 스냅샷을 로드하면 어떤 문제가 생기는가?

<details>
<summary>해설 보기</summary>

이 상황은 실제로 발생할 수 있지만, 올바르게 설계된 시스템에서는 문제가 되지 않습니다.

CQRS + Event Sourcing에서 동시 Command는 낙관적 잠금으로 처리됩니다. 두 Command 중 하나가 먼저 이벤트를 저장하고 커밋합니다. 다른 Command는 낙관적 잠금 충돌로 실패하고 재시도합니다.

재시도 시 스냅샷 로드는 다음과 같이 동작합니다. 재시도 때 스냅샷을 다시 로드하면 (새로 저장된 스냅샷 또는 기존 스냅샷 중 최신 것) + 그 이후 이벤트를 리플레이해 정확한 현재 상태를 얻습니다. 스냅샷이 잘못된 상태를 담고 있어도, 이후 이벤트 리플레이가 그것을 덮어쓰므로 최종 상태는 항상 정확합니다.

따라서 스냅샷은 "캐시"처럼 동작합니다. 스냅샷이 오래됐어도 이후 이벤트로 보정되므로, 스냅샷 자체의 정확성보다 이벤트 스토어의 정확성이 더 중요합니다.

</details>

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[이전: Aggregate 재구성 ⬅️](./03-aggregate-reconstitution.md)** | **[다음: 이벤트 스키마 진화 ➡️](./05-event-schema-evolution.md)**

</div>
