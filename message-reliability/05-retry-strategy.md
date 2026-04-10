# 재시도 전략 설계 — 무한 루프 없는 안전한 재처리

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 즉시 `requeue=true`가 왜 무한 루프와 CPU 폭발을 만드는가?
- TTL + Dead Letter Exchange로 Exponential Backoff를 구현하는 방법은?
- 재시도 횟수를 제한하고, 최종 실패 메시지를 어떻게 처리하는가?
- Spring AMQP의 `RetryTemplate`은 어떻게 동작하고 한계는 무엇인가?
- 재시도와 DLQ의 전략적 조합은 어떻게 설계하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

처리 실패는 반드시 발생한다. DB가 일시적으로 응답하지 않거나, 외부 API가 5초 동안 다운되거나, 일시적 네트워크 오류가 생길 수 있다. 이때 즉시 재큐하면 같은 메시지를 초당 수천 번 처리 시도하며 CPU를 점령한다. Exponential Backoff는 첫 실패 후 1초, 2초, 4초, 8초... 점점 길게 대기하며 재시도한다. RabbitMQ에서 이를 구현하는 방법은 TTL + DLX 체인이다. 원리를 알아야 재시도 횟수 제한, 최종 DLQ 처리까지 완전하게 설계할 수 있다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: 즉시 requeue=true → 무한 루프

  } catch (Exception e) {
    channel.basicNack(tag, false, true);  // 즉시 재큐
  }

  결과:
    DB 연결 오류 → Nack(requeue=true) → 즉시 Queue 복귀
    같은 Consumer가 즉시 재수신 → 또 같은 오류
    초당 수천 번 반복 → CPU 100%
    다른 정상 메시지들이 이 메시지에 막혀 처리 안 됨
    Prefetch=1이면 Consumer 완전히 멈춤

실수 2: 무한 재시도 (횟수 제한 없음)

  // x-death count 확인 없이 무한 재시도
  @RabbitListener(queues = "retry.queue")
  public void retry(Message msg, Channel channel, long tag) {
    channel.basicNack(tag, false, true);  // 무한 반복
  }
  
  결과:
    메시지가 영원히 재처리됨
    수정 불가능한 오류 메시지가 영원히 시스템을 돌아다님

실수 3: Spring RetryTemplate만으로 해결 시도

  // Consumer 스레드 내에서 재시도
  RetryTemplate retry = new RetryTemplate();
  retry.setBackOffPolicy(new ExponentialBackOffPolicy(1000, 2.0, 30000));
  
  retry.execute(ctx -> {
    process(message);
    return null;
  });
  
  문제:
    재시도 중 스레드가 블로킹됨 (슬립 상태)
    Prefetch 슬롯 점유 (다른 메시지 처리 안 됨)
    브로커 Heartbeat 누락 → Connection 강제 종료 가능
    → RabbitMQ 레벨 Backoff가 Consumer 스레드 점유 문제 없음
```

---

## ✨ 올바른 접근 (After — TTL + DLX Exponential Backoff)

```
전체 재시도 아키텍처:

처리 실패 (일시적 오류)
    │
    ▼
Nack(requeue=false)
    │
    ▼ DLX → wait.1s.queue (TTL=1000ms, DLX=main.exchange)
    │        1초 후 TTL 만료 → main.exchange → order.queue
    │
    ▼ 2차 실패
    ▼ DLX → wait.10s.queue (TTL=10000ms, DLX=main.exchange)
    │        10초 후 TTL 만료 → main.exchange → order.queue
    │
    ▼ 3차 실패
    ▼ DLX → wait.60s.queue (TTL=60000ms, DLX=main.exchange)
    │        60초 후 TTL 만료 → main.exchange → order.queue
    │
    ▼ 4차 실패 (재시도 횟수 초과)
    ▼ 영구 DLQ (order.permanent.dlq) → 수동 처리

핵심 원리:
  TTL이 설정된 Queue는 Consumer 없음
  메시지가 TTL 동안 Queue에 머문 후 만료 → DLX로 이동
  DLX가 원본 Queue로 재라우팅 → Consumer에게 재전달
```

---

## 🔬 내부 동작 원리

### 1. 즉시 requeue가 무한 루프를 만드는 정확한 이유

```
=== 즉시 재큐의 타임라인 ===

Queue: [msg1]
Consumer A (Prefetch=1):

  T=0ms:  msg1 수신 (Unacked)
  T=1ms:  처리 시작 → DB 연결 오류 예외
  T=2ms:  basicNack(requeue=true)
  T=2ms:  msg1 → Queue 앞으로 복귀 (Ready)
  T=2ms:  Consumer A: Prefetch 슬롯 회복 → 즉시 msg1 재수신
  T=3ms:  처리 시작 → DB 연결 오류 예외 (DB 아직 다운)
  T=4ms:  basicNack(requeue=true)
  ...반복...

1초에 수백~수천 번 반복 가능:
  각 반복: DB 연결 시도 → 오류 → 스택 트레이스 로그 → Nack
  → CPU 폭발, 로그 폭발, 다른 메시지 차단

=== Prefetch와 무한 루프의 관계 ===

Prefetch=1:
  Consumer가 msg1을 처리하는 동안 새 메시지 안 받음
  msg1 Nack → 즉시 재수신 → 다른 msg2, msg3는 영원히 대기

Prefetch=10:
  10개 중 1개가 무한 루프 → 나머지 9개는 처리 가능
  하지만 재시도 메시지가 계속 Prefetch 슬롯 차지

결론: Prefetch 값에 무관하게 즉시 requeue는 위험
```

### 2. TTL + DLX Exponential Backoff 상세

```
=== Queue 구성 ===

1단계: 대기 Queue 생성 (Consumer 없음)
  wait.1s.queue:
    x-message-ttl: 1000           ← 1초 후 만료
    x-dead-letter-exchange: order.exchange  ← 만료 시 이동
    x-dead-letter-routing-key: order.placed ← 원본 Queue의 Routing Key
    durable: true

  wait.10s.queue:
    x-message-ttl: 10000          ← 10초 후 만료
    x-dead-letter-exchange: order.exchange
    x-dead-letter-routing-key: order.placed
    durable: true

  wait.60s.queue:
    x-message-ttl: 60000          ← 60초 후 만료
    ...

2단계: 원본 Queue DLX 설정
  order.queue:
    x-dead-letter-exchange: retry.dlx.exchange
    → 처리 실패 시 retry.dlx.exchange로 이동

3단계: retry.dlx.exchange Routing
  retry.dlx.exchange (Direct):
    Binding Key "retry.1"  → wait.1s.queue
    Binding Key "retry.2"  → wait.10s.queue
    Binding Key "retry.3"  → wait.60s.queue
    Binding Key "retry.dead" → order.permanent.dlq

=== 재시도 횟수 추적 ===

x-death 헤더 활용:
  Consumer에서 재시도 횟수 확인:
  
  List<Map<String, Object>> xDeath = message.getMessageProperties()
    .getXDeathHeader();
  
  long retryCount = xDeath == null ? 0 :
    xDeath.stream()
      .filter(d -> "order.queue".equals(d.get("queue")))
      .mapToLong(d -> (Long) d.get("count"))
      .sum();

재시도 횟수별 대기 Queue 선택:
  retryCount == 0 → wait.1s.queue  (처음 실패)
  retryCount == 1 → wait.10s.queue (두 번째 실패)
  retryCount == 2 → wait.60s.queue (세 번째 실패)
  retryCount >= 3 → order.permanent.dlq (최종 실패)

=== 전체 흐름 ===

order.queue → Consumer (처리 실패)
  → basicNack(requeue=false)
  → DLX: retry.dlx.exchange
  → Consumer가 설정한 Routing Key에 따라:
    retryCount=0 → Binding Key "retry.1" → wait.1s.queue
  
  wait.1s.queue: 1초 대기 (Consumer 없음)
  → TTL 만료 → DLX: order.exchange
  → Routing Key "order.placed" → order.queue
  
  order.queue → Consumer (2차 시도)
  → 또 실패 → retryCount=1 → wait.10s.queue
  → 10초 대기 → order.queue
  
  3차 시도 실패 → retryCount=2 → wait.60s.queue
  → 60초 대기 → order.queue
  
  4차 시도 실패 → retryCount=3 → order.permanent.dlq
  → 수동 처리
```

### 3. Spring AMQP 완전한 구현

```java
=== Queue/Exchange 설정 ===

@Configuration
public class RetryConfig {

  // 원본 Queue (DLX 설정)
  @Bean
  public Queue orderQueue() {
    return QueueBuilder.durable("order.queue")
      .deadLetterExchange("retry.dlx.exchange")
      .build();
  }

  // 재시도 DLX
  @Bean
  public DirectExchange retryDlxExchange() {
    return ExchangeBuilder.directExchange("retry.dlx.exchange").durable(true).build();
  }

  // 1초 대기 Queue
  @Bean
  public Queue wait1sQueue() {
    return QueueBuilder.durable("wait.1s.queue")
      .ttl(1_000)
      .deadLetterExchange("order.exchange")
      .deadLetterRoutingKey("order.placed")
      .build();
  }

  // 10초 대기 Queue
  @Bean
  public Queue wait10sQueue() {
    return QueueBuilder.durable("wait.10s.queue")
      .ttl(10_000)
      .deadLetterExchange("order.exchange")
      .deadLetterRoutingKey("order.placed")
      .build();
  }

  // 60초 대기 Queue
  @Bean
  public Queue wait60sQueue() {
    return QueueBuilder.durable("wait.60s.queue")
      .ttl(60_000)
      .deadLetterExchange("order.exchange")
      .deadLetterRoutingKey("order.placed")
      .build();
  }

  // 영구 DLQ
  @Bean
  public Queue permanentDlq() {
    return QueueBuilder.durable("order.permanent.dlq").build();
  }

  // Binding
  @Bean public Binding retry1Binding() {
    return BindingBuilder.bind(wait1sQueue()).to(retryDlxExchange()).with("retry.1");
  }
  @Bean public Binding retry2Binding() {
    return BindingBuilder.bind(wait10sQueue()).to(retryDlxExchange()).with("retry.2");
  }
  @Bean public Binding retry3Binding() {
    return BindingBuilder.bind(wait60sQueue()).to(retryDlxExchange()).with("retry.3");
  }
  @Bean public Binding retryDeadBinding() {
    return BindingBuilder.bind(permanentDlq()).to(retryDlxExchange()).with("retry.dead");
  }
}

=== Consumer 구현 ===

@RabbitListener(queues = "order.queue")
public void handle(Message message, Channel channel,
    @Header(AmqpHeaders.DELIVERY_TAG) long tag) throws IOException {

  long retryCount = getRetryCount(message);

  try {
    orderService.process(message);
    channel.basicAck(tag, false);

  } catch (BusinessException e) {
    // 재처리 불가 → 즉시 DLQ
    log.error("비즈니스 예외, 즉시 DLQ 이동: {}", e.getMessage());
    sendToRetryDlx(message, "retry.dead");
    channel.basicAck(tag, false);  // 원본 Queue에서 제거

  } catch (TransientException e) {
    // 일시적 오류 → Backoff 재시도
    String retryKey = getRetryKey(retryCount);
    log.warn("일시적 오류 (retryCount={}), {} 대기 후 재시도", retryCount, retryKey);
    sendToRetryDlx(message, retryKey);
    channel.basicAck(tag, false);  // 직접 재발행했으므로 원본 Ack
  }
}

private long getRetryCount(Message message) {
  List<Map<String, Object>> xDeath = message.getMessageProperties().getXDeathHeader();
  if (xDeath == null || xDeath.isEmpty()) return 0;
  return xDeath.stream()
    .filter(d -> "order.queue".equals(d.get("queue")))
    .mapToLong(d -> (Long) d.get("count"))
    .sum();
}

private String getRetryKey(long retryCount) {
  if (retryCount == 0) return "retry.1";   // 1초
  if (retryCount == 1) return "retry.2";   // 10초
  if (retryCount == 2) return "retry.3";   // 60초
  return "retry.dead";                      // 영구 DLQ
}

private void sendToRetryDlx(Message message, String routingKey) {
  rabbitTemplate.send("retry.dlx.exchange", routingKey, message);
}
```

### 4. Spring AMQP RetryTemplate 방식과 한계

```
=== RetryTemplate 방식 (Consumer 스레드 내 재시도) ===

설정:
  @Bean
  public SimpleRabbitListenerContainerFactory containerFactory() {
    SimpleRabbitListenerContainerFactory factory = new ...;
    factory.setAdviceChain(
      RetryInterceptorBuilder.stateless()
        .maxAttempts(3)
        .backOffOptions(1000, 2.0, 30000)  // 초기1초, 배수2, 최대30초
        .recoverer(new RejectAndDontRequeueRecoverer())  // 최종 실패 → DLQ
        .build()
    );
    return factory;
  }

동작:
  처리 실패 → 1초 sleep → 재시도 → 실패 → 2초 sleep → 재시도
  → 최종 실패 → AmqpRejectAndDontRequeueException → DLQ

=== RetryTemplate의 한계 ===

문제 1: Consumer 스레드 블로킹
  재시도 대기 중 Thread.sleep() 실행
  → 이 Consumer 스레드는 다른 메시지 처리 불가
  → Prefetch 슬롯 점유 지속
  → 대기 시간이 길면 (30초) Heartbeat 누락 위험

문제 2: 재시작 시 재시도 횟수 초기화
  Stateless RetryTemplate: 재시작 후 재시도 카운트 0으로 리셋
  → 실제 재시도 횟수보다 더 많이 재시도

Stateful RetryTemplate:
  재시도 상태를 메모리(또는 Redis)에 저장
  재시작 후에도 카운트 유지
  → 설정 복잡, 상태 저장소 관리 필요

=== 언제 RetryTemplate, 언제 TTL+DLX ===

RetryTemplate 적합:
  재시도 간격이 짧음 (수 초 이내)
  Consumer 스레드 블로킹이 허용됨 (처리량 여유)
  단순한 재시도 로직

TTL+DLX 적합:
  재시도 간격이 길거나 불확실 (수십 초~분)
  Consumer 스레드 블로킹 허용 안 됨 (고처리량)
  RabbitMQ 레벨의 신뢰성 필요 (재시작해도 재시도 상태 보존)
```

---

## 💻 실전 실험

### 실험 1: TTL 기반 Backoff Queue 구성

```bash
# Exchange 생성
rabbitmqadmin declare exchange name=order.exchange type=direct durable=true
rabbitmqadmin declare exchange name=retry.dlx.exchange type=direct durable=true

# 원본 Queue (DLX 설정)
rabbitmqadmin declare queue name=order.queue durable=true \
  arguments='{"x-dead-letter-exchange":"retry.dlx.exchange"}'

# 대기 Queue들
rabbitmqadmin declare queue name=wait.1s.queue durable=true \
  arguments='{"x-message-ttl":1000,"x-dead-letter-exchange":"order.exchange","x-dead-letter-routing-key":"order"}'

rabbitmqadmin declare queue name=wait.10s.queue durable=true \
  arguments='{"x-message-ttl":10000,"x-dead-letter-exchange":"order.exchange","x-dead-letter-routing-key":"order"}'

rabbitmqadmin declare queue name=order.permanent.dlq durable=true

# Binding
rabbitmqadmin declare binding source=order.exchange destination=order.queue routing_key=order
rabbitmqadmin declare binding source=retry.dlx.exchange destination=wait.1s.queue routing_key=retry.1
rabbitmqadmin declare binding source=retry.dlx.exchange destination=wait.10s.queue routing_key=retry.2
rabbitmqadmin declare binding source=retry.dlx.exchange destination=order.permanent.dlq routing_key=retry.dead

# 흐름 테스트
rabbitmqadmin publish exchange=order.exchange routing_key=order payload='{"orderId":1}'

# 처리 실패 시뮬레이션: order.queue에서 꺼내서 retry.dlx로 전달
# 1초 후 wait.1s.queue TTL 만료 → order.queue로 복귀 확인
sleep 2
rabbitmqctl list_queues name messages
# order.queue  1  ← 재전달됨
```

### 실험 2: x-death 헤더로 재시도 횟수 확인

```bash
# 재시도 후 메시지의 x-death 헤더 확인
rabbitmqadmin get queue=order.queue ackmode=ack_requeue_true

# Management UI → Queues → order.queue → Get Messages
# x-death 헤더 예시:
# [{
#   "queue": "order.queue",
#   "reason": "rejected",
#   "exchange": "order.exchange",
#   "routing-keys": ["order"],
#   "count": 2,              ← 2번째 재시도
#   "time": "2024-03-15T..."
# }]
```

---

## 📊 Kafka vs RabbitMQ 비교

```
=== 재시도 전략 비교 ===

RabbitMQ (TTL + DLX):
  재시도 로직이 브로커에 있음
  대기 Queue가 물리적으로 존재
  재시작 후에도 재시도 상태 보존 (메시지가 Queue에 있음)
  Consumer 스레드 블로킹 없음

Kafka (Spring Kafka RetryableTopic):
  @RetryableTopic(
    attempts = "4",
    backoff = @Backoff(delay = 1000, multiplier = 2.0, maxDelay = 60000)
  )
  → order-events-retry-0, order-events-retry-1, order-events-retry-2 Topic 자동 생성
  → 최종 실패 → order-events-dlt (Dead Letter Topic)
  
  재시도 로직이 Consumer 측에 있음 (Spring Kafka 라이브러리)
  재시도 Topic이 자동 생성됨

비교:
  RabbitMQ: 브로커 레벨 (Queue/Exchange 설정)
  Kafka:    Consumer 라이브러리 레벨 (별도 Topic 생성)
  
  재시작 후 재시도 상태:
    RabbitMQ: 대기 Queue에 메시지가 있으므로 자동 복구
    Kafka: RetryableTopic의 오프셋으로 관리 (자동 복구)
```

---

## ⚖️ 트레이드오프

```
재시도 전략 비교:

즉시 requeue=true:
  ✅ 구현 단순
  ❌ 무한 루프, CPU 폭발 위험
  → 실용적으로 사용 금지 (일시적 오류에도)

RetryTemplate (스레드 내 sleep):
  ✅ 구현 단순 (Spring AMQP 내장)
  ❌ Consumer 스레드 블로킹
  ❌ 재시작 시 재시도 카운트 초기화 (Stateless)
  → 짧은 재시도 간격, 낮은 처리량 환경에서 사용

TTL + DLX Backoff:
  ✅ Consumer 스레드 블로킹 없음
  ✅ 재시작 후 재시도 상태 보존
  ✅ 재시도 횟수 x-death로 추적 가능
  ❌ 구성 복잡 (Queue, Exchange, Binding 다수)
  → 프로덕션 권장

Exponential Backoff 간격:
  1초 → 10초 → 60초 → 영구DLQ (일반적)
  1초 → 5초 → 30초 → 영구DLQ (더 적극적)
  → 서비스 특성에 맞게 조정
```

---

## 📌 핵심 정리

```
재시도 전략 핵심:

즉시 requeue 금지:
  DB 오류 → Nack(requeue=true) → 즉시 루프 → CPU 폭발
  → 반드시 지연 재시도 사용

TTL + DLX Exponential Backoff:
  order.queue (실패) → retry.dlx.exchange
    → wait.1s.queue (TTL=1초) → order.queue (재시도 1)
    → wait.10s.queue (TTL=10초) → order.queue (재시도 2)
    → wait.60s.queue (TTL=60초) → order.queue (재시도 3)
    → order.permanent.dlq (최종 실패)

재시도 횟수 추적:
  x-death 헤더의 count 필드 사용
  count >= N → 영구 DLQ로 이동

예외 분류:
  일시적 오류 (DB 연결) → 지연 재시도
  영구적 오류 (메시지 형식) → 즉시 DLQ

RetryTemplate:
  짧은 재시도, 낮은 처리량에 적합
  긴 재시도 필요 시 TTL+DLX 권장
```

---

## 🤔 생각해볼 문제

**Q1.** 재시도 중 Consumer가 재시작됐다. TTL+DLX 방식과 RetryTemplate 방식에서 각각 재시도 상태는 어떻게 되는가?

<details>
<summary>해설 보기</summary>

**TTL+DLX 방식**: 재시작에 완전히 무관합니다.
- 메시지가 `wait.10s.queue`에 있다면 재시작 후에도 그 Queue에 그대로 있음
- TTL 만료 후 자동으로 원본 Queue로 복귀
- Consumer 재시작과 무관하게 재시도 흐름 유지

**RetryTemplate (Stateless) 방식**: 재시도 카운트가 초기화됩니다.
- Consumer 재시작 → 메모리의 재시도 카운트 소실
- 다시 0회부터 재시도 → 실제로 더 많이 재시도됨
- 최악의 경우 영구 DLQ로 이동하지 않고 계속 재시도

TTL+DLX가 신뢰성 측면에서 우월한 이유입니다. 메시지 자체가 Queue라는 영속적인 저장소에 있으므로, 어떤 방식의 재시작에도 재시도 상태가 보존됩니다.

</details>

---

**Q2.** 재시도 중 같은 메시지가 여러 Consumer 인스턴스에게 동시에 전달될 수 있는가? 어떻게 방지하는가?

<details>
<summary>해설 보기</summary>

**일반적으로는 동시에 하나의 Consumer에게만 전달됩니다.** RabbitMQ의 Queue는 Consumer 간 경쟁 소비로 동작하며, 하나의 메시지는 한 번에 하나의 Consumer에게만 전달됩니다.

**예외 상황**:
- Consumer A가 메시지를 받아 처리 중
- Consumer A의 Connection 끊김 → Unacked 메시지 Ready 복귀
- Consumer B가 같은 메시지 수신
- Consumer A도 (재연결 후) 같은 메시지를 다시 받을 수 있음
→ 동시는 아니지만 **중복 처리 가능**

방지 방법:
1. **Consumer 멱등성**: 메시지 ID를 DB에 저장, 중복 처리 시 skip
```java
if (messageRepository.existsById(messageId)) {
  channel.basicAck(tag, false);  // 중복 → 무시하고 Ack
  return;
}
// 처리
messageRepository.save(messageId);
```
2. **분산 Lock**: Redis 기반 Lock으로 중복 처리 방지
3. **Outbox Pattern**: DB 트랜잭션과 메시지 처리를 원자적으로

결론: 중복 전달은 피할 수 없으므로 **Consumer 멱등성**이 필수입니다.

</details>

---

**Q3.** wait.60s.queue의 TTL이 60초인데, Consumer가 재시작되어 order.queue가 소비 중단됐다. 60초 후 메시지가 order.queue로 복귀했을 때, Consumer가 없으면 메시지는 어떻게 되는가?

<details>
<summary>해설 보기</summary>

**메시지는 order.queue에 그대로 쌓입니다.** Consumer가 없어도 durable Queue에 메시지는 보존됩니다.

동작:
1. wait.60s.queue TTL 만료 → order.exchange → order.queue (메시지 도달)
2. order.queue에 Consumer 없음 → 메시지 Ready 상태로 대기
3. Consumer 재시작 후 order.queue 구독 → 쌓인 메시지 순서대로 소비

이것이 durable Queue + PERSISTENT 메시지의 핵심 가치입니다.

단, Consumer가 오래 없으면:
- order.queue에 재시도 메시지가 계속 쌓임
- Queue 메모리/디스크 사용량 증가
- 모니터링: `order.queue` 메시지 수 급증 → 알림 발동

실무 대응:
- Consumer 다운 → 최대한 빨리 재시작 (Kubernetes liveness probe)
- order.queue 메시지 수 임계값 알림 (Prometheus + Grafana)

</details>

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[이전: Consumer Acknowledgement ⬅️](./04-consumer-ack.md)** | **[다음: Prefetch Count 완전 분해 ➡️](./06-prefetch-count.md)**

</div>
