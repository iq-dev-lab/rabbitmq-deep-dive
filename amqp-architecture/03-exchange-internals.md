# Exchange 완전 분해 — 라우팅 결정자의 내부 구조

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Exchange는 메시지를 어떻게 받아서 어느 Queue로 보낼지 결정하는가?
- Binding이란 무엇이고, Binding Key는 어떻게 라우팅 결정에 사용되는가?
- Default Exchange(이름 없는 Exchange)는 왜 존재하고, 어떻게 동작하는가?
- Exchange에 Binding된 Queue가 없거나, 일치하는 Binding이 없으면 메시지는 어떻게 되는가?
- Exchange 자체가 메시지를 저장하지 않는다는 것의 실무적 의미는 무엇인가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

Exchange를 모르면 메시지가 어디로 가는지 설명할 수 없다. "Queue에 직접 발행한다"고 생각하고 코드를 짜다가, 메시지가 사라지는 상황을 디버깅할 수 없게 된다. 특히 Default Exchange의 동작 방식을 모르면 간단한 예제 코드가 왜 동작하는지 이해 못하고, 실제 서비스 설계에서 Exchange 구조를 엉뚱하게 구성하게 된다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: "메시지가 사라졌다" — Binding 없는 Exchange에 발행

  rabbitTemplate.convertAndSend("my.exchange", "some.key", message);
  // Exchange는 있는데 "some.key"에 매칭되는 Binding이 없음
  
  결과:
    메시지가 Exchange까지는 도달
    Exchange가 평가: "이 Routing Key에 매칭되는 Binding 없음"
    → 메시지를 Queue로 전달하지 않고 폐기(Drop)
    → 에러 없음, 예외 없음, 로그 없음
    → "분명히 발행했는데 Queue에 없다"

  원인:
    mandatory=false (기본값) → 라우팅 불가 시 조용히 폐기
    mandatory=true → 라우팅 불가 시 Publisher에게 Return

실수 2: "Default Exchange가 뭔지 모르고 Queue 이름으로 직접 발행"

  // 이게 왜 동작하는지 모름
  rabbitTemplate.convertAndSend("myQueue", message);
  // exchange=""(Default), routingKey="myQueue"
  
  실제 동작:
    exchange="" (Default Exchange) 사용
    routingKey = Queue 이름 "myQueue"
    → Default Exchange는 Queue 이름을 Routing Key로 직접 Queue에 전달
    
  문제:
    "Default Exchange는 모든 Queue에 자동 Binding됨"을 몰라서
    실제 서비스에서도 Default Exchange에 의존 → Exchange 설계 없음
    → 라우팅 유연성 0, 구독자 추가 불가

실수 3: Exchange 삭제 시 메시지 유실

  rabbitAdmin.deleteExchange("order.exchange");
  // Exchange는 메시지를 저장하지 않지만...
  // Consumer가 없어서 Queue에 쌓인 메시지는?
  
  실제 문제:
    Exchange 삭제 → 새 메시지 발행 불가
    이미 Queue에 쌓인 메시지는 유지 (Exchange와 무관)
    → 하지만 Exchange 없으면 새 메시지 라우팅 불가
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계)

```
올바른 Exchange 설계:

1. mandatory=true + ReturnCallback으로 라우팅 실패 감지

  rabbitTemplate.setMandatory(true);
  rabbitTemplate.setReturnsCallback(returned -> {
    log.error("라우팅 실패: exchange={}, routingKey={}, replyCode={}",
      returned.getExchange(),
      returned.getRoutingKey(),
      returned.getReplyCode());
    // 재처리 또는 알림
  });

2. Exchange 유형 명시적 선택

  @Bean
  public TopicExchange orderExchange() {
    return ExchangeBuilder
      .topicExchange("order.exchange")
      .durable(true)    // 브로커 재시작 후에도 유지
      .build();
  }
  
  @Bean
  public Binding orderPlacedBinding() {
    return BindingBuilder
      .bind(orderNotificationQueue())
      .to(orderExchange())
      .with("order.placed.*");  // Routing Key 패턴
  }

3. Exchange 타입별 적합한 사용 사례 선택

  서비스 간 특정 라우팅  → Direct Exchange (정확한 Key 매칭)
  이벤트 계층 라우팅     → Topic Exchange (와일드카드 패턴)
  모든 서비스 브로드캐스트 → Fanout Exchange (Key 무시)
  헤더 조건 라우팅       → Headers Exchange (헤더 기반)
```

---

## 🔬 내부 동작 원리

### 1. Exchange의 역할 — 메시지 라우터

```
=== Exchange는 메시지를 저장하지 않는다 ===

Exchange의 역할:
  "메시지를 받아서 어느 Queue로 보낼지 결정한다"
  결정 후 즉시 Queue로 전달하거나 폐기
  자체적으로 메시지를 보관하지 않음
  
  Exchange = 도로 교차로의 신호등
  Queue = 메시지가 실제로 쌓이는 창고

=== Exchange의 내부 처리 흐름 ===

Producer: 메시지 발행
  exchange: "order.exchange"
  routing_key: "order.payment.failed"
  body: {...}
  
  ↓ RabbitMQ 수신
  
Exchange 처리 (order.exchange, type=topic):
  1. Binding 테이블 조회:
     ┌────────────────────────┬──────────────────┬──────────────┐
     │ Exchange               │ Binding Key      │ Queue        │
     ├────────────────────────┼──────────────────┼──────────────┤
     │ order.exchange         │ order.#          │ payment-queue│
     │ order.exchange         │ order.payment.*  │ audit-queue  │
     │ order.exchange         │ order.shipped    │ ship-queue   │
     │ order.exchange         │ #.failed         │ alert-queue  │
     └────────────────────────┴──────────────────┴──────────────┘
  
  2. Routing Key "order.payment.failed" 매칭 평가:
     "order.#"          → ✅ ("order" 뒤 아무거나) → payment-queue
     "order.payment.*"  → ✅ ("order.payment" 뒤 단어 1개) → audit-queue
     "order.shipped"    → ❌ (정확히 "order.shipped"여야 함)
     "#.failed"         → ✅ (아무거나 뒤에 "failed") → alert-queue
  
  3. 매칭된 Queue로 메시지 복사 전달:
     payment-queue → 메시지 복사본 저장
     audit-queue   → 메시지 복사본 저장
     alert-queue   → 메시지 복사본 저장
  
  4. Exchange의 메시지 참조 해제 (저장 안 함)

핵심:
  같은 메시지가 여러 Queue에 "복사"됨
  각 Queue의 메시지는 독립적 (한 Queue에서 삭제해도 다른 Queue는 유지)
```

### 2. Binding — Exchange와 Queue를 연결하는 규칙

```
=== Binding의 구성 요소 ===

Binding = (Exchange, Queue, Binding Key, Arguments)

  Exchange: 어느 Exchange에서 오는 메시지를 필터링하는가
  Queue: 필터링된 메시지를 어느 Queue로 보내는가
  Binding Key: (Exchange 유형에 따라 사용 방식이 다름)
    Direct/Topic: Routing Key와 비교하는 패턴
    Fanout: 무시 (모든 메시지 전달)
    Headers: 무시 (Arguments의 헤더 조건 사용)
  Arguments: 추가 속성 (Headers Exchange의 헤더 조건 등)

=== Binding 생성과 삭제 ===

CLI로 Binding 생성:
  rabbitmqadmin declare binding \
    source=order.exchange \
    destination=audit-queue \
    routing_key="order.#"

Spring AMQP:
  @Bean
  public Binding binding() {
    return BindingBuilder
      .bind(auditQueue())
      .to(orderExchange())
      .with("order.#");
  }
  
  // RabbitAdmin이 앱 시작 시 자동 선언
  // 이미 존재하면 멱등적으로 처리 (오류 없음)

=== Queue가 Exchange에 여러 Binding 가능 ===

같은 Queue, 다른 Exchange:
  audit-queue ← order.exchange, "order.#"
  audit-queue ← payment.exchange, "payment.#"
  → audit-queue는 두 Exchange에서 메시지 수신 가능

같은 Exchange, 다른 Binding Key:
  audit-queue ← order.exchange, "order.placed"
  audit-queue ← order.exchange, "order.cancelled"
  → "order.placed" OR "order.cancelled" 모두 수신

같은 Exchange, 같은 Queue, 같은 Key:
  중복 Binding은 무시 (멱등적)
```

### 3. Default Exchange — 이름 없는 특수 Exchange

```
=== Default Exchange 정의 ===

RabbitMQ의 모든 Queue는 Default Exchange에 자동으로 Binding됨
Binding Key = Queue 이름 (자동)
삭제 불가, 타입 변경 불가

=== Default Exchange의 동작 ===

Queue 선언:
  rabbitmqctl declare queue name=my-queue durable=true
  → 자동으로 Binding 생성: (""(default), "my-queue", "my-queue")

메시지 발행 (Default Exchange 사용):
  exchange=""  (빈 문자열)
  routing_key="my-queue"  (Queue 이름과 동일)
  
  → Default Exchange: "my-queue" Queue에 직접 전달

Spring AMQP:
  rabbitTemplate.convertAndSend("my-queue", message);
  // exchange=""가 내부적으로 사용됨
  
  rabbitTemplate.convertAndSend("", "my-queue", message);
  // 명시적으로 동일한 동작

=== Default Exchange의 용도와 한계 ===

용도:
  간단한 1:1 Queue 발행
  테스트, 로컬 개발 환경
  단순 Worker Queue (Work Queue 패턴)

한계:
  하나의 메시지를 여러 Queue로 분배 불가
  Exchange 유형별 라우팅 기능 사용 불가
  라우팅 키로 패턴 매칭 불가 (정확한 Queue 이름만)
  → 실제 서비스에서는 명시적 Exchange 선언 권장

=== Queue 이름 = Routing Key 설계의 함정 ===

Default Exchange 의존 시 문제:
  OrderService: rabbitTemplate.convertAndSend("notification-queue", msg)
  → 나중에 "email-queue", "sms-queue"로 분리하고 싶으면?
  → OrderService 코드 수정 필요

Exchange 사용 시:
  OrderService: rabbitTemplate.convertAndSend("order.exchange", "order.placed", msg)
  → Exchange에 Binding만 추가: order.exchange → email-queue, sms-queue
  → OrderService 코드 변경 없음
```

### 4. 라우팅 불가 메시지 — mandatory와 alternate-exchange

```
=== Routing Key 매칭 실패 시 시나리오 ===

시나리오 1: mandatory=false (기본값)
  Exchange가 매칭 Binding 없음 → 메시지 조용히 폐기(Drop)
  Producer는 이 사실을 알 수 없음
  → 메시지 유실, 에러 없음

시나리오 2: mandatory=true
  Exchange가 매칭 Binding 없음 → Basic.Return 프레임 발송
  Producer에게 메시지가 반환됨
  Spring AMQP: ReturnsCallback 호출
  → 유실 감지 가능, 재처리 또는 알림

시나리오 3: Alternate Exchange (AE) 설정
  Primary Exchange가 라우팅 실패 시 AE로 메시지 전달
  AE를 Fanout으로 설정 → 모든 라우팅 실패 메시지를 Dead Letter Queue에 수집
  
  Exchange 선언 시 AE 지정:
  rabbitAdmin.declareExchange(
    ExchangeBuilder.topicExchange("order.exchange")
      .durable(true)
      .alternate("order.exchange.dlq")  // AE 지정
      .build()
  );

=== AE를 활용한 라우팅 실패 수집 패턴 ===

order.exchange (Topic, AE=order.unrouted)
  ├── "order.placed" → order.notification.queue
  ├── "order.cancelled" → order.cancel.queue
  └── 그 외 → order.unrouted Exchange (AE)

order.unrouted (Fanout)
  └── "unrouted" → dlq.order.queue (모니터링, 재처리)

이점:
  individual mandatory 설정 없이도 전체 Exchange 레벨에서 라우팅 실패 수집
  라우팅 실패 메시지를 별도 Queue에서 분석 가능
```

---

## 💻 실전 실험

### 실험 1: Exchange 생성과 Binding 구성

```bash
# Exchange 생성
rabbitmqadmin declare exchange \
  name=order.exchange \
  type=topic \
  durable=true

# Queue 생성
rabbitmqadmin declare queue name=order.payment.queue durable=true
rabbitmqadmin declare queue name=order.audit.queue durable=true
rabbitmqadmin declare queue name=order.alert.queue durable=true

# Binding 생성
rabbitmqadmin declare binding \
  source=order.exchange \
  destination=order.payment.queue \
  routing_key="order.#"

rabbitmqadmin declare binding \
  source=order.exchange \
  destination=order.audit.queue \
  routing_key="order.payment.*"

rabbitmqadmin declare binding \
  source=order.exchange \
  destination=order.alert.queue \
  routing_key="#.failed"

# Binding 목록 확인
rabbitmqadmin list bindings
```

### 실험 2: 라우팅 결과 확인

```bash
# 메시지 발행: "order.payment.failed"
rabbitmqadmin publish \
  exchange=order.exchange \
  routing_key="order.payment.failed" \
  payload='{"orderId": 1}'

# 각 Queue에 메시지가 도달했는지 확인
rabbitmqctl list_queues name messages
# order.payment.queue  1   ← 도달 (order.# 매칭)
# order.audit.queue    1   ← 도달 (order.payment.* 매칭)
# order.alert.queue    1   ← 도달 (#.failed 매칭)

# 메시지 발행: "order.shipped" (일부만 매칭)
rabbitmqadmin publish \
  exchange=order.exchange \
  routing_key="order.shipped" \
  payload='{"orderId": 2}'

rabbitmqctl list_queues name messages
# order.payment.queue  2   ← 도달 (order.# 매칭)
# order.audit.queue    1   ← 미도달 (order.payment.*는 "payment" 필요)
# order.alert.queue    1   ← 미도달 (#.failed는 "failed" 필요)
```

### 실험 3: Default Exchange 동작 확인

```bash
# Queue만 선언 (Default Exchange Binding 자동 생성)
rabbitmqadmin declare queue name=direct-test durable=true

# Default Exchange로 발행 (exchange="" 사용)
rabbitmqadmin publish \
  exchange="" \
  routing_key="direct-test" \
  payload="direct message"

# Binding 목록에서 Default Exchange 확인
rabbitmqadmin list bindings
# source=""  destination="direct-test"  routing_key="direct-test"
# ← Default Exchange가 자동으로 Binding 생성됨
```

### 실험 4: mandatory=true 동작 확인

```bash
# Binding이 없는 Exchange에 발행
rabbitmqadmin declare exchange name=empty.exchange type=direct durable=true
# Binding 생성 없음

# Management UI → Exchanges → empty.exchange → Bindings
# → 없음

# mandatory=true로 발행 시 Return 발생
# Spring AMQP 코드:
# rabbitTemplate.setMandatory(true);
# rabbitTemplate.setReturnsCallback(returned -> 
#   log.warn("Returned: {}", returned.getMessage()));
# rabbitTemplate.convertAndSend("empty.exchange", "any.key", "test");
# → ReturnsCallback 호출됨 (312 NO_ROUTE 응답)
```

---

## 📊 Kafka vs RabbitMQ 비교

```
=== 라우팅 레이어 비교 ===

RabbitMQ Exchange:
  Producer: exchange="order.exchange", routingKey="order.payment.failed"
  Exchange → Binding 평가 → Queue들로 분배
  
  특징:
  - 라우팅 로직이 브로커(RabbitMQ)에 있음
  - Consumer는 Queue에서 꺼내기만 함
  - 새 Queue/Binding 추가 → Producer 변경 없음
  - "Smart Broker" 방식

Kafka Topic/Partition:
  Producer: topic="order-events", key="payment" (선택적)
  Partition 결정: hash(key) % partitionCount 또는 Round-Robin
  
  특징:
  - 라우팅 로직이 Producer에 있음 (어느 Topic에 발행할지)
  - Consumer Group이 Partition을 나눠 소비
  - 새 Consumer Group 추가 → 독립적으로 오프셋부터 소비 가능
  - "Dumb Broker" 방식

=== 같은 요구사항, 다른 접근 ===

요구사항: "결제 실패 이벤트를 알림팀, 분석팀, 감사팀이 처리"

RabbitMQ:
  Producer → order.exchange, "order.payment.failed"
  Binding:
    order.exchange → notification.queue (알림팀 소비)
    order.exchange → analytics.queue   (분석팀 소비)
    order.exchange → audit.queue       (감사팀 소비)
  새 팀 추가: Binding 하나 추가, Producer 변경 없음

Kafka:
  Producer → "order-events" Topic
  Consumer Group A (알림팀): offset 독립 추적
  Consumer Group B (분석팀): offset 독립 추적
  Consumer Group C (감사팀): offset 독립 추적
  새 팀 추가: Consumer Group 생성, 과거 메시지부터 처리 가능
```

---

## ⚖️ 트레이드오프

```
Exchange 3단계 라우팅의 장단점:

장점:
  ① 유연한 라우팅 — 4가지 Exchange 유형으로 다양한 패턴
  ② Producer 독립성 — Queue 구조 변경 시 Producer 코드 무관
  ③ Binding 변경 용이 — 런타임에 Binding 추가/삭제 가능
  ④ 브로드캐스트 지원 — 하나의 메시지를 여러 Queue에 복사

단점:
  ① 설정 복잡성 — Exchange, Queue, Binding 각각 관리
  ② 라우팅 실패 가능성 — Binding 없으면 메시지 폐기 (mandatory=false 시)
  ③ 브로커 부하 — 복잡한 라우팅일수록 브로커 CPU 사용 증가
  ④ 가시성 낮음 — "이 메시지가 어디로 갔는가" 추적이 어려울 수 있음

Exchange 타입 선택 기준:
  "항상 특정 Queue로만" → Direct
  "계층적 이벤트 분류" → Topic
  "모든 구독자에게 동일하게" → Fanout
  "조건이 Routing Key 이상의 표현력 필요" → Headers
```

---

## 📌 핵심 정리

```
Exchange 핵심:

역할:
  메시지를 받아 Binding 규칙 평가 → Queue로 전달
  메시지를 자체 저장하지 않음 (라우터)

Binding:
  Exchange + Queue + Binding Key (or Arguments)
  Exchange 유형에 따라 Binding Key 사용 방식 다름

Default Exchange:
  이름 없는 Exchange (exchange="")
  모든 Queue가 자동으로 Binding (Key = Queue 이름)
  단순 테스트/개발에 편리, 실서비스에서는 명시적 Exchange 권장

라우팅 실패 시:
  mandatory=false → 조용히 폐기 (기본값, 위험)
  mandatory=true  → Producer에게 Return
  Alternate Exchange → 별도 Queue에 수집

Exchange 유형:
  Direct → Routing Key 정확 매칭
  Topic  → 와일드카드 패턴 (* = 단어 하나, # = 여러 단어)
  Fanout → 모든 Binding된 Queue에 복사
  Headers → 메시지 헤더 조건 매칭
```

---

## 🤔 생각해볼 문제

**Q1.** `exchange=""` (Default Exchange)에 발행할 때 Routing Key를 존재하지 않는 Queue 이름으로 설정하면 어떻게 되는가? mandatory=false일 때와 mandatory=true일 때 각각?

<details>
<summary>해설 보기</summary>

Default Exchange는 선언된 Queue와 1:1로 자동 Binding됩니다. 존재하지 않는 Queue 이름을 Routing Key로 사용하면:

**mandatory=false (기본)**: Binding이 없으므로 메시지 조용히 폐기. 에러 없음. 메시지 유실.

**mandatory=true**: `Basic.Return` 프레임이 Producer에게 전달됨. `reply-code=312`, `reply-text="NO_ROUTE"`. Spring AMQP의 `ReturnsCallback`이 호출됨.

실무 함의: `rabbitTemplate.convertAndSend("not-existing-queue", message)`를 mandatory 없이 호출하면 메시지가 유실되고 에러도 없습니다. RabbitMQ Management UI의 Exchange → Default Exchange → Returns 차트에서만 확인 가능합니다.

</details>

---

**Q2.** 하나의 Queue가 두 개의 서로 다른 Exchange에 Binding되어 있다. 두 Exchange 모두에서 같은 메시지(같은 메시지 ID)가 해당 Queue로 라우팅됐을 때, Queue에는 몇 개의 메시지가 쌓이는가?

<details>
<summary>해설 보기</summary>

**2개**의 메시지가 Queue에 쌓입니다.

AMQP 라우팅은 Exchange → Queue 전달 시 메시지를 복사합니다. Exchange A와 Exchange B가 각각 독립적으로 같은 Queue로 라우팅하면, Queue는 같은 내용의 메시지를 2개 받습니다. RabbitMQ는 메시지 ID로 중복을 제거하지 않습니다.

이는 설계할 때 중요한 고려사항입니다. 같은 Queue가 여러 Exchange로부터 메시지를 받으면, Consumer는 중복 처리 가능성을 고려해야 합니다. Consumer가 멱등적(Idempotent)으로 설계되어야 합니다.

</details>

---

**Q3.** Exchange를 삭제하기 전에 확인해야 할 것은 무엇이고, 삭제 후 발행된 메시지는 어떻게 되는가?

<details>
<summary>해설 보기</summary>

삭제 전 확인 사항:
1. **Binding 목록**: 해당 Exchange에 Binding된 Queue들 확인 (`rabbitmqadmin list bindings source=my.exchange`)
2. **Publisher 목록**: 해당 Exchange에 발행하는 Producer 서비스 목록 (코드 레벨 검색)
3. **Alternate Exchange**: 이 Exchange가 다른 Exchange의 AE로 설정되어 있는지 확인

Exchange 삭제 후:
- 기존 Queue에 쌓인 메시지: **영향 없음** (Queue와 Exchange는 독립)
- 삭제된 Exchange로 발행 시도: `NOT_FOUND - no exchange 'my.exchange' in vhost '/'` 에러 발생
- Producer가 존재하지 않는 Exchange에 발행 시도 → Connection 또는 Channel 강제 종료 가능

안전한 Exchange 삭제 절차:
1. Producer 코드에서 해당 Exchange 발행 제거 또는 다른 Exchange로 교체
2. 배포 완료 후 Exchange에 Incoming 메시지 없는 것 확인 (Management UI)
3. Exchange 삭제

</details>

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[이전: AMQP 프로토콜 완전 분해 ⬅️](./02-amqp-protocol.md)** | **[다음: Queue 내부 구조 ➡️](./04-queue-internals.md)**

</div>
