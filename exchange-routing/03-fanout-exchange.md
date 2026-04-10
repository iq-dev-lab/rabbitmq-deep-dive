# Fanout Exchange — 모든 Queue에 브로드캐스트

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Fanout Exchange는 Routing Key를 왜 완전히 무시하는가?
- Pub/Sub 패턴에서 각 Consumer가 자신만의 임시 Queue를 가져야 하는 이유는?
- Fanout Exchange에 Binding된 Queue가 없으면 메시지는 어떻게 되는가?
- 새 서비스가 Fanout Exchange를 구독하면 기존 서비스에 영향이 있는가?
- Fanout Exchange와 Topic Exchange `"#"` 패턴의 차이는 무엇인가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

"주문 발생 이벤트를 재고팀, 알림팀, 분석팀, 배송팀이 동시에 받아야 한다"는 요구사항은 가장 흔한 마이크로서비스 패턴이다. Fanout Exchange는 이 패턴을 가장 단순하게 구현한다. 구독자 추가/제거 시 발행자 코드를 전혀 건드리지 않아도 된다는 것이 핵심 가치다. 하지만 임시 Queue와 영구 Queue를 혼용하는 실수, Binding이 없을 때의 메시지 폐기를 모르면 Pub/Sub 패턴이 의도대로 동작하지 않는다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: Consumer 수만큼 Queue 없이 하나의 Queue 공유

  Fanout Exchange → notification.queue
  
  Consumer A, B, C 모두 notification.queue 구독
  
  결과:
    메시지 하나를 A, B, C 중 하나만 받음 (경쟁 소비)
    → A가 받으면 B, C는 못 받음
    → Pub/Sub이 아닌 Work Queue가 됨

  Pub/Sub의 올바른 구조:
    각 Consumer마다 자신만의 Queue
    Consumer A → a.queue → Fanout Exchange
    Consumer B → b.queue → Fanout Exchange
    Consumer C → c.queue → Fanout Exchange
    → 메시지 하나가 세 Queue에 복사 → 각 Consumer가 독립 처리

실수 2: Binding 없는 Fanout Exchange에 발행

  아직 구독자(Queue)가 없는 Fanout Exchange에 메시지 발행
  
  결과:
    Binding된 Queue 없음 → 메시지 폐기
    이후 구독자가 연결해도 폐기된 메시지 복구 불가
    
  실무 문제:
    "서비스 배포 중 Exchange에는 발행하는데 Consumer가 아직 안 떴다"
    → 이 사이 메시지 유실 (Fanout에 Binding이 없으므로)
    → Consumer가 뜨기 전에 durable Queue와 Binding을 미리 생성 필요

실수 3: Fanout Exchange에 Routing Key를 의미 있게 사용

  rabbitTemplate.convertAndSend("notify.fanout", "vip.customer", message);
  // "vip.customer" Routing Key를 설정했지만 Fanout은 무시

  결과:
    routingKey="vip.customer"는 완전히 무시됨
    모든 Binding Queue에 전달됨
    "VIP 고객에게만 보내려 했는데 전체에게 보내짐"
    → VIP 필터링이 필요하면 Topic Exchange 사용
```

---

## ✨ 올바른 접근 (After — Pub/Sub 올바른 구현)

```
Pub/Sub 패턴 두 가지:

=== 패턴 1: 영구 Queue (서비스 수만큼) ===

각 Consumer 서비스마다 durable Queue 생성

order.fanout (Fanout Exchange)
├── Binding → inventory.queue (durable) → InventoryService
├── Binding → notification.queue (durable) → NotificationService
└── Binding → analytics.queue (durable) → AnalyticsService

특징:
  Consumer 오프라인 시 Queue에 메시지 축적
  Consumer 복구 후 축적된 메시지 처리 가능
  서비스별 처리 속도 독립적

=== 패턴 2: 임시 Queue (로그 스트리밍, 실시간 대시보드) ===

각 Consumer 연결 시 임시 Queue 동적 생성

order.fanout (Fanout Exchange)
├── Binding → amq.gen-Xfj1... (exclusive, autoDelete) → 대시보드 인스턴스 1
└── Binding → amq.gen-Yrk2... (exclusive, autoDelete) → 대시보드 인스턴스 2

특징:
  Consumer 연결 시 Queue 생성, 연결 해제 시 삭제
  오프라인 시 메시지 축적 없음 (임시 Queue 삭제됨)
  실시간 이벤트 뷰어, 로그 스트리밍에 적합

Spring AMQP 임시 Queue 패턴:
  @RabbitListener(
    bindings = @QueueBinding(
      value = @Queue(exclusive = "true", autoDelete = "true"),
      exchange = @Exchange(value = "order.fanout", type = "fanout")
    )
  )
  public void handleOrderEvent(OrderEvent event) {
    // 이 인스턴스만의 임시 Queue에서 수신
  }
```

---

## 🔬 내부 동작 원리

### 1. Fanout Exchange 라우팅 — Routing Key 완전 무시

```
=== Fanout 라우팅 알고리즘 ===

Direct: Routing Key == Binding Key ? 전달 : 폐기
Topic:  Routing Key가 패턴에 매칭 ? 전달 : 폐기
Fanout: Binding된 Queue 전부 전달 (조건 없음)

내부 처리:
  메시지 수신 → Binding 목록 조회 → 전체 전달 (Key 평가 없음)

성능:
  Direct: O(1) 해시 조회
  Topic:  O(B × L) 패턴 매칭
  Fanout: O(B) — Binding 수만큼 순회 (매칭 연산 없음)

=== Routing Key가 무시된다는 것의 의미 ===

아래 두 발행은 완전히 동일한 결과:
  rabbitTemplate.convertAndSend("order.fanout", "order.placed", msg)
  rabbitTemplate.convertAndSend("order.fanout", "",             msg)
  rabbitTemplate.convertAndSend("order.fanout", "ignored",      msg)

→ 세 경우 모두 Binding된 모든 Queue에 전달
→ Fanout에서 Routing Key는 관례상 ""(빈 문자열) 사용 권장
```

### 2. Pub/Sub 패턴 상세 — Queue 구조

```
=== 영구 Queue Pub/Sub 구조 ===

[order.fanout Exchange]
        │
        ├──── inventory.queue (durable=true)
        │         │
        │         └─→ InventoryService Consumer
        │               Prefetch=10, autoAck=false
        │               오프라인 → Queue에 메시지 쌓임
        │               복구 후 순서대로 처리
        │
        ├──── notification.queue (durable=true)
        │         │
        │         └─→ NotificationService Consumer
        │               오프라인 → Queue에 메시지 쌓임
        │
        └──── analytics.queue (durable=true)
                  │
                  └─→ AnalyticsService Consumer
                        배치 처리 (Prefetch=100)

메시지 1개 발행 → 3개 Queue에 복사 → 3개 독립 처리
각 Queue의 처리 속도 완전히 독립적
한 서비스의 장애가 다른 서비스에 영향 없음

=== 임시 Queue Pub/Sub 구조 ===

[order.fanout Exchange]
        │
        ├──── amq.gen-Xfj1 (exclusive, autoDelete)
        │         │
        │         └─→ 대시보드 인스턴스 1
        │               연결 해제 시 Queue + Binding 자동 삭제
        │
        └──── amq.gen-Yrk2 (exclusive, autoDelete)
                  │
                  └─→ 대시보드 인스턴스 2

대시보드가 오프라인 상태일 때 발행된 메시지는 소실 (Queue 없으므로)
→ "놓친 메시지는 포기해도 되는" 실시간 스트리밍 용도

=== 구독자 추가 — 기존 서비스 영향 없음 ===

기존:
  order.fanout → inventory.queue
  order.fanout → notification.queue

배송 서비스 추가:
  1. delivery.queue 생성 (durable)
  2. Binding: order.fanout → delivery.queue
  3. DeliveryService 배포

Producer(OrderService) 변경: 없음
기존 Consumer 변경: 없음
→ 완전히 독립적인 서비스 추가
```

### 3. 브로드캐스트 복사 — 메시지가 복사되는 시점

```
=== 메시지 복사 타이밍 ===

Fanout Exchange → 3개 Queue 상황에서:

1. Exchange 메시지 수신
2. Binding 목록 조회: [inventory.queue, notification.queue, analytics.queue]
3. 메시지 복사본 3개 생성
4. 각 Queue에 독립적으로 저장

복사본은 독립적:
  inventory.queue 메시지 Ack → inventory.queue에서 삭제
  notification.queue 메시지는 그대로 유지

메모리 비용:
  메시지 크기 × Binding 수 만큼 메모리 사용
  큰 메시지를 많은 Queue로 브로드캐스트 → 메모리 주의

=== Binding이 없을 때 ===

Fanout Exchange에 Binding된 Queue가 0개:
  → 메시지 폐기 (아무 곳도 없음)
  → mandatory=true여도 Return 안 함 (Queue 없으므로 라우팅 실패 아님)
  
주의: "아직 구독자가 없는 초기 상태에서 발행 시 메시지 유실"
해결: 영구 Queue를 Exchange보다 먼저 생성
     또는 발행 전 Binding 확인
```

---

## 💻 실전 실험

### 실험 1: Pub/Sub 패턴 — 영구 Queue

```bash
# Fanout Exchange 생성
rabbitmqadmin declare exchange name=order.fanout type=fanout durable=true

# 서비스별 Queue 생성
rabbitmqadmin declare queue name=inventory.queue durable=true
rabbitmqadmin declare queue name=notification.queue durable=true
rabbitmqadmin declare queue name=analytics.queue durable=true

# Fanout Exchange에 Binding (routing_key 무시됨)
rabbitmqadmin declare binding source=order.fanout \
  destination=inventory.queue routing_key=""
rabbitmqadmin declare binding source=order.fanout \
  destination=notification.queue routing_key=""
rabbitmqadmin declare binding source=order.fanout \
  destination=analytics.queue routing_key=""

# 메시지 1개 발행
rabbitmqadmin publish exchange=order.fanout routing_key="" \
  payload='{"orderId":1,"userId":100}'

# 3개 Queue 모두에 도달 확인
rabbitmqctl list_queues name messages
# inventory.queue     1
# notification.queue  1
# analytics.queue     1

# 새 서비스 추가 (기존 Queue/Publisher 변경 없음)
rabbitmqadmin declare queue name=delivery.queue durable=true
rabbitmqadmin declare binding source=order.fanout \
  destination=delivery.queue routing_key=""

rabbitmqadmin publish exchange=order.fanout routing_key="" \
  payload='{"orderId":2}'

rabbitmqctl list_queues name messages
# inventory.queue     2
# notification.queue  2
# analytics.queue     2
# delivery.queue      1  ← 새 서비스만 1개 (이전 메시지는 없음)
```

### 실험 2: 임시 Queue Pub/Sub — Spring AMQP

```java
@Configuration
public class FanoutConfig {

  @Bean
  public FanoutExchange orderFanout() {
    return ExchangeBuilder
      .fanoutExchange("order.fanout")
      .durable(true)
      .build();
  }
}

// 실시간 대시보드 Consumer (임시 Queue)
@Service
public class RealtimeDashboard {

  @RabbitListener(
    bindings = @QueueBinding(
      value = @Queue(
        exclusive = "true",
        autoDelete = "true",
        durable = "false"
      ),
      exchange = @Exchange(
        value = "order.fanout",
        type = ExchangeTypes.FANOUT
      )
    )
  )
  public void onOrderEvent(Map<String, Object> event) {
    // 이 인스턴스만의 임시 Queue에서 수신
    // 다른 인스턴스는 자신만의 임시 Queue에서 독립적으로 수신
    dashboard.update(event);
  }
}
```

### 실험 3: Binding 없는 Fanout — 메시지 폐기 확인

```bash
# Binding이 없는 Fanout Exchange
rabbitmqadmin declare exchange name=empty.fanout type=fanout durable=true

# 메시지 발행
rabbitmqadmin publish exchange=empty.fanout routing_key="" \
  payload='{"test":"message"}'

# Management UI에서 확인:
# Exchanges → empty.fanout
# Incoming: 1 (Exchange 진입)
# Outgoing: 0 (Queue로 안 나감)
# → 메시지 폐기, 에러 없음
```

---

## 📊 Kafka vs RabbitMQ 비교

```
=== 같은 이벤트를 여러 서비스에 전달 ===

RabbitMQ Fanout:
  메시지 1개 → Exchange → N개 Queue에 복사
  각 Queue에 Consumer 서비스 1개씩
  Consumer가 처리 후 Queue에서 삭제 (복사본만 삭제)
  
  메모리: 메시지 크기 × Queue 수
  과거 메시지 재소비: 불가 (처리 후 삭제)

Kafka Consumer Group:
  메시지 1개 → Topic Partition에 저장 (복사 없음)
  Consumer Group A, B, C가 각자 오프셋으로 독립 소비
  
  메모리: 메시지 크기 × 1 (복사 없음)
  과거 메시지 재소비: 가능 (오프셋 리셋)

=== 구독자 추가 비교 ===

N번째 새 서비스 추가:

RabbitMQ Fanout:
  새 Queue 생성 + Binding 추가
  → 추가 시점 이후 새 메시지부터 수신
  → 이전 메시지는 수신 불가 (이미 처리/삭제됨)

Kafka:
  새 Consumer Group 생성
  offset=earliest → 과거 모든 메시지부터 처리 가능
  → 데이터 재처리, 분석에 Kafka가 훨씬 유리

=== 결론 ===

실시간 이벤트 배포 (처리 후 삭제 OK): RabbitMQ Fanout
이벤트 로그 보관 + 재처리 + 분석: Kafka
```

---

## ⚖️ 트레이드오프

```
Fanout Exchange 장단점:

장점:
  ① 가장 단순한 브로드캐스트 구현
  ② 구독자 추가/제거 시 Publisher 변경 없음
  ③ 각 구독자가 독립적인 Queue (처리 속도 독립)
  ④ 라우팅 연산 없음 (성능 측면)

단점:
  ① 선택적 라우팅 불가 (일부 Queue에만 보내기 불가)
  ② 메시지 복사로 메모리 증가 (Queue 수만큼)
  ③ Binding 없으면 메시지 폐기
  ④ 과거 메시지 재처리 불가 (처리 후 삭제)

Fanout vs Topic ("order.#") 선택:
  Fanout: "이 Exchange의 모든 메시지를 전부 받겠다" → Fanout
  Topic:  "이 중 특정 패턴만 받겠다" → Topic
  → 조건 없는 전체 브로드캐스트 = Fanout, 더 명확한 의도 표현
```

---

## 📌 핵심 정리

```
Fanout Exchange 핵심:

라우팅 규칙:
  Routing Key 완전 무시
  Binding된 모든 Queue에 메시지 복사 전달

Pub/Sub 패턴:
  영구 Queue: Consumer 서비스별 durable Queue (Consumer 오프라인 시 축적)
  임시 Queue: exclusive + autoDelete (연결 시 생성, 해제 시 삭제)

구독자 확장:
  새 Queue + Binding 추가만으로 완료
  Publisher 코드 변경 없음

주의사항:
  Binding이 없으면 메시지 폐기 (에러 없음)
  메시지 복사로 Queue 수만큼 메모리 사용
  처리 완료 메시지 재소비 불가
```

---

## 🤔 생각해볼 문제

**Q1.** 5개의 서비스가 Fanout Exchange를 구독한다. 그 중 AnalyticsService는 배치 처리를 위해 메시지를 24시간 모아두어야 한다. 나머지 4개 서비스가 즉시 처리할 때, AnalyticsService의 느린 소비가 다른 서비스에 영향을 주는가?

<details>
<summary>해설 보기</summary>

**다른 서비스에 영향을 주지 않습니다.** 각 서비스가 자신만의 Queue를 가지기 때문입니다.

- `inventory.queue`, `notification.queue` 등: 즉시 처리 → Ack → 삭제
- `analytics.queue`: 24시간 메시지 축적 (처리 안 함)

각 Queue는 완전히 독립적으로 동작합니다. `analytics.queue`에 메시지가 아무리 쌓여도 `inventory.queue`에는 영향이 없습니다.

단, **주의할 점**: `analytics.queue`에 메시지가 대량 쌓이면 해당 Queue의 메모리/디스크 사용량이 증가합니다. 브로커의 `vm_memory_high_watermark`에 도달하면 **모든 vHost의 Publisher**에 Flow Control이 발동됩니다. 이것이 간접적으로 다른 서비스에 영향을 줄 수 있습니다.

해결: `analytics.queue`를 Lazy Queue로 설정하여 메시지를 즉시 디스크에 저장.

</details>

---

**Q2.** 하나의 Fanout Exchange에서 `inventory.queue`는 `durable=true`, `analytics.queue`는 `durable=false`로 설정되어 있다. RabbitMQ를 재시작하면 각각 어떻게 되는가?

<details>
<summary>해설 보기</summary>

- `inventory.queue` (durable=true): Queue와 메시지(DeliveryMode=PERSISTENT) 보존
- `analytics.queue` (durable=false): Queue 삭제, 메시지 소실

재시작 후:
- Fanout Exchange (durable=true가 보통): 정상 존재
- `inventory.queue` Binding: 정상 존재
- `analytics.queue` Binding: **자동 삭제** (Queue가 없으므로)

이후 AnalyticsService가 연결 시:
- `analytics.queue`가 없으면 `RabbitAdmin`이 자동 재선언 (Spring AMQP)
- 재선언 전까지 해당 Queue로 메시지가 전달되지 않음

혼합 durable 설정의 위험: 재시작 후 Binding 누락으로 일부 서비스가 메시지를 받지 못하는 무소음 장애 발생 가능. 모든 Queue를 `durable=true`로 통일하는 것이 운영 안전성을 높입니다.

</details>

---

**Q3.** "알림 서비스는 VIP 고객의 이벤트만 받아야 한다"는 요구사항이 생겼다. Fanout Exchange로 이 요구사항을 구현할 수 있는가? 어떤 Exchange로 변경해야 하는가?

<details>
<summary>해설 보기</summary>

**Fanout Exchange로는 불가능합니다.** Fanout은 Routing Key를 무시하고 모든 Queue에 전달합니다.

선택지:

**방법 1: Topic Exchange 전환**
Routing Key에 고객 등급 포함:
- `"order.vip.placed"`, `"order.regular.placed"` 형식
- VIP Queue Binding: `"order.vip.#"`
- 일반 Queue Binding: `"order.#"`

**방법 2: Headers Exchange 전환**
메시지 헤더에 `customerType: "vip"` 포함:
- VIP Queue Binding: `x-match=all, customerType=vip`

**방법 3: Consumer 측 필터링 (Fanout 유지)**
모든 메시지를 받되 Consumer가 직접 필터:
```java
@RabbitListener(queues = "notification.queue")
public void handle(OrderEvent event) {
  if (!"VIP".equals(event.getCustomerType())) return;
  // VIP만 처리
}
```
→ 불필요한 메시지 전달로 네트워크/처리 낭비

**권장**: 조건부 라우팅이 필요한 경우 Topic 또는 Headers Exchange 사용. Fanout은 "조건 없는 전체 브로드캐스트"에만 적합합니다.

</details>

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[이전: Topic Exchange ⬅️](./02-topic-exchange.md)** | **[다음: Headers Exchange ➡️](./04-headers-exchange.md)**

</div>
