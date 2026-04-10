# Direct Exchange — Routing Key 정확 매칭 라우팅

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Direct Exchange는 어떤 기준으로 메시지를 Queue에 라우팅하는가?
- 1:1 라우팅과 1:N 라우팅(같은 Routing Key에 여러 Queue)은 어떻게 동작하는가?
- 하나의 Queue가 같은 Exchange에 여러 Binding Key로 연결될 수 있는가?
- Default Exchange는 Direct Exchange의 특수 형태인가?
- Direct Exchange가 적합한 경우와 Topic Exchange로 전환해야 하는 경우는?

---

## 🔍 왜 이 개념이 실무에서 중요한가

Direct Exchange는 가장 단순하고 가장 많이 쓰이는 Exchange 유형이다. "특정 서비스에게만 메시지를 전달"하는 요구사항이 있을 때 직접 설계하는 첫 번째 선택지다. 단순해 보이지만, 1:N 라우팅과 다중 Binding Key를 이해하지 못하면 Exchange를 불필요하게 많이 만들거나, 반대로 라우팅이 꼬이는 설계를 만든다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: 서비스마다 Exchange를 별도 생성

  order-service → order-to-payment.exchange → payment-queue
  order-service → order-to-inventory.exchange → inventory-queue
  order-service → order-to-notification.exchange → notification-queue

  문제:
    Exchange 3개, 발행 코드 3벌
    새 서비스 추가 → 새 Exchange 추가 + 발행 코드 수정
    Exchange 관리 복잡성 폭발

  올바른 설계:
    order-service → order.exchange (Direct) → payment-queue (key="payment")
                                            → inventory-queue (key="inventory")
                                            → notification-queue (key="notification")
    Exchange 1개, 발행 시 routingKey만 변경

실수 2: Direct Exchange에서 패턴 매칭 기대

  Binding Key: "order.payment"
  발행 Routing Key: "order.payment.failed"  ← Direct에서는 정확히 일치해야 함

  결과:
    매칭 실패 → 메시지 폐기 (mandatory=false 시)
    에러 없음, 조용한 유실

  해결: 패턴 필요 시 Topic Exchange로 전환

실수 3: 같은 Exchange에 같은 Queue를 중복 Binding

  // Spring AMQP @Bean 선언 시 실수로 같은 Binding 두 번
  @Bean Binding binding1() { return BindingBuilder.bind(q).to(ex).with("order"); }
  @Bean Binding binding2() { return BindingBuilder.bind(q).to(ex).with("order"); }  // 중복

  결과:
    중복 Binding은 멱등적으로 처리됨 (에러 없음)
    실제로는 하나의 Binding만 존재
    의도치 않게 중복 선언 → 설계 의도 불명확
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계)

```
Direct Exchange 설계 원칙:

1. 서비스별 라우팅: 하나의 Exchange + 서비스별 다른 Routing Key

  order.exchange (Direct)
  ├── routingKey="payment"    → payment.queue
  ├── routingKey="inventory"  → inventory.queue
  └── routingKey="notify"     → notification.queue

2. 이벤트 유형별 라우팅:

  order.exchange (Direct)
  ├── routingKey="order.placed"    → order.placed.queue
  ├── routingKey="order.cancelled" → order.cancelled.queue
  └── routingKey="order.shipped"   → order.shipped.queue

3. Spring AMQP 설정:

  @Bean
  public DirectExchange orderExchange() {
    return ExchangeBuilder
      .directExchange("order.exchange")
      .durable(true)
      .build();
  }

  @Bean
  public Binding paymentBinding() {
    return BindingBuilder
      .bind(paymentQueue())
      .to(orderExchange())
      .with("payment");
  }

  // 발행
  rabbitTemplate.convertAndSend("order.exchange", "payment", message);
```

---

## 🔬 내부 동작 원리

### 1. Direct Exchange 라우팅 알고리즘

```
=== 정확 매칭(Exact Match) 알고리즘 ===

Direct Exchange의 라우팅 로직:
  Routing Key == Binding Key → 전달
  Routing Key != Binding Key → 전달 안 함

예시:

  Exchange: order.exchange (type=direct)
  Binding 테이블:
  ┌──────────────────┬──────────────────────┐
  │ Binding Key      │ Queue                │
  ├──────────────────┼──────────────────────┤
  │ "payment"        │ payment.queue        │
  │ "inventory"      │ inventory.queue      │
  │ "notification"   │ notification.queue   │
  │ "payment"        │ payment-audit.queue  │ ← 같은 Key, 다른 Queue
  └──────────────────┴──────────────────────┘

  발행: routingKey="payment"
    "payment" == "payment" → payment.queue ✅
    "payment" == "payment" → payment-audit.queue ✅ (같은 Key, 두 Queue 모두 전달)
    "payment" == "inventory" → ❌
    "payment" == "notification" → ❌

  발행: routingKey="inventory"
    → inventory.queue만 전달

=== 1:N 라우팅 — 같은 Key, 여러 Queue ===

  같은 Binding Key → 여러 Queue:
  "payment" → payment.queue
  "payment" → payment-audit.queue

  → routingKey="payment" 발행 시 두 Queue에 복사 전달

  사용 사례:
  - 메인 처리 Queue + 감사 로그 Queue
  - 메인 처리 Queue + 모니터링 Queue

=== 하나의 Queue, 여러 Binding Key ===

  하나의 Queue에 여러 Binding:
  "order.placed"    → order.all.queue
  "order.cancelled" → order.all.queue
  "order.shipped"   → order.all.queue

  → 어느 Key로 발행해도 order.all.queue에 도달 (OR 조건)
```

### 2. Default Exchange와의 관계

```
=== Default Exchange는 특수한 Direct Exchange ===

Default Exchange (exchange=""):
  type=direct
  이름: 빈 문자열 ""
  모든 Queue에 자동 Binding (Key = Queue 이름)

  발행: exchange="", routingKey="my-queue"
  = Direct Exchange에서 routingKey="my-queue"로 발행한 것과 동일

차이:
  Default Exchange: 삭제/수정 불가, 자동 Binding
  일반 Direct Exchange: 생성/삭제 가능, 수동 Binding
  
  실무:
  Default Exchange → 테스트, 단순 작업 큐
  명시적 Direct Exchange → 라우팅 설계가 있는 서비스

=== Direct Exchange vs Topic Exchange 선택 ===

Direct Exchange 적합:
  "결제 서비스에게만 보낸다" → routingKey="payment"
  이벤트 유형이 명확히 고정되어 있음
  패턴 매칭 필요 없음

Topic Exchange로 전환 시점:
  "order.*.failed 패턴의 이벤트를 모두 수집하고 싶다"
  "실패 관련 이벤트는 모니터링 서비스로 보내고 싶다"
  Routing Key가 계층적 구조를 가질 때
```

### 3. Direct Exchange 처리 흐름 (AMQP 레벨)

```
=== AMQP 프레임 레벨 라우팅 ===

Producer:
  Channel.basicPublish(
    exchange="order.exchange",
    routingKey="payment",
    mandatory=false,
    props={deliveryMode=2},
    body={...}
  )

RabbitMQ 수신:
  1. "order.exchange" Exchange 조회
  2. "payment" Key로 Binding 테이블 조회 (O(1) 해시 조회)
  3. 매칭된 Queue: payment.queue, payment-audit.queue
  4. 각 Queue에 메시지 복사 전달
  5. mandatory=true이고 매칭 없으면 Basic.Return 발송

Queue에서 Consumer로:
  payment.queue → Consumer A (basicConsume)
  payment-audit.queue → Consumer B (별도 Consumer)

성능:
  Direct Exchange 라우팅: O(1) 해시 테이블 조회
  Topic Exchange 라우팅: O(B) — Binding 수에 비례한 패턴 매칭
  → Direct Exchange가 가장 빠른 라우팅
```

---

## 💻 실전 실험

### 실험 1: Direct Exchange 1:1, 1:N 동작 확인

```bash
# Exchange 생성
rabbitmqadmin declare exchange name=order.exchange type=direct durable=true

# Queue 생성
rabbitmqadmin declare queue name=payment.queue durable=true
rabbitmqadmin declare queue name=payment-audit.queue durable=true
rabbitmqadmin declare queue name=inventory.queue durable=true

# Binding 생성
rabbitmqadmin declare binding source=order.exchange \
  destination=payment.queue routing_key=payment
rabbitmqadmin declare binding source=order.exchange \
  destination=payment-audit.queue routing_key=payment  # 같은 Key, 다른 Queue
rabbitmqadmin declare binding source=order.exchange \
  destination=inventory.queue routing_key=inventory

# routingKey="payment" 발행
rabbitmqadmin publish exchange=order.exchange routing_key=payment \
  payload='{"orderId":1,"amount":10000}'

# 결과 확인
rabbitmqctl list_queues name messages
# payment.queue        1  ← 도달
# payment-audit.queue  1  ← 도달 (1:N 라우팅)
# inventory.queue      0  ← 미도달

# routingKey="inventory" 발행
rabbitmqadmin publish exchange=order.exchange routing_key=inventory \
  payload='{"orderId":1,"item":"book"}'

rabbitmqctl list_queues name messages
# inventory.queue  1  ← 도달
```

### 실험 2: Spring AMQP Direct Exchange 설정

```java
@Configuration
public class DirectExchangeConfig {

  @Bean
  public DirectExchange orderExchange() {
    return ExchangeBuilder
      .directExchange("order.exchange")
      .durable(true)
      .build();
  }

  @Bean public Queue paymentQueue() {
    return QueueBuilder.durable("payment.queue").build();
  }
  @Bean public Queue inventoryQueue() {
    return QueueBuilder.durable("inventory.queue").build();
  }

  @Bean
  public Binding paymentBinding(Queue paymentQueue, DirectExchange orderExchange) {
    return BindingBuilder.bind(paymentQueue).to(orderExchange).with("payment");
  }

  @Bean
  public Binding inventoryBinding(Queue inventoryQueue, DirectExchange orderExchange) {
    return BindingBuilder.bind(inventoryQueue).to(orderExchange).with("inventory");
  }
}

// 발행
@Service
public class OrderEventPublisher {

  @Autowired private RabbitTemplate rabbitTemplate;

  public void publishPaymentEvent(PaymentMessage msg) {
    rabbitTemplate.convertAndSend("order.exchange", "payment", msg);
  }

  public void publishInventoryEvent(InventoryMessage msg) {
    rabbitTemplate.convertAndSend("order.exchange", "inventory", msg);
  }
}

// 소비
@RabbitListener(queues = "payment.queue")
public void handlePayment(PaymentMessage msg) {
  // 결제 처리
}

@RabbitListener(queues = "inventory.queue")
public void handleInventory(InventoryMessage msg) {
  // 재고 처리
}
```

### 실험 3: Routing Key 불일치 시 메시지 폐기 확인

```bash
# Binding 없는 Key로 발행
rabbitmqadmin publish exchange=order.exchange routing_key=unknown_key \
  payload='{"orderId":2}'

rabbitmqctl list_queues name messages
# 모든 Queue 변화 없음 ← 조용히 폐기

# Management UI에서 확인
# Exchanges → order.exchange → Incoming / Outgoing 차트
# Incoming: 1 (Exchange 진입)
# Outgoing: 0 (Queue로 나가지 않음)
# → 폐기된 메시지가 차트에서 보임
```

---

## 📊 Kafka vs RabbitMQ 비교

```
=== 특정 서비스에게 메시지 전달 ===

RabbitMQ Direct Exchange:
  Producer: exchange="order.exchange", routingKey="payment"
  Exchange → Binding 평가 → payment.queue
  Consumer: payment.queue 구독
  
  특징: 브로커가 라우팅 결정 (Smart Broker)
       새 서비스 추가 → Binding 추가만 (Producer 변경 없음)

Kafka Topic:
  Producer: topic="payment-events" (또는 "order-events" + key="payment")
  Consumer Group: "payment-group"이 "payment-events" 구독
  
  특징: 라우팅은 Topic 이름으로 결정 (Producer가 결정)
       새 서비스 추가 → 새 Consumer Group이 Topic 구독
       과거 메시지부터 소비 가능

=== 선택 기준 ===

"결제 서비스에게만 특정 메시지를 보내고 싶다":
  둘 다 가능
  RabbitMQ: Exchange에 Binding 설정
  Kafka: 전용 Topic 사용 또는 filter

"결제 + 감사팀이 동시에 같은 메시지를 받아야 한다":
  RabbitMQ: 같은 Key에 두 Queue Binding (메시지 복사)
  Kafka: 두 Consumer Group이 동일 Topic 구독 (메시지 공유, 복사 없음)
  → Kafka가 더 효율적 (복사본 없이 독립 소비)
```

---

## ⚖️ 트레이드오프

```
Direct Exchange의 장단점:

장점:
  ① 가장 빠른 라우팅 (O(1) 해시 조회)
  ② 단순한 설정, 이해하기 쉬움
  ③ 명확한 의도 (정확한 서비스 지정)

단점:
  ① 패턴 매칭 불가 (유연성 낮음)
  ② 새 Routing Key 추가 시 Binding 수동 추가 필요
  ③ Routing Key 오타 → 조용한 메시지 폐기

Direct → Topic으로 전환 시점:
  Routing Key가 5개 이상이고 패턴이 보인다면
  "order.payment.*", "order.inventory.*" 패턴이 생긴다면
  → Topic Exchange로 전환이 더 유연
```

---

## 📌 핵심 정리

```
Direct Exchange 핵심:

라우팅 규칙:
  Routing Key == Binding Key → 해당 Queue로 전달
  정확한 문자열 일치 (대소문자 구분)

1:N 라우팅:
  같은 Binding Key → 여러 Queue
  → 두 Queue에 메시지 복사 전달

N:1 라우팅:
  여러 Binding Key → 같은 Queue
  → OR 조건으로 여러 Key의 메시지를 하나의 Queue에 수집

Default Exchange:
  Direct Exchange의 특수형
  Queue 이름 = Binding Key (자동)

성능:
  4가지 Exchange 유형 중 가장 빠름 (O(1))
  
적합한 사용 사례:
  특정 서비스/역할에게 메시지 전달
  이벤트 유형별 Queue 분리 (확정된 유형)
```

---

## 🤔 생각해볼 문제

**Q1.** Direct Exchange에서 Binding Key에 공백이나 특수문자를 사용할 수 있는가? `routingKey="order-placed"` vs `routingKey="order.placed"` 중 Direct Exchange에서 어떤 것이 더 적합한가?

<details>
<summary>해설 보기</summary>

Routing Key/Binding Key는 UTF-8 문자열로 최대 255바이트까지 허용됩니다. 공백, 특수문자, 점(`.`) 모두 사용 가능합니다.

**Direct Exchange에서는 `"order-placed"`가 더 적합합니다.**

이유:
- Direct는 정확 매칭이므로 점(`.`)의 계층적 의미가 없습니다
- `"order.placed"` 형식은 Topic Exchange의 와일드카드(`order.*`, `order.#`) 활용을 전제한 네이밍
- Direct에서 `"order.placed"`를 쓰면 나중에 Topic으로 전환 시 혼동 가능

컨벤션 권장:
- Direct Exchange: 단순 동사/명사 (`"payment"`, `"inventory"`)
- Topic Exchange: 계층적 구조 (`"order.payment.failed"`)

</details>

---

**Q2.** Direct Exchange에 Binding된 Queue를 삭제하면 Binding도 자동으로 삭제되는가? 이 상태에서 해당 Routing Key로 메시지를 발행하면 어떻게 되는가?

<details>
<summary>해설 보기</summary>

**Queue 삭제 시 해당 Queue와 연결된 Binding도 자동으로 삭제됩니다.**

이후 동작:
- Exchange는 살아있지만 해당 Key에 매칭되는 Binding 없음
- `mandatory=false`: 메시지 조용히 폐기
- `mandatory=true`: `Basic.Return` (312 NO_ROUTE) 반환

실무 위험:
배포 중 Queue를 재생성하는 경우, 순간적으로 Binding이 없는 상태가 발생. 이 사이에 발행된 메시지는 `mandatory=false`이면 유실됩니다.

안전한 배포 순서:
1. 새 Queue 생성 + Binding 생성
2. Consumer 배포
3. 구 Queue 삭제 (Binding 자동 삭제)

</details>

---

**Q3.** `routingKey=""`(빈 문자열)로 Direct Exchange에 발행하면 어떤 Queue로 라우팅되는가?

<details>
<summary>해설 보기</summary>

**Binding Key가 `""`(빈 문자열)인 Queue로 라우팅됩니다.**

Direct는 정확 매칭이므로 `routingKey=""`는 `bindingKey=""`인 Binding에만 매칭됩니다.

이것을 활용하면:
```bash
rabbitmqadmin declare binding source=my.exchange \
  destination=catch-all.queue routing_key=""
```
빈 Routing Key로 발행하는 메시지를 수집하는 "catch-all" Queue를 만들 수 있습니다.

단, 빈 Routing Key로 발행하는 것은 의미가 불명확해서 실무에서는 권장하지 않습니다. 이 용도로는 Alternate Exchange 패턴이 더 명확합니다.

</details>

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[다음: Topic Exchange ➡️](./02-topic-exchange.md)**

</div>
