<div align="center">

# 🐇 RabbitMQ Deep Dive

**"메시지를 보내는 것과, Exchange가 어떤 Queue로 어떻게 라우팅하는지 아는 것은 다르다"**

<br/>

> *"`@RabbitListener` 붙이면 메시지를 받겠지 — 와 — Direct/Topic/Fanout Exchange가 메시지를 어떻게 라우팅하는지, Acknowledgement 모드가 메시지 내구성에 어떤 영향을 미치는지, Prefetch Count가 왜 Consumer 처리량의 핵심인지 아는 것의 차이를 만드는 레포"*

AMQP 0-9-1 프레임이 TCP 위에서 Channel을 멀티플렉싱하는 원리, Exchange → Binding → Queue 3단계 라우팅이 Kafka의 Topic 직접 발행과 근본적으로 다른 이유, autoAck=true가 왜 메시지를 유실시키는지, Dead Letter Exchange가 어떻게 실패한 메시지를 보존하는지  
**왜 이렇게 설계됐는가** 라는 질문으로 RabbitMQ 내부를 끝까지 파헤칩니다

<br/>

[![GitHub](https://img.shields.io/badge/GitHub-dev--book--lab-181717?style=flat-square&logo=github)](https://github.com/dev-book-lab)
[![RabbitMQ](https://img.shields.io/badge/RabbitMQ-3.12-FF6600?style=flat-square&logo=rabbitmq&logoColor=white)](https://www.rabbitmq.com/documentation.html)
[![Spring](https://img.shields.io/badge/Spring_AMQP-3.x-6DB33F?style=flat-square&logo=spring&logoColor=white)](https://docs.spring.io/spring-amqp/docs/current/reference/html/)
[![Docs](https://img.shields.io/badge/Docs-38개-blue?style=flat-square&logo=readthedocs&logoColor=white)](./README.md)
[![License](https://img.shields.io/badge/License-MIT-yellow?style=flat-square&logo=opensourceinitiative&logoColor=white)](./LICENSE)

</div>

---

## 🎯 이 레포에 대하여

RabbitMQ에 관한 자료는 넘쳐납니다. 하지만 대부분은 **"어떻게 쓰나"** 에서 멈춥니다.

| 일반 자료 | 이 레포 |
|----------|---------|
| "`@RabbitListener`를 붙이면 메시지를 받습니다" | `SimpleMessageListenerContainer` → AMQP `basic.consume` → Channel Prefetch 전 과정, 어떻게 메시지가 Push되는지 |
| "Exchange를 설정하세요" | Direct/Topic/Fanout/Headers Exchange가 Binding 규칙을 어떻게 평가하는지, Routing Key가 매칭되지 않으면 메시지가 어디로 가는지 |
| "RabbitMQ는 메시지 큐입니다" | AMQP 0-9-1 프레임 구조(Method/Header/Body), Channel 멀티플렉싱이 하나의 TCP 연결에서 어떻게 동작하는지 |
| "autoAck를 false로 설정하세요" | autoAck=true일 때 Consumer 장애 시 메시지가 사라지는 정확한 시점, basicNack(requeue=false)가 Dead Letter Exchange로 메시지를 보내는 원리 |
| "Prefetch Count를 설정하세요" | Prefetch=1이면 Consumer가 처리 중인 메시지 1개만 받는 이유, 크게 설정하면 불균등 분배가 발생하는 원인 |
| "Dead Letter Queue를 쓰세요" | 메시지가 Dead Letter가 되는 3가지 조건(Nack/TTL/Queue Full), DLX로 실패 메시지를 별도 Queue로 라우팅하는 설정 방법 |
| "Kafka와 비교해보세요" | Push(RabbitMQ) vs Pull(Kafka), 메시지 삭제(RabbitMQ) vs 로그 보관(Kafka) — 언제 어떤 선택이 맞는지 결정 기준 |
| 이론 나열 | 실행 가능한 RabbitMQ Management UI 실험 + Spring AMQP 설정 + Docker Compose 클러스터 환경 |

---

## 🚀 빠른 시작

각 챕터의 첫 문서부터 바로 학습을 시작하세요!

[![Architecture](https://img.shields.io/badge/🔹_Architecture-RabbitMQ가_해결하는_문제-FF6600?style=for-the-badge&logo=rabbitmq&logoColor=white)](./amqp-architecture/01-why-rabbitmq.md)
[![Exchange](https://img.shields.io/badge/🔹_Exchange-Direct_Exchange_라우팅-FF6600?style=for-the-badge&logo=rabbitmq&logoColor=white)](./exchange-routing/01-direct-exchange.md)
[![Reliability](https://img.shields.io/badge/🔹_Reliability-메시지_유실의_세_지점-FF6600?style=for-the-badge&logo=rabbitmq&logoColor=white)](./message-reliability/01-message-loss-points.md)
[![Patterns](https://img.shields.io/badge/🔹_Patterns-Work_Queue_경쟁_소비자-FF6600?style=for-the-badge&logo=rabbitmq&logoColor=white)](./messaging-patterns/01-work-queue.md)
[![Ops](https://img.shields.io/badge/🔹_Operations-처리량_최적화-FF6600?style=for-the-badge&logo=rabbitmq&logoColor=white)](./performance-operations/01-throughput-optimization.md)
[![Kafka](https://img.shields.io/badge/🔹_vs_Kafka-근본적_철학_차이-FF6600?style=for-the-badge&logo=apachekafka&logoColor=white)](./rabbitmq-vs-kafka/01-philosophy-difference.md)
[![Spring](https://img.shields.io/badge/🔹_Spring_AMQP-RabbitTemplate_아키텍처-FF6600?style=for-the-badge&logo=spring&logoColor=white)](./spring-amqp/01-spring-amqp-architecture.md)

---

## 📚 전체 학습 지도

> 💡 각 섹션을 클릭하면 상세 문서 목록이 펼쳐집니다

<br/>

### 🔹 Chapter 1: RabbitMQ 아키텍처와 AMQP 프로토콜

> **핵심 질문:** RabbitMQ는 어떤 문제를 해결하는가? AMQP 0-9-1은 어떻게 Channel을 멀티플렉싱하고, Exchange → Binding → Queue 3단계 라우팅은 왜 Kafka와 근본적으로 다른가?

<details>
<summary><b>Producer-Consumer 결합도 문제부터 Quorum Queue 클러스터링까지 (6개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. RabbitMQ가 해결하는 문제](./amqp-architecture/01-why-rabbitmq.md) | 동기 호출 체인의 결합도 문제, Producer가 Consumer의 존재를 몰라도 되는 설계, 트래픽 버퍼로서의 메시지 큐, RabbitMQ(메시지 브로커) vs Kafka(분산 로그)의 근본적 철학 차이 |
| [02. AMQP 프로토콜 완전 분해](./amqp-architecture/02-amqp-protocol.md) | AMQP 0-9-1 프레임 구조(Method/Header/Body/Heartbeat), Channel 멀티플렉싱(하나의 TCP 연결에 여러 채널), Connection vs Channel 비용 차이, AMQP의 3단계 라우팅(Exchange → Binding → Queue) |
| [03. Exchange 완전 분해](./amqp-architecture/03-exchange-internals.md) | Exchange가 메시지를 받아 어떤 Queue로 보낼지 결정하는 라우팅 컴포넌트, Binding이 Exchange와 Queue를 연결하는 규칙, Routing Key의 역할, Default Exchange(이름 없는 Exchange)의 동작 |
| [04. Queue 내부 구조](./amqp-architecture/04-queue-internals.md) | Queue의 FIFO 구조, 메시지 저장 방식(메모리 vs 디스크), Queue 속성(Durable/Exclusive/AutoDelete), Quorum Queue vs Classic Queue(내구성과 가용성 트레이드오프) |
| [05. 가상 호스트(vHost)](./amqp-architecture/05-vhost.md) | vHost로 브로커 내 논리적 분리, 팀/환경별(dev/staging/prod) vHost 설계 전략, 권한 관리와 격리 보장 범위 |
| [06. RabbitMQ 클러스터링](./amqp-architecture/06-clustering.md) | 브로커 클러스터 구성 원리, Queue 미러링(Mirrored Queue) vs Quorum Queue(Raft 기반 합의), 클러스터 네트워크 파티션(Split-Brain) 감지와 처리 전략 |

</details>

<br/>

### 🔹 Chapter 2: Exchange 유형과 라우팅 패턴

> **핵심 질문:** Direct/Topic/Fanout/Headers Exchange는 각각 어떤 규칙으로 Queue를 선택하는가? Dead Letter Exchange는 실패한 메시지를 어떻게 잡아내고, 도메인 이벤트를 Exchange로 어떻게 설계하는가?

<details>
<summary><b>Direct Exchange 1:1 라우팅부터 마이크로서비스 이벤트 라우팅 설계까지 (6개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. Direct Exchange](./exchange-routing/01-direct-exchange.md) | Routing Key가 Binding Key와 정확히 일치할 때 라우팅하는 원리, 1:1 라우팅과 1:N 라우팅(같은 Routing Key에 여러 Queue 바인딩), 사용 사례와 Kafka Topic 직접 발행과의 비교 |
| [02. Topic Exchange](./exchange-routing/02-topic-exchange.md) | 와일드카드 패턴(`*`=단어 하나, `#`=0개 이상)으로 유연한 라우팅, Routing Key를 계층적으로 설계하는 방법(`order.payment.failed`), 구독 패턴의 유연성과 Kafka Consumer Group 필터링과의 차이 |
| [03. Fanout Exchange](./exchange-routing/03-fanout-exchange.md) | Routing Key를 무시하고 바인딩된 모든 Queue에 브로드캐스트하는 원리, Pub/Sub 패턴 구현, 이벤트 하나를 여러 서비스가 처리하는 설계, Kafka Fanout과의 차이 |
| [04. Headers Exchange](./exchange-routing/04-headers-exchange.md) | Routing Key 대신 메시지 헤더로 라우팅하는 원리, `x-match=all`(모든 헤더 일치) vs `x-match=any`(하나 이상 일치), Direct/Topic보다 유연한 경우와 그렇지 않은 경우 |
| [05. Dead Letter Exchange(DLX)](./exchange-routing/05-dead-letter-exchange.md) | 메시지가 Dead Letter가 되는 3가지 조건(소비 실패/TTL 만료/Queue 가득 참), DLX로 실패 메시지를 별도 Queue로 라우팅하는 설정, Dead Letter Queue 모니터링과 재처리 전략 |
| [06. 라우팅 패턴 설계 가이드](./exchange-routing/06-routing-design-guide.md) | 도메인 이벤트를 Exchange로 설계하는 방법(`OrderExchange` → `order.placed`, `order.cancelled`), 마이크로서비스 간 이벤트 라우팅 설계, Exchange 재사용 vs 서비스별 Exchange 분리 기준 |

</details>

<br/>

### 🔹 Chapter 3: 메시지 신뢰성 — 유실 없는 처리

> **핵심 질문:** 메시지는 어디서 유실되는가? Publisher Confirm은 어떻게 Broker 수신을 보장하고, autoAck=true는 왜 위험하며, Prefetch Count는 Consumer 처리량에 어떤 영향을 주는가?

<details>
<summary><b>메시지 유실 3지점부터 Outbox + Publisher Confirm 완전 보장 패턴까지 (7개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. 메시지 유실의 세 지점](./message-reliability/01-message-loss-points.md) | Publisher → Broker 전송 중 실패, Broker Queue 저장 중 장애, Consumer 처리 중 실패 — 각 지점에서 메시지가 사라지는 정확한 시점과 지점별 해결 방법 |
| [02. Publisher Confirm](./message-reliability/02-publisher-confirm.md) | Broker가 메시지 수신을 확인하는 비동기 확인 메커니즘, `confirmSelect()`로 Confirm 모드 활성화, 개별 Confirm vs 배치 Confirm 처리량 비교, Mandatory 플래그(라우팅 불가 메시지 반환) |
| [03. 메시지 영속성(Persistence)](./message-reliability/03-message-persistence.md) | Queue Durable=true + Message `DeliveryMode=PERSISTENT` 조합이 필요한 이유, 메시지가 디스크에 `fsync`되는 시점, 영속성이 처리량에 미치는 영향, Lazy Queue로 메모리를 절약하는 방법 |
| [04. Consumer Acknowledgement 완전 분해](./message-reliability/04-consumer-ack.md) | `autoAck=true`의 위험(처리 전 메시지 삭제되는 정확한 시점), 수동 `basicAck`, `basicNack` vs `basicReject` 차이, `requeue=true` vs `requeue=false` 선택 기준 |
| [05. 재시도 전략 설계](./message-reliability/05-retry-strategy.md) | 처리 실패 시 즉시 requeue의 무한 루프 문제, Exponential Backoff 구현(TTL + Dead Letter + 재발행 패턴), 재시도 횟수 제한, 최종 실패 메시지의 Dead Letter Queue 처리 |
| [06. Prefetch Count(QoS) 완전 분해](./message-reliability/06-prefetch-count.md) | Prefetch=1이면 Consumer가 처리 중인 메시지 1개만 받는 원리, Prefetch를 크게 설정하면 처리량은 높지만 불균등 분배가 발생하는 이유, 최적 Prefetch Count 산정 방법 |
| [07. 트랜잭션 vs Publisher Confirm](./message-reliability/07-transaction-vs-confirm.md) | AMQP 트랜잭션(`txSelect`/`txCommit`)의 성능 비용(처리량 25배 하락), Publisher Confirm이 트랜잭션보다 선호되는 이유, 완전한 메시지 처리 보장 패턴(Outbox + Publisher Confirm) |

</details>

<br/>

### 🔹 Chapter 4: 실무 메시징 패턴

> **핵심 질문:** Work Queue는 어떻게 경쟁 소비자를 공정하게 분배하는가? RPC는 Reply-To Queue로 어떻게 구현하고, Saga 패턴에서 RabbitMQ는 어떤 역할을 하는가?

<details>
<summary><b>Work Queue부터 Choreography Saga 보상 트랜잭션까지 (6개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. Work Queue(경쟁 소비자)](./messaging-patterns/01-work-queue.md) | 여러 Consumer가 하나의 Queue를 공유하여 병렬 처리하는 원리, Round-Robin 분배의 기본 동작, Prefetch Count로 공정한 분배 구현, 처리 시간이 긴 작업의 병렬화 설계 |
| [02. Publish/Subscribe](./messaging-patterns/02-pubsub.md) | Fanout Exchange로 브로드캐스트, 각 Consumer가 자신만의 임시 Queue를 생성하는 방법, Consumer 추가/제거 시 자동 확장/축소, 로그 시스템·알림 시스템 구현 |
| [03. RPC(Remote Procedure Call)](./messaging-patterns/03-rpc.md) | Reply-To Queue와 Correlation ID로 요청-응답 패턴을 구현하는 방법, 동기 RPC의 한계와 타임아웃 처리, RabbitMQ를 RPC 백엔드로 쓰는 경우와 그러지 않는 경우 |
| [04. Priority Queue](./messaging-patterns/04-priority-queue.md) | Queue에 `x-max-priority` 설정으로 메시지 우선순위 처리하는 원리, 높은 우선순위 메시지가 먼저 소비되는 내부 구조, 낮은 우선순위 메시지 굶주림(Starvation) 방지 전략 |
| [05. Delayed/Scheduled Message](./messaging-patterns/05-delayed-message.md) | RabbitMQ Delayed Message Plugin 또는 TTL + DLX 조합으로 지연 메시지를 구현하는 두 가지 방법, 정확한 지연 보장의 한계, Quartz 스케줄러와의 책임 분리 |
| [06. Saga 패턴 구현](./messaging-patterns/06-saga-pattern.md) | Choreography Saga에서 각 서비스가 이벤트를 발행하고 구독하는 흐름, 보상 트랜잭션 이벤트 라우팅 설계, Orchestration Saga에서 Orchestrator의 명령 발행 패턴 |

</details>

<br/>

### 🔹 Chapter 5: 성능 튜닝과 운영

> **핵심 질문:** Consumer 처리 속도보다 발행 속도가 빠를 때 Queue Depth는 어떻게 제어하고, Flow Control은 언제 발동하며, 클러스터 노드 장애 시 Quorum Queue는 어떻게 자동 복구하는가?

<details>
<summary><b>처리량 최적화부터 클러스터 Rolling Upgrade까지 (5개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. 처리량 최적화](./performance-operations/01-throughput-optimization.md) | Batch Publishing, Publisher Confirm 비동기 처리, Consumer 병렬화(동시 Consumer 수), 메시지 압축, 적절한 Prefetch Count 설정이 처리량에 미치는 복합적 영향 |
| [02. 메모리 관리](./performance-operations/02-memory-management.md) | RabbitMQ 메모리 임계값(`vm_memory_high_watermark`), 메모리 초과 시 Publisher 차단(Flow Control) 동작 방식, Lazy Queue로 메시지를 디스크에 저장하여 메모리를 절약하는 방법 |
| [03. 모니터링 핵심 지표](./performance-operations/03-monitoring.md) | Queue Depth(메시지 적체), Consumer Utilisation(소비자 효율), Publish Rate vs Deliver Rate 비교, 연결/채널 수 관리, RabbitMQ Management UI와 Prometheus Exporter 설정 |
| [04. 운영 중 발생하는 문제 패턴](./performance-operations/04-operational-issues.md) | Queue 메시지 무한 적체(Consumer 처리 속도 < 발행 속도), Unacknowledged 메시지 누적(Prefetch 크고 처리 느림), 연결 폭풍(서버 재시작 시 수백 개 클라이언트 동시 재연결) 진단과 해결 |
| [05. 클러스터 운영](./performance-operations/05-cluster-operations.md) | 노드 추가/제거 절차, Queue 균등 분산 전략, 노드 장애 시 Quorum Queue 자동 복구 원리, Rolling Upgrade 절차와 무중단 배포 |

</details>

<br/>

### 🔹 Chapter 6: RabbitMQ vs Kafka — 언제 무엇을 선택하는가

> **핵심 질문:** RabbitMQ와 Kafka는 어떤 문제를 각각 잘 해결하는가? Push vs Pull, 메시지 삭제 vs 로그 보관의 차이가 시스템 설계에 어떤 영향을 주는가?

<details>
<summary><b>메시지 브로커 vs 분산 로그 철학 차이부터 두 시스템 동시 운영 아키텍처까지 (4개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. 근본적 철학 차이 완전 분해](./rabbitmq-vs-kafka/01-philosophy-difference.md) | RabbitMQ(메시지 브로커 — Consumer가 처리하면 삭제) vs Kafka(분산 로그 — 메시지를 보관, Consumer가 오프셋으로 읽기), Push 기반 vs Pull 기반 소비 방식의 트레이드오프 |
| [02. 사용 사례 비교](./rabbitmq-vs-kafka/02-use-case-comparison.md) | RabbitMQ가 적합한 경우(복잡한 라우팅, 요청-응답 패턴, 메시지 우선순위, 작업 큐, 낮은 지연시간 < 10ms), Kafka가 적합한 경우(이벤트 소싱, 로그 집계, 대용량 스트리밍, 재처리, 여러 Consumer 그룹) |
| [03. 성능 특성 비교](./rabbitmq-vs-kafka/03-performance-comparison.md) | RabbitMQ 낮은 지연시간(마이크로초~밀리초) vs Kafka 높은 처리량(초당 수백만 메시지), 각각의 최적 메시지 크기와 규모, Prefetch(RabbitMQ) vs Batch Size(Kafka) 튜닝 비교 |
| [04. 함께 쓰는 경우](./rabbitmq-vs-kafka/04-using-both.md) | RabbitMQ(작업 큐, 실시간 알림) + Kafka(이벤트 스트림, 감사 로그)를 동시에 운영하는 아키텍처, 각 미들웨어의 책임 분리, 데이터 흐름 설계 |

</details>

<br/>

### 🔹 Chapter 7: Spring AMQP 통합

> **핵심 질문:** `RabbitTemplate`은 어떻게 AMQP 연결을 관리하고, `@RabbitListener`는 내부적으로 어떤 Container를 사용하며, RetryTemplate과 DeadLetterPublishingRecoverer는 어떻게 연동되는가?

<details>
<summary><b>RabbitTemplate 아키텍처부터 EmbeddedRabbitMQ 테스트 전략까지 (4개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. Spring AMQP 아키텍처](./spring-amqp/01-spring-amqp-architecture.md) | `RabbitTemplate`(발행), `@RabbitListener`(소비), `RabbitAdmin`(Exchange/Queue/Binding 자동 생성), `SimpleMessageListenerContainer` vs `DirectMessageListenerContainer` 차이와 선택 기준 |
| [02. 메시지 직렬화 전략](./spring-amqp/02-message-serialization.md) | `Jackson2JsonMessageConverter`(JSON), `SimpleMessageConverter`(String/byte[]), 커스텀 `MessageConverter` 구현, 메시지 헤더에 타입 정보를 포함하는 방법과 역직렬화 시 타입 안전성 확보 |
| [03. 에러 처리와 재시도](./spring-amqp/03-error-handling-retry.md) | `RetryTemplate`으로 Consumer 재시도 설정, `RejectAndDontRequeueRecoverer`로 최종 실패 처리, `DeadLetterPublishingRecoverer`로 DLX 연동, `@RabbitListener concurrency` 설정으로 병렬 소비자 수 제어 |
| [04. Spring Boot Auto-configuration](./spring-amqp/04-spring-boot-autoconfiguration.md) | `RabbitAutoConfiguration` 동작 방식, `application.yml`로 Exchange/Queue/Binding 선언, 테스트에서 `EmbeddedRabbitMQ`(또는 Testcontainers) 활용, `@SpringBootTest` 통합 테스트 전략 |

</details>

---

## 🧪 실험 환경

```yaml
# docker-compose.yml
services:
  rabbitmq:
    image: rabbitmq:3.12-management
    hostname: rabbitmq-node1
    ports:
      - "5672:5672"    # AMQP
      - "15672:15672"  # Management UI
      - "15692:15692"  # Prometheus metrics (rabbitmq_prometheus plugin)
    environment:
      RABBITMQ_DEFAULT_USER: admin
      RABBITMQ_DEFAULT_PASS: admin
      RABBITMQ_ERLANG_COOKIE: secret-cookie  # 클러스터 구성 시 모든 노드 동일해야 함
    volumes:
      - ./rabbitmq.conf:/etc/rabbitmq/rabbitmq.conf
      - rabbitmq-data:/var/lib/rabbitmq
    healthcheck:
      test: ["CMD", "rabbitmq-diagnostics", "check_port_connectivity"]
      interval: 10s
      timeout: 5s
      retries: 5

  # 클러스터 실험용 노드 (단일 노드 실험 시 제거)
  rabbitmq-node2:
    image: rabbitmq:3.12-management
    hostname: rabbitmq-node2
    environment:
      RABBITMQ_DEFAULT_USER: admin
      RABBITMQ_DEFAULT_PASS: admin
      RABBITMQ_ERLANG_COOKIE: secret-cookie  # 메인 노드와 반드시 동일
    depends_on:
      rabbitmq:
        condition: service_healthy
    # node1 클러스터에 참가 (컨테이너 기동 후 수동 실행 또는 entrypoint에 추가)
    # docker exec rabbitmq-node2 rabbitmqctl stop_app
    # docker exec rabbitmq-node2 rabbitmqctl join_cluster rabbit@rabbitmq-node1
    # docker exec rabbitmq-node2 rabbitmqctl start_app

  spring-app:
    build: .
    environment:
      SPRING_RABBITMQ_HOST: rabbitmq
      SPRING_RABBITMQ_PORT: 5672
      SPRING_RABBITMQ_USERNAME: admin
      SPRING_RABBITMQ_PASSWORD: admin
    depends_on:
      rabbitmq:
        condition: service_healthy

volumes:
  rabbitmq-data:
```

> 💡 **Prometheus 메트릭** — `kbudde/rabbitmq-exporter` 같은 외부 exporter 대신, RabbitMQ 3.8+에 내장된 `rabbitmq_prometheus` 플러그인을 사용합니다. `rabbitmq.conf`에 한 줄만 추가하면 `http://localhost:15692/metrics`에서 바로 수집할 수 있습니다.
>
> ```ini
> # rabbitmq.conf
> prometheus.tcp.port = 15692
> ```

```bash
# Management UI 접속
open http://localhost:15672  # admin / admin

# 실험용 공통 명령어 세트

# Exchange 목록 확인
rabbitmqctl list_exchanges

# Queue 상태 확인 (메시지 수, Consumer 수)
rabbitmqctl list_queues name messages consumers

# Binding 목록 확인
rabbitmqctl list_bindings

# Connection / Channel 수 확인
rabbitmqctl list_connections
rabbitmqctl list_channels

# Unacknowledged 메시지 확인
rabbitmqctl list_queues name messages_unacknowledged

# Consumer 상세 정보 (Prefetch Count 등)
rabbitmqctl list_consumers

# 노드 상태 확인
rabbitmqctl status

# Quorum Queue 상태 확인
rabbitmqctl quorum_queue_status <queue-name>
```

---

## 📖 각 문서 구성 방식

모든 문서는 동일한 구조로 작성됩니다.

| 섹션 | 설명 |
|------|------|
| 🎯 **핵심 질문** | 이 문서를 읽고 나면 답할 수 있는 질문 |
| 🔍 **왜 이 개념이 실무에서 중요한가** | 실무에서 마주치는 문제 상황과 이 개념의 연결 |
| 😱 **흔한 실수** | Before — RabbitMQ 내부를 블랙박스로 둔 설정과 그 결과(메시지 유실 시나리오) |
| ✨ **올바른 접근** | After — 올바른 설정과 패턴 |
| 🔬 **내부 동작 원리** | AMQP 프로토콜, 라우팅 알고리즘, 메시지 흐름 다이어그램 |
| 💻 **실전 실험** | Spring AMQP + RabbitMQ Management UI 실험 |
| 📊 **Kafka vs RabbitMQ 비교** | 같은 문제를 어떻게 다르게 해결하는가 |
| ⚖️ **트레이드오프** | 이 설계의 장단점, 언제 다른 접근을 택할 것인가 |
| 📌 **핵심 정리** | 한 화면 요약 |
| 🤔 **생각해볼 문제** | 개념을 더 깊이 이해하기 위한 질문 + 해설 |

---

## 🗺️ 추천 학습 경로

<details>
<summary><b>🟢 "autoAck=true로 쓰다가 메시지 유실 경험" — 신뢰성 긴급 보강 (3일)</b></summary>

<br/>

```
Day 1  Ch3-01  메시지 유실의 세 지점 → 어디서 왜 사라지는지 파악
       Ch3-04  Consumer Ack 완전 분해 → autoAck vs 수동 Ack 차이
Day 2  Ch3-02  Publisher Confirm → Broker 수신 보장 방법
       Ch2-05  Dead Letter Exchange → 실패 메시지 보존 방법
Day 3  Ch3-05  재시도 전략 설계 → 무한 루프 없는 Exponential Backoff
       Ch7-03  Spring 에러 처리 → RetryTemplate + DLX 연동
```

</details>

<details>
<summary><b>🟡 "Exchange 없이 Queue에 직접 발행하고 있다" — 라우팅 구조 이해 (1주)</b></summary>

<br/>

```
Day 1  Ch1-02  AMQP 프로토콜 → Channel 멀티플렉싱 원리
       Ch1-03  Exchange 완전 분해 → Default Exchange 동작
Day 2  Ch2-01  Direct Exchange → 1:1, 1:N 라우팅
       Ch2-02  Topic Exchange → 와일드카드 패턴 설계
Day 3  Ch2-03  Fanout Exchange → Pub/Sub 구현
       Ch2-04  Headers Exchange → x-match 설정
Day 4  Ch2-05  Dead Letter Exchange → DLX + DLQ 구성
       Ch2-06  라우팅 패턴 설계 가이드 → 도메인 이벤트 설계
Day 5  Ch4-01  Work Queue → Prefetch로 공정 분배
       Ch4-02  Pub/Subscribe → 임시 Queue 전략
Day 6  Ch6-01  근본적 철학 차이 → Kafka와 비교 정리
       Ch6-02  사용 사례 비교 → 언제 어떤 선택인가
Day 7  Ch7-01  Spring AMQP 아키텍처 → RabbitTemplate 동작
       Ch7-04  Auto-configuration → application.yml 선언 방식
```

</details>

<details>
<summary><b>🔴 "AMQP 프로토콜부터 클러스터 운영까지 완전히 정복한다" — 전체 정복 (7주)</b></summary>

<br/>

```
1주차  Chapter 1 전체 — RabbitMQ 아키텍처와 AMQP 프로토콜
        → AMQP 프레임을 Wireshark로 캡처, Management UI로 Connection/Channel 모니터링

2주차  Chapter 2 전체 — Exchange 유형과 라우팅 패턴
        → 4가지 Exchange를 직접 생성하고 Routing Key 매칭 실험, DLX 체인 구성

3주차  Chapter 3 전체 — 메시지 신뢰성
        → autoAck=true 메시지 유실 재현, Publisher Confirm 비동기 처리 구현, Prefetch Count별 처리량 비교

4주차  Chapter 4 전체 — 실무 메시징 패턴
        → Work Queue 경쟁 소비자 실험, RPC Correlation ID 구현, Saga 보상 트랜잭션 설계

5주차  Chapter 5 전체 — 성능 튜닝과 운영
        → Flow Control 발동 시뮬레이션, Prometheus Exporter + Grafana 대시보드 구성

6주차  Chapter 6 전체 — RabbitMQ vs Kafka
        → 동일 시나리오를 두 시스템으로 구현하여 지연시간·처리량 비교 실험

7주차  Chapter 7 전체 — Spring AMQP 통합
        → 직렬화 방식별 메시지 크기 측정, RetryTemplate + DLX 전체 연동, Testcontainers 통합 테스트
```

</details>

---

## 🔗 연관 레포지토리

| 레포 | 주요 내용 | 연관 챕터 |
|------|----------|-----------|
| [kafka-deep-dive](https://github.com/dev-book-lab/kafka-deep-dive) | Kafka 파티션, Consumer Group, 오프셋 관리, 분산 로그 원리 | Ch6 전체(RabbitMQ vs Kafka 비교), Ch1-01(메시지 브로커 vs 분산 로그) |
| [network-deep-dive](https://github.com/dev-book-lab/network-deep-dive) | TCP 연결 수립, 흐름 제어, 소켓 | Ch1-02(AMQP의 TCP 기반 Channel 멀티플렉싱) |
| [spring-boot-internals](https://github.com/dev-book-lab/spring-boot-internals) | Auto-configuration, 조건부 Bean 등록 | Ch7-04(RabbitAutoConfiguration 동작 방식) |
| [observability-deep-dive](https://github.com/dev-book-lab/observability-deep-dive) | Prometheus, Grafana, 메트릭 수집 | Ch5-03(RabbitMQ Prometheus Exporter + 대시보드) |

> 💡 이 레포는 **RabbitMQ 내부 동작과 AMQP 프로토콜**에 집중합니다. Spring을 모르더라도 Chapter 1~6을 순수 RabbitMQ 관점으로 학습할 수 있습니다. kafka-deep-dive를 먼저 학습하면 Chapter 6의 비교 분석을 더 깊이 이해할 수 있습니다.

---

## 🙏 Reference

- [RabbitMQ 공식 문서](https://www.rabbitmq.com/documentation.html)
- [AMQP 0-9-1 프로토콜 스펙](https://www.amqp.org/)
- [RabbitMQ in Depth — Gavin Roy](https://www.manning.com/books/rabbitmq-in-depth)
- [Spring AMQP Reference](https://docs.spring.io/spring-amqp/docs/current/reference/html/)
- [CloudAMQP Blog](https://www.cloudamqp.com/blog/) — 실전 운영 사례
- [RabbitMQ 공식 블로그](https://blog.rabbitmq.com/) — 핵심 개발자 기술 아티클

---

<div align="center">

**⭐️ 도움이 되셨다면 Star를 눌러주세요!**

Made with ❤️ by [Dev Book Lab](https://github.com/dev-book-lab)

<br/>

**"메시지를 보내는 것과, Exchange가 어떤 Queue로 어떻게 라우팅하는지 아는 것은 다르다"**

</div>
