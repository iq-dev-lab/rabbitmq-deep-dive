# 함께 쓰는 경우 — 두 시스템의 역할을 명확히 나누는 아키텍처

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- RabbitMQ와 Kafka를 동시에 운영해야 하는 상황은 언제인가?
- 두 시스템의 책임을 어떻게 명확히 나누는가?
- RabbitMQ → Kafka, Kafka → RabbitMQ 데이터 흐름을 어떻게 설계하는가?
- 두 시스템 중복 운영의 비용과 이점은 무엇인가?
- 시작은 하나로 하고 나중에 하나를 추가하는 마이그레이션 전략은?

---

## 🔍 왜 이 개념이 실무에서 중요한가

실무에서 RabbitMQ 또는 Kafka 하나만 쓰는 것이 이상적이지만, 두 시스템이 각각 잘 해결하는 영역이 다르므로 함께 운영하는 사례가 많다. "RabbitMQ로 실시간 작업 처리, Kafka로 이벤트 로그 수집"처럼 각각의 장점을 살리는 아키텍처를 설계할 수 있다. 두 시스템 사이의 데이터 흐름을 명확히 정의하지 않으면 중복과 혼란이 생긴다.

---

## 😱 흔한 실수 (Before)

```
실수 1: 책임 구분 없이 두 시스템 혼용

  팀 A: RabbitMQ로 주문 이벤트 발행
  팀 B: Kafka에서도 주문 이벤트 발행 (별도로)
  
  결과:
    두 채널의 이벤트가 동기화 안 됨
    어떤 것이 진실의 소스(Source of Truth)인지 불명확
    이벤트 중복, 누락 발생

실수 2: 동일 이벤트를 두 시스템에 동시 발행 (이중 발행)

  OrderService:
    rabbitTemplate.convertAndSend("order.exchange", ...);
    kafkaTemplate.send("order-events", ...);
    // DB 트랜잭션과 원자적이지 않음
    // RabbitMQ 성공 + Kafka 실패 → 불일치

실수 3: 메시지 브리지 없이 두 시스템을 직접 연결

  "RabbitMQ의 메시지를 Kafka로 옮기고 싶다"
  직접 구현: RabbitMQ Consumer → Kafka Producer
  
  문제:
    Consumer 장애 시 메시지 중복 또는 손실
    트랜잭션 없는 At-Least-Once 보장 어려움
    성능/신뢰성 직접 튜닝 필요
```

---

## ✨ 올바른 접근 (After — 명확한 책임 분리)

```
두 시스템 역할 분리 원칙:

RabbitMQ 담당 (실시간 작업 처리):
  - 주문 처리 작업 큐 (결제, 재고, 배송)
  - 실시간 알림 발송 (Push 알림, 이메일, SMS)
  - 서비스 간 RPC
  - 우선순위 처리 (VIP 주문)

Kafka 담당 (이벤트 로그 + 분석):
  - 모든 비즈니스 이벤트의 영구 로그
  - 분석/BI 파이프라인 데이터 소스
  - 감사 로그 (Audit Log)
  - 이벤트 소싱 (Event Sourcing)

데이터 흐름:
  비즈니스 이벤트 발생 → Kafka (이벤트 로그)
                        → Kafka Consumer → RabbitMQ (실시간 처리)
  
  "Kafka가 이벤트 허브, RabbitMQ가 작업 실행기"
```

---

## 🔬 내부 동작 원리

### 1. 실제 아키텍처 패턴

```
=== 패턴 1: Kafka 허브 + RabbitMQ 작업 실행기 ===

[OrderService]
    │ order.placed 이벤트
    ▼
[Kafka: order-events Topic]
    │                        │
    ▼                        ▼
[Audit Consumer]      [Worker Bridge]
(감사 로그 저장)       RabbitMQ에 작업 발행
                            │
                            ▼
                  [RabbitMQ: payment.queue]
                            │
                            ▼
                   [PaymentService Worker]
                   (결제 처리 + DLX 재시도)

이 구조에서:
  Kafka: 이벤트의 진실 소스 (모든 이벤트 영구 보관)
  RabbitMQ: 실제 작업 처리 (DLX, Prefetch, 우선순위 등 활용)
  Worker Bridge: Kafka Consumer → RabbitMQ Publisher (변환 책임)

=== 패턴 2: 이벤트 게이트웨이 ===

외부 이벤트 (대용량): ─────→ [Kafka]
                                │
                          Consumer Bridge
                                │
내부 작업 처리:         ─────→ [RabbitMQ] → 각 서비스

흐름:
  초당 100만 건의 IoT 이벤트 → Kafka (대용량 수집)
  Kafka Consumer가 배치 처리 → RabbitMQ에 변환 발행
  RabbitMQ → 각 서비스에 Push (낮은 레이턴시 처리)

=== 패턴 3: 마이크로서비스 이벤트 버스 ===

서비스 간 실시간 통신:  [RabbitMQ Exchange]
이벤트 로그/분석:       [Kafka Topic]

RabbitMQ에서 발행 후 미러링:
  @RabbitListener(queues = "internal.events")
  public void mirrorToKafka(OrderEvent event) {
    kafkaTemplate.send("order-events-archive", event);  // Kafka에 복사
    // 원본 처리는 이미 RabbitMQ Consumer가 처리
  }
```

### 2. Kafka → RabbitMQ 브리지 구현

```
=== KafkaConsumer → RabbitMQ Publisher 브리지 ===

@Component
public class KafkaToRabbitBridge {

  @Autowired private RabbitTemplate rabbitTemplate;

  @KafkaListener(
    topics = "order-events",
    groupId = "rabbitmq-bridge-group",
    containerFactory = "kafkaListenerContainerFactory"
  )
  public void bridge(ConsumerRecord<String, OrderEvent> record,
      Acknowledgment ack) {

    OrderEvent event = record.value();

    try {
      // 이벤트 유형에 따라 RabbitMQ 적절한 Exchange/Key로 발행
      String routingKey = resolveRoutingKey(event);
      rabbitTemplate.convertAndSend("order.exchange", routingKey, event);

      // Publisher Confirm 받고 나서 Kafka 오프셋 커밋
      // (단순화를 위해 여기서는 즉시 커밋, 실제로는 ConfirmCallback 후 커밋)
      ack.acknowledge();

    } catch (AmqpException e) {
      log.error("RabbitMQ 발행 실패: {}", event.getOrderId(), e);
      // 재처리: Kafka 오프셋 커밋 안 함 → 자동 재시도
      // 또는 Dead Letter Topic으로 이동
      throw e;
    }
  }

  private String resolveRoutingKey(OrderEvent event) {
    return switch (event.getType()) {
      case "ORDER_PLACED"    -> "order.placed";
      case "PAYMENT_REQUESTED" -> "payment.requested";
      default -> "order.unknown";
    };
  }
}

=== 신뢰성 보장 브리지 (Outbox 패턴 활용) ===

원자성 문제:
  Kafka에서 읽기(오프셋 커밋) + RabbitMQ 발행 → 두 단계
  RabbitMQ 발행 실패 → Kafka 커밋 안 함 → 재처리 (중복 가능)
  RabbitMQ 발행 성공 → Kafka 커밋 실패 → 재처리 (중복 가능)

At-Least-Once 보장:
  Kafka: 오프셋 커밋 실패 → 재처리 (중복 발행 가능)
  RabbitMQ: Publisher Confirm + 멱등성 Consumer로 중복 처리

완전한 해결:
  브리지 서비스에 Outbox 테이블:
    1. Kafka 메시지 수신 → DB에 저장 (트랜잭션)
    2. 오프셋 커밋
    3. Outbox Publisher가 RabbitMQ로 발행 + Confirm
    → DB 트랜잭션으로 At-Least-Once 보장
```

### 3. RabbitMQ → Kafka 브리지 구현

```
=== RabbitMQ Consumer → Kafka Producer 브리지 ===

시나리오:
  RabbitMQ로 처리한 작업 이벤트를 Kafka에 아카이브

@RabbitListener(queues = "completed.events.queue")
public void archiveToKafka(ProcessedEvent event, Channel channel,
    @Header(AmqpHeaders.DELIVERY_TAG) long tag) throws IOException {

  try {
    // Kafka에 아카이브 발행
    SendResult<String, ProcessedEvent> result =
      kafkaTemplate.send("events-archive", event.getId().toString(), event).get();

    // Kafka 발행 성공 → RabbitMQ Ack
    channel.basicAck(tag, false);
    log.info("Kafka 아카이브 완료: {}", event.getId());

  } catch (Exception e) {
    log.error("Kafka 발행 실패: {}", event.getId(), e);
    // Kafka 실패 → RabbitMQ Nack → DLQ 또는 재큐
    channel.basicNack(tag, false, false);
  }
}

=== Kafka Connect를 이용한 브리지 (권장) ===

직접 코드 작성 대신 Kafka Connect 활용:
  RabbitMQ Source Connector:
    rabbitmq-source-connector.json:
    {
      "name": "rabbitmq-source",
      "config": {
        "connector.class": "com.ibm.eventstreams.connect.rabbitmqsource.RabbitMQSourceConnector",
        "kafka.topic": "events-archive",
        "rabbitmq.queue": "completed.events.queue",
        ...
      }
    }
  
  → Kafka Connect가 RabbitMQ → Kafka 자동 브리지
  → 별도 코드 없음, 장애 복구 내장
  → 단, Connector 플러그인 필요
```

### 4. 마이그레이션 전략

```
=== 시나리오: RabbitMQ → Kafka 이벤트 버스 전환 ===

현재: 모든 이벤트 RabbitMQ
목표: 이벤트 로그는 Kafka, 작업 처리는 RabbitMQ 유지

단계적 마이그레이션:

Phase 1: 듀얼 발행 (Dual Write)
  OrderService:
    기존 RabbitMQ 발행 유지
    새로 Kafka에도 동시 발행 (미러링)
  → 두 채널 모두 이벤트 수신
  → Kafka 소비자 점진적 전환

Phase 2: Kafka 소비자 전환
  각 팀이 RabbitMQ Queue 대신 Kafka Consumer Group으로 전환
  RabbitMQ: 발행만 유지 (소비자 없음)

Phase 3: RabbitMQ 이벤트 발행 제거
  OrderService에서 RabbitMQ 발행 코드 제거
  Kafka만 남음

Phase 4: RabbitMQ는 작업 처리 역할만 유지
  Kafka Consumer → RabbitMQ 작업 큐로 발행
  RabbitMQ: 작업 실행기 역할만

=== 시나리오: 새 프로젝트에서 어느 것부터 시작? ===

작은 팀 (5명 이하), 초기 MVP:
  → RabbitMQ로 시작
  이유: 낮은 운영 복잡도, 빠른 개발
       재처리 필요성이 생기면 Kafka 추가

처음부터 이벤트 중심 아키텍처:
  → Kafka로 시작
  이유: 나중에 추가하기 쉬운 Consumer Group
       이벤트 히스토리 자연스럽게 보존

작업 큐 + 이벤트 필요:
  → RabbitMQ + Kafka 처음부터 병행
  이유: 각 역할이 명확히 다름
       하나로 억지로 구현하면 복잡도 증가
```

---

## 💻 실전 실험

### 실험: 브리지 서비스 설정

```yaml
# application.yml: RabbitMQ + Kafka 동시 연결

spring:
  rabbitmq:
    host: rabbitmq
    port: 5672
    username: admin
    password: password
    listener:
      simple:
        acknowledge-mode: manual
        prefetch: 10

  kafka:
    bootstrap-servers: kafka:9092
    consumer:
      group-id: rabbitmq-bridge-group
      auto-offset-reset: earliest
      enable-auto-commit: false
    producer:
      acks: all
      retries: 3

# Kafka → RabbitMQ 브리지 활성화
bridge:
  kafka-to-rabbit:
    enabled: true
    kafka-topic: order-events
    rabbit-exchange: order.exchange
  rabbit-to-kafka:
    enabled: true
    rabbit-queue: completed.events.queue
    kafka-topic: events-archive
```

### 실험: 두 시스템 모니터링 통합

```yaml
# Prometheus에서 두 시스템 지표 통합 수집
scrape_configs:
  - job_name: 'rabbitmq'
    static_configs:
      - targets: ['rabbitmq:15692']

  - job_name: 'kafka'
    static_configs:
      - targets: ['kafka:9308']  # JMX Exporter

# Grafana 통합 대시보드
# - RabbitMQ Queue Depth
# - Kafka Consumer Lag
# - 브리지 처리량 (RabbitMQ Deliver Rate == Kafka Produce Rate여야 함)
# - 불일치 알림 (두 시스템 처리량 차이 > 10%)
```

---

## 📊 함께 쓰는 경우 아키텍처 비교

```
아키텍처 패턴:

패턴 1: Kafka 허브 + RabbitMQ 작업 실행기
  이벤트 발행: 모두 Kafka로
  작업 처리: Kafka Consumer → RabbitMQ → Worker
  적합: 대용량 이벤트 + 복잡한 작업 처리

패턴 2: RabbitMQ 실시간 + Kafka 아카이브
  실시간 처리: RabbitMQ
  처리 결과 아카이브: RabbitMQ → Kafka
  적합: 기존 RabbitMQ 유지 + 이벤트 히스토리 필요

패턴 3: 완전 분리
  서비스 간 실시간 통신: RabbitMQ
  이벤트 버스/분석: Kafka
  두 시스템 독립 운영 (직접 연결 없음)
  적합: 역할이 완전히 다를 때
```

---

## ⚖️ 트레이드오프

```
두 시스템 병행 운영 비용 vs 이점:

비용:
  운영 복잡도 2배 (두 시스템 모니터링, 장애 대응)
  학습 비용 (두 시스템 모두 이해 필요)
  인프라 비용 증가

이점:
  각 시스템의 장점 최대 활용
  작업 처리(RabbitMQ) + 이벤트 분석(Kafka) 최적화
  하나의 시스템으로 억지 구현 시 발생하는 복잡도 없음

하나로 유지하는 경우:
  RabbitMQ만: 재처리 불가, 대용량 한계
  Kafka만: RPC/우선순위/복잡 라우팅 어려움

결론:
  규모가 작으면 하나로 유지 (단순성 우선)
  규모가 크고 요구사항이 복잡하면 병행이 더 단순
```

---

## 📌 핵심 정리

```
두 시스템 함께 쓰는 경우 핵심:

RabbitMQ 역할:
  실시간 작업 처리 (결제, 재고, 배송)
  낮은 레이턴시 알림
  복잡한 라우팅
  DLX 기반 재처리

Kafka 역할:
  이벤트 로그 (영구 보관)
  분석/BI 데이터 소스
  이벤트 소싱
  대용량 스트리밍

브리지:
  Kafka → RabbitMQ: Consumer → Publisher (or Kafka Connect)
  RabbitMQ → Kafka: Consumer → Producer (아카이브)
  At-Least-Once: Outbox + 멱등성 Consumer

마이그레이션:
  듀얼 발행 → 점진적 전환 → 이전 시스템 제거

결론:
  역할을 명확히 나누면 두 시스템 병행이 오히려 단순
  경계가 모호하면 중복과 혼란
```

---

## 🤔 생각해볼 문제

**Q1.** RabbitMQ에서 Kafka로 이벤트를 브리지할 때 "정확히 한 번(Exactly-Once) 전달"을 보장할 수 있는가?

<details>
<summary>해설 보기</summary>

**일반적으로 매우 어렵습니다.** At-Least-Once + Consumer 멱등성이 현실적 해결책입니다.

Exactly-Once 달성이 어려운 이유:
1. RabbitMQ Ack 후 Kafka Producer 전송 사이에 브리지가 죽으면 → 재시작 시 재처리 → Kafka 중복
2. Kafka Producer 성공 후 RabbitMQ Ack 전에 브리지가 죽으면 → RabbitMQ 재전달 → Kafka 중복

**Kafka Transactional Producer 활용:**
```java
@Transactional("kafkaTransactionManager")
public void bridgeWithTransaction(OrderEvent event, Channel channel, long tag) throws Exception {
  kafkaTemplate.send("order-events", event);  // Kafka 트랜잭션 내
  channel.basicAck(tag, false);               // RabbitMQ Ack
}
```
이것도 RabbitMQ Ack와 Kafka 트랜잭션이 원자적이지 않아 Exactly-Once 미보장.

**현실적 접근: At-Least-Once + Kafka Consumer 멱등성**
- 브리지: At-Least-Once (중복 발행 가능)
- Kafka Consumer: eventId로 중복 처리 감지 → skip
- 결과: 사실상 Exactly-Once 효과

</details>

---

**Q2.** "우리 팀은 이미 RabbitMQ를 잘 쓰고 있다. Kafka가 필요한 요구사항이 생겼을 때 어떤 기준으로 추가 도입을 결정해야 하는가?"

<details>
<summary>해설 보기</summary>

**추가 도입 결정 체크리스트:**

추가 도입을 정당화하는 요구사항:
- [ ] 처리 후 메시지 재처리가 반드시 필요하다
- [ ] 새 서비스가 과거 이벤트부터 읽어야 한다
- [ ] 초당 수십만 건 이상의 처리량이 필요하다
- [ ] 여러 팀이 동일 이벤트를 독립적으로 소비해야 한다
- [ ] 이벤트 소싱 아키텍처를 도입한다
- [ ] 실시간 데이터 파이프라인(Kafka Streams, Flink)이 필요하다

추가 도입을 미뤄도 되는 경우:
- 위 조건 중 해당사항 없음
- 재처리: DB에서 재발행으로 충분히 해결 가능
- 처리량: 현재 RabbitMQ로 충분 (CPU, 메모리 여유 있음)
- 팀이 Kafka 운영 경험 없음 (운영 부담 > 이점)

**도입 비용 고려:**
- 학습 비용: Kafka 개념 + Kafka Streams + 운영 도구
- 인프라 비용: Kafka 클러스터 (ZooKeeper or KRaft) 구성
- 운영 비용: 파티션 관리, Consumer Lag 모니터링 추가

조언: 하나의 명확한 요구사항이 Kafka를 정당화하면 도입. 막연히 "나중에 필요할 수도"는 이유가 되지 않습니다.

</details>

---

**Q3.** Kafka와 RabbitMQ를 함께 운영하는 팀에서 새 서비스가 특정 이벤트를 구독하고 싶다. 어느 시스템을 구독해야 하는가?

<details>
<summary>해설 보기</summary>

**구독 목적에 따라 다릅니다.**

**RabbitMQ를 구독하는 경우:**
- "이 이벤트가 발생하면 즉시 처리해야 한다" (실시간 작업)
- 처리 실패 시 DLX로 재처리 필요
- 우선순위에 따라 처리 순서가 달라야 한다
- 예: 결제 완료 → 배송 준비 시작 (즉시 처리)

**Kafka를 구독하는 경우:**
- "과거 이벤트부터 처음부터 읽어야 한다"
- "처리 완료 후에도 이 이벤트를 다른 용도로 재소비할 수 있어야 한다"
- 분석, 리포트, 데이터 적재 목적
- 예: 주문 이벤트 분석 → 월별 통계 생성

**모호한 경우의 판단 기준:**
1. 처리 속도가 중요한가? → RabbitMQ
2. 이벤트 보관/재처리가 중요한가? → Kafka
3. 다른 서비스와 같은 이벤트를 독립적으로 소비하는가? → Kafka (Consumer Group)

아키텍처 문서화 권장: "어떤 이벤트가 RabbitMQ에 있고 어떤 것이 Kafka에 있는지" 명확히 문서화. 혼란의 근본 원인은 대부분 이 경계가 불명확한 것.

</details>

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[이전: 성능 특성 비교 ⬅️](./03-performance-comparison.md)** | **[다음: Chapter 7 — Spring AMQP 아키텍처 ➡️](../spring-amqp/01-spring-amqp-architecture.md)**

</div>
