# 처리량 최적화 — 병목 진단과 복합적 튜닝 전략

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 처리량 병목을 Publisher 측과 Consumer 측으로 어떻게 구분하는가?
- Batch Publishing은 처리량에 얼마나 영향을 주는가?
- Publisher Confirm 비동기 처리가 동기 처리보다 처리량에서 유리한 이유는?
- 메시지 압축은 언제 도움이 되고 언제 오히려 해가 되는가?
- 복합적 튜닝(Prefetch + Concurrency + Batch)이 어떻게 상호작용하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

"RabbitMQ가 느리다"는 말은 대부분 "설정이 잘못됐다"는 뜻이다. Publisher Confirm을 동기 방식으로 매 메시지마다 대기하거나, Prefetch=1로 고처리량을 기대하거나, 압축 없이 큰 메시지를 발행하거나, Consumer 스레드가 1개인 상태로 처리량을 기대하면 이론적 한계에 한참 못 미친다. 처리량 최적화는 단일 요소가 아닌 여러 요소의 복합 조정이다.

---

## 😱 흔한 실수 (Before)

```
실수 1: 동기 Publisher Confirm (매 메시지 대기)

  for (Event event : events) {
    rabbitTemplate.convertAndSend(exchange, key, event);
    // 내부적으로 convertSendAndReceive와 혼용하거나
    // waitForConfirmsOrDie() 동기 대기
  }
  → 1000개 이벤트: 1000번 왕복 = 10~50초 소요

실수 2: 단일 스레드 Consumer로 고처리량 기대

  @RabbitListener(queues = "order.queue")  // 기본 concurrency=1
  public void handle(OrderEvent event) {
    // 처리 시간 50ms
  }
  → 초당 최대 20개 처리
  → Queue가 쌓이는 이유 파악 못 함

실수 3: 모든 메시지에 압축 적용

  // 1KB 메시지에 GZIP 압축
  byte[] compressed = gzip(message);  // 압축 결과 1.1KB (더 커짐!)
  // CPU 시간만 낭비
```

---

## ✨ 올바른 접근 (After)

```
처리량 최적화 조합:

Publisher 측:
  비동기 Publisher Confirm (ConfirmCallback)
  배치 발행 (여러 메시지를 연속 발행 후 일괄 Confirm)
  메시지 압축 (1KB 이상 대형 메시지에만)
  Connection 재사용 (매 요청 재연결 금지)

Consumer 측:
  concurrency 적정값 설정 (처리 시간 기반)
  Prefetch 적정값 설정 (지연 기반)
  배치 소비 (Spring AMQP BatchMessageListener)
  처리 로직 비동기화 (I/O 바운드 작업)
```

---

## 🔬 내부 동작 원리

### 1. Publisher 처리량 최적화

```
=== 동기 vs 비동기 Confirm 처리량 비교 ===

동기 Confirm (waitForConfirmsOrDie):
  T=0ms:    basicPublish(msg1)
  T=1ms:    waitForConfirmsOrDie(5000)  ← 대기
  T=11ms:   Ack 수신 (RTT 10ms 가정)
  T=11ms:   basicPublish(msg2)
  T=21ms:   Ack 수신
  ...
  1000개 × 11ms = 11초

비동기 Confirm (ConfirmCallback):
  T=0ms:    basicPublish(msg1)  → Confirm 대기 없이 즉시 다음
  T=0ms:    basicPublish(msg2)
  T=0ms:    basicPublish(msg3)
  ...
  T=5ms:    msg1 Ack 수신 (ConfirmCallback 비동기 호출)
  T=5ms:    msg2 Ack 수신
  ...
  1000개 × 0.1ms + RTT = ~100ms + 10ms = 0.11초
  → 동기 대비 100배 처리량

=== 배치 발행 최적화 ===

기본 발행 (1개씩):
  [basicPublish][basicPublish][basicPublish]...
  각 basicPublish마다 TCP 패킷 가능성 (소켓 write)

배치 발행 (연속 발행):
  channel.setConfirmCallback(...)
  for (Event e : batch) {
    channel.basicPublish(exchange, key, props, body);
    // Nagle 알고리즘: 소켓 버퍼에 쌓아 한 번에 전송
  }
  // 브로커: 한 번에 여러 메시지 처리 → 배치 Ack

TCP Nagle 알고리즘:
  소켓 버퍼가 찰 때까지 또는 일정 시간 후 한 번에 전송
  → 연속 basicPublish가 하나의 TCP 세그먼트로 전송
  → 네트워크 효율 향상

=== 메시지 압축 가이드 ===

압축 유리한 경우:
  메시지 크기 > 1KB (JSON, XML, 큰 페이로드)
  네트워크 대역폭이 병목인 경우
  CPU 여유 있고 네트워크가 느린 경우

압축 불리한 경우:
  메시지 크기 < 1KB (압축 오버헤드 > 절약량)
  CPU가 병목인 경우
  이미 압축된 데이터 (JPEG, ZIP 등 재압축 효과 없음)

Spring AMQP 압축 설정:
  @Bean
  public RabbitTemplate rabbitTemplate(ConnectionFactory cf) {
    RabbitTemplate template = new RabbitTemplate(cf);
    // MessageConverter에 압축 래핑
    GZipPostProcessor postProcessor = new GZipPostProcessor();
    template.setBeforePublishPostProcessors(postProcessor);
    return template;
  }

  Consumer 측 압축 해제:
  containerFactory.setAfterReceivePostProcessors(new GUnzipPostProcessor());
```

### 2. Consumer 처리량 최적화

```
=== Concurrency 최적값 계산 ===

단일 Consumer 스레드 처리량:
  처리 시간 50ms → 초당 20개

필요 처리량 초당 1000개:
  필요 스레드 = 1000 / 20 = 50개

Spring AMQP 설정:
  spring:
    rabbitmq:
      listener:
        simple:
          concurrency: 50
          max-concurrency: 80  # 피크 대응
          prefetch: 5          # 스레드당 5개 미리 수신

총 Prefetch = concurrency × prefetch = 50 × 5 = 250개

=== I/O 바운드 작업 처리량 개선 ===

문제: 처리 시간 50ms 중 45ms가 DB 조회 대기
  → 스레드가 대부분 대기 상태
  → concurrency 증가로 해결 (단, 스레드 오버헤드)

더 효율적인 방법: 비동기 처리
  @RabbitListener(queues = "order.queue")
  public void handle(OrderEvent event, Channel channel,
      @Header(AmqpHeaders.DELIVERY_TAG) long tag) {

    // DB 조회를 비동기로
    CompletableFuture<OrderDetails> future = orderRepository.findByIdAsync(event.getOrderId());

    future.whenComplete((details, ex) -> {
      try {
        if (ex != null) {
          channel.basicNack(tag, false, true);
        } else {
          processOrder(details);
          channel.basicAck(tag, false);
        }
      } catch (IOException e) { ... }
    });
  }
  // 주의: 비동기 Ack는 채널 스레드 안전성 주의 필요

=== 배치 소비 (Spring AMQP BatchMessageListener) ===

@Bean
public SimpleRabbitListenerContainerFactory batchFactory(ConnectionFactory cf) {
  SimpleRabbitListenerContainerFactory factory = new SimpleRabbitListenerContainerFactory();
  factory.setConnectionFactory(cf);
  factory.setBatchListener(true);    // 배치 소비 활성화
  factory.setConsumerBatchEnabled(true);
  factory.setBatchSize(100);         // 100개 배치
  factory.setReceiveTimeout(1000);   // 1초 내 수집된 메시지만 배치 처리
  factory.setPrefetchCount(200);     // 배치보다 크게 설정
  return factory;
}

@RabbitListener(queues = "analytics.queue",
  containerFactory = "batchFactory")
public void handleBatch(List<Message> messages, Channel channel) throws Exception {
  // 100개를 한 번에 처리 (DB 배치 insert 등)
  analyticsRepository.batchInsert(messages.stream()
    .map(this::convert).collect(toList()));

  // 마지막 deliveryTag로 배치 전체 Ack
  long lastTag = messages.get(messages.size() - 1)
    .getMessageProperties().getDeliveryTag();
  channel.basicAck(lastTag, true);  // multiple=true
}
```

### 3. 처리량 측정과 병목 진단

```
=== 병목 위치 파악 ===

Queue Depth 증가 속도로 병목 파악:

상황 1: Queue Depth 계속 증가
  발행 속도 > 소비 속도
  → Consumer 처리량 부족 (Consumer Scale Out)

상황 2: Queue Depth 0, Consumer Utilisation 낮음
  Consumer가 놀고 있음
  → 발행 속도 부족 또는 Prefetch 낮음

상황 3: Queue Depth 0, Consumer Utilisation 100%
  Consumer가 최대 처리 중
  → 처리량 최대 도달 (더 이상 발행이 필요 없거나 Consumer 추가)

=== 처리량 측정 ===

Management UI 지표:
  Queues → [Queue] → Message Rates
  - publish/s: 초당 발행 수
  - deliver/s: 초당 전달 수
  - ack/s:    초당 Ack 수

ack/s = 실제 Consumer 처리량

Prometheus 쿼리:
  rate(rabbitmq_queue_messages_published_total[1m])  # 발행량
  rate(rabbitmq_queue_messages_delivered_ack_total[1m])  # 처리량

=== 복합 튜닝 시나리오 ===

초기 상태:
  처리량: 200 msg/sec (목표: 2000 msg/sec)
  concurrency=1, prefetch=1

단계 1: Prefetch 증가
  prefetch=10 → 처리량: 400 msg/sec (+100%)
  (네트워크 대기 시간 제거)

단계 2: Concurrency 증가
  concurrency=10 → 처리량: 1500 msg/sec (+275%)

단계 3: Publisher 비동기 Confirm
  비동기 ConfirmCallback → 발행 병목 제거
  → 처리량: 2000 msg/sec (목표 달성)

단계 4: 배치 소비 (필요 시)
  BatchListener → 처리량: 2500 msg/sec (여유 확보)
```

---

## 💻 실전 실험

### 실험 1: 처리량 벤치마크

```java
@Component
public class ThroughputBenchmark {

  @Autowired private RabbitTemplate rabbitTemplate;

  public void runBenchmark(int messageCount) throws Exception {
    // 비동기 Confirm 카운터
    AtomicLong confirmed = new AtomicLong(0);
    CountDownLatch latch = new CountDownLatch(messageCount);

    rabbitTemplate.setConfirmCallback((correlation, ack, cause) -> {
      if (ack) confirmed.incrementAndGet();
      latch.countDown();
    });

    long start = System.currentTimeMillis();
    
    for (int i = 0; i < messageCount; i++) {
      CorrelationData cd = new CorrelationData(String.valueOf(i));
      rabbitTemplate.convertAndSend("bench.exchange", "bench", 
        "message-" + i, cd);
    }

    latch.await(30, TimeUnit.SECONDS);
    long elapsed = System.currentTimeMillis() - start;

    System.out.printf("처리량: %d msg/sec (총 %dms, 확인: %d/%d)%n",
      messageCount * 1000L / elapsed, elapsed,
      confirmed.get(), messageCount);
  }
}
```

---

## 📊 Kafka vs RabbitMQ 비교

```
=== 처리량 비교 ===

RabbitMQ 최적화 설정:
  비동기 Confirm + concurrency=50 + prefetch=10
  일반 서버: 수만~수십만 msg/sec

Kafka Producer:
  acks=all + linger.ms=5 (배치) + compression.type=snappy
  일반 서버: 수십만~수백만 msg/sec

Kafka가 처리량에서 유리한 이유:
  append-only 로그 구조 → 순차 디스크 쓰기 (빠름)
  Consumer가 오프셋으로 배치 pull → 브로커 부하 낮음
  Partition 병렬 처리 → 자연스러운 수평 확장

RabbitMQ 처리량 한계:
  Exchange → Binding 평가 → Queue → Consumer push
  각 단계에서 오버헤드 발생
  복잡한 라우팅일수록 처리량 감소

선택:
  수만 msg/sec 이하: RabbitMQ 충분
  수십만~수백만 msg/sec: Kafka 고려
```

---

## ⚖️ 트레이드오프

```
처리량 vs 신뢰성:

최대 처리량:
  TRANSIENT + autoAck + 비동기 Confirm 없음
  → 메시지 유실 가능, 최고 처리량

균형:
  PERSISTENT + 수동 Ack + 비동기 Confirm
  → 신뢰성 + 높은 처리량 (10~30% 감소)

완전 신뢰성:
  Quorum Queue + Outbox + 수동 Ack + Confirm
  → 가장 낮은 처리량, 완전 보장

처리량 튜닝 순서:
  1. Prefetch 증가 (즉각 효과, 리스크 낮음)
  2. Concurrency 증가 (CPU/DB 한도 확인 필요)
  3. 비동기 Confirm (구현 복잡도 증가)
  4. 배치 소비 (특정 패턴에만 적합)
  5. 메시지 압축 (1KB 이상 메시지에만)
```

---

## 📌 핵심 정리

```
처리량 최적화 핵심:

Publisher 측:
  동기 Confirm → 비동기 ConfirmCallback (100배 처리량)
  연속 발행으로 TCP Nagle 효과 활용
  1KB 이상 메시지만 압축

Consumer 측:
  concurrency = 목표처리량 / (1/처리시간) 공식으로 계산
  prefetch = 처리시간ms / RTTms (지연 기반)
  배치 소비: 배치 Insert 등 작업에 유리

병목 진단:
  Queue Depth 증가 → Consumer 처리량 부족
  Consumer Utilisation 낮음 → Prefetch 부족
  ack/s = 실제 처리량 지표

튜닝 순서:
  Prefetch → Concurrency → 비동기 Confirm → 배치
```

---

## 🤔 생각해볼 문제

**Q1.** 비동기 Publisher Confirm에서 미확인(Unconfirmed) 메시지가 계속 쌓인다. 어떻게 제어해야 하는가?

<details>
<summary>해설 보기</summary>

**Semaphore로 미확인 메시지 수를 제한합니다.**

```java
Semaphore semaphore = new Semaphore(1000);  // 최대 미확인 1000개

// 발행 전 슬롯 획득
semaphore.acquire();
CorrelationData cd = new CorrelationData(UUID.randomUUID().toString());
rabbitTemplate.convertAndSend(exchange, key, payload, cd);

// ConfirmCallback에서 슬롯 반환
rabbitTemplate.setConfirmCallback((correlation, ack, cause) -> {
  semaphore.release();
  if (!ack) handleNack(correlation.getId());
});
```

`semaphore.acquire()`가 블로킹되므로 미확인 메시지가 1000개를 초과하면 Publisher가 자동으로 느려집니다. 이것이 Back-pressure입니다.

1000개 제한이 적절한지는 메시지 크기에 따라 다릅니다:
- 메시지 1KB × 1000개 = 1MB 메모리 (적정)
- 메시지 100KB × 1000개 = 100MB 메모리 (주의)

</details>

---

**Q2.** Concurrency=50으로 설정했더니 DB 연결 풀이 고갈됐다. 어떻게 해결하는가?

<details>
<summary>해설 보기</summary>

**DB 연결 풀 크기를 Consumer Concurrency에 맞게 조정하거나, Consumer를 줄이는 방법 중 하나를 선택합니다.**

연결 풀 크기 공식:
```
DB Pool Size = Consumer Concurrency × 최대 동시 DB 쿼리 수
```

Consumer 50개, 각 1개 DB 쿼리: Pool Size = 50

**HikariCP 설정:**
```yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 60    # 여유 포함
      minimum-idle: 10
      connection-timeout: 5000 # 5초 대기 (기본 30초 → 줄임)
```

**연결 풀 고갈 방지:**
1. Pool Size를 Concurrency에 맞게 증가
2. Concurrency를 Pool Size에 맞게 감소
3. DB 연결을 배치로 재사용 (Batch Insert)
4. 읽기 전용 쿼리는 Read Replica 사용 (Pool 분리)

권장: DB Pool Size = Consumer Concurrency × 1.2 (20% 여유)

</details>

---

**Q3.** 처리 시간이 10ms인 메시지가 초당 5000개 발행된다. 최적의 concurrency와 prefetch는 무엇인가?

<details>
<summary>해설 보기</summary>

**이론적 계산:**

필요 Concurrency = 발행량 / (1/처리시간)
= 5000 / (1/0.01) = 5000 / 100 = **50개 스레드**

Prefetch 계산 (RTT 5ms 가정):
= 처리시간 / RTT = 10ms / 5ms = **2~5개**

**권장 설정:**
```yaml
spring:
  rabbitmq:
    listener:
      simple:
        concurrency: 50
        max-concurrency: 70   # 피크 20% 여유
        prefetch: 5           # 각 스레드 5개 미리 수신
```

**총 Unacked = 50 × 5 = 250개**

**실제 검증:**
1. 위 설정으로 시작
2. Consumer Utilisation 측정:
   - 90%+: 설정 적정 또는 concurrency 증가
   - 70%~: prefetch 증가 검토
   - 50%~: concurrency가 많거나 prefetch 부족
3. Queue Depth 모니터링: 0 유지되면 충분

처리 시간 10ms는 짧으므로 네트워크 효율을 위해 prefetch를 5~10 이상으로 설정하는 것이 좋습니다.

</details>

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[이전: Chapter 4 — Saga 패턴 ⬅️](../messaging-patterns/06-saga-pattern.md)** | **[다음: 메모리 관리 ➡️](./02-memory-management.md)**

</div>
