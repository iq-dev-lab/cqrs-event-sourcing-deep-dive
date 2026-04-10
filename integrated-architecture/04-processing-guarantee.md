# 처리 보장(Processing Guarantee)

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- At-Least-Once, At-Most-Once, Exactly-Once의 차이는 무엇인가?
- Kafka 환경에서 Exactly-Once 프로젝션 업데이트를 구현하려면 무엇이 필요한가?
- 이벤트 중복 처리를 방지하는 멱등키 기반 설계는 어떻게 동작하는가?
- Offset 기반 체크포인트 관리에서 "At-Least-Once + 멱등 처리"가 Exactly-Once와 동등한 이유는 무엇인가?
- 실제 운영에서 처리 보장 수준을 어떻게 선택하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

Kafka는 기본적으로 At-Least-Once 전달을 보장한다. 즉, 이벤트가 최소 한 번은 전달되지만 네트워크 장애나 재시작 시 중복 전달될 수 있다. 프로젝션이 이벤트를 두 번 처리하면 읽기 모델에 잘못된 값이 들어갈 수 있다. 잔고가 두 번 차감되거나, 카운터가 두 번 증가하거나, 주문이 두 번 생성되는 상황이다. 이 문제를 해결하는 방법이 멱등 처리이고, 이것이 Exactly-Once와 동등한 효과를 만든다.

---

## 😱 흔한 실수 (Before — 멱등성 없는 프로젝션)

```
실수 1: 중복 이벤트가 그대로 적용됨

  @KafkaListener(topics = "account-events")
  public void on(MoneyDeposited event) {
      // 이벤트가 두 번 처리되면?
      jdbcTemplate.update(
          "UPDATE account_summary SET balance = balance + ? WHERE account_id = ?",
          event.getAmount(), event.getAccountId()
      );
      // ❌ 잔고가 두 번 증가 (50만원 입금 → 100만원 증가)
  }

실수 2: 카운터가 두 번 증가

  @EventHandler
  public void on(OrderConfirmed event) {
      monthlyStats.incrementConfirmedCount(); // ❌ 두 번 호출 시 2 증가
      monthlyStats.addRevenue(event.getAmount()); // ❌ 두 번 더해짐
  }

실수 3: 오프셋 커밋 후 처리 → 이벤트 유실

  // At-Most-Once 패턴 (더 나쁨)
  acknowledgment.acknowledge();   // 오프셋 먼저 커밋
  updateReadModel(event);         // 처리 중 실패 → 이벤트 유실 (다시 받지 못함)
```

---

## ✨ 올바른 접근 (After — At-Least-Once + 멱등 처리)

```
처리 보장 전략:

At-Most-Once (오프셋 먼저 커밋):
  이벤트 유실 가능 (처리 실패해도 다시 받지 못함)
  → 금융/주문 시스템에서 절대 사용 금지

At-Least-Once (처리 완료 후 오프셋 커밋):
  중복 처리 가능 (재시작 시 마지막 오프셋부터 재처리)
  + 멱등 처리 → Exactly-Once 효과

Exactly-Once (Kafka Transactions):
  Kafka와 외부 DB를 단일 트랜잭션으로 묶음
  → Kafka Streams 환경에서 지원, 일반 Consumer에서 복잡
  → 성능 오버헤드 있음

실무 권장:
  At-Least-Once + 멱등 프로젝션 = Exactly-Once 효과
  → 구현 단순, 성능 우수, 안전
```

---

## 🔬 내부 동작 원리

### 1. 오프셋 커밋 타이밍의 중요성

```
오프셋 커밋 = "이 이벤트까지 처리 완료"를 Kafka에 알리는 것

At-Most-Once (오프셋 먼저 커밋):

  T=0   offset=100 이벤트 수신
  T=1   acknowledgment.acknowledge()  ← 오프셋 101 커밋
  T=2   updateReadModel(event)        ← 처리 실패 (DB 오류)
  T=3   Consumer 재시작
  T=4   offset=101부터 다시 받음
  → offset=100 이벤트 유실!

At-Least-Once (처리 완료 후 커밋):

  T=0   offset=100 이벤트 수신
  T=1   updateReadModel(event)        ← 처리 완료
  T=2   acknowledgment.acknowledge()  ← 오프셋 101 커밋
  T=3   Consumer 재시작 (이 시나리오에서는 정상)

  재시작 시나리오:
  T=0   offset=100 이벤트 수신
  T=1   updateReadModel(event)        ← 처리 완료
  T=2   프로세스 크래시 (커밋 전)
  T=3   Consumer 재시작
  T=4   offset=100부터 다시 받음     ← 중복!
  T=5   updateReadModel(event)        ← 두 번 처리!
  → 멱등 처리가 없으면 문제

결론:
  At-Least-Once + 멱등 처리 = 안전
  At-Most-Once = 이벤트 유실 위험 → 금지
```

### 2. 멱등키 기반 중복 방지

```
전략 1 — 이벤트 ID 기반 중복 감지:

  processed_events 테이블:
    event_id   VARCHAR(36) PRIMARY KEY
    processed_at TIMESTAMPTZ
    projection_name VARCHAR(100)

  처리 흐름:
    이벤트 수신 (eventId = "EVT-001")
    → processed_events에서 조회
    → 있음: 중복 → 건너뜀
    → 없음: 처리 → processed_events에 기록 (같은 트랜잭션)

  @Transactional
  public void handle(StoredEvent event) {
      if (processedEventRepo.existsById(event.eventId())) {
          log.debug("중복 이벤트 무시: {}", event.eventId());
          return;
      }
      updateReadModel(event);
      processedEventRepo.save(event.eventId());
      // ← 읽기 모델 업데이트 + 처리 기록이 원자적
  }

전략 2 — Upsert로 자연스럽게 멱등 처리:

  // INSERT ON CONFLICT: 이미 있으면 UPDATE (덮어쓰기)
  INSERT INTO account_summary (account_id, balance, last_event_seq)
  VALUES ('ACC-001', 400000, 16)
  ON CONFLICT (account_id) DO UPDATE SET
      balance = EXCLUDED.balance,
      last_event_seq = EXCLUDED.last_event_seq
  WHERE account_summary.last_event_seq < EXCLUDED.last_event_seq;

  // last_event_seq 조건: 더 오래된 이벤트로 덮어쓰기 방지
  // 같은 이벤트 두 번 처리 → 두 번째는 WHERE 조건 미충족 → 업데이트 없음

전략 비교:
  이벤트 ID 기반:
    장점: 완전한 중복 방지, 복잡한 업데이트에도 적용 가능
    단점: processed_events 테이블 크기 증가

  Upsert 기반:
    장점: 단순, 추가 테이블 불필요
    단점: 카운터 증가 같은 누적 연산에 직접 적용 어려움
```

### 3. 카운터/집계 연산의 멱등 처리

```
문제: 카운터 증가는 멱등하지 않음

  // balance += amount 는 멱등하지 않음
  // 같은 이벤트 두 번 처리 → 두 번 증가

해결 1 — 절대값으로 저장:

  // ❌ 누적 방식 (멱등 불가)
  UPDATE account_summary SET balance = balance + 100000 WHERE account_id = ?

  // ✅ 절대값 방식 (멱등)
  UPDATE account_summary SET balance = 400000 WHERE account_id = ?
  // 이벤트 페이로드에 balanceAfter(400000) 포함 → 절대값으로 업데이트

  이벤트 설계 핵심:
    MoneyDeposited { amount: 100000, balanceAfter: 400000 }
    → balanceAfter를 이벤트에 포함 → 읽기 모델에 직접 저장
    → 중복 처리 시 동일 값으로 업데이트 → 안전

해결 2 — 집계 연산을 배치로 재계산:

  // 매월 매출 집계를 실시간 누적 대신 주기적 재계산
  @Scheduled(cron = "0 0 * * * *")
  void recalculateMonthlyRevenue() {
      BigDecimal total = eventStore.sumAmounts(
          "OrderConfirmed", YearMonth.now());
      revenueStats.update(YearMonth.now(), total);
      // 재계산이므로 중복 처리 문제없음
  }

해결 3 — 처리 기록 + 보정 이벤트:

  // 집계 테이블에 각 이벤트 기여분을 별도 행으로 저장
  INSERT INTO revenue_contributions (event_id, year_month, amount)
  VALUES ('EVT-001', '2024-01', 150000)
  ON CONFLICT (event_id) DO NOTHING  ← 중복 이벤트 무시

  // 집계는 contributions 테이블 SUM으로 계산
  SELECT SUM(amount) FROM revenue_contributions WHERE year_month = '2024-01'
  → 중복 삽입해도 ON CONFLICT DO NOTHING으로 안전
```

### 4. Offset 체크포인트 관리 — DB vs Kafka

```
방법 1 — Kafka Consumer Group 오프셋:

  Kafka 자체 오프셋 관리:
    consumer.commitSync()  → Kafka에 오프셋 저장
    재시작 시 마지막 커밋 오프셋부터 재처리

  장점: 추가 저장소 없음, Kafka 네이티브
  단점: 읽기 모델 업데이트와 오프셋 커밋의 원자성 없음
        처리 완료 → 오프셋 커밋 사이 크래시 → 중복 처리

방법 2 — DB 체크포인트 (읽기 모델 DB에 저장):

  projection_checkpoints 테이블:
    projection_name, last_offset (Kafka offset or global_seq)

  @Transactional
  public void handle(ConsumerRecord record, Acknowledgment ack) {
      updateReadModel(event);  // 읽기 모델 업데이트

      // 같은 DB 트랜잭션에 체크포인트 저장
      checkpointRepo.save(projectionName, record.offset());

      ack.acknowledge();  // Kafka 오프셋 커밋 (포기 가능 — DB가 진실)
  }

  재시작 시:
    DB에서 last_offset 로드 → 해당 오프셋부터 재처리 요청
    (Kafka에 seek 요청)

  장점: 읽기 모델 업데이트 + 체크포인트가 원자적 → 정확한 한 번 처리
  단점: DB 체크포인트 읽기 + seek 로직 추가 구현 필요

  @Bean
  public void seekToSavedOffsets(ConsumerSeekAware.ConsumerSeekCallback callback) {
      Map<TopicPartition, Long> savedOffsets = checkpointRepo.loadAll(projectionName);
      savedOffsets.forEach((tp, offset) -> callback.seek(tp.topic(), tp.partition(), offset + 1));
  }
```

---

## 💻 실전 코드

```java
// ✅ Exactly-Once 효과 — DB 체크포인트 + 멱등 Upsert
@Component
public class AccountSummaryProjection implements ConsumerSeekAware {

    private final CheckpointRepository checkpointRepo;
    private final JdbcTemplate jdbcTemplate;
    private static final String NAME = "AccountSummaryProjection";

    // 재시작 시 저장된 오프셋으로 seek
    @Override
    public void onPartitionsAssigned(Map<TopicPartition, Long> assignments,
                                      ConsumerSeekCallback callback) {
        assignments.keySet().forEach(tp -> {
            long savedOffset = checkpointRepo.load(NAME, tp.topic(), tp.partition());
            if (savedOffset >= 0) {
                callback.seek(tp.topic(), tp.partition(), savedOffset + 1);
                log.info("Seek to saved offset: {} partition={} offset={}",
                    NAME, tp.partition(), savedOffset + 1);
            }
        });
    }

    @KafkaListener(topics = "account-events", groupId = "account-summary-projection")
    @Transactional
    public void handle(ConsumerRecord<String, String> record, Acknowledgment ack) {
        try {
            StoredEvent stored = deserialize(record.value());
            DomainEvent event = upcasterChain.apply(stored);

            // 멱등 처리
            switch (event) {
                case MoneyDeposited e  -> onDeposited(e, stored.eventSeq());
                case MoneyWithdrawn e  -> onWithdrawn(e, stored.eventSeq());
                case MoneyTransferred e -> onTransferred(e, stored.eventSeq());
                default -> {} // 무관심 이벤트 무시
            }

            // 체크포인트 저장 (같은 트랜잭션)
            checkpointRepo.save(NAME,
                record.topic(), record.partition(), record.offset());

            ack.acknowledge();

        } catch (Exception e) {
            log.error("처리 실패: topic={} partition={} offset={} error={}",
                record.topic(), record.partition(), record.offset(), e.getMessage());
            // 트랜잭션 롤백 → 체크포인트 미저장 → 재시작 시 재처리
            throw e;
        }
    }

    // ✅ 절대값 기반 멱등 Upsert
    private void onDeposited(MoneyDeposited event, long seq) {
        int updated = jdbcTemplate.update("""
            INSERT INTO account_summary (account_id, balance, last_event_seq, updated_at)
            VALUES (?, ?, ?, NOW())
            ON CONFLICT (account_id) DO UPDATE SET
                balance = EXCLUDED.balance,
                last_event_seq = EXCLUDED.last_event_seq,
                updated_at = NOW()
            WHERE account_summary.last_event_seq < EXCLUDED.last_event_seq
            """,
            event.accountId(),
            event.balanceAfter(),  // 절대값 (누적 아님)
            seq
        );
        if (updated == 0)
            log.debug("중복 이벤트 무시 (멱등): accountId={} seq={}", event.accountId(), seq);
    }

    // ✅ 카운터 집계 — event_id 기반 중복 방지
    private void recordRevenueContribution(OrderConfirmed event) {
        jdbcTemplate.update("""
            INSERT INTO revenue_contributions (event_id, year_month, amount)
            VALUES (?, ?, ?)
            ON CONFLICT (event_id) DO NOTHING
            """,
            event.eventId(),
            YearMonth.from(event.occurredAt().atZone(ZoneId.systemDefault())),
            event.totalAmount()
        );
        // 중복 삽입 시 DO NOTHING → 집계에 두 번 포함 안 됨
    }
}

// ✅ 체크포인트 Repository
@Repository
public class CheckpointRepository {

    public long load(String projectionName, String topic, int partition) {
        try {
            return jdbcTemplate.queryForObject("""
                SELECT last_offset FROM projection_checkpoints
                WHERE projection_name = ? AND topic = ? AND partition_num = ?
                """,
                Long.class,
                projectionName, topic, partition);
        } catch (EmptyResultDataAccessException e) {
            return -1L; // 첫 실행
        }
    }

    @Transactional  // 읽기 모델 업데이트 트랜잭션에 참여
    public void save(String projectionName, String topic, int partition, long offset) {
        jdbcTemplate.update("""
            INSERT INTO projection_checkpoints
                (projection_name, topic, partition_num, last_offset, updated_at)
            VALUES (?, ?, ?, ?, NOW())
            ON CONFLICT (projection_name, topic, partition_num)
            DO UPDATE SET last_offset = ?, updated_at = NOW()
            """,
            projectionName, topic, partition, offset, offset
        );
    }
}
```

---

## 📊 패턴 비교

```
처리 보장 수준 비교:

┌────────────────────┬──────────────┬──────────────┬──────────────────┐
│ 방식               │ 이벤트 유실  │ 중복 처리    │ 구현 복잡도      │
├────────────────────┼──────────────┼──────────────┼──────────────────┤
│ At-Most-Once       │ 가능         │ 없음         │ 낮음             │
├────────────────────┼──────────────┼──────────────┼──────────────────┤
│ At-Least-Once      │ 없음         │ 가능         │ 낮음             │
├────────────────────┼──────────────┼──────────────┼──────────────────┤
│ At-Least-Once      │ 없음         │ 없음(멱등처리)│ 중간             │
│ + 멱등 처리        │              │              │                  │
├────────────────────┼──────────────┼──────────────┼──────────────────┤
│ Exactly-Once       │ 없음         │ 없음         │ 높음             │
│ (Kafka Tx)         │              │              │ (성능 오버헤드)  │
└────────────────────┴──────────────┴──────────────┴──────────────────┘

실무 권장:
  금융/주문 → At-Least-Once + 멱등 처리 (절대값 저장 + Upsert)
  통계/로그 → At-Least-Once (약간의 중복 허용 가능)
  채팅/알림 → At-Most-Once (유실 허용, 중복이 더 나쁜 경우)
```

---

## ⚖️ 트레이드오프

```
DB 체크포인트 방식 비용:
  추가 테이블 (projection_checkpoints)
  재시작 시 seek 로직 구현 필요
  트랜잭션 크기 증가 (읽기 모델 + 체크포인트 함께)

편익:
  읽기 모델 업데이트와 체크포인트가 원자적
  → 중복 처리 최소화 (At-Least-Once에서 가능한 한 Exactly-Once에 근접)

이벤트 ID 방식 비용:
  processed_events 테이블 → 영구 증가
  매 이벤트마다 SELECT(중복 체크) → I/O 증가
  편익: 가장 완전한 중복 방지
```

---

## 📌 핵심 정리

```
처리 보장 핵심:

선택 원칙:
  이벤트 유실 > 중복 처리 → At-Least-Once 필수
  중복 허용 불가 → 멱등 처리 추가

멱등 처리 패턴:
  절대값 저장 (balanceAfter): 누적 연산 → 마지막 상태값으로
  Upsert + last_event_seq: 오래된 이벤트로 덮어쓰기 방지
  event_id ON CONFLICT DO NOTHING: 집계/카운터 안전하게

체크포인트:
  DB에 저장 (읽기 모델 업데이트와 같은 트랜잭션)
  재시작 시 체크포인트 이후부터 seek
  → 최소한의 중복 재처리

실무 선택:
  At-Least-Once + 멱등 = Exactly-Once 효과
  Kafka Exactly-Once 트랜잭션은 성능 비용 고려 후 선택
```

---

## 🤔 생각해볼 문제

**Q1.** "이벤트를 처리하고 체크포인트를 같은 트랜잭션에 저장"하는 방식은 읽기 모델이 RDB일 때만 가능하다. Redis나 Elasticsearch를 읽기 모델로 쓸 때는 어떻게 처리하는가?

<details>
<summary>해설 보기</summary>

Redis나 Elasticsearch는 RDB 트랜잭션에 참여할 수 없으므로 원자적 처리가 불가능합니다. 이 경우 두 가지 접근이 있습니다.

첫째, 중간 레이어로 RDB를 사용합니다. 이벤트를 RDB에 먼저 멱등하게 기록(체크포인트 포함)한 후, 별도 프로세스가 RDB → Redis/Elasticsearch로 동기화합니다. RDB가 진실의 원천이 되고, Redis/Elasticsearch는 캐시 역할입니다.

둘째, 이벤트 ID 기반 중복 감지를 Redis 자체에서 구현합니다. `SET processed:EVT-001 1 NX EX 86400` (이미 있으면 실패)로 중복을 감지합니다. 단, Redis 재시작 시 processed 키가 사라질 수 있으므로 TTL과 재시작 처리 전략이 필요합니다.

Elasticsearch의 경우 문서 ID를 이벤트 ID로 사용하면 `PUT /index/_doc/{event_id}`가 자연스럽게 멱등합니다. 같은 이벤트 ID로 두 번 인덱싱해도 마지막 값으로 업데이트됩니다.

실무에서는 체크포인트를 RDB에 저장하고, Redis/Elasticsearch 업데이트 실패 시 재처리하는 구조를 가장 많이 사용합니다.

</details>

---

**Q2.** processed_events 테이블이 무한히 커지는 것을 방지하려면 어떻게 관리해야 하는가?

<details>
<summary>해설 보기</summary>

processed_events 테이블을 무한히 보존할 필요는 없습니다. 중복 처리 방지 목적이라면 "최근 N일"의 기록만 유지해도 충분합니다. 이미 처리된 이벤트가 N일이 지나서 다시 올 가능성은 극히 낮고, 체크포인트 기반 오프셋 관리로 N일 이전 이벤트가 재처리될 가능성 자체가 없어지기 때문입니다.

주기적 삭제로 `DELETE FROM processed_events WHERE processed_at < NOW() - INTERVAL '7 days'`를 스케줄 잡으로 실행합니다. 대량 삭제 시 성능을 고려해 배치로 나눠 삭제합니다.

파티셔닝이 더 효율적입니다. `processed_events`를 `processed_at` 기준으로 주별 파티션으로 나누면, 오래된 파티션을 `DROP TABLE processed_events_2024_w01`로 즉시 삭제할 수 있어 DELETE보다 훨씬 빠릅니다.

보존 기간은 "Kafka 오프셋 retention period"와 일치시키는 것이 안전합니다. Kafka가 7일치 이벤트를 보존한다면, processed_events도 7일 보존으로 충분합니다.

</details>

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[이전: DDD와의 통합 ⬅️](./03-ddd-integration.md)** | **[다음: 성능 최적화 ➡️](./05-performance-optimization.md)**

</div>
