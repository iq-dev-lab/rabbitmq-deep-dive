# Topic Exchange — 와일드카드 패턴 라우팅

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `*`(별표)와 `#`(샵)은 각각 어떤 규칙으로 Routing Key를 매칭하는가?
- Routing Key를 계층적으로 설계해야 하는 이유는 무엇인가?
- 같은 메시지가 여러 Topic 패턴에 동시에 매칭될 때 어떻게 되는가?
- Topic Exchange는 언제 Direct Exchange보다 유리하고 언제 불리한가?
- 실무에서 Routing Key 설계 실수를 어떻게 방지하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

마이크로서비스 환경에서 이벤트 라우팅은 점점 복잡해진다. "결제 실패 이벤트는 알림팀과 감사팀이 모두 받아야 한다", "특정 도메인의 모든 이벤트를 모니터링 서비스가 수집해야 한다"는 요구가 생긴다. Direct Exchange로는 Binding이 폭발적으로 늘어난다. Topic Exchange의 와일드카드 패턴을 이해하면, 단 몇 개의 Binding으로 복잡한 라우팅을 표현할 수 있다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: * 와 # 의 차이를 모르고 잘못 사용

  Routing Key: "order.payment.failed"
  
  패턴 "order.*":
    * = 단어 하나 (점으로 구분된 세그먼트 1개)
    "order.*" → order.결제 ← 세그먼트 2개 → 매칭 ❌
    
  패턴 "order.#":
    # = 0개 이상의 단어
    "order.#" → order.payment.failed ← 3세그먼트 → 매칭 ✅

  착각: "order.*로 하면 order 하위 모든 이벤트를 받는다"
  실제: "order.*"는 세그먼트 2개만 매칭 (order.payment는 OK, order.payment.failed는 ❌)

실수 2: Routing Key를 의미 없이 설계

  "orderplaced", "ordercancelled", "ordershipped"  ← 계층 없음

  문제:
    "모든 주문 이벤트를 수집"하는 패턴 표현 불가
    Direct와 다를 바 없음 (Topic의 이점 없음)

  올바른 설계: "order.placed", "order.cancelled", "order.payment.failed"
    → "order.#"로 모든 주문 이벤트 수집
    → "order.payment.*"로 결제 이벤트만 수집
    → "#.failed"로 모든 실패 이벤트 수집

실수 3: Binding이 너무 많아 관리 불가

  order.placed         → service-a.queue
  order.cancelled      → service-a.queue
  order.shipped        → service-a.queue
  payment.success      → service-a.queue
  payment.failed       → service-a.queue
  ... (50개 Binding)

  해결: Routing Key 계층 설계 후 와일드카드로 통합
    "order.#"     → service-a.queue  ← 주문 이벤트 전체
    "payment.#"   → service-a.queue  ← 결제 이벤트 전체
```

---

## ✨ 올바른 접근 (After — 계층적 Routing Key 설계)

```
Routing Key 계층 설계 원칙:
  형식: <도메인>.<이벤트유형>.<세부상태>
  예시:
    order.placed
    order.cancelled
    order.payment.requested
    order.payment.completed
    order.payment.failed
    order.inventory.reserved
    order.shipped
    order.delivered

이 설계로 가능한 Binding:
  "order.#"          → 모든 주문 이벤트 (감사 서비스)
  "order.payment.*"  → 결제 관련 이벤트 (결제 모니터링)
  "#.failed"         → 모든 실패 이벤트 (알림 서비스)
  "order.placed"     → 주문 접수만 (정확히 하나)
  "#"                → 모든 이벤트 (중앙 로그 수집)

Spring AMQP 설정:
  @Bean
  public TopicExchange orderExchange() {
    return ExchangeBuilder
      .topicExchange("order.exchange")
      .durable(true)
      .build();
  }

  @Bean
  public Binding auditBinding() {
    return BindingBuilder
      .bind(auditQueue())
      .to(orderExchange())
      .with("order.#");  // 모든 주문 이벤트
  }

  @Bean
  public Binding failureAlertBinding() {
    return BindingBuilder
      .bind(alertQueue())
      .to(orderExchange())
      .with("#.failed");  // 모든 실패 이벤트
  }
```

---

## 🔬 내부 동작 원리

### 1. 와일드카드 매칭 규칙

```
=== * (Star) — 단어 하나 ===

"단어" = 점(.)으로 구분된 세그먼트 하나

패턴 "order.*":
  "order.placed"       ✅ (order + placed = 2세그먼트)
  "order.cancelled"    ✅
  "order.payment"      ✅
  "order.payment.failed" ❌ (3세그먼트, *는 하나만)
  "order"              ❌ (세그먼트 1개, *자리 없음)

패턴 "order.*.failed":
  "order.payment.failed"   ✅
  "order.inventory.failed" ✅
  "order.failed"           ❌ (* 자리가 없음)
  "order.a.b.failed"       ❌ (*는 단 하나의 세그먼트)

=== # (Hash) — 0개 이상의 단어 ===

패턴 "order.#":
  "order"              ✅ (0개 단어)
  "order.placed"       ✅ (1개 단어)
  "order.payment.failed" ✅ (2개 단어)
  "order.a.b.c.d"     ✅ (4개 단어)
  "payment"            ❌ (order로 시작 안 함)

패턴 "#.failed":
  "failed"             ✅ (#가 0개를 의미 가능)
  "payment.failed"     ✅
  "order.payment.failed" ✅
  "failed.payment"     ❌ (failed로 끝나야 함)

패턴 "#":
  모든 Routing Key에 매칭 → Fanout처럼 동작

=== 복합 패턴 매칭 ===

Routing Key: "order.payment.failed"

Binding 패턴들:
  "order.#"              → ✅ (order + payment.failed)
  "order.payment.*"      → ✅ (order.payment + failed)
  "order.payment.failed" → ✅ (정확 매칭)
  "#.failed"             → ✅ (order.payment + failed)
  "*.payment.*"          → ✅ (order + payment + failed)
  "#.payment.#"          → ✅
  "order.*"              → ❌ (세그먼트 2개가 아님)
  "payment.#"            → ❌ (payment로 시작 안 함)

→ 5개 패턴 모두 매칭 시 5개 Queue에 메시지 복사 전달
```

### 2. Topic Exchange 내부 매칭 구현

```
=== 트라이(Trie) 기반 매칭 ===

RabbitMQ는 Binding Key를 Trie(접두사 트리) 구조로 관리하여
패턴 매칭을 효율적으로 수행합니다.

Binding Key들:
  "order.#"
  "order.payment.*"
  "#.failed"

Trie 구조 (간략화):
  root
  ├── "order"
  │   ├── "#" → {queue: audit.queue}
  │   └── "payment"
  │       └── "*" → {queue: payment-monitor.queue}
  └── "#"
      └── "failed" → {queue: alert.queue}

Routing Key "order.payment.failed" 매칭:
  1. 세그먼트 분리: ["order", "payment", "failed"]
  2. Trie 순회:
     - "order" → 매칭 노드 진입
     - "#" 노드: 나머지 "payment.failed" 모두 소비 → audit.queue 추가
     - "payment" → 매칭 노드 진입
     - "*" 노드: "failed" 하나 소비 → payment-monitor.queue 추가
  3. "#.failed" 패턴: 역방향 매칭도 처리
  4. 결과: [audit.queue, payment-monitor.queue, alert.queue]

성능:
  Direct Exchange: O(1) 해시
  Topic Exchange:  O(B * L) — Binding 수 × Key 길이
  → Binding이 많을수록 라우팅 비용 증가
  → 실무: Binding 수백 개까지는 무시할 수준
```

### 3. Routing Key 설계 가이드

```
=== 계층 설계 원칙 ===

권장 형식: <도메인>.<집합>.<이벤트>

도메인: order, payment, inventory, user, notification
집합:   서브 도메인 또는 집계 (optional)
이벤트: placed, cancelled, failed, completed, updated

예시:
  order.placed
  order.payment.requested
  order.payment.completed
  order.payment.failed
  order.inventory.reserved
  order.inventory.insufficient
  payment.card.approved
  payment.card.declined
  user.registered
  user.profile.updated

이 구조로 표현 가능한 구독 패턴:
  "order.#"            → 모든 주문 관련
  "order.payment.#"    → 주문-결제 관련
  "#.failed"           → 전 도메인 실패
  "#.completed"        → 전 도메인 완료
  "order.*"            → 주문 최상위 이벤트 (order.placed, order.shipped 등)
  "#"                  → 전체 이벤트 수집

=== 설계 실수 방지 체크리스트 ===

1. 세그먼트는 소문자 + 하이픈 또는 점으로 구분
   ✅ "order.payment.failed"
   ❌ "Order_PaymentFailed"  ← 계층 없음

2. 너무 깊은 계층은 피함 (최대 4~5단계)
   ✅ "order.payment.failed"
   ❌ "service.domain.subdomain.entity.action.result"

3. 와일드카드로 그룹화 가능한지 검토 후 설계
   "같은 패턴 구독자가 필요한 이벤트들은 같은 prefix를 공유"

4. 이벤트 이름은 과거형 (이미 발생한 사건)
   ✅ "order.placed" (주문이 접수됐다)
   ❌ "place.order" (명령형)
```

---

## 💻 실전 실험

### 실험 1: 와일드카드 매칭 직접 확인

```bash
# Topic Exchange 생성
rabbitmqadmin declare exchange name=order.exchange type=topic durable=true

# Queue 생성
rabbitmqadmin declare queue name=audit.queue durable=true
rabbitmqadmin declare queue name=payment-monitor.queue durable=true
rabbitmqadmin declare queue name=alert.queue durable=true

# Binding 생성
rabbitmqadmin declare binding source=order.exchange \
  destination=audit.queue routing_key="order.#"
rabbitmqadmin declare binding source=order.exchange \
  destination=payment-monitor.queue routing_key="order.payment.*"
rabbitmqadmin declare binding source=order.exchange \
  destination=alert.queue routing_key="#.failed"

# 테스트 1: "order.payment.failed" 발행
rabbitmqadmin publish exchange=order.exchange \
  routing_key="order.payment.failed" payload='{"orderId":1}'

rabbitmqctl list_queues name messages
# audit.queue           1  ← order.# 매칭
# payment-monitor.queue 1  ← order.payment.* 매칭
# alert.queue           1  ← #.failed 매칭

# 테스트 2: "order.placed" 발행 (* 한 세그먼트)
rabbitmqadmin publish exchange=order.exchange \
  routing_key="order.placed" payload='{"orderId":2}'

rabbitmqctl list_queues name messages
# audit.queue           2  ← order.# 매칭
# payment-monitor.queue 1  ← order.payment.* 불일치 (payment 없음)
# alert.queue           1  ← #.failed 불일치 (failed 없음)

# 테스트 3: "payment.failed" 발행 (다른 도메인 실패)
rabbitmqadmin publish exchange=order.exchange \
  routing_key="payment.failed" payload='{"paymentId":99}'

rabbitmqctl list_queues name messages
# alert.queue  2  ← #.failed 매칭 (order 없어도 됨)
# 나머지 변화 없음
```

### 실험 2: Topic Exchange에서 * vs # 차이 실험

```bash
# 패턴 비교 실험용 Queue
rabbitmqadmin declare queue name=star.queue durable=true
rabbitmqadmin declare queue name=hash.queue durable=true

rabbitmqadmin declare binding source=order.exchange \
  destination=star.queue routing_key="order.*"
rabbitmqadmin declare binding source=order.exchange \
  destination=hash.queue routing_key="order.#"

# 2세그먼트: order.placed
rabbitmqadmin publish exchange=order.exchange \
  routing_key="order.placed" payload="2seg"
# star.queue: 1 ✅, hash.queue: 1 ✅

# 3세그먼트: order.payment.failed
rabbitmqadmin publish exchange=order.exchange \
  routing_key="order.payment.failed" payload="3seg"
# star.queue: 1 ← 변화 없음 ❌ (*는 하나만)
# hash.queue: 2 ← 증가 ✅

# 1세그먼트: order (자체)
rabbitmqadmin publish exchange=order.exchange \
  routing_key="order" payload="1seg"
# star.queue: 1 ← 변화 없음 ❌ (*자리 없음)
# hash.queue: 3 ← 증가 ✅ (#는 0개 이상)
```

---

## 📊 Kafka vs RabbitMQ 비교

```
=== 패턴 기반 이벤트 구독 비교 ===

요구사항: "결제 실패 이벤트만 알림 서비스로"

RabbitMQ Topic Exchange:
  Binding: "#.failed" → alert.queue
  → 모든 *.failed Routing Key가 자동 라우팅
  → 새 도메인의 실패 이벤트도 자동 포함

Kafka Consumer:
  topic="order-events" 전체 구독 → filter(event.type == "payment.failed")
  또는 전용 topic="order-payment-failed-events" 생성
  
  필터링 방식:
  - KafkaStreams filter(): 브로커가 아닌 Consumer에서 필터
  - 전용 Topic: 발행 시 Producer가 라우팅 결정

핵심 차이:
  RabbitMQ: 라우팅 로직이 브로커에 있음 (Binding)
            Consumer는 자신에게 필요한 메시지만 받음
  Kafka:    라우팅 로직이 Consumer에 있음 (filter)
            Consumer가 전체를 받아 필터링하거나
            Producer가 전용 Topic에 발행

=== 유연성 비교 ===

"새 서비스가 '모든 실패 이벤트'를 구독하고 싶다"

RabbitMQ:
  새 Queue + Binding("#.failed") 추가
  → 즉시 모든 실패 이벤트 수신 시작 (기존 Producer 변경 없음)
  단, 과거 메시지 소급 불가

Kafka:
  새 Consumer Group 생성
  offset=earliest로 과거 메시지부터 소비 가능
  → 과거 실패 이벤트도 재처리 가능
```

---

## ⚖️ 트레이드오프

```
Topic Exchange의 장단점:

장점:
  ① 와일드카드로 유연한 구독 표현
  ② Binding 수 최소화 (패턴 하나로 다수 이벤트 포함)
  ③ Producer 변경 없이 새 구독자 추가 가능

단점:
  ① Direct보다 라우팅 비용 높음 (패턴 매칭)
  ② 잘못 설계된 패턴 → 의도치 않은 Queue에 전달
  ③ Routing Key 설계를 처음부터 잘 해야 나중에 문제 없음
  ④ "#"이나 너무 넓은 패턴 → 성능 저하

Binding 수와 성능:
  Binding < 100개: 무시할 수준
  Binding 1000개+: 라우팅 지연 측정 필요
  → 실무에서 대부분 문제 없음
```

---

## 📌 핵심 정리

```
Topic Exchange 핵심:

와일드카드 규칙:
  * = 점으로 구분된 세그먼트 정확히 하나
  # = 0개 이상의 세그먼트 (점 포함)

Routing Key 설계:
  <도메인>.<집합>.<이벤트> 형식
  소문자, 점(.) 구분
  과거형 이벤트 이름 (placed, failed, cancelled)

매칭 결과:
  여러 패턴에 동시 매칭 → 모든 해당 Queue에 복사 전달
  매칭 없음 → 폐기 (mandatory=false 기본)

적합한 사용 사례:
  도메인 이벤트 라우팅 (마이크로서비스 이벤트 버스)
  장애/실패 이벤트 중앙 수집 ("#.failed")
  도메인별 감사 로그 수집 ("order.#")
```

---

## 🤔 생각해볼 문제

**Q1.** `"#"` 패턴 하나로 Exchange에 Binding한 Queue는 어떤 메시지를 받는가? 이것이 Fanout Exchange와 기능적으로 동일한가?

<details>
<summary>해설 보기</summary>

`"#"` 패턴은 0개 이상의 모든 세그먼트에 매칭됩니다. 즉, **모든 Routing Key에 매칭**됩니다.

기능적으로 Fanout과 비슷하지만 차이가 있습니다:
- **Fanout Exchange**: Routing Key를 완전히 무시. Binding Key 자체가 없음. 성능 최고
- **Topic `"#"`**: `"#"` 패턴이 모든 Key에 매칭. 매칭 연산은 수행

실용적 차이:
- 하나의 Topic Exchange에서 일부 Queue는 `"order.#"`, 다른 Queue는 `"#"` 패턴 혼용 가능
- Fanout Exchange는 모든 Queue에 무조건 전달 (선택 불가)
- `"#"` 패턴은 다른 Binding과 공존 가능

결론: 기능적으로 유사하지만 목적이 다릅니다. 전체 이벤트를 하나의 Queue에 수집하는 용도라면 `"#"` 패턴 Topic Exchange보다 Fanout Exchange가 더 명확합니다.

</details>

---

**Q2.** Routing Key `"a..b"` (점이 연속으로 두 번)를 발행하면 어떻게 되는가? `"*"` 패턴이 빈 세그먼트에 매칭되는가?

<details>
<summary>해설 보기</summary>

RabbitMQ는 Routing Key를 점(`.`)으로 분리하여 세그먼트 배열을 만듭니다. `"a..b"`는 `["a", "", "b"]` — 빈 문자열 세그먼트가 생깁니다.

`"*"` 패턴은 **세그먼트 하나(빈 문자열 포함)**에 매칭됩니다. 따라서:
- `"*.*.*"` 패턴은 `"a..b"`에 매칭됩니다 (`*`가 빈 문자열에도 매칭)

실무적으로는 `"a..b"` 같은 이중 점 Routing Key는 실수이므로 발생하지 않아야 합니다. Routing Key 검증 로직을 Publisher에 추가하는 것이 권장됩니다:

```java
private void validateRoutingKey(String key) {
  if (key.contains("..") || key.startsWith(".") || key.endsWith(".")) {
    throw new IllegalArgumentException("Invalid routing key: " + key);
  }
}
```

</details>

---

**Q3.** 동일한 Routing Key `"order.payment.failed"`가 `"order.#"`, `"order.payment.*"`, `"#.failed"` 세 패턴 모두에 매칭된다면, Consumer는 같은 메시지를 세 번 처리해야 하는가?

<details>
<summary>해설 보기</summary>

**각 Binding이 다른 Queue에 연결되어 있다면 세 Queue가 각각 메시지 복사본을 받습니다.**

- `"order.#"` → `audit.queue` → Consumer A
- `"order.payment.*"` → `payment-monitor.queue` → Consumer B
- `"#.failed"` → `alert.queue` → Consumer C

각 Consumer는 **다른 Queue에서 독립적으로 처리**합니다. 같은 메시지를 "세 번 처리"하는 것이 아니라 세 가지 목적으로 각각 처리하는 것입니다.

**문제가 되는 경우**: 세 Binding이 **같은 Queue**에 연결된 경우
```bash
rabbitmqadmin declare binding ... routing_key="order.#"          destination=same.queue
rabbitmqadmin declare binding ... routing_key="order.payment.*"  destination=same.queue  # 중복 가능성
rabbitmqadmin declare binding ... routing_key="#.failed"          destination=same.queue
```
`"order.payment.failed"` 발행 → `same.queue`에 **최대 3개** 메시지 (패턴마다 하나씩)
→ Consumer가 같은 메시지를 3번 처리 → 멱등성 필수

이를 방지하려면 하나의 Queue에 중복 매칭되는 패턴을 걸지 않도록 설계해야 합니다.

</details>

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[이전: Direct Exchange ⬅️](./01-direct-exchange.md)** | **[다음: Fanout Exchange ➡️](./03-fanout-exchange.md)**

</div>
