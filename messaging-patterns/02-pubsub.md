# Publish/Subscribe — 하나의 이벤트, 여러 독립 처리

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Pub/Sub에서 각 Consumer가 자신만의 Queue를 가져야 하는 이유는?
- 영구 구독과 임시 구독은 어떻게 다르게 설계하는가?
- Consumer 추가 시 기존 서비스에 어떤 영향이 있는가?
- 로그 스트리밍과 서비스 이벤트 배포에 각각 어떤 Queue 유형을 쓰는가?
- Work Queue와 Pub/Sub을 잘못 혼용하면 어떤 문제가 생기는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

"주문 이벤트 하나를 재고팀, 알림팀, 분석팀이 각각 독립적으로 처리해야 한다"는 요구사항은 Pub/Sub의 전형적인 사례다. Work Queue처럼 하나의 Queue를 공유하면 세 팀 중 하나만 메시지를 받는다. 각 팀이 자신만의 Queue를 가져야 한다. 더 나아가 실시간 대시보드처럼 Consumer가 오프라인일 때 메시지를 보관할 필요가 없는 경우에는 임시 Queue를 쓴다. 영구/임시 Queue의 차이를 이해해야 메모리를 낭비하지 않는 Pub/Sub 시스템을 설계할 수 있다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: 하나의 Queue를 여러 Consumer가 공유 (Work Queue와 혼용)

  order.fanout → order.event.queue
  
  Consumer A (재고팀): order.event.queue 구독
  Consumer B (알림팀): order.event.queue 구독

  결과:
    메시지 1개 → A 또는 B 중 하나만 수신 (경쟁 소비)
    A가 받으면 B는 못 받음 → 알림이 안 감
    "간헐적으로 재고는 업데이트되는데 알림이 안 온다"

  올바른 구조:
    order.fanout → inventory.queue (재고팀 전용)
    order.fanout → notification.queue (알림팀 전용)

실수 2: 실시간 뷰어에 영구 Queue 사용

  관리자 실시간 대시보드 (상시 연결 아님):
    order.dashboard.queue (durable=true) ← 영구 Queue
    
  문제:
    관리자 오프라인 → Queue에 메시지 무한 축적
    Queue 수십만 건 → 메모리/디스크 과부하
    관리자 접속 시 오래된 데이터 수백만 건 쏟아짐

  올바른 구조:
    exclusive=true + autoDelete=true → 연결 시 생성, 해제 시 삭제

실수 3: Consumer 추가 후 기존 메시지 소급 불가 인식 안 함

  분석팀 Queue 추가:
    order.fanout → analytics.queue (신규)

  기대: "이전 주문 데이터도 받아야지"
  실제: analytics.queue가 생성된 이후 발행분만 수신
         이전 메시지는 이미 다른 Queue들에서 처리/삭제됨
```

---

## ✨ 올바른 접근 (After — 목적별 Queue 유형 선택)

```
Pub/Sub Queue 유형 선택:

1. 영구 구독 (서비스 간 이벤트):
   Queue: durable=true, exclusive=false, autoDelete=false
   서비스 오프라인 → Queue에 메시지 보관
   재시작 후 보관된 메시지 처리 가능

   @Bean
   public Queue inventoryQueue() {
     return QueueBuilder.durable("inventory.order.queue")
       .deadLetterExchange("dlx.exchange")
       .build();
   }

2. 임시 구독 (실시간 스트리밍, 대시보드):
   Queue: durable=false, exclusive=true, autoDelete=true
   Consumer 연결 시 생성 → 해제 시 삭제
   오프라인 시 메시지 보관 없음 (놓친 메시지 포기)

   @RabbitListener(
     bindings = @QueueBinding(
       value = @Queue(exclusive = "true", autoDelete = "true", durable = "false"),
       exchange = @Exchange(value = "order.fanout", type = "fanout")
     )
   )
   public void liveOrderStream(OrderEvent event) {
     dashboard.push(event);
   }
```

---

## 🔬 내부 동작 원리

### 1. 독립적인 Queue가 필요한 이유

```
=== 메시지 복사 원리 ===

Fanout Exchange → 2개 Queue:
  order.fanout Exchange
    │
    ├── Binding → inventory.queue (복사본 1)
    └── Binding → notification.queue (복사본 2)

메시지 1개 발행:
  Exchange: 메시지 복사본 2개 생성
  inventory.queue에 복사본 1 저장
  notification.queue에 복사본 2 저장

재고팀 Consumer:
  inventory.queue에서 복사본 1 소비 → Ack → inventory.queue에서 삭제
  notification.queue의 복사본 2는 영향 없음

알림팀 Consumer:
  notification.queue에서 복사본 2 소비 → 독립적 처리

=== 복사본 메모리 ===

Exchange가 메시지를 N개 Queue에 전달:
  메모리: 메시지 크기 × N

Queue별 독립 처리 속도:
  inventory.queue: 재고팀 Consumer 처리 속도로 소비
  notification.queue: 알림팀 Consumer 처리 속도로 소비
  → 한 팀이 느려도 다른 팀에 영향 없음
  (단, 느린 Queue가 메모리 사용 증가 → 브로커 전체 영향 가능)

=== 새 구독자 추가 ===

Before:
  order.fanout → inventory.queue
  order.fanout → notification.queue

분석팀 추가:
  새 Queue 생성: analytics.queue (durable=true)
  새 Binding: order.fanout → analytics.queue
  AnalyticsService 배포

결과:
  기존 OrderService: 변경 없음
  기존 Consumer: 변경 없음
  analytics.queue: Binding 시점 이후 발행 메시지만 수신

과거 데이터 소급 처리:
  RabbitMQ에서는 불가능 (이미 처리/삭제됨)
  → 과거 데이터 재처리 필요 시 DB에서 직접 읽거나 Kafka 활용
```

### 2. 임시 Queue의 생명주기

```
=== exclusive + autoDelete Queue ===

특성:
  exclusive=true: 선언한 Connection에서만 접근 가능
  autoDelete=true: 마지막 Consumer가 해제하면 Queue 삭제

생명주기:
  Consumer 연결 → Queue 자동 생성 (또는 재사용)
              → Binding 생성 (Exchange에 연결)
  Consumer 운영 → 메시지 실시간 수신
  Consumer 종료 → Connection 닫힘
              → autoDelete: Queue 삭제
              → Binding 자동 삭제
  
  RabbitMQ: "잠깐만 구독하고 사라지는" Consumer에 최적

AMQP 동작:
  Consumer가 선언한 Queue 이름:
  - 직접 지정: "dashboard-session-123"
  - 자동 생성: RabbitMQ가 "amq.gen-Xfj1..." 형식으로 생성

Spring AMQP 자동 생성 Queue 이름:
  @Queue 속성에 name 없으면 RabbitMQ가 자동 생성
  앱 재시작마다 다른 Queue 이름 → Fanout Binding 재생성

=== 임시 Queue의 메모리 효율 ===

영구 Queue (대시보드가 오프라인):
  order.dashboard.queue: 10만 건 축적
  → 메모리 수백 MB 소비
  → 브로커 성능 영향

임시 Queue:
  Consumer 없으면 Queue 없음 → 메시지 없음
  Consumer 연결 시 Queue 생성 → 연결 시점 이후 메시지만 수신
  → 메모리 절약, 오래된 데이터 문제 없음
```

### 3. 다중 Exchange 조합 패턴

```
=== Topic Exchange + Fanout Exchange 조합 ===

특정 이벤트는 Topic으로 필터링, 전체 브로드캐스트는 Fanout:

order.topic.exchange (Topic)
  ├── "order.placed"     → order.placed.fanout (Fanout)
  │     ├── inventory.placed.queue
  │     ├── notification.placed.queue
  │     └── analytics.placed.queue
  │
  └── "order.cancelled"  → order.cancelled.fanout (Fanout)
        ├── inventory.cancelled.queue
        └── notification.cancelled.queue

이 구조로:
  Topic Exchange가 이벤트 유형별 라우팅
  Fanout Exchange가 서비스별 브로드캐스트
  → 세밀한 이벤트 구독 + 완전한 격리

=== 로그 시스템 구현 ===

모든 서비스 → logs.fanout Exchange
  ├── logs.error.queue → 에러 로그 수집기
  ├── logs.metrics.queue → 메트릭 수집기
  └── amq.gen-... (임시) → 개발자 실시간 로그 뷰어

로그 뷰어:
  개발자 연결 → 임시 Queue 생성 → logs.fanout Binding
  실시간 로그 수신
  개발자 종료 → 임시 Queue 삭제 → 로그 더 이상 보관 없음
```

---

## 💻 실전 실험

### 실험 1: 영구 Queue Pub/Sub 구성

```bash
# Fanout Exchange + 서비스별 영구 Queue
rabbitmqadmin declare exchange name=order.fanout type=fanout durable=true

rabbitmqadmin declare queue name=inventory.order.queue durable=true
rabbitmqadmin declare queue name=notification.order.queue durable=true
rabbitmqadmin declare queue name=analytics.order.queue durable=true

rabbitmqadmin declare binding source=order.fanout destination=inventory.order.queue routing_key=""
rabbitmqadmin declare binding source=order.fanout destination=notification.order.queue routing_key=""
rabbitmqadmin declare binding source=order.fanout destination=analytics.order.queue routing_key=""

# 메시지 발행
rabbitmqadmin publish exchange=order.fanout routing_key="" \
  payload='{"orderId":1,"amount":50000}'

rabbitmqctl list_queues name messages
# inventory.order.queue    1
# notification.order.queue 1
# analytics.order.queue    1  ← 3개 모두 수신
```

### 실험 2: 임시 Queue Pub/Sub (실시간 대시보드)

```java
@Component
public class RealtimeDashboard {

  // 연결 시 임시 Queue 자동 생성, 연결 해제 시 삭제
  @RabbitListener(
    bindings = @QueueBinding(
      value = @Queue(
        exclusive = "true",
        autoDelete = "true",
        durable = "false"
        // name 없음 → RabbitMQ가 "amq.gen-..." 자동 생성
      ),
      exchange = @Exchange(
        value = "order.fanout",
        type = ExchangeTypes.FANOUT,
        durable = "true"
      )
    )
  )
  public void onOrderEvent(OrderEvent event) {
    // 실시간 WebSocket으로 대시보드에 push
    websocketSession.send(event);
  }
}
// 앱 종료 시 amq.gen-... Queue 자동 삭제 → 메모리 절약
```

### 실험 3: Consumer 추가 시 기존 메시지 미수신 확인

```bash
# 기존 구성: inventory.order.queue만 있음
rabbitmqadmin publish exchange=order.fanout routing_key="" payload='{"orderId":1}'
rabbitmqadmin publish exchange=order.fanout routing_key="" payload='{"orderId":2}'

# 분석팀 Queue 추가 (이후 발행 메시지만 수신)
rabbitmqadmin declare queue name=analytics.order.queue durable=true
rabbitmqadmin declare binding source=order.fanout destination=analytics.order.queue routing_key=""

# 새 메시지 발행
rabbitmqadmin publish exchange=order.fanout routing_key="" payload='{"orderId":3}'

rabbitmqctl list_queues name messages
# inventory.order.queue    3  ← orderId 1,2,3 모두
# analytics.order.queue    1  ← orderId 3만 (이전 메시지 없음)
```

---

## 📊 Kafka vs RabbitMQ 비교

```
=== 같은 이벤트를 여러 서비스에 전달 ===

RabbitMQ Pub/Sub:
  메시지 1개 → N개 Queue에 복사
  각 Queue는 독립적인 Consumer가 소비
  오프라인 Consumer: Queue에 메시지 보관 (영구 Queue)
  새 구독자: Binding 시점 이후 메시지만 수신

Kafka Consumer Group:
  메시지 1개 Topic에 저장 (복사 없음)
  각 Consumer Group이 독립적인 오프셋으로 소비
  오프라인 Consumer Group: 오프셋이 남아있어 재연결 시 소급 소비
  새 구독자: earliest 오프셋 설정 시 과거 메시지 모두 소비 가능

=== 핵심 차이 ===

메시지 보관:
  RabbitMQ: 모든 Queue가 소비할 때까지 보관
  Kafka: 보관 기간 설정 (7일 등) → 모든 Consumer와 무관

새 구독자 과거 데이터:
  RabbitMQ: 소급 불가 (이미 처리/삭제됨)
  Kafka: 오프셋 earliest로 과거 데이터 소급 가능

이벤트 버스 패턴:
  새 팀이 "과거 1달치 이벤트부터 분석하고 싶다" → Kafka
  새 팀이 "지금부터 이벤트를 받겠다" → RabbitMQ 충분
```

---

## ⚖️ 트레이드오프

```
Queue 유형별 트레이드오프:

영구 Queue (durable):
  ✅ Consumer 오프라인 시 메시지 보관
  ✅ 재시작 후 미처리 메시지 처리 가능
  ❌ 느린 Consumer → Queue 무한 축적 → 메모리 압박

임시 Queue (exclusive + autoDelete):
  ✅ 메모리 절약 (Consumer 없으면 Queue 없음)
  ✅ 실시간 이벤트 뷰어에 적합
  ❌ Consumer 오프라인 시 메시지 소실
  ❌ 재연결 시 이전 메시지 받을 수 없음

Pub/Sub vs Work Queue:
  Pub/Sub: 메시지 복사 → 여러 서비스 독립 처리
  Work Queue: 메시지 하나를 Worker 중 하나가 처리
  
  같은 메시지를 여러 목적으로 처리 → Pub/Sub
  같은 작업을 여러 Worker가 나눠서 처리 → Work Queue
```

---

## 📌 핵심 정리

```
Pub/Sub 핵심:

패턴:
  Exchange → N개 Queue → N개 독립 Consumer
  각 Queue는 독립적인 메시지 복사본 보유

Queue 유형 선택:
  영구 Queue: durable=true (서비스 간 이벤트 배포)
  임시 Queue: exclusive=true + autoDelete=true (실시간 스트리밍)

Work Queue와 차이:
  Work Queue: Queue 공유 → 메시지 하나를 하나만 처리
  Pub/Sub: Queue 분리 → 메시지 복사 → 모두 독립 처리

새 구독자 추가:
  새 Queue + Binding 추가만으로 완료
  기존 Publisher/Consumer 변경 없음
  단, 추가 시점 이전 메시지는 소급 수신 불가

메모리 주의:
  느린 Consumer Queue → 메시지 축적 → 브로커 전체 영향
  느린 Consumer: Lazy Queue 적용 또는 Consumer Scale Out
```

---

## 🤔 생각해볼 문제

**Q1.** 5개 서비스가 Fanout Exchange를 구독한다. 1개 서비스 팀이 갑자기 서비스를 6개월 동안 운영 중단했다. 이 기간 동안 발행된 메시지는 어떻게 되는가?

<details>
<summary>해설 보기</summary>

**영구 Queue를 사용한다면**: 해당 서비스의 Queue에 6개월치 메시지가 쌓입니다. Queue Depth가 수백만 건이 될 수 있고, 브로커 메모리/디스크를 잠식합니다.

대응 방법:

**방법 1: Queue 용량 제한**
```bash
rabbitmqadmin declare queue name=paused-service.queue durable=true \
  arguments='{"x-max-length":10000,"x-overflow":"drop-head"}'
```
최대 10,000개 유지, 초과분은 가장 오래된 메시지 삭제.

**방법 2: Queue TTL 설정**
```bash
arguments='{"x-message-ttl":86400000}'  # 24시간 후 자동 삭제
```

**방법 3: 운영 중단 서비스 Queue 삭제**
```bash
rabbitmqadmin delete binding source=order.fanout destination=paused.queue
rabbitmqadmin delete queue name=paused.queue
```
서비스 재시작 시 Queue 재생성.

실무에서는 **Queue 용량 제한 + TTL**을 기본값으로 설정하고, 운영 중단 시 Binding을 제거하는 것이 권장됩니다.

</details>

---

**Q2.** 임시 Queue를 사용하는 실시간 대시보드가 재시작됐다. 재시작 사이에 발행된 메시지를 받을 수 있는 방법이 있는가?

<details>
<summary>해설 보기</summary>

**임시 Queue(exclusive + autoDelete)로는 불가능합니다.** 연결이 끊기는 순간 Queue가 삭제됩니다.

방법별 트레이드오프:

**방법 1: 영구 Queue로 전환**
- durable=true, exclusive=false, 이름 고정
- 재시작 후 보관된 메시지 수신 가능
- 단점: 대시보드 오프라인 시 메시지 무한 축적

**방법 2: 영구 Queue + TTL**
- x-message-ttl=300000 (5분)
- 5분치 메시지만 보관 → 재시작 후 최대 5분치 수신

**방법 3: 대시보드에서 DB 직접 조회로 보완**
- 재시작 감지 → DB에서 누락 기간 데이터 조회
- 임시 Queue는 실시간 이후 수신용으로만 사용

**실무 선택**: 대시보드 재시작 사이 데이터가 없어도 되면 임시 Queue 유지. 재시작 복구가 필요하면 TTL 설정 영구 Queue 사용.

</details>

---

**Q3.** 같은 서비스의 여러 인스턴스(Pod 3개)가 Pub/Sub을 구독할 때, 인스턴스마다 다른 Queue를 가져야 하는가 아니면 하나의 Queue를 공유해야 하는가?

<details>
<summary>해설 보기</summary>

**서비스의 역할에 따라 다릅니다.**

**하나의 Queue 공유 (Work Queue + Pub/Sub 혼합):**
```
order.fanout → notification.queue (하나)
               ↑ NotificationService Pod 1, 2, 3이 공유 소비
```
- 메시지 하나를 Pod 중 하나만 처리 (Work Queue 방식)
- 중복 처리 없음
- Pod Scale Out 시 처리량 증가
- **권장**: 각 서비스가 메시지당 한 번만 처리해야 할 때

**인스턴스마다 다른 Queue (진정한 Pub/Sub):**
```
order.fanout → notification-pod1.queue → Pod 1
             → notification-pod2.queue → Pod 2
             → notification-pod3.queue → Pod 3
```
- 모든 Pod가 모든 메시지를 처리
- 예: 각 Pod가 자신의 캐시를 갱신해야 할 때 (Cache Invalidation)
- Pod 수가 동적으로 변하면 Queue 관리 복잡

**결론**: 비즈니스 로직상 한 번만 처리해야 하면 Queue 공유(Work Queue), 모든 인스턴스가 모두 처리해야 하면 인스턴스별 Queue(임시 Queue 권장).

</details>

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[이전: Work Queue ⬅️](./01-work-queue.md)** | **[다음: RPC 패턴 ➡️](./03-rpc.md)**

</div>
