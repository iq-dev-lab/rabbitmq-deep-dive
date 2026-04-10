# Dead Letter Exchange(DLX) — 실패한 메시지를 잡아내는 안전망

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 메시지가 "Dead Letter"가 되는 정확한 3가지 조건은 무엇인가?
- Dead Letter Exchange는 어떻게 설정하고, 실패 메시지는 어떤 경로로 이동하는가?
- Dead Letter Queue로 이동한 메시지의 헤더에는 어떤 정보가 추가되는가?
- DLX 없이 처리 실패 시 메시지는 어디로 가는가?
- Dead Letter Queue에 쌓인 메시지를 재처리하는 전략은 무엇인가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

Consumer 처리 실패는 반드시 발생한다. DB 연결 오류, 비즈니스 로직 예외, 잘못된 메시지 형식 등 원인은 다양하다. DLX 없이 처리 실패한 메시지는 두 가지 운명 중 하나다: `requeue=true`로 무한 루프 재시도, 또는 `requeue=false`로 조용히 영구 폐기. DLX는 세 번째 선택지를 제공한다 — "실패한 메시지를 버리지 않고, 별도 Queue에 보관하여 나중에 검토/재처리한다".

---

## 😱 흔한 실수 (Before — DLX 없을 때의 상황)

```
실수 1: DLX 없이 requeue=false → 메시지 영구 소실

  @RabbitListener(queues = "order.queue")
  public void process(OrderEvent event) {
    try {
      orderService.process(event);
    } catch (Exception e) {
      // 처리 실패 → basicNack(requeue=false) 또는 예외 throw
      throw new AmqpRejectAndDontRequeueException(e);
    }
  }
  
  결과:
    DLX 설정 없음 → 실패 메시지 폐기
    어떤 메시지가 실패했는지 알 방법 없음
    재처리 불가, 감사 추적 불가
    "주문이 처리 안 됐는데 메시지가 어디 갔는지 모른다"

실수 2: DLX 없이 requeue=true → 무한 루프

  throw new RuntimeException("처리 실패");  // requeue=true로 재처리됨
  
  결과:
    실패 메시지 즉시 Queue 앞으로 복귀
    같은 메시지를 초당 수천 번 재시도
    CPU 과부하, 다른 정상 메시지 처리 차단
    "서비스가 갑자기 CPU 100%가 됐다"

실수 3: DLX 설정했지만 Dead Letter Queue가 없음

  Queue 설정: x-dead-letter-exchange=dlx.exchange
  dlx.exchange는 선언했지만 Queue를 Binding하지 않음
  
  결과:
    Dead Letter → dlx.exchange로 이동
    dlx.exchange에 Binding된 Queue 없음
    → 메시지 폐기 (Exchange에 Binding 없으므로)
    DLX 설정의 의미가 없어짐
```

---

## ✨ 올바른 접근 (After — DLX 완전한 설정)

```
DLX 완전한 설정:

1. Dead Letter Queue 생성
   rabbitmqadmin declare queue name=order.dlq durable=true

2. Dead Letter Exchange 생성
   rabbitmqadmin declare exchange name=dlx.exchange type=direct durable=true

3. DLX에 DLQ Binding
   rabbitmqadmin declare binding \
     source=dlx.exchange destination=order.dlq routing_key=order.queue

4. 원본 Queue에 DLX 설정
   rabbitmqadmin declare queue name=order.queue durable=true \
     arguments='{"x-dead-letter-exchange":"dlx.exchange","x-dead-letter-routing-key":"order.queue"}'

Spring AMQP 설정:
  @Bean
  public Queue orderQueue() {
    return QueueBuilder.durable("order.queue")
      .deadLetterExchange("dlx.exchange")        // DLX 지정
      .deadLetterRoutingKey("order.queue")        // DLQ로 라우팅할 Key
      .build();
  }

  @Bean
  public DirectExchange dlxExchange() {
    return ExchangeBuilder.directExchange("dlx.exchange").durable(true).build();
  }

  @Bean
  public Queue deadLetterQueue() {
    return QueueBuilder.durable("order.dlq").build();
  }

  @Bean
  public Binding dlqBinding() {
    return BindingBuilder.bind(deadLetterQueue()).to(dlxExchange()).with("order.queue");
  }
```

---

## 🔬 내부 동작 원리

### 1. Dead Letter 발생 3가지 조건

```
=== 조건 1: Consumer Nack (requeue=false) ===

  Consumer가 basicNack(deliveryTag, multiple=false, requeue=false) 호출
  또는 basicReject(deliveryTag, requeue=false) 호출
  또는 Spring AMQP에서 AmqpRejectAndDontRequeueException throw

  흐름:
  order.queue → Consumer → 처리 실패
    → basicNack(requeue=false)
    → RabbitMQ: "이 메시지를 Queue로 돌려보내지 않음"
    → x-dead-letter-exchange 설정 있음?
       Yes → dlx.exchange로 전달 (Dead Letter 처리)
       No  → 메시지 폐기

=== 조건 2: TTL 만료 (x-message-ttl) ===

  Queue 설정:
    x-message-ttl: 60000 (60초)
    x-dead-letter-exchange: dlx.exchange
  
  흐름:
  메시지가 Queue에 60초 이상 머물렀을 때
  → TTL 만료 시점에 Dead Letter로 처리
  → dlx.exchange로 이동
  
  활용:
    지연 처리 패턴 (TTL + DLX = Delayed Queue)
    SLA 위반 감지 (60초 내 처리 안 되면 알림)

=== 조건 3: Queue 가득 참 (x-max-length) ===

  Queue 설정:
    x-max-length: 1000  (최대 1000개)
    x-dead-letter-exchange: dlx.exchange
    x-overflow: reject-publish-dlx  (초과분을 DLX로)
  
  흐름:
  Queue가 1000개 차 있을 때 1001번째 메시지 도착
  → x-overflow=reject-publish-dlx 설정 시 DLX로 전달
  → x-overflow=drop-head 설정 시 가장 오래된 메시지가 DLX로

=== Dead Letter 이동 시 추가되는 헤더 ===

Dead Letter로 이동한 메시지의 x-death 헤더:
  {
    "x-death": [
      {
        "queue": "order.queue",            // 원본 Queue 이름
        "reason": "rejected",             // dead-letter 이유 (rejected/expired/maxlen)
        "exchange": "order.exchange",      // 원본 발행 Exchange
        "routing-keys": ["order.placed"], // 원본 Routing Key
        "count": 1,                        // 이 Queue에서 dead-letter된 횟수
        "time": "2024-03-15T10:30:00"    // dead-letter 발생 시각
      }
    ],
    "x-first-death-queue": "order.queue",
    "x-first-death-reason": "rejected",
    "x-first-death-exchange": "order.exchange"
  }

활용:
  재처리 시 x-death[0].reason으로 실패 원인 분석
  x-death[0].count로 재시도 횟수 파악
  x-death[0].time으로 실패 시각 기록
```

### 2. DLX 라우팅 경로

```
=== DLX 처리 전체 흐름 ===

발행:
  Producer → order.exchange → order.queue

Consumer 처리 실패:
  order.queue → Consumer A
    → 예외 발생
    → basicNack(requeue=false)

Dead Letter 처리:
  RabbitMQ:
    1. order.queue의 x-dead-letter-exchange 확인: "dlx.exchange"
    2. x-dead-letter-routing-key 확인: "order.queue"
       (미설정 시 원본 Routing Key 사용)
    3. dlx.exchange로 메시지 전달 (routing_key="order.queue")
    4. dlx.exchange Binding 평가:
       "order.queue" → order.dlq ✅
    5. order.dlq에 저장

결과:
  order.dlq에 실패 메시지 + x-death 헤더 포함

=== Dead Letter Routing Key 설계 ===

옵션 1: x-dead-letter-routing-key 명시적 설정
  여러 Queue의 Dead Letter를 하나의 DLX + 각각의 DLQ로 라우팅
  order.queue → DLX → (key="order.queue") → order.dlq
  payment.queue → DLX → (key="payment.queue") → payment.dlq

옵션 2: x-dead-letter-routing-key 미설정
  원본 Routing Key 유지
  order.queue로 라우팅된 원본 Routing Key를 DLX로 전달
  → DLX에서 원본 Routing Key 기반 라우팅

=== DLX 체인 — 재시도 패턴 ===

Exponential Backoff 구현:
  order.queue (처리 실패)
    → wait.queue.1s (TTL=1초, DLX=main.exchange)
      → 1초 후 Dead Letter → main.exchange → order.queue (재시도)
    → 또 실패 → wait.queue.10s (TTL=10초, DLX=main.exchange)
      → 10초 후 Dead Letter → main.exchange → order.queue
    → 또 실패 → wait.queue.60s (TTL=60초)
    → 최종 실패 → permanent.dlq (수동 처리)

이 패턴은 Ch3-05 재시도 전략에서 상세히 다룹니다.
```

### 3. DLQ 모니터링과 재처리

```
=== DLQ 모니터링 ===

핵심 지표:
  order.dlq의 메시지 수 (정상 = 0, 증가 = 문제)
  DLQ 증가 속도 (갑자기 급증 = Consumer 장애)
  x-death.reason 분포 (rejected vs expired vs maxlen)

Prometheus Exporter 활용:
  rabbitmq_queue_messages{queue="order.dlq"} > 0
  → 알림 발생

Management UI:
  Queues → order.dlq 클릭
  Messages 탭에서 메시지 내용 직접 확인
  x-death 헤더에서 실패 원인 파악

=== DLQ 재처리 전략 ===

전략 1: 수동 재처리
  DLQ 메시지 꺼내기 → 원인 수정 → 원본 Queue 재발행

전략 2: 자동 재처리 Consumer
  @RabbitListener(queues = "order.dlq")
  public void processDlq(Message message) {
    // x-death 헤더 분석
    Map<String, Object> xDeath = (Map) message.getMessageProperties()
      .getHeaders().get("x-death");
    
    String reason = (String) ((List<Map>) xDeath).get(0).get("reason");
    Long count = (Long) ((List<Map>) xDeath).get(0).get("count");
    
    if (count >= 3) {
      // 3회 이상 실패 → 수동 검토 큐로 이동 또는 알림
      alertService.sendFailureAlert(message);
      return;
    }
    
    // 원인 수정 후 재발행
    rabbitTemplate.send("order.exchange", "order.placed",
      fixAndReconvert(message));
  }

전략 3: shovel로 자동 재처리
  RabbitMQ Shovel Plugin:
  order.dlq → (일정 시간 후) → order.queue
  → 수동 개입 없이 자동 재시도
  (무한 루프 방지를 위해 재시도 횟수 제한 필요)
```

---

## 💻 실전 실험

### 실험 1: DLX 완전한 설정과 동작 확인

```bash
# 1. Dead Letter Queue 생성
rabbitmqadmin declare queue name=order.dlq durable=true

# 2. Dead Letter Exchange 생성
rabbitmqadmin declare exchange name=dlx.exchange type=direct durable=true

# 3. DLX에 DLQ Binding
rabbitmqadmin declare binding source=dlx.exchange \
  destination=order.dlq routing_key=order.queue

# 4. 원본 Queue (DLX 설정 포함)
rabbitmqadmin declare queue name=order.queue durable=true \
  arguments='{"x-dead-letter-exchange":"dlx.exchange","x-dead-letter-routing-key":"order.queue"}'

# 5. 메시지 발행
rabbitmqadmin declare exchange name=order.exchange type=direct durable=true
rabbitmqadmin declare binding source=order.exchange \
  destination=order.queue routing_key=order
rabbitmqadmin publish exchange=order.exchange routing_key=order \
  payload='{"orderId":1,"amount":50000}'

# 6. Consumer가 Nack(requeue=false)할 때 시뮬레이션
# Management UI → Queues → order.queue → Get Messages → Nack(requeue=false)

# 7. DLQ 확인
rabbitmqctl list_queues name messages
# order.dlq  1  ← 실패 메시지 도착

# 8. DLQ 메시지 내용 확인 (x-death 헤더 포함)
rabbitmqadmin get queue=order.dlq ackmode=ack_requeue_true
# x-death 헤더에서 실패 원인 확인
```

### 실험 2: TTL Dead Letter 동작 확인

```bash
# TTL + DLX 설정
rabbitmqadmin declare queue name=ttl.queue durable=true \
  arguments='{"x-message-ttl":5000,"x-dead-letter-exchange":"dlx.exchange","x-dead-letter-routing-key":"ttl.queue"}'

rabbitmqadmin declare binding source=dlx.exchange \
  destination=order.dlq routing_key=ttl.queue

# 메시지 발행
rabbitmqadmin declare exchange name=test.exchange type=direct durable=true
rabbitmqadmin declare binding source=test.exchange \
  destination=ttl.queue routing_key=ttl
rabbitmqadmin publish exchange=test.exchange routing_key=ttl \
  payload='{"test":"ttl message"}'

# 5초 대기
sleep 5

# TTL 만료 후 DLQ 확인
rabbitmqctl list_queues name messages
# order.dlq  1  ← TTL 만료로 Dead Letter 처리
# ttl.queue  0  ← 메시지 없음 (DLQ로 이동)
```

### 실험 3: DLQ 모니터링 알림 설정

```yaml
# Prometheus Alert Rule
groups:
  - name: rabbitmq
    rules:
      - alert: DeadLetterQueueAlert
        expr: rabbitmq_queue_messages{queue=~".*dlq.*"} > 0
        for: 1m
        labels:
          severity: warning
        annotations:
          summary: "DLQ에 메시지 발생: {{ $labels.queue }}"
          description: "{{ $value }}개 메시지가 처리 실패하여 DLQ에 쌓임"
```

---

## 📊 Kafka vs RabbitMQ 비교

```
=== 처리 실패 메시지 처리 비교 ===

RabbitMQ DLX:
  Consumer 처리 실패 → DLX → Dead Letter Queue
  DLQ는 별도 Consumer가 분석/재처리
  브로커 레벨에서 처리 (Consumer 코드 변경 없음)
  
  x-death 헤더: 실패 원인, 횟수, 시각 자동 기록

Kafka Dead Letter Topic:
  Spring Kafka: DeadLetterPublishingRecoverer 활용
  Consumer 처리 실패 → Dead Letter Topic에 발행
  Dead Letter Topic: 별도 Consumer가 처리
  
  Spring Kafka 설정:
  @Bean
  public DefaultErrorHandler errorHandler(KafkaTemplate<String, Object> template) {
    return new DefaultErrorHandler(
      new DeadLetterPublishingRecoverer(template),
      new FixedBackOff(1000L, 3)  // 3회 재시도
    );
  }
  
  차이: Kafka DLT는 애플리케이션 레벨 (Consumer가 재발행)
       RabbitMQ DLX는 브로커 레벨 (RabbitMQ가 처리)

=== 재처리 유연성 ===

RabbitMQ DLQ:
  DLQ Consumer가 원본 Queue로 재발행
  과거 메시지 소급 불가 (처리 완료된 메시지 삭제됨)

Kafka Dead Letter Topic:
  오프셋 리셋으로 과거 실패 메시지 일괄 재처리 가능
  → 코드 수정 후 과거 실패 메시지 모두 재처리
  → 데이터 재처리에 Kafka가 유리
```

---

## ⚖️ 트레이드오프

```
DLX 사용의 장단점:

장점:
  ① 실패 메시지 보존 (폐기 대신 DLQ 보관)
  ② 원인 분석 가능 (x-death 헤더)
  ③ 재처리 기회 제공
  ④ Consumer 코드 변경 없이 브로커 레벨 설정

단점:
  ① 추가 Queue/Exchange 관리 필요
  ② DLQ 모니터링 누락 시 문제 파악 어려움
  ③ DLQ 메시지 무한 축적 → 디스크 압박

DLQ 없이 운영 가능한 경우:
  메시지 유실이 허용되는 경우 (로그, 비필수 알림)
  Consumer가 완벽히 멱등적이고 재시도가 안전한 경우
  → 그래도 중요한 서비스는 DLX 권장
```

---

## 📌 핵심 정리

```
Dead Letter Exchange 핵심:

Dead Letter 발생 조건:
  1. basicNack(requeue=false) — Consumer 거절
  2. x-message-ttl 만료 — Queue에 너무 오래 있음
  3. x-max-length 초과 — Queue 가득 참

DLX 설정 (원본 Queue):
  x-dead-letter-exchange: "dlx.exchange"  (필수)
  x-dead-letter-routing-key: "key"        (미설정 시 원본 Key 사용)

완전한 DLX 설정 체크리스트:
  ✅ Dead Letter Exchange 선언
  ✅ Dead Letter Queue 선언 (durable=true)
  ✅ DLX → DLQ Binding
  ✅ 원본 Queue에 DLX 설정
  ✅ DLQ 메시지 수 모니터링 알림

x-death 헤더:
  원본 Queue, 실패 이유(rejected/expired/maxlen), 횟수, 시각 자동 기록

재처리:
  DLQ Consumer → x-death 분석 → 원인 수정 → 원본 Exchange 재발행
```

---

## 🤔 생각해볼 문제

**Q1.** DLQ Consumer가 DLQ 메시지를 처리하다 또 실패했다. DLQ에도 DLX를 설정해야 하는가?

<details>
<summary>해설 보기</summary>

**상황에 따라 다릅니다.** DLQ에도 DLX를 설정하면 "DLQ의 DLQ"(DDLQ)가 생성됩니다.

일반적인 권장 설계:
- **DLQ에는 DLX를 설정하지 않는 것이 일반적**
- DLQ는 "최종 실패 메시지 보관소"이므로 더 이상 자동 처리하지 않음
- DLQ Consumer 실패 시 메시지를 다시 DLQ에 requeue(반환)

DLQ Consumer 처리 전략:
```java
@RabbitListener(queues = "order.dlq")
public void processDlq(Message msg) {
  try {
    // 분석 및 재처리 시도
  } catch (Exception e) {
    // 재처리도 실패 → DLQ에 그대로 두기 (requeue=true 또는 Nack 없음)
    // 또는 수동 알림 발송
    channel.basicNack(deliveryTag, false, true);  // requeue=true
  }
}
```

DLQ가 무한히 쌓이는 것을 막으려면:
- 재처리 횟수 제한 (x-death count 확인)
- 3회 이상 실패 → 개발자 알림 + 수동 처리 큐로 이동

</details>

---

**Q2.** TTL Dead Letter를 이용한 Delayed Queue 패턴 — "10초 후에 재처리하고 싶다"를 어떻게 구현하는가?

<details>
<summary>해설 보기</summary>

TTL + DLX로 Delayed Queue 구현:

```
처리 실패 → wait.queue (TTL=10000ms, DLX=main.exchange, DLK=order.queue)
  10초 후 TTL 만료 → DLX(main.exchange) → order.queue
  → Consumer가 다시 처리
```

설정:
```bash
# 10초 대기 Queue
rabbitmqadmin declare queue name=wait.10s durable=true \
  arguments='{"x-message-ttl":10000,"x-dead-letter-exchange":"main.exchange","x-dead-letter-routing-key":"order.queue"}'
```

처리 실패 시:
```java
// requeue=false + 재시도 Queue로 명시적 발행
channel.basicNack(deliveryTag, false, false);
rabbitTemplate.convertAndSend("", "wait.10s", failedMessage);  // 대기 Queue로
```

주의: wait.10s Queue는 Consumer가 없어야 함 (TTL 만료 후 DLX로 이동되어야 함)
      Consumer가 있으면 TTL 전에 소비해버림

이 패턴의 한계: 정확한 지연 시간 보장 어려움 (RabbitMQ 내부 TTL 처리 주기 영향)

</details>

---

**Q3.** 한 서비스에 30개의 Queue가 있고 모두 같은 DLQ를 사용한다. DLQ에서 메시지를 꺼낼 때 어느 Queue에서 온 메시지인지 어떻게 판별하는가?

<details>
<summary>해설 보기</summary>

`x-death` 헤더에 **원본 Queue 이름**이 포함됩니다:

```java
@RabbitListener(queues = "shared.dlq")
public void processDlq(Message message) {
  List<Map<String, Object>> xDeath = (List<Map<String, Object>>)
    message.getMessageProperties().getHeaders().get("x-death");
  
  if (xDeath != null && !xDeath.isEmpty()) {
    String originalQueue = (String) xDeath.get(0).get("queue");
    String reason = (String) xDeath.get(0).get("reason");
    // originalQueue: "order.queue", "payment.queue", ...
    
    switch (originalQueue) {
      case "order.queue":   reprocessOrder(message);   break;
      case "payment.queue": reprocessPayment(message); break;
      default: sendAlert(originalQueue, message);       break;
    }
  }
}
```

또한 `x-first-death-queue` 헤더(String)로 간편하게 조회 가능:
```java
String originalQueue = (String) message.getMessageProperties()
  .getHeaders().get("x-first-death-queue");
```

실무 권장: Queue별 전용 DLQ 운영이 더 명확합니다. 공유 DLQ는 관리가 편리하지만, DLQ Consumer가 복잡해지고 Queue별 DLQ 모니터링이 어렵습니다.

</details>

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[이전: Headers Exchange ⬅️](./04-headers-exchange.md)** | **[다음: 라우팅 패턴 설계 가이드 ➡️](./06-routing-design-guide.md)**

</div>
