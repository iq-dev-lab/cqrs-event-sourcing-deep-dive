# Event Sourcing의 핵심 아이디어

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 현재 상태를 저장하는 방식의 본질적 한계는 무엇인가?
- "이벤트가 진실의 원천"이라는 것이 구체적으로 무엇을 의미하는가?
- Append-Only 저장이 가져오는 구조적 이점은 무엇인가?
- 회계 장부 원칙이 Event Sourcing 설계에 어떻게 적용되는가?
- 상태 저장 vs 이벤트 저장의 트레이드오프는 무엇인가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

현재 상태만 저장하는 시스템은 과거를 잃는다. 계좌 잔고가 500,000원인 이유를 모르고, 어제 오후 3시의 잔고도 모른다. 감사팀이 "이 계좌에서 왜 돈이 줄었나요?"라고 물으면 답할 수 없다. Event Sourcing은 이 문제를 "현재 상태 대신 과거의 모든 변화를 저장"하는 방식으로 해결한다.

---

## 😱 흔한 실수 (Before — 현재 상태만 저장)

```
은행 계좌 상태 저장 방식:

  accounts 테이블:
  ┌──────────┬─────────┬──────────┐
  │ id       │ balance │ status   │
  ├──────────┼─────────┼──────────┤
  │ ACC-001  │ 500000  │ ACTIVE   │
  └──────────┴─────────┴──────────┘

  이체 처리:
    UPDATE accounts SET balance = 400000 WHERE id = 'ACC-001'
    → 500000은 영원히 사라짐

  나중에 발생하는 질문들:
    "왜 잔고가 400,000원인가?" → 모름
    "어제 오후 3시의 잔고는?" → 모름
    "누가 언제 얼마를 이체했나?" → 별도 로그 테이블 구축 필요
    "이 입금이 정상인가?" → 원본 데이터 없음
    "잔고가 음수였던 순간이 있었나?" → 확인 불가

  히스토리 테이블로 해결?:
    account_history 테이블: 변경 시마다 이전 값 저장
    문제: 원본 테이블과 동기화 관리 필요
          "왜 바뀌었는가"(원인)는 여전히 알 수 없음
          부분 동기화 버그 발생 가능
```

---

## ✨ 올바른 접근 (After — 이벤트를 진실의 원천으로)

```
Event Sourcing 방식:

  event_store 테이블 (stream: "account-ACC-001"):
  ┌─────────┬──────────────────┬──────────────────────────────────┐
  │ version │ event_type       │ payload                          │
  ├─────────┼──────────────────┼──────────────────────────────────┤
  │ 1       │ AccountOpened    │ { owner: "USER-42", initial: 0 } │
  │ 2       │ MoneyDeposited   │ { amount: 500000, by: "ATM" }    │
  │ 3       │ MoneyTransferred │ { to: "ACC-002", amount: 100000 }│
  │ 4       │ MoneyDeposited   │ { amount: 200000, by: "WIRE" }   │
  └─────────┴──────────────────┴──────────────────────────────────┘

  현재 잔고: 이벤트 리플레이로 계산
    0 → 500000 → 400000 → 600000 (현재)

  모든 질문에 답 가능:
    "왜 600,000인가?" → 이벤트 4개 순서로 계산
    "2024-01-15 오후 3시 잔고?" → 그 시점 이전 이벤트만 리플레이
    "누가 언제 이체?" → event_type + payload + occurred_at
    "감사 로그?" → 이벤트 스트림 자체가 완전한 감사 로그
```

---

## 🔬 내부 동작 원리

### 1. 회계 장부 원칙 — Event Sourcing의 원형

```
복식부기 회계 장부:
  "현재 잔액"이 아닌 "모든 거래 이력"이 원본

  거래 원장:
  ┌──────────┬────────┬────────┬──────────┬────────────┐
  │ 날짜     │ 적요   │ 차변   │ 대변     │ 잔액       │
  ├──────────┼────────┼────────┼──────────┼────────────┤
  │ 01/01   │ 개설   │        │ 500,000  │ 500,000    │
  │ 01/05   │ 출금   │ 100,000│          │ 400,000    │
  │ 01/10   │ 입금   │        │ 200,000  │ 600,000    │
  └──────────┴────────┴────────┴──────────┴────────────┘

  원칙:
    ① 한 번 기록된 거래는 삭제/수정 불가 (Append Only)
    ② 실수는 정정 거래로 기록 (역방향 이벤트 추가)
    ③ 현재 잔액 = 모든 거래의 합산 (파생 데이터)

Event Sourcing = 소프트웨어 회계 장부:
  회계: "거래 기록이 원본, 잔액은 계산"
  ES: "이벤트가 원본, 현재 상태는 계산"

  회계 원칙 1 → ES: 이벤트 삭제/수정 불가
  회계 원칙 2 → ES: 실수 정정 = 보상 이벤트 추가
    CancelMoneyTransfer → 이전 MoneyTransferred를 취소하는 이벤트
  회계 원칙 3 → ES: 현재 상태 = 이벤트 리플레이
```

### 2. 상태 저장 vs 이벤트 저장 — 근본적 차이

```
상태 저장 (State-Based):
  "현재 어떤 상태인가"를 저장

  장점: 단순, 현재 상태 조회 빠름, 업데이트 직관적
  단점: 과거 사라짐, 감사 로그 별도 구현, 시간 여행 불가

  UPDATE account SET balance = 600000
  → 이전 값 500000은 사라짐

이벤트 저장 (Event-Sourced):
  "어떤 변화가 있었는가"를 저장

  장점: 완전한 이력, 시간 여행, 새로운 읽기 모델 재구축
  단점: 현재 상태 계산 필요, 이벤트 스키마 진화 어려움

  INSERT INTO events (type='MoneyDeposited', amount=200000)
  → 이전 이벤트 보존, 새 이벤트 추가
```

### 3. Append-Only의 구조적 이점

```
Append-Only 저장의 장점:

  ① 동시성 단순화:
     INSERT는 충돌 없음 (같은 위치에 동시 쓰기 없음)
     낙관적 잠금 = UNIQUE(stream_id, version) 위반 체크만
     UPDATE의 복잡한 동시성 문제 없음

  ② 캐시 효율:
     이벤트는 한 번 쓰면 변하지 않음 → 무기한 캐시 가능
     상태 저장: 매번 갱신 → 캐시 무효화 필요

  ③ 감사 로그 내장:
     이벤트 자체가 "누가 언제 무엇을 변경했는가"의 완전한 기록
     별도 audit_log 테이블 불필요

  ④ 디버깅:
     프로덕션 이슈 → 이벤트 스트림을 로컬에서 리플레이
     "그 시점에 정확히 어떤 상태였는가" 재현 가능

  ⑤ 시간 여행:
     "2024-01-15 오후 3시 잔고"
     → 이 시점 이전 이벤트만 리플레이 → 정확한 과거 상태
```

---

## 💻 실전 코드

```java
// 상태 저장 방식
@Transactional
public void withdrawStateBase(String accountId, BigDecimal amount) {
    Account account = accountRepository.findById(accountId).orElseThrow();
    account.setBalance(account.getBalance().subtract(amount)); // 상태 변경
    accountRepository.save(account); // UPDATE — 이전 값 사라짐
}

// Event Sourcing 방식
@Transactional
public void withdrawEventSourced(String accountId, BigDecimal amount) {
    // 이벤트 로드 → 상태 재현
    List<DomainEvent> events = eventStore.loadEvents("account-" + accountId);
    Account account = Account.reconstitute(events);

    // 불변식 검증 → 이벤트 생성
    account.withdraw(Money.of(amount)); // 내부: MoneyWithdrawn 이벤트 등록

    // 새 이벤트만 저장 (기존 이벤트 보존)
    eventStore.appendEvents("account-" + accountId,
        account.getPendingEvents());
}

// Account.reconstitute — 이벤트 리플레이
public static Account reconstitute(List<DomainEvent> events) {
    Account account = new Account();
    events.forEach(event -> account.apply(event)); // 이벤트 순서대로 상태 재현
    return account;
}

private void apply(DomainEvent event) {
    if (event instanceof AccountOpened e) {
        this.id = e.getAccountId();
        this.balance = Money.ZERO;
        this.status = AccountStatus.ACTIVE;
    } else if (event instanceof MoneyDeposited e) {
        this.balance = this.balance.add(e.getAmount());
    } else if (event instanceof MoneyWithdrawn e) {
        this.balance = this.balance.subtract(e.getAmount());
    }
}

// 시간 여행: 특정 시점의 잔고
public Money getBalanceAt(String accountId, Instant targetTime) {
    List<DomainEvent> events = eventStore.loadEventsBefore(
        "account-" + accountId, targetTime);
    Account account = Account.reconstitute(events);
    return account.getBalance();
}
```

---

## 📊 패턴 비교

```
상태 저장 vs Event Sourcing:

┌───────────────────────┬──────────────────┬──────────────────────┐
│ 항목                  │ 상태 저장        │ Event Sourcing       │
├───────────────────────┼──────────────────┼──────────────────────┤
│ 저장 내용             │ 현재 상태        │ 변화 이력 (이벤트)   │
├───────────────────────┼──────────────────┼──────────────────────┤
│ 현재 상태 조회        │ 빠름 (단순 SELECT)│ 느림 (리플레이 필요) │
├───────────────────────┼──────────────────┼──────────────────────┤
│ 과거 시점 상태        │ 불가 (소실)      │ 가능 (리플레이)      │
├───────────────────────┼──────────────────┼──────────────────────┤
│ 감사 로그             │ 별도 구현 필요   │ 내장                 │
├───────────────────────┼──────────────────┼──────────────────────┤
│ 데이터 수정/삭제      │ 가능             │ 불가 (보상 이벤트)   │
├───────────────────────┼──────────────────┼──────────────────────┤
│ 스키마 변경           │ 상대적으로 쉬움  │ 어려움 (Upcasting)   │
└───────────────────────┴──────────────────┴──────────────────────┘
```

---

## ⚖️ 트레이드오프

```
Event Sourcing 도입이 적합한 도메인:
  ① 완전한 감사 로그가 법적/비즈니스 요구사항 (금융, 의료)
  ② 과거 시점 상태 재현이 필요 (회계, 재고)
  ③ 복잡한 상태 머신 (주문, 예약)
  ④ 새로운 읽기 모델이 자주 필요 (비즈니스 분석 요구사항 변화)

부적합한 도메인:
  ① 단순 CRUD (설정, 프로필)
  ② 감사 로그 불필요
  ③ 과거 상태 재현 불필요
  ④ 팀이 ES 개념에 익숙하지 않은 MVP 단계
```

---

## 📌 핵심 정리

```
Event Sourcing 핵심:

패러다임 전환:
  "현재 상태 저장" → "변화의 이력 저장"
  이벤트 = 진실의 원천, 현재 상태 = 이벤트 리플레이 결과

회계 장부 원칙:
  Append Only — 이벤트는 삭제/수정 불가
  실수 정정 = 보상 이벤트 추가
  현재 상태 = 이벤트 합산

핵심 이점:
  완전한 감사 로그 (내장)
  시간 여행 (과거 상태 재현)
  읽기 모델 재구축 (새 뷰 필요 시)
  디버깅 (이벤트로 이슈 재현)
```

---

## 🤔 생각해볼 문제

**Q1.** GDPR(개인정보보호법)에서는 사용자의 데이터 삭제 요청("잊혀질 권리")을 보장해야 한다. Event Sourcing에서 이벤트를 삭제할 수 없다면 어떻게 이를 준수하는가?

<details>
<summary>해설 보기</summary>

Event Sourcing과 GDPR은 충돌처럼 보이지만 "Crypto Shredding" 기법으로 해결합니다.

사용자 개인정보를 이벤트 페이로드에 직접 저장하지 않고, 이벤트 페이로드는 사용자 ID(식별자)만 저장합니다. 실제 개인정보(이름, 이메일, 주소)는 별도 암호화 키로 암호화하여 저장합니다. 삭제 요청 시 해당 사용자의 암호화 키를 삭제합니다. 이벤트는 남아있지만 개인정보를 복호화할 수 없게 됩니다.

이벤트 내 개인정보가 최소화되도록 설계하는 것도 중요합니다. `MoneyTransferred` 이벤트에 "이체자 이름" 대신 "이체자 ID"만 포함하면, ID와 이름 매핑을 삭제하는 것으로 충분합니다.

</details>

---

**Q2.** Event Sourcing 없이 도입한 시스템에 나중에 Event Sourcing을 도입하려면 어떻게 마이그레이션하는가?

<details>
<summary>해설 보기</summary>

점진적 마이그레이션 전략이 가장 현실적입니다.

1단계로 현재 상태를 "초기 스냅샷 이벤트"로 변환합니다. `AccountInitialized { balance: 500000 }` 이벤트를 각 계좌에 생성합니다. 이 시점 이전의 이력은 복원하지 않습니다.

2단계로 새로운 변경 사항부터 이벤트로 기록합니다. 기존 UPDATE는 계속 사용하면서, 동시에 이벤트도 발행합니다(이중 쓰기).

3단계로 읽기 경로를 이벤트 기반으로 전환합니다. Projection이 이벤트를 소비해 읽기 모델을 생성합니다.

4단계로 쓰기 경로를 Event Sourcing으로 전환합니다. 상태 저장을 제거하고 이벤트 스토어만 사용합니다.

이 마이그레이션에서 초기 스냅샷 이전의 완전한 이력은 복원되지 않습니다. 마이그레이션 시점부터 완전한 이력이 보장됩니다.

</details>

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[이전: Chapter 2 — Command 결과 반환 패턴 ⬅️](../command-side/06-command-result-patterns.md)** | **[다음: 이벤트 스토어 설계 ➡️](./02-event-store-design.md)**

</div>
