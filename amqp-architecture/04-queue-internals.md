# Queue 내부 구조 — 메시지가 쌓이고 소비되는 원리

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Queue는 메시지를 어떻게 저장하고, 메모리와 디스크는 어떻게 사용되는가?
- Durable Queue와 Non-Durable Queue는 재시작 시 어떻게 다르게 동작하는가?
- Exclusive Queue와 AutoDelete Queue는 언제 사용하는가?
- Quorum Queue는 Classic Queue와 어떻게 다르고, 왜 더 안전한가?
- Queue의 메시지가 메모리를 초과하면 어떤 일이 발생하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

Queue 속성을 잘못 설정하면 브로커 재시작 후 모든 메시지가 사라진다. Durable=false Queue에 중요한 주문 메시지를 쌓아두고 배포했다가, 재시작 후 메시지가 증발한 상황은 실제로 자주 발생한다. Quorum Queue를 모르면 단일 노드 장애로 메시지 유실을 경험한다. Queue의 메모리 구조를 모르면 Lazy Queue와 일반 Queue 중 언제 무엇을 선택할지 판단할 수 없다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: Queue는 선언했지만 durable=false

  // 개발 환경에서 테스트하다 그대로 프로덕션에
  @Bean
  public Queue orderQueue() {
    return new Queue("order.queue");  // durable 기본값: true이지만...
  }
  
  // 또는 CLI에서 직접 생성 시 실수
  rabbitmqadmin declare queue name=order.queue durable=false
  
  결과:
    RabbitMQ 재시작 (배포, OOM 재시작, 서버 점검)
    → durable=false Queue 삭제
    → Queue에 쌓여 있던 미처리 메시지 전부 소실
    → 재시작 후 Queue도 없어짐 (Consumer가 없으면 자동 재선언도 안 됨)
    → "Queue not found" 에러

실수 2: 메시지만 durable로 설정 (Queue는 non-durable)

  // 메시지 영속성(DeliveryMode=PERSISTENT)만 설정
  MessageProperties props = new MessageProperties();
  props.setDeliveryMode(MessageDeliveryMode.PERSISTENT);
  
  결과:
    Queue가 non-durable이면 메시지 영속성 의미 없음
    재시작 시 Queue 자체가 삭제됨 → 메시지도 같이 사라짐
    
  올바른 조합:
    Queue durable=true + Message DeliveryMode=PERSISTENT
    → 두 가지 모두 설정해야 재시작 후 메시지 보존

실수 3: 클러스터에서 Classic Queue의 단일 노드 장애

  // 클러스터 3노드, Classic Queue
  rabbitAdmin.declareQueue(new Queue("order.queue", true));
  // durable=true이지만 Classic Queue는 단일 노드에만 저장
  
  결과:
    order.queue가 Node 1에 저장됨
    Node 1 장애 → order.queue 접근 불가 (Node 2, 3에 없음)
    → Consumer 모두 오류, 새 메시지 발행 실패
    → Node 1 복구 후에야 접근 가능
    
  해결: Quorum Queue 사용 (과반수 노드에 복제)
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설정)

```
프로덕션 Queue 설정 기준:

1. 단일 노드 / 개발 환경:
   durable=true + Message DeliveryMode=PERSISTENT
   → 재시작 후 메시지 보존

2. 클러스터 환경 (고가용성):
   Quorum Queue 사용
   @Bean
   public Queue orderQueue() {
     return QueueBuilder
       .durable("order.queue")
       .quorum()             // Quorum Queue 선언
       .build();
   }
   → 과반수 노드에 복제, 노드 장애 시 자동 복구

3. 임시 Queue (Pub/Sub Consumer용):
   Exclusive=true + AutoDelete=true
   → Consumer 연결 시 생성, 연결 끊기면 삭제
   → Fanout Exchange의 각 Consumer가 자신만의 Queue 가짐

4. 대용량 메시지 적재:
   Lazy Queue 설정
   @Bean
   public Queue lazyOrderQueue() {
     return QueueBuilder
       .durable("order.queue.lazy")
       .lazy()           // 메시지를 디스크에 저장
       .build();
   }
   → 메모리 절약, 대량 메시지 적재 가능
```

---

## 🔬 내부 동작 원리

### 1. Queue의 메시지 저장 구조

```
=== Classic Queue의 메시지 저장 계층 ===

메시지 상태에 따른 저장 위치:

  [새 메시지 도착]
        │
        ▼
  ┌─────────────────────────────────────────────────┐
  │            Rabbit Queue Process                 │
  │                                                 │
  │  Alpha (q1): 메모리 + 메타데이터                  │
  │  ┌─────┬─────┬─────┬─────┐                     │
  │  │ msg │ msg │ msg │ msg │ ← 최근 소비 예정 메시지│
  │  └─────┴─────┴─────┴─────┘                     │
  │                                                 │
  │  Beta (q2): 메시지 내용 디스크, 메타 메모리         │
  │  ┌─────┬─────┬─────┐                           │
  │  │[ptr]│[ptr]│[ptr]│ ← 디스크 포인터만 메모리     │
  │  └─────┴─────┴─────┘                           │
  │                                                 │
  │  Gamma (q3): 메시지 내용 + 메타데이터 모두 디스크   │
  │  Delta (q4): 완전히 디스크 (메모리 압박 시)        │
  │                                                 │
  └─────────────────────────────────────────────────┘

메모리 압박 시 (vm_memory_high_watermark 접근):
  Alpha → Beta → Gamma → Delta 순으로 메시지를 디스크로 이동 (Paging)
  Paging 중에는 처리 속도 저하 발생
  
  vm_memory_high_watermark 초과 시:
  → Publisher에게 Flow Control 발동 (새 메시지 수신 차단)
  → Queue가 메모리 확보할 때까지 대기

=== Lazy Queue ===

설정: x-queue-mode=lazy (또는 QueueBuilder.lazy())

일반 Queue: 메모리 우선 저장, 메모리 부족 시 디스크로 이동
Lazy Queue:  메시지 도착 즉시 디스크 저장 (메모리에는 인덱스만)

Lazy Queue 적합한 경우:
  ✅ Consumer가 느려서 메시지가 대량 적재되는 경우
  ✅ 피크 트래픽 버퍼가 필요한 경우 (수백만 메시지 적재)
  ✅ 메모리 절약이 중요한 경우
  
  단점:
  ❌ 디스크 I/O로 인한 처리량 감소
  ❌ Consumer 소비 시 디스크에서 읽어야 하므로 지연 증가
```

### 2. Queue 속성 완전 분해

```
=== Durable ===

durable=true:
  RabbitMQ 재시작 후에도 Queue 정의 유지
  단, 메시지 보존은 DeliveryMode=PERSISTENT도 함께 필요
  
  내부: Mnesia DB에 Queue 정의 저장
  재시작 시 Mnesia에서 Queue 정의 복원 → Queue 재생성

durable=false (transient):
  메모리에만 Queue 정의 저장
  재시작 시 Queue 삭제 (메시지도 소실)
  
  사용 사례: 테스트 환경, 재시작 후 재생성해도 되는 임시 Queue

=== Exclusive ===

exclusive=true:
  선언한 Connection에서만 접근 가능
  Connection 종료 시 Queue 자동 삭제 (durable 무관)
  
  사용 사례:
  Pub/Sub 패턴에서 Consumer마다 자신만의 Queue를 가질 때
  
  @Bean
  public Queue exclusiveQueue() {
    return QueueBuilder
      .nonDurable()    // exclusive는 보통 non-durable
      .exclusive()
      .autoDelete()
      .build();
  }

=== AutoDelete ===

autoDelete=true:
  마지막 Consumer가 연결 해제 시 Queue 자동 삭제
  Consumer가 한 번도 없었던 Queue는 삭제 안 됨
  
  exclusive + autoDelete 조합:
    Consumer 연결 시 Queue 생성 (또는 재사용)
    Consumer 연결 해제 시 Queue 삭제
    → 연결마다 새 Queue (임시 구독자 패턴)

=== TTL (Time-To-Live) ===

메시지 TTL (x-message-ttl):
  Queue 내 메시지의 최대 보관 시간 (밀리초)
  만료된 메시지 → Dead Letter Exchange로 전달 (DLX 설정 시)
  
  @Bean
  public Queue ttlQueue() {
    return QueueBuilder
      .durable("order.queue")
      .ttl(60000)  // 60초 후 만료
      .deadLetterExchange("dlx.exchange")
      .build();
  }

Queue TTL (x-expires):
  Queue 자체의 TTL (Consumer가 없을 때)
  지정 시간 동안 사용되지 않으면 Queue 삭제

=== 최대 길이 (Max Length) ===

x-max-length: Queue의 최대 메시지 수
x-max-length-bytes: Queue의 최대 바이트 크기

초과 시 기본 동작 (x-overflow):
  drop-head: 가장 오래된 메시지 삭제 (기본)
  reject-publish: Publisher에게 메시지 거절 (Confirm 모드에서 Nack)
  reject-publish-dlx: 거절된 메시지를 DLX로 전달
```

### 3. Quorum Queue vs Classic Queue

```
=== Classic Queue의 한계 ===

단일 노드 저장:
  Queue는 마스터 노드 한 곳에만 저장
  마스터 노드 장애 → Queue 접근 불가

미러링(Mirrored Queue, 구식 방법):
  모든 노드에 동기 복제 (x-ha-policy=all)
  성능 저하 크고, RabbitMQ 3.12에서 제거됨

=== Quorum Queue ===

기반 기술: Raft 합의 알고리즘 (etcd, CockroachDB 등과 같은 원리)
3.8.0+ 에서 사용 가능, 3.12+에서 권장 기본값

=== Raft 기반 복제 동작 ===

클러스터 3노드:
  Node 1 (Leader) ← 모든 쓰기
  Node 2 (Follower)
  Node 3 (Follower)

메시지 발행:
  1. Producer → Node 1 (Leader) 메시지 발행
  2. Node 1 → Node 2, Node 3 복제 로그 전달
  3. 과반수(2/3) 확인 → Leader가 Ack 반환
  4. Producer: "메시지 안전하게 저장됨" 확인

Node 2 장애 시:
  과반수(Node 1, Node 3): 여전히 2/3 → 정상 운영
  메시지 손실 없음

Node 1 (Leader) 장애 시:
  Node 2, Node 3 중 새 Leader 선출 (Raft 투표)
  새 Leader가 쓰기 처리 재개
  복구 시간: 수 초 이내

=== Quorum Queue 설정 ===

Spring AMQP:
  @Bean
  public Queue quorumQueue() {
    return QueueBuilder
      .durable("order.queue")
      .quorum()
      .quorumInitialGroupSize(3)  // 초기 복제 노드 수
      .build();
  }

rabbitmqctl:
  rabbitmqctl declare_queue \
    --queue order.queue \
    --durable true \
    --queue-type quorum

=== Classic vs Quorum 비교 ===

특성               | Classic Queue      | Quorum Queue
──────────────────┼────────────────────┼─────────────────────
노드 장애 허용      | 미러링 필요          | 자동 (Raft 기반)
메시지 중복 가능성  | 낮음                | 낮음 (Raft 로그)
처리량             | 높음                | 약간 낮음 (합의 오버헤드)
지연               | 낮음                | 약간 높음 (과반수 확인)
지원 버전          | 모든 버전            | 3.8.0 이상
RabbitMQ 3.12     | Mirrored 제거       | 권장 기본값
우선순위 Queue     | 지원                | 미지원
```

### 4. Queue의 메시지 소비 방식

```
=== Push vs Pull ===

Push 방식 (basicConsume, @RabbitListener):
  Consumer가 구독 등록 → Broker가 메시지를 Consumer에 Push
  Prefetch Count로 한 번에 받을 메시지 수 조절
  
  적합: 지속적인 메시지 소비, 낮은 지연 필요

Pull 방식 (basicGet):
  Consumer가 직접 메시지 1개씩 요청
  메시지 없으면 null 반환 (blocking 없음)
  
  String message = channel.basicGet("queue-name", false); // auto-ack=false
  
  적합: 수동 폴링, 특정 시점에만 메시지 소비

=== 메시지 소비 과정 ===

Unacknowledged 메시지 흐름:

  Queue: [msg1][msg2][msg3][msg4][msg5]
         Ready ──────────────────────────
  
  Consumer A (Prefetch=2):
    basicConsume 등록
    → msg1, msg2 수신 (Unacked 상태)
    
  Queue 상태:
    Ready: msg3, msg4, msg5 (3개)
    Unacked: msg1, msg2 (2개)
    
  Consumer A가 msg1 처리 완료 → basicAck
    → Queue에서 msg1 영구 삭제
    → Prefetch 1 슬롯 확보 → msg3 전달
  
  Consumer A 연결 끊김 (msg2 Ack 안 함):
    → Unacked였던 msg2 → Ready 상태로 복귀
    → 다른 Consumer에게 재전달
```

---

## 💻 실전 실험

### 실험 1: durable vs non-durable Queue 재시작 비교

```bash
# durable=false Queue 생성 및 메시지 발행
rabbitmqadmin declare queue name=transient.queue durable=false
rabbitmqadmin publish exchange="" routing_key=transient.queue payload="중요 메시지"

# durable=true Queue 생성 및 메시지 발행
rabbitmqadmin declare queue name=durable.queue durable=true
rabbitmqadmin publish exchange="" routing_key=durable.queue payload="중요 메시지"

# 상태 확인
rabbitmqctl list_queues name messages durable
# transient.queue  1  false
# durable.queue    1  true

# RabbitMQ 재시작
docker restart rabbitmq-test

# 재시작 후 Queue 상태
rabbitmqctl list_queues name messages durable
# durable.queue  1  true   ← 메시지 보존
# transient.queue 없음     ← Queue 자체 삭제
```

### 실험 2: Quorum Queue 생성 및 확인

```bash
# Quorum Queue 선언 (클러스터 환경 필요)
rabbitmqadmin declare queue name=order.quorum durable=true \
  arguments='{"x-queue-type": "quorum"}'

# Queue 정보 확인
rabbitmqctl list_queues name type durable
# order.quorum  quorum  true

# Quorum Queue 복제 상태 확인
rabbitmqctl quorum_status order.quorum
# Status of quorum queue 'order.quorum' in vhost '/':
# ┌─────────────────────┬───────────────────────┬────────┬──────┐
# │ Node Name           │ Raft State            │ Log    │ Term │
# ├─────────────────────┼───────────────────────┼────────┼──────┤
# │ rabbit@node1        │ leader                │ 10/10  │ 1    │
# │ rabbit@node2        │ follower              │ 10/10  │ 1    │
# │ rabbit@node3        │ follower              │ 10/10  │ 1    │
# └─────────────────────┴───────────────────────┴────────┴──────┘
```

### 실험 3: Lazy Queue 메모리 절약 확인

```bash
# 일반 Queue와 Lazy Queue 메모리 비교
rabbitmqadmin declare queue name=normal.queue durable=true
rabbitmqadmin declare queue name=lazy.queue durable=true \
  arguments='{"x-queue-mode": "lazy"}'

# 10000개 메시지 발행
for i in $(seq 1 10000); do
  rabbitmqadmin publish exchange="" routing_key=normal.queue \
    payload="$(python3 -c 'print("x"*1000)')" &
done
wait

for i in $(seq 1 10000); do
  rabbitmqadmin publish exchange="" routing_key=lazy.queue \
    payload="$(python3 -c 'print("x"*1000)')" &
done
wait

# 메모리 사용량 비교
rabbitmqctl list_queues name messages memory
# normal.queue  10000  45000000 (약 45MB 메모리)
# lazy.queue    10000  2000000  (약 2MB 메모리 — 나머지는 디스크)
```

---

## 📊 Kafka vs RabbitMQ 비교

```
=== 메시지 저장 방식 비교 ===

RabbitMQ Queue:
  메모리 우선 저장 → 메모리 압박 시 디스크로 Paging
  Consumer Ack 후 즉시 삭제
  메시지를 "임시로" 보관 (처리 완료까지만)
  
  메모리 관리:
  vm_memory_high_watermark 초과 → Flow Control 발동
  → Publisher 일시적 차단
  → Queue가 메모리 확보 (디스크로 Paging)

Kafka Topic/Partition:
  디스크 우선 저장 (append-only 로그 파일)
  Consumer가 읽어도 삭제 안 함 (보관 기간까지)
  메모리는 Page Cache로 활용 (OS 관리)
  
  메모리 관리:
  디스크 기반이라 메모리 임계값 개념 없음
  OS Page Cache가 자연스럽게 관리

=== 장기 메시지 보관 비교 ===

"100만 개 메시지를 24시간 보관해야 한다"

RabbitMQ:
  Lazy Queue 사용 → 디스크에 저장
  메모리: 최소 (인덱스만)
  단, 처리 완료된 메시지는 삭제됨
  Consumer가 처리 안 하면 Queue에 쌓임
  → 24시간 후 재처리 불가 (이미 소비됨)

Kafka:
  retention.ms=86400000 (24시간) 설정
  디스크에 저장, Consumer가 읽어도 삭제 안 됨
  24시간 후 자동 삭제
  Consumer Group이 오프셋 리셋하면 재처리 가능
```

---

## ⚖️ 트레이드오프

```
Queue 설정 트레이드오프:

durable=true:
  ✅ 재시작 후 Queue/메시지 보존
  ❌ 디스크 write로 인한 성능 저하 (DeliveryMode=PERSISTENT 함께 필요)

Quorum Queue:
  ✅ 노드 장애 허용, 메시지 유실 방지
  ❌ 합의 오버헤드로 인한 약간의 지연 증가
  ❌ 우선순위 큐 미지원

Lazy Queue:
  ✅ 메모리 절약, 대량 메시지 적재 가능
  ❌ 디스크 I/O로 인한 처리량 감소, 지연 증가

메모리 Queue (durable=false):
  ✅ 최고 처리량, 최저 지연
  ❌ 재시작 시 메시지 소실 (테스트/로그 같은 중요도 낮은 데이터만)
```

---

## 📌 핵심 정리

```
Queue 핵심:

저장 구조:
  메모리 우선 → 메모리 압박 시 디스크 Paging
  Lazy Queue: 즉시 디스크 저장 → 메모리 절약

주요 속성:
  durable=true: 재시작 후 Queue 유지 (필수 조합: DeliveryMode=PERSISTENT)
  exclusive: 선언 Connection에서만 사용, 연결 끊기면 삭제
  autoDelete: 마지막 Consumer 해제 시 삭제

Classic vs Quorum:
  Classic: 단일 노드, 노드 장애 시 접근 불가 (클러스터 미러링 필요)
  Quorum: Raft 과반수 복제, 노드 장애 시 자동 복구 (3.8+ 권장)

메시지 소비:
  Push: Consumer 구독 → Broker가 Push (Prefetch로 속도 조절)
  Pull: basicGet으로 수동 요청 (1개씩)
  Unacked: Consumer가 받았지만 Ack 안 한 메시지 (연결 끊기면 Ready로 복귀)
```

---

## 🤔 생각해볼 문제

**Q1.** `durable=true`인 Queue에 `DeliveryMode=TRANSIENT`(비영속) 메시지를 발행했다. RabbitMQ를 재시작하면 어떤 일이 발생하는가?

<details>
<summary>해설 보기</summary>

**Queue 정의는 살아있지만, TRANSIENT 메시지는 소실됩니다.**

- `durable=true`는 Queue의 **정의(메타데이터)**를 재시작 후에도 유지합니다
- `DeliveryMode=PERSISTENT`는 **메시지 내용**을 디스크에 fsync합니다
- `DeliveryMode=TRANSIENT`는 메시지를 메모리에만 저장

재시작 후:
- Queue는 존재 (durable=true 덕분)
- TRANSIENT 메시지는 소실 (메모리에만 있었으므로)
- PERSISTENT 메시지는 보존 (디스크에 저장됐으므로)

혼합 발행 시 (일부 PERSISTENT, 일부 TRANSIENT):
- 재시작 후 PERSISTENT 메시지만 남아 있음
- 순서가 뒤섞일 수 있음 (TRANSIENT 메시지가 중간에 있었다면)

</details>

---

**Q2.** Quorum Queue에서 3노드 중 2노드가 동시에 장애나면 어떻게 되는가?

<details>
<summary>해설 보기</summary>

Raft 알고리즘은 **과반수(majority)** 노드가 살아있어야 동작합니다.

3노드 클러스터에서 과반수 = 2노드

2노드 동시 장애 시:
- 남은 1노드는 과반수를 충족하지 못함
- Leader를 선출할 수 없음
- Queue는 **읽기/쓰기 모두 불가** (안전 모드)
- 데이터는 손실되지 않음 (1노드에 보존)
- 2노드 복구 후 정상 운영 재개

이것이 Raft의 **안전성(Safety) > 가용성(Availability)** 트레이드오프입니다.

CAP 이론에서 Quorum Queue는 CP(Consistency + Partition Tolerance)를 선택:
- 과반수 미충족 시 쓰기 거절 → 데이터 일관성 보장
- 일시적으로 서비스 불가 → 가용성 일부 포기

이를 피하려면 5노드 클러스터 사용: 5노드 중 2노드 장애해도 과반수(3) 충족

</details>

---

**Q3.** `x-max-length=1000`으로 설정된 Queue에 1001번째 메시지가 도착하면, 기본 overflow 정책(drop-head)에서는 어떤 메시지가 삭제되는가? 이 동작이 위험한 경우는 언제인가?

<details>
<summary>해설 보기</summary>

`drop-head` (기본값): **가장 오래된 메시지 (Queue Head)** 가 삭제됩니다.

Queue: [msg1(가장 오래됨)][msg2]...[msg1000] → msg1001 도착
→ msg1 삭제 → [msg2]...[msg1000][msg1001]

위험한 경우:
1. **순서 의존적인 처리**: msg1이 msg2보다 먼저 처리되어야 하는 경우, msg1이 삭제되면 데이터 일관성 깨짐
2. **중요한 메시지가 먼저 발행된 경우**: 주문 취소(최초 발행) → 주문 처리 순서인데, 취소 메시지가 drop되면 잘못 처리
3. **금융/결제 메시지**: drop-head로 메시지 유실 → 금전적 손실

안전한 대안:
- `reject-publish`: 새 메시지를 거절 (Queue Full 신호)
- `reject-publish-dlx`: 거절된 새 메시지를 DLX로 → Dead Letter Queue에서 모니터링
- Max Length를 충분히 크게 설정 + 모니터링으로 조기 경보

중요한 메시지에는 `drop-head`를 절대 사용하지 않는 것이 권장됩니다.

</details>

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[이전: Exchange 완전 분해 ⬅️](./03-exchange-internals.md)** | **[다음: 가상 호스트(vHost) ➡️](./05-vhost.md)**

</div>
