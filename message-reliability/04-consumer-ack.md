# Consumer Acknowledgement 완전 분해 — 처리 보장의 핵심 메커니즘

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `autoAck=true`는 정확히 언제 메시지를 삭제하는가?
- `basicNack`와 `basicReject`의 차이는 무엇인가?
- `requeue=true`와 `requeue=false`는 언제 각각 선택하는가?
- 수동 Ack에서 처리 완료 후 Ack를 보내지 않으면 어떻게 되는가?
- Spring AMQP에서 Ack 모드는 어떻게 설정하고 제어하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

Consumer Ack는 "이 메시지를 처리 완료했다"는 신호다. 이 신호를 언제 보내느냐가 메시지 유실과 중복 처리의 경계를 결정한다. `autoAck=true`는 Consumer가 메시지를 받는 순간 삭제한다. 처리 중 예외가 발생하면 그 메시지는 영원히 사라진다. 반대로 `requeue=true`를 아무 생각 없이 쓰면 같은 메시지가 무한 루프에 빠진다. Ack의 종류와 타이밍을 정확히 이해해야 "유실 없이, 무한 루프 없이" 메시지를 처리할 수 있다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: autoAck=true로 중요 메시지 처리

  @RabbitListener(queues = "order.queue")
  // 기본 설정 → autoAck=true (Spring AMQP 기본은 AUTO지만 많은 설정에서 true)
  public void handleOrder(OrderEvent event) {
    // 메시지 수신 즉시 Queue에서 삭제됨
    orderService.process(event);  // 예외 발생 시 → 메시지 영구 소실
  }

실수 2: 처리 전에 Ack

  public void handleOrder(Message msg, Channel channel,
      @Header(AmqpHeaders.DELIVERY_TAG) long tag) throws Exception {
    channel.basicAck(tag, false);   // ← 처리 전에 Ack!
    orderService.process(msg);       // 예외 발생해도 이미 Ack됨
  }
  // 결과: 처리 실패해도 메시지 삭제 → 유실

실수 3: 모든 예외에 requeue=true

  } catch (Exception e) {
    channel.basicNack(tag, false, true);  // 무조건 requeue
  }
  // 결과: 수정 불가능한 에러(잘못된 메시지 형식)도
  //       무한 루프로 재처리 → CPU 100%, 다른 메시지 차단

실수 4: Ack 누락 (finally 없음)

  public void handleOrder(Message msg, Channel channel,
      @Header(AmqpHeaders.DELIVERY_TAG) long tag) throws Exception {
    try {
      orderService.process(msg);
      channel.basicAck(tag, false);
    } catch (Exception e) {
      log.error("처리 실패", e);
      // Ack/Nack 없음 → 메시지 Unacked 상태로 무한 대기
    }
  }
  // 결과: Prefetch Count만큼 Unacked 메시지가 쌓이고
  //       해당 Consumer는 새 메시지를 받지 못함
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계)

```
올바른 Ack 전략:

원칙:
  1. 항상 수동 Ack (autoAck=false)
  2. 처리 완료 후 Ack (처리 전 Ack 금지)
  3. 예외 시 반드시 Nack (Ack 누락 금지)
  4. 회복 가능 에러 → requeue=true (일시적 DB 오류 등)
  5. 회복 불가 에러 → requeue=false + DLX (잘못된 메시지 형식 등)

Spring AMQP 설정:
  # application.yml
  spring:
    rabbitmq:
      listener:
        simple:
          acknowledge-mode: manual  # 수동 Ack

  @RabbitListener(queues = "order.queue")
  public void handleOrder(Message message, Channel channel,
      @Header(AmqpHeaders.DELIVERY_TAG) long deliveryTag) throws IOException {
    try {
      orderService.process(message);
      channel.basicAck(deliveryTag, false);   // 처리 완료 후 Ack
    } catch (BusinessException e) {
      // 재처리 의미 없는 비즈니스 예외 → DLQ로
      channel.basicNack(deliveryTag, false, false);
    } catch (TransientException e) {
      // 일시적 오류 (DB 연결 등) → 재큐
      channel.basicNack(deliveryTag, false, true);
    } catch (Exception e) {
      // 알 수 없는 예외 → 안전하게 DLQ로
      channel.basicNack(deliveryTag, false, false);
    }
  }
```

---

## 🔬 내부 동작 원리

### 1. autoAck=true의 정확한 동작 — 언제 삭제되는가

```
=== AMQP 레벨에서 autoAck=true ===

basicConsume(queue, noAck=true, ...)
                         ↑ noAck=true = autoAck=true

서버 동작:
  메시지를 Consumer에게 전송하는 순간
  (Consumer가 실제로 받기도 전에)
  서버 측 Queue에서 메시지 삭제

타임라인:
  Queue: [msg1][msg2][msg3]
  Server: msg1 → Consumer에게 전송
          ← 전송 완료 순간 → Queue에서 msg1 삭제
  Network: msg1 패킷 이동 중...
  Consumer: msg1 수신 (이미 서버에서는 삭제됨)
            msg1 처리 중...
            예외 발생!
  → msg1 처리 실패, 하지만 서버에서 이미 삭제 → 영구 유실

=== autoAck=false (수동 Ack)의 동작 ===

basicConsume(queue, noAck=false, ...)

서버 동작:
  메시지를 Consumer에게 전송
  Queue에서는 삭제하지 않음 → Unacked 상태로 마킹
  
  Queue: [msg2][msg3]  (Ready)
  Unacked: [msg1]      (Consumer가 처리 중)

Consumer: basicAck(deliveryTag=1) 호출
  → 서버: Unacked 목록에서 msg1 제거 → Queue에서 완전 삭제

Consumer 연결 끊김 (Ack 없이):
  → 서버: Unacked → Ready 상태로 복귀
  → 다른 Consumer에게 재전달

=== autoAck=true가 안전한 유일한 경우 ===

✅ 메시지 유실이 허용되는 경우:
  - 실시간 메트릭 (유실 허용)
  - 캐시 워밍 이벤트 (재처리 가능)
  - 처리 자체가 실패할 수 없는 단순 로컬 저장

✅ 처리량 극대화가 필요한 경우:
  - Ack 오버헤드 없애야 할 때

❌ 사용 금지:
  - 주문, 결제, 재고 등 중요 비즈니스 로직
  - 유실이 허용되지 않는 모든 데이터
```

### 2. basicAck vs basicNack vs basicReject

```
=== basicAck — 처리 성공 ===

channel.basicAck(deliveryTag, multiple)

  deliveryTag: 메시지의 고유 식별자 (Channel 내에서 단조 증가)
  multiple: true → deliveryTag 이하 모든 Unacked 메시지 일괄 Ack
            false → 해당 deliveryTag 하나만 Ack

multiple=true 활용:
  Prefetch=100으로 100개를 받아 배치 처리
  → 처리 완료 후 마지막 deliveryTag로 basicAck(lastTag, multiple=true)
  → 100개를 한 번의 Ack로 처리 (네트워크 절약)

=== basicNack — 처리 실패, 배치 지원 ===

channel.basicNack(deliveryTag, multiple, requeue)

  deliveryTag: 실패한 메시지 식별자
  multiple: true → deliveryTag 이하 모두 Nack
  requeue: true → Queue로 돌려보냄 (재처리)
           false → Queue에서 제거 (DLX 있으면 DLX로, 없으면 폐기)

=== basicReject — 처리 실패, 단일 메시지만 ===

channel.basicReject(deliveryTag, requeue)

  basicNack와 동일하지만 multiple 옵션 없음 (항상 단일 메시지)
  AMQP 표준 스펙 (basicNack는 RabbitMQ 확장)
  
  차이 정리:
  basicNack(tag, false, false) = basicReject(tag, false)  ← 동일
  basicNack(tag, true, false)  ← 배치 Nack (basicReject 불가)

=== 세 명령의 결과 흐름 ===

메시지 운명:

                    basicAck
                    → Queue에서 영구 삭제
                    
  메시지 수신
  (Unacked)
                    basicNack(requeue=true)
                    basicReject(requeue=true)
                    → Queue 앞으로 복귀 (Ready 상태)
                    → 재전달

                    basicNack(requeue=false)
                    basicReject(requeue=false)
                    → DLX 설정 있음: DLX로 이동
                    → DLX 설정 없음: 폐기
```

### 3. requeue=true vs requeue=false 선택 기준

```
=== requeue=true 선택 기준 ===

✅ 적합한 경우:
  - 일시적 외부 의존성 오류 (DB 연결 실패, API 타임아웃)
  - "잠시 후 재시도하면 성공할 가능성이 있는" 오류
  
  예시:
    catch (DataAccessException e) {  // DB 연결 실패
      channel.basicNack(tag, false, true);  // 재큐 (DB 복구 후 재처리)
    }

❌ 위험한 경우:
  - 즉시 재큐 → 같은 오류로 즉시 실패 → 무한 루프
  - CPU 폭발, 정상 메시지 처리 차단
  
  해결: 즉시 재큐 대신 지연 재시도 (Ch3-05에서 상세 설명)

=== requeue=false 선택 기준 ===

✅ 적합한 경우:
  - 메시지 자체가 잘못된 경우 (역직렬화 실패, 필수 필드 없음)
  - 비즈니스 규칙 위반 (중복 주문 ID, 잘못된 상태 전이)
  - 재시도해도 의미 없는 오류 (같은 메시지로 항상 같은 예외)
  
  예시:
    catch (MessageConversionException e) {  // 역직렬화 실패
      channel.basicNack(tag, false, false);  // DLQ로 이동
    }

=== 예외 분류 체계 ===

예외 유형 분류 패턴:
  
  try {
    process(message);
    basicAck(tag, false);
  } catch (Exception e) {
    if (isTransient(e)) {
      // 일시적: DB 연결, 네트워크, 외부 API
      basicNack(tag, false, true);   // requeue
    } else {
      // 영구적: 메시지 형식, 비즈니스 로직 오류
      basicNack(tag, false, false);  // DLQ
    }
  }

  private boolean isTransient(Exception e) {
    return e instanceof DataAccessException
        || e instanceof ConnectException
        || e instanceof SocketTimeoutException;
  }
```

### 4. Spring AMQP Ack 모드

```
=== AcknowledgeMode 설정 ===

spring.rabbitmq.listener.simple.acknowledge-mode:

  NONE (= autoAck=true):
    메시지 수신 즉시 Ack → 처리 전 삭제
    사용: 유실 허용, 최고 처리량

  AUTO (기본값):
    메서드 정상 완료 → Ack
    예외 throw → Nack (requeue=true, 기본)
    → 예외 시 무한 루프 위험! (기본 동작)
    
    AmqpRejectAndDontRequeueException throw →
      Nack(requeue=false) → DLQ로 이동
    
    defaultRequeueRejected=false 설정 시 →
      모든 예외 → Nack(requeue=false)

  MANUAL:
    개발자가 직접 basicAck/basicNack 호출
    가장 정밀한 제어
    → 프로덕션 권장

=== AUTO 모드에서 안전하게 DLQ 보내기 ===

@RabbitListener(queues = "order.queue")
public void handleOrder(OrderEvent event) {
  try {
    orderService.process(event);
    // 정상 완료 → AUTO 모드가 Ack 처리
  } catch (MessageConversionException e) {
    // 재처리 의미 없음 → DLQ
    throw new AmqpRejectAndDontRequeueException("Invalid message", e);
  } catch (DataAccessException e) {
    // 일시적 오류 → 재큐 (AUTO 모드 기본 동작)
    throw e;
  }
}

=== MANUAL 모드 완전한 예시 ===

spring:
  rabbitmq:
    listener:
      simple:
        acknowledge-mode: manual

@RabbitListener(queues = "order.queue")
public void handleOrder(
    Message message,
    Channel channel,
    @Header(AmqpHeaders.DELIVERY_TAG) long deliveryTag,
    @Header(value = AmqpHeaders.REDELIVERED, required = false) boolean redelivered
) {
  try {
    orderService.process(message);
    channel.basicAck(deliveryTag, false);

  } catch (JsonParseException | MessageConversionException e) {
    // 잘못된 메시지 → DLQ
    log.error("역직렬화 실패, DLQ로 이동: {}", e.getMessage());
    channel.basicNack(deliveryTag, false, false);

  } catch (DataAccessException e) {
    // DB 오류 → 재큐 (단, 재전달된 메시지면 DLQ)
    if (redelivered) {
      log.error("재전달 메시지도 실패 → DLQ");
      channel.basicNack(deliveryTag, false, false);
    } else {
      channel.basicNack(deliveryTag, false, true);
    }

  } catch (Exception e) {
    log.error("알 수 없는 예외 → DLQ", e);
    channel.basicNack(deliveryTag, false, false);
  }
}
```

---

## 💻 실전 실험

### 실험 1: autoAck=true 유실 재현

```bash
# Queue 생성 및 메시지 발행
rabbitmqadmin declare queue name=ack.test durable=true
rabbitmqadmin publish exchange="" routing_key=ack.test \
  payload='{"orderId":1}' properties='{"delivery_mode":2}'

# autoAck=true 시뮬레이션 (수신 즉시 Ack)
rabbitmqadmin get queue=ack.test ackmode=ack_requeue_false

# 메시지 없음 확인 (처리 완료 여부와 무관하게 삭제됨)
rabbitmqctl list_queues name messages
# ack.test  0  ← 삭제됨

# autoAck=false 시뮬레이션 (Ack 없이 수신)
rabbitmqadmin publish exchange="" routing_key=ack.test payload='{"orderId":2}'
rabbitmqadmin get queue=ack.test ackmode=ack_requeue_true  # requeue=true = Ack 안 함

rabbitmqctl list_queues name messages_ready messages_unacknowledged
# ack.test  0  1  ← Ready=0, Unacked=1 (Consumer가 들고 있음)

# rabbitmqadmin 종료 → Connection 끊김 → Unacked → Ready 복귀
rabbitmqctl list_queues name messages_ready messages_unacknowledged
# ack.test  1  0  ← 복구됨
```

### 실험 2: MANUAL Ack Spring AMQP 설정

```java
// application.yml
spring:
  rabbitmq:
    listener:
      simple:
        acknowledge-mode: manual
        prefetch: 10  # 한 번에 받을 메시지 수

// Consumer
@Component
public class OrderConsumer {

  @RabbitListener(queues = "order.queue")
  public void handle(
      @Payload OrderEvent event,
      Channel channel,
      @Header(AmqpHeaders.DELIVERY_TAG) long tag) throws IOException {

    try {
      // 처리 완료 후 Ack
      processOrder(event);
      channel.basicAck(tag, false);

    } catch (InvalidOrderException e) {
      // 재처리 불가 → DLQ
      channel.basicNack(tag, false, false);

    } catch (ServiceUnavailableException e) {
      // 일시적 오류 → 재큐
      channel.basicNack(tag, false, true);
    }
  }
}
```

### 실험 3: Nack requeue=false → DLQ 이동 확인

```bash
# DLX 설정된 Queue 생성
rabbitmqadmin declare exchange name=dlx.exchange type=direct durable=true
rabbitmqadmin declare queue name=order.dlq durable=true
rabbitmqadmin declare binding source=dlx.exchange destination=order.dlq routing_key=order.queue
rabbitmqadmin declare queue name=order.queue durable=true \
  arguments='{"x-dead-letter-exchange":"dlx.exchange","x-dead-letter-routing-key":"order.queue"}'

# 메시지 발행
rabbitmqadmin publish exchange="" routing_key=order.queue payload='{"orderId":1}'

# Nack(requeue=false) = DLQ로 이동
rabbitmqadmin get queue=order.queue ackmode=reject_requeue_false

# DLQ 확인
rabbitmqctl list_queues name messages
# order.dlq  1  ← DLQ에 도착
# order.queue 0  ← 원본에서 제거
```

---

## 📊 Kafka vs RabbitMQ 비교

```
=== Consumer 처리 보장 비교 ===

RabbitMQ basicAck:
  처리 완료 → basicAck → Queue에서 메시지 삭제
  처리 실패 → basicNack(requeue) → Queue 복귀 또는 DLQ
  Ack 없이 연결 끊김 → 자동으로 Ready 복귀

Kafka commitSync/commitAsync:
  처리 완료 → 오프셋 커밋 → "다음 번 이 오프셋부터 읽겠다" 기록
  처리 실패 → 커밋 안 함 → 재시작 시 같은 오프셋부터 재처리
  커밋 없이 종료 → 마지막 커밋 오프셋부터 재처리

핵심 차이:
  RabbitMQ Ack: 처리 완료 → 메시지 삭제 (Broker에서 제거)
  Kafka Commit: 처리 완료 → 오프셋 전진 (메시지는 보존)

=== At-Least-Once 보장 비교 ===

RabbitMQ:
  autoAck=false + basicAck(처리 후)
  → 처리 완료만 삭제, 미처리는 재전달

Kafka:
  enable.auto.commit=false + 처리 후 commitSync()
  → 처리 완료만 오프셋 전진, 미처리는 같은 오프셋 재처리

두 경우 모두 Consumer 멱등성 필요:
  At-Least-Once = 중복 가능 + 멱등 처리
```

---

## ⚖️ 트레이드오프

```
Ack 방식별 트레이드오프:

autoAck=true (NONE):
  ✅ 최고 처리량 (Ack 오버헤드 없음)
  ❌ 처리 실패 시 메시지 유실 (재처리 불가)

AUTO 모드:
  ✅ 코드 단순 (Ack 직접 호출 없음)
  ❌ 예외 시 기본 requeue=true → 무한 루프 위험
  ❌ 세밀한 예외 처리 어려움

MANUAL 모드:
  ✅ 가장 정밀한 제어 (예외별 다른 처리)
  ✅ 처리 실패 → DLQ 또는 재큐 선택 가능
  ❌ 코드 복잡도 증가
  ❌ Ack 누락 시 Unacked 메시지 누적

배치 Ack (multiple=true):
  ✅ 네트워크 절약 (N개를 1번 Ack)
  ❌ 배치 중 하나 실패 시 일부만 Ack 불가 (all-or-nothing)
```

---

## 📌 핵심 정리

```
Consumer Ack 핵심:

autoAck=true:
  전달 즉시 삭제 (처리 완료 아님)
  → 중요 데이터에 절대 사용 금지

수동 Ack 원칙:
  처리 완료 후 basicAck
  예외 시 반드시 basicNack (Ack 누락 금지)
  finally 블록에서 Ack 처리 보장

requeue 선택:
  requeue=true: 일시적 외부 오류 (DB 연결 등) → 재시도
  requeue=false: 영구적 메시지 오류 → DLQ 이동

basicNack vs basicReject:
  basicNack: multiple 지원 (배치), RabbitMQ 확장
  basicReject: 단일 메시지만, AMQP 표준

Spring AMQP:
  acknowledge-mode: manual → 권장 (프로덕션)
  acknowledge-mode: auto → 주의 (기본 requeue=true)
  acknowledge-mode: none → 유실 허용 시만
```

---

## 🤔 생각해볼 문제

**Q1.** `basicAck(deliveryTag, multiple=true)`로 배치 Ack를 보냈는데, 배치 중 특정 메시지만 처리 실패했다. 어떻게 처리해야 하는가?

<details>
<summary>해설 보기</summary>

`multiple=true`는 "deliveryTag 이하 전체를 성공으로 처리"입니다. 배치 중 일부가 실패했다면 두 가지 방법이 있습니다.

**방법 1: 개별 Ack/Nack**
```java
for (Message msg : batch) {
  try {
    process(msg);
    channel.basicAck(getDeliveryTag(msg), false);  // 개별 Ack
  } catch (Exception e) {
    channel.basicNack(getDeliveryTag(msg), false, false);  // 개별 Nack
  }
}
```
각 메시지의 성공/실패를 독립적으로 처리. Ack/Nack 호출 수가 많아지지만 정확합니다.

**방법 2: 실패 메시지 별도 처리 후 전체 Ack**
```java
List<Message> failed = new ArrayList<>();
for (Message msg : batch) {
  try { process(msg); }
  catch (Exception e) { failed.add(msg); }
}
// 실패 메시지는 DLQ로 직접 발행
failed.forEach(m -> rabbitTemplate.send("dlx.exchange", "order.queue", m));
// 배치 전체 Ack (이미 직접 DLQ 처리했으므로)
channel.basicAck(lastDeliveryTag, true);
```

실무에서는 방법 1이 더 명확합니다. `multiple=true`는 "모두 성공"이 확실할 때만 사용합니다.

</details>

---

**Q2.** Consumer가 Prefetch=10으로 10개 메시지를 받았는데, 첫 번째 메시지 처리 중 `basicNack(tag1, multiple=false, requeue=true)`를 호출했다. 나머지 9개는 어떻게 되는가?

<details>
<summary>해설 보기</summary>

**나머지 9개는 그대로 Consumer가 처리합니다.** `multiple=false`이므로 tag1만 영향을 받습니다.

동작:
- tag1: requeue=true → Queue 앞으로 복귀 (Ready 상태)
- tag2~tag10: Unacked 상태 유지 (Consumer가 계속 처리)
- Consumer Prefetch 슬롯: tag1이 Nack되어 Queue로 복귀했으므로 슬롯 1개 확보 → 새 메시지 1개 수신 가능

주의: tag1이 requeue=true로 돌아왔을 때, 다른 Consumer가 있으면 다른 Consumer에게 전달될 수 있습니다. 동일 Consumer에게 즉시 재전달되지 않습니다. (단, Consumer가 하나뿐이면 동일 Consumer로 재전달됨)

재전달된 메시지는 `redelivered=true` 헤더가 설정됩니다.

</details>

---

**Q3.** Ack를 보내지 않은 채 Consumer가 종료됐다(JVM 강제 종료). `shutdown-executor-threads`가 실행되기 전에 종료되면 Unacked 메시지는 어떻게 되는가?

<details>
<summary>해설 보기</summary>

**Connection 종료 시 RabbitMQ가 자동으로 Unacked 메시지를 Ready 상태로 복귀시킵니다.** 이것은 RabbitMQ가 AMQP Connection 레벨에서 처리하므로, JVM 종료 방식(정상 종료, kill -9, OOM Kill 등)에 무관합니다.

동작 흐름:
1. JVM 종료 → TCP Connection 강제 종료
2. RabbitMQ: Heartbeat 실패 또는 TCP RST 감지
3. RabbitMQ: 해당 Connection의 모든 Channel 닫힘
4. RabbitMQ: Channel에 속한 모든 Unacked 메시지 → Ready 상태로 복귀
5. 다른 Consumer(또는 재시작 후 Consumer)에게 재전달

이것이 `autoAck=false`의 핵심 안전망입니다. Consumer가 어떤 방식으로 종료되어도 Unacked 메시지는 보존됩니다.

단, Heartbeat 감지 시간(기본 60초) 동안은 Unacked 상태가 유지됩니다. 이 시간 동안 다른 Consumer도 해당 메시지를 받지 못합니다. 빠른 복구가 필요하면 Heartbeat를 짧게 설정하세요.

</details>

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[이전: 메시지 영속성 ⬅️](./03-message-persistence.md)** | **[다음: 재시도 전략 설계 ➡️](./05-retry-strategy.md)**

</div>
