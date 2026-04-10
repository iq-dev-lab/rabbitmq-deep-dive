# Priority Queue — 높은 우선순위 메시지 먼저 처리

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `x-max-priority`를 설정하면 내부적으로 어떻게 우선순위별로 메시지를 관리하는가?
- 높은 우선순위 메시지가 낮은 우선순위보다 반드시 먼저 처리되는가?
- 낮은 우선순위 메시지의 Starvation(굶주림)은 어떻게 방지하는가?
- 우선순위 레벨을 몇 개로 설정하는 것이 적절한가?
- Quorum Queue에서 Priority Queue를 사용할 수 있는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

모든 메시지가 같은 중요도를 가지지는 않는다. VIP 고객의 주문은 일반 고객보다 빠르게 처리되어야 한다. 결제 오류 알림은 마케팅 이메일보다 먼저 발송되어야 한다. 단순히 여러 Queue를 나누는 방법도 있지만, 같은 처리 로직을 가진 작업을 우선순위만 다르게 처리하고 싶을 때 Priority Queue가 유용하다. 하지만 우선순위 레벨을 남용하면 메모리와 CPU 오버헤드가 커지고, 낮은 우선순위 메시지가 영원히 처리되지 않는 Starvation 문제가 생긴다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: x-max-priority를 크게 설정 (예: 255)

  rabbitmqadmin declare queue name=order.queue durable=true \
    arguments='{"x-max-priority":255}'

  결과:
    255개 우선순위 레벨을 관리하기 위한 내부 구조 생성
    메모리 사용량 급증 (레벨당 별도 서브큐 관리)
    CPU 오버헤드 증가
    
  실제 필요한 레벨: 대부분 2~5개
  권장: x-max-priority는 5 이하

실수 2: 발행 시 priority 설정 안 함

  // priority 미설정 → 기본값 0 (최하 우선순위)
  rabbitTemplate.convertAndSend("order.queue", message);

  결과:
    Queue에 priority=10인 메시지가 있어도
    priority=0으로 발행된 메시지와 동등하게 처리될 수도 있음
    (명시적 priority 설정 필수)

실수 3: Consumer가 느릴 때 Starvation 무시

  priority=10: VIP 주문 초당 100개
  priority=1:  일반 주문 초당 50개

  Consumer: 초당 80개 처리
  
  결과:
    priority=10 메시지가 계속 들어옴
    priority=1 메시지는 priority=10 처리 후에 처리
    Consumer가 priority=10에 바쁘면 priority=1 메시지 영원히 대기
    → 일반 고객 주문이 시간이 지나도 처리 안 됨
```

---

## ✨ 올바른 접근 (After — 적절한 우선순위 레벨 + Starvation 모니터링)

```
Priority Queue 올바른 설계:

1. 최소한의 우선순위 레벨 사용:
   2단계: 일반(0) / 긴급(1) → 대부분 충분
   3단계: 일반(0) / 중요(5) / 긴급(10)
   권장: x-max-priority=10 이하

2. 발행 시 명시적 priority 설정:
  MessageProperties props = new MessageProperties();
  props.setPriority(order.isVip() ? 10 : 1);
  rabbitTemplate.send(exchange, key,
    new Message(payload, props));

3. Starvation 방지:
   낮은 우선순위 Queue 메시지 수 모니터링
   임계값 초과 시 알림
   임시로 높은 우선순위 Consumer 추가

4. 우선순위 대신 Queue 분리 고려:
   vip.order.queue (Consumer 3개)
   normal.order.queue (Consumer 2개)
   → 더 단순하고 예측 가능한 처리량 분리
```

---

## 🔬 내부 동작 원리

### 1. Priority Queue 내부 구조

```
=== x-max-priority 설정의 내부 ===

x-max-priority=10 설정 시:
  RabbitMQ는 내부적으로 우선순위별 서브큐 구조 유지
  
  논리적 구조:
  priority.queue (x-max-priority=10)
  ├── sub-queue[10] : [msg-A][msg-B]  ← 최고 우선순위
  ├── sub-queue[9]  : []
  ├── sub-queue[8]  : [msg-C]
  ├── ...
  ├── sub-queue[1]  : [msg-D][msg-E][msg-F]
  └── sub-queue[0]  : [msg-G]...      ← 최저 우선순위

Consumer 요청 시 처리 순서:
  sub-queue[10] → 비어있을 때까지 소비
  sub-queue[9]  → 비어있을 때까지 소비
  ...
  sub-queue[0]  → 최후에 소비

=== 우선순위 처리 보장 조건 ===

우선순위가 보장되는 조건:
  Consumer가 처리 가능한 속도 > 발행 속도
  즉, Queue에 메시지가 "쌓여 있을" 때

Consumer가 빠를 때 (Prefetch=1, 즉시 처리):
  메시지 발행 → 즉시 Consumer에게 전달 → 즉시 처리
  → Queue에 메시지가 거의 없음
  → 우선순위를 비교할 대상이 없음
  → 실질적으로 FIFO처럼 동작할 수 있음

우선순위가 의미 있는 조건:
  Queue에 여러 우선순위의 메시지가 함께 쌓여 있을 때
  Consumer 처리 속도 < 발행 속도
  
  실무: Consumer를 의도적으로 느리게 하거나,
        메시지 폭발 상황에서 Priority Queue 효과 극대화
```

### 2. Starvation 분석과 방지

```
=== Starvation 발생 시나리오 ===

Queue: x-max-priority=10
발행 패턴:
  priority=10: 초당 100개 (VIP 주문)
  priority=1:  초당 50개 (일반 주문)

Consumer 처리량: 초당 120개

처리 흐름:
  초당 priority=10: 100개 유입, 100개 처리 (소화)
  초당 priority=1:  50개 유입, 20개 처리 (나머지 120-100=20개 슬롯)
  
  priority=1 누적: 50 - 20 = 초당 30개씩 쌓임
  10초 후: 300개 쌓임, 계속 증가

Consumer를 120 → 150개/sec로 증가 시:
  priority=10: 100개/sec 처리 (항상 우선)
  priority=1:  50개/sec 처리 가능 (슬롯 50개 남음)
  → Starvation 해소

=== Starvation 방지 전략 ===

전략 1: Consumer 처리량 확보
  발행 총량 < Consumer 처리량이면 Starvation 없음
  Consumer Scale Out

전략 2: 우선순위 범위 축소
  10레벨 → 2레벨로 단순화
  priority=1 (일반) / priority=2 (긴급)
  모든 긴급 처리 후 일반 처리 → 축적 방지 쉬움

전략 3: 별도 Queue로 분리 (더 예측 가능)
  vip.order.queue → Consumer 3개 (70% 처리량)
  normal.order.queue → Consumer 2개 (30% 처리량)
  → 각 Queue의 처리량 명시적으로 보장

전략 4: TTL로 오래된 메시지 강제 처리
  낮은 우선순위 메시지에 TTL 설정
  x-message-ttl: 300000 (5분)
  → 5분 후 DLX로 이동 → 별도 처리 (또는 더 빠른 처리 큐로 이동)

=== 모니터링 ===

  알림 조건:
  priority=0 메시지가 N분 이상 Queue에 있으면 알림

  Prometheus 쿼리 (KEDA 또는 custom exporter):
    rabbitmq_queue_messages{queue="order.queue"} > 10000
    → 총 메시지 수 증가는 Starvation 징조
```

### 3. 우선순위 레벨 설계 가이드

```
=== 레벨 수 권장 ===

2레벨 (권장):
  0 = 일반
  1 = 긴급
  → 단순하고 관리 용이

3레벨:
  0 = 낮음
  5 = 중간
  10 = 높음
  → 대부분 서비스에 충분

5레벨 이상:
  실무에서 필요한 경우 드뭄
  레벨 간 경계 모호 → 개발자 혼란

=== 우선순위 vs 별도 Queue ===

Priority Queue 선택:
  처리 로직이 동일한 작업을 우선순위만 다르게 처리
  동적으로 우선순위가 변경될 수 있는 경우
  Consumer 코드가 단일

별도 Queue 선택:
  처리 로직이 다른 경우 (VIP 전용 처리 로직)
  처리량 비율을 명시적으로 보장해야 할 때
  우선순위가 고정적인 경우

=== 발행 시 priority 설정 ===

Spring AMQP:
  // 방법 1: MessagePostProcessor
  rabbitTemplate.convertAndSend(exchange, key, payload,
    message -> {
      message.getMessageProperties().setPriority(priorityLevel);
      return message;
    }
  );

  // 방법 2: MessageProperties 직접
  MessageProperties props = new MessageProperties();
  props.setPriority(isVip ? 10 : 1);
  Message msg = new Message(objectMapper.writeValueAsBytes(payload), props);
  rabbitTemplate.send(exchange, key, msg);
```

---

## 💻 실전 실험

### 실험 1: Priority Queue 우선순위 처리 확인

```bash
# Priority Queue 생성 (max-priority=10)
rabbitmqadmin declare queue name=priority.test durable=true \
  arguments='{"x-max-priority":10}'

# 낮은 우선순위 메시지 먼저 발행
for i in $(seq 1 5); do
  rabbitmqadmin publish exchange="" routing_key=priority.test \
    properties='{"priority":1}' payload="{\"priority\":1,\"id\":$i}"
done

# 높은 우선순위 메시지 나중에 발행
for i in $(seq 1 5); do
  rabbitmqadmin publish exchange="" routing_key=priority.test \
    properties='{"priority":10}' payload="{\"priority\":10,\"id\":$i}"
done

# Queue에서 메시지 꺼내기 → 우선순위 10 먼저 나와야 함
rabbitmqadmin get queue=priority.test ackmode=ack_requeue_false count=10

# 출력 결과: priority=10 메시지가 먼저 출력됨
# {"priority":10,"id":1}
# {"priority":10,"id":2}
# ...
# {"priority":1,"id":1}  ← 낮은 우선순위는 나중에
```

### 실험 2: Starvation 시뮬레이션

```bash
# 높은 우선순위만 계속 발행하면서 낮은 우선순위 확인
# 고우선순위 무한 발행 (백그라운드)
while true; do
  rabbitmqadmin publish exchange="" routing_key=priority.test \
    properties='{"priority":10}' payload='{"priority":10}' 2>/dev/null
  sleep 0.01
done &

# 저우선순위 발행
for i in $(seq 1 100); do
  rabbitmqadmin publish exchange="" routing_key=priority.test \
    properties='{"priority":1}' payload="{\"low\":$i}"
done

# 잠시 후 Queue 상태 확인
sleep 5
rabbitmqctl list_queues name messages
# priority.test 수천개 쌓임
# 저우선순위 메시지가 묻혀 처리 안 될 수 있음

kill %1  # 백그라운드 프로세스 종료
```

---

## 📊 Kafka vs RabbitMQ 비교

```
=== 메시지 우선순위 비교 ===

RabbitMQ Priority Queue:
  x-max-priority 설정으로 브로커 레벨 우선순위 지원
  높은 우선순위 메시지가 Consumer에게 먼저 전달
  Consumer 처리량 < 발행량일 때 효과적

Kafka:
  기본적으로 우선순위 미지원
  Partition 내 FIFO 순서만 보장
  
  우선순위 구현 방법:
  1. 우선순위별 별도 Topic 생성
     high-priority-orders Topic + Consumer 더 많이
     normal-orders Topic + Consumer 더 적게
  2. Consumer에서 필터링 (브로커 레벨 지원 없음)

=== 선택 기준 ===

우선순위 처리가 중요한 서비스:
  RabbitMQ Priority Queue가 기본 지원
  Kafka는 별도 Topic 분리로 복잡한 설정 필요

단순한 FIFO + 높은 처리량:
  Kafka가 더 적합
  우선순위 필요 없으면 Kafka FIFO가 더 단순
```

---

## ⚖️ 트레이드오프

```
Priority Queue 장단점:

장점:
  ① 단일 Queue에서 우선순위 처리
  ② 처리 로직 통일 (Consumer 코드 동일)
  ③ 동적 우선순위 변경 가능 (발행 시 결정)

단점:
  ① 메모리 오버헤드 (레벨수만큼 서브큐)
  ② Starvation 위험
  ③ Consumer 빠를 때 우선순위 의미 없음
  ④ Quorum Queue에서 미지원

별도 Queue 분리 vs Priority Queue:
  별도 Queue: 처리량 보장 명시적, 구현 단순
  Priority Queue: 단일 Consumer, 동적 우선순위
  → 처리량 비율 보장 필요: 별도 Queue
  → 동적 우선순위 필요: Priority Queue
```

---

## 📌 핵심 정리

```
Priority Queue 핵심:

설정:
  x-max-priority: N (N = 우선순위 레벨 최대값)
  권장: 2~5 이하

발행:
  MessageProperties.setPriority(n)
  반드시 명시적 설정 필요 (기본값 0)

처리 보장:
  Queue에 여러 우선순위 메시지가 쌓여 있을 때
  높은 우선순위부터 소비
  Consumer가 빠르면 우선순위 효과 약함

Starvation:
  높은 우선순위 발행량 > Consumer 처리량
  낮은 우선순위 메시지가 영원히 대기
  방지: Consumer 증가, 별도 Queue 분리, TTL

Quorum Queue:
  x-max-priority 미지원
  Classic Queue에서만 사용 가능
```

---

## 🤔 생각해볼 문제

**Q1.** `x-max-priority=255`로 설정된 Queue와 `x-max-priority=5`로 설정된 Queue의 메모리 사용량 차이는 왜 발생하는가?

<details>
<summary>해설 보기</summary>

RabbitMQ는 `x-max-priority=N` 설정 시 내부적으로 우선순위별 서브큐 구조를 관리합니다. 각 우선순위 레벨에 대한 데이터 구조(Erlang 리스트 또는 트리)가 할당됩니다.

`x-max-priority=255`: 256개 레벨의 서브큐 구조
`x-max-priority=5`: 6개 레벨의 서브큐 구조

실제 사용되는 레벨이 적어도 구조 자체의 오버헤드가 발생합니다. RabbitMQ 공식 문서에서도 "우선순위 레벨을 크게 설정하면 성능에 영향을 준다"고 경고합니다.

실측 결과 (참고치):
- 우선순위 없음: 기본 메모리
- x-max-priority=10: 약 5~15% 메모리 증가
- x-max-priority=255: 수십% 메모리 증가 가능

결론: 실제 필요한 우선순위 레벨 수만 설정. 대부분 2~3개로 충분합니다.

</details>

---

**Q2.** priority=10인 메시지가 발행됐는데, Consumer는 priority=1인 메시지를 먼저 받았다. 어떻게 이런 일이 발생하는가?

<details>
<summary>해설 보기</summary>

**Consumer가 이미 메시지를 Prefetch로 가져갔을 때** 발생합니다.

시나리오:
1. Queue: [priority=1 메시지 5개] (초기)
2. Consumer: Prefetch=5로 5개 모두 가져감 (Unacked)
3. priority=10 메시지 발행
4. Consumer: 이미 가져간 priority=1 메시지 처리 중
5. priority=10은 Consumer의 Prefetch 슬롯이 비어야 전달됨

이것이 **Prefetch와 Priority Queue의 상호작용**입니다.

Prefetch=1로 설정하면 완화:
- Consumer가 1개씩 가져가므로, 다음 메시지 수신 시 priority=10이 먼저 전달

하지만 Prefetch=1이어도 Consumer가 이미 1개를 처리 중이면 그 처리가 끝날 때까지 우선순위 역전 상태 유지.

완전한 우선순위 보장: Prefetch=1 + Consumer 처리 속도가 발행 속도보다 충분히 빨 때.

</details>

---

**Q3.** Priority Queue 대신 별도 Queue로 분리하는 것이 더 나은 경우는 구체적으로 어떤 상황인가?

<details>
<summary>해설 보기</summary>

**별도 Queue가 더 나은 5가지 상황:**

1. **처리량 비율 보장이 필요할 때**
   - VIP: 70% 처리량, 일반: 30% 처리량 보장
   - Priority Queue는 VIP가 있으면 100% VIP 처리 → 일반 Starvation
   - 별도 Queue: Consumer 수로 처리량 비율 명시적 설정

2. **처리 로직이 다를 때**
   - VIP 주문: SMS 즉시 발송 + 전담 매니저 알림
   - 일반 주문: 이메일 발송만
   - Priority Queue: Consumer 내 if-else 복잡도 증가

3. **Quorum Queue를 사용해야 할 때**
   - Priority Queue는 Classic Queue만 지원
   - 고가용성 필요 시 Quorum Queue → 별도 Queue 분리

4. **우선순위가 고정적일 때**
   - VIP 여부가 런타임에 변하지 않는 경우
   - 발행 시 대상 Queue만 바꾸면 됨

5. **모니터링/운영이 중요할 때**
   - vip.queue / normal.queue로 각각 Queue Depth 독립 모니터링
   - "VIP 주문만 몇 건 쌓여있나?"를 직접 확인 가능

</details>

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[이전: RPC 패턴 ⬅️](./03-rpc.md)** | **[다음: Delayed/Scheduled Message ➡️](./05-delayed-message.md)**

</div>
