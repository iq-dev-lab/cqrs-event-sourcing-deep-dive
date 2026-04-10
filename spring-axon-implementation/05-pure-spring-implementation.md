# Axon 없이 직접 구현

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Axon이 자동으로 해주는 것들을 직접 구현하려면 무엇이 필요한가?
- PostgreSQL로 이벤트 스토어를 직접 구현할 때 낙관적 잠금은 어떻게 처리하는가?
- Kafka 기반 프로젝션 업데이트를 직접 구현할 때 체크포인트 관리는 어떻게 하는가?
- Command Bus를 직접 구현하면 어떤 구조가 되는가?
- Axon vs 직접 구현 중 어느 쪽이 더 나은 선택인가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

Axon을 도입하지 않거나 도입하지 못하는 팀이 많다. 학습 비용, Axon Server 운영 비용, 기존 팀의 익숙한 Spring 스택 등 이유는 다양하다. 순수 Spring으로 CQRS를 구현하면 Axon이 추상화해주는 부분을 직접 만들어야 하지만, 각 컴포넌트가 어떻게 동작하는지 완전히 이해할 수 있다. 이 문서는 Axon 없이 완전한 CQRS + Event Sourcing 구현을 보여준다.

---

## 😱 흔한 실수 (Before — 직접 구현 시 놓치는 것들)

```
실수 1: 이벤트 스토어에 낙관적 잠금 없음

  jdbcTemplate.update(
      "INSERT INTO event_store (stream_id, version, event_type, payload) VALUES (?, ?, ?, ?)",
      streamId, currentVersion + 1, eventType, payload
  );
  // UNIQUE(stream_id, version) 제약 없음 → 동시 저장 시 중복 버전 가능
  // → 이벤트 순서 보장 없음 → Aggregate 재구성 오류

실수 2: 체크포인트와 읽기 모델 업데이트가 별도 트랜잭션

  readModelRepo.update(event);      // 트랜잭션 1
  checkpointRepo.save(seq);         // 트랜잭션 2 (실패 가능)
  // 읽기 모델 업데이트됨 + 체크포인트 저장 실패
  // 재시작 시 이미 처리한 이벤트 재처리 → 중복 업데이트

실수 3: 역직렬화 실패를 처리하지 않음

  DomainEvent event = objectMapper.readValue(payload, DomainEvent.class);
  // 알 수 없는 이벤트 타입 → JsonMappingException
  // 예외 처리 없음 → 프로젝션 멈춤
```

---

## ✨ 올바른 접근 (After — 순수 Spring CQRS 직접 구현)

```
구현해야 할 컴포넌트:

  1. PostgreSQL 이벤트 스토어
     - UNIQUE(stream_id, version) 낙관적 잠금
     - 이벤트 로드 (stream_id, version ASC)
     - 이벤트 폴링 (global_seq 기반)

  2. Command Bus
     - Command 타입 → Handler 자동 매핑
     - 미들웨어 체인 (로깅, 멱등성, 트랜잭션)

  3. Aggregate Repository
     - 이벤트 스토어에서 Aggregate 재구성
     - 새 이벤트 저장 + Outbox 저장

  4. Projection Runner
     - 이벤트 스토어 폴링 (또는 Kafka Consumer)
     - 체크포인트 관리 (읽기 모델 업데이트와 원자적)
     - DLQ 처리

  5. Query Handler Registry
     - Query 타입 → Handler 매핑
```

---

## 🔬 내부 동작 원리

### 1. Axon이 자동으로 해주는 것 vs 직접 구현

```
Axon 자동 처리 항목 → 직접 구현 필요:

@CommandHandler 메서드 스캔 및 등록
  → CommandBus에 Command 타입별 Handler 맵 직접 관리

Aggregate 재구성 (EventSourcedRepository)
  → 이벤트 스토어에서 이벤트 로드 + apply() 반복 호출 직접 구현

AggregateLifecycle.apply() 
  → pendingEvents 리스트에 추가 + @EventSourcingHandler 호출 직접 구현

Unit of Work (트랜잭션 관리)
  → @Transactional로 대체 (Spring 트랜잭션)

Tracking Event Processor
  → 이벤트 스토어 폴링 + 체크포인트 관리 직접 구현

Segment 병렬 처리
  → Kafka 파티션 키(stream_id) + Consumer Group으로 대체

QueryBus + @QueryHandler 스캔
  → QueryHandler 맵 직접 관리

Subscription Query
  → SSE or WebSocket으로 직접 구현
```

### 2. Command Bus 직접 구현

```
Command Bus 구현 핵심:

  interface Command {}
  interface CommandHandler<C extends Command> {
      void handle(C command);
  }

  @Component
  public class CommandBus {
      private final Map<Class<?>, CommandHandler<?>> handlers = new HashMap<>();

      // Spring Context에서 모든 CommandHandler 자동 수집
      @Autowired
      public void registerHandlers(List<CommandHandler<?>> handlerList) {
          handlerList.forEach(handler -> {
              Class<?> commandType = extractCommandType(handler);
              handlers.put(commandType, handler);
              log.info("Handler 등록: {} → {}", commandType.getSimpleName(),
                  handler.getClass().getSimpleName());
          });
      }

      @SuppressWarnings("unchecked")
      public <C extends Command> void send(C command) {
          CommandHandler<C> handler = (CommandHandler<C>) handlers.get(command.getClass());
          if (handler == null)
              throw new NoHandlerFoundException(command.getClass());
          handler.handle(command);
      }
  }
```

### 3. 이벤트 스토어 + Aggregate Repository

```
이벤트 스토어 + Aggregate 재구성 핵심:

  이벤트 로드:
    SELECT * FROM event_store
    WHERE stream_id = 'account-ACC-001'
    ORDER BY version ASC

  Aggregate 재구성:
    Account account = new Account(); // 빈 생성자
    events.forEach(event -> account.apply(deserialize(event)));
    account.setVersion(events.size());

  새 이벤트 저장:
    expectedVersion = currentVersion (낙관적 잠금 기준)
    새 이벤트들: version = expectedVersion + 1, + 2, ...

    INSERT INTO event_store (stream_id, version, event_type, payload)
    VALUES ('account-ACC-001', 4, 'MoneyWithdrawn', '...')
    -- UNIQUE(stream_id, version) 위반 시 → 낙관적 잠금 충돌

  Outbox 저장 (같은 트랜잭션):
    INSERT INTO outbox (event_id, stream_id, event_type, payload)
    VALUES (...) -- Kafka 발행용
```

---

## 💻 실전 코드

```java
// ✅ 완전한 순수 Spring CQRS + ES 구현

// ── 이벤트 스토어 ──────────────────────────────────────────────
@Repository
public class PostgreSqlEventStore {

    @Transactional
    public void appendEvents(String streamId, long expectedVersion,
                              List<StoredEvent> events) {
        // 현재 버전 확인
        Long currentVersion = jdbcTemplate.queryForObject(
            "SELECT COALESCE(MAX(version), 0) FROM event_store WHERE stream_id = ?",
            Long.class, streamId);

        if (!Objects.equals(currentVersion, expectedVersion))
            throw new OptimisticConcurrencyException(streamId, expectedVersion, currentVersion);

        // 이벤트 저장 + Outbox 저장 (같은 트랜잭션)
        long nextVersion = expectedVersion;
        for (StoredEvent event : events) {
            nextVersion++;
            try {
                jdbcTemplate.update("""
                    INSERT INTO event_store
                        (stream_id, version, event_type, payload, occurred_at)
                    VALUES (?, ?, ?, ?::jsonb, NOW())
                    """,
                    streamId, nextVersion,
                    event.eventType(),
                    objectMapper.writeValueAsString(event.payload())
                );
            } catch (DuplicateKeyException e) {
                throw new OptimisticConcurrencyException(streamId, nextVersion);
            }

            // Outbox에도 저장 (Kafka 발행용)
            jdbcTemplate.update("""
                INSERT INTO outbox (event_id, stream_id, event_type, payload)
                VALUES (?, ?, ?, ?::jsonb)
                """,
                UUID.randomUUID().toString(), streamId,
                event.eventType(),
                objectMapper.writeValueAsString(event.payload())
            );
        }
    }

    public List<StoredEvent> loadEvents(String streamId) {
        return jdbcTemplate.query("""
            SELECT version, event_type, payload::text, occurred_at
            FROM event_store WHERE stream_id = ? ORDER BY version ASC
            """,
            (rs, row) -> new StoredEvent(
                rs.getLong("version"),
                rs.getString("event_type"),
                readPayload(rs.getString("payload")),
                rs.getTimestamp("occurred_at").toInstant()
            ),
            streamId
        );
    }

    public List<StoredEvent> loadBatch(long afterSeq, int limit) {
        return jdbcTemplate.query("""
            SELECT global_seq, stream_id, event_type, payload::text, occurred_at
            FROM event_store WHERE global_seq > ? ORDER BY global_seq ASC LIMIT ?
            """,
            (rs, row) -> new StoredEvent(
                rs.getLong("global_seq"),
                rs.getString("stream_id"),
                rs.getString("event_type"),
                readPayload(rs.getString("payload")),
                rs.getTimestamp("occurred_at").toInstant()
            ),
            afterSeq, limit
        );
    }
}

// ── Aggregate Repository ────────────────────────────────────────
@Repository
public class AccountRepository {

    private final PostgreSqlEventStore eventStore;
    private final SnapshotRepository snapshotRepo;
    private final EventSerializer serializer;

    public Optional<Account> findById(AccountId accountId) {
        String streamId = "account-" + accountId.value();

        // 스냅샷 로드 시도
        Optional<AccountSnapshot> snapshot = snapshotRepo.findLatest(streamId);
        Account account;
        long fromVersion;

        if (snapshot.isPresent()) {
            try {
                account = deserializeSnapshot(snapshot.get());
                fromVersion = snapshot.get().version();
            } catch (Exception e) {
                log.warn("스냅샷 역직렬화 실패, 전체 리플레이: stream={}", streamId);
                account = new Account();
                fromVersion = 0;
            }
        } else {
            account = new Account();
            fromVersion = 0;
        }

        // 스냅샷 이후 이벤트 로드 + 재구성
        List<StoredEvent> events = fromVersion == 0
            ? eventStore.loadEvents(streamId)
            : eventStore.loadEventsAfterVersion(streamId, fromVersion);

        if (fromVersion == 0 && events.isEmpty()) return Optional.empty();

        events.forEach(stored -> {
            DomainEvent event = serializer.deserialize(stored);
            account.apply(event);
        });
        account.setVersion(fromVersion + events.size());

        return Optional.of(account);
    }

    @Transactional
    public void save(Account account) {
        String streamId = "account-" + account.getId().value();
        List<DomainEvent> pending = account.pullPendingEvents();
        long expectedVersion = account.getVersion() - pending.size();

        List<StoredEvent> toStore = pending.stream()
            .map(serializer::serialize)
            .collect(toList());

        eventStore.appendEvents(streamId, expectedVersion, toStore);

        // N개 이벤트마다 스냅샷
        if (account.getVersion() % 100 == 0)
            snapshotRepo.save(account);
    }
}

// ── Projection Runner ────────────────────────────────────────────
@Component
public class AccountSummaryProjectionRunner {

    @Scheduled(fixedDelay = 500) // 500ms 폴링
    @Transactional
    public void run() {
        long lastSeq = checkpointRepo.load("AccountSummaryProjection");

        List<StoredEvent> events = eventStore.loadBatch(lastSeq, 100);
        if (events.isEmpty()) return;

        for (StoredEvent event : events) {
            try {
                processEvent(event);
                checkpointRepo.save("AccountSummaryProjection", event.globalSeq());
                // 읽기 모델 + 체크포인트 = 같은 트랜잭션
            } catch (Exception e) {
                log.error("Projection 처리 실패: seq={} error={}", event.globalSeq(), e.getMessage());
                dlqService.send(event, e.getMessage());
                break; // 오류 이벤트에서 중단 (다음 폴링에서 재시도)
            }
        }
    }

    private void processEvent(StoredEvent event) {
        DomainEvent domainEvent;
        try {
            domainEvent = serializer.deserialize(event);
        } catch (Exception e) {
            log.warn("역직렬화 실패, 건너뜀: type={}", event.eventType());
            return; // 알 수 없는 이벤트 → 무시
        }

        switch (domainEvent) {
            case AccountCreated e -> onAccountCreated(e, event.globalSeq());
            case MoneyDeposited e -> onMoneyDeposited(e, event.globalSeq());
            case MoneyWithdrawn e -> onMoneyWithdrawn(e, event.globalSeq());
            default -> {} // 관심 없는 이벤트 무시
        }
    }

    @Transactional
    private void onAccountCreated(AccountCreated event, long seq) {
        jdbcTemplate.update("""
            INSERT INTO account_summary
                (account_id, owner_id, balance, status, last_event_seq, created_at)
            VALUES (?, ?, ?, 'ACTIVE', ?, NOW())
            ON CONFLICT (account_id) DO UPDATE SET
                balance = EXCLUDED.balance,
                last_event_seq = EXCLUDED.last_event_seq
            WHERE account_summary.last_event_seq < EXCLUDED.last_event_seq
            """,
            event.accountId(), event.ownerId(), event.initialDeposit(), seq
        );
    }
}
```

---

## 📊 패턴 비교

```
Axon Framework vs 직접 구현:

┌──────────────────────────┬──────────────┬──────────────────┐
│ 항목                     │ Axon         │ 직접 구현        │
├──────────────────────────┼──────────────┼──────────────────┤
│ Aggregate 재구성         │ 자동         │ 직접 구현        │
├──────────────────────────┼──────────────┼──────────────────┤
│ Command 라우팅           │ 자동 스캔    │ 맵 직접 관리     │
├──────────────────────────┼──────────────┼──────────────────┤
│ 이벤트 스토어            │ Axon Server  │ PostgreSQL 직접  │
├──────────────────────────┼──────────────┼──────────────────┤
│ Projection 병렬 처리     │ Segment 자동 │ Kafka 파티션     │
├──────────────────────────┼──────────────┼──────────────────┤
│ Projection 재구축        │ UI 클릭      │ API 직접 구현    │
├──────────────────────────┼──────────────┼──────────────────┤
│ Subscription Query       │ 내장         │ SSE 직접 구현    │
├──────────────────────────┼──────────────┼──────────────────┤
│ Saga                     │ @Saga 자동   │ State Machine    │
├──────────────────────────┼──────────────┼──────────────────┤
│ 학습 비용                │ 높음         │ 중간             │
├──────────────────────────┼──────────────┼──────────────────┤
│ 운영 복잡도              │ Axon Server  │ 없음(or Kafka)   │
└──────────────────────────┴──────────────┴──────────────────┘

선택 기준:
  팀이 Axon 경험 있음 + 분산 환경 → Axon
  단일 서비스 + Spring 익숙 + 간단한 ES → 직접 구현
  사이에: Spring + In-Process Axon (Axon Server 없이)
```

---

## ⚖️ 트레이드오프

```
직접 구현의 장점:
  Axon 의존성 없음 → 스택 단순화
  각 컴포넌트를 완전히 이해하고 제어
  Axon Server 운영 비용 없음
  디버깅 용이 (Axon 추상화 없음)

직접 구현의 단점:
  Axon이 자동화해주는 것을 모두 직접 구현
  Saga, Subscription Query 같은 복잡한 패턴 구현 부담
  프로젝션 재구축 UI 없음
  코드량 증가
```

---

## 📌 핵심 정리

```
순수 Spring CQRS + ES 직접 구현 핵심:

이벤트 스토어:
  UNIQUE(stream_id, version): 낙관적 잠금 필수
  Outbox 패턴: 이벤트 스토어 + Outbox 같은 트랜잭션
  global_seq: Projection 폴링 기준

Aggregate Repository:
  이벤트 로드 → apply() 반복 → 현재 상태
  스냅샷 + 이후 이벤트로 성능 최적화

Projection Runner:
  500ms 폴링 (또는 Kafka Consumer)
  체크포인트 + 읽기 모델 = 같은 트랜잭션
  DLQ 처리로 장애 격리

Command Bus:
  List<CommandHandler> 자동 수집
  Command 타입 → Handler 맵
  미들웨어 체인 (로깅, 멱등성)

vs Axon:
  단순 구현: 직접 구현 충분
  분산 환경 + Saga + Subscription Query: Axon 권장
```

---

## 🤔 생각해볼 문제

**Q1.** 직접 구현한 Projection Runner가 폴링 방식을 사용할 때, 이벤트 발행과 Projection 처리 사이의 지연을 최소화하려면 어떻게 해야 하는가?

<details>
<summary>해설 보기</summary>

폴링 지연을 줄이는 방법이 여러 가지 있습니다.

폴링 주기를 줄입니다. `@Scheduled(fixedDelay = 100)`처럼 100ms로 줄이면 평균 지연이 절반으로 줄어듭니다. 단, DB 폴링 빈도가 높아져 DB 부하가 증가합니다.

PostgreSQL `LISTEN/NOTIFY`를 활용합니다. 이벤트 저장 시 `NOTIFY account_events`를 발행하고, Projection Runner가 `LISTEN account_events`로 구독합니다. 새 이벤트가 있을 때만 폴링을 트리거하므로 불필요한 폴링이 없습니다.

Kafka를 Outbox 발행 채널로 사용합니다. Outbox Publisher가 이벤트를 Kafka에 발행하면 Projection이 Kafka Consumer로 거의 실시간으로 처리합니다. 폴링 대신 Push 방식이므로 지연이 크게 줄어듭니다.

단순하게 시작한다면 `LISTEN/NOTIFY`가 추가 인프라 없이 지연을 줄이는 좋은 방법입니다.

</details>

---

**Q2.** Axon의 `AggregateLifecycle.apply()` 없이 직접 구현할 때, `pendingEvents`를 어떻게 관리하는가?

<details>
<summary>해설 보기</summary>

`pendingEvents` 리스트를 Aggregate 내부에 직접 관리합니다.

```java
public class Account {
    private final List<DomainEvent> pendingEvents = new ArrayList<>();
    
    protected void applyAndRecord(DomainEvent event) {
        apply(event);            // 즉시 상태 업데이트 (@EventSourcingHandler 역할)
        pendingEvents.add(event); // 나중에 이벤트 스토어에 저장
    }
    
    // Repository에서 저장 후 호출
    public List<DomainEvent> pullPendingEvents() {
        List<DomainEvent> events = List.copyOf(pendingEvents);
        pendingEvents.clear();
        return events;
    }
}
```

Repository에서 `save()` 호출 시 `pullPendingEvents()`로 미저장 이벤트를 가져와 이벤트 스토어에 저장합니다. 이 흐름이 Axon의 Unit of Work 역할을 합니다.

Aggregate 재구성 시에는 `applyAndRecord()` 대신 `apply()`만 호출합니다. 재구성은 이미 저장된 이벤트를 다시 적용하는 것이므로 `pendingEvents`에 추가하면 안 됩니다.

재구성용 `reconstitute()` 메서드를 별도로 만들어 명확히 분리하는 것이 좋습니다.

</details>

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[이전: Query 처리 ⬅️](./04-query-handling.md)** | **[다음: Chapter 7 — 은행 계좌 도메인 구현 ➡️](../real-world-tradeoffs/01-bank-account-domain.md)**

</div>
