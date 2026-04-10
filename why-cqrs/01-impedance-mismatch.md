# 단일 모델의 임피던스 불일치

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- JPA Entity 하나가 쓰기와 읽기를 모두 담당할 때 왜 구조적으로 문제가 생기는가?
- 정규화된 쓰기 스키마와 비정규화된 읽기 요구사항이 충돌하는 지점은 어디인가?
- N+1 문제, 복잡한 조인, 잠금 충돌이 단일 모델에서 왜 피하기 어려운가?
- 임피던스 불일치(Impedance Mismatch)란 무엇이며, 어떻게 쌓여 시스템을 갉아먹는가?
- 이 문제가 CQRS라는 해법으로 연결되는 이유는 무엇인가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

도메인이 복잡해질수록 JPA Entity 하나가 쓰기와 읽기를 모두 담당하는 구조는 조용히 무너진다. 처음에는 `@OneToMany`를 추가하면 됐고, 다음엔 `@Fetch(LAZY)`로 버텼고, 그다음엔 JPQL에 `LEFT JOIN FETCH`를 추가했다. 어느 순간 Entity는 쓰기를 위한 불변식 보호도 제대로 못 하고, 읽기를 위한 빠른 조회도 제대로 못 하는 어중간한 상태가 된다.

임피던스 불일치는 하루아침에 드러나지 않는다. 조회 API 하나를 추가할 때마다 Entity에 연관관계가 하나씩 늘고, 불변식 로직을 보호하려고 할수록 조회 쿼리가 복잡해진다. 이 문서는 그 붕괴 과정을 구체적인 코드로 추적한다.

---

## 😱 흔한 실수 (Before — 단일 Entity로 모든 것을 처리할 때)

```
상황: 은행 계좌 도메인 — 계좌 잔고 관리 + 거래 내역 조회 + 월별 통계 화면

단일 Entity 설계:
  @Entity
  class Account {
      Long id;
      String ownerId;
      BigDecimal balance;
      AccountStatus status;
      List<Transaction> transactions;  // @OneToMany
      String ownerName;                // ← 조회 화면에 필요해서 추가
      String ownerEmail;               // ← 이메일 발송에 필요해서 추가
      BranchCode branch;               // ← 통계 화면에 필요해서 추가
      BigDecimal monthlyAvgBalance;    // ← 월별 통계에 필요해서 추가
  }

문제 1: N+1 — 계좌 목록 화면
  accountRepository.findAll()
  → SELECT * FROM account         (1번)
  → SELECT * FROM transaction WHERE account_id = 1   (N번)
  → SELECT * FROM transaction WHERE account_id = 2   (N번)
  → ...

  @Fetch(LAZY)로 설정해도:
    Jackson 직렬화 시 transactions 필드에 접근 → LazyInitializationException
    → @Fetch(EAGER)로 변경 → 이번엔 항상 N+1

문제 2: 복잡한 조인 — 계좌 상세 화면
  "계좌 정보 + 최근 거래 내역 10건 + 계좌 소유자 KYC 상태 + 현재 적용 금리"
  → JPQL 하나가 4개 테이블을 JOIN
  → 쿼리 성능 저하, 페이지네이션 불가

문제 3: 잠금 충돌 — 쓰기와 읽기가 같은 테이블을 공유
  이체 처리 중 account row에 ROW LOCK 획득
  → 동시에 계좌 조회 API가 같은 row 읽기 시도
  → 읽기가 쓰기 완료를 대기 → 조회 API 지연
  (InnoDB MVCC로 부분 완화되지만 인덱스 경합은 피할 수 없음)

문제 4: Entity 오염 — 쓰기 불변식이 읽기 요구사항에 잠식
  계좌 잔고 이체 불변식: balance >= transferAmount
  → 불변식 검증 로직이 Account.transfer() 안에

  하지만 조회 최적화를 위해 Entity에 계속 필드가 추가됨
  → ownerName, ownerEmail, monthlyAvgBalance, branchCode ...
  → Account는 더 이상 순수한 도메인 모델이 아님
  → 불변식 로직이 잡동사니 필드들 사이에 묻힘
```

---

## ✨ 올바른 접근 (After — 쓰기 모델과 읽기 모델을 분리)

```
분리 후 구조:

쓰기 모델 (Account Aggregate):
  class Account {
      AccountId id;
      OwnerId ownerId;
      Money balance;
      AccountStatus status;
      // 오직 쓰기 불변식에 필요한 필드만 존재
      // ownerName? 이체 불변식에 불필요 → 없음
      // monthlyAvgBalance? 불변식과 무관 → 없음

      void deposit(Money amount) { ... }    // 불변식 보호
      void withdraw(Money amount) { ... }   // 잔고 >= 출금액
      void transfer(AccountId to, Money amount) { ... }
  }

읽기 모델 (AccountSummaryView — 조회 전용):
  class AccountSummaryView {
      String accountId;
      String ownerName;      // 조인 없이 한 번에 조회 가능하도록 비정규화
      String ownerEmail;
      BigDecimal balance;
      String status;
      LocalDate lastTransactionAt;
      // SELECT에 필요한 모든 것이 이미 포함
      // 조인 없음, 단순 SELECT 1회
  }

읽기 모델 (TransactionHistoryView — 거래 내역 전용):
  class TransactionHistoryView {
      String transactionId;
      String accountId;
      String type;        // DEPOSIT / WITHDRAWAL / TRANSFER
      BigDecimal amount;
      BigDecimal balanceAfter;
      LocalDateTime occurredAt;
      String counterpartAccountId;
  }

결과:
  계좌 목록 조회: SELECT * FROM account_summary_view  (조인 없음, N+1 없음)
  거래 내역 조회: SELECT * FROM transaction_history_view WHERE account_id = ?
  이체 처리:     Account Aggregate 로드 → 불변식 검증 → 저장 (읽기 관심사 없음)
```

---

## 🔬 내부 동작 원리

### 1. 임피던스 불일치(Impedance Mismatch)란

```
임피던스 불일치의 원래 의미:
  전기공학에서 두 회로가 서로 다른 임피던스(저항)를 가질 때
  신호가 손실되거나 왜곡되는 현상

소프트웨어에서의 임피던스 불일치:
  두 개의 서로 다른 패러다임이 하나의 경계에서 충돌하는 현상

객체-관계 임피던스 불일치 (Object-Relational Impedance Mismatch):
  객체 세계: 상속, 다형성, 참조, 그래프 구조
  관계형 DB: 테이블, 행, 외래키, 집합 연산
  → ORM이 중간에서 변환하지만 100% 해결 불가

쓰기-읽기 임피던스 불일치 (Write-Read Impedance Mismatch):
  쓰기 세계: 정규화, 불변식 보호, 트랜잭션, 잠금
  읽기 세계: 비정규화, 빠른 조회, 다양한 뷰, 집계
  → 단일 모델이 두 세계를 모두 만족시키려 할 때 충돌 발생
```

### 2. 은행 계좌 도메인에서 불일치가 쌓이는 과정

```
1단계: 초기 설계 (문제 없음)

  Account 테이블:
  ┌──────────┬───────────┬─────────┬────────────┐
  │ id       │ owner_id  │ balance │ status     │
  ├──────────┼───────────┼─────────┼────────────┤
  │ ACC-001  │ USER-42   │ 500000  │ ACTIVE     │
  └──────────┴───────────┴─────────┴────────────┘

  Transaction 테이블:
  ┌────────────┬──────────┬──────┬─────────┬─────────────────────┐
  │ id         │ acct_id  │ type │ amount  │ occurred_at         │
  ├────────────┼──────────┼──────┼─────────┼─────────────────────┤
  │ TXN-001    │ ACC-001  │ DEP  │ 100000  │ 2024-01-15 09:00:00 │
  │ TXN-002    │ ACC-001  │ WDR  │ 50000   │ 2024-01-15 14:30:00 │
  └────────────┴──────────┴──────┴─────────┴─────────────────────┘

2단계: 계좌 목록 화면 요구사항 추가
  "계좌 소유자 이름도 함께 보여주세요"
  → Account에 ownerName 추가? 또는 매번 User 테이블 JOIN?

  JOIN 방식:
  SELECT a.*, u.name as owner_name
  FROM account a
  JOIN user u ON a.owner_id = u.id
  → 계좌 조회마다 user 테이블 JOIN

3단계: 거래 내역 화면 요구사항 추가
  "최근 거래 내역 + 계좌 정보 + 이체 상대방 이름"
  SELECT a.*, t.*, u1.name, u2.name as counterpart_name
  FROM account a
  JOIN transaction t ON t.account_id = a.id
  JOIN user u1 ON a.owner_id = u1.id
  LEFT JOIN account a2 ON t.counterpart_account_id = a2.id
  LEFT JOIN user u2 ON a2.owner_id = u2.id
  WHERE a.id = ?
  ORDER BY t.occurred_at DESC
  LIMIT 20
  → 5개 테이블 JOIN, 인덱스 설계 복잡, 페이지네이션 불안정

4단계: 통계 대시보드 요구사항 추가
  "월별 입출금 합계, 평균 잔고, 거래 횟수"
  SELECT
    DATE_TRUNC('month', t.occurred_at) as month,
    SUM(CASE WHEN t.type = 'DEPOSIT' THEN t.amount ELSE 0 END) as total_deposit,
    AVG(a.balance) as avg_balance,  ← 이 시점의 잔고는 현재값, 과거 월별 평균이 아님
    COUNT(t.id) as tx_count
  FROM account a
  JOIN transaction t ON t.account_id = a.id
  GROUP BY month
  → 과거 특정 시점의 잔고를 알 방법이 없음 (현재 상태만 저장하므로)
  → 월별 평균 잔고: 계산 불가

5단계: Account Entity의 최종 모습
  @Entity
  class Account {
      Long id;
      String ownerId;
      BigDecimal balance;         // 쓰기
      AccountStatus status;       // 쓰기
      String ownerName;           // 읽기용 (비정규화)
      String ownerEmail;          // 읽기용
      BranchCode branchCode;      // 통계용
      @OneToMany
      List<Transaction> txns;     // 조회용 (N+1 원인)
      LocalDate lastTxnAt;        // 캐싱용 (동기화 필요)
      Integer monthlyTxnCount;    // 통계용 (동기화 필요)

      // 쓰기 불변식
      void transfer(AccountId to, Money amount) {
          if (balance.compareTo(amount) < 0)
              throw new InsufficientBalanceException();
          this.balance = balance.subtract(amount);
          // ownerName, ownerEmail, monthlyTxnCount와 같은 클래스에 존재
          // 불변식 로직이 잡동사니에 묻힘
      }
  }
```

### 3. 정규화 vs 비정규화 충돌 구조

```
정규화 (쓰기 최적화):
  목표: 데이터 중복 없음, 갱신 이상 없음
  설계: 각 테이블은 한 가지 사실만 저장
  결과: INSERT/UPDATE/DELETE 빠름, 일관성 보장

  account    (id, owner_id, balance, status)
  user       (id, name, email, kyc_status)
  transaction(id, account_id, type, amount, occurred_at)
  → 중복 없음, 수정 시 한 곳만 변경

비정규화 (읽기 최적화):
  목표: 조인 없는 빠른 조회, 한 번의 SELECT로 모든 데이터
  설계: 자주 함께 읽히는 데이터를 한 테이블에
  결과: SELECT 빠름, 조인 없음

  account_summary (
    account_id, owner_name, owner_email,  ← user 테이블 복사
    balance, status,
    last_tx_at, monthly_tx_count          ← 집계 결과 사전 계산
  )
  → SELECT 1회, 조인 없음, 응답 빠름

충돌:
  정규화 스키마에서 owner_name이 변경됐을 때:
    user 테이블만 UPDATE → 정규화 관점에서 완료
    하지만 account_summary의 owner_name은 여전히 구버전
    → 두 테이블 동기화 필요
    → 동기화 실패 시 불일치

  단일 모델은 두 요구사항을 동시에 만족할 수 없음:
    정규화 → 읽기 시 JOIN 필요 → 읽기 느림
    비정규화 → 쓰기 시 여러 테이블 동기화 → 쓰기 복잡
```

### 4. 잠금 충돌이 발생하는 구조

```
단일 account 테이블에서 쓰기와 읽기가 경합:

타임라인:
  T=0ms  이체 트랜잭션 시작
         → BEGIN TRANSACTION
         → SELECT * FROM account WHERE id = 'ACC-001' FOR UPDATE  ← 쓰기 잠금 획득

  T=5ms  조회 API 요청 도착
         → SELECT a.*, t.* FROM account a JOIN transaction t ...
         → account row에 접근 시도
         → 쓰기 트랜잭션이 잠금 보유 중 → 대기

  T=20ms 이체 트랜잭션 완료
         → COMMIT, 잠금 해제

  T=20ms 조회 API 잠금 획득, 쿼리 실행
         → 총 지연: 20ms (이체 처리 시간 내내 대기)

MySQL InnoDB Gap Lock:
  이체 처리 중 transaction 테이블에 INSERT
  → 인덱스 범위에 Gap Lock 발생
  → 동일 범위 읽기 쿼리가 LOCK WAIT

CQRS 분리 후:
  쓰기 모델: account 테이블 (쓰기 전용)
  읽기 모델: account_summary 테이블 (읽기 전용)
  → 두 테이블이 분리 → 쓰기 잠금이 읽기에 영향 없음
  → Eventually Consistent (이벤트로 읽기 모델 동기화)
```

---

## 💻 실전 코드

### 단일 모델의 문제 — 실제로 망가지는 코드

```java
// ❌ 단일 Entity로 쓰기와 읽기를 모두 처리
@Entity
@Table(name = "account")
public class Account {

    @Id
    private String id;
    private String ownerId;
    private BigDecimal balance;

    @Enumerated(EnumType.STRING)
    private AccountStatus status;

    // 읽기 화면을 위해 추가된 필드들 ↓
    private String ownerName;          // user 테이블 복사
    private String ownerEmail;         // user 테이블 복사
    private LocalDateTime lastTxAt;    // 집계 캐시

    @OneToMany(mappedBy = "account", fetch = FetchType.LAZY)
    private List<Transaction> transactions;

    // 쓰기 불변식 — 읽기 관련 필드들 사이에 묻혀있음
    public void transfer(String toAccountId, BigDecimal amount) {
        if (this.balance.compareTo(amount) < 0) {
            throw new InsufficientBalanceException(this.id, amount, this.balance);
        }
        this.balance = this.balance.subtract(amount);
        this.lastTxAt = LocalDateTime.now(); // 읽기 모델 동기화를 쓰기 로직에서 처리
    }
}

// ❌ N+1이 발생하는 서비스
@Service
public class AccountService {

    public List<AccountSummaryDto> getAccountList() {
        List<Account> accounts = accountRepository.findAll();
        // accounts.size() = 100 → SELECT 1회

        return accounts.stream()
            .map(account -> {
                // transactions 접근 → 각 Account마다 SELECT 1회 → N+1
                int txCount = account.getTransactions().size();
                return new AccountSummaryDto(
                    account.getId(),
                    account.getOwnerName(),
                    account.getBalance(),
                    txCount
                );
            })
            .collect(toList());
        // 총 101번의 SELECT: 1(계좌 목록) + 100(각 계좌별 거래 내역)
    }

    // ❌ 조회 요구사항이 Aggregate 메서드에 침투
    public AccountDetailDto getAccountDetail(String accountId) {
        Account account = accountRepository.findById(accountId)
            .orElseThrow();

        // 이 계산 로직이 Account 도메인에 있어야 하는가?
        BigDecimal monthlyAvg = account.getTransactions().stream()
            .filter(t -> t.getOccurredAt().getMonth() == LocalDate.now().getMonth())
            .map(Transaction::getAmount)
            .reduce(BigDecimal.ZERO, BigDecimal::add)
            .divide(BigDecimal.valueOf(LocalDate.now().getDayOfMonth()), RoundingMode.HALF_UP);

        return new AccountDetailDto(account, monthlyAvg);
    }
}
```

### 분리 후 — 쓰기 모델과 읽기 모델

```java
// ✅ 쓰기 모델 — 불변식에만 집중
public class Account {  // @Entity 없음 — 순수 도메인 모델

    private final AccountId id;
    private final OwnerId ownerId;
    private Money balance;
    private AccountStatus status;
    // ownerName? 이체 불변식에 불필요 → 없음
    // transactions? 잔고 확인에 불필요 → 없음

    public void deposit(Money amount) {
        validateAmount(amount);
        validateStatus();
        this.balance = this.balance.add(amount);
        registerEvent(new MoneyDeposited(this.id, amount, this.balance));
    }

    public void withdraw(Money amount) {
        validateAmount(amount);
        validateStatus();
        if (this.balance.isLessThan(amount)) {
            throw new InsufficientBalanceException(this.id, amount, this.balance);
        }
        this.balance = this.balance.subtract(amount);
        registerEvent(new MoneyWithdrawn(this.id, amount, this.balance));
    }

    // 불변식만 남음 — 읽기 관심사 없음
}

// ✅ 읽기 모델 — 조회 화면에 맞는 비정규화 뷰
@Entity
@Table(name = "account_summary")
public class AccountSummaryView {

    @Id
    private String accountId;
    private String ownerName;      // user 테이블에서 비정규화
    private String ownerEmail;
    private BigDecimal balance;
    private String status;
    private LocalDateTime lastTxAt;
    private Integer monthlyTxCount;
    // 조인 없이 한 번의 SELECT로 목록 화면 완성
}

// ✅ 읽기 모델 — 거래 내역 전용 뷰
@Entity
@Table(name = "transaction_history")
public class TransactionHistoryView {

    @Id
    private String transactionId;
    private String accountId;
    private String type;
    private BigDecimal amount;
    private BigDecimal balanceAfter;
    private String counterpartAccountId;
    private String counterpartOwnerName; // 비정규화
    private LocalDateTime occurredAt;
}

// ✅ 조회 서비스 — JOIN 없음, N+1 없음
@Service
public class AccountQueryService {

    public List<AccountSummaryView> getAccountList() {
        // 단순 SELECT — 조인 없음, N+1 없음
        return accountSummaryRepository.findAll();
    }

    public List<TransactionHistoryView> getTransactionHistory(
            String accountId, int page, int size) {
        // 인덱스 활용 단순 조회
        return transactionHistoryRepository
            .findByAccountIdOrderByOccurredAtDesc(accountId,
                PageRequest.of(page, size));
    }
}
```

---

## 📊 패턴 비교

```
단일 모델 vs 분리 모델 성능 비교 (계좌 100개, 거래 내역 계좌당 1,000건):

┌─────────────────────────────┬──────────────────┬──────────────────┐
│ 시나리오                     │ 단일 모델         │ 분리 모델         │
├─────────────────────────────┼──────────────────┼──────────────────┤
│ 계좌 목록 조회               │ SELECT 101회      │ SELECT 1회        │
│ (N+1)                       │ 거래내역 100번     │ account_summary  │
├─────────────────────────────┼──────────────────┼──────────────────┤
│ 계좌 상세 조회               │ 4테이블 JOIN       │ SELECT 1회        │
│ (JOIN 복잡도)                │ 쿼리 200줄        │ 조인 없음          │
├─────────────────────────────┼──────────────────┼──────────────────┤
│ 이체 처리 중 계좌 조회       │ 잠금 대기 발생     │ 별도 테이블        │
│ (잠금 충돌)                  │ 최대 이체 시간 지연 │ 잠금 영향 없음     │
├─────────────────────────────┼──────────────────┼──────────────────┤
│ Entity 필드 수               │ 15+개             │ 쓰기: 4개          │
│ (모델 복잡도)                │ 쓰기+읽기 혼재    │ 읽기: 목적별 분리  │
├─────────────────────────────┼──────────────────┼──────────────────┤
│ 과거 시점 잔고 조회          │ 불가능            │ 이벤트 리플레이    │
│ (Event Sourcing 연계)        │ 현재 상태만 존재  │ 모든 시점 재현 가능 │
└─────────────────────────────┴──────────────────┴──────────────────┘

비용:
  단일 모델: 구현 단순, 초기 생산성 높음
  분리 모델: 읽기 모델 동기화 필요, 최종 일관성 허용 필요
```

---

## ⚖️ 트레이드오프

```
단일 모델 유지가 유리한 경우:
  ① 도메인이 단순하고 읽기-쓰기 요구사항이 거의 동일한 경우
  ② 팀 규모가 작고 초기 개발 속도가 중요한 경우
  ③ 데이터 일관성(읽기 모델이 쓰기와 항상 동일해야 하는)이 최우선인 경우
  ④ 트래픽이 낮아 N+1, 잠금 충돌이 실제 문제가 되지 않는 경우

쓰기-읽기 모델 분리가 필요한 신호:
  ① 조회 쿼리에 3개 이상 테이블 JOIN이 반복 등장
  ② N+1 문제를 해결하려 할수록 Entity 구조가 이상해짐
  ③ 쓰기 처리 중 조회 API 지연 민원이 발생
  ④ 역할별(관리자/사용자/통계) 다른 데이터 형태가 필요
  ⑤ 도메인 로직보다 JPQL 쿼리 최적화에 더 많은 시간을 쓰는 경우

분리 시 추가 비용:
  읽기 모델 동기화 메커니즘 필요 (이벤트, CDC, 배치)
  읽기 모델 재구축 전략 필요
  최종 일관성 허용 여부 결정 필요
  운영 복잡도 증가 (읽기 모델 DB, 동기화 파이프라인)
```

---

## 📌 핵심 정리

```
임피던스 불일치 핵심 요약:

문제의 본질:
  쓰기 최적화 요구사항 = 정규화 + 불변식 + 트랜잭션 + 잠금
  읽기 최적화 요구사항 = 비정규화 + 빠른 조회 + 다양한 뷰 + 집계
  → 두 요구사항이 단일 테이블/모델에서 구조적으로 충돌

충돌이 만드는 증상:
  N+1 문제        → 읽기를 위한 연관관계가 쓰기를 복잡하게 만듦
  복잡한 JOIN     → 읽기 요구사항이 늘어날수록 쿼리가 비대해짐
  잠금 충돌       → 쓰기 잠금이 읽기 조회를 차단
  Entity 오염     → 읽기 필드가 쓰기 불변식 모델을 침식

해법의 방향:
  쓰기 모델 = 불변식 보호에만 집중한 최소한의 데이터
  읽기 모델 = 조회 화면에 맞게 비정규화된 별도 뷰
  동기화    = 쓰기 모델의 변경을 이벤트로 읽기 모델에 반영
```

---

## 🤔 생각해볼 문제

**Q1.** `@EntityGraph`나 `fetch join`으로 N+1 문제를 해결했을 때 단일 모델 구조의 본질적인 문제가 해결되는가?

<details>
<summary>해설 보기</summary>

N+1은 해결됩니다. 하지만 단일 모델의 본질적인 문제는 해결되지 않습니다.

`fetch join`으로 N+1을 없애더라도 여전히 남는 문제들이 있습니다. 하나의 Entity가 여전히 쓰기 불변식과 읽기 필드를 동시에 담고 있어 모델이 오염됩니다. 역할별 다른 뷰(관리자/사용자/통계)가 필요할 때 fetch join은 대응하지 못합니다. 과거 시점의 데이터 재현이 불가능한 것도 그대로입니다. 쓰기 잠금이 읽기를 차단하는 문제도 같은 테이블을 공유하는 한 피할 수 없습니다.

`@EntityGraph`는 쿼리 실행 전략의 최적화일 뿐, 쓰기와 읽기가 구조적으로 다른 요구사항을 가진다는 근본 문제를 건드리지 않습니다.

</details>

---

**Q2.** 은행 계좌 잔고를 현재 상태로만 저장할 때 "2024년 1월 15일 오후 3시의 잔고"를 알려면 무엇이 필요한가?

<details>
<summary>해설 보기</summary>

현재 상태만 저장하는 방식에서는 과거 시점의 잔고를 정확히 알기가 매우 어렵습니다.

가능한 방법 중 하나로 별도 히스토리 테이블을 만드는 방식이 있습니다. 잔고 변경 시마다 `account_balance_history(account_id, balance, changed_at)`에 기록하고, 특정 시점 이전 가장 최근 레코드를 찾는 것입니다. 하지만 이 방법에는 문제가 있습니다. 히스토리 테이블을 누가, 언제, 어떤 이유로 갱신했는지 추적이 불완전합니다. 원본 테이블과 히스토리 테이블의 동기화 실패 가능성도 있고, 히스토리가 "잔고가 바뀌었다"는 결과는 알지만 "왜 바뀌었는가"는 알기 어렵습니다.

Event Sourcing 방식에서는 "2024-01-15 이전에 발생한 모든 입출금 이벤트를 리플레이하면 그 시점의 정확한 잔고를 계산할 수 있습니다." 이벤트 자체가 진실의 원천이므로 별도 히스토리 테이블이 필요 없습니다.

</details>

---

**Q3.** `balance` 필드 업데이트와 조회 API가 항상 같은 DB 트랜잭션 안에 있어야 한다는 요구사항이 있다면, CQRS 분리는 여전히 가능한가?

<details>
<summary>해설 보기</summary>

"항상 강한 일관성이 필요한가"를 먼저 검토해야 합니다.

이체 직후 잔고 화면에서 즉시 반영된 잔고를 보여줘야 하는 요구사항이라면, 일반적으로 낙관적 UI 업데이트(Command 완료 시 클라이언트가 직접 임시로 새 잔고를 표시)로 해결 가능합니다. 진짜로 동일 트랜잭션 내 일관성이 필요한 케이스는 생각보다 드뭅니다.

만약 진짜로 동일 트랜잭션 내 일관성이 필요하다면, 쓰기 모델과 읽기 모델이 같은 DB를 공유하는 "Simple CQRS" 수준에서는 가능합니다. 읽기 모델 테이블이 쓰기와 같은 DB에 있고, 쓰기 트랜잭션 내에서 읽기 모델도 함께 갱신하는 방식입니다. 다만 이 경우 DB를 분리하는 CQRS의 주요 이점(읽기 스케일 아웃, 잠금 격리)을 포기해야 합니다.

</details>

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[다음: 쓰기 모델과 읽기 모델의 근본적 차이 ➡️](./02-write-read-model-difference.md)**

</div>
