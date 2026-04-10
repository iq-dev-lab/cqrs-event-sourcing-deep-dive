# CQRS / ES의 실제 비용과 편익

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- CQRS와 Event Sourcing이 적합한 도메인과 부적합한 도메인을 어떻게 구분하는가?
- 팀 학습 비용은 실제로 얼마나 드는가?
- 이벤트 스토어, Projection, 읽기 모델 모니터링의 운영 복잡도는 어느 수준인가?
- 도입 의사결정 시 사용할 수 있는 체크리스트는 무엇인가?
- 실제 도입 사례에서 무엇이 기대와 달랐는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

CQRS와 Event Sourcing은 강력하지만 비용이 있다. 이 비용을 과소평가하면 팀이 예상치 못한 복잡도에 빠진다. 반대로 과대평가하면 실제로 이점을 얻을 수 있는 도메인에서도 도입을 포기한다. 비용과 편익을 구체적으로 이해해야 팀에게 맞는 선택을 할 수 있다.

---

## 😱 흔한 실수 (Before — 비용/편익 오판)

```
오판 1: 학습 비용 과소평가

  팀 결정: "CQRS/ES 도입합니다. 공식 문서 읽으면 됩니다."
  현실:
    이론 이해: 2주
    첫 Aggregate 구현: 1개월 (디버깅 포함)
    프로젝션 장애 대응: 2개월째 처음 경험
    Projection 재구축 시나리오 처음 처리: 3~4개월
    Saga 처음 구현: 4~6개월
  → 팀이 자신감 갖기까지: 6~12개월

오판 2: 편익 과대평가

  기대: "ES 도입하면 감사 로그, 시간 여행, 읽기 모델 재구축이 자동으로 된다"
  현실:
    감사 로그: 이벤트 페이로드 설계 잘못하면 여전히 정보 없음
    시간 여행: 이벤트 리플레이 비용 (스냅샷 없으면 느림)
    재구축: Blue/Green 전략 없으면 서비스 중단

오판 3: 운영 복잡도 무시

  "개발만 잘 하면 된다"
  현실:
    이벤트 스토어 모니터링 추가
    Projection 지연 알람 추가
    DLQ 모니터링 추가
    Consumer Group lag 모니터링 추가
    → 운영 대시보드에 10개 이상 지표 추가
```

---

## ✨ 올바른 접근 (After — 근거 기반 의사결정)

```
도입 의사결정 프레임워크:

  Step 1: 도메인 적합성 확인 (5분)
    감사 로그 법적/비즈니스 요구사항 있음? +3점
    과거 시점 상태 재현 필요? +2점
    읽기/쓰기 비율 불균형 심함? +2점
    도메인 이벤트가 자연스럽게 표현 가능? +2점
    단순 CRUD 도메인? -3점
    도메인 아직 불안정? -3점
    → 5점 이상: CQRS/ES 검토
    → 3점 미만: 상태 저장 유지

  Step 2: 팀 준비도 확인 (5분)
    DDD, Event-Driven 경험 있음? +2점
    Kafka 운영 경험 있음? +1점
    팀원 5인 이상? +1점
    → 3점 미만: 점진적 도입으로 시작

  Step 3: 로드맵 결정
    5점 이상 + 팀 준비 3점 이상: 완전한 CQRS + ES
    3~5점: CQRS만 (ES 제외)
    3점 미만: 현재 아키텍처 유지
```

---

## 🔬 내부 동작 원리

### 1. 적합한 도메인 vs 부적합한 도메인

```
CQRS + ES가 강력한 이점을 주는 도메인:

금융 도메인 (은행 계좌, 결제):
  이점: 완전한 거래 이력 (규제 필수)
        시간 여행 (분쟁 해결)
        출금 불가 → 정확한 불변식 이력
  실제 사례: 모든 핀테크 스타트업
  적합도: ★★★★★

전자상거래 주문:
  이점: 주문 상태 전환 이력 (CS 대응)
        Projection 재구축 (새 비즈니스 분석)
        읽기(조회) >> 쓰기(주문) 비율
  실제 사례: Shopify, Amazon
  적합도: ★★★★☆

재고 관리:
  이점: 재고 변동 이력 (부족 원인 분석)
        여러 읽기 모델 (창고별, 상품별, 지역별)
  적합도: ★★★★☆

CQRS/ES 적용 시 오히려 손해인 도메인:

사용자 설정 / 프로필:
  이벤트: ProfileUpdated {name, bio, avatar} 수시 발생
  이점: 거의 없음 (이전 bio가 필요한 케이스 없음)
  비용: 이벤트 스토어, 재구성 코드
  적합도: ★☆☆☆☆ (상태 저장 권장)

공지사항 / 블로그 포스트:
  대부분 한 번 쓰고 드물게 수정
  이전 버전 이력이 비즈니스 요구사항인가? 대부분 아님
  적합도: ★★☆☆☆

세션 / 캐시 데이터:
  매우 짧은 생명주기, 빠른 만료
  이벤트 이력 저장이 의미 없음
  적합도: ★☆☆☆☆
```

### 2. 팀 학습 비용 — 실제 타임라인

```
CQRS + ES 팀 학습 현실적 타임라인:

Month 1~2: 이론 + 첫 구현
  학습: DDD, Event Sourcing 개념
  구현: 첫 Aggregate 구현 (버그 많음)
  디버깅: @EventSourcingHandler 재구성 원리 이해
  팀 분위기: "어렵지만 흥미롭다"

Month 2~4: 첫 프로덕션 이슈
  Projection 지연 첫 경험 → 원인 파악 수 시간
  DLQ 처음 쌓임 → 재처리 방법 모름
  이벤트 스키마 변경 첫 번째 Upcaster 작성
  팀 분위기: "생각보다 복잡하다"

Month 4~6: 운영 패턴 확립
  Projection 모니터링 대시보드 구축
  DLQ 재처리 절차 문서화
  Saga 첫 번째 구현 (실패 처리 포함)
  팀 분위기: "패턴이 보이기 시작했다"

Month 6~12: 자신감 단계
  새 팀원에게 설명 가능
  장애 발생 시 원인 파악 10분 이내
  이벤트 설계 리뷰 능숙
  팀 분위기: "이제 제대로 쓰는 느낌"

비용 정량화 (5인 팀 기준):
  초기 6개월: 기능 개발 생산성 30~50% 감소
  이후: 생산성 회복 + 유지보수 비용 감소
  손익분기점: 약 12~18개월
```

### 3. 운영 복잡도 — 추가되는 모니터링

```
CQRS + ES 도입 후 추가 운영 항목:

이벤트 스토어 모니터링:
  이벤트 스토어 DB 용량 (연간 성장률 추정)
  스트림당 이벤트 수 분포 (스냅샷 필요 시점)
  이벤트 저장 지연 (p95, p99)

Projection 모니터링:
  Consumer Group lag (Kafka 기준)
    → 임계값: 1000건 초과 시 알람
  Projection 지연 (이벤트 발행 ~ 읽기 모델 반영)
    → 임계값: p99 > 5초 알람
  DLQ 크기
    → DLQ에 메시지 있으면 즉시 알람
  Projection 처리 실패율
    → 1% 이상 알람

읽기 모델 DB 모니터링:
  읽기 모델 DB 응답 시간
  읽기 모델과 이벤트 스토어 불일치 감지 (샘플 검증)

Axon Server (사용 시) 추가 모니터링:
  Axon Server 가용성
  Command 처리 시간 분포
  Query 처리 시간 분포

총 추가 대시보드 패널: 15~25개
총 추가 알람 규칙: 10~15개
운영 부담: 기존 대비 약 30% 증가
```

### 4. 실제 도입 사례 — 기대와 현실

```
사례 1: 핀테크 스타트업 (계좌, 결제 도메인)

  기대: 감사 로그 자동화, 규제 대응 용이
  현실 이점:
    ✅ 금융 감독원 조사 → 이벤트 스토어로 30분 내 대응
    ✅ 고객 분쟁 → 거래 시점 잔고 즉시 조회
    ✅ 새 분석 요구 → Projection 재구축으로 과거 데이터 활용

  예상 못한 비용:
    ❌ 이벤트 스키마 마이그레이션 (GDPR 적용)
    ❌ 초기 6개월 개발 속도 30% 감소
    ❌ Axon Server 장애 → 전체 Command 처리 중단 경험

  결론: "금융 도메인에는 ES가 맞다. 다만 팀 준비 더 필요"

사례 2: 전자상거래 주문 시스템

  기대: 읽기 모델 최적화로 조회 성능 개선
  현실 이점:
    ✅ 주문 목록 조회: 500ms → 5ms (읽기 모델 분리)
    ✅ 새 CS 뷰 추가: 3일 → Projection 1개 추가로 해결
    ✅ 마케팅 데이터: 과거 이벤트 재구축으로 2년치 분석 가능

  예상 못한 비용:
    ❌ Eventual Consistency → 주문 후 목록 미반영 민원
       → 낙관적 UI 업데이트로 해결 (추가 개발)
    ❌ Projection 재구축 소요 시간: 이벤트 300만건 → 40분

  결론: "CQRS는 확실히 효과. ES는 주문 도메인에서도 가치"

사례 3: B2B SaaS (설정, 권한 관리)

  도입 후 6개월 결론:
    ❌ 설정 도메인은 CRUD로 충분했음
    ❌ Event Sourcing 이벤트 수: 이전 설정값 조회 요구사항 없음
    ❌ 팀 생산성 감소가 이점을 초과

  결론: "설정 도메인 ES 제거. CQRS만 유지 (읽기 모델 분리)"
```

---

## 💻 실전 코드

```java
// ✅ 도입 의사결정 체크리스트 코드로 표현
public class CqrsEsAdoptionAdvisor {

    public AdoptionRecommendation evaluate(DomainContext ctx) {
        int score = 0;
        List<String> reasons = new ArrayList<>();
        List<String> concerns = new ArrayList<>();

        // 긍정 요소
        if (ctx.hasLegalAuditRequirement()) {
            score += 3;
            reasons.add("법적 감사 로그 요구사항 — ES 핵심 이점");
        }
        if (ctx.requiresHistoricalStateQuery()) {
            score += 2;
            reasons.add("시간 여행 필요 — ES 없이 불가");
        }
        if (ctx.readToWriteRatio() > 10) { // 읽기가 쓰기의 10배 이상
            score += 2;
            reasons.add("읽기/쓰기 비율 불균형 — 읽기 모델 분리 이점 큼");
        }
        if (ctx.hasNaturalDomainEvents()) {
            score += 2;
            reasons.add("도메인 이벤트 자연스럽게 표현 가능");
        }

        // 부정 요소
        if (ctx.isCrudDomain()) {
            score -= 3;
            concerns.add("단순 CRUD 도메인 — ES 비용만 추가");
        }
        if (ctx.isDomainUnstable()) {
            score -= 3;
            concerns.add("도메인 불안정 — 이벤트 스키마 변경 빈번 예상");
        }
        if (ctx.teamSize() < 3) {
            score -= 2;
            concerns.add("소규모 팀 — 운영 부담 과중");
        }
        if (!ctx.teamHasEventDrivenExperience()) {
            score -= 1;
            concerns.add("이벤트 기반 경험 부족 — 학습 비용 높음");
        }

        AdoptionLevel level;
        if (score >= 5) level = AdoptionLevel.FULL_CQRS_ES;
        else if (score >= 3) level = AdoptionLevel.CQRS_ONLY;
        else if (score >= 1) level = AdoptionLevel.INCREMENTAL;
        else level = AdoptionLevel.NOT_RECOMMENDED;

        return new AdoptionRecommendation(level, score, reasons, concerns);
    }
}

// ✅ 운영 복잡도 — 필수 모니터링 설정
@Configuration
public class CqrsEsMonitoringConfig {

    @Bean
    public MeterRegistryCustomizer<MeterRegistry> addCqrsMetrics() {
        return registry -> {
            // Projection 지연
            Gauge.builder("projection.lag.seconds",
                    projectionMonitor, ProjectionMonitor::getMaxLagSeconds)
                .tag("projection", "account-summary")
                .description("Projection 최대 지연 (초)")
                .register(registry);

            // DLQ 크기
            Gauge.builder("dlq.size",
                    dlqService, DlqService::getTotalSize)
                .description("DLQ 총 메시지 수")
                .register(registry);

            // 이벤트 스토어 크기
            Gauge.builder("event.store.total.events",
                    eventStoreMonitor, EventStoreMonitor::getTotalEvents)
                .description("이벤트 스토어 총 이벤트 수")
                .register(registry);
        };
    }

    // Prometheus AlertManager 규칙 (YAML로도 관리)
    // - alert: ProjectionLagHigh
    //   expr: projection_lag_seconds > 30
    //   → PagerDuty 알람

    // - alert: DLQNotEmpty
    //   expr: dlq_size > 0
    //   → Slack 알람 (즉시 확인 필요)
}
```

---

## 📊 패턴 비교

```
도메인별 CQRS + ES 적합도:

┌──────────────────────┬──────────────┬──────────────┬────────────┐
│ 도메인               │ CQRS 적합도  │ ES 적합도    │ 권장 수준  │
├──────────────────────┼──────────────┼──────────────┼────────────┤
│ 금융 (계좌, 결제)    │ ★★★★★      │ ★★★★★      │ CQRS + ES  │
├──────────────────────┼──────────────┼──────────────┼────────────┤
│ 전자상거래 주문      │ ★★★★★      │ ★★★★☆      │ CQRS + ES  │
├──────────────────────┼──────────────┼──────────────┼────────────┤
│ 재고 관리            │ ★★★★☆      │ ★★★★☆      │ CQRS + ES  │
├──────────────────────┼──────────────┼──────────────┼────────────┤
│ 헬스케어 기록        │ ★★★★☆      │ ★★★★★      │ CQRS + ES  │
├──────────────────────┼──────────────┼──────────────┼────────────┤
│ 배송/물류 추적       │ ★★★★☆      │ ★★★☆☆      │ CQRS 우선  │
├──────────────────────┼──────────────┼──────────────┼────────────┤
│ CMS (공지, 블로그)   │ ★★★☆☆      │ ★★☆☆☆      │ CQRS만     │
├──────────────────────┼──────────────┼──────────────┼────────────┤
│ 사용자 설정          │ ★★☆☆☆      │ ★☆☆☆☆      │ 상태 저장  │
├──────────────────────┼──────────────┼──────────────┼────────────┤
│ 세션 / 캐시          │ ★☆☆☆☆      │ ★☆☆☆☆      │ 적용 불가  │
└──────────────────────┴──────────────┴──────────────┴────────────┘
```

---

## ⚖️ 트레이드오프

```
CQRS 도입 ROI (Return on Investment):

비용:
  초기 구현: 기능 대비 2~3배 개발 시간
  학습: 6~12개월 (팀 역량 구축)
  운영: 모니터링 항목 30% 증가

편익:
  조회 성능: 읽기 모델 도입 후 10~100배 개선
  코드 명확성: Command/Query 분리로 유지보수 용이
  확장성: 읽기/쓰기 독립 스케일 아웃

ES 추가 시 ROI:
  추가 비용: ES 자체 복잡도 (스키마 진화, 스냅샷)
  추가 편익: 감사 로그 자동화, 시간 여행, Projection 재구축

손익분기점:
  CQRS 만: 4~6개월 (빠른 ROI)
  CQRS + ES: 12~18개월 (감사 로그 가치 실현 후)

판단:
  빠른 ROI 필요 → CQRS 먼저, ES 나중에
  감사 로그 즉시 필요 → CQRS + ES 동시
  ROI 불확실 → CQRS만, ES는 보류
```

---

## 📌 핵심 정리

```
CQRS / ES 비용과 편익 핵심:

적합한 도메인:
  감사 로그 법적/비즈니스 필수
  시간 여행 비즈니스 가치
  읽기/쓰기 비율 불균형 (읽기 >> 쓰기)
  도메인 이벤트 자연스럽게 표현 가능

부적합한 도메인:
  단순 CRUD (설정, 공지)
  도메인 불안정 (요구사항 매주 변경)
  소규모 팀 (3인 미만)
  감사 로그 불필요

현실적 비용:
  팀 학습: 6~12개월
  개발 생산성 초기 30~50% 감소
  운영 복잡도 30% 증가

현실적 편익:
  조회 성능 10~100배 개선 (읽기 모델)
  감사 로그 자동화 (ES)
  새 읽기 모델 추가 용이 (Projection)

손익분기점: 12~18개월
```

---

## 🤔 생각해볼 문제

**Q1.** "팀이 CQRS/ES에 자신감이 생기기까지 6~12개월이 걸린다"면, 초기 6개월 동안 팀의 생산성 감소를 최소화하는 방법은 무엇인가?

<details>
<summary>해설 보기</summary>

점진적 도입과 격리가 핵심입니다.

전체 시스템에 한 번에 CQRS/ES를 적용하지 않습니다. 가장 적합한 Aggregate 하나(예: BankAccount)에만 먼저 ES를 적용하고, 나머지는 기존 방식을 유지합니다. 팀이 하나의 Aggregate에서 패턴을 익히는 동안 다른 기능 개발은 계속됩니다.

Axon이나 잘 만들어진 라이브러리를 활용합니다. 이벤트 스토어, Command Bus, Projection Runner를 직접 구현하면 학습 시간이 배로 늘어납니다. 검증된 라이브러리로 보일러플레이트를 줄입니다.

페어 프로그래밍과 코드 리뷰를 강화합니다. CQRS/ES 관련 코드는 반드시 경험 있는 팀원과 함께 작성합니다. 패턴이 몸에 익기 전에 혼자 작성하면 디버깅에 시간이 더 걸립니다.

스파이크(Spike) 타임을 확보합니다. 처음 2~4주는 프로토타입 구현에 집중합니다. 이 기간의 낮은 생산성은 이후 6개월의 효율성으로 보상됩니다.

</details>

---

**Q2.** CQRS/ES 도입을 완료한 후 "도입하지 말았어야 했다"고 판단할 수 있는 기준은 무엇인가?

<details>
<summary>해설 보기</summary>

도입 후 6~12개월 시점에서 평가하는 구체적인 기준들입니다.

첫째, 비즈니스 가치 미실현입니다. ES의 핵심 편익(감사 로그, 시간 여행, Projection 재구축)을 실제로 사용한 적이 없다면 ES 비용만 지불한 것입니다. 예를 들어 "Projection 재구축 기능이 있지만 한 번도 사용하지 않았고, 감사 로그는 요구사항이 없었다"면 ES를 제거를 검토합니다.

둘째, 팀의 지속적인 어려움입니다. 도입 12개월 후에도 팀원들이 Projection 장애 원인 파악에 2시간 이상 걸리거나, 이벤트 스키마 변경할 때마다 공포감을 느낀다면 팀에게 맞지 않는 복잡도입니다.

셋째, 유지보수 비용 증가입니다. 기능 추가 시간이 ES 도입 전보다 오히려 더 길어졌다면 복잡도가 이점을 초과한 것입니다.

이 경우 실용적인 해결책은 ES를 완전 제거하기보다 단계적으로 간소화합니다. Axon Server를 제거하고 단순한 이벤트 스토어로 교체하거나, ES가 필요 없는 Aggregate는 상태 저장으로 전환합니다. "ES를 도입했으니 끝까지 유지해야 한다"는 매몰 비용 오류에 빠지지 않는 것이 중요합니다.

</details>

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[이전: 안티패턴 ⬅️](./04-anti-patterns.md)**

</div>
