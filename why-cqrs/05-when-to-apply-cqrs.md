# CQRS 적용 판단 기준

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 어떤 도메인 특성이 있을 때 CQRS가 편익보다 비용이 적은가?
- 도메인 복잡도, 읽기-쓰기 비율, 팀 역량 각각이 판단에 어떤 가중치를 갖는가?
- CQRS 없이도 해결할 수 있는 문제는 어떤 것들인가?
- CQRS 도입 비용이 편익을 초과하는 안티 사례는 어떤 패턴인가?
- 오늘 결정을 내려야 한다면, 어떤 체크리스트로 판단하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

CQRS가 나쁜 이유는 없다. 하지만 CQRS가 필요하지 않은 곳에 도입하면 팀이 지불할 비용이 있다. 이벤트 기반 동기화, Projection 관리, 최종 일관성 처리, 이벤트 스키마 진화. 이 모든 것은 팀의 시간을 도메인 비즈니스 로직이 아닌 인프라 관리에 쏟게 만든다.

반대로 CQRS가 필요한 곳에 도입하지 않으면 단일 모델이 만드는 N+1, Entity 오염, 잠금 충돌이 점점 더 심해진다. 이 문서는 "언제 도입해야 하는가"와 "언제 도입하지 말아야 하는가"를 구분하는 기준을 제시한다.

---

## 😱 흔한 실수 (Before — 잘못된 판단으로 도입하거나 도입하지 않는 경우)

```
실수 1: 모든 서비스에 CQRS 도입
  팀 결정: "마이크로서비스 전환하면서 모든 서비스에 CQRS + Event Sourcing 도입"

  대상 서비스 중 하나: 사용자 프로필 관리
    - API: 프로필 조회, 이름 수정, 이메일 수정, 비밀번호 변경
    - 트래픽: 쓰기 10%, 읽기 90%
    - 도메인 복잡도: 단순 CRUD

  결과:
    UserUpdated 이벤트 → Projection → profile_summary 업데이트
    프로필 수정: 이벤트 스토어 → Kafka → Projection 처리 → 읽기 모델 반영
    개발자: "비밀번호 변경 후 프로필이 왜 바로 안 바뀌지?"
    팀: 이벤트 스키마 버전 관리 + Projection 재구축 + 이벤트 스토어 운영
    비즈니스 가치: 0 (단순 CRUD를 복잡하게 만든 것)

실수 2: CQRS가 필요한 곳에 도입하지 않음
  "복잡하니까 나중에 도입하자" → 2년 후 기술 부채

  대상 서비스: 주문 처리 (일평균 주문 10만 건)
    - 읽기: 주문 목록, 주문 상태, 배송 추적, 통계 대시보드
    - 쓰기: 주문 생성, 확인, 취소, 반품

  2년 후 상황:
    Order Entity: 3,000줄, 연관관계 15개
    주문 목록 API: 8 테이블 JOIN, 평균 800ms
    이체 처리 중 주문 조회 지연: 초당 3건
    관리자/CS/배송 팀 뷰가 모두 같은 API → 최적화 불가
    "이제 분리하려면 얼마나 걸려요?" → 3개월 추산
```

---

## ✨ 올바른 접근 (After — 기준에 따른 판단)

```
판단 체크리스트:

✅ CQRS 도입이 권장되는 신호 (3개 이상이면 강력히 권장):
  □ 조회 쿼리에 3개 이상 테이블 JOIN이 반복 등장한다
  □ 쓰기와 읽기의 트래픽 비율이 1:5 이상 불균형하다
  □ 역할별로 완전히 다른 뷰(관리자/사용자/통계)가 필요하다
  □ 도메인 불변식이 있고 조회 로직이 그 불변식을 오염시킨다
  □ 감사 로그, 시간 여행, 완전한 이력 추적이 필요하다
  □ 읽기 부하와 쓰기 부하를 독립적으로 스케일 아웃해야 한다
  □ 팀이 이벤트 기반 아키텍처에 익숙하거나 학습 의지가 있다

❌ CQRS 도입이 과잉인 신호 (1개라도 해당되면 재고):
  □ 도메인이 단순 CRUD다 (사용자 설정, 공지사항, FAQ)
  □ 읽기-쓰기 요구사항이 거의 동일하다 (같은 필드를 쓰고 읽는다)
  □ 팀이 3명 이하이고 학습 비용을 감당하기 어렵다
  □ 조회 성능 문제가 QueryDSL DTO 프로젝션으로 해결된다
  □ 강한 일관성이 필요하고 최종 일관성을 허용할 수 없다
  □ 서비스 초기 단계로 요구사항이 아직 불명확하다
```

---

## 🔬 내부 동작 원리

### 1. 판단 기준 1 — 도메인 복잡도

```
CQRS가 편익을 제공하는 도메인 복잡도 임계점:

복잡도 낮음 (CQRS 불필요):
  특징:
    상태 전이가 단순 (생성 → 수정 → 삭제)
    불변식이 없거나 자명함 (빈 필드 없어야 함 정도)
    조회 = 쓴 것 그대로 읽기
  예시:
    공지사항 관리 (작성/수정/삭제/조회)
    사용자 설정 (알림 켜기/끄기)
    FAQ 관리

  CQRS 없는 해결:
    Spring Data JPA + DTO 프로젝션으로 충분
    복잡도 추가 불필요

복잡도 중간 (Simple CQRS ~ 저장소 분리):
  특징:
    상태 전이가 있고 불변식이 있음
    조회 패턴이 쓰기 패턴과 다름 (다양한 필터, 집계)
    역할별 뷰 차이 있음 (관리자/사용자)
  예시:
    상품 관리 (상품 등록 → 심사 → 게시 → 품절)
    회원 관리 (가입 → 활성 → 정지 → 탈퇴)

  권장 수준:
    Simple CQRS (QueryDSL DTO 프로젝션)
    또는 저장소 분리 (읽기 모델 테이블 분리)

복잡도 높음 (Event-Driven CQRS 권장):
  특징:
    복잡한 상태 머신 (10개 이상 상태 전이)
    강한 불변식 (금전적 일관성, 재고 일관성)
    다양한 역할과 뷰 (관리자/CS/배송/회계/통계)
    완전한 감사 로그 필요
    과거 시점 상태 재현 필요
  예시:
    은행 계좌 (입금/출금/이체/동결/해지)
    주문 처리 (생성/결제/확인/배송/반품/환불)
    재고 관리 (입고/출고/조정/예약/반납)

  권장 수준:
    Event-Driven CQRS + Event Sourcing
```

### 2. 판단 기준 2 — 읽기-쓰기 비율

```
비율에 따른 CQRS 편익:

쓰기 비중 높음 (쓰기 : 읽기 = 1:1 ~ 2:1):
  IoT 센서 데이터 수집 (쓰기가 대부분)
  실시간 로그 수집

  CQRS 편익: 낮음
    읽기 모델 최적화보다 쓰기 파이프라인 최적화가 중요
    읽기 모델 동기화 비용이 높은 쓰기 빈도와 비례

읽기 비중 높음 (쓰기 : 읽기 = 1:10 이상):
  전자상거래 상품 조회 (읽기 압도적)
  소셜 피드 (읽기 중심)
  대시보드/통계

  CQRS 편익: 높음
    읽기 모델을 쓰기와 독립적으로 최적화 가능
    읽기 DB만 수평 확장 가능 (Read Replica)
    읽기 모델 캐싱이 쓰기에 영향 없음

판단 공식:
  쓰기 초당 100건, 읽기 초당 1,000건:
    읽기 부하가 10배 → 읽기 DB 독립 스케일 아웃 가치 있음
    읽기 모델 최적화로 읽기 응답 시간 80% 개선 가능

  쓰기 초당 50건, 읽기 초당 60건:
    비율이 비슷 → 읽기 최적화 편익이 크지 않음
    CQRS 비용이 편익 초과할 가능성

실제 전자상거래 트래픽 예시:
  상품 등록/수정: 분당 10건 (쓰기)
  상품 조회: 분당 5,000건 (읽기)
  주문 생성: 분당 200건 (쓰기)
  주문 조회: 분당 2,000건 (읽기)
  → 읽기가 압도적 → CQRS 편익 명확
```

### 3. 판단 기준 3 — 팀 역량

```
팀 역량과 CQRS 수준 매핑:

역량 낮음 (이벤트 기반 경험 없음):
  권장: Simple CQRS만 (QueryDSL DTO 프로젝션)
  이유:
    Projection, 이벤트 스토어, 최종 일관성을 동시에 학습하면
    도메인 개발보다 아키텍처 이해에 시간을 더 쏟게 됨
    결과: 이해도 낮은 코드 + 운영 장애

역량 중간 (Spring, 메시지 큐 경험 있음):
  권장: 저장소 분리 CQRS (Spring Events or Kafka 기반 Projection)
  이유:
    Spring Application Event + @EventListener는 진입 장벽 낮음
    이미 Kafka를 다뤄봤다면 Projection 패턴으로 자연스럽게 연결

역량 높음 (DDD, Event Sourcing 경험 있음):
  권장: Event-Driven CQRS + Axon Framework
  이유:
    Axon이 이벤트 스토어, Projection, Command Bus 표준화
    팀이 패턴을 이해하고 있어 추상화 레이어가 부담이 아님

팀 역량 평가 질문:
  ① 이벤트 기반 아키텍처를 프로덕션에서 운영해본 경험이 있는가?
  ② Kafka Consumer Group 재배치, 오프셋 관리를 이해하는가?
  ③ 최종 일관성이 UX에 미치는 영향을 설계할 수 있는가?
  ④ Projection 실패 시 재처리 전략을 설계할 수 있는가?
  → 4개 중 3개 이상: Event-Driven CQRS 가능
  → 2개: 저장소 분리 CQRS
  → 1개 이하: Simple CQRS
```

### 4. CQRS 없이 해결할 수 있는 문제들

```
CQRS 도입 전 먼저 시도해볼 것들:

문제: N+1 쿼리
  CQRS 전 시도:
    @EntityGraph(attributePaths = {"lines", "customer"})
    JPQL fetch join
    QueryDSL DTO 프로젝션
    Batch Fetch Size 설정 (spring.jpa.properties.hibernate.default_batch_fetch_size)
  → 이 방법들로 해결된다면 CQRS 불필요

문제: 복잡한 조회 쿼리
  CQRS 전 시도:
    QueryDSL Q-class 기반 동적 쿼리
    Native Query + RowMapper
    별도 읽기 전용 View (DB View, Materialized View)
    Read Replica 활용
  → Materialized View로 해결된다면 CQRS 불필요 (DB가 동기화 담당)

문제: 읽기 성능
  CQRS 전 시도:
    Redis 캐시 레이어 추가 (Spring Cache + @Cacheable)
    DB 인덱스 최적화
    Read Replica + 로드밸런서
    CDN + HTTP 캐시 (변경이 적은 데이터)
  → 캐시로 해결된다면 CQRS 불필요

문제: 역할별 뷰
  CQRS 전 시도:
    같은 Entity에서 역할별 DTO 변환 (Projection Interface)
    역할별 다른 응답 필드 선택 (@JsonView)
    별도 API 엔드포인트 + 같은 Service 레이어
  → DTO 변환으로 해결된다면 CQRS 불필요

진짜 CQRS가 필요한 신호:
  위 방법들을 모두 시도했지만:
    "조회 패턴이 너무 다양해서 어떤 인덱스도 모든 쿼리를 커버하지 못한다"
    "Materialized View 갱신 주기로는 실시간성을 보장할 수 없다"
    "캐시 무효화 로직이 쓰기 트랜잭션에 너무 많이 침투했다"
    "역할별 뷰가 너무 달라 같은 Entity에서 파생하는 것이 불가능하다"
```

### 5. CQRS 도입 비용이 편익을 초과하는 안티 사례

```
안티 사례 1: 간단한 블로그 플랫폼
  상황: 포스트 작성, 수정, 삭제, 조회
  CQRS 도입:
    PostCreated, PostUpdated, PostDeleted 이벤트
    post_summary Projection
    이벤트 스토어

  현실:
    "포스트 수정 후 목록에서 바로 안 보여요" (최종 일관성)
    포스트 내용 수정 → 이벤트 스키마 변경 → Projection 재구축
    팀 2명이 이벤트 스토어 운영
    비즈니스 요구사항: 포스트를 쓰고 바로 읽는 것

  결론: JPA + Spring Data로 30분에 완성할 것을 3주 소요

안티 사례 2: MVP 단계 스타트업
  상황: 초기 서비스 런칭, 요구사항이 매주 변경
  CQRS + Event Sourcing 도입:
    이벤트 스키마 설계 → 1주일 후 요구사항 변경 → 이벤트 스키마 변경
    Upcasting, Projection 재구축이 매주 발생

  현실:
    "제품-시장 적합성을 찾는 시간에 이벤트 스키마 마이그레이션을 하고 있었다"
    이벤트 스키마는 영구적 결정 → MVP 단계에서 부담이 너무 큼

  결론: 도메인이 안정화된 후 도입해도 늦지 않음

안티 사례 3: 강한 일관성이 필요한 금융 결제
  상황: 결제 완료 즉시 잔고 변경이 화면에 반영되어야 함
  CQRS 도입:
    PaymentCompleted 이벤트 → 읽기 모델 업데이트 (50ms 지연)

  현실:
    사용자: "결제했는데 왜 잔고가 안 줄었죠?" (50ms 지연이지만 체감)
    UX 해결책: 낙관적 UI 업데이트 (복잡도 증가)

  결론:
    이 케이스만을 위해 CQRS를 선택하면 안 됨
    잔고 확인은 쓰기 모델에서 수행, 표시는 Command 응답 활용

안티 사례 4: 팀 전체가 동의하지 않은 CQRS 도입
  상황: 아키텍트 1명이 CQRS를 결정, 나머지 팀원은 이해 부족
  결과:
    Projection 실패 시 원인 파악 불가
    최종 일관성 이슈를 버그로 신고
    이벤트 스토어 모니터링 없음 → 장애 감지 불가
    "이거 왜 이렇게 복잡해요?" → 기술 부채로 인식
```

---

## 💻 실전 코드

### 판단 분기에 따른 구현 선택

```java
// ==============================
// 경우 1: CQRS 불필요 — 단순 CRUD
// ==============================

// 블로그 포스트: JPA + Spring Data만으로 충분
@Service
public class PostService {

    // 쓰기와 읽기에 같은 JPA Entity 사용
    @Transactional
    public PostDto createPost(CreatePostRequest request) {
        Post post = Post.create(request.getTitle(), request.getContent(),
                                request.getAuthorId());
        Post saved = postRepository.save(post);
        return PostDto.from(saved);
    }

    public Page<PostSummaryDto> getPosts(Pageable pageable) {
        // JPQL DTO 프로젝션 — 필요한 컬럼만 SELECT
        return postRepository.findAllSummaries(pageable);
    }
}

@Query("SELECT new com.example.PostSummaryDto(p.id, p.title, p.authorName, p.createdAt) " +
       "FROM Post p ORDER BY p.createdAt DESC")
Page<PostSummaryDto> findAllSummaries(Pageable pageable);

// ==============================
// 경우 2: Simple CQRS — 조회 복잡도만 있는 경우
// ==============================

// 상품 관리: 쓰기는 JPA, 읽기는 QueryDSL DTO 프로젝션
@Service
public class ProductCommandService {

    @Transactional
    public void handle(CreateProductCommand command) {
        Product product = Product.create(command);
        productRepository.save(product);
    }
}

@Service
public class ProductQueryService {

    // QueryDSL로 복잡한 조건 쿼리
    public Page<ProductListDto> searchProducts(ProductSearchCondition condition, Pageable pageable) {
        return productQueryRepository.search(condition, pageable);
        // 같은 DB, 별도 읽기 전용 Repository
    }
}

// ==============================
// 경우 3: 저장소 분리 CQRS — 읽기 부하 분산이 필요한 경우
// ==============================

// 주문 관리: 쓰기 DB와 읽기 DB 분리
@Service
public class OrderCommandService {

    @Transactional
    public void handle(ConfirmOrderCommand command) {
        Order order = orderRepository.findById(command.getOrderId()).orElseThrow();
        order.confirm(inventoryChecker);
        orderRepository.save(order);

        // 이벤트 발행 — Projection 트리거
        eventPublisher.publish(OrderConfirmedEvent.from(order));
    }
}

@Component
public class OrderSummaryProjection {

    // 이벤트 소비 → 읽기 DB 업데이트
    @EventHandler
    @Transactional("readTransactionManager")  // 읽기 DB 트랜잭션
    public void on(OrderConfirmedEvent event) {
        OrderSummaryView summary = buildSummaryFrom(event);
        orderSummaryRepository.save(summary);
    }
}

// ==============================
// 경우 4: Event-Driven CQRS — 감사 로그, 시간 여행이 필요한 경우
// ==============================

// 은행 계좌: Axon Framework + Event Sourcing
@Aggregate
public class Account {

    @AggregateIdentifier
    private AccountId accountId;
    private Money balance;
    private AccountStatus status;

    @CommandHandler
    public void handle(WithdrawMoneyCommand command) {
        // 불변식 검증만 — 조회 관심사 없음
        if (balance.isLessThan(command.getAmount()))
            throw new InsufficientBalanceException();
        if (status == AccountStatus.FROZEN)
            throw new AccountFrozenException();

        apply(new MoneyWithdrawn(accountId, command.getAmount(),
                                  balance.subtract(command.getAmount())));
    }

    @EventSourcingHandler
    public void on(MoneyWithdrawn event) {
        this.balance = event.getBalanceAfter();
    }
}

// 판단 기준 요약:
// 도메인 복잡도 낮음 + 불변식 없음 → 경우 1 (JPA만)
// 조회 쿼리 복잡 + 불변식 있음 → 경우 2 (Simple CQRS)
// 읽기-쓰기 부하 불균형 + 비정규화 필요 → 경우 3 (저장소 분리)
// 감사 로그 + 시간 여행 + 복잡한 상태 전이 → 경우 4 (Event-Driven)
```

---

## 📊 패턴 비교

```
도메인별 CQRS 적용 수준 가이드:

┌─────────────────────────────┬──────────────┬────────────────┬────────────────────┐
│ 도메인 예시                  │ 권장 수준     │ 주요 이유       │ 대안               │
├─────────────────────────────┼──────────────┼────────────────┼────────────────────┤
│ 공지사항, FAQ, 설정          │ 없음 (CRUD)   │ 단순 상태,      │ JPA + DTO 프로젝션  │
│                             │              │ 불변식 없음    │                    │
├─────────────────────────────┼──────────────┼────────────────┼────────────────────┤
│ 상품 관리, 회원 관리          │ Simple CQRS  │ 조회 복잡,      │ QueryDSL           │
│                             │              │ 불변식 있음    │ Materialized View  │
├─────────────────────────────┼──────────────┼────────────────┼────────────────────┤
│ 주문 처리, 배달 관리          │ 저장소 분리   │ 읽기-쓰기 불균형 │ CDC(Debezium) +    │
│                             │ CQRS         │ 역할별 뷰 다양  │ 읽기 모델 테이블    │
├─────────────────────────────┼──────────────┼────────────────┼────────────────────┤
│ 은행 계좌, 재고 관리,         │ Event-Driven │ 감사 로그 필수  │ -                  │
│ 예약 시스템                  │ CQRS + ES    │ 시간 여행 필요 │                    │
│                             │              │ 복잡한 상태 전이│                    │
└─────────────────────────────┴──────────────┴────────────────┴────────────────────┘

판단 매트릭스:

도메인 복잡도:
  낮음(CRUD)  중간(불변식)  높음(복잡한 상태전이)
     │            │                 │
     ▼            ▼                 ▼
    없음      Simple CQRS     Event-Driven

읽기-쓰기 비율:
  1:1 ~ 1:3   1:5 ~ 1:10   1:10 이상
     │             │            │
     ▼             ▼            ▼
    없음        저장소 분리    저장소 분리 + 스케일 아웃

감사 로그 / 시간 여행 필요:
  없음                       있음
   │                          │
   ▼                          ▼
  현재 수준 유지          Event Sourcing 추가
```

---

## ⚖️ 트레이드오프

```
CQRS 도입의 총비용 계산:

개발 비용:
  Command 모델 설계: 2주
  이벤트 설계 + 스키마: 1주
  Projection 구현: 팀 규모에 따라 1~4주
  최종 일관성 UX 처리: 1~2주
  테스트 (이벤트 기반): 추가 30%

운영 비용 (월간):
  이벤트 스토어 모니터링: 4시간
  Projection 재구축 (분기 1회): 8시간
  이벤트 스키마 마이그레이션 (반기 1회): 16시간

편익 계산:
  읽기 응답 시간 개선: 기존 800ms → 20ms (95% 개선)
  쓰기 트랜잭션 복잡도 감소: 기존 50줄 → 20줄
  새 역할별 뷰 추가 시간: 기존 3주 → 3일 (Projection만 추가)
  감사 로그 문제 발생 시 대응 시간: 기존 3일 → 3시간

편익 > 비용인 경우:
  도메인이 복잡하고 오래 유지될 서비스
  읽기 부하가 높고 다양한 조회 패턴이 있는 서비스
  감사 로그가 비즈니스 핵심 요구사항인 서비스

비용 > 편익인 경우:
  단순 CRUD가 대부분인 서비스
  MVP 단계로 요구사항이 자주 변하는 서비스
  팀 규모가 작고 운영 여력이 없는 경우
```

---

## 📌 핵심 정리

```
CQRS 적용 판단 기준 요약:

도입 권장:
  ✅ 도메인 불변식이 있고 조회 요구사항이 그것을 오염시킴
  ✅ 읽기-쓰기 비율이 1:5 이상
  ✅ 역할별 완전히 다른 뷰가 3개 이상 필요
  ✅ 감사 로그 / 시간 여행이 핵심 요구사항
  ✅ 팀이 이벤트 기반 아키텍처를 운영할 역량이 있음

도입 보류:
  ❌ 단순 CRUD 도메인
  ❌ MVP 단계 (요구사항 불안정)
  ❌ 강한 일관성이 반드시 필요
  ❌ 팀이 이벤트 기반 아키텍처에 미숙
  ❌ QueryDSL, Materialized View로 문제 해결 가능

수준 선택 원칙:
  현재 겪는 구체적인 고통 → 그것을 해결하는 최소 수준
  Simple CQRS → 저장소 분리 → Event-Driven 순서로 점진적 도입

핵심 질문:
  "지금 겪는 문제가 무엇인가?"
  "그 문제가 CQRS 없이 해결 가능한가?"
  "CQRS 도입 비용을 팀이 감당할 수 있는가?"
```

---

## 🤔 생각해볼 문제

**Q1.** "우리 서비스는 읽기가 99%, 쓰기가 1%입니다. CQRS를 도입해야 하나요?" 어떻게 판단하겠는가?

<details>
<summary>해설 보기</summary>

읽기-쓰기 비율만으로 CQRS를 판단하면 안 됩니다. 추가 정보가 필요합니다.

먼저 읽기 패턴이 다양한가를 봐야 합니다. 모든 읽기가 같은 데이터를 같은 형태로 조회한다면(단순 목록 + 상세), CQRS가 불필요합니다. 반면 역할별로 다른 뷰가 필요하고 집계 쿼리가 복잡하다면 CQRS가 유효합니다.

쓰기가 1%라도 불변식이 있는가를 확인해야 합니다. 쓰기가 거의 없더라도 쓰기 시 복잡한 불변식이 있다면(재고 차감, 잔고 검증) 쓰기 모델 분리가 의미 있습니다.

현재 읽기 성능에 문제가 있는가도 중요합니다. 현재 읽기 응답 시간이 충분히 빠르고 N+1이 없다면, 읽기 비중이 높더라도 CQRS 없이 유지하는 것이 합리적입니다.

결론적으로 읽기 99%는 CQRS 도입의 충분조건이 아닙니다. 읽기 패턴의 복잡성, 현재 성능 이슈 여부, 팀 역량을 함께 고려해야 합니다.

</details>

---

**Q2.** 팀이 CQRS를 도입하기로 결정했다. 첫 번째로 분리할 조회는 어떤 것을 선택해야 하는가?

<details>
<summary>해설 보기</summary>

"가장 고통스러운 조회"를 첫 번째 대상으로 선택하는 것이 좋습니다.

판단 기준으로 다음 세 가지를 활용합니다. 첫째, 현재 조회 응답 시간이 가장 긴 것을 선택합니다. 이 조회를 개선하면 팀이 CQRS의 편익을 즉시 체감할 수 있습니다. 둘째, JOIN이 가장 많은 조회를 선택합니다. 읽기 모델로 분리했을 때 단순 SELECT로 바뀌는 효과가 가장 극적입니다. 셋째, 쓰기와 독립적으로 스케일 아웃이 필요한 조회를 선택합니다.

피해야 할 첫 번째 대상은 강한 일관성이 필요한 조회입니다. "결제 직후 잔고 조회"처럼 최종 일관성 지연이 즉시 문제가 되는 것을 첫 번째로 선택하면, 팀이 CQRS의 복잡도만 경험하고 편익은 보지 못합니다.

"주문 목록 대시보드"처럼 복잡한 JOIN이 있고, 수 초의 지연이 허용 가능하고, 쓰기 횟수가 적은 것을 첫 번째로 선택하는 것이 좋습니다.

</details>

---

**Q3.** 6개월 후 도메인이 성장해서 Simple CQRS에서 Event-Driven CQRS로 업그레이드가 필요해졌다. 이 마이그레이션을 무중단으로 수행할 수 있는가?

<details>
<summary>해설 보기</summary>

가능하지만 계획이 필요합니다.

단계별 마이그레이션 전략이 있습니다.

1단계로 이벤트 발행을 먼저 추가합니다. 기존 쓰기 코드는 유지하면서 Command 처리 완료 후 Domain Event를 발행하는 코드를 추가합니다. 처음엔 이벤트를 소비하는 쪽이 없어도 됩니다.

2단계로 새 읽기 모델과 Projection을 추가합니다. 이벤트를 소비하는 Projection을 추가하고, 새 읽기 모델 테이블을 생성합니다. 이 시점에 기존 JPA 기반 읽기와 새 읽기 모델이 병렬 운영됩니다.

3단계로 이벤트 스토어로 전환합니다. 기존 쓰기 DB를 이벤트 스토어로 마이그레이션합니다. 기존 상태를 이벤트로 변환하는 마이그레이션 스크립트가 필요합니다("현재 잔고 500,000원"을 "AccountOpened + MoneyDeposited 이벤트 시퀀스"로 변환).

4단계로 기존 읽기 코드를 새 읽기 모델로 전환합니다.

가장 어려운 부분은 3단계로, 기존 상태에서 이벤트로의 변환은 비즈니스 규칙을 역공학해야 합니다. 이 때문에 처음부터 이벤트를 발행하도록 설계하는 것(1단계)이 중요합니다.

</details>

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[이전: CQRS가 해결하는 실제 문제 ⬅️](./04-problems-cqrs-solves.md)** | **[다음: Chapter 2 — Command와 Query의 명확한 분리 ➡️](../command-side/01-command-query-separation.md)**

</div>
