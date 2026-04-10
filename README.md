<div align="center">

# ⚡ CQRS + Event Sourcing Deep Dive

**"이벤트가 두 모델의 동기화 수단이 되는 순간, CQRS는 패턴이 아닌 아키텍처가 된다"**

<br/>

> *"읽기와 쓰기를 나누는 것과, 왜 이벤트가 진실의 원천이 되어야 하는지 — 그리고 그것이 시스템 전체 설계를 어떻게 바꾸는지 아는 것은 다르다"*

단일 모델의 임피던스 불일치가 왜 발생하는지, 이벤트 스토어가 일반 DB와 어떻게 다른지, 프로젝션이 이벤트 스트림에서 읽기 모델을 만드는 원리, Transactional Outbox로 이벤트 저장과 발행의 원자성을 보장하는 방법까지  
**왜 이렇게 설계됐는가** 라는 질문으로 CQRS + Event Sourcing 내부를 끝까지 파헤칩니다

<br/>

[![GitHub](https://img.shields.io/badge/GitHub-dev--book--lab-181717?style=flat-square&logo=github)](https://github.com/dev-book-lab)
[![Axon](https://img.shields.io/badge/Axon_Framework-4.x-F7931E?style=flat-square&logo=axway&logoColor=white)](https://docs.axoniq.io/)
[![Spring](https://img.shields.io/badge/Spring_Boot-3.x-6DB33F?style=flat-square&logo=spring&logoColor=white)](https://spring.io/projects/spring-boot)
[![Kafka](https://img.shields.io/badge/Apache_Kafka-3.x-231F20?style=flat-square&logo=apachekafka&logoColor=white)](https://kafka.apache.org/)
[![Docs](https://img.shields.io/badge/Docs-40개-blue?style=flat-square&logo=readthedocs&logoColor=white)](./README.md)
[![License](https://img.shields.io/badge/License-MIT-yellow?style=flat-square&logo=opensourceinitiative&logoColor=white)](./LICENSE)

</div>

---

## 🎯 이 레포에 대하여

CQRS + Event Sourcing에 관한 자료는 넘쳐납니다. 하지만 대부분은 **"어떻게 나누나"** 에서 멈춥니다.

| 일반 자료 | 이 레포 |
|----------|---------|
| "Command 패키지와 Query 패키지를 분리하세요" | 단일 모델의 임피던스 불일치가 왜 발생하는지, 쓰기 최적화(정규화 + 불변식)와 읽기 최적화(비정규화 + 빠른 조회)가 왜 하나의 모델로 공존하기 어려운가 |
| "이벤트 소싱은 이벤트를 저장합니다" | Append-Only 이벤트 스토어 구조(Stream ID / Version / EventType / Payload), apply() 패턴으로 이벤트를 리플레이하여 Aggregate 현재 상태를 계산하는 전 과정 |
| "읽기 모델이 따로 있습니다" | 프로젝션이 이벤트 스트림을 소비하여 RDB / Redis / Elasticsearch 읽기 모델을 만드는 원리, Eventual Consistency와 지연 처리 UX 전략 |
| "Outbox 패턴을 쓰세요" | 이벤트 스토어 저장과 Kafka 발행의 원자성 문제, Transactional Outbox와 CDC(Change Data Capture)의 차이와 선택 기준 |
| "감사 로그는 Event Sourcing으로 해결됩니다" | 시간 여행(특정 시점 Aggregate 상태 재현), 프로덕션 이슈를 이벤트로 재현하는 디버깅, 새로운 읽기 모델이 필요할 때 과거 이벤트부터 재구축하는 실전 전략 |
| "Axon Framework을 쓰면 됩니다" | `@CommandHandler` / `@EventSourcingHandler` / `@QueryHandler`의 내부 흐름, Axon 없이 순수 Spring + PostgreSQL 이벤트 스토어를 직접 구현하는 방법 |
| 이론 나열 | 실행 가능한 은행 계좌 도메인 코드 + Docker Compose(Axon Server + Kafka + PostgreSQL + Redis) 실험 환경 |

---

## 🚀 빠른 시작

각 챕터의 첫 문서부터 바로 학습을 시작하세요!

[![WhyCQRS](https://img.shields.io/badge/🔹_Ch1-단일_모델의_임피던스_불일치-6366F1?style=for-the-badge)](./why-cqrs/01-impedance-mismatch.md)
[![CommandSide](https://img.shields.io/badge/🔹_Ch2-Command와_Query의_분리-6366F1?style=for-the-badge)](./command-side/01-command-query-separation.md)
[![EventSourcing](https://img.shields.io/badge/🔹_Ch3-이벤트가_진실의_원천-6366F1?style=for-the-badge)](./event-sourcing/01-event-sourcing-core.md)
[![ReadModel](https://img.shields.io/badge/🔹_Ch4-프로젝션_완전_분해-6366F1?style=for-the-badge)](./read-model-projection/01-projection-deep-dive.md)
[![Architecture](https://img.shields.io/badge/🔹_Ch5-완전한_통합_흐름-6366F1?style=for-the-badge)](./integrated-architecture/01-complete-flow.md)
[![Axon](https://img.shields.io/badge/🔹_Ch6-Axon_아키텍처-6366F1?style=for-the-badge)](./spring-axon-implementation/01-axon-architecture.md)
[![Tradeoffs](https://img.shields.io/badge/🔹_Ch7-은행_계좌_도메인_구현-6366F1?style=for-the-badge)](./real-world-tradeoffs/01-bank-account-domain.md)

---

## 📚 전체 학습 지도

> 💡 각 섹션을 클릭하면 상세 문서 목록이 펼쳐집니다

<br/>

### 🔹 Chapter 1: 왜 CQRS인가 — 단일 모델의 한계

> **핵심 질문:** JPA Entity 하나가 저장과 조회를 모두 담당할 때 왜 문제가 생기는가? 쓰기 최적화와 읽기 최적화는 왜 근본적으로 충돌하며, CQRS는 어떤 수준으로 적용해야 하는가?

<details>
<summary><b>단일 모델의 임피던스 불일치부터 CQRS 적용 판단 기준까지 (5개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. 단일 모델의 임피던스 불일치](./why-cqrs/01-impedance-mismatch.md) | 저장에 최적화된 정규화 스키마가 조회에 최적인 비정규화와 충돌하는 원리, JPA Entity 하나가 쓰기/읽기 모두를 담당할 때 발생하는 N+1, 복잡한 조인, 잠금 충돌 문제, 임피던스 불일치가 쌓이는 과정 |
| [02. 쓰기 모델과 읽기 모델의 근본적 차이](./why-cqrs/02-write-read-model-difference.md) | 쓰기 모델의 역할(불변식 보호, 트랜잭션, 일관성)과 읽기 모델의 역할(조인 없는 빠른 조회, 비정규화, 다양한 뷰)이 왜 하나의 모델로 공존하기 어려운가, 두 요구사항을 억지로 합쳤을 때 각 모델이 얼마나 타협하는가 |
| [03. CQRS의 스펙트럼](./why-cqrs/03-cqrs-spectrum.md) | 단순 클래스 분리(같은 DB)부터 완전 분리(다른 DB)까지 CQRS의 세 가지 수준(Simple CQRS / CQRS with Separate Storage / Event-Driven CQRS), 각 수준의 복잡도와 편익, 어느 단계에서 진입할지 판단 기준 |
| [04. CQRS가 해결하는 실제 문제](./why-cqrs/04-problems-cqrs-solves.md) | 복잡한 조회 쿼리가 Aggregate를 오염시키는 문제, 읽기 성능을 위해 쓰기 모델이 타협하는 문제, 역할 기반으로 다른 뷰(관리자 / 사용자 / 통계)가 필요한 문제, 각 문제를 CQRS가 구조적으로 해결하는 방식 |
| [05. CQRS 적용 판단 기준](./why-cqrs/05-when-to-apply-cqrs.md) | 모든 시스템에 CQRS가 답인가, 도메인 복잡도 / 읽기-쓰기 비율 / 팀 역량에 따른 선택 기준, CQRS 없이도 해결 가능한 경우, CQRS 도입 비용이 편익을 초과하는 안티 사례 |

</details>

<br/>

### 🔹 Chapter 2: Command 처리 — 쓰기 모델 설계

> **핵심 질문:** Command는 DTO와 무엇이 다른가? Command Handler는 어디까지 책임지는가? 동시 Command 처리 시 충돌은 어떻게 감지하고, 비동기 Command의 결과는 어떻게 클라이언트에 전달하는가?

<details>
<summary><b>CQS 원칙부터 비동기 Command 결과 반환까지 (6개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. Command와 Query의 명확한 분리](./command-side/01-command-query-separation.md) | Command는 상태를 변경하고 응답을 최소화, Query는 상태를 변경하지 않고 데이터 반환, CQS(Command-Query Separation) 원칙과 CQRS의 차이, 분리가 만드는 설계 명확성 |
| [02. Command 객체 설계](./command-side/02-command-object-design.md) | Command가 의도(Intent)를 표현해야 하는 이유, DTO와의 차이(Command는 검증 포함, 의미가 명확), 구문 검증(형식 오류) vs 의미 검증(불변식 위반)의 책임 분리, Command Handler의 책임 범위 |
| [03. Command Handler와 Aggregate](./command-side/03-command-handler-aggregate.md) | Command Handler가 Repository에서 Aggregate를 로드하고, 도메인 메서드를 호출하고, 저장하는 완전한 흐름, Application Service와 Command Handler의 역할 구분, Aggregate 경계에서 Command를 검증하는 방식 |
| [04. Command Bus 패턴](./command-side/04-command-bus.md) | Command를 Handler로 라우팅하는 미들웨어 구조, Spring에서 Command Bus를 직접 구현하는 방법, Command 실행 전후 횡단 관심사(로깅, 검증, 트랜잭션)를 Command Bus에서 처리하는 방식, Axon Command Gateway와의 비교 |
| [05. Optimistic Locking in CQRS](./command-side/05-optimistic-locking.md) | 동시 Command 처리 시 충돌 감지, Version 필드 기반 낙관적 잠금이 CQRS 쓰기 모델에서 동작하는 방식, 충돌 시 재시도 전략, Event Sourcing에서의 낙관적 잠금(Expected Version) 구현 |
| [06. Command 결과 반환 패턴](./command-side/06-command-result-patterns.md) | Fire-and-Forget vs Acknowledgement vs Result 세 가지 패턴의 차이, 비동기 Command의 결과를 클라이언트에게 전달하는 방법(Polling, WebSocket, SSE), 각 방법의 복잡도와 UX 트레이드오프 |

</details>

<br/>

### 🔹 Chapter 3: Event Sourcing — 이벤트가 진실의 원천

> **핵심 질문:** 현재 상태 대신 이벤트를 저장하면 무엇이 달라지는가? 이벤트 스토어는 어떻게 설계하고, 이벤트가 쌓일수록 느려지는 문제를 스냅샷으로 어떻게 해결하며, 이벤트 스키마가 바뀌면 어떻게 대응하는가?

<details>
<summary><b>Event Sourcing 핵심 아이디어부터 실제 어려움까지 (7개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. Event Sourcing의 핵심 아이디어](./event-sourcing/01-event-sourcing-core.md) | 현재 상태 대신 상태 변화의 이력(이벤트)을 저장하는 발상, "현재 잔고"가 아닌 "모든 입출금 이력"이 원본 데이터인 이유, 회계 장부 설계 원칙에서 배우는 Append-Only 저장의 본질 |
| [02. 이벤트 스토어 설계](./event-sourcing/02-event-store-design.md) | 이벤트 스토어가 일반 DB와 다른 이유(Append Only, 이벤트는 수정/삭제 불가), 이벤트 스트림 구조(Stream ID / Version / EventType / Payload / Timestamp), PostgreSQL로 이벤트 스토어를 직접 구현하는 방법 vs EventStoreDB |
| [03. Aggregate 재구성](./event-sourcing/03-aggregate-reconstitution.md) | 이벤트 스트림을 처음부터 리플레이하여 Aggregate 현재 상태를 계산하는 apply() 메서드 패턴, 리플레이 비용과 이벤트 수가 많아질 때의 성능 문제, 로딩 시간과 이벤트 수의 관계 측정 |
| [04. 스냅샷(Snapshot) 패턴](./event-sourcing/04-snapshot-pattern.md) | 이벤트가 수천 개 쌓였을 때 리플레이 비용 최소화, 스냅샷 저장 시점 결정 기준(N개 이벤트마다 / 주기적으로), 스냅샷과 이후 이벤트를 함께 사용하는 로딩 전략, 스냅샷이 오래됐을 때의 마이그레이션 |
| [05. 이벤트 스키마 진화](./event-sourcing/05-event-schema-evolution.md) | 이벤트는 영원히 저장되므로 스키마 변경이 어렵다는 근본 문제, Upcasting 전략(v1 이벤트를 v2로 변환하는 레이어), Backward Compatibility를 유지하면서 필드를 추가/제거하는 방법, 이벤트 버전 관리 실전 패턴 |
| [06. Event Sourcing의 장점 완전 분해](./event-sourcing/06-event-sourcing-benefits.md) | 완전한 감사 로그(누가 언제 무엇을 바꿨는가), 시간 여행(특정 과거 시점의 Aggregate 상태 재현), 프로덕션 이슈를 이벤트 스트림으로 로컬에서 재현하는 디버깅, 새로운 읽기 모델을 과거 이벤트에서 재구축하는 능력 |
| [07. Event Sourcing의 실제 어려움](./event-sourcing/07-event-sourcing-challenges.md) | Eventually Consistent 읽기 모델이 만드는 UX 복잡성, 이벤트 스키마 마이그레이션의 실제 비용, 이벤트가 너무 많아질 때의 쿼리 어려움, 팀 학습 비용, Event Sourcing을 선택하지 말아야 할 도메인 |

</details>

<br/>

### 🔹 Chapter 4: 읽기 모델과 프로젝션

> **핵심 질문:** 프로젝션은 어떻게 이벤트 스트림을 읽기 모델로 변환하는가? Command 후 읽기 모델이 즉시 업데이트되지 않는 문제를 어떻게 처리하고, 읽기 모델이 오염됐을 때 어떻게 재구축하는가?

<details>
<summary><b>프로젝션 원리부터 장애 처리까지 (6개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. 프로젝션(Projection) 완전 분해](./read-model-projection/01-projection-deep-dive.md) | 이벤트 스트림을 소비하여 읽기 모델을 구축하는 과정, 프로젝션이 Kafka Consumer처럼 동작하는 이유(오프셋 추적), 읽기 모델 DB 선택 기준(RDB / Redis / Elasticsearch), 이벤트 타입별 프로젝션 라우팅 |
| [02. 읽기 모델 설계 원칙](./read-model-projection/02-read-model-design.md) | 읽기 요구사항 중심 비정규화로 설계하는 방법, 조인 없이 한 번에 필요한 데이터를 제공하는 스키마, 역할별 다른 읽기 모델(관리자 뷰 vs 사용자 뷰 vs 통계 뷰)이 같은 이벤트 스트림에서 나오는 구조 |
| [03. Eventual Consistency 처리](./read-model-projection/03-eventual-consistency.md) | Command 후 읽기 모델이 즉시 업데이트되지 않는 문제, 클라이언트에서 지연을 처리하는 UX 전략(낙관적 업데이트, 로딩 인디케이터, 재조회), 읽기 모델 업데이트 지연 모니터링 지표 설계 |
| [04. 프로젝션 재구축(Rebuild)](./read-model-projection/04-projection-rebuild.md) | 읽기 모델이 오염되거나 새로운 뷰가 필요할 때 과거 이벤트부터 다시 빌드하는 방법, 운영 중 무중단 재구축 전략, Blue/Green 프로젝션으로 기존 읽기 모델을 유지하면서 새 버전을 병렬 구축하는 실전 패턴 |
| [05. 여러 읽기 모델의 공존](./read-model-projection/05-multiple-read-models.md) | 같은 이벤트 스트림에서 다양한 읽기 모델 생성(주문 목록 / 주문 상세 / 통계 대시보드 / 검색 인덱스), 각 읽기 모델의 업데이트 시점 조정, 읽기 모델 간 일관성 트레이드오프 |
| [06. 프로젝션 장애 처리](./read-model-projection/06-projection-failure-handling.md) | 프로젝션 처리 실패 시 재처리 전략, 멱등 프로젝션 구현(같은 이벤트를 두 번 처리해도 안전하게), Dead Letter Queue 처리, 프로젝션 처리 실패가 다른 프로젝션에 영향을 주지 않도록 격리하는 방법 |

</details>

<br/>

### 🔹 Chapter 5: CQRS + Event Sourcing 통합 아키텍처

> **핵심 질문:** Command부터 읽기 모델 조회까지 전체 사이클에서 각 단계의 실패는 어떻게 처리하는가? 이벤트 저장과 Kafka 발행의 원자성은 어떻게 보장하고, Exactly-Once 프로젝션 업데이트는 어떻게 구현하는가?

<details>
<summary><b>완전한 통합 흐름부터 마이크로서비스 통합까지 (6개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. 완전한 흐름](./integrated-architecture/01-complete-flow.md) | Command → Aggregate → 이벤트 저장 → Kafka 발행 → 프로젝션 → 읽기 모델 → Query 전체 사이클, 각 단계의 책임과 실패 시 처리, 성공 경로와 실패 경로의 완전한 시퀀스 다이어그램 |
| [02. 이벤트 저장과 발행의 원자성](./integrated-architecture/02-event-store-kafka-atomicity.md) | 이벤트 스토어 저장과 Kafka 발행을 원자적으로 처리해야 하는 이유(저장됐지만 발행 안 된 경우 vs 발행됐지만 저장 안 된 경우), Transactional Outbox 패턴 구현, CDC(Change Data Capture) 방식과 Outbox 방식의 차이와 선택 기준 |
| [03. DDD와의 통합](./integrated-architecture/03-ddd-integration.md) | Aggregate가 이벤트를 생성하고, 이벤트 스토어가 저장하고, 프로젝션이 읽기 모델을 만들고, Saga가 여러 Aggregate를 조율하는 완전한 DDD + CQRS + Event Sourcing 아키텍처 |
| [04. 처리 보장(Processing Guarantee)](./integrated-architecture/04-processing-guarantee.md) | Exactly-Once 프로젝션 업데이트를 구현하는 방법, 이벤트 중복 처리 방지(멱등키 기반), Offset 기반 체크포인트 관리, At-Least-Once 환경에서 멱등 처리로 Exactly-Once 효과를 내는 방법 |
| [05. 성능 최적화](./integrated-architecture/05-performance-optimization.md) | 이벤트 스토어 읽기 성능 최적화(인덱스 전략 — Stream ID + Version 복합 인덱스), 프로젝션 병렬 처리(파티션별 독립 처리), 읽기 모델 Redis 캐싱 전략, Hot Path(실시간 조회)와 Cold Path(통계/배치)의 분리 |
| [06. 마이크로서비스와의 통합](./integrated-architecture/06-microservices-integration.md) | CQRS가 서비스 내부 패턴인 경우 vs Bounded Context 간 통합 패턴인 경우의 차이, MSA 환경에서 읽기 모델을 서비스 간 공유하는 전략, 서비스 간 이벤트 계약(Event Contract) 관리 |

</details>

<br/>

### 🔹 Chapter 6: Spring + Axon Framework 실전 구현

> **핵심 질문:** Axon의 `@CommandHandler` / `@EventSourcingHandler` / `@QueryHandler`는 내부적으로 어떻게 동작하는가? Axon 없이 순수 Spring으로 CQRS를 구현하면 무엇을 직접 구현해야 하는가?

<details>
<summary><b>Axon 아키텍처부터 순수 Spring 직접 구현까지 (5개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. Axon Framework 아키텍처](./spring-axon-implementation/01-axon-architecture.md) | `@CommandHandler` / `@EventHandler` / `@QueryHandler` 어노테이션의 내부 흐름, Axon Server의 역할(이벤트 스토어 + 메시지 라우터 + Tracking Event Processor), Spring Boot AutoConfiguration이 Axon을 등록하는 방식 |
| [02. Aggregate 구현](./spring-axon-implementation/02-aggregate-implementation.md) | `@Aggregate` / `@AggregateIdentifier` / `@CommandHandler` / `@EventSourcingHandler` 어노테이션, 이벤트를 `AggregateLifecycle.apply()`로 발행하고 `@EventSourcingHandler`로 상태를 변경하는 패턴, Expected Version으로 낙관적 잠금 처리 |
| [03. 프로젝션 구현](./spring-axon-implementation/03-projection-implementation.md) | `@EventHandler`로 이벤트를 수신하여 읽기 모델 업데이트, `@ResetHandler`로 프로젝션 재구축 지원, JPA Repository로 읽기 모델 저장, Tracking Event Processor의 Thread와 Segment 기반 병렬 처리 |
| [04. Query 처리](./spring-axon-implementation/04-query-handling.md) | `@QueryHandler`로 읽기 모델 조회, Subscription Query로 읽기 모델이 업데이트될 때 클라이언트에 실시간 Push, Scatter-Gather Query로 여러 서비스의 읽기 모델을 조합하는 분산 읽기 |
| [05. Axon 없이 직접 구현](./spring-axon-implementation/05-pure-spring-implementation.md) | Axon이 없는 순수 Spring 기반 CQRS 구현 전체, PostgreSQL 이벤트 스토어 직접 구현(스키마 + 낙관적 잠금 + 스트림 조회), Kafka 기반 프로젝션 업데이트, Command Bus 직접 구현 |

</details>

<br/>

### 🔹 Chapter 7: 실전 프로젝트와 트레이드오프

> **핵심 질문:** 기존 시스템에 CQRS를 점진적으로 도입하려면 어디서부터 시작해야 하는가? 모든 도메인에 Event Sourcing이 맞는가? CQRS / ES의 실제 비용은 얼마인가?

<details>
<summary><b>은행 계좌 도메인 구현부터 실제 비용과 편익까지 (5개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. 은행 계좌 도메인 구현](./real-world-tradeoffs/01-bank-account-domain.md) | Account Aggregate(Deposit / Withdraw / Transfer Command), 잔고 부족 불변식 검증, 읽기 모델(계좌 요약 / 거래 내역 / 월별 통계), 모든 거래의 완전한 감사 로그, 시간 여행으로 특정 시점 잔고 조회 |
| [02. Event Sourcing 없는 CQRS](./real-world-tradeoffs/02-cqrs-without-event-sourcing.md) | Event Sourcing 없이 DB 트리거 / CDC로 읽기 모델을 동기화하는 방법, 단순 CQRS의 실용적 구현(같은 DB, 다른 읽기 전용 스키마), Event Sourcing 도입 없이 CQRS의 이점을 얻는 수준별 전략 |
| [03. 점진적 도입 전략](./real-world-tradeoffs/03-incremental-adoption.md) | 기존 시스템에 CQRS를 점진적으로 적용하는 방법, 가장 복잡한 조회 쿼리부터 분리하는 전략, 읽기 모델 도입 → Command 분리 → Event Sourcing 순서로 단계별로 도입하는 실전 로드맵 |
| [04. 안티패턴](./real-world-tradeoffs/04-anti-patterns.md) | CQRS를 단순 폴더 분리(command / query 패키지)로 구현, 읽기 모델을 쓰기 모델과 항상 강결합, 모든 도메인에 Event Sourcing 적용, 이벤트 스키마를 자주 변경하는 잘못된 패턴과 그 결과 |
| [05. CQRS / ES의 실제 비용과 편익](./real-world-tradeoffs/05-cost-benefit-analysis.md) | 적합한 도메인(금융, 재고, 주문 — 감사 로그 필수, 읽기-쓰기 비율 불균형)과 부적합한 도메인(단순 CRUD), 팀 학습 비용, 운영 복잡도(이벤트 스토어 + 프로젝션 + 읽기 모델 모니터링), 실제 도입 사례 |

</details>

---

## 🛠️ 실험 환경

```yaml
# docker-compose.yml
services:
  axon-server:
    image: axoniq/axonserver:latest
    ports:
      - "8024:8024"   # Dashboard UI
      - "8124:8124"   # gRPC (Command / Event / Query 라우팅)

  # 실제 서비스 구현 시 build 경로를 프로젝트에 맞게 설정하세요
  command-service:
    build: ./command-service
    ports:
      - "8081:8080"
    environment:
      AXON_AXONSERVER_SERVERS: axon-server:8124
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/cqrs_db

  query-service:
    build: ./query-service
    ports:
      - "8082:8080"
    environment:
      AXON_AXONSERVER_SERVERS: axon-server:8124
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/cqrs_db
      SPRING_REDIS_HOST: redis

  kafka:
    image: confluentinc/cp-kafka:7.5.0
    ports:
      - "9092:9092"
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 1
      KAFKA_TRANSACTION_STATE_LOG_MIN_ISR: 1
      KAFKA_AUTO_CREATE_TOPICS_ENABLE: "true"

  zookeeper:
    image: confluentinc/cp-zookeeper:7.5.0
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181

  postgres:
    image: postgres:15
    ports:
      - "5432:5432"
    environment:
      POSTGRES_DB: cqrs_db
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres

  redis:
    image: redis:7.0
    ports:
      - "6379:6379"
```

```sql
-- 이벤트 스토어 스키마 (PostgreSQL 직접 구현 시)
CREATE TABLE event_store (
  global_seq  BIGSERIAL    PRIMARY KEY,            -- 전역 순번 (Projection 폴링 기준)
  stream_id   VARCHAR(255) NOT NULL,               -- Aggregate ID (예: "account-ACC-001")
  version     BIGINT       NOT NULL,               -- 스트림 내 순번 (낙관적 잠금 버전)
  event_type  VARCHAR(255) NOT NULL,               -- 이벤트 타입
  payload     JSONB        NOT NULL,               -- 이벤트 데이터
  metadata    JSONB,                               -- 상관관계 ID, 원인 ID 등
  occurred_at TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
  UNIQUE (stream_id, version)                      -- 낙관적 잠금 보장
);

-- Aggregate 로드: 특정 스트림의 이벤트를 version 순으로 읽기
CREATE INDEX idx_event_store_stream ON event_store (stream_id, version);

-- Projection 폴링: 마지막으로 처리한 global_seq 이후 이벤트를 순서대로 로드
CREATE INDEX idx_event_store_seq ON event_store (global_seq);
```

---

## 📖 각 문서 구성 방식

모든 문서는 동일한 구조로 작성됩니다.

| 섹션 | 설명 |
|------|------|
| 🎯 **핵심 질문** | 이 문서를 읽고 나면 답할 수 있는 질문 |
| 🔍 **왜 이 개념이 중요한가** | 실무에서 마주치는 문제 상황과 이 개념의 연결 |
| 😱 **흔한 실수** | Before — 단일 모델 / 이벤트 없는 설계로 접근할 때의 문제 |
| ✨ **올바른 접근** | After — CQRS / Event Sourcing을 적용한 설계 |
| 🔬 **내부 동작 원리** | 이벤트 스토어 스키마, 프로젝션 처리 흐름, Command 처리 시퀀스 |
| 💻 **실전 코드** | Axon Framework + Spring Boot + Kafka 기반 완전한 예제 |
| 📊 **패턴 비교** | 상태 저장 vs 이벤트 저장, 동기 읽기 모델 vs 비동기 읽기 모델 |
| ⚖️ **트레이드오프** | 이 설계의 장단점, 언제 다른 접근을 택할 것인가 |
| 📌 **핵심 정리** | 한 화면 요약 |
| 🤔 **생각해볼 문제** | 개념을 더 깊이 이해하기 위한 질문 + 해설 |

---

## 🗺️ 추천 학습 경로

<details>
<summary><b>🟢 "CQRS를 폴더 분리로만 이해한다" — 핵심 개념 빠른 진입 (3일)</b></summary>

<br/>

```
Day 1  Ch1-01  단일 모델의 임피던스 불일치 → 왜 분리가 필요한가
       Ch1-02  쓰기 모델과 읽기 모델의 근본적 차이

Day 2  Ch2-01  Command와 Query의 명확한 분리 (CQS vs CQRS)
       Ch2-02  Command 객체 설계 → DTO와 무엇이 다른가

Day 3  Ch4-01  프로젝션 완전 분해 → 읽기 모델이 만들어지는 원리
       Ch4-03  Eventual Consistency 처리 → UX 전략
```

</details>

<details>
<summary><b>🟡 "Event Sourcing 내부 구조를 제대로 이해하고 싶다" — 핵심 집중 (1주)</b></summary>

<br/>

```
Day 1  Ch1-01  단일 모델의 임피던스 불일치
       Ch1-03  CQRS의 스펙트럼 → 어느 수준으로 적용할 것인가

Day 2  Ch3-01  Event Sourcing의 핵심 아이디어
       Ch3-02  이벤트 스토어 설계 → PostgreSQL 직접 구현

Day 3  Ch3-03  Aggregate 재구성 → apply() 리플레이 패턴
       Ch3-04  스냅샷 패턴 → 수천 개 이벤트 로딩 비용 해결

Day 4  Ch4-01  프로젝션 완전 분해
       Ch4-04  프로젝션 재구축 → Blue/Green 프로젝션 전략

Day 5  Ch5-01  완전한 흐름 → 전체 사이클 시퀀스
       Ch5-02  이벤트 저장과 발행의 원자성 → Transactional Outbox

Day 6  Ch6-01  Axon Framework 아키텍처
       Ch6-02  Aggregate 구현 → @EventSourcingHandler 패턴

Day 7  Ch7-01  은행 계좌 도메인 완전 구현
       Ch7-05  CQRS / ES의 실제 비용과 편익
```

</details>

<details>
<summary><b>🔴 "CQRS + Event Sourcing 아키텍처를 실무에 도입한다" — 전체 정복 (7주)</b></summary>

<br/>

```
1주차  Chapter 1 전체 — 왜 CQRS인가
        → 기존 JPA 코드의 임피던스 불일치 지점 직접 찾기

2주차  Chapter 2 전체 — Command 처리 설계
        → Command Bus 직접 구현, 낙관적 잠금 충돌 시뮬레이션

3주차  Chapter 3 전체 — Event Sourcing 완전 분해
        → PostgreSQL 이벤트 스토어 직접 구현, 스냅샷 도입 전후 성능 비교

4주차  Chapter 4 전체 — 읽기 모델과 프로젝션
        → 3개 읽기 모델(목록 / 상세 / 통계) 동시 구성, 프로젝션 재구축 실전

5주차  Chapter 5 전체 — 통합 아키텍처
        → Transactional Outbox 구현, Exactly-Once 프로젝션 업데이트 구현

6주차  Chapter 6 전체 — Spring + Axon 실전 구현
        → Axon Server + Kafka + PostgreSQL Docker Compose 전체 실행

7주차  Chapter 7 전체 — 실전 프로젝트와 트레이드오프
        → 은행 계좌 도메인 완전 구현, 점진적 도입 전략 설계
```

</details>

---

## 🔗 선행 학습 및 연관 레포지토리

> 💡 이 레포는 아래 레포지토리와 긴밀하게 연결됩니다. 선행 학습을 마쳤을 때 각 챕터의 깊이가 달라집니다.

| 레포 | 연관 챕터 | 연결 지점 |
|------|----------|-----------|
| [ddd-deep-dive](https://github.com/dev-book-lab/ddd-deep-dive) | Ch2, Ch3, Ch5-03 | Aggregate가 이벤트를 생성하는 방식, Bounded Context 간 이벤트 계약 |
| [kafka-deep-dive](https://github.com/dev-book-lab/kafka-deep-dive) | Ch4, Ch5-02, Ch5-04 | 이벤트 스트림, Consumer Group, Exactly-Once 보장, Outbox → Kafka 발행 |
| [mysql-deep-dive](https://github.com/dev-book-lab/mysql-deep-dive) | Ch2-05, Ch4-02 | MVCC와 낙관적 잠금, 읽기 모델 인덱스 최적화 |
| [redis-deep-dive](https://github.com/dev-book-lab/redis-deep-dive) | Ch5-05 | 읽기 모델 캐싱 전략, Cache-Aside와 읽기 모델 무효화 |
| [observability-deep-dive](https://github.com/dev-book-lab/observability-deep-dive) | Ch4-03, Ch5-05 | 프로젝션 처리 지연 모니터링, 이벤트 스토어 성능 추적 |

---

## 🙏 Reference

- [Implementing Domain-Driven Design — Vaughn Vernon](https://www.oreilly.com/library/view/implementing-domain-driven-design/9780133039900/) — CQRS + Event Sourcing 챕터
- [Martin Fowler — CQRS](https://martinfowler.com/bliki/CQRS.html)
- [Greg Young — CQRS Documents](https://cqrs.wordpress.com/)
- [Event Sourcing Pattern — microservices.io](https://microservices.io/patterns/data/event-sourcing.html)
- [CQRS Journey — Microsoft](https://docs.microsoft.com/en-us/previous-versions/msp-n-p/jj554200)
- [Axon Framework 공식 문서](https://docs.axoniq.io/)
- [EventStoreDB 공식 문서](https://www.eventstore.com/eventstoredb)
- [Designing Data-Intensive Applications — Martin Kleppmann](https://dataintensive.net/)

---

<div align="center">

**⭐️ 도움이 되셨다면 Star를 눌러주세요!**

Made with ❤️ by [Dev Book Lab](https://github.com/dev-book-lab)

<br/>

*"읽기와 쓰기를 나누는 것과, 왜 이벤트가 진실의 원천이 되어야 하는지 — 그리고 그것이 시스템 전체 설계를 어떻게 바꾸는지 아는 것은 다르다"*

</div>
