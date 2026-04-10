# Event Sourcing의 장점 완전 분해

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- "완전한 감사 로그"가 별도 audit 테이블과 구체적으로 무엇이 다른가?
- 시간 여행이란 무엇이고, 어떤 비즈니스 문제를 해결하는가?
- 이벤트 스트림으로 프로덕션 이슈를 로컬에서 재현하는 방법은 무엇인가?
- 과거 이벤트에서 새 읽기 모델을 재구축할 수 있다는 것이 왜 강력한가?
- 이 장점들이 실제로 비즈니스 가치를 만드는 사례는 무엇인가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

Event Sourcing은 상당한 복잡도를 가져온다. 이 복잡도를 감수하는 이유를 구체적으로 이해해야 한다. "완전한 감사 로그가 좋다"는 추상적인 설명으로는 팀을 설득하기 어렵다. "이 감사 로그로 금융 감독원 조사에 30분 만에 대응했다", "이벤트 리플레이로 장애 원인을 5분 만에 찾았다" 같은 구체적 사례가 필요하다.

---

## 😱 흔한 실수 (Before — 상태 저장의 한계가 드러날 때)

```
상황 1: 금융 감독원 감사

  감사관: "2024년 1월 15일 ACC-001 계좌에서
          오후 2시 37분에 300만원이 출금됐는데,
          그 직전 잔고와 해당 거래를 승인한 직원을 확인해주세요."

  상태 저장 시스템:
    accounts 테이블: 현재 잔고 1,250,000원
    transactions 테이블: amount=-3000000, datetime=2024-01-15T14:37
    → 직전 잔고? 없음 (역산해야 하는데 다른 거래가 개입될 수 있음)
    → 승인한 직원? transactions에 없음
    → 응답: "시스템에 기록이 없습니다" → 감사 실패

  Event Sourcing 시스템:
    event_store WHERE stream_id='account-ACC-001'
    → MoneyWithdrawn {
        amount: 3000000,
        balanceAfter: 1250000,
        requestedBy: "TELLER-42",  ← 승인 직원
        approvedBy: "MANAGER-07",  ← 2차 승인자
        occurredAt: 2024-01-15T14:37:22,
        balanceBefore: 4250000      ← 직전 잔고
      }
    → 30초 만에 모든 정보 제공

상황 2: 재고 불일치 디버깅

  "product-ITEM-001 재고가 음수(-5)가 됐는데 언제, 어떤 순서로?"

  상태 저장: inventory.stock = -5 → 역산 불가, 로그 없음
  Event Sourcing: StockReserved, StockReleased 이벤트 시퀀스 리플레이
    → "동시 예약 6건이 동일 ms에 처리, 재고 체크 경쟁 조건 확인"
```

---

## ✨ 올바른 접근 (After — Event Sourcing의 장점 활용)

```
4가지 핵심 장점:

1. 완전한 감사 로그 (Built-in Audit Trail)
   "누가, 언제, 무엇을, 왜 변경했는가"가 이벤트에 내장

2. 시간 여행 (Time Travel)
   특정 과거 시점의 Aggregate 상태 정확히 재현

3. 디버깅 (Reproducible Production Issues)
   이벤트 스트림으로 프로덕션 이슈를 로컬에서 재현

4. 읽기 모델 재구축 (Projection Rebuild)
   새 비즈니스 요구사항 → 과거 이벤트에서 새 뷰 생성
```

---

## 🔬 내부 동작 원리

### 1. 완전한 감사 로그 — 별도 audit 테이블과의 차이

```
별도 audit 테이블 방식:

  accounts 테이블:
    account_id, balance, status, updated_at, updated_by

  account_audit_log 테이블:
    log_id, account_id, old_balance, new_balance,
    changed_at, changed_by, change_reason

  한계:
    ① 개발자가 audit 로그 코드를 매번 추가해야 함 → 실수로 누락 가능
    ② 비즈니스 이벤트가 아닌 기술적 변경(DB 마이그레이션)도 기록됨
    ③ 어떤 비즈니스 행동이 원인인지 알기 어려움
       "잔고가 바뀌었다" vs "이체가 발생했다" — 원인 정보 없음
    ④ 트랜잭션 롤백 시 audit 로그도 함께 롤백 → 실패한 시도 기록 없음

Event Sourcing의 감사 로그:

  이벤트 자체가 감사 로그:
    MoneyTransferred {
        amount: 1000000,
        fromAccountId: "ACC-001",
        toAccountId: "ACC-002",
        requestedBy: "USER-42",      ← 누가
        requestedAt: Instant,        ← 언제
        approvedBy: "MANAGER-07",    ← 승인자
        correlationId: UUID,         ← 이 요청을 추적하는 ID
        causationId: UUID,           ← 이 이벤트의 원인 이벤트
        occurredAt: Instant
    }

  장점:
    ① 이벤트 발행 = 자동으로 감사 기록 (추가 코드 불필요)
    ② 비즈니스 의미가 명확한 감사 로그 (이벤트 이름이 행동 표현)
    ③ 실패한 Command도 별도 기록 가능 (CommandFailed 이벤트)
    ④ 롤백 불가 (Append Only) → 실패 이력도 보존
    ⑤ correlationId/causationId로 이벤트 체인 추적
```

### 2. 시간 여행 — 과거 상태 재현

```
시간 여행 구현:

  목표: "2024-01-15 오후 3시의 ACC-001 잔고"

  상태 저장: 불가 (현재 상태만 있음)

  Event Sourcing:
    events = SELECT * FROM event_store
             WHERE stream_id = 'account-ACC-001'
               AND occurred_at <= '2024-01-15T15:00:00Z'
             ORDER BY version ASC

    Account account = Account.reconstitute(events);
    return account.getBalance(); // 2024-01-15 15:00의 정확한 잔고

비즈니스 활용 사례:

  케이스 1: 분쟁 해결
    고객: "1월 15일에 분명히 돈이 있었는데 출금이 왜 거절됐나요?"
    → 시간 여행으로 해당 시점 잔고 확인
    → "출금 요청 시점 잔고 450,000원, 출금 요청액 500,000원 → 잔액 부족"

  케이스 2: 세금 계산
    "2024년 1분기 말(3월 31일) 기준 모든 계좌 잔고 합계"
    → 3월 31일 기준으로 모든 account stream 시간 여행
    → 실시간 계산 vs 그때그때 스냅샷 없이도 가능

  케이스 3: 규제 요구사항
    "분기별 최고/최저 잔고 보고서 (금융 규제)"
    → 각 날짜 기준으로 잔고 계산 → 리포트 생성
    → 과거 데이터가 이미 있으므로 새 보고서 요구에도 즉시 대응
```

### 3. 디버깅 — 이벤트 스트림으로 이슈 재현

```
프로덕션 이슈 재현 방법:

  "ACC-001에서 2024-01-15에 이상한 거래가 발생했다"

  Step 1: 이벤트 스트림 내보내기
    SELECT * FROM event_store
    WHERE stream_id = 'account-ACC-001'
    ORDER BY version;

    → events.json 파일로 내보내기

  Step 2: 로컬에서 재현
    List<DomainEvent> events = loadFromFile("events.json");
    Account account = Account.reconstitute(events);

    // 이 시점의 Aggregate 상태 검사
    System.out.println("잔고: " + account.getBalance());
    System.out.println("상태: " + account.getStatus());

  Step 3: 문제 이벤트까지만 재현
    List<DomainEvent> eventsUntilBug = events.subList(0, 42); // 42번째 이벤트까지
    Account accountBeforeBug = Account.reconstitute(eventsUntilBug);
    // 버그 직전 상태 확인

  상태 저장 방식과 비교:
    상태 저장: 현재 DB 스냅샷을 복사해 로컬에서 재현하는 것은 어려움
              프로덕션 데이터 접근 필요, 마스킹 필요, 대용량 데이터
    Event Sourcing: 해당 stream의 이벤트만 JSON으로 내보내면 됨
                    소량 데이터, 민감 정보 마스킹 용이

비즈니스 로직 버그 재현:

  "특정 조건에서 잔고가 음수가 되는 버그"

  1. 버그가 발생한 account의 이벤트 스트림 추출
  2. 로컬에서 Account.reconstitute(events)
  3. withdraw(amount)로 버그 재현
  4. 스텝별 상태 확인으로 근본 원인 파악
  5. 수정 후 동일한 이벤트 시퀀스로 테스트 → 버그 없음 확인
```

### 4. 읽기 모델 재구축 — 새 비즈니스 요구사항 대응

```
Projection 재구축의 힘:

  시나리오: 2년간 운영 후 새 요구사항
    "월별 이체 수수료 통계가 필요합니다"
    "VIP 고객(잔고 1억 이상)의 거래 패턴 분석이 필요합니다"
    "지역별 계좌 개설 추이를 2년치 그래프로 보여주세요"

  상태 저장 시스템:
    "2년치 데이터가 없습니다. 오늘부터 수집하겠습니다."
    → 새 통계 = 도입 시점부터만 가능

  Event Sourcing 시스템:
    새 Projection 작성:
      @EventHandler
      void on(MoneyTransferred event) {
          feeStats.increment(event.feeAmount(), event.month());
      }

    전체 이벤트 스트림 재구축 실행:
      projectionRebuildService.rebuild("FeeSummaryProjection",
          fromGlobalSeq: 0); // 처음부터

    결과: 2년치 데이터가 수 시간 만에 완성

  재구축 방법:
    1. 기존 projection_checkpoint 리셋 (from_seq = 0)
    2. 새 Projection이 이벤트 스트림 처음부터 소비
    3. 읽기 모델 테이블에 데이터 누적
    4. 완료 후 새 API 제공

  재구축 중 운영 무중단:
    기존 읽기 모델은 살아있는 채로 새 모델 구축
    구축 완료 후 트래픽 전환 (Blue/Green)
```

---

## 💻 실전 코드

```java
// ==============================
// 4가지 장점 활용 코드
// ==============================

// ✅ 장점 1 — 감사 로그 조회
@Service
public class AuditService {

    public List<AuditEntry> getAuditTrail(String accountId,
                                           Instant from, Instant to) {
        return eventStore.loadEventsBetween(
            "account-" + accountId, from, to
        ).stream()
        .map(event -> new AuditEntry(
            event.eventType(),
            event.payload().get("requestedBy"),    // 누가
            event.occurredAt(),                     // 언제
            event.metadata().get("correlationId"), // 추적 ID
            summarize(event)                        // 무슨 변경
        ))
        .collect(toList());
    }
}

// ✅ 장점 2 — 시간 여행
@Service
public class TimeTravel {

    public Money getBalanceAt(String accountId, Instant pointInTime) {
        List<DomainEvent> events = eventStore
            .loadEventsUntil("account-" + accountId, pointInTime);

        Account account = Account.reconstitute(events);
        return account.getBalance();
    }

    // 분기 말 기준 전체 계좌 잔고 합계 (세금 보고용)
    public Money getTotalBalanceAt(Instant pointInTime) {
        List<String> allStreams = eventStore.findAllStreams("account-");
        return allStreams.parallelStream()
            .map(streamId -> getBalanceAt(
                streamId.replace("account-", ""), pointInTime))
            .reduce(Money.ZERO, Money::add);
    }
}

// ✅ 장점 3 — 디버깅: 이벤트 스트림으로 이슈 재현
@RestController
@RequestMapping("/debug")
@Profile("development") // 개발 환경에서만 노출
public class DebugController {

    @GetMapping("/accounts/{id}/replay")
    public DebugResult replayAccount(
            @PathVariable String id,
            @RequestParam(required = false) Long untilVersion) {

        List<DomainEvent> events = untilVersion != null
            ? eventStore.loadEventsUntilVersion("account-" + id, untilVersion)
            : eventStore.loadAllEvents("account-" + id);

        Account account = Account.reconstitute(events);

        return new DebugResult(
            id,
            account.getBalance(),
            account.getStatus(),
            events.size(),
            untilVersion
        );
    }
}

// ✅ 장점 4 — Projection 재구축
@Service
public class ProjectionRebuildService {

    @Transactional
    public void rebuild(String projectionName) {
        log.info("Projection 재구축 시작: {}", projectionName);

        // 1. 체크포인트 리셋
        checkpointRepository.reset(projectionName);

        // 2. 읽기 모델 테이블 초기화 (또는 별도 임시 테이블 사용)
        readModelRepository.truncate(projectionName);

        // 3. 이벤트 스트림 처음부터 재처리
        long lastSeq = 0;
        int batchSize = 1000;

        while (true) {
            List<StoredEvent> batch = eventStore.loadBatch(lastSeq, batchSize);
            if (batch.isEmpty()) break;

            batch.forEach(event ->
                projectionDispatcher.dispatch(projectionName, event));

            lastSeq = batch.get(batch.size() - 1).globalSeq();
            log.info("재구축 진행: projection={} lastSeq={}", projectionName, lastSeq);
        }

        log.info("Projection 재구축 완료: {}", projectionName);
    }
}
```

---

## 📊 패턴 비교

```
상태 저장 vs Event Sourcing — 장점 비교:

┌─────────────────────────┬───────────────┬─────────────────────────┐
│ 능력                    │ 상태 저장     │ Event Sourcing          │
├─────────────────────────┼───────────────┼─────────────────────────┤
│ 현재 상태 조회          │ ✅ 빠름       │ ⚠️ 리플레이 필요        │
├─────────────────────────┼───────────────┼─────────────────────────┤
│ 과거 시점 상태 조회     │ ❌ 불가       │ ✅ 시간 여행 가능       │
├─────────────────────────┼───────────────┼─────────────────────────┤
│ 완전한 감사 로그        │ ❌ 별도 구현  │ ✅ 내장                 │
├─────────────────────────┼───────────────┼─────────────────────────┤
│ 이슈 재현               │ ⚠️ 어려움    │ ✅ 이벤트 스트림으로    │
├─────────────────────────┼───────────────┼─────────────────────────┤
│ 새 읽기 모델 과거 데이터 │ ❌ 도입 후만  │ ✅ 과거 이벤트 재구축  │
├─────────────────────────┼───────────────┼─────────────────────────┤
│ 구현 복잡도             │ ✅ 낮음       │ ⚠️ 높음                 │
└─────────────────────────┴───────────────┴─────────────────────────┘
```

---

## ⚖️ 트레이드오프

```
장점을 실현하기 위한 비용:

감사 로그 장점:
  이벤트 페이로드에 감사 정보 포함 설계 필요 (requestedBy, approvedBy 등)
  이벤트가 감사 목적 데이터를 포함하면 크기 증가

시간 여행 장점:
  모든 이벤트를 영구 보존해야 함 (스토리지 비용)
  이벤트 수가 많으면 과거 재현도 오래 걸릴 수 있음

디버깅 장점:
  이벤트 스트림 내보내기/가져오기 도구 필요
  민감 데이터 마스킹 처리 필요

재구축 장점:
  재구축 시간 = 전체 이벤트 수 × 처리 시간 (수 시간 ~ 수 일)
  재구축 중 무중단 운영 전략 필요 (Blue/Green Projection)
```

---

## 📌 핵심 정리

```
Event Sourcing 4가지 핵심 장점:

1. 완전한 감사 로그:
   이벤트 = 감사 로그 (별도 구현 불필요)
   "누가, 언제, 무엇을, 왜" 이벤트에 내장

2. 시간 여행:
   occur_at 기준 이벤트 필터링 → 과거 상태 재현
   분쟁 해결, 세금 계산, 규제 대응

3. 디버깅:
   이벤트 스트림 추출 → 로컬 재현
   버그 시점까지만 리플레이 → 근본 원인 파악

4. 읽기 모델 재구축:
   새 비즈니스 요구 → 새 Projection → 과거 이벤트에서 재구축
   "도입 후부터만" 제한 없음

활용 가치:
  규제/감사 요구사항: 이벤트 스토어로 즉시 대응
  새 분석 요구사항: Projection 재구축으로 과거 데이터 생성
  이슈 재현: 이벤트 스트림으로 로컬 디버깅
```

---

## 🤔 생각해볼 문제

**Q1.** "새 Projection을 과거 이벤트에서 재구축할 수 있다"는 장점이 실제로 팀의 작업 방식을 어떻게 바꾸는가?

<details>
<summary>해설 보기</summary>

이 장점은 "미래의 질문에 현재의 데이터로 답할 수 있다"는 패러다임 변화를 만듭니다.

상태 저장 시스템에서는 미래에 어떤 분석이 필요할지 미리 예측해 데이터를 저장해야 합니다. 예측을 못 하면 과거 데이터가 없어 분석이 불가능합니다. 이는 초기 설계 부담을 높이고 "혹시 나중에 필요할까봐" 불필요한 컬럼을 추가하는 결정을 낳습니다.

Event Sourcing에서는 이 부담이 사라집니다. 이벤트 스트림이 원본 데이터이므로, 새로운 질문이 생기면 새 Projection을 작성해 과거 이벤트부터 재구축하면 됩니다. "우리 서비스 론칭 이후 모든 이체 데이터로 새 통계를 내고 싶다"가 가능합니다.

팀 작업 방식의 변화로는 읽기 모델 설계를 더 늦게, 더 자주 바꿀 수 있게 됩니다. 초기에 완벽한 스키마를 설계할 필요가 줄어들고, 비즈니스 요구사항이 명확해지면 그때 Projection을 추가하는 점진적 접근이 가능해집니다.

</details>

---

**Q2.** correlationId와 causationId를 이벤트 메타데이터에 포함하면 어떤 분석이 가능해지는가?

<details>
<summary>해설 보기</summary>

이 두 필드는 분산 시스템에서 이벤트 체인을 추적하는 핵심입니다.

correlationId는 하나의 비즈니스 요청 전체를 추적합니다. 예를 들어 "주문 확인" 요청이 correlationId=UUID-001을 가지면, 이 요청에서 발생한 OrderConfirmed, InventoryReserved, PaymentProcessed, EmailSent 등 모든 이벤트가 같은 correlationId를 공유합니다. "이 주문 확인 요청에서 어떤 일이 일어났는가"를 전체 시스템에 걸쳐 추적할 수 있습니다.

causationId는 "이 이벤트가 어떤 이벤트/커맨드 때문에 발생했는가"를 나타냅니다. OrderConfirmed → InventoryReserved의 causationId는 OrderConfirmed 이벤트의 ID입니다. 이를 통해 이벤트 인과 관계 그래프를 그릴 수 있습니다.

분석 가능한 것들로는 특정 이슈가 어떤 Command에서 시작됐는지 추적, 하나의 Command가 몇 개의 이벤트를 생성했는지 파악, 이벤트 처리 시간 계산(Command 수신 ~ 마지막 이벤트 처리), 장애 전파 경로 분석 등이 있습니다.

</details>

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[이전: 이벤트 스키마 진화 ⬅️](./05-event-schema-evolution.md)** | **[다음: Event Sourcing의 실제 어려움 ➡️](./07-event-sourcing-challenges.md)**

</div>
