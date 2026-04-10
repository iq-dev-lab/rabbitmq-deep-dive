# Saga 패턴 구현 — 분산 트랜잭션을 메시지로 조율하기

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Choreography Saga와 Orchestration Saga는 어떻게 다른가?
- RabbitMQ에서 보상 트랜잭션(Compensating Transaction) 이벤트는 어떻게 라우팅하는가?
- Saga 중간에 메시지가 유실되면 어떻게 되는가?
- Orchestrator는 어떤 Queue 구조로 명령을 발행하는가?
- Saga 상태를 어떻게 추적하고 모니터링하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

마이크로서비스 환경에서 "주문 → 결제 → 재고 → 배송"처럼 여러 서비스에 걸친 트랜잭션을 어떻게 처리하는가? DB가 분리되어 있으므로 전통적인 2PC(2단계 커밋) 트랜잭션은 불가능하다. Saga 패턴은 각 단계를 독립적인 로컬 트랜잭션으로 처리하고, 실패 시 이전 단계를 되돌리는 보상 트랜잭션을 발행한다. RabbitMQ는 Saga의 단계별 이벤트/명령을 전달하고, 신뢰성 있는 메시지 보장으로 분산 트랜잭션을 안전하게 진행한다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: 보상 트랜잭션 없는 Saga (롤백 미구현)

  OrderService: 주문 저장 ✅
  PaymentService: 결제 처리 ✅
  InventoryService: 재고 차감 실패 ❌

  보상 트랜잭션 없으면:
    결제는 됐는데 재고가 없음 → 상품 발송 불가
    주문은 됐는데 결제 환불 안 됨
    → 수동으로 데이터 정합성 복구 필요

실수 2: Saga 상태 추적 없음

  "주문이 어떤 단계에서 실패했는지 모른다"
  주문 1: 결제까지 성공, 재고에서 멈춤
  주문 2: 결제 실패, 주문 취소됐는지 불명확

  Saga 상태 저장 없으면:
    장애 분석 불가
    수동 복구 시 어느 단계부터 재처리할지 불명확

실수 3: Choreography에서 이벤트 폭발

  서비스가 5개, 각 서비스가 3종류 이벤트:
    OrderService 구독: PaymentCompleted, InventoryReserved, ShippingStarted...
    PaymentService 구독: OrderPlaced, OrderCancelled...
    → N개 서비스 × M개 이벤트 = 거미줄 의존관계

  서비스 5개에서 10개로 늘어나면 의존관계 폭발
  → 특정 규모 이상에서는 Orchestration이 더 관리 용이
```

---

## ✨ 올바른 접근 (After — 패턴 선택과 보상 트랜잭션)

```
Choreography vs Orchestration 선택:

Choreography (이벤트 기반):
  서비스 < 5개, 단계 < 5개: 적합
  각 서비스가 독립적으로 이벤트 구독
  중앙 조율자 없음 → 단순한 MSA 초기에 적합

Orchestration (명령 기반):
  서비스 > 5개, 복잡한 조건 분기: 적합
  Orchestrator 서비스가 각 서비스에 명령 발행
  Saga 상태 중앙에서 관리 → 추적 용이

보상 트랜잭션 필수 설계:
  모든 성공 이벤트에 대응하는 보상 이벤트 정의
  
  성공            | 보상
  ─────────────── | ─────────────────
  OrderCreated    | OrderCancelled
  PaymentCharged  | PaymentRefunded
  StockReserved   | StockReleased
  ShipmentStarted | ShipmentCancelled (가능한 경우만)
```

---

## 🔬 내부 동작 원리

### 1. Choreography Saga — 이벤트 기반 조율

```
=== 정상 흐름 ===

[OrderService]
  1. 주문 저장 (로컬 트랜잭션)
  2. 발행: order.exchange → "order.placed"
     {orderId, items, userId, amount}

[PaymentService]
  구독: "order.placed"
  3. 결제 처리 (로컬 트랜잭션)
  4. 발행: payment.exchange → "order.payment.completed"
     {orderId, paymentId, amount}

[InventoryService]
  구독: "order.payment.completed"
  5. 재고 예약 (로컬 트랜잭션)
  6. 발행: inventory.exchange → "order.inventory.reserved"
     {orderId, items}

[ShippingService]
  구독: "order.inventory.reserved"
  7. 배송 시작 (로컬 트랜잭션)
  8. 발행: shipping.exchange → "order.shipped"
     {orderId, trackingId}

[OrderService]
  구독: "order.shipped"
  9. 주문 상태 → SHIPPED 업데이트

=== 실패 시 보상 트랜잭션 흐름 ===

5단계에서 재고 부족 실패:

[InventoryService]
  5. 재고 부족 감지
  6. 발행: inventory.exchange → "order.inventory.failed"
     {orderId, reason: "INSUFFICIENT_STOCK"}

[PaymentService]
  구독: "order.inventory.failed"
  7. 결제 환불 처리 (보상 트랜잭션)
  8. 발행: payment.exchange → "order.payment.refunded"
     {orderId, paymentId, amount}

[OrderService]
  구독: "order.inventory.failed" + "order.payment.refunded"
  9. 주문 상태 → FAILED 업데이트
  10. 고객 알림 발행

=== Exchange/Queue 구조 ===

order.exchange (Topic):
  "order.placed" → [payment.order.queue, notification.queue]
  "order.shipped" → [order-update.queue, notification.queue]

payment.exchange (Topic):
  "order.payment.completed" → [inventory.payment.queue]
  "order.payment.refunded"  → [order-update.queue, notification.queue]
  "order.inventory.failed"  → [payment.compensation.queue]  ← 보상 구독

inventory.exchange (Topic):
  "order.inventory.reserved" → [shipping.inventory.queue]
  "order.inventory.failed"   → [compensation.queue]
```

### 2. Orchestration Saga — 명령 기반 조율

```
=== Orchestrator 구조 ===

[OrderSagaOrchestrator]
  Saga 상태 DB:
    sagas 테이블: {sagaId, orderId, currentStep, status, createdAt}

정상 흐름:
  1. 주문 접수 → Saga 시작 (status=STARTED)
  2. 발행: saga.commands.exchange → "payment.charge.command"
     {sagaId, orderId, amount}

  [PaymentService]
    구독: payment.command.queue
    3. 결제 처리
    4. 발행 응답: saga.replies.exchange → "payment.charge.reply"
       {sagaId, success: true, paymentId}

  [Orchestrator]
    구독: saga.replies.queue
    5. sagaId로 Saga 상태 조회
    6. 성공 → 다음 단계: 발행 "inventory.reserve.command"

  [InventoryService]
    구독: inventory.command.queue
    7. 재고 예약
    8. 발행 응답: "inventory.reserve.reply"
       {sagaId, success: false, reason: "INSUFFICIENT_STOCK"}

  [Orchestrator]
    9. 실패 응답 수신 → 보상 트랜잭션 시작
    10. 발행: "payment.refund.command" (결제 환불 명령)
    11. PaymentService 환불 처리 → 완료 응답
    12. Saga status = COMPENSATED

=== Exchange/Queue 구조 ===

saga.commands.exchange (Direct):
  "payment.charge"    → payment.command.queue
  "payment.refund"    → payment.command.queue  (동일 Queue, 다른 명령)
  "inventory.reserve" → inventory.command.queue
  "inventory.release" → inventory.command.queue

saga.replies.exchange (Direct):
  "payment.reply"    → orchestrator.reply.queue
  "inventory.reply"  → orchestrator.reply.queue

=== Orchestrator 코드 구조 ===

@Component
public class OrderSagaOrchestrator {

  @RabbitListener(queues = "orchestrator.reply.queue")
  public void handleReply(SagaReply reply, Channel channel,
      @Header(AmqpHeaders.DELIVERY_TAG) long tag) throws IOException {

    SagaState saga = sagaRepository.findById(reply.getSagaId());
    
    try {
      switch (saga.getCurrentStep()) {
        case PAYMENT_PENDING:
          if (reply.isSuccess()) {
            // 다음 단계: 재고 예약
            saga.setCurrentStep(INVENTORY_PENDING);
            sagaRepository.save(saga);
            sendInventoryReserveCommand(saga, reply.getPaymentId());
          } else {
            // 결제 실패 → Saga 실패 (보상 불필요)
            saga.setStatus(FAILED);
            sagaRepository.save(saga);
            notifyOrderFailed(saga);
          }
          break;

        case INVENTORY_PENDING:
          if (reply.isSuccess()) {
            // 다음 단계: 배송 시작
            saga.setCurrentStep(SHIPPING_PENDING);
            sagaRepository.save(saga);
            sendShippingStartCommand(saga);
          } else {
            // 재고 실패 → 결제 환불 보상
            saga.setCurrentStep(COMPENSATING);
            sagaRepository.save(saga);
            sendPaymentRefundCommand(saga);
          }
          break;

        case COMPENSATING:
          // 보상 완료
          saga.setStatus(COMPENSATED);
          sagaRepository.save(saga);
          notifyOrderCancelled(saga);
          break;
      }
      channel.basicAck(tag, false);
      
    } catch (Exception e) {
      log.error("Saga 처리 실패: sagaId={}", reply.getSagaId(), e);
      channel.basicNack(tag, false, false);  // DLQ로 이동
    }
  }
}
```

### 3. Saga 신뢰성 보장

```
=== 메시지 유실 방지 ===

Outbox Pattern + Saga:
  OrderService DB 트랜잭션:
    orders 테이블: INSERT
    outbox 테이블: INSERT {event: "order.placed", sagaId: UUID}

  Outbox Publisher:
    order.placed 이벤트 발행 + Publisher Confirm
    Ack → outbox.status = PUBLISHED

결과: DB 저장 + 이벤트 발행 원자성 보장

=== 중복 처리 방지 (멱등성) ===

각 서비스의 Command 처리:
  // sagaId를 멱등성 키로 사용
  if (processedCommandRepository.existsBySagaIdAndStep(sagaId, "payment.charge")) {
    // 이미 처리됨 → 동일한 응답 재발행
    sendSuccessReply(sagaId, cachedPaymentId);
    channel.basicAck(tag, false);
    return;
  }
  
  // 처리 (로컬 트랜잭션)
  PaymentResult result = paymentService.charge(amount);
  processedCommandRepository.save(sagaId, "payment.charge", result.getPaymentId());
  
  // 응답 발행
  sendReply(sagaId, true, result.getPaymentId());
  channel.basicAck(tag, false);

=== Saga 타임아웃 처리 ===

Orchestrator에서 타임아웃:
  Saga 시작 시 타임아웃 지연 메시지 발행 (예: 10분)
  "10분 후 이 sagaId가 완료 안 됐으면 보상 시작"

  rabbitTemplate.convertAndSend("delayed.exchange", "saga.timeout",
    new SagaTimeoutMessage(sagaId),
    msg -> {
      msg.getMessageProperties().setHeader("x-delay", 600000);  // 10분
      return msg;
    }
  );

  타임아웃 Consumer:
  @RabbitListener(queues = "saga.timeout.queue")
  public void handleTimeout(SagaTimeoutMessage msg) {
    SagaState saga = sagaRepository.findById(msg.getSagaId());
    if (saga.getStatus() != COMPLETED) {
      // 타임아웃 → 보상 트랜잭션 시작
      startCompensation(saga);
    }
    // 이미 완료됐으면 무시
  }
```

---

## 💻 실전 실험

### 실험 1: Choreography Saga Exchange 구성

```bash
# Saga 관련 Exchange 생성
rabbitmqadmin declare exchange name=order.exchange type=topic durable=true
rabbitmqadmin declare exchange name=payment.exchange type=topic durable=true
rabbitmqadmin declare exchange name=inventory.exchange type=topic durable=true

# 각 서비스 Queue 생성
rabbitmqadmin declare queue name=payment.order.queue durable=true
rabbitmqadmin declare queue name=inventory.payment.queue durable=true
rabbitmqadmin declare queue name=payment.compensation.queue durable=true

# 정상 흐름 Binding
rabbitmqadmin declare binding source=order.exchange destination=payment.order.queue routing_key="order.placed"
rabbitmqadmin declare binding source=payment.exchange destination=inventory.payment.queue routing_key="order.payment.completed"

# 보상 Binding
rabbitmqadmin declare binding source=inventory.exchange destination=payment.compensation.queue routing_key="order.inventory.failed"

# 정상 흐름 시뮬레이션
rabbitmqadmin publish exchange=order.exchange routing_key="order.placed" \
  payload='{"orderId":"ord-1","amount":50000}'

rabbitmqctl list_queues name messages
# payment.order.queue  1  ← 결제 서비스 처리 대기

# 결제 완료 시뮬레이션
rabbitmqadmin publish exchange=payment.exchange routing_key="order.payment.completed" \
  payload='{"orderId":"ord-1","paymentId":"pay-1"}'

rabbitmqctl list_queues name messages
# inventory.payment.queue  1  ← 재고 서비스 처리 대기

# 재고 실패 시뮬레이션 (보상 트랜잭션)
rabbitmqadmin publish exchange=inventory.exchange routing_key="order.inventory.failed" \
  payload='{"orderId":"ord-1","reason":"INSUFFICIENT_STOCK"}'

rabbitmqctl list_queues name messages
# payment.compensation.queue  1  ← 결제 환불 처리 대기
```

### 실험 2: Saga 상태 추적 DB 스키마

```sql
-- Saga 상태 테이블
CREATE TABLE order_sagas (
  saga_id       UUID PRIMARY KEY,
  order_id      VARCHAR(50) NOT NULL,
  current_step  VARCHAR(50) NOT NULL,
  status        VARCHAR(20) NOT NULL DEFAULT 'STARTED',
  payment_id    VARCHAR(50),
  inventory_ids JSONB,
  created_at    TIMESTAMP DEFAULT NOW(),
  updated_at    TIMESTAMP DEFAULT NOW()
);

-- 처리된 명령 이력 (멱등성)
CREATE TABLE processed_saga_commands (
  id        UUID PRIMARY KEY,
  saga_id   UUID NOT NULL,
  step      VARCHAR(50) NOT NULL,
  result    JSONB,
  processed_at TIMESTAMP DEFAULT NOW(),
  UNIQUE (saga_id, step)
);

-- 인덱스
CREATE INDEX idx_sagas_status ON order_sagas(status);
CREATE INDEX idx_sagas_order ON order_sagas(order_id);

-- 10분 이상 진행 중인 Saga 조회 (타임아웃 감지)
SELECT * FROM order_sagas
WHERE status = 'STARTED'
  AND current_step != 'COMPLETED'
  AND created_at < NOW() - INTERVAL '10 minutes';
```

---

## 📊 Kafka vs RabbitMQ 비교

```
=== Saga 구현 비교 ===

RabbitMQ Choreography Saga:
  이벤트 → Exchange → 각 서비스 Queue
  각 서비스가 자신의 Queue에서 이벤트 수신
  보상 이벤트도 동일한 메커니즘
  Publisher Confirm + DLX로 신뢰성 보장

Kafka Choreography Saga:
  이벤트 → Topic → 각 서비스 Consumer Group
  각 서비스가 관련 Topic 구독
  실패 → Dead Letter Topic으로 이동
  오프셋 기반으로 재처리 용이

핵심 차이:
  RabbitMQ: 처리 후 메시지 삭제 (재처리 어려움)
  Kafka:    처리 후 메시지 보존 (Saga 재현/디버깅 용이)

=== Saga 이벤트 소싱 ===

Kafka의 장점:
  모든 Saga 이벤트가 Topic에 보존
  "주문 123의 모든 이벤트를 처음부터 재현" 가능
  Saga 상태 복구가 이벤트 스트림에서 가능

RabbitMQ의 장점:
  낮은 지연 (각 서비스가 빠르게 이벤트 수신)
  복잡한 Exchange 라우팅 (보상 이벤트 필터링)
  기존 RabbitMQ 인프라 활용
```

---

## ⚖️ 트레이드오프

```
Choreography vs Orchestration:

Choreography:
  ✅ 중앙 실패 지점 없음 (Orchestrator 없음)
  ✅ 서비스 독립성 높음
  ❌ 이벤트 의존관계 파악 어려움 (코드에 분산)
  ❌ 단계 수가 많으면 거미줄처럼 복잡
  → 서비스 < 5개, 단계 < 5개

Orchestration:
  ✅ Saga 흐름이 Orchestrator에 명시적으로 정의
  ✅ 상태 추적 중앙화 → 모니터링 용이
  ❌ Orchestrator가 단일 실패 지점
  ❌ Orchestrator 서비스 추가 관리 필요
  → 서비스 > 5개, 복잡한 조건 분기

RabbitMQ Saga 신뢰성:
  ✅ Publisher Confirm + DLX → 메시지 보장
  ✅ Outbox Pattern → DB + 메시지 원자성
  ❌ 메시지 처리 후 삭제 → Saga 이벤트 재현 어려움
  → 중요 Saga는 별도 DB에 상태 저장 필수
```

---

## 📌 핵심 정리

```
Saga 패턴 핵심:

개념:
  분산 트랜잭션을 여러 로컬 트랜잭션으로 분해
  실패 시 보상 트랜잭션으로 이전 상태 복원

Choreography:
  서비스가 이벤트를 발행/구독
  중앙 조율자 없음
  RabbitMQ: 이벤트별 Exchange + 서비스별 Queue

Orchestration:
  Orchestrator가 각 서비스에 명령 발행
  응답 수신 → 다음 단계 결정
  RabbitMQ: 명령 Exchange + 응답 Exchange

신뢰성 보장:
  Outbox Pattern: DB + 이벤트 발행 원자성
  Publisher Confirm: 브로커 수신 확인
  멱등성: sagaId로 중복 처리 방지
  DLX: 처리 실패 메시지 보존

Saga 상태:
  DB에 sagaId, 현재 단계, 상태 저장
  타임아웃: Delayed Message로 미완료 감지
```

---

## 🤔 생각해볼 문제

**Q1.** Choreography Saga에서 InventoryService가 재고 예약에 성공했지만, "order.inventory.reserved" 이벤트 발행 직전에 서버가 다운됐다. 이 경우 어떤 일이 발생하는가?

<details>
<summary>해설 보기</summary>

**재고는 예약됐지만 이후 단계(배송)가 진행되지 않는 불일치 상태**가 됩니다.

구체적 상황:
1. InventoryService: 재고 차감 성공 (DB 커밋됨)
2. 서버 다운 → "order.inventory.reserved" 발행 못 함
3. ShippingService: 이벤트 없음 → 배송 시작 안 함
4. 주문이 "재고 예약됨" 상태에서 영원히 멈춤

**해결: Outbox Pattern**
```java
@Transactional
public void reserveInventory(OrderEvent event) {
  inventoryRepository.reserve(event.getItems());  // 재고 차감
  outboxRepository.save(new OutboxEvent(          // 같은 트랜잭션
    "inventory.exchange", "order.inventory.reserved",
    new InventoryReservedEvent(event.getOrderId())
  ));
}
// 서버 다운 전 트랜잭션 커밋 → outbox에 이벤트 저장됨
// 서버 재시작 후 outbox 스케줄러가 이벤트 재발행
```

Outbox 없으면: 재고 차감 + 이벤트 발행 사이의 장애 → 데이터 불일치.
Outbox 있으면: DB 트랜잭션에 이벤트 저장 → 재시작 후 발행 보장.

</details>

---

**Q2.** Orchestration Saga에서 Orchestrator가 InventoryService에 "재고 예약" 명령을 보냈지만 응답이 10분 동안 오지 않는다. 어떻게 처리해야 하는가?

<details>
<summary>해설 보기</summary>

**타임아웃 처리 전략:**

**단계 1: 타임아웃 감지**
```java
// Saga 시작 시 타임아웃 지연 메시지 발행
rabbitTemplate.convertAndSend("delayed.exchange", "saga.timeout",
  new SagaTimeoutMessage(sagaId, "INVENTORY_PENDING"),
  msg -> {
    msg.getMessageProperties().setHeader("x-delay", 600000);  // 10분
    return msg;
  }
);
```

**단계 2: 타임아웃 처리**
```java
@RabbitListener(queues = "saga.timeout.queue")
public void handleTimeout(SagaTimeoutMessage msg) {
  SagaState saga = sagaRepository.findById(msg.getSagaId());
  if (saga.getCurrentStep().equals(msg.getExpectedStep())) {
    // 아직 이 단계 → 타임아웃
    startCompensation(saga);  // 결제 환불 명령 발행
  }
  // 이미 다음 단계로 진행됐으면 무시
}
```

**InventoryService 타임아웃의 두 가능성:**
1. **처리 중 지연**: InventoryService가 느리게 처리 중 → 나중에 응답 도착
2. **처리 실패**: InventoryService가 명령을 받지 못함 (메시지 유실)

타임아웃 후 보상 트랜잭션 시작. InventoryService가 나중에 응답을 보내면 sagaId로 "이미 보상 완료된 Saga"임을 확인 후 무시해야 합니다.

</details>

---

**Q3.** Choreography Saga vs Orchestration Saga 중 어떤 것을 기본으로 선택해야 하고, 언제 전환을 고려해야 하는가?

<details>
<summary>해설 보기</summary>

**기본 선택: Choreography**

이유:
- 설계가 단순 (중앙 Orchestrator 서비스 없음)
- 서비스 독립성 높음 (각 서비스가 자신의 이벤트만 책임)
- MSA 초기 단계에 빠른 개발 가능

**Orchestration 전환 신호:**

1. **이벤트 의존 관계를 파악하기 어려워질 때**
   - "A 서비스가 B 이벤트를 구독하는지 모른다"
   - 새 팀원이 전체 Saga 흐름을 파악하기 어려울 때

2. **조건 분기가 복잡해질 때**
   - "VIP면 빠른 처리, 해외 주문이면 세관 단계 추가"
   - Choreography에서는 조건 분기를 각 서비스에 분산 → 파악 어려움

3. **서비스 수 > 5개, 단계 수 > 5개**
   - 이벤트 의존 그래프가 거미줄처럼 복잡

4. **Saga 진행 상태를 중앙에서 모니터링해야 할 때**
   - "현재 진행 중인 주문 Saga가 어느 단계에 있는가?"
   - Orchestration: Orchestrator DB에서 단일 조회
   - Choreography: 각 서비스의 로그/이벤트를 모아서 파악

**실용적 조언:**
- 작은 팀, 초기 단계: Choreography
- 팀 5개 이상, Saga 복잡: Orchestration (Temporal.io 같은 워크플로 엔진 고려)

</details>

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[이전: Delayed Message ⬅️](./05-delayed-message.md)** | **[다음: Chapter 5 — 처리량 최적화 ➡️](../performance-operations/01-throughput-optimization.md)**

</div>
