# Work Queue(경쟁 소비자) — 병렬 처리와 공정한 분배

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Work Queue 패턴에서 여러 Consumer가 어떻게 하나의 Queue를 나눠 처리하는가?
- Round-Robin 분배는 왜 불공정할 수 있고, Prefetch로 어떻게 해결하는가?
- 처리 시간이 불균일한 작업을 어떻게 효율적으로 병렬화하는가?
- Work Queue 패턴은 언제 Pub/Sub 패턴과 어떻게 다른가?
- Consumer 수를 늘리면 처리량이 선형으로 증가하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

이미지 리사이징, 이메일 발송, PDF 생성, 결제 처리처럼 CPU/IO를 많이 쓰는 작업을 HTTP 요청 내에서 처리하면 응답 시간이 길어진다. Work Queue 패턴은 이 작업들을 Queue에 넣고 여러 Worker가 병렬로 처리한다. 응답 시간과 처리 용량을 분리하는 핵심 패턴이다. 단순해 보이지만 Prefetch 설정 없이는 분배가 불공정해지고, Worker 장애 시 처리 중인 작업이 유실되는 문제가 생긴다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: Prefetch 설정 없는 Round-Robin → 불공정 분배

  Consumer A (빠름, 처리 10ms): prefetch=무제한
  Consumer B (느림, 처리 500ms): prefetch=무제한

  RabbitMQ 기본 동작:
    메시지 100개 → Consumer A: 50개, Consumer B: 50개 (Round-Robin)
    Consumer A: 50개 × 10ms = 0.5초에 완료 → 대기
    Consumer B: 50개 × 500ms = 25초에 완료
    Consumer A가 놀고 있는 동안 B는 과부하

  원인: Prefetch=무제한이면 RabbitMQ가 미리 50개씩 밀어넣음
        이미 받은 메시지는 B가 처리하든 A가 도와주든 불가

실수 2: autoAck=true로 Worker 장애 시 작업 유실

  Worker A: 이미지 10개 수신 (autoAck=true → 즉시 삭제)
  Worker A: 처리 중 OOM으로 종료
  → Queue에서 이미 삭제된 10개 작업 영구 유실
  → 이미지 10개가 처리 안 됨, 알 방법 없음

실수 3: 모든 작업을 하나의 Queue에 섞음

  image.resize.queue에 이미지 리사이징(10ms)과
  PDF 생성(2000ms)을 섞어 발행

  결과:
    PDF 작업 1개가 Worker 1개를 2초 동안 점유
    빠른 이미지 리사이징 작업이 대기
    → 작업 유형별 Queue 분리 필요
```

---

## ✨ 올바른 접근 (After — Prefetch + 수동 Ack)

```
Work Queue 올바른 설계:

1. Prefetch=1 (또는 소수) 설정:
   Consumer는 처리 완료 후 다음 메시지 수신
   → 빠른 Consumer가 더 많이, 느린 Consumer가 덜 처리
   → 자연스러운 부하 분산

2. 수동 Ack (autoAck=false):
   처리 완료 후 basicAck
   Worker 장애 시 Unacked → Ready 복귀 → 다른 Worker 처리

3. 작업 유형별 Queue 분리:
   image.resize.queue  (Consumer 5개, Prefetch=5)
   pdf.generate.queue  (Consumer 2개, Prefetch=1)
   email.send.queue    (Consumer 10개, Prefetch=10)

Spring AMQP 설정:
  spring:
    rabbitmq:
      listener:
        simple:
          acknowledge-mode: manual
          prefetch: 1
          concurrency: 3
          max-concurrency: 10  # 동적 스케일
```

---

## 🔬 내부 동작 원리

### 1. Round-Robin 기본 분배와 Prefetch의 역할

```
=== Prefetch 없는 Round-Robin ===

Queue: [T1][T2][T3][T4][T5][T6]  (처리 시간: T1=10ms, T2=500ms...)

Consumer A 연결 → RabbitMQ: T1, T3, T5 전달 (3개)
Consumer B 연결 → RabbitMQ: T2, T4, T6 전달 (3개)

(Prefetch=무제한 = 한 번에 모든 메시지 전달 가능)

T=0ms:
  A: T1(10ms), T3(10ms), T5(10ms) 순차 처리 시작
  B: T2(500ms) 처리 시작 (T4, T5는 로컬 버퍼에 대기)

T=30ms:
  A: T1, T3, T5 완료 → 새 메시지 요청
  RabbitMQ: Queue 비어있음 (B가 T4, T6 가지고 있으므로)
  A: 놀고 있음

T=1530ms:
  B: T2, T4, T6 완료
  전체 완료: 1530ms

=== Prefetch=1 적용 ===

Consumer A: prefetch=1
Consumer B: prefetch=1

T=0ms:   RabbitMQ → A: T1 / B: T2
T=10ms:  A: T1 완료 Ack → RabbitMQ: T3 전달
T=20ms:  A: T3 완료 Ack → RabbitMQ: T5 전달
T=30ms:  A: T5 완료 Ack → Queue에 T6 남음
T=500ms: B: T2 완료 Ack → RabbitMQ: T4 전달 (T6은 A가 이미 처리)
T=510ms: B: T4 완료 Ack
T=530ms: B: T6 완료

A 처리: T1, T3, T5 + T6 = 4개 (빠르니까 더 많이)
B 처리: T2, T4 = 2개 (느리니까 덜)
전체 완료: 530ms (Prefetch 없을 때 1530ms vs 530ms → 2.9배 빠름)
```

### 2. Worker 장애와 메시지 복구

```
=== Unacked 메시지 복구 흐름 ===

정상 동작:
  Queue: [job1][job2][job3]
  Worker A: job1 수신 → Unacked 상태
  Worker B: job2 수신 → Unacked 상태
  Queue Ready: [job3]

  Worker A: job1 처리 완료 → basicAck → 삭제
  Worker B: job2 처리 완료 → basicAck → 삭제
  Worker C: job3 수신 → ...

Worker A 장애 (처리 중 kill):
  job1: Unacked 상태 → Worker A Connection 끊김
  RabbitMQ: Heartbeat 실패 감지 → Connection 닫음
  RabbitMQ: job1 Unacked → Ready 복귀
  Worker B 또는 C: job1 재수신 → 처리

=== 메시지 재전달 표시 ===

재전달된 메시지는 redelivered=true 헤더:

  @RabbitListener(queues = "job.queue")
  public void processJob(Message message, Channel channel,
      @Header(AmqpHeaders.DELIVERY_TAG) long tag,
      @Header(value = AmqpHeaders.REDELIVERED, required = false)
          Boolean redelivered) throws IOException {

    if (Boolean.TRUE.equals(redelivered)) {
      // 이전 Worker가 처리 중 실패한 메시지
      // 멱등성 확인 필요 (이미 처리됐을 수도 있음)
      if (jobRepository.isCompleted(getJobId(message))) {
        channel.basicAck(tag, false);  // 이미 완료됨 → Ack
        return;
      }
    }

    processJob(message);
    channel.basicAck(tag, false);
  }

=== 장기 실행 작업의 Heartbeat 문제 ===

작업 처리 시간 > Heartbeat 타임아웃:
  처리 시간: 5분 (대용량 이미지 처리)
  Heartbeat: 60초 (기본)

  RabbitMQ: 60초 × 2 = 120초 동안 Heartbeat 없으면 Connection 종료
  → 5분 처리 중 Connection 끊김 → 작업 중단, 재전달

해결:
  1. Heartbeat 비활성화 (권장 안 함, 좀비 Connection 위험)
  2. 처리 중 주기적으로 Channel에 활동 신호:
     // 주기적으로 호출
     connection.createChannel().close();  // heartbeat 대신 활동 유지
  3. 작업을 청크로 분할 (각 청크 30초 이내)
  4. 별도 Thread에서 Heartbeat 유지
```

### 3. Worker 수 동적 조정

```
=== Consumer 수와 처리량의 관계 ===

이상적 경우 (처리량 선형 증가):
  Worker 1개: 100 job/min
  Worker 2개: 200 job/min
  Worker N개: N × 100 job/min

실제 병목 요인:
  1. Queue 자체 처리량 (브로커 성능)
  2. 공유 DB 연결 풀 (Worker 증가 → DB 경합)
  3. 공유 외부 API Rate Limit
  4. 네트워크 대역폭

처리량 선형 증가가 멈추는 지점 탐지:
  Worker N개에서 Queue Depth 감소율 측정
  Worker N+1개에서 변화 없으면 → 다른 병목 존재

=== Spring AMQP 동적 Concurrency ===

  spring:
    rabbitmq:
      listener:
        simple:
          concurrency: 3      # 초기 Consumer 스레드 수
          max-concurrency: 20 # 최대 Consumer 스레드 수

  동작:
    Queue Depth 증가 → 자동으로 스레드 추가 (최대 max-concurrency)
    Queue Depth 감소 → 자동으로 스레드 제거 (최소 concurrency)

  @RabbitListener(
    queues = "job.queue",
    concurrency = "3-20"  // 최소 3, 최대 20
  )
  public void processJob(JobMessage job) { ... }

=== 외부 오토스케일링 (Kubernetes HPA) ===

  Kubernetes HPA + RabbitMQ KEDA:
    Queue Depth > 100 → Pod 추가
    Queue Depth < 10 → Pod 제거

  keda ScaledObject:
    triggers:
      - type: rabbitmq
        metadata:
          queueName: job.queue
          queueLength: "50"  # Pod당 50개 메시지 기준
```

---

## 💻 실전 실험

### 실험 1: Prefetch=1 vs 무제한 처리 시간 비교

```bash
# Queue 생성 및 메시지 100개 발행 (처리 시간 불균일 시뮬레이션)
rabbitmqadmin declare queue name=job.queue durable=true
for i in $(seq 1 100); do
  rabbitmqadmin publish exchange="" routing_key=job.queue \
    payload="{\"jobId\":$i,\"type\":\"$([ $((i % 10)) -eq 0 ] && echo slow || echo fast)\"}"
done
```

```java
// Prefetch=1 Consumer
@RabbitListener(queues = "job.queue")
public void processJob(JobMessage job, Channel channel,
    @Header(AmqpHeaders.DELIVERY_TAG) long tag) throws Exception {

  int delay = "slow".equals(job.getType()) ? 500 : 10;
  Thread.sleep(delay);  // 처리 시간 시뮬레이션

  channel.basicAck(tag, false);
  log.info("Processed: jobId={}, type={}, thread={}",
    job.getJobId(), job.getType(), Thread.currentThread().getName());
}

// Management UI에서 Consumer별 처리 수 확인
// Prefetch=1: 빠른 Consumer가 더 많이 처리 (공정 분배)
// Prefetch=50: 초반에 받은 수만큼만 처리 (불공정)
```

### 실험 2: Worker 장애 복구 확인

```bash
# 메시지 발행
rabbitmqadmin publish exchange="" routing_key=job.queue payload='{"jobId":1}'

# Consumer A: 메시지 수신 (Unacked 상태)
rabbitmqadmin get queue=job.queue ackmode=ack_requeue_true

rabbitmqctl list_queues name messages_ready messages_unacknowledged
# job.queue  0  1  ← Unacked=1

# rabbitmqadmin 종료 (Worker 장애 시뮬레이션)
# → Connection 끊김 → Unacked → Ready 복귀

rabbitmqctl list_queues name messages_ready messages_unacknowledged
# job.queue  1  0  ← 복구됨
```

---

## 📊 Kafka vs RabbitMQ 비교

```
=== Work Queue 패턴 비교 ===

RabbitMQ Work Queue:
  하나의 Queue → 여러 Consumer 경쟁 소비
  Consumer 수 = 병렬 처리 수
  Consumer 추가/제거: 즉시 (Queue 공유)
  메시지 처리 후 삭제 → 재처리 불가

Kafka Consumer Group:
  하나의 Topic → Consumer Group으로 병렬 처리
  Consumer 수 ≤ Partition 수 (Partition이 병렬 단위)
  Consumer 추가 → Rebalancing (수 초 중단)
  메시지 보존 → 재처리 가능

=== 선택 기준 ===

"이미지 리사이징 Worker 10개로 병렬 처리":
  RabbitMQ: Consumer 10개 → 즉시 병렬화, 단순한 설정
  Kafka: Partition 10개 설정 필요, Partition 수 = 최대 병렬도

"작업 처리 실패 시 재처리":
  RabbitMQ: DLX + DLQ → 수동 재발행
  Kafka: 오프셋 리셋 → 자동 재처리 (더 강력)

"Worker 수 동적 변경":
  RabbitMQ: Consumer 추가/제거 즉시 적용
  Kafka: Rebalancing 발생 (수 초 처리 중단)
  → 동적 스케일링: RabbitMQ 더 유리
```

---

## ⚖️ 트레이드오프

```
Work Queue 설계 트레이드오프:

Prefetch=1:
  ✅ 공정한 분배 (처리 속도 비례)
  ✅ Worker 장애 시 최소 재처리
  ❌ 처리량 약간 낮음 (네트워크 대기)
  → 처리 시간 불균일할 때 권장

Prefetch 높음:
  ✅ 처리량 높음
  ❌ 불공정 분배, 장애 시 많은 재처리
  → 처리 시간 균일하고 처리량 최우선일 때

Queue 하나 vs 여러 개:
  하나: 관리 단순, 모든 작업이 같은 처우
  여러 개: 작업 유형별 우선순위/Worker 수 독립 설정
  → 처리 시간이 크게 다르면 Queue 분리 권장
```

---

## 📌 핵심 정리

```
Work Queue 핵심:

패턴:
  하나의 Queue → 여러 Consumer 경쟁 소비
  각 메시지는 하나의 Consumer만 처리 (Pub/Sub과 차이)

핵심 설정:
  Prefetch=1: 공정 분배 (처리 완료 후 다음 수신)
  autoAck=false: Worker 장애 시 메시지 복구
  Durable Queue + PERSISTENT: 재시작 후 작업 보존

처리량 확장:
  Consumer 수 증가 → 처리량 증가 (병목 지점까지)
  Spring AMQP: concurrency/max-concurrency로 동적 조정
  Kubernetes HPA + KEDA: Pod 기반 오토스케일링

적합한 사용 사례:
  이미지/동영상 처리, PDF 생성 (CPU 집약)
  이메일/SMS 발송 (IO 집약)
  배치 데이터 처리
  결제, 정산 등 신뢰성 필요한 비동기 작업
```

---

## 🤔 생각해볼 문제

**Q1.** Worker 5개가 같은 Queue를 소비한다. 그 중 하나가 처리 실패로 `basicNack(requeue=true)`를 보냈다. 해당 메시지는 반드시 다른 Worker에게 전달되는가?

<details>
<summary>해설 보기</summary>

**반드시 다른 Worker에게 가지는 않습니다.** RabbitMQ는 Nack 후 재큐된 메시지를 동일한 Consumer에게 다시 전달할 수 있습니다.

RabbitMQ의 Round-Robin은 "현재 Prefetch 슬롯이 남은 Consumer"에게 전달합니다. Nack로 슬롯을 비운 Consumer A가 다시 슬롯이 생기면 같은 메시지를 받을 수 있습니다.

이것이 문제가 되는 경우:
- 같은 Worker에서 반복 실패 → 무한 루프
- 해결: 즉시 requeue=true 대신 TTL+DLX Backoff (Ch3-05 참고)

다른 Worker에게 반드시 보내고 싶다면:
- requeue=false + DLX → 별도 재처리 Queue로 이동
- 재처리 Queue의 Consumer가 처리 (원래 Consumer와 다를 수 있음)

</details>

---

**Q2.** Worker A가 처리 중인 메시지를 절반쯤 처리했다(DB에 중간 저장). 그 상태에서 Worker A가 죽으면, 재전달된 메시지를 Worker B가 처음부터 다시 처리한다. 이때 중간 저장된 데이터는 어떻게 처리해야 하는가?

<details>
<summary>해설 보기</summary>

이것이 **멱등성 처리**의 핵심 문제입니다.

시나리오:
1. Worker A: 이미지 변환 50% 완료 → 중간 결과 S3 저장
2. Worker A: 장애로 종료 (Ack 없음)
3. Worker B: 같은 이미지 처리 재시작

해결 방법:

**방법 1: 중간 결과 덮어쓰기 (Idempotent)**
- Worker B가 처음부터 다시 처리 → S3 중간 결과 덮어씀
- 완료 후 최종 결과 저장
- 여러 번 처리해도 결과가 동일하면 OK

**방법 2: 작업 상태 테이블 활용**
```java
if (jobRepository.isInProgress(jobId)) {
  // 이미 처리 중인 작업 → 재처리 전 정리
  cleanupPartialResult(jobId);
}
// 처음부터 처리
processJob(message);
// 완료 표시
jobRepository.markComplete(jobId);
channel.basicAck(tag, false);
```

**방법 3: 작업 청크 분할**
- 큰 작업을 N개의 작은 청크로 분할
- 각 청크를 별도 메시지로 발행
- 각 청크는 독립적으로 멱등 처리 가능

핵심: Work Queue에서는 **Worker 멱등성**이 필수입니다.

</details>

---

**Q3.** "초당 1000개 이미지를 처리해야 한다. 이미지 하나 처리에 100ms 걸린다. Worker가 최소 몇 개 필요한가?"

<details>
<summary>해설 보기</summary>

**이론적 계산:**

하나의 Worker가 1초에 처리하는 이미지: 1000ms / 100ms = 10개

필요 Worker 수: 1000개/초 ÷ 10개/초 = **100개 Worker**

**실제 고려사항:**

1. **안전 마진**: 피크 시 20~30% 여유를 두고 130개로 설정
2. **DB 연결 풀**: 100개 Worker × 각 1개 DB 연결 = 100개 연결 (DB 풀 한도 확인)
3. **네트워크 I/O**: 이미지 다운로드/업로드가 있으면 네트워크 대역폭 확인
4. **CPU**: 100개 스레드가 동시에 CPU를 사용 → CPU 코어 수 고려

**실제 접근:**
- Worker 10개로 시작 → 처리량 측정
- 처리량 ∝ Worker 수면 선형 확장
- 처리량 증가 멈추는 지점 = 병목 식별 → 해당 병목 해결 후 다시 확장

Spring AMQP concurrency=100은 **100개 스레드**를 하나의 Pod에서 실행합니다. Pod 리소스(CPU/메모리)가 이를 감당하지 못하면 여러 Pod로 분산해야 합니다.

</details>

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[이전: Chapter 3 — 트랜잭션 vs Publisher Confirm ⬅️](../message-reliability/07-transaction-vs-confirm.md)** | **[다음: Publish/Subscribe ➡️](./02-pubsub.md)**

</div>
