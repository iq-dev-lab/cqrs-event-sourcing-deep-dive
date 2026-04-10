# 프로젝션 구현

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `@EventHandler`로 이벤트를 수신해 읽기 모델을 업데이트하는 완전한 구현은 어떻게 되는가?
- `@ResetHandler`는 무엇이고, 언제 필요한가?
- Tracking Event Processor의 Thread와 Segment 기반 병렬 처리는 어떻게 설정하는가?
- 프로젝션 재구축(Replay)은 어떻게 실행하는가?
- 프로젝션 처리 실패 시 Axon이 기본적으로 어떻게 처리하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

Axon 프로젝션은 `@EventHandler` 하나면 이벤트를 받을 수 있다. 하지만 실무에서는 멱등 처리, 재구축 지원, 병렬 처리, 장애 복구까지 고려해야 한다. `@ResetHandler`를 구현하지 않으면 재구축 시 기존 데이터가 남아 중복이 생긴다. Segment 설정을 모르면 단일 스레드로 처리해 프로젝션 지연이 발생한다.

---

## 😱 흔한 실수 (Before)

```
실수 1: @ResetHandler 없이 재구축

  @ProcessingGroup("account-summary")
  @Component
  public class AccountSummaryProjection {
      @EventHandler
      void on(AccountCreated event) {
          accountSummaryRepo.save(new AccountSummary(event.accountId(), ...));
      }
      // @ResetHandler 없음
  }

  재구축 실행:
    기존 account_summary 데이터 그대로
    + 이벤트 처음부터 재처리
    → UNIQUE 제약 위반 (이미 있는 accountId INSERT 시도)
    → 재구축 실패

실수 2: 단일 스레드 Tracking Processor로 처리량 부족

  # application.yml 설정 없음 → 기본값: 1개 스레드
  # 이벤트 1000건/s 발생 → 처리 속도 500건/s → 지연 누적

실수 3: @EventHandler에서 예외를 catch하지 않음

  @EventHandler
  void on(OrderConfirmed event) {
      // NullPointerException 발생 시
      orderSummaryRepo.update(event.orderId(),
          event.getCustomer().getName()); // ← NPE
      // 예외 → Tracking Processor가 재시도
      // → 동일 이벤트 무한 재시도 → Projection 멈춤
  }
```

---

## ✨ 올바른 접근 (After — 완전한 프로젝션 구현)

```
올바른 프로젝션 구조:

  @ProcessingGroup("account-summary")  // 독립 처리 그룹
  @Component
  public class AccountSummaryProjection {

      @EventHandler              // 이벤트 처리 (읽기 모델 업데이트)
      void on(AccountCreated event) { ... }

      @ResetHandler              // 재구축 전 읽기 모델 초기화
      void reset() {
          accountSummaryRepo.deleteAll(); // 기존 데이터 삭제
      }

      @AllowReplay               // @EventHandler에 추가 시 재구축 중에도 처리
  }

application.yml:
  axon:
    eventhandling:
      processors:
        account-summary:
          mode: tracking
          threadCount: 4    # 4개 병렬 스레드
          batchSize: 100    # 100건씩 배치
```

---

## 🔬 내부 동작 원리

### 1. Tracking Event Processor 동작 방식

```
Tracking Event Processor:

  별도 스레드에서 이벤트 스토어를 주기적으로 폴링
  각 Processor의 진행 위치를 Tracking Token으로 추적
  재시작 시 마지막 Token 위치부터 재처리

  token_entry 테이블 (Axon이 자동 관리):
  ┌──────────────────────┬──────────┬─────────────────┐
  │ processor_name       │ segment  │ token           │
  ├──────────────────────┼──────────┼─────────────────┤
  │ account-summary      │ 0        │ globalIndex=10500│
  │ account-summary      │ 1        │ globalIndex=10498│
  │ order-search         │ 0        │ globalIndex=9870 │
  └──────────────────────┴──────────┴─────────────────┘

  Segment:
    Processor를 N개 세그먼트로 분할
    각 세그먼트가 독립 스레드에서 처리
    이벤트를 Segment에 배정: hash(aggregateId) % segmentCount

    account-ACC-001 → hash % 4 = 0 → Segment 0
    account-ACC-002 → hash % 4 = 1 → Segment 1
    account-ACC-003 → hash % 4 = 2 → Segment 2

    → 같은 Aggregate 이벤트 = 같은 Segment = 순서 보장
    → 다른 Aggregate = 다른 Segment = 병렬 처리

Subscribing Event Processor와 차이:
  Subscribing: 이벤트 발행 스레드에서 동기 처리
    → Command 처리 지연, 리플레이 불가
    → 테스트, 감사 목적에 적합

  Tracking: 별도 스레드, 비동기, 리플레이 가능
    → 프로덕션 Projection에 적합
```

### 2. @ResetHandler와 재구축 흐름

```
재구축 실행 방법:

  방법 1: Axon Server Dashboard에서 클릭
  방법 2: EventProcessingConfiguration API

  EventProcessingConfiguration processingConfig;

  void rebuildProjection() {
      // 1. Tracking Processor 중단
      processingConfig.shutdownProcessor("account-summary");

      // 2. @ResetHandler 실행 (읽기 모델 초기화)
      //    processingConfig.resetTokens("account-summary") 내부에서 호출

      // 3. Token을 처음으로 리셋
      processingConfig.resetTokens("account-summary");

      // 4. Tracking Processor 재시작 (처음부터 재처리)
      processingConfig.startProcessor("account-summary");
  }

@ResetHandler의 역할:
  재구축 전 기존 읽기 모델 데이터 초기화
  Token 리셋과 함께 자동 호출

  @ResetHandler
  public void reset() {
      // 전체 초기화
      accountSummaryRepo.deleteAll();
      log.info("AccountSummary 읽기 모델 초기화 완료");
  }

  // 또는 특정 기간만 초기화 (부분 재구축)
  @ResetHandler
  public void reset(ResetContext context) {
      if (context instanceof MyResetContext ctx) {
          accountSummaryRepo.deleteByCreatedAtAfter(ctx.fromDate());
      }
  }
```

### 3. 멱등 처리 — 재구축 안전성

```
문제: 재구축 시 같은 이벤트를 두 번 처리할 수 있음

  재구축 중단 (70% 완료) → 재구축 재시작 → 70%부터 다시 처리
  → @ResetHandler는 처음에만 호출 → 기존 데이터 있음
  → INSERT 중복 발생 가능

해결: Upsert 패턴

  @EventHandler
  @Transactional
  void on(AccountCreated event) {
      accountSummaryRepo.upsert(  // INSERT ON CONFLICT DO UPDATE
          event.accountId(),
          event.ownerId(),
          event.initialDeposit(),
          "ACTIVE"
      );
  }

  또는 JPA의 save() (merge 동작):
    @EventHandler
    void on(AccountCreated event) {
        AccountSummary summary = new AccountSummary(
            event.accountId(),
            event.ownerId(),
            event.initialDeposit()
        );
        accountSummaryRepo.save(summary);  // 이미 있으면 merge
    }

last_event_seq로 이벤트 순서 보장:
  event_seq가 있는 이벤트 → last_event_seq < 현재 seq만 처리
  이미 더 최신 이벤트가 처리됐으면 건너뜀
```

### 4. 예외 처리와 재시도

```
Axon 기본 예외 처리:

  @EventHandler에서 예외 발생 시:
    Tracking Processor: 해당 이벤트 재시도 (3회 기본)
    3회 후에도 실패 → ErrorHandler 호출
    ErrorHandler 기본 동작: 프로세서 중단 (Processor PAUSED)

  위험: 하나의 이벤트 처리 실패 → Processor 전체 중단
         → 모든 이후 이벤트 처리 중단

ListenerInvocationErrorHandler 커스터마이즈:

  @Bean
  public ListenerInvocationErrorHandler projectionErrorHandler() {
      return (exception, event, eventHandler) -> {
          if (isTransient(exception)) {
              // 일시적 오류 → 재시도 (기본 동작)
              throw exception;
          }
          // 영구 오류 → 건너뜀 (DLQ 저장)
          dlqService.save(event, exception.getMessage());
          log.error("이벤트 처리 실패, 건너뜀: type={} error={}",
              event.getPayloadType().getSimpleName(), exception.getMessage());
          // 예외 던지지 않음 → 건너뜀
      };
  }

  @Configuration
  class ProjectionConfig implements EventHandlingConfiguration {
      @Override
      public void configure(EventHandlingConfigurer configurer) {
          configurer
              .byDefaultAssignTo("account-summary")
              .registerListenerInvocationErrorHandler(
                  "account-summary",
                  conf -> projectionErrorHandler()
              );
      }
  }
```

---

## 💻 실전 코드

```java
// ✅ 완전한 AccountSummary 프로젝션
@ProcessingGroup("account-summary")
@Component
@Transactional
public class AccountSummaryProjection {

    private final AccountSummaryRepository repo;

    // ── 읽기 모델 업데이트 ────────────────────────────────────

    @EventHandler
    public void on(AccountCreated event, @SequenceNumber long seq) {
        repo.save(AccountSummary.builder()
            .accountId(event.accountId())
            .ownerId(event.ownerId())
            .balance(event.initialDeposit())
            .status("ACTIVE")
            .lastEventSeq(seq)
            .createdAt(event.occurredAt())
            .build()
        );
    }

    @EventHandler
    public void on(MoneyDeposited event, @SequenceNumber long seq) {
        int updated = repo.updateBalanceIfNewer(
            event.accountId(),
            event.balanceAfter(),  // 절대값 업데이트
            seq
        );
        if (updated == 0)
            log.debug("이미 최신 상태 (멱등): accountId={} seq={}", event.accountId(), seq);
    }

    @EventHandler
    public void on(MoneyWithdrawn event, @SequenceNumber long seq) {
        repo.updateBalanceIfNewer(event.accountId(), event.balanceAfter(), seq);
    }

    @EventHandler
    public void on(AccountFrozen event) {
        repo.updateStatus(event.accountId(), "FROZEN");
    }

    // ── 재구축 지원 ───────────────────────────────────────────

    @ResetHandler
    public void reset() {
        log.info("AccountSummary 프로젝션 재구축 시작 — 기존 데이터 삭제");
        repo.deleteAll();
    }

    // ── 재구축 시 제외할 Handler ──────────────────────────────

    @EventHandler
    @DisallowReplay  // 재구축 중에는 이 메서드 호출 안 함
    public void on(AccountCreated event, ReplayStatus replayStatus) {
        if (!replayStatus.isReplay()) {
            // 실시간 이벤트에서만 알림 발송
            notificationService.notifyNewAccount(event.ownerId());
        }
    }
}

// ✅ 읽기 모델 Repository — 멱등 업데이트
@Repository
public interface AccountSummaryRepository extends JpaRepository<AccountSummary, String> {

    // 더 최신 이벤트로만 업데이트 (멱등 보장)
    @Modifying
    @Query("""
        UPDATE AccountSummary a
        SET a.balance = :balance, a.lastEventSeq = :seq
        WHERE a.accountId = :accountId AND a.lastEventSeq < :seq
        """)
    int updateBalanceIfNewer(@Param("accountId") String accountId,
                              @Param("balance") BigDecimal balance,
                              @Param("seq") long seq);
}

// ✅ application.yml — 병렬 처리 설정
/*
axon:
  eventhandling:
    processors:
      account-summary:
        mode: tracking
        threadCount: 4         # 4개 스레드 = 4개 세그먼트
        batchSize: 100         # 100건씩 배치 폴링
        initialSegmentCount: 4 # 초기 세그먼트 수
      order-search:
        mode: tracking
        threadCount: 2
        batchSize: 50
      payment-notification:
        mode: subscribing      # 동기 처리 (결제 완료 즉시 알림)
*/

// ✅ 프로젝션 재구축 API (Admin 전용)
@RestController
@RequestMapping("/admin/projections")
public class ProjectionAdminController {

    @Autowired EventProcessingConfiguration processingConfig;

    @PostMapping("/{name}/rebuild")
    public ResponseEntity<String> rebuild(@PathVariable String name) {
        try {
            processingConfig.shutdownProcessor(name);
            processingConfig.resetTokens(name);   // @ResetHandler 호출
            processingConfig.startProcessor(name);
            return ResponseEntity.ok("재구축 시작: " + name);
        } catch (Exception e) {
            return ResponseEntity.internalServerError().body(e.getMessage());
        }
    }

    @GetMapping("/{name}/status")
    public Map<String, Object> status(@PathVariable String name) {
        EventTrackerStatus status = processingConfig
            .processorStatus(name).get(0); // Segment 0 상태
        return Map.of(
            "name", name,
            "isRunning", status.isRunning(),
            "isMerging", status.isMerging(),
            "isReplaying", status.isReplaying(),
            "currentPosition", status.getTrackingToken()
        );
    }
}
```

---

## 📊 패턴 비교

```
Tracking Processor 설정별 처리량:

threadCount=1 (기본): 초당 ~1,000건 (단일 스레드)
threadCount=4:         초당 ~4,000건 (병렬, 각 Segment 독립)
threadCount=8:         초당 ~8,000건 (단, DB 병목 주의)

배치 크기 효과:
batchSize=1:    이벤트당 DB 트랜잭션 (오버헤드 높음)
batchSize=100:  100건씩 DB 트랜잭션 (권장)
batchSize=1000: 대량 처리 시 적합 (재구축 빠름)

재구축 예상 시간:
이벤트 100만건 / 처리량 10,000건/s = 100초 ≈ 2분
threadCount=4, batchSize=100 설정 기준
```

---

## ⚖️ 트레이드오프

```
Tracking Processor threadCount 증가:
  장점: 처리량 선형 증가
  단점: DB 커넥션 수 증가, 세그먼트 간 순서 의존 이벤트 주의

@DisallowReplay 사용:
  장점: 재구축 중 부수효과(이메일, 알림) 방지
  단점: 재구축 시 이 Handler는 건너뜀 → 부수효과 관련 데이터 누락 가능

ResetHandler에서 deleteAll():
  장점: 재구축 후 데이터 완전히 새로워짐
  단점: 재구축 중 읽기 모델이 비어있음 → 서비스 영향
  → Blue/Green Projection으로 해결 (05-multiple-read-models.md 참고)
```

---

## 📌 핵심 정리

```
Axon 프로젝션 구현 핵심:

@EventHandler:
  읽기 모델 업데이트 (멱등 Upsert 권장)
  @SequenceNumber로 이벤트 순서 추적
  @DisallowReplay로 재구축 시 부수효과 방지

@ResetHandler:
  재구축 전 읽기 모델 초기화
  deleteAll() or 기간별 삭제

Tracking Event Processor:
  threadCount: 병렬 스레드 수 (= 세그먼트 수)
  batchSize: 배치 처리 단위
  segment: hash(aggregateId) % threadCount → 순서 보장

재구축:
  shutdownProcessor → resetTokens(@ResetHandler 호출) → startProcessor
  Axon Server: Dashboard에서 클릭으로 재구축
```

---

## 🤔 생각해볼 문제

**Q1.** Tracking Processor의 `threadCount`를 8로 설정하면 이벤트 처리량이 정확히 8배가 되는가?

<details>
<summary>해설 보기</summary>

이론적으로는 8배이지만 실제로는 그렇지 않습니다. 병목이 여러 곳에 있습니다.

DB 커넥션 풀이 병목이 될 수 있습니다. 8개 스레드가 동시에 DB에 쓰면 커넥션 풀이 부족해집니다. 커넥션 풀 크기도 함께 늘려야 합니다(최소 threadCount 이상).

DB 락 경합이 발생할 수 있습니다. 같은 테이블의 다른 행을 업데이트하면 락 없이 병렬 처리되지만, 인덱스 리빌딩이나 테이블 락이 발생하면 병렬 효과가 줄어듭니다.

이벤트 스토어 폴링이 병목이 될 수 있습니다. 8개 스레드가 모두 이벤트 스토어를 폴링하면 이벤트 스토어 DB 부하가 증가합니다.

실무에서는 threadCount를 2부터 늘려가며 처리량과 DB 부하를 모니터링해 최적값을 찾습니다. 일반적으로 4~8개 사이에서 수렴합니다.

</details>

---

**Q2.** 프로젝션 재구축 중에 새 이벤트가 계속 발생한다면 어떻게 처리되는가?

<details>
<summary>해설 보기</summary>

Tracking Event Processor는 처음부터 순서대로 처리하므로, 재구축 중 발생한 새 이벤트도 결국 처리됩니다.

재구축 시작 시 Token을 0으로 리셋합니다. Processor가 Token 0부터 현재까지 순서대로 이벤트를 처리합니다. 재구축 중에도 새 이벤트는 이벤트 스토어에 쌓입니다. Processor가 과거 이벤트를 모두 따라잡으면 새 이벤트를 실시간으로 처리합니다.

문제는 재구축 속도가 새 이벤트 발생 속도보다 느리면 영원히 따라잡지 못한다는 것입니다. 이 경우 threadCount나 batchSize를 늘려 재구축 속도를 높이거나, 업무 시간 외에 재구축을 실행합니다.

재구축 중 @ResetHandler로 읽기 모델이 비워졌다면, 사용자에게 "데이터 업데이트 중"이라는 안내가 필요합니다. Blue/Green Projection 방식을 쓰면 재구축 중에도 기존 읽기 모델로 서비스를 유지할 수 있습니다.

</details>

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[이전: Aggregate 구현 ⬅️](./02-aggregate-implementation.md)** | **[다음: Query 처리 ➡️](./04-query-handling.md)**

</div>
