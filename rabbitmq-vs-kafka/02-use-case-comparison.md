# 사용 사례 비교 — RabbitMQ와 Kafka가 각각 빛나는 상황

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- RabbitMQ가 확실히 유리한 구체적 사용 사례는 무엇인가?
- Kafka가 확실히 유리한 구체적 사용 사례는 무엇인가?
- 복잡한 라우팅이 필요할 때 왜 RabbitMQ가 더 적합한가?
- 이벤트 소싱과 로그 집계에서 Kafka가 필수적인 이유는?
- 잘못된 선택이 나중에 어떤 문제를 일으키는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

올바른 도구를 선택하지 못하면 나중에 재설계 비용이 발생한다. "Kafka로 시작했는데 요청-응답 패턴 때문에 결국 RabbitMQ를 추가했다"거나 "RabbitMQ로 이벤트 소싱을 구현하다가 재처리가 안 돼서 Kafka로 마이그레이션했다"는 사례가 실무에서 자주 발생한다. 이 문서는 "이 상황이면 이것"이라는 명확한 판단 기준을 제공한다.

---

## 😱 흔한 실수 (Before)

```
실수 1: Kafka로 RPC 구현 시도

  Kafka 환경에서 재고 조회 RPC 구현:
    요청 Topic: "inventory-request"
    응답 Topic: "inventory-response-{clientId}"
    
  문제:
    Topic 수 폭발 (클라이언트마다 응답 Topic)
    레이턴시: 50~100ms (RabbitMQ는 1~5ms)
    구현 복잡 (Partition 관리, 응답 매핑)
    
  → RabbitMQ RPC가 이 목적에 더 적합

실수 2: RabbitMQ로 1년치 로그 재분석 시도

  "1년치 주문 이벤트를 새 분석 서비스로 재처리"
  RabbitMQ로는 불가능 (이미 Ack된 메시지 삭제됨)
  → DB에서 1년치 데이터를 다시 발행해야 함
  → 발행량: 수억 건 → 브로커 과부하
  
  Kafka였다면:
    새 Consumer Group이 retention 내 이벤트부터 재소비
    (retention이 1년이면 바로 재처리 가능)
```

---

## ✨ 올바른 접근 (After — 사용 사례별 선택)

```
RabbitMQ 선택 시나리오:
  ① 복잡한 라우팅이 필요한 이벤트 기반 시스템
  ② 요청-응답(RPC) 패턴
  ③ 메시지 우선순위 처리
  ④ 낮은 레이턴시 요구 (< 10ms)
  ⑤ 작업 큐 (이미지 처리, PDF 생성 등)
  ⑥ 규모가 작거나 중간 (초당 수만 메시지 이하)

Kafka 선택 시나리오:
  ① 이벤트 소싱 (Event Sourcing)
  ② 로그 집계 (Log Aggregation)
  ③ 대용량 스트리밍 (초당 수십만~수백만 메시지)
  ④ 여러 Consumer Group이 과거부터 독립 소비
  ⑤ 데이터 파이프라인 (ETL, CDC)
  ⑥ 재처리/재분석이 자주 필요한 시스템
```

---

## 🔬 내부 동작 원리

### 1. RabbitMQ가 적합한 사용 사례

```
=== 사례 1: 복잡한 라우팅 ===

요구사항:
  - VIP 고객의 결제 실패는 긴급팀과 CS팀 모두에게
  - 일반 고객의 결제 실패는 CS팀에게만
  - 모든 결제 이벤트는 감사팀에게
  
RabbitMQ 구현:
  Exchange: payment.exchange (Topic)
  Binding:
    "#.failed.vip" → vip-alert.queue  (긴급팀)
    "#.failed.*"   → cs.queue         (CS팀: VIP 포함 모든 실패)
    "payment.#"    → audit.queue      (감사팀: 모든 결제)
  
  발행 시 Routing Key: "payment.failed.vip" 또는 "payment.failed.regular"
  → Exchange가 자동 라우팅

Kafka로 구현하면:
  Consumer가 전체 구독 후 직접 필터링 (코드에 라우팅 로직)
  또는 별도 Topic 3개 생성 (Producer가 라우팅 결정)
  → 라우팅 변경 시 코드 배포 필요

=== 사례 2: 요청-응답(RPC) ===

요구사항:
  - OrderService가 재고 가용 여부를 InventoryService에 동기 조회
  - 응답 시간: 500ms 이내
  
RabbitMQ RPC:
  - Reply-To Queue + Correlation ID
  - 레이턴시: 1~5ms (HTTP 직접 호출과 유사)
  - Worker 자동 확장 (Queue 공유)
  
Kafka로 구현하면:
  - 요청/응답 Topic 필요
  - 레이턴시: 50~200ms (Poll 주기 포함)
  - → 요청-응답 패턴에는 RabbitMQ 또는 HTTP가 적합

=== 사례 3: 작업 큐 (Work Queue) ===

요구사항:
  - 이미지 리사이징 작업을 여러 Worker가 병렬 처리
  - 작업 처리 실패 시 DLQ에 보관 후 재처리
  - 처리 실패한 작업이 유실되면 안 됨

RabbitMQ:
  - image.resize.queue → Worker N개 경쟁 소비
  - DLX + DLQ: 실패 메시지 자동 보관
  - Prefetch=1: 공정 분배
  - autoAck=false: Worker 장애 시 재전달

Kafka:
  - 파티션 수 = Worker 수 (Worker 늘리면 Rebalancing)
  - DLQ는 애플리케이션 레벨 구현 (Spring Kafka의 DeadLetterPublishingRecoverer)
  - → 작업 큐 패턴은 RabbitMQ가 더 자연스럽게 지원

=== 사례 4: 메시지 우선순위 ===

요구사항:
  - VIP 주문은 일반 주문보다 먼저 처리
  - 같은 Consumer가 두 유형 모두 처리 (코드 공유)

RabbitMQ Priority Queue:
  x-max-priority=10
  VIP: priority=10, 일반: priority=1
  → 같은 Queue, 자동 우선순위 처리

Kafka:
  우선순위 미지원 (FIFO만)
  → 별도 Topic + Consumer 비율 조정으로 우회 구현
  → 복잡한 구현 필요
```

### 2. Kafka가 적합한 사용 사례

```
=== 사례 1: 이벤트 소싱(Event Sourcing) ===

요구사항:
  - 주문의 모든 상태 변경 이벤트를 영구 보관
  - 어느 시점의 주문 상태든 이벤트 재생으로 복원
  - 새 분석 서비스가 과거 이벤트부터 처리

Kafka 구현:
  Topic: order-events (retention: 1년)
  이벤트: OrderPlaced, PaymentCompleted, Shipped, Delivered, Cancelled
  
  새 분석 서비스:
    Consumer Group: analytics-group
    offset: earliest
    → 1년치 이벤트 처음부터 처리 가능

RabbitMQ로는:
  처리 후 삭제 → 과거 이벤트 재처리 불가
  이벤트 소싱의 핵심 특성을 지원할 수 없음
  → 이벤트 소싱: Kafka 필수

=== 사례 2: 로그 집계 ===

요구사항:
  - 100개 마이크로서비스가 초당 수십만 건의 로그 발행
  - Elasticsearch, S3, 실시간 대시보드가 각각 독립적으로 소비
  - 일부 Consumer가 다운되어도 재처리 가능

Kafka 구현:
  Topic: service-logs
  Consumer Group A: Elasticsearch (실시간 인덱싱)
  Consumer Group B: S3 (배치 저장)
  Consumer Group C: Dashboard (실시간 집계)
  
  초당 수십만 건 처리량 → Kafka가 적합
  Consumer Group B가 다운됐다가 복구 → 오프셋 기반 재처리

RabbitMQ로는:
  Fanout Exchange → 3개 Queue (메시지 3배 복사)
  초당 수십만 건 × 3배 = 브로커 메모리 압박
  Consumer B 다운 중 Queue 무한 적체 → 메모리/디스크 과부하
  → 대용량 로그 집계: Kafka 적합

=== 사례 3: CDC(Change Data Capture) / 데이터 파이프라인 ===

요구사항:
  - DB 변경사항을 실시간으로 DW(데이터웨어하우스), 검색 엔진, 캐시에 반영
  - 각 대상 시스템이 다른 처리 속도

Kafka + Debezium:
  Debezium이 DB WAL을 읽어 Kafka에 발행
  Topic: orders.changes
  Consumer A: PostgreSQL → DW (배치, 느림)
  Consumer B: PostgreSQL → Elasticsearch (실시간)
  Consumer C: PostgreSQL → Redis (캐시 동기화)
  
  각 Consumer의 오프셋 독립 → 처리 속도 차이 흡수

RabbitMQ로는:
  Debezium → Kafka → RabbitMQ로 변환 (불필요한 중간 단계)
  또는 별도 CDC 도구 + RabbitMQ → 복잡도 증가

=== 사례 4: 대용량 스트리밍 ===

요구사항:
  - 초당 100만 건의 사용자 행동 이벤트 처리
  - 실시간 추천 엔진이 소비

Kafka:
  Partition 30개 → 초당 수백만 메시지 처리
  append-only 로그 → 순차 쓰기 (SSD 최적)
  Consumer가 배치로 pull → 높은 처리량

RabbitMQ:
  초당 수만~수십만 메시지까지 실용적
  초당 100만 건 → 메모리, Flow Control 문제
  → 대용량 스트리밍: Kafka 필수
```

### 3. 중간 지대 — 상황에 따른 판단

```
=== 중간 규모 이벤트 버스 ===

요구사항:
  - 마이크로서비스 5개가 이벤트를 주고받음
  - 초당 수천 건
  - 재처리는 가끔만 필요

판단:
  RabbitMQ: Topic Exchange로 이벤트 버스 구현
             복잡한 라우팅 가능, 낮은 운영 복잡도
  Kafka:    단순 Topic 구독으로 충분
             재처리 가능성이 있으면 Kafka 장점

결론: 재처리 필요성이 판단 기준
  재처리 필요 → Kafka
  재처리 불필요 + 복잡한 라우팅 → RabbitMQ

=== 알림 서비스 ===

요구사항:
  - 주문 이벤트 발생 시 실시간 Push 알림 (< 5ms 목표)
  - 알림 발송 실패 시 재시도

판단:
  RabbitMQ: Push 방식 → 낮은 레이턴시
             TTL + DLX로 재시도 가능
  Kafka:    Pull 방식 → 10~50ms 레이턴시
             → 5ms 목표 달성 어려움

결론: 낮은 레이턴시 → RabbitMQ

=== 배치 데이터 처리 ===

요구사항:
  - 매일 자정 전날 주문 데이터를 DW에 적재
  - 데이터 양: 하루 수백만 건

판단:
  실시간이 아닌 배치 → Kafka Stream 또는 직접 DB 쿼리가 적합
  RabbitMQ: 배치 처리 패턴 지원 어려움
  Kafka: 오프셋 기반 배치 소비 가능
  → 하지만 이 경우 Spark, Flink 같은 배치 처리 도구가 더 적합
```

---

## 💻 실전 실험

### 실험: 동일 요구사항을 두 시스템으로 구현 비교

```java
// 요구사항: "결제 완료 이벤트를 재고팀과 알림팀이 처리, 실패 시 재처리 가능"

// === RabbitMQ 구현 ===
// Fanout Exchange → 두 Queue (메시지 복사)
@Bean FanoutExchange paymentExchange() { ... }
@Bean Queue inventoryQueue() { return QueueBuilder.durable("inventory.payment").build(); }
@Bean Queue notificationQueue() { return QueueBuilder.durable("notification.payment").build(); }

// 재처리: DLQ에서 수동 재발행 (처리 완료된 메시지 재처리 불가)

// === Kafka 구현 ===
// Topic 하나 → 두 Consumer Group (메시지 복사 없음)
@KafkaListener(topics = "payment-events", groupId = "inventory-group")
public void inventoryHandler(PaymentEvent event) { reserveInventory(event); }

@KafkaListener(topics = "payment-events", groupId = "notification-group")
public void notificationHandler(PaymentEvent event) { sendNotification(event); }

// 재처리: 오프셋 리셋으로 언제든지 가능
// kafka-consumer-groups.sh --reset-offsets --to-datetime "2024-03-15T00:00:00"
```

---

## 📊 사용 사례별 선택 가이드

```
사용 사례                    | 권장      | 이유
────────────────────────────┼──────────┼──────────────────────────────
요청-응답(RPC)               | RabbitMQ | 낮은 레이턴시, Reply-To Queue
복잡한 라우팅               | RabbitMQ | Exchange 패턴 (Topic/Headers)
메시지 우선순위              | RabbitMQ | x-max-priority 네이티브 지원
작업 큐 (Worker 패턴)       | RabbitMQ | DLX, Prefetch, 경쟁 소비
낮은 레이턴시 알림           | RabbitMQ | Push 방식 (1~5ms)
이벤트 소싱                  | Kafka    | 메시지 보존 + 오프셋 재소비
로그 집계                    | Kafka    | 고처리량 + 다중 Consumer Group
CDC / 데이터 파이프라인     | Kafka    | Debezium 통합, 재처리
대용량 스트리밍 (> 10만/sec) | Kafka    | 처리량 최적화 설계
여러 팀 독립 소비 + 재처리   | Kafka    | Consumer Group 오프셋
감사 로그 보관               | Kafka    | 장기 보존 + 재분석
소규모 마이크로서비스 통합   | RabbitMQ | 단순 운영, 낮은 복잡도
```

---

## ⚖️ 트레이드오프

```
잘못된 선택의 비용:

Kafka를 RabbitMQ 대신 잘못 선택:
  복잡한 라우팅 → Consumer 코드에 라우팅 로직 (유지보수 어려움)
  요청-응답 → Topic 수 폭발, 높은 레이턴시
  우선순위 → 복잡한 우회 구현

RabbitMQ를 Kafka 대신 잘못 선택:
  재처리 필요 → DB에서 재발행 (추가 개발 비용)
  대용량 → 메모리 압박, Flow Control
  이벤트 소싱 → 근본적으로 구현 불가 → 재설계 비용
```

---

## 📌 핵심 정리

```
사용 사례 선택 기준:

RabbitMQ가 답:
  복잡한 라우팅, RPC, 우선순위, 작업 큐, 낮은 레이턴시
  메시지 처리 후 삭제해도 되는 경우
  초당 수만 건 이하 규모

Kafka가 답:
  이벤트 소싱, 로그 집계, CDC, 대용량 스트리밍
  여러 Consumer Group이 독립적으로 소비해야 하는 경우
  과거 이벤트 재처리가 필요한 경우
  초당 수십만 건 이상 규모

가장 빠른 판단:
  "처리 후 메시지를 삭제해도 되는가?" → NO면 Kafka
  "재처리/과거 소급이 필요한가?" → YES면 Kafka
  "나머지는?" → RabbitMQ로 시작, 필요시 Kafka 추가
```

---

## 🤔 생각해볼 문제

**Q1.** "마이크로서비스 3개가 서로 이벤트를 주고받는다. 초당 이벤트는 1000건이다." RabbitMQ와 Kafka 중 어떤 것을 선택하고 그 이유는?

<details>
<summary>해설 보기</summary>

**이 정보만으로는 명확히 결정하기 어렵습니다.** 추가 질문이 필요합니다:

"재처리가 필요한가?" → 가장 중요한 질문
- 예: Kafka (오프셋 리셋으로 재처리)
- 아니오: RabbitMQ 충분

초당 1000건은 두 시스템 모두 처리 가능한 수준이므로 처리량은 결정 요소가 아닙니다.

**RabbitMQ를 선택하는 경우:**
- 재처리 불필요
- 서비스 간 복잡한 라우팅 규칙이 있을 때
- 팀이 RabbitMQ에 익숙하고 빠른 구현이 필요할 때
- 운영 단순성 우선 (Kafka는 Zookeeper/KRaft 등 추가 운영 부담)

**Kafka를 선택하는 경우:**
- 과거 이벤트 재처리 가능성이 있을 때
- 4번째 서비스가 나중에 합류할 때 (Consumer Group으로 간단히 추가)
- 이벤트 히스토리가 비즈니스 가치를 가질 때

일반 원칙: 불확실하면 RabbitMQ로 시작, 재처리 필요성이 생기면 Kafka 추가 검토.

</details>

---

**Q2.** 현재 RabbitMQ로 주문 이벤트를 처리하는 서비스가 있다. "새 AI 분석 팀이 지난 6개월치 주문 데이터부터 분석을 시작하고 싶다"는 요구가 들어왔다. RabbitMQ에서 이를 어떻게 처리할 수 있는가?

<details>
<summary>해설 보기</summary>

**RabbitMQ에서 직접 처리할 수 없습니다.** 처리된 메시지는 이미 삭제됐습니다.

현실적인 대안:

**방법 1: DB에서 재발행**
```java
// 6개월치 주문을 DB에서 조회해 분석 Queue에 발행
@Scheduled(fixedDelay = 100)
public void publishHistoricalOrders() {
  List<Order> orders = orderRepository.findFromLast6Months(lastPublishedId, BATCH_SIZE);
  orders.forEach(order -> rabbitTemplate.convertAndSend("analytics.queue", order));
  lastPublishedId = orders.get(orders.size()-1).getId();
}
```
단점: 많은 시간과 발행 비용, 브로커 부하

**방법 2: Outbox 테이블 활용**
이미 Outbox Pattern을 쓰고 있다면 outbox_events 테이블에 6개월치 이벤트가 남아있을 수도 있음 → 재발행 가능

**방법 3: Kafka 도입 (근본적 해결)**
향후 이런 요구가 반복될 것이 예상되면, 이번 기회에 이벤트 스트림을 Kafka로 이관. RabbitMQ는 실시간 작업 처리에 유지.

이것이 RabbitMQ와 Kafka를 함께 운영하는 실제 동기 중 하나입니다.

</details>

---

**Q3.** "우리 서비스는 초당 500만 건의 IoT 센서 데이터를 받는다. 이 데이터를 실시간 이상 감지, 데이터 저장, 대시보드 시각화 3개 서비스가 소비한다." RabbitMQ 또는 Kafka 중 어떤 것을 선택하는가?

<details>
<summary>해설 보기</summary>

**Kafka가 압도적으로 적합합니다.** 이유:

1. **처리량**: 초당 500만 건 × 3개 Consumer = 1500만 ops/sec
   - RabbitMQ: Fanout Exchange → 3개 Queue = 데이터 3배 복사 → 브로커 한계 초과
   - Kafka: 3개 Consumer Group이 동일 파티션을 독립 읽기 = 복사 없음

2. **재처리**: IoT 이상 감지 알고리즘 업데이트 → 과거 데이터 재분석 필요
   - RabbitMQ: 불가
   - Kafka: 오프셋 리셋으로 재처리 가능

3. **Consumer 독립성**: 이상 감지가 느려도 데이터 저장에 영향 없어야 함
   - RabbitMQ: 느린 Consumer Queue 적체 → 브로커 메모리 압박
   - Kafka: 각 Consumer Group의 오프셋 독립 → 서로 영향 없음

4. **스케일**: 파티션 수를 늘려 선형 확장 (Kafka 설계 목적)

결론: 초당 수십만 이상, 여러 Consumer Group, 재처리 필요 → **Kafka 명확한 선택**.

</details>

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[이전: 근본적 철학 차이 ⬅️](./01-philosophy-difference.md)** | **[다음: 성능 특성 비교 ➡️](./03-performance-comparison.md)**

</div>
