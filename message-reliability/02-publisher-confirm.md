# Publisher Confirm — Broker 수신 보장의 원리

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Publisher Confirm은 어떻게 Broker의 메시지 수신을 보장하는가?
- 개별 Confirm과 배치 Confirm의 처리량 차이는 어떻게 되는가?
- Nack를 받으면 어떻게 재처리해야 하는가?
- mandatory 플래그와 Publisher Confirm은 어떻게 함께 동작하는가?
- Spring AMQP에서 Publisher Confirm을 올바르게 설정하는 방법은?

---

## 🔍 왜 이 개념이 실무에서 중요한가

"메시지를 발행했다"와 "브로커가 메시지를 받았다"는 다른 말이다. 네트워크 오류, 브로커 과부하, Exchange 라우팅 실패 등으로 발행이 조용히 실패할 수 있다. Publisher Confirm은 브로커가 "받았다"는 명시적 확인을 Publisher에게 돌려주는 메커니즘이다. Confirm 없이는 발행 성공 여부를 알 수 없고, 메시지 유실을 탐지할 수 없다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: Confirm 없이 발행 후 "성공"으로 처리

  rabbitTemplate.convertAndSend("order.exchange", "order.placed", event);
  // return void → 성공으로 착각
  
  결과:
    브로커가 받았는지 확인 없음
    발행 실패해도 예외 없음 → 조용한 유실

실수 2: 동기 Confirm을 루프에서 사용 (처리량 붕괴)

  for (OrderEvent event : events) {
    channel.basicPublish(...);
    channel.waitForConfirmsOrDie(5000);  // 매 메시지마다 동기 대기
  }
  
  결과:
    1000개 메시지 × RTT 10ms = 10초 소요
    비동기 Confirm이면 1~2초로 단축 가능

실수 3: Confirm 콜백에서 예외 무시

  rabbitTemplate.setConfirmCallback((correlation, ack, cause) -> {
    if (!ack) {
      log.warn("Nack received");  // 로그만, 재처리 없음
    }
  });
  
  결과:
    Nack 받은 메시지 재처리 안 함 → 유실
```

---

## ✨ 올바른 접근 (After — 비동기 Confirm + 재처리)

```
Spring AMQP 올바른 Publisher Confirm 설정:

application.yml:
  spring:
    rabbitmq:
      publisher-confirm-type: correlated  # CORRELATED 모드 (개별 Confirm 추적)
      publisher-returns: true             # Return 콜백 활성화

@Configuration:
  @Bean
  public CachingConnectionFactory connectionFactory() {
    CachingConnectionFactory factory = new CachingConnectionFactory();
    factory.setPublisherConfirmType(CachingConnectionFactory.ConfirmType.CORRELATED);
    factory.setPublisherReturns(true);
    return factory;
  }

  @Bean
  public RabbitTemplate rabbitTemplate(ConnectionFactory connectionFactory) {
    RabbitTemplate template = new RabbitTemplate(connectionFactory);
    template.setMandatory(true);  // Return 콜백 활성화 필수
    
    // Confirm 콜백 (비동기)
    template.setConfirmCallback((correlation, ack, cause) -> {
      if (!ack) {
        // Nack → 재발행 또는 DB에 실패 기록
        String messageId = correlation.getId();
        outboxService.markFailed(messageId);
        log.error("Publisher Nack: messageId={}, cause={}", messageId, cause);
      }
    });
    
    // Return 콜백 (라우팅 실패)
    template.setReturnsCallback(returned -> {
      log.error("Message returned: exchange={}, routingKey={}, replyCode={}",
        returned.getExchange(), returned.getRoutingKey(), returned.getReplyCode());
    });
    
    return template;
  }
```

---

## 🔬 내부 동작 원리

### 1. Publisher Confirm 동작 흐름

```
=== AMQP 레벨 Confirm ===

1. Confirm 모드 활성화:
   Publisher → Broker: Confirm.Select
   Broker → Publisher: Confirm.SelectOk

2. 메시지 발행:
   Publisher → Broker: Basic.Publish (sequenceNumber=1)
   Publisher → Broker: Basic.Publish (sequenceNumber=2)
   Publisher → Broker: Basic.Publish (sequenceNumber=3)

3. Broker 처리 후 Confirm:
   Broker → Publisher: Basic.Ack (deliveryTag=1, multiple=false)
   Broker → Publisher: Basic.Ack (deliveryTag=3, multiple=true)
   ← multiple=true: deliveryTag=3 이하 모두 Ack (1,2,3 모두 확인)

Broker가 Nack를 보내는 경우:
   Broker → Publisher: Basic.Nack (deliveryTag=N, multiple=false)
   → 브로커 내부 오류, 큐 가득 참 등

=== Confirm 시점 ===

durable=false Queue에 발행:
  메시지 Queue에 저장 시 Ack

durable=true Queue + DeliveryMode=PERSISTENT:
  메시지 디스크에 기록 시 Ack (fsync 완료)
  → 더 느리지만 더 강한 보장

Quorum Queue:
  과반수 노드에 WAL 기록 완료 시 Ack
  → 가장 강한 보장

=== sequenceNumber 추적 ===

발행 시 채널의 sequenceNumber 증가 (1부터 시작):
  channel.getNextPublishSeqNo() → 다음 발행 번호 반환

미확인 메시지 추적:
  ConcurrentHashMap<Long, Message> unconfirmed = new ConcurrentHashMap<>();
  
  long seqNo = channel.getNextPublishSeqNo();
  channel.basicPublish(...);
  unconfirmed.put(seqNo, message);
  
  Ack 수신 시:
    unconfirmed.remove(deliveryTag);  // 성공 확인
  
  Nack 수신 시:
    Message failed = unconfirmed.remove(deliveryTag);
    republish(failed);  // 재발행
```

### 2. 개별 Confirm vs 배치 Confirm 성능

```
=== 동기 개별 Confirm (가장 느림) ===

for each message:
  basicPublish(msg)
  waitForConfirmsOrDie(timeout)  // 동기 대기
  
처리량: 1 / (처리시간 + RTT)
  RTT 10ms이면 최대 100 msg/sec

=== 동기 배치 Confirm (중간) ===

BATCH_SIZE = 100
for each message:
  basicPublish(msg)
  if (count++ % BATCH_SIZE == 0):
    waitForConfirmsOrDie(timeout)  // 100개마다 동기 대기

처리량: 100 / (처리시간 × 100 + RTT)
  RTT 10ms이면 최대 ~1,000 msg/sec

=== 비동기 Confirm (가장 빠름) ===

channel.addConfirmListener(ackCallback, nackCallback)  // 콜백 등록
for each message:
  basicPublish(msg)  // 바로 다음 메시지 발행 (Ack 기다리지 않음)

Ack/Nack는 비동기 콜백으로 처리

처리량: 브로커 처리 속도에 의존 (RTT 비용 없음)
  → 수십만 msg/sec 가능 (서버 스펙에 따라)

Spring AMQP 비동기 Confirm:
  publisher-confirm-type: correlated → 내부적으로 비동기 처리
  ConfirmCallback → 비동기 호출됨 (Publisher Thread가 아닌 리스너 Thread)
```

### 3. Mandatory 플래그와 Return 콜백

```
=== mandatory=false (기본) ===

Exchange에 Binding 없거나 Routing Key 매칭 실패:
  → 메시지 조용히 폐기
  → Publisher Confirm은 Ack 반환 (Exchange는 받았으므로)
  → Publisher는 메시지가 Queue에 도달 못 한 것을 모름

=== mandatory=true ===

Exchange에서 라우팅 실패 시:
  Broker → Publisher: Basic.Return (replyCode=312 NO_ROUTE, message 포함)
  이후 Broker → Publisher: Basic.Ack
  
  Publisher: ReturnsCallback 호출 → 라우팅 실패 처리
  Publisher: ConfirmCallback도 호출 (Ack) — Exchange는 받았으므로

두 콜백의 순서:
  ReturnsCallback → ConfirmCallback (Return이 먼저)
  → Return 받았으면 ConfirmCallback의 ack=true여도 Queue 미도달

=== 완전한 방어 ===

  mandatory=true
  + ConfirmCallback: Nack 처리 (브로커 내부 오류)
  + ReturnsCallback: 라우팅 실패 처리 (Queue 미도달)
  
  둘 다 설정해야 모든 유실 탐지 가능
```

---

## 💻 실전 실험

### 실험 1: Publisher Confirm 동작 확인

```java
// 직접 Channel 사용 예시 (원리 이해용)
@Component
public class ConfirmPublisher {
  @Autowired private Connection connection;

  public void publishWithConfirm(String exchange, String key, byte[] body) 
      throws Exception {
    try (Channel channel = connection.createChannel()) {
      channel.confirmSelect();  // Confirm 모드 활성화
      
      ConcurrentNavigableMap<Long, String> unconfirmed = 
        new ConcurrentSkipListMap<>();
      
      channel.addConfirmListener(
        (seqNo, multiple) -> {  // Ack
          if (multiple) unconfirmed.headMap(seqNo, true).clear();
          else unconfirmed.remove(seqNo);
        },
        (seqNo, multiple) -> {  // Nack
          log.error("Nack received for seqNo={}", seqNo);
          // 재발행 로직
        }
      );
      
      long seqNo = channel.getNextPublishSeqNo();
      unconfirmed.put(seqNo, new String(body));
      channel.basicPublish(exchange, key,
        MessageProperties.PERSISTENT_TEXT_PLAIN, body);
      
      // 1초 내 모든 Confirm 대기
      if (!channel.waitForConfirms(1000)) {
        throw new RuntimeException("Confirm timeout");
      }
    }
  }
}
```

### 실험 2: Spring AMQP Confirm 처리량 비교

```java
@Component
public class ConfirmBenchmark {
  @Autowired private RabbitTemplate rabbitTemplate;
  
  // 비동기 Confirm: ConfirmCallback으로 처리
  public void sendAsync(List<OrderEvent> events) {
    AtomicInteger ackCount = new AtomicInteger(0);
    AtomicInteger nackCount = new AtomicInteger(0);
    
    for (OrderEvent event : events) {
      CorrelationData correlation = new CorrelationData(event.getId().toString());
      rabbitTemplate.convertAndSend("order.exchange", "order.placed", event, correlation);
    }
    
    // ConfirmCallback은 비동기로 호출됨
    // 모든 Confirm을 받을 때까지 대기 필요 시 CountDownLatch 사용
  }
}
```

---

## 📊 Kafka vs RabbitMQ 비교

```
=== Producer 발행 보장 비교 ===

RabbitMQ Publisher Confirm:
  acks 설정:
  - Confirm 없음: 발행 후 확인 없음 (최고 성능, 유실 가능)
  - Confirm 활성화: Exchange → Queue 저장 확인 후 Ack
  - Quorum Queue: 과반수 노드 저장 후 Ack (최강 보장)

Kafka Producer acks:
  acks=0: 전송 즉시 완료 (확인 없음, 최고 처리량)
  acks=1: Leader 수신 확인 (ISR 복제 전 Leader 장애 시 유실 가능)
  acks=all: ISR 전체 확인 (가장 강한 보장, min.insync.replicas 연동)

비교:
  acks=0 ≈ Confirm 없음
  acks=1 ≈ Confirm 활성화 (단일 노드 보장)
  acks=all ≈ Quorum Queue Confirm (과반수 보장)

=== Idempotent Producer ===

Kafka:
  enable.idempotence=true → Producer ID + Sequence Number로 중복 발행 방지
  브로커 레벨에서 중복 감지

RabbitMQ:
  브로커 레벨 Idempotent 발행 미지원
  → Consumer가 메시지 ID로 중복 감지 (at-least-once + 멱등 Consumer)
```

---

## ⚖️ 트레이드오프

```
Publisher Confirm 설정별 트레이드오프:

Confirm 없음:
  ✅ 최고 처리량
  ❌ 발행 실패 탐지 불가, 유실 가능

개별 동기 Confirm:
  ✅ 즉시 발행 결과 파악
  ❌ 처리량 최저 (RTT 대기)

배치 동기 Confirm:
  ✅ 개별보다 높은 처리량
  ❌ 배치 실패 시 어느 메시지가 실패했는지 불명확

비동기 Confirm (권장):
  ✅ 최고 처리량
  ✅ 모든 메시지 Ack/Nack 개별 추적
  ❌ 복잡한 구현 (상태 추적 필요)

Outbox Pattern + 비동기 Confirm:
  ✅ DB 트랜잭션 + 발행 원자성 보장
  ✅ 발행 실패 시 DB에서 재발행 가능
  ❌ 구현 복잡, DB I/O 추가
```

---

## 📌 핵심 정리

```
Publisher Confirm 핵심:

동작:
  channel.confirmSelect() → Confirm 모드 활성화
  basicPublish → sequenceNumber 부여
  Ack: 메시지 Queue 저장 완료
  Nack: 브로커 오류 (재발행 필요)

Confirm 시점:
  durable=false Queue: Queue 저장 시
  durable=true + PERSISTENT: 디스크 기록 시
  Quorum Queue: 과반수 노드 WAL 기록 시 (가장 강함)

성능 순서:
  비동기 Confirm > 배치 동기 Confirm > 개별 동기 Confirm

mandatory=true + ReturnCallback:
  라우팅 실패(Queue 미도달) 탐지
  Confirm Ack여도 Return이면 Queue 미도달

Spring AMQP:
  publisher-confirm-type: correlated
  ConfirmCallback + ReturnsCallback 설정 필수
```

---

## 🤔 생각해볼 문제

**Q1.** Publisher Confirm Ack를 받은 메시지도 유실될 수 있는가?

<details>
<summary>해설 보기</summary>

**이론적으로는 매우 드물게 가능합니다.** 실용적으로는 Quorum Queue를 사용하면 거의 불가능합니다.

가능한 시나리오 (Classic Queue 단일 노드):
1. PERSISTENT 메시지가 OS 페이지 캐시에 있을 때 Ack 반환 (fsync 완료 전)
2. Ack 직후 전원 차단 → 페이지 캐시 소실 → 메시지 유실

완화 방법:
- **Quorum Queue**: Raft WAL이 과반수 노드에 기록된 후 Ack → 전원 차단 내성
- `no-ack`가 아닌 `persistent` + Quorum Queue 조합이 최강 보장

현실적으로 완전한 Exactly-Once는 분산 시스템에서 달성하기 어렵습니다. At-Least-Once + 멱등 Consumer 조합이 실용적 해결책입니다.

</details>

---

**Q2.** 발행 속도가 Confirm 수신 속도보다 빠르면 어떻게 되는가?

<details>
<summary>해설 보기</summary>

**미확인(Unconfirmed) 메시지가 메모리에 누적됩니다.**

ConcurrentHashMap에 `{seqNo → message}` 로 추적하는 경우, 발행 속도가 Confirm 처리 속도를 초과하면 Map의 크기가 계속 증가합니다.

대응 방법:
1. **Channel-level flow control**: 브로커가 Confirm 전에 다음 메시지를 받지 않도록 Publisher throttling
2. **Unconfirmed 임계값 설정**: 미확인 메시지가 N개 초과 시 발행 중단 (Semaphore)

```java
Semaphore semaphore = new Semaphore(1000);  // 최대 미확인 1000개

// 발행 시
semaphore.acquire();
channel.basicPublish(...);

// Ack 수신 시
semaphore.release();
```

3. **Back-pressure**: 미확인 누적 시 발행자에게 느리게 보내라는 신호

Spring AMQP의 `CachingConnectionFactory.setPublisherReturns(true)`와 함께 사용 시 반환된 채널이 Confirm 처리 스레드와 경합할 수 있으므로 Thread-safe한 구현 필요.

</details>

---

**Q3.** mandatory=false이고 Exchange에 Binding이 없을 때, Publisher Confirm은 Ack를 받는가 Nack를 받는가?

<details>
<summary>해설 보기</summary>

**Ack를 받습니다.** 이것이 mandatory=false의 위험성입니다.

동작 순서:
1. Publisher: basicPublish(exchange="order.exchange", routingKey="no.binding", ...)
2. Broker: Exchange 수신 ✅ → "메시지를 받았다" → Basic.Ack 발송
3. Broker: Binding 없음 → 메시지 폐기 (mandatory=false이므로 Return 없음)
4. Publisher: ConfirmCallback(ack=true) 수신

Publisher는 Ack를 받았으므로 성공으로 처리합니다. 하지만 메시지는 Queue에 도달하지 않고 폐기됐습니다.

이것을 막으려면:
- `mandatory=true` + `ReturnsCallback` 설정
- ReturnsCallback에서 라우팅 실패 메시지 재처리

따라서 **완전한 방어 = Publisher Confirm + mandatory=true + ReturnsCallback** 세 가지 모두 필요합니다.

</details>

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[이전: 메시지 유실의 세 지점 ⬅️](./01-message-loss-points.md)** | **[다음: 메시지 영속성 ➡️](./03-message-persistence.md)**

</div>
