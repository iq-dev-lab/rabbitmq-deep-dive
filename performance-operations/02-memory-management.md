# 메모리 관리 — Flow Control과 Lazy Queue로 안정성 확보

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `vm_memory_high_watermark`는 무엇이고, 초과 시 어떤 일이 발생하는가?
- Flow Control은 Publisher에게 어떻게 작동하는가?
- Lazy Queue는 메모리를 어떻게 절약하는가?
- 메모리 부족 시 RabbitMQ는 어떤 순서로 메시지를 디스크로 내리는가?
- 디스크 여유 공간이 부족할 때는 어떻게 되는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

RabbitMQ 브로커가 갑자기 메시지를 받지 않는다. Publisher는 연결돼 있는데 발행이 차단된다. 이것이 Flow Control이다. 메모리 임계값을 초과하면 RabbitMQ가 모든 Publisher를 차단한다. 이 원리를 모르면 "RabbitMQ가 다운됐다"고 오진하고 재시작한다 — 재시작해도 메모리 문제가 해결 안 됐으면 다시 같은 상황이 된다. 메모리 관리를 이해해야 안정적인 브로커 운영이 가능하다.

---

## 😱 흔한 실수 (Before)

```
실수 1: Flow Control을 브로커 다운으로 오진

  Publisher: 메시지 발행 안 됨 (block됨)
  모니터링: 브로커 연결은 살아있음
  팀: "RabbitMQ 재시작 !"
  재시작 후: 잠깐 괜찮다가 5분 후 또 차단
  
  실제 원인: Consumer 없이 메시지만 쌓여 메모리 초과
  해결: Consumer 처리량 증가 또는 Lazy Queue

실수 2: vm_memory_high_watermark를 너무 높게 설정

  vm_memory_high_watermark = 0.9  # 메모리 90%까지 허용
  
  문제:
    90% 도달 순간 Flow Control 발동
    이 시점에 이미 OS가 스왑 사용 중
    Flow Control 해제되려면 메모리 회수 필요
    → 브로커 전체 응답 느려짐
    
  권장: 0.4 (40%) — 여유 있게 경보

실수 3: 디스크 여유 공간 미모니터링

  RabbitMQ: disk_free_limit 기본값 50MB
  디스크 여유 < 50MB → 모든 Publisher 차단 (disk alarm)
  
  로그 파일이 계속 증가하다 디스크 꽉 참
  → 갑작스러운 발행 차단
```

---

## ✨ 올바른 접근 (After)

```
메모리 관리 설정:

rabbitmq.conf:
  vm_memory_high_watermark.relative = 0.4      # 40% 초과 시 Flow Control
  vm_memory_high_watermark_paging_ratio = 0.5  # 20% 도달 시 디스크 Paging
  disk_free_limit.relative = 1.5               # 총 메모리의 1.5배 디스크 여유 필요

대용량 메시지 적재 예상:
  Lazy Queue 설정:
    rabbitmqctl set_policy LazyQueues ".*" \
      '{"queue-mode":"lazy"}' --apply-to queues
  
  또는 Queue 별로:
    QueueBuilder.durable("order.queue").lazy()
```

---

## 🔬 내부 동작 원리

### 1. 메모리 임계값과 Flow Control

```
=== 메모리 계층 ===

100% ├─────────────────────────────────────────────
     │  OS 커널 / 다른 프로세스 사용
 60% ├─────────────────────────────────────────────
     │  ④ Erlang VM 메모리 (RabbitMQ 프로세스)
 40% ├─────── vm_memory_high_watermark = 0.4 ──────  ← Flow Control 발동
     │  ③ RabbitMQ 메시지 메모리 (alpha/beta)
 20% ├─────── 0.5 × 0.4 = 0.2 ────────────────────  ← Paging 시작
     │  ② RabbitMQ 메시지 (gamma → delta)
  0% └─────────────────────────────────────────────

=== Flow Control 동작 ===

1. 메모리 사용량 모니터링 (memory_monitor 프로세스)
   → 40% 초과 감지

2. memory alarm 발동:
   - 모든 Publisher Connection에 credit 0 설정
   - Connection.Blocked 알림 발송
   - 새 basicPublish 요청 → 차단 (TCP 소켓 버퍼에 대기)

3. Publisher 측 동작:
   Spring AMQP: BlockedChannelPublisher 상태로 전환
   rabbitTemplate.send() → 블로킹 (또는 예외)
   ConnectionListener:
     void connectionBlocked(String reason);   // Flow Control 시작
     void connectionUnblocked();              // Flow Control 해제

4. Consumer는 계속 동작:
   Flow Control은 Publisher만 차단
   Consumer가 메시지 소비 → 메모리 확보 → Flow Control 해제

=== 메모리 회수 순서 ===

RabbitMQ의 메모리 압박 대응:
  1. alpha(메모리 full) → beta(메타데이터만) 전환
  2. beta → gamma(디스크+메타) 전환
  3. gamma → delta(디스크 only) 전환
  
  각 단계를 "Paging"이라 부름
  Paging 중에는 처리 속도 저하 (디스크 I/O)
  
  vm_memory_high_watermark_paging_ratio = 0.5:
    high_watermark × paging_ratio = 0.4 × 0.5 = 20%
    → 20%에서 Paging 시작 (40% Flow Control 전에 미리 디스크로)
```

### 2. Lazy Queue 메모리 절약 원리

```
=== 일반 Queue vs Lazy Queue 메모리 비교 ===

시나리오: 100만 개 메시지 적재 (각 1KB)

일반 Queue:
  초기: 메시지를 메모리에 저장
  메모리 40% 도달 → Paging 시작 (디스크로 이동)
  Paging 중 성능 저하
  메모리 사용: 수백 MB ~ 수 GB

Lazy Queue:
  메시지 도착 즉시 디스크 저장
  메모리에는 인덱스 + 메타데이터만 보관
  100만 개 × 메타데이터 크기(~100B) = ~100MB
  → Paging 없음, 메모리 안정

=== Lazy Queue 설정 ===

선언 시:
  QueueBuilder.durable("order.queue").lazy()

기존 Queue에 Policy로 적용:
  rabbitmqctl set_policy LazyPolicy "order\\.queue" \
    '{"queue-mode":"lazy"}' --apply-to queues
  
  // 기존 메시지도 즉시 디스크로 이동 (단, 처리 중단 없음)

Lazy Queue 해제:
  rabbitmqctl clear_policy LazyPolicy
  // 이후 메시지는 메모리에 저장 (기존 디스크 메시지도 점진적으로 메모리로)

=== 디스크 임계값 ===

disk_free_limit:
  default: 50MB (너무 낮음)
  권장: 총 메모리의 1.5배 이상

  # 총 메모리 8GB이면 → 12GB 이상 디스크 여유 필요
  disk_free_limit.relative = 1.5

디스크 임계값 초과 시:
  disk alarm 발동 → memory alarm과 동일하게 Publisher 차단
  disk_free_limit 미만 디스크 → "no space to write new messages"

=== 메모리 사용량 분석 ===

rabbitmqctl status:
  {memory, [
    {connection_readers, 16MB},
    {connection_writers, 4MB},
    {queue_procs, 1.2GB},      ← Queue 메시지 메모리
    {msg_index, 200MB},        ← Queue 인덱스
    {binary, 800MB},           ← 바이너리 데이터
    {mnesia, 50MB},            ← 메타데이터 DB
    {total, 2.3GB}
  ]}

queue_procs가 크면 → Lazy Queue로 전환 검토
binary가 크면 → 메시지 크기 검토 또는 압축
```

---

## 💻 실전 실험

### 실험 1: Flow Control 발동 확인

```bash
# 메모리 임계값 낮춰서 Flow Control 테스트
rabbitmqctl set_vm_memory_high_watermark 0.1  # 10%로 임시 낮춤

# 메시지 대량 발행 (Flow Control 유발)
for i in $(seq 1 10000); do
  rabbitmqadmin publish exchange="" routing_key=test.queue \
    payload="$(python3 -c 'print("x"*10000)')" 2>/dev/null
done

# Flow Control 상태 확인
rabbitmqctl status | grep -A5 "alarms"
# [{memory,<rabbit@node>}]  ← memory alarm 발동

# Management UI → Overview → "Nodes" 섹션
# 노드 옆에 주황색 "memory" 경고 표시

# Flow Control 해제 (메모리 다시 높게)
rabbitmqctl set_vm_memory_high_watermark 0.4
```

### 실험 2: Lazy Queue 메모리 비교

```bash
# 일반 Queue vs Lazy Queue 메모리 비교
rabbitmqadmin declare queue name=normal.q durable=true
rabbitmqadmin declare queue name=lazy.q durable=true \
  arguments='{"x-queue-mode":"lazy"}'

# 1000개 대형 메시지 발행 (각 10KB)
msg=$(python3 -c 'print("x"*10000)')
for i in $(seq 1 1000); do
  rabbitmqadmin publish exchange="" routing_key=normal.q payload="$msg" &
  rabbitmqadmin publish exchange="" routing_key=lazy.q payload="$msg" &
done
wait

# 메모리 비교
rabbitmqctl list_queues name messages memory
# normal.q  1000  12000000  (약 12MB 메모리)
# lazy.q    1000  300000    (약 300KB 메모리)
```

---

## 📊 Kafka vs RabbitMQ 비교

```
=== 메모리 관리 비교 ===

RabbitMQ:
  메시지 메모리 우선 저장 → 임계값 초과 시 디스크 Paging
  메모리 임계값 설정 (vm_memory_high_watermark)
  Flow Control로 Publisher 차단 (Consumer 계속 동작)
  Lazy Queue: 즉시 디스크 저장

Kafka:
  디스크 우선 저장 (append-only log)
  OS Page Cache로 읽기 성능 최적화
  메모리 관련 Flow Control 없음
  Consumer Lag으로 처리 지연 표현

=== 대량 메시지 적재 ===

Consumer 없이 100만 건 적재:
  RabbitMQ: 메모리 문제 → Flow Control 위험
  → Lazy Queue 필수
  
  Kafka: 디스크에 저장 → 메모리 이슈 없음
  → 기본 동작으로 안정적

결론: 대량 메시지 장기 보관: Kafka 유리
      메모리 관리 신경 쓸 필요 없음
```

---

## ⚖️ 트레이드오프

```
메모리 설정 트레이드오프:

vm_memory_high_watermark 높음 (예: 0.7):
  ✅ 메모리 더 많이 활용 (처리량 높음)
  ❌ OS가 이미 스왑 중인 상태에서 Flow Control
  ❌ Flow Control 해제 늦음

vm_memory_high_watermark 낮음 (예: 0.3):
  ✅ 일찍 Flow Control → 여유 있게 대응
  ❌ 메모리 여유가 있는데도 Flow Control

Lazy Queue:
  ✅ 메모리 극적 절약
  ❌ Consumer 처리 속도 감소 (디스크 I/O)
  → Consumer가 따라잡을 수 없을 때만 사용

권장 설정:
  vm_memory_high_watermark = 0.4
  vm_memory_high_watermark_paging_ratio = 0.5
  disk_free_limit.relative = 1.5
  Lazy Queue: 메시지 폭발 Queue에 선택적 적용
```

---

## 📌 핵심 정리

```
메모리 관리 핵심:

임계값:
  vm_memory_high_watermark = 0.4 (40%)
  초과 시 모든 Publisher 차단 (Flow Control)
  Consumer는 계속 동작 → 메모리 회수

Paging:
  paging_ratio 도달 시 (기본 20%)
  메시지를 디스크로 이동 시작 (alpha→delta)
  Paging 중 성능 저하

Lazy Queue:
  즉시 디스크 저장 → 메모리 절약
  Consumer 처리 속도 다소 감소
  Consumer가 따라가지 못할 때 적합

Flow Control 대응:
  원인: Consumer 처리 속도 < 발행 속도
  해결: Consumer Scale Out, Lazy Queue
  임시: Flow Control 임계값 확인 후 Consumer 증가

디스크 임계값:
  disk_free_limit = 메모리 × 1.5 권장
  디스크 부족 → Publisher 차단
```

---

## 🤔 생각해볼 문제

**Q1.** Flow Control이 발동됐을 때 Publisher 애플리케이션은 어떻게 동작하는가? 타임아웃이 없으면 어떤 문제가 생기는가?

<details>
<summary>해설 보기</summary>

**RabbitMQ가 Connection.Blocked 프레임을 전송** → Publisher가 TCP 수준에서 차단됩니다.

Spring AMQP 동작:
- `rabbitTemplate.send()`가 블로킹됨
- HTTP 요청 스레드가 `send()` 내부에서 대기
- 타임아웃 없으면 → **스레드 무한 대기** → 스레드 풀 고갈 → 서비스 전체 마비

대응:
```java
// Connection 차단 감지
connectionFactory.addConnectionListener(new ConnectionListener() {
  void connectionBlocked(String reason) {
    isBlocked.set(true);
    log.warn("RabbitMQ Flow Control: {}", reason);
    // 알림 발송, 발행 중단 결정
  }
  void connectionUnblocked() {
    isBlocked.set(false);
    log.info("RabbitMQ Flow Control 해제");
  }
});

// 발행 전 차단 상태 확인
if (isBlocked.get()) {
  throw new ServiceUnavailableException("RabbitMQ Flow Control 중");
}
```

**추가 권장**: HTTP 요청 내에서 RabbitMQ 발행 시 타임아웃 설정:
```java
template.setChannelTransacted(false);
template.setReceiveTimeout(5000);  // 5초 제한
```

</details>

---

**Q2.** Lazy Queue로 전환하면 기존 메모리에 있는 메시지는 어떻게 되는가?

<details>
<summary>해설 보기</summary>

**기존 메시지는 점진적으로 디스크로 이동합니다.** 즉각적이지 않으며 처리 중단도 없습니다.

Policy로 Lazy 전환:
```bash
rabbitmqctl set_policy LazyMode "my.queue" '{"queue-mode":"lazy"}' --apply-to queues
```

전환 과정:
1. Policy 적용 즉시 → 새 메시지는 즉시 디스크 저장
2. 기존 메모리 메시지 → 백그라운드에서 점진적으로 디스크 이동
3. 이동 중에도 Consumer는 정상 동작

일시적 효과:
- 전환 직후 잠깐 디스크 I/O 증가 (기존 메시지 이동)
- 메모리 점진적 감소

롤백 (일반 모드로 복귀):
```bash
rabbitmqctl clear_policy LazyMode
```
- 이후 새 메시지는 메모리에 저장
- 디스크의 기존 메시지는 필요 시 메모리로 로드

</details>

---

**Q3.** `vm_memory_high_watermark = 0.4`인데 실제 메모리 사용량이 35%다. 그런데 Management UI에서 "memory alarm" 경고가 표시된다. 왜인가?

<details>
<summary>해설 보기</summary>

**RabbitMQ가 측정하는 메모리와 OS가 측정하는 메모리가 다를 수 있습니다.**

원인 후보:

1. **RabbitMQ의 메모리 계산 방식**: Erlang VM의 메모리 사용량과 OS의 RSS(Resident Set Size)가 다를 수 있습니다. RabbitMQ는 Erlang VM 기준으로 계산합니다.

2. **Erlang binary 메모리**: 메시지 페이로드가 Erlang binary 힙에 할당되며, 이것이 `vm_memory_high_watermark` 계산에 포함됩니다.

3. **메모리 계산 방식 설정**:
```bash
# 기본: rss (OS RSS)
rabbitmqctl set_vm_memory_high_watermark_absolute 2GB
# 또는
rabbitmq.conf: vm_memory_calculation_strategy = rss | erlang | legacy
```

진단:
```bash
rabbitmqctl status | grep -A20 "memory"
# {memory, [{total, N}, {erlang, M}, {rss, K}]}
# total vs rss 비교로 실제 계산 방식 확인
```

해결: `vm_memory_calculation_strategy = rss` 설정으로 OS RSS 기준으로 변경하면 더 직관적인 임계값 적용이 가능합니다.

</details>

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[이전: 처리량 최적화 ⬅️](./01-throughput-optimization.md)** | **[다음: 모니터링 핵심 지표 ➡️](./03-monitoring.md)**

</div>
