# 라우팅 패턴 설계 가이드 — 도메인 이벤트를 Exchange로 설계하기

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 도메인 이벤트를 Exchange와 Routing Key로 설계하는 원칙은 무엇인가?
- 마이크로서비스 환경에서 Exchange를 도메인별로 나눌 것인가, 하나로 통합할 것인가?
- 어떤 상황에서 Direct, Topic, Fanout을 각각 선택하는가?
- Exchange 설계 시 미래 변경에 유연하게 대응하는 방법은?
- 실제 주문 도메인을 예시로 전체 Exchange 구조는 어떻게 그려지는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

Exchange 유형은 이해했지만, "실제로 어떻게 설계해야 하는가"라는 질문에 막히는 경우가 많다. Exchange 하나에 모든 이벤트를 넣으면 Routing Key가 복잡해지고, 너무 잘게 나누면 Exchange 수가 폭발한다. 이 문서는 도메인 이벤트 → Exchange → Queue → Consumer로 이어지는 전체 설계 의사결정을 정리한다.

---

## 😱 흔한 실수 (Before — 설계 없이 Exchange를 추가한 결과)

```
실수 1: 기능별 Exchange 난립

  order-created.exchange
  order-cancelled.exchange
  order-shipped.exchange
  payment-success.exchange
  payment-failed.exchange
  inventory-reserved.exchange
  ... (이벤트마다 Exchange 하나)

  결과:
    Exchange 30개 관리
    Producer가 이벤트마다 다른 Exchange 이름 알아야 함
    Binding 50개 이상 → Management UI에서 파악 불가

실수 2: 모든 이벤트를 하나의 Exchange에 Topic으로

  global.exchange (Topic)
  routingKey: "order.placed", "order.cancelled", "payment.success"
  
  결과:
    Exchange 자체는 단순 → Binding 관리 복잡
    payment-service Queue에 order.* Binding → 불필요한 이벤트 수신 가능
    팀 간 Binding 충돌 위험 (서로 다른 팀이 같은 Exchange 수정)

실수 3: Exchange와 Queue 명명 불일치

  Exchange: "orders" (복수)
  Queue: "order.notification" (단수)
  Routing Key: "OrderPlaced" (PascalCase)
  
  결과:
    일관성 없는 네이밍 → 혼란
    새 팀원이 구조 파악하기 어려움
```

---

## ✨ 올바른 접근 (After — 도메인 기반 Exchange 설계)

```
설계 원칙:

1. 도메인별 Exchange (Domain-per-Exchange)
   order.exchange → 주문 도메인의 모든 이벤트
   payment.exchange → 결제 도메인
   inventory.exchange → 재고 도메인

2. Exchange 유형 선택 기준
   같은 이벤트를 여러 서비스가 받는다 → Topic 또는 Fanout
   특정 서비스에만 보낸다 → Direct
   도메인 내 이벤트가 다양하고 구독자가 선택적 → Topic

3. Routing Key 네이밍 컨벤션
   <도메인>.<엔티티>.<행동> 또는 <도메인>.<행동>
   소문자, 점(.) 구분, 과거형
   예: order.placed, order.payment.failed, inventory.item.reserved

4. Queue 네이밍 컨벤션
   <소비_서비스>.<도메인>.<이벤트> 또는 <도메인>.<소비_서비스>
   예: notification.order.queue, order.inventory-service.queue
```

---

## 🔬 내부 동작 원리

### 1. 도메인별 Exchange 설계 패턴

```
=== 주문 도메인 Exchange 구조 ===

order.exchange (Topic)
│
├── Routing Key: "order.placed"
│   ├── → payment.order.queue (결제 서비스 처리)
│   ├── → inventory.order.queue (재고 서비스 처리)
│   └── → notification.order.queue (알림 서비스)
│
├── Routing Key: "order.payment.completed"
│   ├── → shipping.order.queue (배송 서비스)
│   └── → analytics.order.queue (분석 서비스)
│
├── Routing Key: "order.payment.failed"
│   ├── → notification.order.queue (실패 알림)
│   └── → alert.order.queue (긴급 알림)
│
└── Routing Key: "order.cancelled"
    ├── → inventory.order.queue (재고 복원)
    └── → notification.order.queue (취소 알림)

Binding 설계:
  "order.#"            → analytics.order.queue (모든 주문 이벤트 수집)
  "order.placed"       → payment.order.queue (정확한 이벤트만)
  "order.placed"       → inventory.order.queue
  "#.failed"           → alert.order.queue (모든 실패)
  "order.payment.*"    → payment-monitor.queue (결제 모니터링)

=== 서비스 간 이벤트 흐름 ===

OrderService ─── order.exchange ──→ payment.order.queue ─── PaymentService
                                 ──→ inventory.order.queue ── InventoryService
                                 ──→ notification.order.queue ─ NotificationService

PaymentService ─ payment.exchange ──→ order.payment.queue ─── OrderService
                                  ──→ notification.payment.queue ─ NotificationService

InventoryService ─ inventory.exchange ──→ order.inventory.queue ─── OrderService
```

### 2. Exchange 재사용 vs 서비스별 Exchange

```
=== 전략 1: 도메인별 Exchange 재사용 ===

order.exchange (Topic)
payment.exchange (Topic)
inventory.exchange (Topic)
notification.exchange (Fanout)

장점:
  Exchange 수 최소화
  도메인 경계가 명확
  팀별 책임 명확 (OrderTeam이 order.exchange 관리)

단점:
  한 Exchange에 많은 Binding → 복잡성 증가
  팀 간 Exchange 공유 시 충돌 가능성

=== 전략 2: 발행자별 Exchange ===

order-service.exchange → OrderService가 발행하는 모든 이벤트
payment-service.exchange → PaymentService가 발행하는 모든 이벤트

장점:
  발행자가 자신의 Exchange만 관리
  다른 팀의 Exchange 설정에 영향 없음

단점:
  소비자가 여러 Exchange를 구독해야 할 수 있음
  "결제 관련 이벤트를 모두 모으기"가 어려울 수 있음

=== 전략 3: 이벤트 버스 (Event Bus) — 주의 필요 ===

event.bus (Topic) — 모든 이벤트 하나의 Exchange

장점:
  Exchange 1개로 단순

단점:
  모든 팀이 같은 Exchange 수정
  Routing Key 충돌 위험
  장애 시 전체 영향
  → 소규모 팀에서만 권장, 성장 후 분리 필요

=== 권장: 도메인별 Exchange ===

도메인 수 ≤ 5개: 도메인별 Topic Exchange
도메인 수 > 5개: 도메인별 Exchange + 팀별 vHost 분리 검토
단순 Worker Queue: Direct Exchange 또는 Default Exchange
```

### 3. 실전 설계 — 주문 서비스 완전한 Exchange 구조

```
=== 전체 Exchange 구조 ===

[OrderService]
  발행: order.exchange (Topic, durable)
  Routing Key 패턴:
    order.placed          → 주문 접수
    order.cancelled       → 주문 취소
    order.payment.requested  → 결제 요청
    order.payment.completed  → 결제 완료
    order.payment.failed     → 결제 실패
    order.inventory.reserved → 재고 예약
    order.shipped         → 배송 시작
    order.delivered       → 배달 완료

[Queue 구조]
  order.exchange
  ├── "order.placed" → payment.svc.queue
  │                 → inventory.svc.queue
  │                 → notification.svc.queue
  ├── "order.payment.completed" → shipping.svc.queue
  ├── "order.payment.failed" → notification.svc.queue
  │                         → finance.alert.queue
  ├── "order.cancelled" → inventory.svc.queue (재고 복원)
  │                     → payment.svc.queue (환불 처리)
  │                     → notification.svc.queue
  ├── "order.#" → audit.svc.queue (감사 로그)
  └── "#.failed" → ops.alert.queue (운영팀 알림)

[Dead Letter 구조]
  각 Queue에 DLX 설정:
  payment.svc.queue → dlx.exchange → payment.dlq
  inventory.svc.queue → dlx.exchange → inventory.dlq
  notification.svc.queue → dlx.exchange → notification.dlq

[Spring AMQP 설정]

@Configuration
public class OrderExchangeConfig {

  // Exchange
  @Bean
  public TopicExchange orderExchange() {
    return ExchangeBuilder.topicExchange("order.exchange")
      .durable(true).build();
  }

  // Queue + DLX
  @Bean
  public Queue paymentServiceQueue() {
    return QueueBuilder.durable("payment.svc.queue")
      .deadLetterExchange("dlx.exchange")
      .deadLetterRoutingKey("payment.svc")
      .build();
  }

  // Binding
  @Bean
  public Binding paymentOrderPlacedBinding() {
    return BindingBuilder.bind(paymentServiceQueue())
      .to(orderExchange()).with("order.placed");
  }

  @Bean
  public Binding paymentOrderCancelledBinding() {
    return BindingBuilder.bind(paymentServiceQueue())
      .to(orderExchange()).with("order.cancelled");
  }

  // Audit — 모든 주문 이벤트
  @Bean
  public Binding auditBinding() {
    return BindingBuilder.bind(auditQueue())
      .to(orderExchange()).with("order.#");
  }

  // 운영 알림 — 모든 실패
  @Bean
  public Binding opsAlertBinding() {
    return BindingBuilder.bind(opsAlertQueue())
      .to(orderExchange()).with("#.failed");
  }
}
```

### 4. Exchange 설계 의사결정 트리

```
=== Exchange 유형 선택 ===

"이 이벤트의 구독자가 누구인가?"
  │
  ├── 항상 정해진 하나의 서비스
  │   └── Direct Exchange (명확한 1:1)
  │
  ├── 도메인 내 여러 서비스 (선택적 구독)
  │   └── Topic Exchange (패턴 기반 구독)
  │
  ├── 모든 구독 서비스가 예외 없이 전부 받아야 함
  │   └── Fanout Exchange (조건 없는 브로드캐스트)
  │
  └── 조건이 Routing Key로 표현 불가 (다차원)
      └── Headers Exchange

=== Exchange 분리 기준 ===

"Exchange를 나눌 것인가?"
  │
  ├── 발행자가 다른 팀인가? → Yes → 별도 Exchange
  ├── 이벤트 도메인이 다른가? → Yes → 도메인별 Exchange
  ├── vHost가 다른가? → 이미 격리됨
  └── 이벤트 수 < 5개 → 하나의 Exchange

=== Routing Key 설계 원칙 ===

"Topic Exchange를 쓴다면"
  │
  ├── 계층적 구조 사용: domain.entity.action
  ├── 소문자 + 점(.) 구분
  ├── 과거형 동사: placed, failed, completed
  ├── 3단계 이하 권장: order.payment.failed (3단계 OK)
  └── 와일드카드로 그룹화 가능한지 사전 검토
```

---

## 💻 실전 실험

### 실험 1: 주문 도메인 Exchange 전체 구성

```bash
# Exchange 선언
rabbitmqadmin declare exchange name=order.exchange type=topic durable=true
rabbitmqadmin declare exchange name=dlx.exchange type=direct durable=true

# Service Queues (with DLX)
for svc in payment inventory notification shipping audit; do
  rabbitmqadmin declare queue name=${svc}.svc.queue durable=true \
    arguments="{\"x-dead-letter-exchange\":\"dlx.exchange\",\"x-dead-letter-routing-key\":\"${svc}.svc\"}"
  
  rabbitmqadmin declare queue name=${svc}.dlq durable=true
  rabbitmqadmin declare binding source=dlx.exchange \
    destination=${svc}.dlq routing_key=${svc}.svc
done

# Bindings
rabbitmqadmin declare binding source=order.exchange destination=payment.svc.queue routing_key="order.placed"
rabbitmqadmin declare binding source=order.exchange destination=payment.svc.queue routing_key="order.cancelled"
rabbitmqadmin declare binding source=order.exchange destination=inventory.svc.queue routing_key="order.placed"
rabbitmqadmin declare binding source=order.exchange destination=inventory.svc.queue routing_key="order.cancelled"
rabbitmqadmin declare binding source=order.exchange destination=notification.svc.queue routing_key="order.#"
rabbitmqadmin declare binding source=order.exchange destination=shipping.svc.queue routing_key="order.payment.completed"
rabbitmqadmin declare binding source=order.exchange destination=audit.svc.queue routing_key="order.#"

# 이벤트 발행 테스트
rabbitmqadmin publish exchange=order.exchange routing_key="order.placed" \
  payload='{"orderId":1,"userId":100,"amount":50000}'

rabbitmqctl list_queues name messages
# payment.svc.queue    1
# inventory.svc.queue  1
# notification.svc.queue 1
# audit.svc.queue      1
# shipping.svc.queue   0 (order.placed는 shipping 미구독)
```

### 실험 2: Exchange 구조 시각화 (Management UI)

```
Management UI 활용:
  1. Exchanges 탭 → order.exchange 클릭
  2. Bindings 섹션에서 전체 Binding 목록 확인
  3. "Publish message" 기능으로 각 Routing Key 테스트
  4. Queues 탭에서 각 Queue의 메시지 수 확인

Exchange Graph (비공식 도구):
  RabbitMQ Management UI → Overview
  → Community Plugin: rabbitmq-web-stomp-examples
  또는 rabbitmqadmin list bindings 명령으로 구조 파악

구조 문서화 템플릿:
  ## Exchange: order.exchange (Topic)
  | Routing Key        | Queue                  | 설명              |
  |--------------------|------------------------|-------------------|
  | order.placed       | payment.svc.queue      | 결제 서비스 처리    |
  | order.placed       | inventory.svc.queue    | 재고 예약          |
  | order.cancelled    | inventory.svc.queue    | 재고 복원          |
  | order.#            | audit.svc.queue        | 감사 로그          |
  | #.failed           | ops.alert.queue        | 운영팀 알림        |
```

---

## 📊 Kafka vs RabbitMQ 비교

```
=== 도메인 이벤트 라우팅 아키텍처 비교 ===

RabbitMQ (Exchange-based):
  order.exchange (Topic)
    "order.placed" → 3개 Queue → 3개 서비스
  
  특징:
    브로커가 라우팅 결정 (Binding 기반)
    새 구독자: Queue + Binding 추가
    과거 이벤트 소급 불가

Kafka (Topic-based):
  order-events Topic (Partition 3개)
  Consumer Group: payment-group, inventory-group, notification-group
  
  특징:
    Consumer가 라우팅 결정 (오프셋 추적)
    새 구독자: Consumer Group 생성 (과거 이벤트 소급 가능)
    Topic 내 필터링: Kafka Streams 또는 Consumer 필터

=== 설계 복잡성 비교 ===

RabbitMQ:
  설계 복잡성: Exchange + Queue + Binding 3가지 관리
  유연성: 브로커 레벨 라우팅 변경 가능
  가시성: Management UI에서 전체 구조 확인

Kafka:
  설계 복잡성: Topic + Consumer Group (단순)
  유연성: Topic 추가 시 Producer/Consumer 코드 변경
  가시성: Topic/Consumer Group 상태 확인

결론:
  복잡한 라우팅 + 낮은 지연 + 메시지 처리 후 삭제 → RabbitMQ
  단순한 Topic 구독 + 과거 재처리 + 높은 처리량 → Kafka
```

---

## ⚖️ 트레이드오프

```
Exchange 설계 전략별 트레이드오프:

도메인별 Exchange:
  ✅ 명확한 도메인 경계
  ✅ 팀별 Exchange 소유권
  ❌ Exchange 수 증가 (5~10개)

단일 Exchange:
  ✅ 관리 단순
  ❌ 팀 간 Binding 충돌 위험
  ❌ 확장 시 복잡성 폭발

Exchange 수 vs Binding 수:
  Exchange를 줄이면 Binding이 복잡해짐
  Exchange를 늘리면 Binding 단순해지지만 Exchange 관리 복잡
  → 도메인당 1개 Exchange가 균형점

미래 변경 대비:
  Routing Key 설계를 Topic으로 미리 계층화
  Direct로 시작해도 나중에 Topic으로 전환 가능
  (Binding 재설정만으로 가능, Producer 변경 없음)
```

---

## 📌 핵심 정리

```
Exchange 설계 핵심 원칙:

Exchange 유형 선택:
  조건 없는 브로드캐스트 → Fanout
  도메인 이벤트 선택적 구독 → Topic
  특정 서비스 지정 → Direct
  다차원 조건 → Headers

도메인별 Exchange:
  order.exchange, payment.exchange, inventory.exchange
  팀별 소유권 명확
  Exchange당 모든 이벤트 포함 (Routing Key로 구분)

Routing Key 설계:
  <도메인>.<엔티티>.<행동> 계층 구조
  소문자, 점 구분, 과거형
  와일드카드 활용 가능하도록 설계

Queue 네이밍:
  <소비서비스>.<도메인>.queue 또는 <도메인>.<소비서비스>.queue
  DLQ: <서비스>.dlq
  Dead Letter Exchange: dlx.exchange (공통)

설계 검증:
  각 이벤트 → 어느 Queue → 어느 Consumer 흐름 문서화
  Binding 목록 관리 (코드 또는 문서)
  새 서비스 추가 시 어떤 Binding이 필요한지 사전 계획
```

---

## 🤔 생각해볼 문제

**Q1.** 주문 서비스가 기존에 Direct Exchange를 사용하고 있다. 새 요구사항으로 "결제 관련 이벤트는 모두 감사팀이 구독해야 한다"가 추가됐다. 어떻게 대응하는가?

<details>
<summary>해설 보기</summary>

**Exchange 유형을 Direct → Topic으로 변경**하거나, 기존 Direct Exchange를 유지하면서 모든 결제 관련 Routing Key에 감사 Queue Binding을 추가하는 두 가지 방법이 있습니다.

**방법 1: Topic Exchange로 마이그레이션**
```bash
# 새 Topic Exchange 생성
rabbitmqadmin declare exchange name=order.exchange.v2 type=topic durable=true

# 기존 Routing Key를 계층적으로 재설계
# "payment" → "order.payment.completed"
# "payment_failed" → "order.payment.failed"

# 감사팀 Binding 추가 가능
rabbitmqadmin declare binding source=order.exchange.v2 \
  destination=audit.queue routing_key="order.payment.*"
```

**방법 2: Direct 유지 + 모든 Key에 Binding 추가**
```bash
rabbitmqadmin declare binding source=order.exchange destination=audit.queue routing_key="payment"
rabbitmqadmin declare binding source=order.exchange destination=audit.queue routing_key="payment_failed"
rabbitmqadmin declare binding source=order.exchange destination=audit.queue routing_key="payment_refunded"
# ... 결제 관련 모든 Key마다 Binding 추가
```

이것이 Topic Exchange의 장점을 보여주는 상황입니다. Direct에서는 Binding을 하나씩 추가해야 하지만, Topic에서는 `"order.payment.*"` Binding 하나로 모든 결제 이벤트를 포함합니다.

</details>

---

**Q2.** 마이크로서비스가 100개이고 각각 이벤트를 발행한다. Exchange를 서비스마다 1개씩 100개 만들어야 하는가?

<details>
<summary>해설 보기</summary>

**아니요.** 서비스마다 Exchange를 만들면 오히려 복잡성이 증가합니다.

권장 설계:
- **도메인별 Exchange** (5~15개): `order.exchange`, `payment.exchange`, `user.exchange`, `inventory.exchange`
- 같은 도메인의 여러 서비스가 같은 Exchange를 공유

예시:
- `OrderService`, `OrderHistoryService`가 모두 `order.exchange`에 발행
- Routing Key로 서비스와 이벤트 구분: `order.placed`, `order-history.viewed`

100개 서비스라면 현실적으로:
1. **도메인 그룹화**: 비즈니스 도메인별 Exchange 5~20개
2. **vHost 분리**: 팀 경계로 vHost 구분 (Exchange 간 격리)
3. **Event Mesh**: 복잡도가 높으면 Service Mesh + 이벤트 버스 도입 검토

핵심은 "Exchange 수 = 서비스 수"가 아니라 "Exchange 수 = 도메인 수"로 생각하는 것입니다.

</details>

---

**Q3.** RabbitMQ Exchange 구조를 코드로 관리할 것인가, 인프라(Terraform/Ansible)로 관리할 것인가? 각각의 장단점은?

<details>
<summary>해설 보기</summary>

두 방식의 장단점:

**코드 관리 (Spring AMQP @Bean 선언 - 현재 일반적)**
```java
@Bean public Queue orderQueue() { ... }
@Bean public Binding binding() { ... }
// RabbitAdmin이 앱 시작 시 자동 선언
```
✅ 서비스와 Queue/Exchange 코드가 함께 관리
✅ PR에서 변경 추적 가능
✅ 환경별 설정 분리 쉬움
❌ 여러 서비스가 같은 Exchange를 선언 시 충돌 가능
❌ 서비스가 configure 권한 필요 (보안 약화)

**인프라 관리 (Terraform/Ansible)**
```hcl
resource "rabbitmq_exchange" "order" { name = "order.exchange"; type = "topic" }
resource "rabbitmq_queue" "payment_svc" { name = "payment.svc.queue" }
resource "rabbitmq_binding" "payment_order_placed" { ... }
```
✅ 서비스는 read/write 권한만 (보안 강화)
✅ 인프라 변경 이력 관리
✅ 여러 서비스 독립적으로 같은 Exchange 사용
❌ Exchange/Queue 변경이 인프라 배포 사이클을 따라야 함
❌ 서비스와 인프라 설정 불일치 위험

**권장 전략**:
- 개발/스테이징: Spring AMQP 자동 선언 (편의성)
- 프로덕션: 인프라 도구로 Exchange/Queue 사전 생성, 서비스는 write/read만

</details>

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[이전: Dead Letter Exchange ⬅️](./05-dead-letter-exchange.md)** | **[다음: Chapter 3 — 메시지 유실의 세 지점 ➡️](../message-reliability/01-message-loss-points.md)**

</div>
