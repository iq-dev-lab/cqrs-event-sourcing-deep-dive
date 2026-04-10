# Command 객체 설계

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Command는 DTO와 무엇이 다른가? 이름을 바꾸는 것 이상의 차이가 있는가?
- Command가 "의도(Intent)를 표현"한다는 것은 구체적으로 무엇을 의미하는가?
- 구문 검증(Syntactic Validation)과 의미 검증(Semantic Validation)은 어디서 수행해야 하는가?
- Command 하나가 여러 Aggregate에 영향을 주어야 할 때 어떻게 설계하는가?
- Command 설계의 안티패턴은 무엇인가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

잘못 설계된 Command는 CQRS의 모든 이점을 무너뜨린다. Command가 DTO와 다를 바 없으면 Command Handler가 매번 "이게 어떤 의도인가?"를 유추해야 한다. Command가 너무 많은 검증을 담으면 도메인 불변식이 Command에 새 나간다. Command가 너무 범용적이면 동일한 Handler가 서로 다른 의도를 처리하게 된다.

Command 설계는 도메인 언어를 코드에 담는 작업이다. `UpdateOrder`가 아니라 `ConfirmOrder`, `CancelOrder`, `RequestRefund`다. 이 명명이 도메인 전문가와 개발자 사이의 언어 일치(Ubiquitous Language)를 결정한다.

---

## 😱 흔한 실수 (Before — DTO처럼 설계된 Command)

```
실수 1: UpdateXxx 형태의 범용 Command

  // ❌ 범용 업데이트 Command — 의도 불명확
  public class UpdateOrderRequest {
      String orderId;
      String status;          // "CONFIRMED", "CANCELLED", "SHIPPED" ...
      String cancelReason;    // status가 CANCELLED일 때만 의미 있음
      String confirmedBy;     // status가 CONFIRMED일 때만 의미 있음
      String trackingNumber;  // status가 SHIPPED일 때만 의미 있음
  }

  // Command Handler가 의도를 추론해야 함
  void handle(UpdateOrderRequest request) {
      if ("CONFIRMED".equals(request.getStatus())) {
          // 확인 처리
      } else if ("CANCELLED".equals(request.getStatus())) {
          // 취소 처리
      } else if ("SHIPPED".equals(request.getStatus())) {
          // 배송 시작 처리
      }
      // 분기가 늘어날수록 Handler가 커짐
  }

실수 2: DTO와 동일한 구조

  // ❌ HTTP 요청 구조와 동일 — 도메인 의도 없음
  public class OrderDto {
      String orderId;
      String customerId;
      List<OrderItemDto> items;
      String address;
      String paymentMethod;
  }

  // CreateOrderCommand로 이름만 바꾼 것
  public class CreateOrderCommand extends OrderDto { }

  // 문제: 도메인 유효성이 없음
  // amount가 음수여도 통과, customerId가 null이어도 통과

실수 3: 과도한 검증을 Command에 담음

  public class WithdrawMoneyCommand {
      String accountId;
      BigDecimal amount;

      // ❌ 불변식 검증을 Command에서 수행
      public WithdrawMoneyCommand(String accountId, BigDecimal amount) {
          if (accountId == null) throw new IllegalArgumentException();
          if (amount.compareTo(BigDecimal.ZERO) <= 0) throw new ...();
          // 여기까지는 OK — 구문 검증

          // ❌ 이것은 Command에서 하면 안 됨 — 의미 검증
          Account account = accountRepository.findById(accountId);
          if (account.getBalance().compareTo(amount) < 0)
              throw new InsufficientBalanceException();
          // Command가 Repository에 의존 → 테스트 불가, 책임 분리 위반
      }
  }
```

---

## ✨ 올바른 접근 (After — 의도를 명확히 표현하는 Command)

```
올바른 Command 설계 원칙:

1. 의도별 별도 Command
  ConfirmOrderCommand  → 주문 확인 의도
  CancelOrderCommand   → 주문 취소 의도 (취소 사유 포함)
  ShipOrderCommand     → 배송 시작 의도 (운송장 번호 포함)
  → "이 Command가 어떤 일을 하는가?" 이름만으로 명확

2. 구문 검증은 Command 생성 시점 (형식 오류)
  amount > 0 인가? (음수는 형식 오류)
  accountId가 null이 아닌가?
  → Repository 없이 Command 객체 내부에서 검증 가능

3. 의미 검증은 Aggregate에서 (불변식 위반)
  잔고가 출금액 이상인가? → Account.withdraw()에서
  계좌가 동결 상태인가? → Account.withdraw()에서
  → Repository 접근 + 도메인 규칙 → Aggregate 책임

4. Command에 포함하는 것
  작업에 필요한 식별자(accountId)
  작업에 필요한 값(amount)
  메타데이터(요청자ID, 요청 시각 — 감사 목적)
  → 그 외는 포함하지 않음
```

---

## 🔬 내부 동작 원리

### 1. Command vs DTO — 근본적 차이

```
DTO (Data Transfer Object):
  목적: 계층 간 데이터 전달
  의미: "이런 데이터가 있습니다"
  검증: 없거나 최소 (형식 검증만)
  이름: 데이터 구조를 반영 (OrderData, ProductInfo)

Command:
  목적: 시스템에 변경을 요청
  의미: "이런 의도로 이런 작업을 수행해주세요"
  검증: 구문 검증 포함
  이름: 동사 + 명사 형태로 의도를 표현 (ConfirmOrder, WithdrawMoney)

차이를 만드는 것 — 이름:
  DTO: OrderStatusUpdateRequest
       → "주문 상태 업데이트 요청" — 의도가 모호

  Command: ConfirmOrderCommand, CancelOrderCommand, ShipOrderCommand
           → "주문 확인", "주문 취소", "배송 시작" — 의도가 명확

  왜 중요한가:
    Command 이름 = 도메인 이벤트의 원인
    ConfirmOrderCommand → OrderConfirmed 이벤트
    CancelOrderCommand  → OrderCancelled 이벤트
    → Command 이름이 이벤트 스트림의 언어를 결정

차이를 만드는 것 — 불변성:
  DTO: 일반적으로 가변 (setter 있음)
  Command: 불변(Immutable)이어야 함

  이유:
    Command는 사용자 의도의 스냅샷
    생성된 후 내용이 바뀌면 처음 의도와 달라짐
    Command Handler에서 Command 수정 → 감사 추적 불가

  Java record로 구현:
    public record ConfirmOrderCommand(
        String orderId,
        String confirmedBy,
        Instant requestedAt
    ) {}
    // record는 자동으로 불변, equals, hashCode, toString 제공
```

### 2. 구문 검증 vs 의미 검증

```
두 검증의 책임 분리:

구문 검증 (Syntactic Validation):
  "이 Command가 처리될 수 있는 형태인가?"
  데이터베이스 없이 검증 가능
  Command 객체 내부 or Bean Validation (@NotNull, @Min 등)

  예시:
    amount > 0          → amount가 숫자 형식이고 양수인가
    orderId != null     → 필수 식별자가 있는가
    email 형식          → 이메일 형식이 맞는가
    amount < 1억        → 1회 거래 금액 상한 (비즈니스 형식 규칙)

의미 검증 (Semantic Validation):
  "이 Command가 비즈니스 규칙을 만족하는가?"
  데이터베이스 or Aggregate 상태가 필요
  Aggregate 도메인 메서드 내부에서 검증

  예시:
    잔고 >= 출금액      → Account 현재 상태 필요
    주문 상태가 PENDING → Order 현재 상태 필요
    재고 >= 주문 수량   → Inventory 현재 상태 필요
    중복 주문이 없는가  → 기존 주문 데이터 조회 필요

검증 흐름:
  HTTP 요청
    → @Valid (Bean Validation) → 구문 검증 실패 시 400 Bad Request
    → Command 생성 (생성자 내 구문 검증)
    → Command Handler
    → Aggregate.doAction() → 의미 검증 실패 시 도메인 예외
    → HTTP 422 Unprocessable Entity (불변식 위반)

구문 검증이 Command에 있어야 하는 이유:
  Command Handler, Aggregate가 항상 유효한 Command를 받는다고 가정 가능
  "null 체크를 Handler에서도 해야 하나?" → 아니요
  Defense in Depth: 여러 레이어에서 검증하되 각 레이어 책임을 명확히
```

### 3. Command 이름 — 도메인 언어와의 일치

```
Ubiquitous Language와 Command 이름:

도메인 전문가가 사용하는 언어:
  "주문을 확인한다" → ConfirmOrderCommand
  "주문을 취소한다" → CancelOrderCommand
  "환불을 요청한다" → RequestRefundCommand
  "계좌를 동결한다" → FreezeAccountCommand
  "이체한다"         → TransferMoneyCommand

잘못된 이름 (기술적 표현):
  UpdateOrderStatusCommand  → 어떤 상태로? 왜?
  ModifyOrderCommand        → 어떤 수정?
  ProcessOrderCommand       → 어떤 처리?

이름이 이벤트 스트림을 결정:
  ConfirmOrderCommand  → OrderConfirmed  이벤트
  CancelOrderCommand   → OrderCancelled  이벤트
  RefundRequestCommand → RefundRequested 이벤트

  도메인 전문가에게 이벤트 이름을 보여줄 때:
    OrderStatusUpdated → "어떤 상태로 업데이트됐나요?"
    OrderConfirmed     → "주문이 확인됐네요" ← 즉시 이해

Command 분리의 실익:
  Command별 별도 Handler → 각 Handler가 단일 책임
  Command별 별도 검증 → 검증 규칙이 명확
  Command별 별도 이벤트 → 이벤트 스트림이 명확한 언어를 가짐
  Command별 별도 권한 검사 → 접근 제어가 명확
    ConfirmOrderCommand → SELLER 역할만
    CancelOrderCommand  → BUYER, SELLER, ADMIN
    FreezeAccountCommand → ADMIN만
```

### 4. 메타데이터 — Command에 포함해야 하는 감사 정보

```
Command 메타데이터의 역할:

핵심 데이터 (Command의 주요 내용):
  WithdrawMoneyCommand:
    accountId: "ACC-001"  ← 어느 계좌에서
    amount: 100000        ← 얼마를

메타데이터 (Command의 맥락):
  requestedBy: "USER-42"  ← 누가 요청했는가 (감사)
  requestedAt: Instant    ← 언제 요청했는가 (감사)
  correlationId: UUID     ← 이 요청의 추적 ID (분산 추적)
  idempotencyKey: UUID    ← 중복 처리 방지 키

왜 Command에 포함하는가:
  이벤트 스토어에 Command 메타데이터를 함께 저장 가능
  "2024-01-15 오전 9시에 USER-42가 ACC-001에서 10만원 출금 요청"
  → 이벤트 스토어에서 완전한 감사 로그

Command 메타데이터를 Handler에서 추가하는 방식:
  SecurityContext에서 현재 사용자 추출
  MDC에서 correlationId 추출
  → 호출자가 매번 넘기지 않아도 됨

  @CommandHandler
  public void handle(WithdrawMoneyCommand command) {
      String requestedBy = SecurityContextHolder.getContext()
          .getAuthentication().getName();
      String correlationId = MDC.get("correlationId");
      // command에 없어도 Handler에서 맥락 정보 보완 가능
  }
```

---

## 💻 실전 코드

```java
// ==============================
// 올바른 Command 설계 패턴
// ==============================

// ✅ 의도별 별도 Command — Java record 활용
public record ConfirmOrderCommand(
    @NotBlank String orderId,
    @NotBlank String confirmedBy
) implements Command {
    // record의 Compact Constructor로 구문 검증
    public ConfirmOrderCommand {
        Objects.requireNonNull(orderId, "orderId는 필수입니다");
        Objects.requireNonNull(confirmedBy, "confirmedBy는 필수입니다");
    }
}

public record CancelOrderCommand(
    @NotBlank String orderId,
    @NotBlank @Size(min = 10, max = 500) String cancelReason,
    @NotBlank String cancelledBy
) implements Command {
    public CancelOrderCommand {
        Objects.requireNonNull(orderId);
        if (cancelReason == null || cancelReason.isBlank())
            throw new IllegalArgumentException("취소 사유는 필수입니다 (최소 10자)");
    }
}

public record WithdrawMoneyCommand(
    @NotBlank String accountId,
    @NotNull @Positive BigDecimal amount, // @Positive: 구문 검증 (양수인가)
    @NotBlank String requestedBy,
    @NotNull Instant requestedAt,
    @NotNull UUID idempotencyKey          // 중복 처리 방지
) implements Command {
    // 구문 검증 (amount 형식만)
    public WithdrawMoneyCommand {
        Objects.requireNonNull(accountId);
        Objects.requireNonNull(amount);
        if (amount.scale() > 2)
            throw new IllegalArgumentException("소수점 2자리까지만 허용");
        if (amount.compareTo(new BigDecimal("100000000")) > 0)
            throw new IllegalArgumentException("1회 한도는 1억원");
        // 잔고 확인은 여기서 하지 않음 — Account.withdraw()에서 의미 검증
    }
}

// ✅ Bean Validation 활용
public record TransferMoneyCommand(
    @NotBlank String fromAccountId,
    @NotBlank String toAccountId,
    @NotNull @Positive @DecimalMax("100000000") BigDecimal amount,
    @NotBlank String requestedBy,
    @NotNull Instant requestedAt,
    @NotNull UUID idempotencyKey
) implements Command {
    public TransferMoneyCommand {
        if (fromAccountId != null && fromAccountId.equals(toAccountId))
            throw new IllegalArgumentException("출금 계좌와 입금 계좌가 동일합니다"); // 구문 검증
    }
}

// ✅ Command Handler — 구문 검증 통과 후 의미 검증은 Aggregate에서
@Component
public class AccountCommandHandler {

    @CommandHandler
    @Transactional
    public void handle(WithdrawMoneyCommand command) {
        // Command가 도착했으면 구문 검증은 이미 통과
        // 의미 검증은 Account.withdraw() 내부에서 수행
        Account account = accountRepository.findById(
            new AccountId(command.accountId()))
            .orElseThrow(() -> new AccountNotFoundException(command.accountId()));

        // 멱등성 처리 — 동일 idempotencyKey면 이미 처리된 요청
        if (commandLog.isDuplicate(command.idempotencyKey())) {
            log.info("중복 요청 무시: {}", command.idempotencyKey());
            return;
        }

        account.withdraw(
            new Money(command.amount()),
            command.requestedBy()    // 감사 목적
        );
        // Account.withdraw() 내부:
        //   if (balance < amount) throw InsufficientBalanceException  ← 의미 검증
        //   if (status == FROZEN) throw AccountFrozenException         ← 의미 검증

        accountRepository.save(account);
        commandLog.record(command.idempotencyKey());
    }
}

// ✅ Bean Validation 적용 — Controller에서 @Valid
@RestController
@RequestMapping("/accounts")
@Validated
public class AccountCommandController {

    @PostMapping("/{id}/withdraw")
    @ResponseStatus(HttpStatus.ACCEPTED)
    public void withdraw(
            @PathVariable String id,
            @RequestBody @Valid WithdrawMoneyRequest request,
            @AuthenticationPrincipal UserDetails user) {

        commandBus.send(new WithdrawMoneyCommand(
            id,
            request.amount(),
            user.getUsername(),
            Instant.now(),
            request.idempotencyKey()  // 클라이언트가 생성한 UUID
        ));
    }

    // 검증 실패 처리
    @ExceptionHandler(MethodArgumentNotValidException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public ProblemDetail handleValidation(MethodArgumentNotValidException e) {
        return ProblemDetail.forStatus(400); // 구문 검증 실패: 400
    }

    @ExceptionHandler(InsufficientBalanceException.class)
    @ResponseStatus(HttpStatus.UNPROCESSABLE_ENTITY)
    public ProblemDetail handleDomainException(InsufficientBalanceException e) {
        return ProblemDetail.forStatus(422); // 의미 검증 실패: 422
    }
}
```

---

## 📊 패턴 비교

```
Command 설계 스타일 비교:

❌ 범용 Command (UpdateXxx):
  UpdateOrderCommand { orderId, status, reason, trackingNo, ... }
  장점: Command 클래스 수가 적음
  단점: Handler가 분기로 가득 참, 의도 불명확, 검증이 복잡

✅ 의도별 Command:
  ConfirmOrderCommand { orderId, confirmedBy }
  CancelOrderCommand  { orderId, reason, cancelledBy }
  ShipOrderCommand    { orderId, trackingNumber, carrier }
  장점: Handler 단일 책임, 의도 명확, 검증 단순, 이벤트 이름 자연스러움
  단점: Command 클래스 수 증가

검증 위치별 비교:

  Command 생성자:
    장점: 항상 유효한 Command가 Handler에 도달
    단점: Repository 의존 불가 (의미 검증 불가)

  Bean Validation (@Valid):
    장점: 선언적, 재사용 가능
    단점: 복잡한 검증 로직 표현 어려움

  Command Handler:
    장점: Repository 접근 가능
    단점: Aggregate 책임과 중복될 수 있음

  Aggregate 도메인 메서드:
    장점: 불변식이 도메인 안에 응집
    단점: Handler에서 예외 처리 필요

권장:
  구문 검증 → Command 생성자 + Bean Validation
  의미 검증 → Aggregate 도메인 메서드
```

---

## ⚖️ 트레이드오프

```
Command별 분리의 비용:
  Command 클래스 수 증가 (도메인 행동 수만큼)
  각 Command별 Handler, 테스트 필요

Command별 분리의 편익:
  각 Command가 단일 책임 → 테스트 용이
  이벤트 이름이 Command 이름과 자연스럽게 대응
  접근 제어가 Command 단위로 명확
  이벤트 스트림 언어가 도메인 언어와 일치

불변 Command의 비용:
  record 또는 final 필드로 구현 시 수정 불가
  테스트에서 일부 필드만 바꿔 테스트하기 어려울 수 있음

불변 Command의 편익:
  Command가 Handler에 전달된 후 변경 불가 → 안전
  멀티스레드 환경에서 공유 안전
  감사 로그에서 원래 의도 보존
```

---

## 📌 핵심 정리

```
Command 설계 핵심:

Command vs DTO:
  DTO: "이런 데이터가 있습니다" — 데이터 컨테이너
  Command: "이런 의도로 이걸 해주세요" — 의도 표현
  이름: 동사 + 명사 (ConfirmOrder, WithdrawMoney)
  불변: 생성 후 수정 불가 (Java record 활용)

검증 책임 분리:
  구문 검증 (형식): Command 생성자 + Bean Validation
    → 양수인가, null이 아닌가, 길이 제한
  의미 검증 (불변식): Aggregate 도메인 메서드
    → 잔고 충분한가, 상태 전이 가능한가

Command 이름의 중요성:
  도메인 전문가 언어와 일치 (Ubiquitous Language)
  UpdateXxx → ConfirmXxx, CancelXxx, ShipXxx
  Command 이름 → 이벤트 이름 자연스럽게 대응

포함해야 할 것:
  작업에 필요한 식별자와 값
  메타데이터 (요청자, 요청 시각, 멱등성 키)

포함하면 안 되는 것:
  다른 Aggregate의 현재 상태
  읽기 모델 데이터
  Repository 의존성
```

---

## 🤔 생각해볼 문제

**Q1.** Command에 멱등성 키(Idempotency Key)가 필요한 이유는 무엇인가? 멱등성 키 없이 중복 Command를 방지할 수 있는가?

<details>
<summary>해설 보기</summary>

멱등성 키가 필요한 근본 이유는 네트워크의 불안정성입니다. 클라이언트가 `WithdrawMoney` Command를 전송했을 때, 서버가 처리를 완료했지만 응답 도중 네트워크가 끊어지면 클라이언트는 처리 여부를 알 수 없습니다. 재시도하면 이중 출금이 발생할 수 있습니다.

멱등성 키가 있으면, 클라이언트가 같은 UUID로 재시도할 때 서버는 이미 처리된 요청임을 인식하고 실제 처리 없이 성공 응답을 반환합니다.

멱등성 키 없이 방지하는 방법도 있습니다. 낙관적 잠금 + Expected Version을 활용하면 됩니다. 클라이언트가 현재 Account Version을 포함해 Command를 전송하고, 서버는 해당 Version에서만 처리합니다. 같은 Version에서 두 번 처리하려 하면 두 번째는 Version 불일치로 실패합니다. 하지만 이 방법은 클라이언트가 Account Version을 알아야 한다는 전제가 필요합니다.

실무에서는 멱등성 키 방식이 더 일반적입니다.

</details>

---

**Q2.** 은행 이체(TransferMoney)는 출금 계좌와 입금 계좌 두 Aggregate에 영향을 준다. Command 하나로 두 Aggregate를 처리해야 하는가, 아니면 두 Command로 분리해야 하는가?

<details>
<summary>해설 보기</summary>

두 Aggregate를 단일 트랜잭션으로 처리하려는 것은 DDD의 Aggregate 경계 원칙과 충돌합니다. Aggregate는 일관성 경계이며, 단일 트랜잭션이 여러 Aggregate를 수정하면 Aggregate 경계가 의미를 잃습니다.

권장 방식은 두 가지입니다. 첫째, Saga 패턴을 활용합니다. `TransferMoneyCommand` 하나를 받아 두 단계로 분리합니다. 1단계로 출금 계좌에서 출금 처리(`WithdrawFromAccount`)를 진행하고, 출금 완료 이벤트(`MoneyWithdrawn`) 발행 후 2단계로 입금 계좌에 입금 처리(`DepositToAccount`)를 진행합니다. 입금 실패 시 보상 트랜잭션(출금 취소)을 실행합니다.

둘째, Aggregate 설계를 재검토합니다. "이체"가 독립적인 Aggregate(`MoneyTransfer`)가 될 수 있습니다. 이체 Aggregate가 출금과 입금을 각각 이벤트로 발행하고, 각 계좌 Aggregate가 해당 이벤트를 구독합니다.

단일 트랜잭션이 절대적으로 필요하다면(강한 일관성), 두 계좌를 같은 Aggregate로 묶어야 합니다. 하지만 이는 Aggregate가 너무 커지는 문제를 가져옵니다.

</details>

---

**Q3.** `CreateOrderCommand`와 `PlaceOrderCommand` 중 어느 이름이 더 좋은가? 도메인 언어 관점에서 차이가 있는가?

<details>
<summary>해설 보기</summary>

도메인 언어 관점에서 `PlaceOrderCommand`가 더 좋습니다.

`CreateOrderCommand`는 기술적 표현입니다. 시스템이 주문 레코드를 "생성"한다는 관점입니다. 이 언어는 데이터베이스 INSERT에 가깝습니다.

`PlaceOrderCommand`는 도메인 언어입니다. 고객이 주문을 "넣는다(place)"는 비즈니스 행위를 표현합니다. 도메인 전문가(영업팀, 운영팀)는 "주문 생성"이 아니라 "주문 접수" 또는 "주문 넣기"라고 말합니다.

이 차이가 이벤트 이름까지 이어집니다. `CreateOrderCommand` → `OrderCreated` vs `PlaceOrderCommand` → `OrderPlaced`. 도메인 전문가에게 "OrderPlaced 이벤트가 발생했습니다"는 자연스럽지만, "OrderCreated 이벤트가 발생했습니다"는 어색합니다.

Ubiquitous Language의 핵심은 도메인 전문가가 사용하는 언어를 코드에 그대로 반영하는 것입니다. 도메인 전문가와 함께 Event Storming을 진행하면 이런 이름이 자연스럽게 드러납니다.

</details>

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[이전: Command와 Query의 명확한 분리 ⬅️](./01-command-query-separation.md)** | **[다음: Command Handler와 Aggregate ➡️](./03-command-handler-aggregate.md)**

</div>
