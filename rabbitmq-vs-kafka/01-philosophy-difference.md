# 근본적 철학 차이 완전 분해 — 메시지 브로커 vs 분산 로그

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- RabbitMQ와 Kafka는 각각 어떤 문제를 해결하기 위해 설계됐는가?
- "Consumer가 처리하면 삭제" vs "로그로 보관"의 차이가 시스템 설계에 어떤 영향을 주는가?
- Push 기반 소비(RabbitMQ)와 Pull 기반 소비(Kafka)의 트레이드오프는 무엇인가?
- "Smart Broker / Dumb Consumer" vs "Dumb Broker / Smart Consumer" 철학의 실무적 차이는?
- 두 시스템 중 하나를 선택할 때 가장 먼저 물어야 하는 질문은 무엇인가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

"RabbitMQ vs Kafka 중 어떤 것을 써야 하나요?"는 가장 자주 받는 질문 중 하나다. 기능 목록을 비교해서는 답이 나오지 않는다. 두 시스템은 해결하려는 문제 자체가 다르다. RabbitMQ는 "메시지를 올바른 Consumer에게 빠르게 전달"에 최적화됐고, Kafka는 "대량의 이벤트를 순서대로 영구 보관하고 누구든지 읽을 수 있게"에 최적화됐다. 이 철학 차이를 이해하면 기능 비교 없이도 상황에 맞는 선택을 할 수 있다.

---

## 😱 흔한 실수 (Before — 철학 차이를 모를 때)

```
실수 1: Kafka로 RabbitMQ를 대체하려다 복잡성 폭발

  "요청-응답 패턴이 필요하다"
  Kafka로 구현:
    요청 Topic + 응답 Topic + Correlation ID + Consumer Group
    → 수백 줄의 코드, 오프셋 관리, 파티션 제약
    
  RabbitMQ로 구현:
    Reply-To Queue + Correlation ID
    → 수십 줄, 브로커가 라우팅 담당

실수 2: RabbitMQ로 이벤트 소싱을 구현하려다 실패

  "주문 서비스의 모든 이벤트를 여러 팀이 각자의 속도로 읽어야 한다"
  RabbitMQ로 구현:
    Consumer가 처리 후 삭제 → 새 팀이 과거 이벤트 읽기 불가
    각 팀마다 Queue 복사 → 메모리 낭비
    
  Kafka로 구현:
    Consumer Group마다 독립 오프셋 → 각자의 속도로 과거부터 읽기
    → 이것이 Kafka가 설계된 목적

실수 3: "Kafka가 더 빠르니까 RabbitMQ를 Kafka로 교체"

  낮은 레이턴시 알림 서비스 (< 10ms):
    Kafka: 배치 처리 최적화 → 10~50ms 레이턴시
    RabbitMQ: Push 기반 → 1~5ms 레이턴시
    → 레이턴시가 우선이면 RabbitMQ가 유리
```

---

## ✨ 올바른 접근 (After — 철학으로 선택)

```
선택 질문 1: 메시지를 처리 후 삭제할 수 있는가?

  YES → RabbitMQ 가능 (처리 후 삭제가 기본 모델)
  NO (과거 재처리, 다중 Consumer 재소비) → Kafka

선택 질문 2: 라우팅 로직이 복잡한가?

  YES (Exchange 패턴, 우선순위, RPC) → RabbitMQ
  NO (단순 Topic 구독) → Kafka 충분

선택 질문 3: Consumer 수가 고정적인가?

  YES → RabbitMQ (Queue별 Consumer)
  NO (새 팀이 언제든 과거부터 구독 가능해야 함) → Kafka

선택 질문 4: 초당 메시지가 수십만 이상인가?

  YES → Kafka (처리량 우선 설계)
  NO (수만 이하) → RabbitMQ 충분
```

---

## 🔬 내부 동작 원리

### 1. 핵심 철학 대비

```
=== RabbitMQ: 메시지 브로커 ===

설계 목적: "메시지를 올바른 Consumer에게 전달하고, 처리되면 삭제"

핵심 개념:
  Queue: 메시지를 Consumer가 처리할 때까지 임시 보관
  Consumer: Queue에서 메시지를 가져가 처리 → 처리 후 삭제
  Exchange: 메시지를 올바른 Queue로 라우팅
  Ack: "나 처리 완료, 삭제해도 돼"

메시지의 생애:
  발행 → Exchange → Queue → Consumer → Ack → 삭제
  메시지는 처리되면 사라짐 (영구 보관이 목적이 아님)

비유: 우편 시스템
  편지(메시지)는 수신자(Consumer)에게 전달되면 소임 완료
  우체국(Queue)은 임시 보관소
  편지 전달 후 우체국에 복사본 없음

=== Kafka: 분산 로그 ===

설계 목적: "대량의 이벤트를 순서대로 영구 보관, 누구든지 읽기 가능"

핵심 개념:
  Topic: 이벤트 스트림 (로그 파일처럼)
  Partition: Topic을 병렬 처리 단위로 분할
  Offset: Consumer가 어디까지 읽었는지 기록
  Consumer Group: 독립적인 읽기 커서

메시지의 생애:
  발행 → Topic Partition에 append → Consumer가 오프셋으로 읽기
  메시지는 보관 기간(retention)까지 유지 (처리 여부 무관)
  여러 Consumer Group이 같은 메시지를 독립적으로 읽기 가능

비유: 신문 구독 시스템
  신문(이벤트)은 날짜별로 보관
  구독자(Consumer Group)가 각자의 속도로 읽기
  어제 신문을 오늘 읽어도 OK (보관 기간 내)
  새 구독자가 창간호부터 읽기도 가능
```

### 2. Push vs Pull 소비 방식

```
=== RabbitMQ: Push 기반 ===

브로커가 Consumer에게 메시지를 "밀어 넣음(Push)"

동작:
  Consumer: basicConsume(queue) 등록
  RabbitMQ: Consumer가 준비되면 메시지 자동 전달
  → Consumer가 요청하지 않아도 메시지 도착

Prefetch로 유량 제어:
  basicQos(prefetch=10) → 최대 10개까지 Push
  Consumer 처리 속도 > 도착 속도면 Consumer가 놀기도 함

장점:
  낮은 레이턴시 (메시지 도착 → 즉시 Consumer에게)
  Consumer가 수동으로 polling 불필요

단점:
  Consumer가 느리면 메시지 폭발 위험 (Prefetch로 제어)
  브로커가 Consumer 상태를 추적해야 함 (복잡한 브로커)

=== Kafka: Pull 기반 ===

Consumer가 브로커에서 메시지를 "당겨옴(Pull)"

동작:
  Consumer: poll(timeout=100ms) 호출
  Kafka: 현재 오프셋 이후 메시지 반환 (없으면 대기)
  Consumer: 메시지 처리 → commitOffset → 다음 poll

배치 처리:
  poll()이 한 번에 max.poll.records 개수만큼 반환
  Consumer가 배치로 처리 → 높은 처리량

장점:
  Consumer가 자신의 속도에 맞게 처리 (브로커 과부하 없음)
  배치 처리로 높은 처리량

단점:
  레이턴시 높음 (poll 주기만큼 지연)
  Consumer가 반드시 활성 상태여야 함

=== 레이턴시 비교 ===

Push (RabbitMQ):
  메시지 발행 → Exchange → Queue → 즉시 Consumer에게 Push
  일반적 레이턴시: 1~5ms

Pull (Kafka):
  메시지 발행 → Partition append
  Consumer: 다음 poll() 때 수신 (poll 주기: 기본 100ms)
  일반적 레이턴시: 10~50ms
  
  linger.ms=0 + fetch.min.bytes=1 설정으로 줄일 수 있지만
  처리량과 트레이드오프
```

### 3. Smart Broker vs Dumb Broker 철학

```
=== RabbitMQ: Smart Broker, Dumb Consumer ===

브로커(RabbitMQ)가 복잡한 책임:
  - Exchange 유형별 라우팅 로직 (Direct, Topic, Fanout, Headers)
  - Consumer에게 Push, Prefetch 제어
  - Ack 상태 추적, 재전달 관리
  - DLX 처리, TTL 관리
  - Priority Queue, Lazy Queue

Consumer는 단순:
  - Queue에서 메시지 수신 → 처리 → Ack
  - 라우팅/분배 로직 불필요

결과:
  Consumer 코드 단순 (라우팅 로직 없음)
  브로커 복잡 (다양한 설정, 상태 추적)
  브로커가 과부하면 전체 영향

=== Kafka: Dumb Broker, Smart Consumer ===

브로커(Kafka)가 단순:
  - 메시지를 Partition에 append-only 저장
  - 오프셋 위치의 메시지 반환 (Consumer 요청 시)
  - 복제 (replication-factor)
  - 보관 기간 초과 메시지 삭제

Consumer(Client Library)가 복잡:
  - 어느 Partition을 읽을지 결정
  - 오프셋 추적 및 커밋
  - Consumer Group Rebalancing
  - 실패 시 재처리 오프셋 관리
  - Filter (브로커가 아닌 Consumer가 처리)

결과:
  브로커 단순 → 수평 확장 쉬움, 높은 처리량
  Consumer/Client Library 복잡 (Kafka Streams, KSQL 등 도구 필요)
  브로커 장애가 Consumer에 미치는 영향 줄어듦

=== 라우팅 로직 위치 ===

"결제 실패 이벤트만 알림팀이 받아야 한다"

RabbitMQ:
  Exchange Binding: "#.failed" → alert.queue
  → 브로커가 라우팅 (Consumer 코드 변경 없음)
  → 새 팀 추가: Queue + Binding만 추가

Kafka:
  알림팀 Consumer: 전체 이벤트 poll → "payment.failed"만 filter
  또는 별도 "payment-failed-events" Topic 생성 (Producer가 라우팅 결정)
  → 라우팅 로직이 Consumer 또는 Producer에 있음
  → 새 팀 추가: 새 Consumer Group (코드 배포 필요)
```

### 4. 메시지 보존과 재처리

```
=== RabbitMQ: 처리 후 삭제 ===

Consumer Ack → 메시지 삭제 (복원 불가)

재처리가 필요한 경우:
  Producer가 원본 데이터를 다시 발행해야 함
  또는 DB에서 재조회 후 재발행

새 Consumer 과거 이벤트 불가:
  재고팀이 신규 합류 → 지난 1달치 주문 이벤트 소급 불가
  → DB에서 직접 조회 필요

DLQ로 실패 메시지 보관:
  처리 실패 → DLQ에 보관 (성공 메시지는 삭제)
  DLQ에서 수동 재처리

=== Kafka: 보관 기간 내 유지 ===

Consumer 오프셋 커밋 후에도 메시지 유지
(보관 기간: retention.ms, 기본 7일)

재처리:
  Consumer Group: 오프셋 리셋으로 과거부터 재처리
  kafka-consumer-groups.sh --reset-offsets --to-earliest

새 Consumer 과거 이벤트 소급:
  재고팀 신규 합류 → offset=earliest 설정
  → 보관 기간 내 모든 이벤트 처음부터 처리 가능

특정 시각부터 재처리:
  --to-datetime "2024-03-15T00:00:00.000"
  → "어제 오후 3시부터 발생한 이벤트만 재처리"

이벤트 소싱(Event Sourcing):
  모든 상태 변경을 이벤트로 저장 (Kafka Topic)
  어느 시점의 상태든 이벤트 재생으로 복원 가능
  → Kafka가 이벤트 소싱에 적합한 이유
```

---

## 💻 실전 실험

### 실험 1: 철학 차이를 코드로 확인

```java
// RabbitMQ: Consumer가 메시지를 처리하면 삭제됨
@RabbitListener(queues = "order.queue")
public void handleOrder(OrderEvent event) {
  processOrder(event);
  // 메서드 종료 후 자동 Ack → 메시지 삭제
  // 다른 Consumer는 이 메시지를 절대 받지 못함
}

// ================================================================

// Kafka: 오프셋으로 같은 메시지를 여러 Consumer Group이 읽음
@KafkaListener(topics = "order-events", groupId = "inventory-group")
public void handleForInventory(OrderEvent event) {
  reserveInventory(event);
  // 처리 후 오프셋 커밋 → 메시지는 Kafka에 남아있음
}

@KafkaListener(topics = "order-events", groupId = "notification-group")
public void handleForNotification(OrderEvent event) {
  sendNotification(event);
  // 같은 메시지를 다른 Consumer Group이 독립적으로 처리
}
// inventory-group과 notification-group 모두 같은 메시지 처리
// RabbitMQ처럼 두 Queue에 복사 없이 하나의 이벤트로
```

### 실험 2: 과거 메시지 재처리

```bash
# Kafka: 오프셋 리셋으로 과거 재처리
kafka-consumer-groups.sh \
  --bootstrap-server kafka:9092 \
  --group inventory-group \
  --topic order-events \
  --reset-offsets --to-earliest \
  --execute

# → inventory-group이 처음부터 모든 이벤트 재처리

# ================================================================

# RabbitMQ: 과거 메시지 재처리 방법 없음
# 이미 Ack된 메시지는 삭제됨
# → DB에서 데이터 조회 후 재발행만 가능
rabbitmqadmin publish exchange=order.exchange routing_key="order.placed" \
  payload='{"orderId":1,...}'  # 재발행 필요
```

---

## 📊 핵심 철학 비교표

```
특성                    | RabbitMQ                    | Kafka
───────────────────────┼─────────────────────────────┼─────────────────────────────
설계 목적               | 메시지 전달 + 삭제           | 이벤트 로그 보관 + 재소비
메시지 보존             | 처리 후 삭제                 | 보관 기간 동안 유지
소비 방식               | Push (브로커가 밀어넣음)     | Pull (Consumer가 당겨옴)
브로커 역할             | Smart (라우팅, 상태 추적)    | Dumb (저장 + 반환)
Consumer 역할           | Dumb (Queue에서 수신 처리)  | Smart (오프셋 관리, 필터)
레이턴시                | 낮음 (1~5ms)                | 높음 (10~50ms)
처리량                  | 높음 (수만 msg/sec)          | 매우 높음 (수백만 msg/sec)
재처리                  | 불가 (삭제됨)                | 가능 (오프셋 리셋)
새 Consumer 과거 소급   | 불가                         | 가능 (보관 기간 내)
라우팅 복잡도            | 브로커가 처리 (유연)         | Topic 단위 (단순)
순서 보장               | Queue 내 FIFO               | Partition 내 FIFO
```

---

## ⚖️ 트레이드오프

```
RabbitMQ Push 방식:
  ✅ 낮은 레이턴시 (즉시 Push)
  ✅ Consumer 코드 단순 (polling 없음)
  ❌ Consumer 느리면 Prefetch로 제어 필요
  ❌ 브로커 복잡 (상태 추적)

Kafka Pull 방식:
  ✅ Consumer 자신의 속도로 처리 (브로커 과부하 없음)
  ✅ 배치 처리로 높은 처리량
  ❌ 레이턴시 높음 (poll 주기)
  ❌ Consumer 복잡 (오프셋 관리)

메시지 삭제 vs 보관:
  RabbitMQ 삭제:
    ✅ 스토리지 효율 (처리 후 삭제)
    ❌ 재처리 불가
    ❌ 새 Consumer 과거 소급 불가

  Kafka 보관:
    ✅ 재처리 가능 (오프셋 리셋)
    ✅ 새 Consumer 과거부터 읽기 가능
    ❌ 스토리지 사용 증가 (보관 기간 동안)
```

---

## 📌 핵심 정리

```
철학 차이 핵심:

RabbitMQ = 우편 시스템:
  메시지 → Consumer에게 전달 → 삭제
  브로커가 라우팅/상태 관리 (Smart Broker)
  Push 방식 → 낮은 레이턴시

Kafka = 신문 구독 시스템:
  이벤트 → Topic에 보관 → 누구든지 읽기 가능
  Consumer가 오프셋 관리 (Smart Consumer)
  Pull 방식 → 높은 처리량, 재처리 가능

가장 중요한 판단 기준:
  "메시지 처리 후 삭제해도 되는가?"
  YES → RabbitMQ 가능
  NO (재처리, 과거 소급 필요) → Kafka

"브로커에 복잡한 라우팅이 필요한가?"
  YES (Exchange 패턴, RPC, 우선순위) → RabbitMQ
  NO (단순 Topic 구독) → Kafka로 충분
```

---

## 🤔 생각해볼 문제

**Q1.** "결제 서비스가 처리한 이벤트를 재고 서비스와 알림 서비스가 동시에 받아야 한다"는 요구사항에서, RabbitMQ와 Kafka 중 어느 것이 더 효율적인가?

<details>
<summary>해설 보기</summary>

**메모리 효율면에서 Kafka가 더 효율적입니다.** 단, 기능적으로는 둘 다 가능합니다.

**RabbitMQ 방식:**
- Fanout Exchange → inventory.queue + notification.queue (두 Queue에 메시지 복사)
- 메시지 1개 → 복사본 2개 (메모리 × 2)
- 처리 완료 → 각 Queue에서 삭제

**Kafka 방식:**
- order-events Topic → inventory-group + notification-group (두 Consumer Group)
- 메시지 1개 → 복사 없이 두 Group이 독립적으로 읽기
- 메모리 효율적 (복사 없음)
- 추가로 analytics-group, audit-group이 붙어도 메시지 복사 없음

**선택 기준:**
- 재고팀과 알림팀만이고 메시지 처리 후 삭제 OK → RabbitMQ 충분
- 새 팀이 언제든 붙고 과거 이벤트도 읽어야 함 → Kafka

</details>

---

**Q2.** Kafka의 Pull 방식에서 Consumer가 poll()을 아주 느리게 호출한다. 이때 발생할 수 있는 문제는?

<details>
<summary>해설 보기</summary>

**Consumer Group에서 해당 Consumer가 강제 퇴출됩니다.**

Kafka Consumer는 `session.timeout.ms`(기본 45초) 내에 브로커에 Heartbeat를 보내야 합니다. poll() 호출 사이의 간격이 `max.poll.interval.ms`(기본 5분)를 초과하면 브로커가 해당 Consumer를 "응답 없음"으로 간주하고 Consumer Group에서 제거합니다.

결과:
1. Rebalancing 발생 → 다른 Consumer가 해당 Partition 인수
2. 느린 Consumer가 처리 완료 후 커밋 시도 → 거부됨 (이미 Rebalancing됨)
3. 다른 Consumer가 같은 메시지를 재처리 → 중복 처리 가능성

**해결:**
```java
// 배치 크기를 줄여 poll당 처리 시간 단축
spring.kafka.consumer.max-poll-records=50  // 기본 500 → 50으로 줄임

// 또는 poll 간격 제한 늘리기 (처리 시간이 정말 길면)
spring.kafka.consumer.max-poll-interval-ms=600000  // 10분
```

</details>

---

**Q3.** "RabbitMQ는 Kafka보다 항상 레이턴시가 낮다"는 주장이 맞는가? 어떤 상황에서 Kafka가 RabbitMQ보다 낮은 레이턴시를 달성할 수 있는가?

<details>
<summary>해설 보기</summary>

**항상 맞지는 않습니다.** 특정 설정과 상황에서 Kafka가 RabbitMQ와 비슷하거나 낮은 레이턴시를 달성할 수 있습니다.

Kafka 레이턴시 최적화 설정:
```properties
# Producer: 즉시 전송 (배치 없음)
linger.ms=0
batch.size=1

# Consumer: 즉시 fetch
fetch.min.bytes=1
fetch.max.wait.ms=0
```

이 설정에서 Kafka 레이턴시: 5~15ms (RabbitMQ Push의 1~5ms와 비교)

여전히 RabbitMQ가 낮은 이유:
- Kafka는 Partition Leader에 쓰고 ISR 복제 후 Consumer에게 보임 (복제 오버헤드)
- acks=all이면 ISR 모두 확인 → 추가 지연
- RabbitMQ는 Queue에 쓰고 Consumer에게 즉시 Push

**Kafka가 유리한 경우:**
- 대용량 처리 (수십만~수백만 msg/sec): Kafka의 배치 처리가 총 처리량에서 RabbitMQ를 능가
- 레이턴시보다 처리량이 우선인 경우

결론: 단일 메시지 레이턴시 = RabbitMQ 우위. 총 처리량 = Kafka 우위.

</details>

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[이전: Chapter 5 — 클러스터 운영 ⬅️](../performance-operations/05-cluster-operations.md)** | **[다음: 사용 사례 비교 ➡️](./02-use-case-comparison.md)**

</div>
