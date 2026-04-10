# Headers Exchange — 메시지 헤더 기반 라우팅

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Headers Exchange는 어떤 기준으로 메시지를 Queue에 라우팅하는가?
- `x-match=all`과 `x-match=any`는 각각 어떤 조건으로 매칭하는가?
- Headers Exchange가 Direct/Topic Exchange보다 유연한 경우는 언제인가?
- 헤더 값의 타입(String, Integer, Boolean)이 라우팅에 영향을 주는가?
- Headers Exchange의 성능과 복잡성 트레이드오프는?

---

## 🔍 왜 이 개념이 실무에서 중요한가

Routing Key는 단일 문자열이다. "결제 방법이 카드이면서, 금액이 100만원 이상이고, VIP 고객인 주문"을 Direct나 Topic으로 표현하려면 Routing Key가 복잡해지거나 여러 Exchange가 필요하다. Headers Exchange는 메시지의 임의 헤더를 라우팅 기준으로 사용한다. 조건이 복잡하고 다차원적일 때 Routing Key 단일 문자열의 한계를 극복하는 도구다. 하지만 과도한 사용은 관리 복잡성을 크게 높인다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: x-match 설정 없이 Binding

  // x-match를 생략하면 기본값: all
  rabbitmqadmin declare binding source=order.headers \
    destination=vip.queue \
    arguments='{"customerType":"vip","paymentMethod":"card"}'
  
  발행:
    headers: {customerType: "vip"} 만 포함
    → x-match=all → 두 헤더 모두 일치 필요 → 매칭 실패 ❌
  
  예상: "vip이니까 vip.queue로 가겠지"
  실제: paymentMethod 헤더 없으므로 미매칭

실수 2: 헤더 값 타입 불일치

  // Binding Arguments (String 값):
  {"amount": "1000000"}  ← String "1000000"
  
  // 발행 시 헤더 (Integer 값):
  MessageProperties props = new MessageProperties();
  props.setHeader("amount", 1000000);  ← Integer 1000000
  
  결과:
    String "1000000" != Integer 1000000
    → 타입 불일치로 매칭 실패 ❌

실수 3: Headers Exchange로 간단한 라우팅 구현

  Direct로 충분한 것을 Headers로:
  
  Binding: {service: "payment"}  → payment.queue
  Binding: {service: "inventory"} → inventory.queue
  
  발행:
    headers: {service: "payment"}
    → payment.queue로 라우팅
  
  이것은 Direct Exchange로 훨씬 단순하게 구현 가능:
    routingKey="payment" → payment.queue
  
  Headers Exchange는 Routing Key로 표현하기 어려운
  다차원 조건에서만 사용해야 함
```

---

## ✨ 올바른 접근 (After — 다차원 조건 라우팅)

```
Headers Exchange 적합한 사용 사례:

사례: 결제 방식 + 고객 등급 + 금액대 조합별 라우팅

Binding 설정:
  x-match=all, {paymentMethod:"card", customerType:"vip"}
    → vip-card.queue (VIP 카드 결제)
  
  x-match=all, {paymentMethod:"card", amount:"large"}
    → large-card.queue (대금액 카드 결제)
  
  x-match=any, {paymentMethod:"bitcoin", customerType:"suspicious"}
    → fraud-check.queue (결제수단 또는 의심 고객)

발행:
  headers: {paymentMethod:"card", customerType:"vip", amount:"large"}
  → vip-card.queue ✅ (all: card ✅, vip ✅)
  → large-card.queue ✅ (all: card ✅, large ✅)
  → fraud-check.queue ❌ (any: bitcoin ❌, suspicious ❌)

Spring AMQP 설정:
  @Bean
  public Binding vipCardBinding() {
    Map<String, Object> args = new HashMap<>();
    args.put("x-match", "all");
    args.put("paymentMethod", "card");
    args.put("customerType", "vip");
    
    return BindingBuilder
      .bind(vipCardQueue())
      .to(headersExchange())
      .whereAll(args).match();
  }
```

---

## 🔬 내부 동작 원리

### 1. Headers Exchange 라우팅 알고리즘

```
=== 메시지 헤더 매칭 ===

메시지 구조:
  Exchange에 도달할 때 헤더(AMQP Content Header 프레임)를 함께 전달
  
  MessageProperties에 설정한 헤더들:
    content-type, delivery-mode, ... (AMQP 표준 헤더)
    + 사용자 정의 헤더: paymentMethod, customerType, amount 등

Binding Arguments:
  x-match: "all" 또는 "any"  ← 필수
  나머지 key-value 쌍: 라우팅 조건

=== x-match=all (AND 조건) ===

Binding: {x-match:"all", paymentMethod:"card", customerType:"vip"}

매칭 조건:
  메시지 헤더에 paymentMethod="card" 있어야 함 AND
  메시지 헤더에 customerType="vip" 있어야 함

발행 테스트:
  {paymentMethod:"card", customerType:"vip"}  → ✅ (둘 다 있음)
  {paymentMethod:"card", customerType:"regular"} → ❌ (regular != vip)
  {paymentMethod:"card"}                      → ❌ (customerType 없음)
  {paymentMethod:"card", customerType:"vip", extra:"ok"} → ✅ (추가 헤더 무관)

=== x-match=any (OR 조건) ===

Binding: {x-match:"any", paymentMethod:"bitcoin", customerType:"suspicious"}

매칭 조건:
  메시지 헤더에 paymentMethod="bitcoin" 있거나 OR
  메시지 헤더에 customerType="suspicious" 있거나

발행 테스트:
  {paymentMethod:"bitcoin"}            → ✅ (bitcoin 있음)
  {customerType:"suspicious"}          → ✅ (suspicious 있음)
  {paymentMethod:"bitcoin", customerType:"suspicious"} → ✅ (둘 다 있음)
  {paymentMethod:"card", customerType:"vip"} → ❌ (둘 다 불일치)

=== 헤더 값 비교 ===

값 비교는 타입과 값 모두 일치해야 함:
  Binding: {amount: "1000000"}   ← String
  메시지:  {amount: "1000000"}   → ✅
  메시지:  {amount: 1000000}     → ❌ (Integer vs String)

지원 타입:
  String, Integer, Long, Boolean, Float, Double, BigDecimal
  → 발행 시 헤더 타입과 Binding Arguments 타입을 맞춰야 함

값 없는 헤더 (존재 여부만 확인):
  Binding: {x-match:"all", requiredFlag: ""}  ← 빈 문자열
  → 메시지에 requiredFlag 헤더가 존재하기만 하면 매칭 (값 무관)
  → Header의 존재 여부 조건으로 활용
```

### 2. Headers Exchange vs Topic Exchange 비교

```
=== 같은 조건, 다른 접근 ===

요구사항: "카드 결제 + VIP 고객"을 vip-card.queue로 라우팅

Topic Exchange로 구현:
  Routing Key: "payment.card.vip"
  Binding Key: "payment.*.vip"  → vip-card.queue
  
  한계:
    "카드 결제이거나 VIP인 경우"(OR)를 Routing Key 하나로 표현 불가
    조건이 늘어날수록 Routing Key 형식 복잡해짐
    "카드 결제이면서 100만원 이상이면서 VIP이면서 첫 구매"
    → Routing Key: "payment.card.large.vip.first" (순서 중요, 관리 어려움)

Headers Exchange로 구현:
  Binding: {x-match:all, paymentMethod:card, customerType:vip}
  메시지 헤더: {paymentMethod:card, customerType:vip, amount:1500000}
  
  유연성:
    헤더는 순서 무관 (딕셔너리)
    조건 추가: Binding Arguments에 키-값 추가
    OR 조건: x-match=any
    다른 Queue: 다른 Binding Arguments 조합
    → 다차원 조건 표현에 Routing Key보다 자연스러움

=== Headers Exchange가 불리한 경우 ===

성능:
  Direct: O(1) 해시
  Topic:  O(B × L)
  Headers: O(B × H) — Binding 수 × 헤더 쌍 수

가시성:
  Routing Key: Exchange → Queue 흐름을 한눈에 파악 가능
  Headers: 메시지 내부 헤더를 알아야 라우팅 추적 가능
  → Management UI에서 Binding 보기만으로 이해하기 어려움

사용 빈도:
  실무에서 Headers Exchange는 드물게 사용
  대부분 Topic Exchange로 충분
  진짜 다차원 조건이 필요할 때만 사용 권장
```

### 3. 실무 사용 패턴

```
=== 적합한 사용 사례 ===

1. 국가/언어별 라우팅:
   {x-match:all, region:"KR", language:"ko"} → kr-ko.queue
   {x-match:all, region:"US", language:"en"} → us-en.queue

2. 환경/버전별 라우팅:
   {x-match:all, env:"prod", version:"v2"} → prod-v2.queue
   {x-match:all, env:"prod", version:"v1"} → prod-v1.queue

3. 컨텐츠 타입별 라우팅:
   {x-match:all, contentType:"image/jpeg"} → image.queue
   {x-match:all, contentType:"video/mp4"}  → video.queue

4. 우선순위 복합 조건:
   {x-match:all, priority:"high", type:"payment"} → urgent.queue
   {x-match:any, priority:"critical"}             → alert.queue

=== 부적합한 사용 사례 ===

단순 서비스 구분:
  {service:"payment"} → payment.queue
  → Direct Exchange: routingKey="payment"이 훨씬 단순

단순 이벤트 유형:
  {event:"order.placed"} → placed.queue
  → Topic Exchange: routingKey="order.placed"이 더 명확

규칙: 조건이 1차원(단일 값 비교)이면 Direct/Topic
      조건이 다차원(여러 헤더의 AND/OR 조합)이면 Headers 검토
```

---

## 💻 실전 실험

### 실험 1: x-match=all 동작 확인

```bash
# Headers Exchange 생성
rabbitmqadmin declare exchange name=order.headers type=headers durable=true

# Queue 생성
rabbitmqadmin declare queue name=vip-card.queue durable=true
rabbitmqadmin declare queue name=fraud-check.queue durable=true

# x-match=all Binding
rabbitmqadmin declare binding source=order.headers \
  destination=vip-card.queue \
  arguments='{"x-match":"all","paymentMethod":"card","customerType":"vip"}'

# x-match=any Binding
rabbitmqadmin declare binding source=order.headers \
  destination=fraud-check.queue \
  arguments='{"x-match":"any","paymentMethod":"bitcoin","customerType":"suspicious"}'

# 테스트 1: VIP + 카드 → vip-card.queue 매칭
rabbitmqadmin publish exchange=order.headers routing_key="" \
  properties='{"headers":{"paymentMethod":"card","customerType":"vip"}}' \
  payload='{"orderId":1}'

rabbitmqctl list_queues name messages
# vip-card.queue   1  ← x-match=all: card ✅, vip ✅
# fraud-check.queue 0 ← x-match=any: 둘 다 불일치

# 테스트 2: bitcoin 결제 → fraud-check.queue 매칭
rabbitmqadmin publish exchange=order.headers routing_key="" \
  properties='{"headers":{"paymentMethod":"bitcoin","customerType":"normal"}}' \
  payload='{"orderId":2}'

rabbitmqctl list_queues name messages
# fraud-check.queue 1 ← x-match=any: bitcoin ✅
```

### 실험 2: Spring AMQP Headers Binding 설정

```java
@Configuration
public class HeadersExchangeConfig {

  @Bean
  public HeadersExchange orderHeaders() {
    return ExchangeBuilder
      .headersExchange("order.headers")
      .durable(true)
      .build();
  }

  @Bean
  public Queue vipCardQueue() {
    return QueueBuilder.durable("vip-card.queue").build();
  }

  @Bean
  public Binding vipCardBinding() {
    return BindingBuilder
      .bind(vipCardQueue())
      .to(orderHeaders())
      .whereAll("paymentMethod", "card")  // 단일 헤더 매칭
      .and("customerType", "vip")         // AND 추가 (x-match=all)
      .match();
  }
}

// 헤더 포함 발행
@Service
public class OrderPublisher {
  @Autowired private RabbitTemplate rabbitTemplate;
  
  public void publish(OrderEvent event) {
    MessageProperties props = new MessageProperties();
    props.setHeader("paymentMethod", event.getPaymentMethod()); // String
    props.setHeader("customerType", event.getCustomerType());   // String
    props.setHeader("amount", event.getAmount());               // Integer
    
    Message message = MessageBuilder
      .withBody(objectMapper.writeValueAsBytes(event))
      .andProperties(props)
      .build();
    
    rabbitTemplate.send("order.headers", "", message);
  }
}
```

---

## 📊 Kafka vs RabbitMQ 비교

```
=== 다차원 조건 필터링 비교 ===

RabbitMQ Headers Exchange:
  브로커가 헤더 조건으로 라우팅
  Consumer는 자신의 조건에 맞는 메시지만 받음
  → 네트워크 효율적 (불필요한 메시지 전송 없음)

Kafka:
  헤더 기반 라우팅 개념 없음
  Consumer가 전체 토픽 구독 후 조건 필터링
  → KafkaStreams, KSQL로 필터링 가능
  → 브로커는 단순 저장, Consumer가 필터링 책임

Headers Exchange:
  ✅ 네트워크 효율 (필터링된 메시지만 전달)
  ❌ 브로커 부하 (복잡한 헤더 매칭)

Kafka Consumer 필터:
  ✅ 브로커 단순 (저장만)
  ❌ Consumer가 불필요한 메시지도 수신
  ✅ 재처리 시 조건 변경 용이 (Consumer 코드만 변경)
```

---

## ⚖️ 트레이드오프

```
Headers Exchange 장단점:

장점:
  ① 다차원 조건 라우팅 (AND/OR 조합)
  ② Routing Key 포맷 제약 없음 (임의 헤더 키-값)
  ③ 헤더 순서 무관 (딕셔너리 기반)

단점:
  ① Direct/Topic보다 복잡한 설정
  ② 타입 불일치로 인한 매칭 실패 위험
  ③ Management UI에서 라우팅 추적 어려움
  ④ 성능: 헤더 매칭 비용 (헤더 수만큼)
  ⑤ 실무 사용 빈도 낮음 → 팀 내 이해도 낮을 수 있음

사용 결정 기준:
  Topic으로 표현 가능 → Topic 사용 (더 단순)
  조건이 진짜 다차원(2개 이상 속성 AND/OR) → Headers 검토
```

---

## 📌 핵심 정리

```
Headers Exchange 핵심:

라우팅 기준:
  Routing Key 무시
  메시지 헤더의 키-값 쌍으로 Binding Arguments와 비교

x-match:
  all: 모든 키-값 쌍 일치 (AND)
  any: 하나 이상 키-값 쌍 일치 (OR)

주의:
  헤더 값 타입 일치 필수 (String vs Integer 구분)
  x-match 설정 누락 시 기본값 all

적합한 경우:
  다차원 조건 (여러 속성의 AND/OR 조합)
  Routing Key 단일 문자열로 표현 불가한 복합 조건

부적합한 경우:
  단순 서비스/이벤트 구분 → Direct/Topic으로 충분
  성능 민감 → Direct/Topic이 빠름
```

---

## 🤔 생각해볼 문제

**Q1.** Headers Exchange Binding에 `{x-match:"all", key1:"val1", key2:"val2"}`가 설정돼 있다. 발행 메시지 헤더에 `{key1:"val1", key2:"val2", key3:"val3"}`가 있다면 매칭되는가?

<details>
<summary>해설 보기</summary>

**매칭됩니다.** x-match=all은 "Binding Arguments의 모든 키-값이 메시지 헤더에 존재하면" 매칭입니다.

- `key1:"val1"` → 메시지에 있고 값 일치 ✅
- `key2:"val2"` → 메시지에 있고 값 일치 ✅
- `key3:"val3"` → Binding에 없으므로 평가 안 함 (무관)

즉, **메시지 헤더가 Binding 조건의 슈퍼셋이면 매칭됩니다.** 추가 헤더는 라우팅에 영향을 주지 않습니다.

이를 활용하면: 메시지에 다양한 메타데이터 헤더를 포함하고, 각 Binding은 자신이 관심 있는 헤더 조건만 정의할 수 있습니다.

</details>

---

**Q2.** Headers Exchange에서 `x-match=all`과 `x-match=any`를 동시에 표현하는 복합 조건 `"(A AND B) OR C"`를 단일 Binding으로 표현할 수 있는가?

<details>
<summary>해설 보기</summary>

**단일 Binding으로는 불가능합니다.** x-match는 "all" 또는 "any" 둘 중 하나만 선택 가능합니다.

`"(A AND B) OR C"` 구현 방법:

**방법 1: 두 개의 Binding**
```
Binding 1: {x-match:all, A:valA, B:valB} → target.queue
Binding 2: {x-match:all, C:valC}         → target.queue
```
같은 Queue에 두 Binding → "Binding 1 OR Binding 2" = "(A AND B) OR C"

**방법 2: Consumer 필터링**
더 단순한 Binding으로 메시지를 받은 후 Consumer에서 조건 판단

RabbitMQ Headers Exchange는 복잡한 불리언 로직보다 단순한 AND/OR 조건이 기본 설계입니다. 복잡한 조건이 필요하면 Consumer 측 필터링이 더 명확할 수 있습니다.

</details>

---

**Q3.** Headers Exchange Binding에서 `x-match="all"`이고 조건이 10개인 경우와, 조건이 2개인 경우의 라우팅 성능 차이는 어느 정도이며, 실무에서 우려해야 하는 수준인가?

<details>
<summary>해설 보기</summary>

Headers Exchange 매칭 비용: **O(B × H)** — Binding 수 × 헤더 조건 수

조건 10개 vs 2개:
- 이론상 5배 차이
- 실제: 헤더 매칭은 딕셔너리 조회 (O(1)/조건)이므로 절대값은 매우 낮음

**실무에서 우려할 수준이 아닙니다** — 단, 전제 조건이 있습니다:
- Binding 수가 수백 개 미만
- 초당 수만 메시지 이하

Headers Exchange가 성능 병목이 되는 경우:
- Binding 1000개 이상
- 헤더 조건 20개 이상
- 초당 수십만 메시지

이런 극단적 상황이라면 Headers Exchange 대신 Application-level 필터링 또는 Topic Exchange + Consumer 필터를 고려하세요.

결론: 일반적인 서비스에서 Headers Exchange의 성능 차이는 무시할 수준입니다. 복잡성(설정 오류, 디버깅 어려움)이 더 큰 실무 문제입니다.

</details>

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[이전: Fanout Exchange ⬅️](./03-fanout-exchange.md)** | **[다음: Dead Letter Exchange ➡️](./05-dead-letter-exchange.md)**

</div>
