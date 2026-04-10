# Axon Framework 아키텍처

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `@CommandHandler` / `@EventHandler` / `@QueryHandler` 어노테이션이 내부적으로 어떻게 동작하는가?
- Axon Server는 어떤 역할을 하며, 없으면 무엇을 직접 구현해야 하는가?
- Tracking Event Processor는 Subscribing Event Processor와 무엇이 다른가?
- Spring Boot AutoConfiguration이 Axon을 어떻게 자동으로 등록하는가?
- Axon의 메시지 버스 구조에서 Command, Event, Query가 각각 어떻게 라우팅되는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

Axon Framework는 CQRS + Event Sourcing 구현을 위한 가장 성숙한 Java 프레임워크다. 하지만 어노테이션 뒤에서 어떤 일이 일어나는지 모르면 장애 발생 시 원인 파악이 어렵다. `@CommandHandler`가 어떻게 Command를 받고, Aggregate가 어떻게 로드되며, `@EventSourcingHandler`가 리플레이와 새 이벤트 발행 시 모두 호출되는 이유를 이해해야 한다.

---

## 😱 흔한 실수 (Before — Axon 내부를 모를 때)

```
실수 1: @EventSourcingHandler에서 부수효과 발생

  @EventSourcingHandler
  public void on(MoneyWithdrawn event) {
      this.balance = balance.subtract(event.amount());
      emailService.sendWithdrawalNotification(event); // ❌ 이메일 발송
  }
  // 리플레이 시마다 이메일 발송!
  // 재구성 = 동일한 @EventSourcingHandler 호출

실수 2: @CommandHandler에서 이벤트를 apply() 없이 직접 저장

  @CommandHandler
  public void handle(WithdrawMoneyCommand cmd) {
      this.balance = balance.subtract(cmd.amount()); // ❌ 직접 상태 변경
      // apply() 없음 → 이벤트 스토어에 저장 안 됨
      // → Event Sourcing 동작 불가
  }

실수 3: Tracking/Subscribing 차이 모르고 사용

  // Subscribing: 동기, 이벤트 발행 스레드에서 처리
  // Tracking: 비동기, 별도 스레드, 리플레이 가능
  // 차이 모르고 @ProcessingGroup 설정 누락
  // → 모든 프로젝션이 동기 처리 → Command 처리 지연
```

---

## ✨ 올바른 접근 (After — Axon 메시지 흐름 이해)

```
Axon 3가지 메시지 버스:

  CommandBus:  Command → 단 하나의 CommandHandler로 라우팅
               (1:1 관계)

  EventBus:    Event → 0개 이상의 EventHandler로 브로드캐스트
               (1:N 관계)

  QueryBus:    Query → 쿼리 타입에 맞는 QueryHandler로 라우팅
               (1:1 또는 분산)

Axon Server:
  위 세 버스의 중앙 서버 구현
  + 이벤트 스토어 역할
  + 분산 환경에서 Command/Query 라우팅

Spring Boot AutoConfiguration:
  axon-spring-boot-starter 의존성 추가만으로
  CommandBus, EventBus, EventStore, QueryBus 자동 Bean 등록
  @CommandHandler, @EventHandler, @QueryHandler 스캔 + 등록
```

---

## 🔬 내부 동작 원리

### 1. @CommandHandler 내부 흐름

```
commandGateway.send(new WithdrawMoneyCommand("ACC-001", 100000))

Step 1: CommandGateway → CommandBus
  CommandGateway는 CommandBus의 편의 래퍼
  send() = dispatch() + 응답 대기

Step 2: CommandBus 라우팅
  Command 타입: WithdrawMoneyCommand
  대상 찾기: "account-ACC-001" Aggregate 인스턴스

  Axon Server 있을 때:
    Command를 Axon Server로 전송
    Axon Server가 해당 Aggregate를 처리할 노드로 라우팅

  로컬 CommandBus:
    @CommandHandler 어노테이션 스캔으로 Handler 맵 구성
    WithdrawMoneyCommand → AccountAggregate::handle

Step 3: Aggregate 로드 (Repository)
  EventSourcedRepository.load("ACC-001")
    → EventStore에서 "account-ACC-001" 스트림 이벤트 로드
    → AccountAggregate 인스턴스 생성
    → 이벤트 순서대로 @EventSourcingHandler 호출 (재구성)
    → 현재 상태 완성

Step 4: @CommandHandler 실행
  accountAggregate.handle(WithdrawMoneyCommand)
    → 불변식 검증 (balance >= amount)
    → AggregateLifecycle.apply(new MoneyWithdrawn(...))
       → 이벤트 스토어에 저장
       → 즉시 @EventSourcingHandler 호출 (상태 업데이트)
       → Event Bus에 발행 (비동기 Projection 처리)

Step 5: 응답 반환
  CommandHandler return 값 → CommandGateway.send() 응답
  (void or 생성된 ID)
```

### 2. @EventSourcingHandler vs @EventHandler

```
두 어노테이션의 근본적 차이:

@EventSourcingHandler (Aggregate 안):
  목적: Aggregate 상태 변경 (순수 상태 전환)
  호출 시점:
    ① AggregateLifecycle.apply() 호출 시 (Command 처리 중)
    ② 이벤트 스토어에서 Aggregate 재구성 시 (리플레이)
  → 같은 메서드가 두 컨텍스트에서 호출됨
  → 부수효과 절대 금지 (이메일, DB 저장, 외부 API 호출)
  → 순수 상태 업데이트만

  @EventSourcingHandler
  public void on(MoneyWithdrawn event) {
      this.balance = balance.subtract(event.amount()); // 상태만
      // 외부 통신 금지!
  }

@EventHandler (Projection, Saga):
  목적: 이벤트에 반응해 읽기 모델 업데이트 or Saga 진행
  호출 시점: 이벤트가 EventBus에 발행될 때 (비동기)
  → 부수효과 허용 (DB 업데이트, 이메일, 외부 API)
  → Aggregate 외부의 별도 컴포넌트

  @EventHandler
  public void on(MoneyWithdrawn event) {
      accountSummaryRepo.updateBalance(event.accountId(), event.balanceAfter());
      // DB 업데이트, 이메일 발송 모두 OK
  }
```

### 3. Tracking vs Subscribing Event Processor

```
Subscribing Event Processor:
  이벤트 발행 스레드에서 동기적으로 처리
  Command 처리 → Event 발행 → Projection 즉시 처리 → Command 완료

  장점: 강한 일관성 (Command 완료 시 읽기 모델 최신화)
  단점: Projection 처리 지연이 Command 처리에 영향
        리플레이 불가 (발행 시점에만 처리)

  사용: 테스트, 단순한 In-Process CQRS

Tracking Event Processor:
  별도 스레드에서 비동기 처리
  이벤트 스토어를 주기적으로 폴링 (Tracking Token 기반)
  재시작 시 Tracking Token 위치부터 재처리

  장점:
    Projection 처리가 Command 처리에 영향 없음
    리플레이 가능 (Token 리셋으로 처음부터 재처리)
    병렬 처리 (Segment 분할)
    장애 복구 (토큰 위치부터 재처리)

  단점: Eventual Consistency (지연 발생)

  Axon Server:
    Tracking Event Processor가 Axon Server에 연결
    Axon Server가 각 Processor의 진행 상황 관리
    "이 Processor는 global_seq 10500까지 처리했다"

Segment 병렬 처리:
  Event Processor를 N개 Segment로 분할
  각 Segment를 독립 스레드에서 처리
  stream_id 해시 기반으로 Segment 배정
  → 같은 Aggregate 이벤트 = 같은 Segment = 순서 보장

  @ProcessingGroup("order-summary")
  @EventHandler
  class OrderSummaryProjection {
      // 기본: Segment 4개 (설정 가능)
  }
```

### 4. Spring Boot AutoConfiguration

```
axon-spring-boot-starter 추가 시 자동 구성:

  AxonAutoConfiguration:
    → CommandBus (SimpleCommandBus or AxonServerCommandBus) Bean 등록
    → EventBus (SimpleEventBus or AxonServerEventBus) Bean 등록
    → EventStore (EmbeddedEventStore or AxonServerEventStore) Bean 등록
    → QueryBus (SimpleQueryBus or AxonServerQueryBus) Bean 등록
    → CommandGateway Bean 등록 (CommandBus의 편의 래퍼)
    → QueryGateway Bean 등록

  ApplicationContext 스캔:
    @Aggregate 클래스 → AggregateFactory 등록
    @CommandHandler 메서드 → CommandBus에 Handler 등록
    @EventHandler 메서드 → EventBus에 Listener 등록
    @QueryHandler 메서드 → QueryBus에 Handler 등록

  Axon Server 연결 (axon.axonserver.servers 설정 있으면):
    로컬 버스 대신 Axon Server 연결
    분산 Command/Event/Query 라우팅
    중앙 이벤트 스토어 사용

  Tracking Event Processor 자동 시작:
    @ProcessingGroup 어노테이션 기반으로 그룹 구성
    각 그룹의 Tracking Event Processor 스레드 시작
```

---

## 💻 실전 코드

```java
// ✅ Axon 기본 설정 (application.yml)
/*
axon:
  axonserver:
    servers: localhost:8124   # Axon Server 주소
  serializer:
    general: jackson          # JSON 직렬화
    events: jackson
  eventhandling:
    processors:
      order-summary:          # @ProcessingGroup("order-summary")
        mode: tracking        # Tracking (비동기, 재구축 가능)
        threadCount: 2        # 2개 스레드
        batchSize: 100        # 100건씩 배치 처리
      payment:
        mode: subscribing     # Subscribing (동기, 강한 일관성)
*/

// ✅ Axon 메시지 흐름 확인 — 직접 디버깅
@Configuration
public class AxonMessageInterceptorConfig {

    @Bean
    public CommandHandlerInterceptor loggingCommandInterceptor() {
        return (unitOfWork, interceptorChain) -> {
            CommandMessage<?> cmd = unitOfWork.getMessage();
            log.info("[Command] type={} id={}",
                cmd.getPayloadType().getSimpleName(),
                cmd.getIdentifier());
            long start = System.currentTimeMillis();
            try {
                Object result = interceptorChain.proceed();
                log.info("[Command] SUCCESS type={} elapsed={}ms",
                    cmd.getPayloadType().getSimpleName(),
                    System.currentTimeMillis() - start);
                return result;
            } catch (Exception e) {
                log.error("[Command] FAILED type={} error={}",
                    cmd.getPayloadType().getSimpleName(), e.getMessage());
                throw e;
            }
        };
    }

    @Autowired
    public void configureInterceptors(CommandBus commandBus,
                                       CommandHandlerInterceptor loggingInterceptor) {
        if (commandBus instanceof SimpleCommandBus bus) {
            bus.registerHandlerInterceptor(loggingInterceptor);
        }
    }
}

// ✅ Axon Server 없이 In-Process 구성 (개발/테스트)
@Configuration
@ConditionalOnProperty(name = "axon.axonserver.enabled", havingValue = "false")
public class InProcessAxonConfig {

    @Bean
    public EventStorageEngine eventStorageEngine(Serializer serializer,
                                                   DataSource dataSource) {
        // JPA 기반 이벤트 스토어 (Axon Server 대신 로컬 DB)
        return JpaEventStorageEngine.builder()
            .entityManagerProvider(new ContainerManagedEntityManagerProvider())
            .transactionManager(new SpringTransactionManager(
                new JpaTransactionManager()))
            .build();
    }

    @Bean
    public EventStore eventStore(EventStorageEngine storageEngine) {
        return EmbeddedEventStore.builder()
            .storageEngine(storageEngine)
            .build();
    }
}
```

---

## 📊 패턴 비교

```
Axon Server 있음 vs 없음:

┌──────────────────────┬──────────────────────┬──────────────────────┐
│ 항목                 │ Axon Server 있음     │ 없음 (In-Process)    │
├──────────────────────┼──────────────────────┼──────────────────────┤
│ 이벤트 스토어        │ Axon Server          │ JPA / JDBC           │
├──────────────────────┼──────────────────────┼──────────────────────┤
│ Command 라우팅       │ Axon Server          │ 로컬 CommandBus      │
├──────────────────────┼──────────────────────┼──────────────────────┤
│ 분산 환경            │ 지원                 │ 불가                  │
├──────────────────────┼──────────────────────┼──────────────────────┤
│ 모니터링 대시보드    │ Axon Dashboard 제공  │ 없음                  │
├──────────────────────┼──────────────────────┼──────────────────────┤
│ 프로젝션 재구축      │ UI로 클릭            │ API 직접 호출        │
├──────────────────────┼──────────────────────┼──────────────────────┤
│ 추가 운영 비용       │ 있음 (서버 운영)     │ 없음                  │
└──────────────────────┴──────────────────────┴──────────────────────┘

Tracking vs Subscribing:
  Tracking: 비동기, 리플레이 가능, Eventual Consistency
  Subscribing: 동기, 리플레이 불가, 강한 일관성
  → 프로덕션 Projection: Tracking 권장
  → 테스트, 감사 로그: Subscribing 허용
```

---

## ⚖️ 트레이드오프

```
Axon Framework 도입 비용:
  학습 곡선 (어노테이션 기반 추상화 이해 필요)
  Axon Server 운영 복잡도
  버전 업그레이드 관리

편익:
  CQRS + ES 보일러플레이트 최소화
  프로젝션 재구축 UI 제공 (Axon Server)
  분산 환경 Command/Query 라우팅 내장
  Saga 구현 패턴 내장
```

---

## 📌 핵심 정리

```
Axon 아키텍처 핵심:

3개 메시지 버스:
  CommandBus: 1:1 라우팅 (하나의 Handler)
  EventBus: 1:N 브로드캐스트 (여러 Handler)
  QueryBus: 1:1 또는 분산 조회

@EventSourcingHandler:
  Aggregate 재구성 + Command 처리 시 모두 호출
  순수 상태 변경만 (부수효과 금지)

Tracking Event Processor:
  비동기, 별도 스레드, 리플레이 가능
  Segment 분할로 병렬 처리
  Token 기반 재시작 위치 추적

AutoConfiguration:
  starter 의존성 하나로 모든 Bean 자동 등록
  @CommandHandler, @EventHandler, @QueryHandler 자동 스캔
```

---

## 🤔 생각해볼 문제

**Q1.** Axon에서 `AggregateLifecycle.apply(event)`를 호출하면 이벤트가 즉시 이벤트 스토어에 저장되는가, 아니면 나중에 저장되는가?

<details>
<summary>해설 보기</summary>

`apply()`를 호출하는 즉시 이벤트 스토어에 저장되지 않습니다. Axon의 Unit of Work 패턴 덕분에 트랜잭션이 커밋될 때 저장됩니다.

`apply()` 호출 시 발생하는 일은 다음과 같습니다. 이벤트가 현재 Unit of Work의 이벤트 큐에 추가됩니다. 즉시 `@EventSourcingHandler`가 호출되어 Aggregate 상태가 업데이트됩니다. 이벤트 스토어에는 아직 저장되지 않습니다.

Unit of Work 커밋 시(트랜잭션 커밋 시) 이벤트 큐의 이벤트들이 이벤트 스토어에 일괄 저장됩니다. 그 후 `@EventHandler`(Projection, Saga)에게 이벤트가 발행됩니다.

이 순서가 중요한 이유는 하나의 Command 처리에서 여러 번 `apply()`를 호출해도 모두 같은 트랜잭션에 포함되기 때문입니다. 중간에 예외가 발생하면 트랜잭션이 롤백되어 이벤트 스토어에 저장되지 않습니다.

</details>

---

**Q2.** 프로젝션 재구축(Replay) 시 `@EventHandler` 메서드가 호출되는데, 재구축 중 외부 이메일 발송 같은 부수효과가 다시 실행되는 문제를 어떻게 방지하는가?

<details>
<summary>해설 보기</summary>

재구축 중 부수효과 방지에는 두 가지 방법이 있습니다.

첫째, `ReplayStatus` 파라미터를 활용합니다. `@EventHandler` 메서드에 `@AllowReplay` 또는 `ReplayStatus` 파라미터를 추가해 재구축 여부를 확인합니다.

```java
@EventHandler
public void on(OrderConfirmed event, ReplayStatus replayStatus) {
    orderSummaryRepo.update(event); // 항상 실행 (읽기 모델 업데이트)
    
    if (!replayStatus.isReplay()) {
        emailService.sendConfirmation(event); // 재구축 시 건너뜀
    }
}
```

둘째, 부수효과가 있는 Handler를 별도 `@ProcessingGroup`으로 분리합니다. 재구축 시 해당 그룹을 포함하지 않습니다. 이메일 발송 Handler는 Subscribing Event Processor(동기)로 설정해 재구축 대상에서 제외합니다.

Axon Framework는 기본적으로 `@ResetHandler`를 사용해 재구축 전 초기화 로직을 정의합니다. 재구축 대상이 되는 Processor만 리셋하고, 부수효과 Processor는 리셋에서 제외합니다.

</details>

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[이전: Chapter 5 — 마이크로서비스와의 통합 ⬅️](../integrated-architecture/06-microservices-integration.md)** | **[다음: Aggregate 구현 ➡️](./02-aggregate-implementation.md)**

</div>
