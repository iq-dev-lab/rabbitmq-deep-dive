# 메시지 영속성(Persistence) — 재시작 후에도 살아있는 메시지

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Queue durable=true와 DeliveryMode=PERSISTENT는 왜 둘 다 필요한가?
- 메시지가 실제로 디스크에 fsync되는 정확한 시점은 언제인가?
- 영속성 설정이 처리량에 미치는 영향은 어느 정도인가?
- Lazy Queue는 언제, 왜 사용하는가?
- Quorum Queue의 영속성이 Classic Queue보다 강한 이유는?

---

## 🔍 왜 이 개념이 실무에서 중요한가

"Queue를 durable로 설정했으니 안전하다"는 절반만 맞다. Queue 정의(메타데이터)는 보존되지만, 메시지 내용은 DeliveryMode=PERSISTENT를 함께 설정해야 디스크에 기록된다. 이 둘의 차이를 모르면, 재시작 후 Queue는 있는데 메시지는 없는 상황을 경험한다. 더 나아가 Lazy Queue로 대용량 적재를 처리하는 방법, Quorum Queue가 더 강한 내구성을 보장하는 원리까지 이해해야 신뢰할 수 있는 메시지 시스템을 설계할 수 있다.

---

## 😱 흔한 실수 (Before)

```
실수 1: durable=true만 설정 (DeliveryMode 미설정)

  Queue: durable=true ← Queue 정의 보존
  Message: DeliveryMode=TRANSIENT (기본값) ← 메모리에만

  결과:
    재시작 후 Queue는 존재
    메시지는 전부 소실
    "Queue가 있는데 메시지가 없다"

실수 2: DeliveryMode=PERSISTENT만 설정 (Queue durable=false)

  Queue: durable=false ← 재시작 시 Queue 삭제
  Message: DeliveryMode=PERSISTENT ← 디스크에 쓰려 했지만

  결과:
    재시작 후 Queue 자체가 없어짐
    메시지도 함께 소실 (Queue가 없으므로)
```

---

## ✨ 올바른 접근 (After)

```
올바른 조합 (반드시 두 가지 모두):

Queue:
  durable=true ← 재시작 후 Queue 정의 보존

Message:
  DeliveryMode=PERSISTENT (deliveryMode=2) ← 메시지 디스크 기록

Spring AMQP:
  // Queue
  @Bean
  public Queue orderQueue() {
    return QueueBuilder.durable("order.queue").build();  // durable=true 기본
  }

  // 발행 (MessageProperties)
  MessageProperties props = new MessageProperties();
  props.setDeliveryMode(MessageDeliveryMode.PERSISTENT);
  
  // 또는 Jackson2JsonMessageConverter 사용 시 기본 PERSISTENT
  rabbitTemplate.convertAndSend("order.exchange", "order.placed", event);
  // Jackson2JsonMessageConverter: 기본 DeliveryMode=PERSISTENT
```

---

## 🔬 내부 동작 원리

### 1. durable vs DeliveryMode — 독립적인 두 설정

```
=== Queue durable ===

durable=true:
  Queue 정의를 Mnesia DB에 저장
  재시작 후 Mnesia에서 복원 → Queue 재생성
  
  저장 내용: Queue 이름, 속성(durable, exclusive, autoDelete, arguments)
  저장 위치: /var/lib/rabbitmq/mnesia/ (Erlang Mnesia DB)

durable=false:
  Queue 정의를 메모리에만 보관
  재시작 → Queue 삭제

=== Message DeliveryMode ===

DeliveryMode=1 (TRANSIENT):
  메시지를 메모리에만 저장
  Queue가 durable이어도 재시작 시 소실

DeliveryMode=2 (PERSISTENT):
  메시지를 디스크에 기록
  Queue가 durable이어야 의미 있음 (Queue 없으면 메시지도 없음)

=== 조합 결과 ===

Queue      | DeliveryMode | 재시작 후 결과
───────────┼──────────────┼───────────────────
durable=F  | any          | Queue + 메시지 소실
durable=T  | TRANSIENT    | Queue 존재, 메시지 소실
durable=T  | PERSISTENT   | Queue + 메시지 보존 ✅
```

### 2. PERSISTENT 메시지의 실제 디스크 기록 시점

```
=== 기록 흐름 ===

메시지 수신
    │
    ▼
메모리 버퍼에 쓰기
    │
    ▼
[배치 축적 또는 임계값 도달]
    │
    ▼
디스크 쓰기 (write() 시스템 콜)
    │
    ▼
fsync() 시스템 콜 (OS 버퍼 → 물리 디스크)
    │
    ▼
Publisher Confirm Ack 반환

=== Classic Queue의 파일 구조 ===

메시지 저장 파일:
  /var/lib/rabbitmq/mnesia/<node>/msg_stores/vhosts/<vhost-id>/
    ├── msg_store_persistent/
    │   └── *.rdq  ← PERSISTENT 메시지 데이터 파일

인덱스 파일:
  Queue별 인덱스 (메시지 위치, 전달 상태)
  
배치 쓰기:
  RabbitMQ는 fsync를 매 메시지마다 하지 않음
  일정량 축적 또는 일정 시간 후 배치 fsync
  → Publisher Confirm 모드에서는 fsync 후 Ack

=== Quorum Queue의 WAL ===

Quorum Queue (Raft 기반):
  메시지 → WAL(Write-Ahead Log) 기록
  → 과반수 노드에 WAL 복제
  → 과반수 확인 완료 → Ack

WAL 기록이 완전하므로:
  노드 재시작 → WAL에서 복구
  fsync 타이밍에 무관하게 데이터 안전
  → Classic PERSISTENT보다 강한 내구성
```

### 3. Lazy Queue — 대용량 메시지 적재

```
=== Lazy Queue 동작 원리 ===

일반 Queue:
  메시지 도착 → 메모리에 저장
  메모리 부족 → 디스크로 Paging (느림)
  Consumer 소비 시 메모리에서 즉시 전달

Lazy Queue (x-queue-mode=lazy):
  메시지 도착 → 즉시 디스크에 저장
  메모리에는 인덱스만 유지
  Consumer 소비 시 디스크에서 읽어 전달

=== Lazy Queue 적합한 상황 ===

1. Consumer가 느려서 메시지 대량 적재:
   Producer 1000 msg/sec vs Consumer 100 msg/sec
   → Queue에 계속 쌓임 → 메모리 부족
   → Lazy Queue: 디스크에 저장 → 메모리 절약

2. 피크 트래픽 흡수:
   평상시: 100 msg/sec 처리
   피크: 10,000 msg/sec 유입
   → Lazy Queue: 수백만 메시지 디스크에 적재 가능

3. 메모리 제한 환경:
   브로커 메모리 제한 (vm_memory_high_watermark)
   → Lazy Queue로 메모리 사용 최소화

=== Lazy Queue 설정 ===

Queue 선언 시:
  rabbitmqadmin declare queue name=lazy.queue durable=true \
    arguments='{"x-queue-mode":"lazy"}'

Spring AMQP:
  @Bean
  public Queue lazyQueue() {
    return QueueBuilder.durable("order.queue.lazy")
      .lazy()
      .build();
  }

Policy로 기존 Queue에 적용:
  rabbitmqctl set_policy Lazy ".*" '{"queue-mode":"lazy"}' --apply-to queues
  ← 모든 Queue에 Lazy 적용 (기존 Queue도 포함)

=== Lazy Queue 트레이드오프 ===

장점:
  메모리 사용량 대폭 감소 (수백만 메시지도 적재 가능)
  메모리 과부하로 인한 Flow Control 방지

단점:
  Consumer 소비 시 디스크 I/O 발생 → 지연 증가
  처리량 감소 (메모리 Queue 대비 20~50%)
  Consumer 속도가 빠를 때 Lazy Queue는 오히려 병목
```

### 4. 영속성이 처리량에 미치는 영향

```
=== 처리량 비교 (참고치, 환경에 따라 다름) ===

설정               | 초당 메시지 수 | 비고
──────────────────┼───────────────┼──────────────────────
TRANSIENT + 메모리  | 최고           | 재시작 시 소실
PERSISTENT + HDD   | ~50% 감소      | 디스크 fsync 비용
PERSISTENT + SSD   | ~20% 감소      | SSD는 fsync 빠름
Quorum Queue       | ~30% 감소      | Raft 합의 오버헤드

=== 성능 최적화 전략 ===

전략 1: 메시지 중요도별 분리
  중요 메시지: durable Queue + PERSISTENT (느리지만 안전)
  일반 메시지: 동일 설정 (일관성)
  로그/메트릭: TRANSIENT (빠르고 유실 허용)

전략 2: Batch Publish
  개별 발행: fsync 여러 번
  배치 발행 후 waitForConfirms: fsync 1회 (여러 메시지)
  → 처리량 향상

전략 3: Publisher Confirm 비동기화
  동기 Confirm: 처리량 낮음
  비동기 Confirm: Ack 기다리지 않고 계속 발행 → 높은 처리량
```

---

## 💻 실전 실험

### 실험 1: durable/DeliveryMode 조합 테스트

```bash
# 네 가지 조합 Queue 생성
rabbitmqadmin declare queue name=q1 durable=false  # durable=F, any
rabbitmqadmin declare queue name=q2 durable=true   # durable=T, TRANSIENT
rabbitmqadmin declare queue name=q3 durable=true   # durable=T, PERSISTENT

# 메시지 발행
rabbitmqadmin publish exchange="" routing_key=q1 \
  properties='{"delivery_mode":2}' payload="q1 msg"
rabbitmqadmin publish exchange="" routing_key=q2 \
  properties='{"delivery_mode":1}' payload="q2 msg (transient)"
rabbitmqadmin publish exchange="" routing_key=q3 \
  properties='{"delivery_mode":2}' payload="q3 msg (persistent)"

# RabbitMQ 재시작
docker restart rabbitmq-test && sleep 5

# 재시작 후 확인
rabbitmqctl list_queues name messages
# q2  0  ← TRANSIENT 소실 (Queue는 있음)
# q3  1  ← PERSISTENT 보존
# q1 없음 ← Queue 자체 삭제 (durable=false)
```

### 실험 2: Lazy Queue 메모리 비교

```bash
# 일반 Queue vs Lazy Queue 메모리 사용 비교
rabbitmqadmin declare queue name=normal.q durable=true
rabbitmqadmin declare queue name=lazy.q durable=true \
  arguments='{"x-queue-mode":"lazy"}'

# 1000개 메시지 발행 (각 1KB)
python3 -c "
import subprocess, json
msg = 'x' * 1000
for i in range(1000):
  subprocess.run(['rabbitmqadmin', 'publish', 'exchange=', 'routing_key=normal.q',
    f'payload={msg}'], capture_output=True)
  subprocess.run(['rabbitmqadmin', 'publish', 'exchange=', 'routing_key=lazy.q',
    f'payload={msg}'], capture_output=True)
"

# 메모리 사용량 비교
rabbitmqctl list_queues name messages memory
# normal.q  1000  ~1200000  (약 1.2MB 메모리)
# lazy.q    1000  ~50000    (약 50KB 메모리 — 나머지 디스크)
```

---

## 📊 Kafka vs RabbitMQ 비교

```
=== 메시지 저장 방식 비교 ===

RabbitMQ:
  메모리 우선 → 부족 시 디스크 Paging
  PERSISTENT: 디스크에도 기록 (메모리 + 디스크 동시)
  Lazy Queue: 즉시 디스크 (메모리 최소화)

Kafka:
  디스크 우선 (append-only 로그 파일)
  OS Page Cache로 읽기 속도 최적화
  메모리/디스크 비율 설정 불필요

=== 대용량 메시지 적재 ===

RabbitMQ Lazy Queue:
  수백만 메시지 디스크 적재 가능
  단, Consumer 소비 시 디스크 I/O 발생

Kafka:
  기본적으로 디스크 적재
  메시지 보관 기간까지 유지 (삭제 안 함)
  Consumer 소비 후에도 유지 → 재처리 가능
  → 장기 보관 + 재처리: Kafka가 설계상 유리
```

---

## ⚖️ 트레이드오프

```
영속성 설정 트레이드오프:

TRANSIENT:
  ✅ 최고 처리량, 최저 지연
  ❌ 재시작 시 메시지 소실 (중요 데이터에 절대 금지)

PERSISTENT (Classic Queue):
  ✅ 재시작 후 보존
  ❌ 처리량 ~20% 감소 (SSD 기준)
  ❌ 단일 노드 → 노드 장애 시 소실 가능

PERSISTENT (Quorum Queue):
  ✅ 과반수 노드 WAL → 가장 강한 내구성
  ❌ 처리량 ~30% 감소 (Raft 합의 오버헤드)
  ✅ 노드 장애 내성

Lazy Queue:
  ✅ 메모리 극적 절약 (수백만 메시지 적재)
  ❌ Consumer 처리 속도 감소 (디스크 I/O)
  → Consumer가 느린 경우에만 사용
```

---

## 📌 핵심 정리

```
메시지 영속성 핵심:

필수 조합:
  Queue durable=true + Message DeliveryMode=PERSISTENT
  → 둘 중 하나만으로는 재시작 보장 안 됨

내구성 강도 순서:
  TRANSIENT < PERSISTENT (Classic) < PERSISTENT (Quorum Queue)

Lazy Queue:
  적합: Consumer 느려서 대량 적재, 메모리 제한 환경
  부적합: Consumer 빠르고 지연 민감한 처리
  설정: x-queue-mode=lazy 또는 QueueBuilder.lazy()

처리량 영향:
  PERSISTENT: 약 20% 처리량 감소 (SSD 기준)
  Quorum Queue: 약 30% 처리량 감소
  → 중요 데이터에는 감수할 만한 비용
```

---

## 🤔 생각해볼 문제

**Q1.** durable=true Queue에 TRANSIENT 메시지와 PERSISTENT 메시지가 섞여 있다. 재시작 후 Queue에 남은 메시지의 순서는 보장되는가?

<details>
<summary>해설 보기</summary>

**보장되지 않을 수 있습니다.** 재시작 후 Queue에는 PERSISTENT 메시지만 남습니다. TRANSIENT 메시지가 빠지면서 순서가 달라집니다.

예시:
- 발행 순서: [PERSISTENT msg1][TRANSIENT msg2][PERSISTENT msg3]
- 재시작 후: [msg1][msg3] — msg2가 빠져서 원래 순서(1→2→3)가 아닌 (1→3)

실무 영향:
- 순서 의존적 처리(A 처리 후 B 처리)가 있는 경우 문제
- 혼합 DeliveryMode는 예측 불가능 동작을 만듦

권장: 모든 메시지를 동일한 DeliveryMode로 통일. 중요 큐에는 모두 PERSISTENT 사용.

</details>

---

**Q2.** Lazy Queue를 사용하는데 Consumer가 갑자기 빠르게 처리하기 시작했다. 처리량이 기대만큼 나오지 않는다. 왜인가?

<details>
<summary>해설 보기</summary>

**Lazy Queue는 메시지를 디스크에서 읽어야 하므로 디스크 I/O가 병목이 됩니다.**

일반 Queue: 메모리에서 직접 전달 → 낮은 지연
Lazy Queue: 디스크에서 읽기 → 높은 지연

Consumer가 빠를 때 Lazy Queue의 문제:
- HDD: 수십 ms/읽기 → 심각한 병목
- SSD: 수 ms/읽기 → 여전히 메모리보다 느림

해결:
1. Lazy Queue → 일반 Queue로 전환 (Policy 변경)
   ```bash
   rabbitmqctl clear_policy LazyPolicy
   # 또는
   rabbitmqctl set_policy FastMode "my.queue" '{"queue-mode":"default"}' --apply-to queues
   ```
2. 전환 중에는 기존 메시지가 메모리로 로드됨 (메모리 일시 증가)

Lazy Queue는 "Consumer가 느려서 메시지가 많이 쌓이는 경우"에만 사용하고, Consumer가 따라잡으면 일반 Queue로 전환이 좋습니다.

</details>

---

**Q3.** Publisher Confirm Ack가 "디스크 fsync 완료" 시점에 반환된다면, fsync 전에 재시작되는 경우는 어떻게 처리되는가?

<details>
<summary>해설 보기</summary>

**Classic Queue 단일 노드**: fsync 전 재시작이면 메시지 소실 가능합니다.

Publisher Confirm의 미묘한 동작:
- Classic durable Queue + PERSISTENT: Ack가 "Queue에 기록됨"을 의미하지만, 반드시 fsync 완료를 의미하지는 않음 (RabbitMQ 구현에 따라 다름)
- RabbitMQ는 배치 fsync를 사용 → Ack 이후 실제 fsync까지 짧은 시간이 있음

**Quorum Queue의 해결**:
WAL(Write-Ahead Log) + Raft:
1. 메시지 수신
2. WAL에 기록 + 과반수 노드에 복제
3. 과반수 확인 후 → Ack 반환
4. fsync는 비동기로 별도 수행 (WAL에 있으므로 복구 가능)

→ Quorum Queue에서 Ack는 "과반수 노드의 WAL에 기록됨"을 의미
→ 재시작해도 WAL에서 복구 가능
→ 가장 강한 내구성 보장

결론: 완전한 내구성이 필요하면 Quorum Queue 사용. Classic Queue의 fsync 타이밍 문제는 실용적으로 무시할 수 있는 수준이지만, 이론적으로는 위험이 있습니다.

</details>

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[이전: Publisher Confirm ⬅️](./02-publisher-confirm.md)** | **[다음: Consumer Acknowledgement 완전 분해 ➡️](./04-consumer-ack.md)**

</div>
