# Event Sourcing의 실제 어려움

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Eventually Consistent 읽기 모델이 만드는 UX 문제는 어떤 패턴으로 나타나는가?
- 이벤트 스키마 마이그레이션의 실제 비용은 상태 저장과 어떻게 다른가?
- 이벤트가 많아질 때 쿼리가 어려운 이유는 무엇인가?
- 팀 학습 비용이 과소평가되는 이유와 실제 영향은 무엇인가?
- Event Sourcing을 선택하지 말아야 하는 도메인과 상황은 어디인가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

Event Sourcing의 장점을 알고 도입했다가 예상치 못한 어려움으로 팀이 고통받는 사례가 많다. "이벤트 스트림으로 모든 것을 해결할 수 있다"는 과도한 기대, Eventual Consistency가 UX에 미치는 영향 과소평가, 이벤트 스키마 변경의 실제 비용 인식 부족이 주요 원인이다. 이 어려움들을 정직하게 다뤄야 Event Sourcing을 적절한 곳에 적절한 방식으로 적용할 수 있다.

---

## 😱 흔한 실수 (Before — 어려움을 과소평가)

```
실수 1: "Eventual Consistency는 금방 해결된다"는 과신

  팀 결정: "읽기 모델 지연이 50ms면 사용자가 모를 거야"

  현실:
    사용자가 이체 버튼을 누름
    → 즉시 잔고 화면으로 이동
    → 아직 구버전 잔고 표시 (50ms 지연)
    → 사용자: "이체가 안 됐나? 다시 눌러야 하나?"
    → 이중 이체 발생

    모바일 네트워크 불안정 시:
    → 읽기 모델 지연 50ms → 500ms로 증가
    → 사용자 페이지 새로고침 → 또 구버전

실수 2: "이벤트 스키마는 나중에 고치면 된다"

  6개월 후 상황:
    event_type='OrderConfirmed' 이벤트 300만 개
    모두 customerName 필드가 있음
    신규 요구: customerName 제거 (개인정보 최소화)

    해결책:
      모든 이벤트에서 customerName 제거? → Append-Only 원칙 위반
      Upcaster 작성? → 300만 이벤트마다 변환 적용
      Projection 재구축? → Upcaster + Projection 재구축 = 수 시간

    "DB 컬럼 하나 삭제하면 되는" 상태 저장과 달리
    → 이벤트 스키마 마이그레이션은 훨씬 복잡

실수 3: 이벤트 스트림을 OLAP 쿼리에 사용

  "계좌 유형별, 지역별, 분기별 입출금 통계"

  이벤트 스토어로 쿼리:
    SELECT SUM(payload->>'amount'), DATE_TRUNC('quarter', occurred_at)
    FROM event_store
    WHERE event_type = 'MoneyDeposited'
    GROUP BY 2
    → 이벤트 수 1억 건 → 쿼리 수십 분

  올바른 접근:
    이벤트 스토어는 쓰기 최적화
    집계 분석은 별도 OLAP DB (ClickHouse, BigQuery)로 Projection
```

---

## ✨ 올바른 접근 (After — 어려움을 인식하고 설계에 반영)

```
어려움 1 대응 — Eventual Consistency:
  낙관적 UI 업데이트로 지연 숨기기
  Command 응답에 예상 상태 포함
  실시간 상태 업데이트 (SSE/WebSocket)
  "현재 처리 중" UI 상태로 사용자 안내

어려움 2 대응 — 스키마 마이그레이션:
  처음부터 스키마 변경 비용 고려한 설계
  개인정보는 이벤트에 포함하지 않고 ID만
  Upcaster 테스트 자동화
  스키마 변경 절차 문서화

어려움 3 대응 — 쿼리 어려움:
  이벤트 스토어 = 쓰기 최적화, OLAP 쿼리 불가
  Projection으로 읽기 모델 분리
  복잡한 집계 → 별도 분석 DB로 이벤트 스트리밍

어려움 4 대응 — 팀 학습 비용:
  점진적 도입 (일부 Aggregate에만 먼저 적용)
  팀 내 학습 세션, 실습 프로젝트
  외부 의존성 없는 단순 ES 구현으로 시작
```

---

## 🔬 내부 동작 원리

### 1. Eventual Consistency가 만드는 UX 문제 패턴

```
패턴 1 — "방금 한 행동이 반영되지 않음":

  타임라인:
    T=0   사용자: "이체" 버튼 클릭
    T=10ms  Command 처리 완료, 이벤트 스토어 저장
    T=10ms  Kafka 발행
    T=60ms  Projection 이벤트 처리, 읽기 모델 업데이트

    T=20ms  사용자: 잔고 화면으로 이동
    T=20ms  GET /accounts/ACC-001 → 읽기 모델 조회 (아직 업데이트 전)
    T=20ms  구버전 잔고 표시

  해결: 낙관적 UI 업데이트
    이체 버튼 클릭 → UI 즉시 새 잔고 표시 (서버 응답 전)
    Command 성공 → UI 유지
    Command 실패 → UI 롤백

패턴 2 — "새로고침해도 안 보임":

  사용자가 주문 완료 후 주문 목록 페이지로 이동
  → 새 주문이 목록에 없음 (Projection 미반영)
  → 사용자 새로고침
  → 이번엔 보임 (300ms 후 Projection 반영)

  해결:
    Order 생성 Command 응답에 새 orderId 포함
    클라이언트: 목록 조회 시 새 orderId가 있을 때까지 폴링
    또는: SSE로 Projection 완료 이벤트 수신 후 목록 갱신

패턴 3 — "읽기 모델로 쓰기 결정하면 안 됨":

  // ❌ 읽기 모델로 재고 확인 후 주문 생성
  OrderSummaryView view = orderSummaryRepository.findById(orderId);
  if (view.getStock() > 0) {  // 읽기 모델 — 지연된 데이터!
      commandBus.send(new CreateOrderCommand(...));
  }
  // 읽기 모델 지연 → 실제 재고 소진 후에도 주문 가능

  // ✅ 불변식 검증은 쓰기 모델(Aggregate)에서
  // Order.confirm() → InventoryService.checkStock() → 실시간 재고 확인
```

### 2. 이벤트 스키마 마이그레이션의 실제 비용

```
상태 저장 스키마 변경 비용:

  "customers 테이블에 tier 컬럼 추가"
  ALTER TABLE customers ADD COLUMN tier VARCHAR(20) DEFAULT 'STANDARD';
  → 실행 시간: 수 초 ~ 수 분 (행 수 의존)
  → 즉시 완료, 기존 레코드에 기본값 적용

Event Sourcing 스키마 변경 비용:

  "CustomerRegistered 이벤트에 tier 필드 추가"
  
  새 이벤트부터: 자동 (기본값 처리)
  
  기존 이벤트 (300만 건):
    옵션 1: Upcaster 작성 (이벤트 로드마다 변환)
      코드: 10줄
      구현 시간: 2시간
      테스트: Upcaster 단위 테스트 + 통합 테스트 = 4시간
      Projection 재구축 필요 여부 확인: 2시간
      총: 하루

    옵션 2: Projection 재구축
      읽기 모델에 tier 없는 기존 데이터 → tier 포함 재구축
      재구축 시간: 이벤트 300만 건 × 처리 시간 = 수 시간

  "CustomerRegistered에서 address 필드 제거 (GDPR)"
    새 이벤트부터: address 없이 발행
    기존 이벤트 300만 건의 address: 삭제 불가 (Append Only)
    Crypto Shredding: 암호화 키 삭제로 사실상 읽기 불가
    → 키 관리 인프라 필요

  장기 비용 누적:
    스키마 변경 10번 = Upcaster 10개 = 관리해야 할 변환 로직 10개
    이벤트 로드마다 Upcaster 체인 실행 (성능 영향은 미미하지만 복잡도 증가)
```

### 3. 이벤트 스토어의 쿼리 한계

```
이벤트 스토어 = 쓰기 최적화 구조

  적합한 쿼리:
    SELECT * FROM event_store WHERE stream_id = ? ORDER BY version
    → 특정 Aggregate 로드 (이것이 기본 사용 패턴)

  부적합한 쿼리 (가능하지만 느림):
    "계좌 잔고가 100만원 이상인 계좌 목록"
    → 이벤트 스토어에는 "현재 잔고"가 없음
    → 모든 account stream을 리플레이해야 함 → 불가

    "이번 달 이체 건수 top 10 계좌"
    → event_type='MoneyTransferred' WHERE 이번 달 GROUP BY stream_id
    → 인덱스 없으면 전체 스캔
    → 이벤트 1억 건 → 수십 분

  올바른 접근:
    이벤트 스토어 → Projection → 읽기 모델
    "계좌 잔고 top 10" → account_summary.balance 기준 SELECT
    "이번 달 이체 top 10" → monthly_transfer_stats Projection

  복잡한 집계 분석:
    이벤트 스토어 → Kafka → ClickHouse/BigQuery (OLAP)
    Analytical Query는 OLAP에서 처리
    이벤트 스토어는 Operational Query에만 사용
```

### 4. Event Sourcing을 선택하지 말아야 할 도메인

```
선택하지 말아야 할 신호:

① 단순 CRUD 도메인:
  사용자 설정, 공지사항, 카테고리 관리
  "저장 → 읽기 → 수정 → 삭제" 패턴
  불변식 없음, 감사 로그 불필요
  → Event Sourcing 비용만 있고 이점 없음

② 감사 로그가 법적/비즈니스 요구사항이 아닌 경우:
  "나중에 필요할 수도 있으니" → YAGNI 원칙 위반
  실제 감사 요구사항이 없으면 Append-Only 이벤트의 이점 없음

③ 팀이 아직 CQRS도 익숙하지 않은 경우:
  CQRS → Event Sourcing 순서로 학습해야
  둘을 동시에 도입하면 학습 부담 과중
  → 첫 3~6개월: CQRS (상태 저장) 익히기
  → 이후: Event Sourcing 추가

④ MVP 단계 / 도메인이 불안정한 경우:
  요구사항이 매주 바뀜 → 이벤트 스키마도 매주 바뀜
  Upcaster를 매주 작성하는 상황 → 생산성 급락
  → 도메인이 안정화되면 Event Sourcing 도입

⑤ 강한 일관성이 필수인 단순 케이스:
  "모든 조회가 항상 최신 상태를 보여야 한다"
  → Eventual Consistency 모든 경우에 적용하면 UX 문제
  → 강한 일관성이 필요한 부분만 상태 저장 + Event Sourcing 혼용

팀 학습 비용 현실:
  이론 이해: 2주
  실습 프로젝트: 1개월
  프로덕션 자신감: 3~6개월
  
  비용 과소평가 이유:
    이론(이벤트 저장, Projection)은 단순해 보임
    실제 어려움: 이벤트 스키마 진화, Projection 실패 복구,
                  Eventual Consistency UX 처리, 이벤트 스토어 운영
    → 이 문제들은 실제로 운영해봐야 체감
```

---

## 💻 실전 코드

```java
// ==============================
// 어려움 해결 패턴 코드
// ==============================

// ✅ Eventual Consistency 처리 — Command 응답에 예상 상태 포함
@PostMapping("/accounts/{id}/transfer")
public ResponseEntity<TransferResult> transfer(
        @PathVariable String id,
        @RequestBody TransferRequest request) {

    commandBus.send(new TransferMoneyCommand(
        id, request.toAccountId(), request.amount(), ...));

    // Command 내용에서 예상 결과 도출 (읽기 모델 조회 없음)
    TransferResult result = TransferResult.builder()
        .fromAccountId(id)
        .toAccountId(request.toAccountId())
        .amount(request.amount())
        .status("PROCESSING")  // 비동기 처리 중임을 명시
        .message("이체가 접수됐습니다. 잔고는 잠시 후 반영됩니다.")
        .build();

    return ResponseEntity.accepted().body(result);
}

// ✅ Projection 실패 복구 — Dead Letter Queue
@Component
public class ResilientProjection {

    @KafkaListener(topics = "account-events")
    public void process(ConsumerRecord<String, String> record) {
        try {
            DomainEvent event = deserialize(record.value());
            projectionHandler.handle(event);

        } catch (Exception e) {
            log.error("Projection 처리 실패: offset={} error={}",
                record.offset(), e.getMessage());

            // Dead Letter Queue로 이동 (재처리 대기)
            dlqPublisher.send(record.value(), e.getMessage());

            // 현재 이벤트는 건너뜀 — Projection 전체 중단 방지
            // 나중에 DLQ에서 수동 또는 자동 재처리
        }
    }
}

// ✅ 이벤트 스토어 쿼리 한계 — Projection으로 분리
// ❌ 이벤트 스토어에서 직접 집계 쿼리 (느림)
public List<AccountSummary> getTopAccountsByBalance_WRONG() {
    return jdbcTemplate.query(
        "SELECT stream_id, MAX(payload->>'balanceAfter') FROM event_store " +
        "WHERE event_type IN ('MoneyDeposited','MoneyWithdrawn') GROUP BY 1",
        // → 이벤트 1억 건 스캔 → 수십 분
        ...
    );
}

// ✅ 읽기 모델에서 집계 쿼리 (빠름)
public List<AccountSummary> getTopAccountsByBalance() {
    return accountSummaryRepository.findTop10ByOrderByBalanceDesc();
    // account_summary 테이블 — balance 인덱스 → 즉시
}

// ✅ ES 도입 체크리스트 실행
public class EventSourcingAdoptionChecker {

    public AdoptionRecommendation evaluate(DomainContext context) {
        List<String> reasons = new ArrayList<>();
        int score = 0;

        if (context.hasComplexInvariants()) { score++; reasons.add("복잡한 불변식"); }
        if (context.requiresAuditLog()) { score += 2; reasons.add("감사 로그 필수"); }
        if (context.requiresTimeTravel()) { score += 2; reasons.add("시간 여행 필요"); }
        if (context.hasFrequentViewChanges()) { score++; reasons.add("읽기 모델 자주 변경"); }
        if (context.teamIsExperienced()) { score++; reasons.add("팀 역량 있음"); }

        if (context.isCrudDomain()) { score -= 3; reasons.add("단순 CRUD (감점)"); }
        if (context.isMvpStage()) { score -= 2; reasons.add("MVP 단계 (감점)"); }
        if (context.teamIsNew()) { score -= 2; reasons.add("팀 경험 부족 (감점)"); }

        return score >= 3
            ? new AdoptionRecommendation("권장", reasons)
            : new AdoptionRecommendation("보류", reasons);
    }
}
```

---

## 📊 패턴 비교

```
어려움별 대응 전략:

┌────────────────────────┬─────────────────────────────────────────┐
│ 어려움                 │ 대응 전략                                │
├────────────────────────┼─────────────────────────────────────────┤
│ Eventual Consistency  │ 낙관적 UI, SSE, Command 응답에 예상 상태 │
├────────────────────────┼─────────────────────────────────────────┤
│ 스키마 마이그레이션    │ Upcaster, Crypto Shredding, 초기 설계 주의│
├────────────────────────┼─────────────────────────────────────────┤
│ 쿼리 어려움            │ Projection으로 읽기 모델, OLAP 분리       │
├────────────────────────┼─────────────────────────────────────────┤
│ 팀 학습 비용           │ 점진적 도입, 일부 Aggregate만 먼저        │
├────────────────────────┼─────────────────────────────────────────┤
│ Projection 장애        │ DLQ, 재처리, 재구축 자동화              │
└────────────────────────┴─────────────────────────────────────────┘
```

---

## ⚖️ 트레이드오프

```
Event Sourcing의 복잡도 수용 조건:

수용 가치 있는 경우:
  감사 로그 / 규제 요구사항이 핵심 비즈니스 요구사항
  과거 상태 재현이 비즈니스에 실질적으로 필요
  Projection 재구축으로 새 비즈니스 인사이트 실현
  팀이 학습 비용을 감당할 역량과 의지가 있음

수용하기 어려운 경우:
  단순 CRUD + 감사 로그 불필요
  팀이 CQRS도 아직 익숙하지 않음
  MVP 단계로 도메인이 불안정
  모든 조회가 강한 일관성 필요

현실적인 도입 전략:
  1단계: CQRS (상태 저장 + 읽기 모델 분리)로 시작
  2단계: 감사 로그가 정말 필요한 Aggregate에만 ES 도입
  3단계: 팀이 익숙해지면 점진적으로 확장
  → "일부 도메인에 ES, 나머지는 상태 저장" 혼용도 현실적
```

---

## 📌 핵심 정리

```
Event Sourcing의 실제 어려움:

1. Eventual Consistency UX:
   읽기 모델 지연 → "방금 한 행동이 안 보임"
   해결: 낙관적 UI + SSE + 명확한 UX 안내

2. 스키마 마이그레이션:
   이벤트 수정 불가 → 기존 이벤트 처리 복잡
   해결: Upcaster, Crypto Shredding, 초기 설계 주의

3. 쿼리 어려움:
   이벤트 스토어는 OLAP 쿼리 부적합
   해결: Projection으로 읽기 모델, 분석은 OLAP

4. 팀 학습 비용:
   이론 쉽고 실무 어려움 (운영 이슈들)
   해결: 점진적 도입, 일부 Aggregate만 먼저

5. 선택하지 말아야 할 상황:
   단순 CRUD, MVP, 팀 경험 부족, 강한 일관성 필수

현실적 조언:
  Event Sourcing은 모든 시스템의 답이 아님
  특정 도메인에서 강력한 이점
  어려움을 이해하고 수용 가능할 때 도입
```

---

## 🤔 생각해볼 문제

**Q1.** Projection이 이벤트를 처리하다 실패했을 때, 해당 이벤트를 건너뛰고 계속 진행해야 하는가, 아니면 중단해야 하는가?

<details>
<summary>해설 보기</summary>

상황에 따라 다르지만, 대부분의 경우 건너뛰고 계속 진행이 권장됩니다.

중단이 더 나은 경우는 이벤트 처리 순서가 이후 이벤트 처리에 영향을 주는 경우입니다. 예를 들어 AccountOpened가 실패하면 이후 MoneyDeposited를 처리할 계좌 레코드가 없어 실패 연쇄가 발생합니다. 이 경우 중단 후 수동 조치가 필요합니다.

건너뛰고 계속 진행이 나은 경우는 실패한 이벤트가 다른 이벤트와 독립적인 경우입니다. Dead Letter Queue에 실패 이벤트를 저장하고, Projection은 다음 이벤트를 처리합니다. DLQ의 이벤트는 나중에 문제를 수정하고 재처리합니다.

실무 권장사항으로는 Projection을 이벤트 타입별로 독립 Consumer Group으로 분리하고, 각 Consumer Group이 자체 DLQ를 가지도록 설계합니다. 한 이벤트 타입의 실패가 다른 타입 처리를 막지 않도록 격리합니다. 알림을 설정해 DLQ에 쌓이는 즉시 감지할 수 있도록 합니다.

</details>

---

**Q2.** "Event Sourcing을 전체 시스템에 적용하지 않고 일부 Aggregate에만 적용하는 혼용 전략"의 구체적인 어려움은 무엇인가?

<details>
<summary>해설 보기</summary>

혼용 전략의 실제 어려움이 여러 가지입니다.

개발자 혼란 문제가 있습니다. 어떤 Aggregate는 JPA로 저장하고, 어떤 것은 이벤트 스토어로 저장합니다. 새 기능을 개발할 때 어느 패턴을 따라야 하는지 명확하지 않습니다. 코드베이스에 두 가지 패턴이 공존하면 일관성이 없어 보입니다.

경계 간 트랜잭션 문제도 있습니다. ES Aggregate와 상태 저장 Aggregate가 같은 트랜잭션에서 처리되어야 할 때 복잡해집니다. ES는 이벤트 스토어에 INSERT, 상태 저장은 RDB에 UPDATE — 이 두 저장소를 원자적으로 처리하기 어렵습니다.

읽기 모델 설계 복잡도도 증가합니다. 어떤 읽기 모델은 ES Projection으로 만들고, 어떤 것은 JPA 쿼리로 만듭니다. Projection이 ES Aggregate와 상태 저장 Aggregate 데이터를 결합해야 할 때 설계가 복잡해집니다.

그럼에도 혼용이 현실적인 이유는 한 번에 전체 시스템을 전환하는 것보다 위험이 적고, 팀이 ES를 점진적으로 학습할 수 있으며, 도메인별로 적합한 패턴을 선택할 수 있기 때문입니다.

</details>

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[이전: Event Sourcing의 장점 완전 분해 ⬅️](./06-event-sourcing-benefits.md)** | **[다음: Chapter 4 — 읽기 모델과 Projection ➡️](../read-model-projection/01-projection-fundamentals.md)**

</div>
