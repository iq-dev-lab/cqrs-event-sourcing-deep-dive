# 성능 최적화

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 이벤트 스토어 읽기 성능에서 인덱스 전략이 왜 핵심인가?
- 프로젝션 병렬 처리에서 순서 보장과 처리량을 어떻게 동시에 달성하는가?
- 읽기 모델에 Redis 캐싱을 적용할 때 Cache-Aside, Write-Through, Cache-Invalidation 중 어느 패턴이 CQRS에 적합한가?
- Hot Path(실시간 조회)와 Cold Path(통계/배치)를 분리하는 이유와 방법은 무엇인가?
- 이벤트 스토어 테이블이 수십억 건으로 커졌을 때 어떻게 접근하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

CQRS + Event Sourcing을 처음 도입할 때는 성능이 충분하다. 이벤트가 수백만 건으로 쌓이고, 프로젝션이 수십 개가 되고, 동시 접속자가 수천 명이 되면 병목이 드러난다. 이벤트 스토어 로드가 느려지고, 프로젝션 지연이 증가하고, 읽기 모델 조회가 타임아웃된다. 이 문제들의 원인과 해결책을 미리 이해해야 적시에 대응할 수 있다.

---

## 😱 흔한 실수 (Before — 성능 고려 없는 초기 설계)

```
실수 1: 이벤트 스토어 인덱스 부재

  CREATE TABLE event_store (
      global_seq BIGSERIAL,
      stream_id  VARCHAR(255),
      version    BIGINT,
      event_type VARCHAR(255),
      payload    JSONB,
      occurred_at TIMESTAMPTZ
  );
  -- 인덱스 없음
  -- "account-ACC-001" 스트림 로드:
  -- SELECT * FROM event_store WHERE stream_id = 'account-ACC-001' ORDER BY version
  -- → 전체 테이블 순차 스캔 (1억 건 테이블에서 수 초)

실수 2: 프로젝션 단일 스레드 처리

  while (true) {
      events = eventStore.loadBatch(lastSeq, 100);
      events.forEach(projection::handle); // 순차 처리
      // 초당 1000건 처리 가능한데 1000건 이벤트가 쌓이면 1초 지연
      // 이벤트 발생 속도가 처리 속도를 넘으면 지연이 선형 증가
  }

실수 3: 읽기 모델 조회마다 DB 직접 접근

  // 초당 10,000건 잔고 조회 → DB 직접 접근
  @GetMapping("/accounts/{id}/balance")
  public BigDecimal getBalance(@PathVariable String id) {
      return accountSummaryRepo.findById(id).getBalance();
      // DB 커넥션 풀 고갈, 응답 시간 증가
  }
```

---

## ✨ 올바른 접근 (After — 4단계 최적화 전략)

```
최적화 4단계:

1. 이벤트 스토어 인덱스 최적화
   (stream_id, version): Aggregate 로드 핵심 쿼리
   (global_seq): Projection 폴링
   파티셔닝: 수십억 건 테이블 관리

2. 프로젝션 병렬 처리
   stream_id 해시 기반 파티셔닝
   같은 stream은 같은 스레드 (순서 보장)
   다른 stream은 병렬 처리 (처리량 향상)

3. 읽기 모델 Redis 캐싱
   Hot 데이터 (자주 조회): Redis 캐시
   Cold 데이터 (드물게 조회): DB 직접

4. Hot Path / Cold Path 분리
   Hot: 실시간 조회 → Redis + 인덱싱된 RDB
   Cold: 통계/집계 → OLAP DB (ClickHouse, BigQuery)
```

---

## 🔬 내부 동작 원리

### 1. 이벤트 스토어 인덱스 전략

```
핵심 쿼리 패턴 → 인덱스 설계:

쿼리 1: Aggregate 로드 (가장 빈번)
  SELECT * FROM event_store
  WHERE stream_id = 'account-ACC-001'
  ORDER BY version ASC

  인덱스: CREATE INDEX idx_stream_version
              ON event_store (stream_id, version);
  → B-Tree 인덱스: stream_id로 그룹, version으로 정렬
  → 인덱스 범위 스캔으로 해당 스트림만 읽음

쿼리 2: 스냅샷 이후 이벤트 로드
  SELECT * FROM event_store
  WHERE stream_id = 'account-ACC-001'
    AND version > 4990
  ORDER BY version ASC

  같은 인덱스 사용: (stream_id, version) → version > 4990 조건

쿼리 3: Projection 폴링 (주기적)
  SELECT * FROM event_store
  WHERE global_seq > 10000
  ORDER BY global_seq ASC
  LIMIT 1000

  인덱스: CREATE UNIQUE INDEX idx_global_seq
              ON event_store (global_seq);
  → BIGSERIAL이므로 자동으로 효율적

쿼리 4: 이벤트 타입별 조회 (드물게)
  SELECT * FROM event_store
  WHERE event_type = 'MoneyWithdrawn'
    AND occurred_at BETWEEN ? AND ?

  인덱스: CREATE INDEX idx_type_time
              ON event_store (event_type, occurred_at);
  → 감사/분석 목적 쿼리

파티셔닝 전략 (수십억 건):

  Range Partitioning (occurred_at):
    event_store_2024_q1: 2024-01 ~ 03
    event_store_2024_q2: 2024-04 ~ 06
    → 오래된 파티션: 콜드 스토리지 이동 또는 DROP

  Hash Partitioning (stream_id):
    10개 파티션으로 균등 분산
    특정 스트림 집중 조회 시 파티션 프루닝 효과
```

### 2. 프로젝션 병렬 처리

```
순서 보장 + 병렬 처리의 딜레마:

  요구사항:
    같은 Aggregate(같은 stream_id)의 이벤트는 순서대로 처리
    다른 Aggregate의 이벤트는 병렬로 처리해도 됨

  해결: stream_id 해시 기반 파티셔닝

  Thread 0: account-ACC-001 (해시 % 4 = 0)
  Thread 1: account-ACC-002 (해시 % 4 = 1)
  Thread 2: account-ACC-003 (해시 % 4 = 2)
  Thread 3: account-ACC-004 (해시 % 4 = 3)
  → 같은 스트림 → 같은 스레드 (순서 보장)
  → 다른 스트림 → 다른 스레드 (병렬 처리)

Kafka 파티셔닝 활용:

  이벤트 발행 시 stream_id를 Kafka 파티션 키로 사용:
    kafkaTemplate.send("account-events",
        streamId,  ← Partition Key
        eventJson
    );
  → 같은 stream_id → 같은 Kafka 파티션 → 순서 보장
  → Consumer 인스턴스 = Kafka 파티션 수

  파티션 3개, Consumer 3개:
    Partition 0 → Consumer 0: account-ACC-001, ACC-004, ACC-007 ...
    Partition 1 → Consumer 1: account-ACC-002, ACC-005, ACC-008 ...
    Partition 2 → Consumer 2: account-ACC-003, ACC-006, ACC-009 ...

처리량 계산:
  단일 스레드: 초당 5,000건
  4 스레드: 초당 20,000건
  Kafka 파티션 10개: 초당 50,000건 (수평 확장)

배치 처리로 DB 왕복 최소화:

  // 이벤트 1건씩 처리 (느림)
  for (event in events):
      jdbcTemplate.update("UPDATE account_summary SET ...", ...)

  // 배치 UPDATE (빠름)
  jdbcTemplate.batchUpdate(
      "UPDATE account_summary SET ... WHERE account_id = ?",
      events.stream()
          .map(e -> new Object[]{ e.balance(), e.accountId() })
          .collect(toList())
  );
  → N번 왕복 → 1번 왕복
```

### 3. 읽기 모델 Redis 캐싱 전략

```
CQRS에 적합한 캐싱 패턴:

패턴 선택 기준:
  이벤트가 Projection을 통해 읽기 모델을 업데이트하는 구조
  → Projection이 캐시도 함께 업데이트하는 것이 자연스러움

Write-Through (Projection이 DB + Redis 동시 업데이트):

  @EventHandler
  void on(MoneyDeposited event) {
      // DB 업데이트
      jdbcTemplate.update("UPDATE account_summary SET balance = ?", event.balanceAfter());

      // Redis 캐시도 즉시 업데이트 (Write-Through)
      redis.set("account:balance:" + event.accountId(),
          event.balanceAfter().toString(),
          Duration.ofMinutes(10)  // TTL: 10분
      );
  }

  조회:
    cache hit: Redis에서 즉시 반환 (< 1ms)
    cache miss: DB 조회 → Redis 저장 → 반환

  장점: 읽기 속도 최대
  단점: Projection에 Redis 의존성, Redis 다운 시 처리 실패 가능

Cache-Aside (조회 시 캐시 없으면 DB 조회):

  public AccountBalance getBalance(String accountId) {
      String cached = redis.get("account:balance:" + accountId);
      if (cached != null) return new AccountBalance(new BigDecimal(cached));

      AccountSummary summary = summaryRepo.findById(accountId).orElseThrow();
      redis.setEx("account:balance:" + accountId,
          summary.getBalance().toString(), 600); // 10분 TTL
      return new AccountBalance(summary.getBalance());
  }

  Projection에서 캐시 무효화:
    @EventHandler
    void on(MoneyDeposited event) {
        summaryRepo.update(event);
        redis.delete("account:balance:" + event.accountId()); // 캐시 무효화
    }

  장점: Projection과 Redis 결합도 낮음
  단점: Cache miss 시 DB 조회 (첫 조회 느림)
        캐시 무효화 누락 시 구버전 캐시

CQRS 권장:
  잔고/재고 (자주 변경): Cache-Aside + 짧은 TTL
  상품 카탈로그 (드물게 변경): Write-Through + 긴 TTL
  통계 데이터 (주기적 재계산): TTL 기반 (Projection이 캐시 갱신)
```

### 4. Hot Path / Cold Path 분리

```
Hot Path (실시간, 저지연):
  대상: 잔고 조회, 주문 상태, 재고 현황
  요구사항: < 10ms 응답
  구현:
    Redis 캐시 + 인덱싱된 PostgreSQL

  GET /accounts/{id}/balance
  → Redis 조회 (< 1ms)
  → 캐시 미스 → PostgreSQL account_summary (인덱스 사용, < 5ms)

Cold Path (배치, 고처리량):
  대상: 매출 통계, 사용자 행동 분석, 리포트
  요구사항: 정확성, 대용량 처리, 수 초~수 분 허용
  구현:
    ClickHouse / BigQuery (OLAP)

  이벤트 스토어 → Kafka → ClickHouse Kafka Engine
  → 실시간 스트리밍 삽입
  → 복잡한 집계 쿼리 (초당 수억 건 처리 가능)

  SELECT DATE_TRUNC('hour', occurred_at), SUM(amount)
  FROM clickhouse.account_events
  WHERE event_type = 'MoneyDeposited'
    AND occurred_at >= NOW() - INTERVAL '30 days'
  GROUP BY 1
  ORDER BY 1
  -- ClickHouse: 수억 건에서 수 초

  vs PostgreSQL: 수억 건에서 수 분 이상

분리 기준:
  응답 시간 목표:
    < 100ms → Hot Path (Redis + RDB)
    수 초 허용 → Cold Path (OLAP)
  데이터 규모:
    수천만 건 이하 → RDB
    수억 건 이상 → OLAP

혼용 사례:
  대시보드 메인 지표 (어제 매출): Cold Path OLAP 사전 계산
  실시간 현재 잔고: Hot Path Redis
  주문 상세 조회: Hot Path PostgreSQL
  월별 정산 리포트: Cold Path BigQuery
```

---

## 💻 실전 코드

```java
// ✅ 이벤트 스토어 최적화 — 파티셔닝 + 인덱스
// DDL (migration)
/*
CREATE TABLE event_store (
    global_seq  BIGSERIAL,
    stream_id   VARCHAR(255) NOT NULL,
    version     BIGINT       NOT NULL,
    event_type  VARCHAR(255) NOT NULL,
    payload     JSONB        NOT NULL,
    occurred_at TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
    CONSTRAINT uq_stream_version UNIQUE (stream_id, version)
) PARTITION BY RANGE (occurred_at);

CREATE TABLE event_store_2024_q1 PARTITION OF event_store
    FOR VALUES FROM ('2024-01-01') TO ('2024-04-01');
CREATE TABLE event_store_2024_q2 PARTITION OF event_store
    FOR VALUES FROM ('2024-04-01') TO ('2024-07-01');

CREATE INDEX idx_sv_2024_q1 ON event_store_2024_q1 (stream_id, version);
CREATE INDEX idx_gs_2024_q1 ON event_store_2024_q1 (global_seq);
*/

// ✅ 프로젝션 병렬 처리 — 파티션별 독립 처리
@Configuration
public class ProjectionConfig {

    @Bean
    public ConcurrentKafkaListenerContainerFactory<String, String>
            parallelProjectionFactory(ConsumerFactory<String, String> cf) {

        ConcurrentKafkaListenerContainerFactory<String, String> factory =
            new ConcurrentKafkaListenerContainerFactory<>();
        factory.setConsumerFactory(cf);
        factory.setConcurrency(6); // Kafka 파티션 수와 일치
        factory.getContainerProperties().setAckMode(AckMode.MANUAL);
        return factory;
    }
}

@KafkaListener(
    topics = "account-events",
    groupId = "account-summary-projection",
    containerFactory = "parallelProjectionFactory"
)
public void handle(ConsumerRecord<String, String> record, Acknowledgment ack) {
    // 같은 파티션의 이벤트는 같은 스레드에서 순서대로 처리됨 (Kafka 보장)
    processEvent(record);
    ack.acknowledge();
}

// ✅ Redis 캐싱 — Write-Through
@Component
public class AccountBalanceCache {

    private static final Duration TTL = Duration.ofMinutes(10);

    public void update(String accountId, BigDecimal balance) {
        redis.opsForValue().set(
            "account:balance:" + accountId,
            balance.toString(),
            TTL
        );
    }

    public Optional<BigDecimal> get(String accountId) {
        String cached = redis.opsForValue().get("account:balance:" + accountId);
        return Optional.ofNullable(cached).map(BigDecimal::new);
    }

    public void evict(String accountId) {
        redis.delete("account:balance:" + accountId);
    }
}

// ✅ Hot Path / Cold Path 분기 조회
@RestController
@RequestMapping("/accounts")
public class AccountQueryController {

    // Hot Path — Redis + RDB
    @GetMapping("/{id}/balance")
    public BalanceResponse getBalance(@PathVariable String id) {
        return balanceCache.get(id)
            .map(balance -> new BalanceResponse(id, balance, "CACHED"))
            .orElseGet(() -> {
                AccountSummary summary = summaryRepo.findById(id).orElseThrow();
                balanceCache.update(id, summary.getBalance());
                return new BalanceResponse(id, summary.getBalance(), "DB");
            });
    }

    // Cold Path — OLAP (통계)
    @GetMapping("/stats/monthly-revenue")
    public MonthlyRevenueResponse getMonthlyRevenue(
            @RequestParam int year, @RequestParam int month) {
        // ClickHouse 또는 사전 계산된 stats 테이블
        return analyticsService.getMonthlyRevenue(year, month);
        // 응답 시간: 수 초 허용 (내부 대시보드 용도)
    }
}
```

---

## 📊 패턴 비교

```
최적화 기법별 효과:

┌─────────────────────────┬─────────────┬─────────────┬──────────────┐
│ 최적화                  │ 적용 대상   │ 효과        │ 복잡도       │
├─────────────────────────┼─────────────┼─────────────┼──────────────┤
│ 복합 인덱스             │ 이벤트 스토어│ 조회 100배↑ │ 낮음         │
├─────────────────────────┼─────────────┼─────────────┼──────────────┤
│ 파티셔닝                │ 이벤트 스토어│ 스캔 범위↓  │ 중간         │
├─────────────────────────┼─────────────┼─────────────┼──────────────┤
│ 프로젝션 병렬화         │ 프로젝션     │ 처리량 N배↑ │ 중간         │
├─────────────────────────┼─────────────┼─────────────┼──────────────┤
│ 배치 처리               │ 프로젝션     │ DB I/O 감소 │ 낮음         │
├─────────────────────────┼─────────────┼─────────────┼──────────────┤
│ Redis 캐싱              │ 읽기 모델    │ < 1ms 응답  │ 중간         │
├─────────────────────────┼─────────────┼─────────────┼──────────────┤
│ Hot/Cold 분리           │ 전체         │ 아키텍처↑   │ 높음         │
└─────────────────────────┴─────────────┴─────────────┴──────────────┘
```

---

## ⚖️ 트레이드오프

```
Redis 캐싱 비용:
  Redis 인프라 추가 운영
  캐시 무효화 로직 관리
  Redis 장애 시 DB 부하 증가 (Cache Stampede)
  → Cache Stampede 방지: 분산 락 또는 Probabilistic Early Expiration

OLAP 분리 비용:
  ClickHouse/BigQuery 운영 복잡도
  이벤트 스토어 → OLAP 파이프라인 관리
  편익: PostgreSQL에서 수억 건 집계 제거
```

---

## 📌 핵심 정리

```
성능 최적화 핵심:

이벤트 스토어:
  (stream_id, version) 복합 인덱스 필수
  (global_seq) 인덱스 필수
  수십억 건: Range 파티셔닝으로 오래된 데이터 관리

프로젝션:
  Kafka 파티션 키 = stream_id → 순서 보장 + 병렬화
  배치 DB 업데이트 → N번 왕복 → 1번 왕복
  파티션 수 = Consumer 인스턴스 수 = 최대 병렬도

읽기 모델:
  Hot 데이터 → Redis (< 1ms)
  Cold 데이터 → OLAP (수 초 허용)
  Write-Through or Cache-Invalidation 선택

Hot/Cold 분리:
  Hot (< 100ms): Redis + 인덱싱된 RDB
  Cold (수 초): ClickHouse / BigQuery
```

---

## 🤔 생각해볼 문제

**Q1.** Kafka 파티션 수를 늘려 프로젝션을 더 병렬화하면 처리량은 무한정 증가하는가? 한계는 무엇인가?

<details>
<summary>해설 보기</summary>

처리량 증가에는 한계가 있습니다. 병목이 Kafka Consumer에서 DB 업데이트로 이동합니다.

Kafka 파티션 10개 → Consumer 10개가 동시에 DB에 업데이트를 실행합니다. DB 커넥션 풀이 부족하거나, 같은 테이블의 행을 동시에 업데이트하면 락 경합이 발생합니다. 이 경우 파티션을 더 늘려도 Consumer가 DB 응답을 기다리느라 오히려 처리량이 줄어들 수 있습니다.

한계 지점을 찾는 방법으로는 부하 테스트에서 Consumer 수를 2, 4, 8, 16으로 늘려가며 DB CPU, 커넥션 수, 쿼리 지연을 모니터링합니다. 더 이상 처리량이 늘지 않는 지점이 현재 DB의 한계입니다.

DB 한계를 넘으려면 읽기 모델 DB를 수평 분산(Sharding)하거나, 배치 업데이트로 DB 왕복을 줄이거나, 읽기 모델을 Redis로 전환해 DB 부하를 줄이는 방법이 있습니다.

</details>

---

**Q2.** Redis 캐시에 저장된 잔고와 DB의 읽기 모델 잔고가 불일치하는 상황은 언제 발생하고 어떻게 감지하는가?

<details>
<summary>해설 보기</summary>

불일치가 발생하는 시나리오입니다. Projection이 DB 업데이트 후 Redis 캐시 업데이트 전에 크래시하면, DB에는 최신 잔고가 있지만 Redis에는 구버전이 남습니다. TTL이 만료되기 전까지 구버전 캐시가 반환됩니다.

또한 Redis 네트워크 오류로 캐시 무효화 명령이 실패하면 DB는 업데이트됐지만 캐시는 구버전을 유지합니다.

감지 방법으로는 샘플링 기반 일관성 검증이 있습니다. 주기적으로 랜덤 계좌를 선택해 Redis 캐시값과 DB 읽기 모델값을 비교합니다. 불일치율이 임계값(예: 0.1%)을 초과하면 알람을 발생시킵니다.

해결은 불일치를 감지한 계좌의 캐시를 삭제(evict)하면 다음 조회에서 DB에서 새로 읽어 캐시를 채웁니다. 또한 TTL을 짧게(5~10분) 설정해 자연스럽게 만료되도록 하면 최대 TTL 시간 후 자동 복구됩니다.

완전한 일관성이 필요하다면 Cache-Aside + 짧은 TTL 전략을 사용하고, Write-Through는 피하거나 Redis 업데이트 실패 시 롤백 처리를 추가합니다.

</details>

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[이전: 처리 보장 ⬅️](./04-processing-guarantee.md)** | **[다음: 마이크로서비스와의 통합 ➡️](./06-microservices-integration.md)**

</div>
