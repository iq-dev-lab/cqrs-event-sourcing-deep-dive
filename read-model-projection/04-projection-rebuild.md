# 프로젝션 재구축(Rebuild)

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 읽기 모델을 재구축해야 하는 상황은 언제인가?
- 운영 중 무중단으로 재구축하는 방법은 무엇인가?
- Blue/Green Projection 전략은 어떻게 동작하는가?
- 재구축 중 신규 이벤트가 계속 발생할 때 어떻게 처리하는가?
- 재구축 완료 후 트래픽 전환은 어떻게 하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

읽기 모델은 언제든 오염되거나 새로운 요구사항으로 재설계될 수 있다. Event Sourcing의 핵심 이점 중 하나가 "과거 이벤트에서 언제든 읽기 모델을 재구축할 수 있다"는 것이다. 하지만 재구축 중에도 서비스는 계속 운영돼야 한다. Blue/Green Projection 전략이 이 문제를 해결한다.

---

## 😱 흔한 실수 (Before — 재구축 시 서비스 중단)

```
실수 1: 재구축을 위해 서비스 중단

  // 재구축 절차:
  1. 서비스 점검 공지
  2. 트래픽 차단
  3. 기존 읽기 모델 테이블 TRUNCATE
  4. 이벤트 처음부터 Projection 재실행 (수 시간)
  5. 완료 후 서비스 재개
  // 문제: 고객이 수 시간 동안 서비스 이용 불가

실수 2: 재구축 중 새 이벤트 처리 누락

  TRUNCATE order_summary;  // 기존 데이터 삭제
  // 재구축 실행 중 새 주문 발생 (OrderPlaced 이벤트)
  // 재구축은 과거 이벤트만 처리 → 새 이벤트 누락
  // 재구축 완료 후 최신 주문이 읽기 모델에 없음

실수 3: 재구축 실패 시 롤백 불가

  // 재구축 진행 중 50% 완료 시점에 오류
  // 기존 테이블은 TRUNCATE되어 없음
  // 복구 불가 → 처음부터 다시
```

---

## ✨ 올바른 접근 (After — Blue/Green Projection)

```
Blue/Green Projection 전략:

  현재: Blue Projection → order_summary 테이블
  재구축: Green Projection → order_summary_v2 테이블 (새 테이블)

  단계:
    1. order_summary_v2 테이블 생성 (기존 유지)
    2. Green Projection 시작 (과거 이벤트부터 처리)
    3. Green Projection 재구축 완료
    4. 트래픽 전환: order_summary → order_summary_v2
    5. Blue Projection 중단, 기존 테이블 삭제

  이점:
    재구축 중 기존 읽기 모델은 정상 서비스
    재구축 실패 시 Blue 그대로 유지 (롤백 필요 없음)
    트래픽 전환 전 충분한 검증 가능
```

---

## 🔬 내부 동작 원리

### 1. 재구축이 필요한 상황

```
상황 1: 읽기 모델 오염
  버그 있는 Projection 코드가 잘못된 데이터 생성
  → 읽기 모델 전체 데이터 신뢰 불가
  → 재구축 필요

상황 2: 새로운 읽기 요구사항
  "주문 목록에 고객 등급(tier)도 표시해주세요"
  → order_summary에 customer_tier 컬럼 추가
  → 기존 행에 tier 값 없음
  → 재구축으로 모든 주문에 tier 채우기

상황 3: Projection 로직 변경
  "total_amount 계산에 버그 있음 — 쿠폰 금액을 빼지 않음"
  → 코드 수정 후 기존 order_summary의 total_amount 잘못됨
  → 재구축으로 올바른 금액 재계산

상황 4: 읽기 모델 구조 변경
  "order_summary를 주문 목록용/상세용 두 테이블로 분리하겠습니다"
  → 새 구조로 재구축

상황 5: 저장소 교체
  "order_summary를 PostgreSQL에서 Elasticsearch로 이전"
  → 새 Elasticsearch 인덱스로 재구축
```

### 2. Blue/Green Projection 완전한 흐름

```
전체 흐름:

Phase 1: 준비
  ① 새 읽기 모델 테이블/인덱스 생성
     CREATE TABLE order_summary_v2 (
         ...새 스키마...
         customer_tier VARCHAR(20)  ← 새 필드
     );

  ② Green Projection 코드 작성
     기존 Projection 로직 + 새 필드 처리 추가

  ③ Green Projection의 체크포인트 리셋
     UPDATE projection_checkpoints
     SET last_global_seq = 0
     WHERE projection_name = 'OrderSummaryProjectionV2'
     (또는 새 체크포인트 생성)

Phase 2: 재구축 실행
  ④ Green Projection 시작 (global_seq = 0부터)
     → order_summary_v2 테이블에 데이터 쌓기 시작

  ⑤ 재구축 진행 중 모니터링
     재구축 진행률 = Green Projection seq / 최신 global_seq
     ETA 계산: 처리 속도 × 남은 이벤트 수

  ⑥ Blue Projection도 계속 실행
     → 기존 order_summary 최신 상태 유지
     → 트래픽은 order_summary 사용 중

Phase 3: 전환 준비
  ⑦ Green Projection이 최신 이벤트에 따라잡음
     (Green seq ≈ Blue seq)

  ⑧ 두 테이블 데이터 검증
     SELECT COUNT(*) FROM order_summary
     SELECT COUNT(*) FROM order_summary_v2
     -- 건수 일치 확인

     SELECT * FROM order_summary WHERE order_id = 'ORD-001'
     SELECT * FROM order_summary_v2 WHERE order_id = 'ORD-001'
     -- 데이터 샘플 비교

Phase 4: 트래픽 전환
  ⑨ 애플리케이션 설정 변경 (무중단)
     read_model.table = order_summary_v2  ← 설정 변경

     또는 DB View로 전환:
     CREATE OR REPLACE VIEW order_summary_current AS
     SELECT * FROM order_summary_v2;  ← v2로 교체

  ⑩ 전환 후 모니터링 (1~24시간)
     에러율, 응답 시간, 데이터 정합성

Phase 5: 정리
  ⑪ Blue Projection 중단
  ⑫ 기존 order_summary 테이블 삭제 (또는 백업 후 삭제)
```

### 3. 재구축 중 신규 이벤트 처리

```
문제:
  재구축 시작: global_seq = 0에서 시작
  재구축 중: global_seq = 10,000까지 새 이벤트 발생
  → Green Projection은 순서대로 처리
  → 재구축 도중 새 이벤트가 order_summary_v2에 반영됨

이 과정이 올바른 이유:
  Projection은 이벤트 순서대로 처리 (version 오름차순)
  global_seq = 5000까지 처리 중
    → global_seq = 10000 이벤트도 존재
    → 아직 처리 안 됨 (순서대로이므로)
  global_seq = 10000까지 처리 완료
    → 이 시점에 새 이벤트 = 10001, 10002, ...
    → 계속 따라가며 처리

  결론: 재구축은 단순히 "처음부터 순서대로 재처리"
        신규 이벤트가 나타나도 순서대로 처리되므로 문제없음

catch-up 속도:
  재구축 처리 속도 > 새 이벤트 발생 속도 → 결국 따라잡음
  재구축 속도 < 새 이벤트 속도 → 영원히 따라잡지 못함
    해결: 재구축 병렬화 (배치 처리, 멀티스레드)
```

### 4. 재구축 성능 최적화

```
병렬 배치 처리:

  단일 이벤트씩 처리 (느림):
    for event in events:
        projection.handle(event)
        saveCheckpoint(event.seq)

  배치 처리 (빠름):
    for batch in events.chunks(1000):
        projection.handleBatch(batch)  // 1000개 일괄 처리
        saveCheckpoint(batch.last.seq) // 배치 완료 후 한 번 커밋

  배치 읽기 모델 업데이트:
    INSERT INTO order_summary (...)
    VALUES (row1), (row2), ..., (row1000)  // 1000건 한 번에 INSERT
    ON CONFLICT DO UPDATE ...

병렬화 (이벤트 파티션별):
  stream_id 해시 기반으로 파티션 분할
    Partition 0: account-ACC-001 ~ ACC-100
    Partition 1: account-ACC-101 ~ ACC-200

  각 파티션을 독립 스레드/프로세스로 처리
  → N배 병렬화 → N배 빠른 재구축

  단, 같은 stream의 이벤트는 순서 보장 필요
  → stream_id 해시로 같은 파티션에 배치

재구축 예상 시간:
  이벤트 1,000만 건 / 처리 속도 10,000건/s = 1,000초 ≈ 17분
  병렬화 4배 적용 → 약 4분
```

---

## 💻 실전 코드

```java
// ✅ Blue/Green Projection 재구축 서비스
@Service
public class ProjectionRebuildService {

    @Transactional
    public RebuildJob startRebuild(String projectionName, String targetTable) {
        // 1. 새 테이블 생성 (DDL은 별도 마이그레이션)
        String jobId = UUID.randomUUID().toString();

        // 2. 재구축 작업 등록
        RebuildJob job = new RebuildJob(jobId, projectionName, targetTable,
            RebuildStatus.STARTED, Instant.now());
        rebuildJobRepository.save(job);

        // 3. 비동기 재구축 시작
        rebuildExecutor.submit(() -> executeRebuild(job));

        return job;
    }

    private void executeRebuild(RebuildJob job) {
        long lastSeq = 0;
        int batchSize = 1000;
        long totalEvents = eventStore.countAll();

        try {
            while (true) {
                List<StoredEvent> batch = eventStore.loadBatch(lastSeq, batchSize);
                if (batch.isEmpty()) break;

                // 배치 처리
                projectionDispatcher.handleBatch(
                    job.projectionName(), job.targetTable(), batch);

                lastSeq = batch.get(batch.size() - 1).globalSeq();

                // 진행률 업데이트
                double progress = (double) lastSeq / totalEvents * 100;
                rebuildJobRepository.updateProgress(job.jobId(), lastSeq, progress);

                log.info("[{}] 재구축 진행: {:.1f}% (seq={})",
                    job.projectionName(), progress, lastSeq);
            }

            rebuildJobRepository.updateStatus(job.jobId(), RebuildStatus.COMPLETED);
            log.info("[{}] 재구축 완료", job.projectionName());

        } catch (Exception e) {
            rebuildJobRepository.updateStatus(job.jobId(), RebuildStatus.FAILED);
            log.error("[{}] 재구축 실패: {}", job.projectionName(), e.getMessage());
        }
    }

    // 트래픽 전환 — DB View 교체
    @Transactional
    public void switchTraffic(String projectionName, String newTable) {
        // 검증: 새 테이블 데이터 건수 확인
        long blueCount = jdbcTemplate.queryForObject(
            "SELECT COUNT(*) FROM order_summary", Long.class);
        long greenCount = jdbcTemplate.queryForObject(
            "SELECT COUNT(*) FROM " + newTable, Long.class);

        if (Math.abs(blueCount - greenCount) > blueCount * 0.01) { // 1% 이상 차이
            throw new RebuildValidationException(
                "데이터 건수 불일치: blue=" + blueCount + " green=" + greenCount);
        }

        // View 교체 (무중단 전환)
        jdbcTemplate.execute(
            "CREATE OR REPLACE VIEW order_summary_current AS SELECT * FROM " + newTable);

        log.info("트래픽 전환 완료: {} → {}", "order_summary", newTable);
    }
}

// ✅ 재구축 진행 상황 API
@RestController
@RequestMapping("/admin/projections")
public class ProjectionAdminController {

    @PostMapping("/{name}/rebuild")
    public ResponseEntity<RebuildJob> startRebuild(@PathVariable String name) {
        RebuildJob job = rebuildService.startRebuild(name, name + "_v" + nextVersion(name));
        return ResponseEntity.accepted().body(job);
    }

    @GetMapping("/rebuild/{jobId}/status")
    public RebuildJobStatus getStatus(@PathVariable String jobId) {
        return rebuildJobRepository.findById(jobId)
            .map(job -> new RebuildJobStatus(
                job.jobId(),
                job.status(),
                job.progress(),
                job.startedAt(),
                estimateCompletion(job)
            ))
            .orElseThrow();
    }

    @PostMapping("/rebuild/{jobId}/switch")
    public void switchTraffic(@PathVariable String jobId) {
        RebuildJob job = rebuildJobRepository.findById(jobId).orElseThrow();
        if (job.status() != RebuildStatus.COMPLETED)
            throw new IllegalStateException("재구축 완료 후에만 전환 가능");
        rebuildService.switchTraffic(job.projectionName(), job.targetTable());
    }
}
```

---

## 📊 패턴 비교

```
재구축 전략 비교:

┌─────────────────────┬──────────────┬──────────────┬───────────────┐
│ 전략                │ 서비스 중단  │ 롤백 용이성  │ 구현 복잡도   │
├─────────────────────┼──────────────┼──────────────┼───────────────┤
│ 서비스 중단 재구축  │ 수 시간      │ 불가         │ 낮음          │
├─────────────────────┼──────────────┼──────────────┼───────────────┤
│ Blue/Green 재구축   │ 없음         │ 즉시         │ 중간          │
├─────────────────────┼──────────────┼──────────────┼───────────────┤
│ 점진적 마이그레이션  │ 없음         │ 중간         │ 높음          │
└─────────────────────┴──────────────┴──────────────┴───────────────┘
```

---

## ⚖️ 트레이드오프

```
Blue/Green 비용:
  두 배의 저장 공간 (재구축 기간 동안)
  두 배의 Projection 처리 비용 (재구축 중 두 Projection 동시 실행)
  전환 전 데이터 검증 로직 필요

편익:
  무중단 재구축
  즉각적 롤백 (트래픽 전환 취소만 하면 됨)
  재구축 중 충분한 검증 시간
```

---

## 📌 핵심 정리

```
프로젝션 재구축 핵심:

Blue/Green 전략:
  현재: Blue Projection → 기존 테이블 (서비스 중)
  재구축: Green Projection → 새 테이블 (병렬)
  완료: 트래픽 전환 → 기존 테이블 삭제

재구축 단계:
  새 테이블 생성 → Green Projection 시작
  → 재구축 완료 → 데이터 검증
  → 트래픽 전환 → Blue 정리

성능 최적화:
  배치 처리 (1000건씩)
  stream_id 파티셔닝 병렬화

안전망:
  트래픽 전환 전 데이터 검증 (건수, 샘플)
  재구축 실패 시 Blue 그대로 (자동 롤백)
```

---

## 🤔 생각해볼 문제

**Q1.** 재구축 중 Blue Projection과 Green Projection이 동시에 동일한 이벤트를 처리한다. 이것이 문제가 되는가?

<details>
<summary>해설 보기</summary>

문제가 되지 않습니다. Blue와 Green은 서로 다른 테이블을 업데이트하기 때문입니다. Blue는 order_summary를, Green은 order_summary_v2를 각각 독립적으로 업데이트합니다. 두 Projection은 같은 이벤트를 읽지만 서로 다른 목적지에 씁니다.

다만 이벤트 스토어에서 같은 이벤트를 두 번 읽는 비용(DB I/O)이 발생합니다. 이벤트 스토어 부하가 두 배가 될 수 있습니다. 이 경우 Kafka를 사용한다면 Consumer Group을 각각 별도로 두면 Kafka의 Consumer Group 독립성 덕분에 이벤트 저장소 부하는 공유됩니다.

또한 두 Projection의 처리 결과가 다를 수 있습니다. 이것이 재구축의 목적입니다. Green은 수정된 로직으로 데이터를 다시 계산합니다.

</details>

---

**Q2.** 재구축이 완료됐지만 트래픽을 전환하기 전에 새로운 이벤트가 발생한다면 어떻게 처리되는가?

<details>
<summary>해설 보기</summary>

재구축 완료 후에도 Green Projection은 계속 실행되어야 합니다. "재구축 완료"는 과거 이벤트를 모두 처리했다는 의미이고, 이후 발생하는 새 이벤트도 Green Projection이 계속 처리합니다.

트래픽 전환 전까지 두 Projection 모두 실시간으로 이벤트를 처리합니다. Blue는 order_summary를, Green은 order_summary_v2를 동시에 최신 상태로 유지합니다.

트래픽 전환 직전에 "두 테이블이 동일한 상태인가" 검증이 중요합니다. 동일하지 않다면 Green Projection 로직에 버그가 있거나 특정 이벤트 타입을 처리하지 않은 것입니다. 전환 전 샘플 데이터 비교와 건수 비교로 이를 검증합니다.

</details>

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[이전: Eventual Consistency 처리 ⬅️](./03-eventual-consistency.md)** | **[다음: 여러 읽기 모델의 공존 ➡️](./05-multiple-read-models.md)**

</div>
