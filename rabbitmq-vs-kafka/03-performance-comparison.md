# 성능 특성 비교 — 레이턴시, 처리량, 튜닝 전략

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- RabbitMQ와 Kafka의 레이턴시 차이는 어디에서 비롯되는가?
- Kafka가 더 높은 처리량을 달성하는 근본 이유는 무엇인가?
- Prefetch(RabbitMQ)와 Batch Size(Kafka)는 어떻게 대응되는 개념인가?
- 메시지 크기가 성능에 미치는 영향은 두 시스템에서 어떻게 다른가?
- 처리량과 레이턴시의 트레이드오프를 어떻게 조절하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

"Kafka가 더 빠르다"는 말은 반만 맞다. Kafka는 처리량이 높고, RabbitMQ는 레이턴시가 낮다. 알림 서비스에서 Kafka의 50ms 레이턴시는 허용 불가할 수 있고, 분석 파이프라인에서 RabbitMQ의 처리량 한계는 병목이 될 수 있다. 각 시스템의 성능 특성을 이해해야 요구사항에 맞는 선택과 튜닝이 가능하다.

---

## 😱 흔한 실수 (Before)

```
실수 1: "Kafka가 더 빠르니까 Kafka를 써야 한다"

  실시간 Push 알림 서비스:
    목표 레이턴시: 메시지 발행 → 앱 수신 < 10ms
    
    RabbitMQ Push: 발행 → 즉시 Consumer에게 → 2~5ms ✅
    Kafka Pull:    발행 → Consumer poll() → 10~50ms ❌
    
  레이턴시 기준이면 RabbitMQ가 더 적합

실수 2: 작은 메시지를 Kafka에서 하나씩 발행

  // Kafka Producer에서 매 메시지마다 flush
  producer.send(record);
  producer.flush();  // 즉시 전송 강제
  
  성능:
    flush()마다 네트워크 호출 → 처리량 급감
    Kafka의 장점(배치)을 포기한 것
    
  올바른 방법: linger.ms=5 로 배치 허용

실수 3: RabbitMQ에서 Persistent 없이 처리량 비교

  "RabbitMQ가 Kafka보다 처리량이 높다"고 측정
  → 알고 보니 RabbitMQ: TRANSIENT + autoAck=true
     Kafka: acks=all + PERSISTENT
  → 공정한 비교가 아님
```

---

## ✨ 올바른 접근 (After — 목적에 맞는 성능 튜닝)

```
RabbitMQ 처리량 최대화:
  비동기 Publisher Confirm
  TRANSIENT (신뢰성 허용 시)
  autoAck=true (처리 보장 불필요 시)
  concurrency + prefetch 최적화

Kafka 레이턴시 최소화:
  linger.ms=0 (배치 없이 즉시 전송)
  fetch.min.bytes=1 (Consumer: 최소 1바이트면 즉시 반환)
  acks=1 (ISR 전체 대신 Leader만)
  → 레이턴시: 10~20ms로 감소

Kafka 처리량 최대화:
  linger.ms=5~10 (배치 허용)
  batch.size=1MB
  compression.type=snappy
  fetch.min.bytes=1MB (Consumer: 1MB 채워야 반환)
```

---

## 🔬 내부 동작 원리

### 1. 레이턴시 구조 비교

```
=== RabbitMQ 레이턴시 분해 ===

단계별 레이턴시:
  1. Producer → TCP → Broker: ~1ms (같은 데이터센터)
  2. Exchange 라우팅: ~0.1ms (O(1) Direct, O(B×L) Topic)
  3. Queue 저장: ~0.1ms (메모리) / ~1ms (디스크 fsync)
  4. Consumer에게 Push: ~0.5ms
  5. Consumer 수신: ~0.1ms

총 레이턴시:
  TRANSIENT + autoAck: ~2ms (메모리, fsync 없음)
  PERSISTENT + 수동 Ack: ~5ms (fsync 포함)
  Quorum Queue: ~10ms (Raft 합의 포함)

=== Kafka 레이턴시 분해 ===

단계별 레이턴시:
  1. Producer → TCP → Broker: ~1ms
  2. Partition append: ~1ms (append-only, 순차 쓰기)
  3. ISR 복제: ~2ms (acks=all)
  4. Consumer poll() 대기: 0~fetch.max.wait.ms (기본 500ms)
  5. Consumer 수신: ~0.1ms

총 레이턴시 (기본 설정):
  linger.ms=5, fetch.min.bytes=1: 5~50ms
  
레이턴시 최소화 설정:
  Producer: linger.ms=0, acks=1
  Consumer: fetch.min.bytes=1, fetch.max.wait.ms=0
  결과: ~10~20ms

=== poll() 대기가 레이턴시를 높이는 이유 ===

Consumer가 poll(timeout=100)을 호출:
  "최대 100ms를 기다리고, 메시지 있으면 즉시 반환"
  
  메시지가 있을 때: 즉시 반환 (낮은 레이턴시)
  메시지가 없을 때: 100ms 대기 후 반환 (높은 레이턴시)

fetch.min.bytes=1000 설정:
  "1000바이트가 쌓일 때까지 또는 fetch.max.wait.ms까지 대기"
  → 처리량 증가, 레이턴시 증가

fetch.min.bytes=1 + fetch.max.wait.ms=10:
  "바이트가 있으면 최대 10ms만 기다리고 반환"
  → 레이턴시 감소, 처리량 약간 감소
```

### 2. 처리량 구조 비교

```
=== Kafka가 높은 처리량을 달성하는 이유 ===

1. 순차 디스크 쓰기 (append-only):
   Kafka: 파일 끝에 append → 순차 I/O (SSD: 수 GB/s)
   RabbitMQ: Queue 인덱스 + 메시지 파일 → 랜덤 I/O 포함

   순차 I/O vs 랜덤 I/O:
   SSD 순차 읽기: ~3000 MB/s
   SSD 랜덤 읽기: ~400 MB/s (4KB 블록)
   → 순차 I/O가 7~8배 빠름

2. 배치 처리:
   Producer: linger.ms=5로 5ms 동안 메시지 모아서 한 번에 전송
   Consumer: poll()로 최대 max.poll.records개씩 배치 처리
   → 네트워크 왕복 수 감소 (N개 메시지에 1번 왕복)

3. Zero-copy 전송:
   Kafka: sendfile() 시스템 콜로 OS 커널에서 직접 Consumer로 전송
   → 데이터를 User Space로 복사하지 않음 (CPU 절약)

4. 파티션 병렬 처리:
   Topic Partition 30개 → 30개 Consumer가 병렬 읽기
   → 선형 확장 가능

=== RabbitMQ 처리량 한계 ===

RabbitMQ Exchange → Queue → Consumer 흐름:
  메시지마다 Exchange 평가 → Queue 저장 → Consumer Push
  Ack 수신 → Queue에서 삭제 (인덱스 업데이트)
  → 여러 단계의 상태 관리가 처리량 상한을 결정

Queue당 단일 Erlang 프로세스:
  하나의 Queue는 단일 Erlang 프로세스가 관리
  → 한 Queue의 처리량 상한: 수만 msg/sec

여러 Queue로 분산:
  여러 Queue를 활용하면 병렬 처리 가능
  → 총 처리량은 큐 수에 비례 (Kafka의 파티션과 유사)

=== 실제 처리량 수치 (참고치, 환경에 따라 다름) ===

RabbitMQ (일반 서버, 최적화):
  - PERSISTENT + 수동 Ack: ~20,000 msg/sec (단일 Queue)
  - TRANSIENT + autoAck: ~100,000 msg/sec (단일 Queue)
  - 여러 Queue + 비동기 Confirm: ~500,000 msg/sec

Kafka (일반 서버, 최적화):
  - acks=all + snappy 압축: ~500,000 msg/sec (단일 브로커)
  - acks=1 + 배치 + 압축: ~3,000,000 msg/sec (단일 브로커)
  - 클러스터 확장: 선형 증가 (파티션 수 비례)
```

### 3. Prefetch(RabbitMQ) vs Batch Size(Kafka) 대응

```
=== 개념 대응 ===

RabbitMQ Prefetch:
  "브로커가 한 번에 Consumer에게 Push할 수 있는 최대 메시지 수"
  Consumer가 처리 완료(Ack) 전까지 추가 Push 차단
  
  역할: 브로커 → Consumer 유량 제어

Kafka max.poll.records:
  "Consumer가 poll() 한 번에 가져오는 최대 메시지 수"
  Consumer가 직접 가져오므로 브로커 차단 없음
  
  역할: Consumer가 한 번에 처리할 배치 크기 결정

=== 트레이드오프 비교 ===

                   | RabbitMQ Prefetch | Kafka max.poll.records
───────────────────┼───────────────────┼───────────────────────────
증가 시           | 처리량 ↑, 분배 불균형 | 배치 크기 ↑, 레이턴시 ↑
감소 시           | 분배 공정, 처리량 ↓  | 작은 배치, 레이턴시 ↓
주체              | 브로커가 Push 제어  | Consumer가 Pull 제어

=== 공정 분배 비교 ===

RabbitMQ Prefetch=1:
  Worker A와 B 중 B가 느릴 때:
  A: 처리 완료 → Prefetch 슬롯 확보 → 다음 메시지 수신
  B: 처리 중 → Prefetch 슬롯 점유 → 다음 메시지 대기
  → A가 더 많이 처리 (자연스러운 부하 분산)

Kafka Partition 할당:
  Partition 6개, Consumer 2개: 각 3개씩 고정 할당
  Consumer A가 빨라도 Consumer B의 Partition을 도울 수 없음
  → Partition 수 = 최대 Consumer 수 (분산 단위 고정)
  
  공정 분배를 위해서는 Partition 수 = Consumer 수로 맞추거나
  Consumer 수를 Partition 수 이하로 유지
```

### 4. 메시지 크기별 성능

```
=== 최적 메시지 크기 ===

RabbitMQ:
  최적: 1KB~64KB
  작은 메시지 (< 100B): 오버헤드 비율 높음
  큰 메시지 (> 1MB): Queue 메모리 압박 (Lazy Queue 필요)
  → 큰 메시지: 스토리지(S3 등)에 저장 후 URL만 메시지로

Kafka:
  최적: 1KB~1MB
  작은 메시지 (< 100B): 배치 처리로 오버헤드 극복 가능
  큰 메시지 (> 1MB): max.message.bytes 설정 필요
  배치 처리가 작은 메시지 오버헤드를 상쇄
  → 작은 메시지 대량 처리에서 Kafka 유리

=== 압축 효과 ===

Kafka 압축 (snappy/lz4/gzip):
  JSON 이벤트 (1KB) → snappy 압축 후 300~500B
  배치 메시지 함께 압축 → 더 높은 압축률
  CPU 비용 < 네트워크 절약 (대부분 경우)
  → 대용량 처리 시 압축 권장

RabbitMQ 압축:
  메시지별 개별 압축
  작은 메시지는 압축 오버헤드 > 절약
  1KB 이상 메시지에서 효과적
```

---

## 💻 실전 실험

### 실험 1: 레이턴시 측정 코드

```java
// RabbitMQ 레이턴시 측정
@Component
public class LatencyMeasurement {

  @Autowired private RabbitTemplate rabbitTemplate;
  
  public void measureRabbitMQLatency(int samples) {
    List<Long> latencies = new ArrayList<>();
    
    for (int i = 0; i < samples; i++) {
      long startMs = System.currentTimeMillis();
      
      // 동기 전송 + 즉시 수신 (RPC 패턴으로 레이턴시 측정)
      rabbitTemplate.convertSendAndReceive(
        "latency.exchange", "ping", "ping");
      
      latencies.add(System.currentTimeMillis() - startMs);
    }
    
    double avg = latencies.stream().mapToLong(l -> l).average().orElse(0);
    long p99 = latencies.stream().sorted().skip((long)(samples * 0.99)).findFirst().orElse(0);
    
    System.out.printf("RabbitMQ 레이턴시 - 평균: %.1fms, P99: %dms%n", avg, p99);
  }
}

// Kafka 레이턴시 측정 (최소 설정)
Properties props = new Properties();
props.put("bootstrap.servers", "kafka:9092");
props.put("linger.ms", "0");        // 즉시 전송
props.put("acks", "1");             // Leader만
props.put("fetch.min.bytes", "1");  // Consumer: 즉시 반환
props.put("fetch.max.wait.ms", "0");
```

### 실험 2: 처리량 벤치마크 비교 기준

```
공정한 비교를 위한 동일 조건:
  메시지 크기: 1KB
  내구성: 둘 다 디스크 저장 + 복제
  신뢰성: 둘 다 Producer Confirm/acks=all

RabbitMQ 설정:
  durable Queue + PERSISTENT + Publisher Confirm(비동기)
  Quorum Queue (3노드 복제)

Kafka 설정:
  acks=all + replication-factor=3 + min.insync.replicas=2

이 조건에서:
  RabbitMQ: ~20,000~50,000 msg/sec
  Kafka: ~200,000~500,000 msg/sec (파티션 30개)

결론: 동일 내구성 조건에서 Kafka 처리량이 5~10배 높음
     단, 레이턴시는 RabbitMQ 2~5ms vs Kafka 10~50ms
```

---

## 📊 성능 특성 비교표

```
특성                      | RabbitMQ            | Kafka
─────────────────────────┼─────────────────────┼──────────────────────
단일 메시지 레이턴시       | 1~5ms               | 10~50ms
레이턴시 최소화 가능 수준   | 1~2ms (TRANSIENT)   | 5~10ms (최적화 시)
처리량 (단일 노드)         | 수만~수십만/sec      | 수백만/sec
처리량 확장               | Queue 수 증가        | Partition 수 증가
최적 메시지 크기           | 1KB~64KB            | 1KB~1MB
소비 방식                 | Push (즉시)          | Pull (배치)
유량 제어                 | Prefetch Count       | max.poll.records
배치 처리                 | 배치 소비(선택적)    | 기본 배치 Pull
압축                      | 선택적 (큰 메시지)   | 기본 지원 (권장)
디스크 I/O 패턴            | 혼합 (순차+랜덤)     | 순차 append
```

---

## ⚖️ 트레이드오프

```
레이턴시 vs 처리량 (두 시스템 모두):

낮은 레이턴시:
  RabbitMQ: TRANSIENT + autoAck → 2ms, 처리량 제한
  Kafka: linger.ms=0 + fetch.min=1 → 10ms, 처리량 감소

높은 처리량:
  RabbitMQ: 비동기 Confirm + 배치 소비 → 처리량 ↑, 레이턴시 약간 ↑
  Kafka: linger.ms=10 + 배치 + 압축 → 처리량 ↑, 레이턴시 ↑

RabbitMQ vs Kafka 선택:
  레이턴시 < 10ms 요구: RabbitMQ 유리
  처리량 > 10만/sec 요구: Kafka 유리
  둘 다 필요: 두 시스템 병행 사용
```

---

## 📌 핵심 정리

```
성능 특성 핵심:

레이턴시:
  RabbitMQ Push: 1~5ms (메모리, TRANSIENT)
  Kafka Pull: 10~50ms (기본), 5~10ms (최적화)
  → 낮은 레이턴시: RabbitMQ 유리

처리량:
  RabbitMQ: 수만~수십만 msg/sec
  Kafka: 수백만 msg/sec (순차 쓰기 + 배치 + 압축)
  → 높은 처리량: Kafka 압도적

Prefetch vs max.poll.records:
  RabbitMQ Prefetch: 브로커가 Push 제어
  Kafka max.poll.records: Consumer가 Pull 크기 결정
  둘 다 처리량/레이턴시 트레이드오프

최적 메시지 크기:
  RabbitMQ: 1KB~64KB
  Kafka: 1KB~1MB (배치로 작은 메시지 오버헤드 극복)

핵심 판단:
  레이턴시 우선 → RabbitMQ
  처리량 우선 → Kafka
  재처리 필요 → Kafka (처리량 무관)
```

---

## 🤔 생각해볼 문제

**Q1.** RabbitMQ에서 TRANSIENT + autoAck=true로 측정한 처리량과, Kafka에서 acks=all + 수동 커밋으로 측정한 처리량을 비교하면 어떤 문제가 있는가?

<details>
<summary>해설 보기</summary>

**내구성 보장 수준이 다른 불공정한 비교입니다.**

RabbitMQ TRANSIENT + autoAck=true:
- 재시작 시 메시지 소실
- 처리 중 장애 시 메시지 유실
- 실제로는 신뢰성 보장 없음

Kafka acks=all + 수동 커밋:
- ISR 전체 복제 확인 후 Producer Ack
- 처리 완료 후 오프셋 커밋
- 강한 신뢰성 보장

공정한 비교를 위한 동일 조건:
- **RabbitMQ**: durable Queue + PERSISTENT + 비동기 Publisher Confirm + Quorum Queue
- **Kafka**: acks=all + replication-factor=3 + min.insync.replicas=2 + 수동 커밋

이 조건에서는 Kafka가 처리량 면에서 우위이지만, RabbitMQ도 초당 수만 건을 처리하므로 많은 시스템에서 충분합니다.

</details>

---

**Q2.** Kafka에서 파티션 수를 늘리면 처리량이 선형으로 증가한다고 한다. 파티션을 100개로 설정하면 처리량이 10파티션보다 10배 빠른가?

<details>
<summary>해설 보기</summary>

**이론적으로는 맞지만 실제로는 한계가 있습니다.**

파티션 증가에 따른 처리량 증가:
- 파티션 수 ≤ CPU 코어 수: 거의 선형 증가
- 파티션 수 > 브로커 수: 오버헤드 발생
- 파티션 수 >> Consumer 수: Consumer 수가 병목 (파티션당 최대 1개 Consumer)

파티션 100개의 실제 한계:
1. **Controller 오버헤드**: Kafka Controller가 파티션 상태를 추적 → 파티션 수 증가 시 메타데이터 오버헤드
2. **파일 디스크립터**: 파티션당 세그먼트 파일 → 100파티션 × replication-factor=3 = 300개 파일
3. **Consumer 수 제한**: 파티션 100개를 처리하려면 Consumer Group에 100개 Consumer 필요
4. **Rebalancing 시간**: Consumer 추가/제거 시 100개 파티션 재할당 → 수십 초 중단

**실용적 권장**:
- 파티션 수 = 예상 Consumer 최대 수 × 1.5~2배
- 소규모 서비스: 3~10 파티션
- 대규모 서비스: 30~100 파티션
- 파티션을 나중에 늘리면 재배치 비용 발생 → 처음부터 여유있게 설정

</details>

---

**Q3.** "우리 서비스는 초당 1만 건을 처리하면 충분하다. RabbitMQ와 Kafka 중 어떤 것이 성능 면에서 더 적합한가?"

<details>
<summary>해설 보기</summary>

**성능 측면에서는 두 시스템 모두 충분합니다.** 초당 1만 건은 RabbitMQ, Kafka 모두 여유 있게 처리 가능한 수준입니다.

이 경우 선택은 성능이 아닌 **다른 요소**로 결정해야 합니다:

1. **재처리 필요성**: 메시지 재처리나 과거 이벤트 소급이 필요하면 → Kafka
2. **라우팅 복잡도**: Exchange 패턴이 필요하면 → RabbitMQ
3. **운영 복잡성**: 단순한 운영을 원하면 → RabbitMQ (Kafka는 Zookeeper/KRaft 등 추가 운영 부담)
4. **팀 익숙도**: 기존 팀의 경험 → 학습 비용 고려
5. **생태계 통합**: 데이터 파이프라인이 Kafka 기반이면 → Kafka 일관성

**결론**: 초당 1만 건이면 성능이 선택 기준이 되지 않습니다. 비즈니스 요구사항(재처리, 라우팅)과 운영 편의성으로 결정하세요.

</details>

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[이전: 사용 사례 비교 ⬅️](./02-use-case-comparison.md)** | **[다음: 함께 쓰는 경우 ➡️](./04-using-both.md)**

</div>
