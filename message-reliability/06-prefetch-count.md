# Prefetch Count(QoS) 완전 분해 — Consumer 처리량과 공정 분배

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Prefetch=1이면 정확히 어떻게 동작하고, 처리량에 어떤 영향이 있는가?
- Prefetch를 크게 설정하면 불균등 분배가 발생하는 이유는 무엇인가?
- Prefetch Count와 Consumer Utilisation은 어떤 관계인가?
- 최적 Prefetch Count를 어떻게 산정하는가?
- Channel-level QoS와 Consumer-level QoS의 차이는?

---

## 🔍 왜 이 개념이 실무에서 중요한가

Prefetch=1로 두면 Consumer가 메시지 하나를 처리하는 동안 다음 메시지를 받지 못한다. 처리 시간이 100ms이면 초당 최대 10개. 처리 시간이 짧은 메시지라면 Prefetch=1은 심각한 처리량 병목이다. 반면 Prefetch를 너무 높이면 Consumer A가 100개를 가져가는 동안 Consumer B는 놀고, Consumer A가 느리면 전체 처리가 느려진다. 최적 Prefetch는 처리 속도, Consumer 수, 메시지 크기를 고려한 수치다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: Prefetch 설정 없이 운영 (기본값 미확인)

  # Spring AMQP 기본 prefetch: 250 (버전에 따라 다름)
  # 이 상태로 Consumer 10개 운영

  결과:
    Consumer A: 250개 획득 → 느린 처리 (처리 중)
    Consumer B: 250개 획득
    Consumer C~J: 각 250개 획득
    → 총 2500개 Unacked 상태 (전부 메모리에서 처리 중)
    → 발행 속도가 빠르면 Queue는 비어있는데
    → 특정 Consumer가 느리면 2500개가 그 Consumer에 묶여 있음

실수 2: Prefetch=1로 고처리량 서비스 운영

  spring:
    rabbitmq:
      listener:
        simple:
          prefetch: 1

  메시지 처리 시간: 10ms
  Prefetch=1이면:
    처리 → Ack → 다음 메시지 수신 → 처리 → Ack...
    10ms 처리 + 네트워크 왕복 2~5ms = 12~15ms/메시지
    초당 약 65~80개
    
  Prefetch=100이면:
    100개를 미리 받아두고 연속 처리
    네트워크 대기 없이 연속 처리 → 초당 약 90~100개
    (처리 속도 20~50% 향상)

실수 3: Channel-level vs Consumer-level 혼동

  channel.basicQos(10, false);  // Consumer-level (false)
  channel.basicQos(100, true);  // Channel-level (true)
  
  두 설정이 동시에 적용될 때 어떻게 동작하는지 모름
  → 의도치 않은 동작
```

---

## ✨ 올바른 접근 (After — 처리 시간 기반 Prefetch 산정)

```
Prefetch 산정 공식 (참고치):

Prefetch = (처리 시간 ms / 네트워크 왕복 ms) × Consumer 수

예시:
  처리 시간: 50ms
  네트워크 RTT: 5ms
  Consumer 인스턴스: 5개

  Prefetch = (50 / 5) × 1 = 10  (인스턴스당)
  → Prefetch=10으로 시작, 실제 Consumer Utilisation 측정 후 조정

Spring AMQP 설정:
  spring:
    rabbitmq:
      listener:
        simple:
          prefetch: 10        # Consumer 인스턴스당
          concurrency: 3      # Consumer 스레드 수 (인스턴스 내)
          max-concurrency: 10

일반 가이드라인:
  처리 시간 < 10ms:  Prefetch=50~200 (네트워크 비용이 지배적)
  처리 시간 10~100ms: Prefetch=5~30
  처리 시간 > 100ms:  Prefetch=1~10
  처리 시간 > 1s:     Prefetch=1~3
  DB/외부 API 의존:   Prefetch=1~5 (느린 메시지가 슬롯 점유)
```

---

## 🔬 내부 동작 원리

### 1. Prefetch Count의 정확한 동작

```
=== basicQos(prefetchCount) ===

AMQP 명령:
  channel.basicQos(10);
  → "한 번에 최대 10개의 Unacked 메시지만 허용"

동작:
  Consumer가 basicConsume 등록
  Broker: 메시지를 Consumer에게 전달
  
  [Unacked 수 < prefetchCount]:
    → 새 메시지 즉시 전달
  
  [Unacked 수 == prefetchCount]:
    → 새 메시지 전달 차단 (Consumer 채움)
    → Consumer가 Ack를 보낼 때까지 대기
    → Ack 수신 → 슬롯 확보 → 새 메시지 전달

=== Prefetch=1 상세 ===

Timeline (처리 시간 50ms, RTT 5ms):

  T=0:    Broker → Consumer: msg1 전달
  T=0:    Consumer: msg1 처리 시작 (Unacked=1, 슬롯 0개 남음)
  T=50:   Consumer: msg1 처리 완료 → basicAck 전송
  T=55:   Broker: Ack 수신 → msg2 전달 (RTT 5ms)
  T=55:   Consumer: msg2 처리 시작
  T=105:  Consumer: msg2 처리 완료 → basicAck 전송
  ...

사이클: 50ms 처리 + 5ms 네트워크 = 55ms/메시지
처리량: 1000/55 ≈ 18 msg/sec (단일 Consumer)

=== Prefetch=10 상세 ===

Timeline:

  T=0:    Broker → Consumer: msg1~msg10 전달 (한 번에 10개)
  T=0:    Consumer: msg1 처리 시작, msg2~10은 로컬 버퍼
  T=50:   Consumer: msg1 완료, msg2 시작 → Ack 전송
  T=50:   Broker: Ack 수신 → msg11 전달 (슬롯 1개 회복)
  T=100:  Consumer: msg2 완료, msg3 시작
  ...

네트워크 대기 없이 연속 처리
처리량: 1000/50 ≈ 20 msg/sec (Prefetch=1 대비 ~11% 향상)
(RTT 5ms 절약 효과가 50ms 처리에서는 상대적으로 작음)

RTT가 처리 시간과 비슷할 때 효과 극대화:
  처리 시간 5ms, RTT 5ms:
    Prefetch=1: 10ms/msg → 100 msg/sec
    Prefetch=10: ~5ms/msg → ~200 msg/sec (2배 향상!)
```

### 2. 불균등 분배 — Prefetch가 높을 때의 문제

```
=== 시나리오: Consumer 2개, Prefetch=100, 메시지 150개 ===

Consumer A: 빠름 (처리 시간 10ms)
Consumer B: 느림 (처리 시간 100ms)

T=0:
  Consumer A: msg1~100 획득 (Prefetch=100)
  Consumer B: msg101~150 획득 (Prefetch=100, 50개만 있음)

Consumer A (10ms/msg):
  T=0~1000ms: msg1~100 처리 완료 (1초에 100개 처리)
  → 큐 비어있음 (Consumer B가 50개 가지고 있음)
  → Consumer A는 놀고 있음

Consumer B (100ms/msg):
  T=0~5000ms: msg101~150 처리 (5초에 50개 처리)

결과:
  A: 1초에 끝남 → 4초 동안 놀음
  B: 5초 소요
  전체 완료: 5초 (A가 B를 도울 수 없음)

Prefetch=1이면:
  A와 B가 번갈아가며 메시지 수신
  A: 빠른 처리 후 즉시 다음 메시지
  B: 느리게 처리하는 동안 A가 더 많이 처리
  전체 완료: 더 빠름 (A가 B를 자연스럽게 보완)

=== Fair Dispatch ===

Prefetch=1이 "공정한 분배(Fair Dispatch)"를 보장하는 이유:
  Consumer가 현재 처리 중인 메시지 수 = 최대 1개
  처리 완료 → 슬롯 확보 → 즉시 다음 메시지 수신
  
  빠른 Consumer가 더 많이 처리, 느린 Consumer가 덜 처리
  → 각 Consumer의 처리 속도에 비례한 자연스러운 부하 분산

Prefetch>1의 분배:
  큰 Prefetch → 빠른 Consumer에게 메시지가 치우침
  (처리 속도에 비례하지 않고, 먼저 요청하는 쪽에 치우침)
```

### 3. Channel-level QoS vs Consumer-level QoS

```
=== basicQos 두 번째 파라미터 ===

channel.basicQos(count, global);

  global=false (기본, Consumer-level):
    "이 Channel의 각 Consumer마다 count개 제한"
    Channel에 Consumer 3개: 각각 count개 → 총 3×count개 Unacked 가능

  global=true (Channel-level):
    "이 Channel 전체에서 count개 제한"
    Channel에 Consumer 3개라도: 전체 합해서 count개 Unacked 가능

실무 사용:
  대부분 Consumer-level (global=false) 사용
  Channel-level은 한 Channel에 여러 Consumer 쓸 때 전체 제한 목적

=== Spring AMQP의 Prefetch 적용 방식 ===

SimpleMessageListenerContainer:
  각 Consumer Thread마다 Channel 하나씩 사용
  prefetch 설정 → 각 Channel의 basicQos 설정

  concurrency=5, prefetch=10이면:
    Consumer Thread 5개 × Channel 5개
    각 Channel: basicQos(10)
    총 Unacked 가능: 50개

  @RabbitListener(queues = "order.queue",
    concurrency = "3-10")  // 최소 3, 최대 10 스레드
  공식:
    총 Prefetch = Prefetch × Concurrency
```

### 4. Consumer Utilisation — 병목 진단

```
=== Consumer Utilisation이란 ===

Management UI에서 확인 가능한 지표:
  Queues → order.queue → Consumer Utilisation

수식:
  Utilisation = 처리 시간 / (처리 시간 + 대기 시간)

  Utilisation = 100%: Consumer가 항상 처리 중 (Prefetch 부족 or 처리량 최대)
  Utilisation = 50%: 절반은 처리, 절반은 새 메시지 대기
  Utilisation = 0%:  Queue 비어있음 (Consumer 놀고 있음)

=== 진단 활용 ===

시나리오 1: Utilisation < 50%
  Consumer가 절반 이상 놀고 있음
  원인: Queue에 메시지 없거나, Prefetch 부족 (네트워크 대기)
  
  처리 시간이 짧고 Prefetch가 낮으면:
    → Prefetch 증가로 해결

시나리오 2: Utilisation ≈ 100%
  Consumer가 항상 바쁨
  원인: 메시지 유입 속도 > Consumer 처리 속도
  
  해결:
    Consumer 인스턴스 추가 (Scale Out)
    또는 처리 최적화

시나리오 3: Utilisation = 100% + Queue 적체
  Consumer가 따라가지 못함
  긴급 Scale Out 필요

=== Prefetch 최적화 실험 ===

1. Prefetch=1로 시작, Utilisation 측정
2. 낮으면 Prefetch 2배 증가
3. Utilisation이 90%+이 될 때까지 반복
4. 처리량(msg/sec) 측정, Prefetch 증가해도 처리량 안 오르면 적정값 도달
```

---

## 💻 실전 실험

### 실험 1: Prefetch=1 vs Prefetch=100 처리량 비교

```bash
# Queue 생성 및 메시지 5000개 발행
rabbitmqadmin declare queue name=throughput.test durable=true
for i in $(seq 1 5000); do
  rabbitmqadmin publish exchange="" routing_key=throughput.test \
    payload="msg$i" > /dev/null
done

echo "5000개 발행 완료"
rabbitmqctl list_queues name messages
# throughput.test  5000
```

```java
// Prefetch=1 측정
@Configuration
class PrefetchOneConfig {
  @Bean
  public SimpleRabbitListenerContainerFactory prefetch1Factory(
      ConnectionFactory connectionFactory) {
    SimpleRabbitListenerContainerFactory factory = new SimpleRabbitListenerContainerFactory();
    factory.setConnectionFactory(connectionFactory);
    factory.setPrefetchCount(1);
    factory.setAcknowledgeMode(AcknowledgeMode.MANUAL);
    return factory;
  }
}

@RabbitListener(queues = "throughput.test", containerFactory = "prefetch1Factory")
public void handlePrefetch1(Message msg, Channel ch,
    @Header(AmqpHeaders.DELIVERY_TAG) long tag) throws Exception {
  // 처리 시간 시뮬레이션 (10ms)
  Thread.sleep(10);
  ch.basicAck(tag, false);
}

// Management UI에서 처리량 확인: ~80-90 msg/sec

// Prefetch=100 측정
// factory.setPrefetchCount(100);
// 처리량 확인: ~95-100 msg/sec
```

### 실험 2: Management UI에서 Consumer Utilisation 확인

```
Management UI → Queues → [Queue 이름] → Consumers 탭

Consumers 섹션:
  ┌──────────────────────────┬────────────┬──────────────────┐
  │ Consumer Tag             │ Prefetch   │ Utilisation      │
  ├──────────────────────────┼────────────┼──────────────────┤
  │ amq.ctag-Xfj1...         │ 1          │ 98.2%            │
  │ amq.ctag-Yrk2...         │ 1          │ 97.8%            │
  └──────────────────────────┴────────────┴──────────────────┘

Utilisation 98%+ → Consumer가 항상 바쁨
→ Prefetch 늘려봐도 효과 없으면 처리 자체가 병목
→ Consumer Scale Out 필요
```

### 실험 3: 불균등 분배 재현

```bash
# 큰 Prefetch로 불균등 분배 재현
rabbitmqadmin declare queue name=fair.test durable=true
for i in $(seq 1 100); do
  rabbitmqadmin publish exchange="" routing_key=fair.test payload="msg$i"
done

# Consumer A (빠름): prefetch=50, 처리 시간 10ms
# Consumer B (느림): prefetch=50, 처리 시간 500ms

# 각 Consumer가 받은 메시지 수 모니터링:
rabbitmqctl list_consumers
# 각 Consumer의 prefetch_count, messages_unacknowledged 확인
```

---

## 📊 Kafka vs RabbitMQ 비교

```
=== Consumer 처리량 제어 비교 ===

RabbitMQ Prefetch:
  basicQos(prefetchCount)
  "브로커가 이 Consumer에게 최대 N개까지 미리 전달"
  Consumer 처리 완료(Ack) 전까지 추가 전달 차단
  → 브로커 레벨에서 유량 제어

Kafka max.poll.records:
  poll() 한 번에 최대 가져올 레코드 수
  Consumer가 배치로 처리
  enable.auto.commit=false → 처리 후 commitSync()
  → Consumer 레벨에서 배치 크기 제어

비교:
  RabbitMQ: 브로커가 "얼마나 보낼지" 제어
  Kafka: Consumer가 "얼마나 가져올지" 제어

=== 분산 처리 비교 ===

RabbitMQ:
  Consumer 인스턴스 추가 → 큐를 공유하며 경쟁 소비
  Prefetch는 인스턴스당 설정
  공정 분배: Prefetch=1이면 처리 속도에 비례

Kafka:
  Consumer 인스턴스 추가 → Partition 재분배 (Rebalancing)
  각 Partition은 하나의 Consumer가 독점
  공정 분배: Partition 수 = Consumer 수일 때 균등
```

---

## ⚖️ 트레이드오프

```
Prefetch Count 설정별 트레이드오프:

Prefetch=1:
  ✅ 공정한 분배 (처리 속도 비례)
  ✅ 처리 실패 시 영향 최소 (1개만 Unacked)
  ❌ 처리량 낮음 (네트워크 대기 시간 상대적으로 큼)
  → 처리 시간 길고 공정 분배 중요할 때

Prefetch 높음 (50~250):
  ✅ 처리량 높음 (네트워크 대기 시간 숨겨짐)
  ❌ 불균등 분배 (빠른 Consumer에게 메시지 쏠림)
  ❌ Consumer 장애 시 많은 메시지가 Unacked
  → 처리 시간 짧고 처리량이 최우선일 때

최적 Prefetch:
  목표: Consumer Utilisation 90%+ 유지
  산정: 처리 시간 / 네트워크 RTT 로 시작
  측정: Utilisation 모니터링으로 조정
```

---

## 📌 핵심 정리

```
Prefetch Count 핵심:

정의:
  Consumer가 동시에 갖고 있을 수 있는 최대 Unacked 메시지 수
  basicQos(prefetchCount) 설정

Prefetch=1:
  처리 중인 메시지 1개만 허용
  공정 분배 보장 (처리 속도에 비례)
  처리량은 상대적으로 낮음

Prefetch 높음:
  처리량 향상 (네트워크 대기 감소)
  불균등 분배 가능성
  Consumer 장애 시 많은 Unacked

최적값 산정:
  (처리 시간 ms) / (네트워크 RTT ms)로 시작
  Consumer Utilisation 90%+ 목표
  실측 후 조정

Spring AMQP:
  spring.rabbitmq.listener.simple.prefetch 설정
  concurrency: 스레드 수 (총 Unacked = prefetch × concurrency)

Consumer Utilisation:
  Management UI에서 확인
  낮으면 Prefetch 증가 검토
  높으면 Consumer Scale Out 검토
```

---

## 🤔 생각해볼 문제

**Q1.** Prefetch=10, concurrency=5로 설정된 Spring AMQP Consumer가 있다. 동시에 처리 가능한 최대 메시지 수와 Unacked 상태가 될 수 있는 최대 메시지 수는 각각 얼마인가?

<details>
<summary>해설 보기</summary>

**최대 Unacked 메시지 수: 50개** (Prefetch 10 × Concurrency 5)

**동시 처리 가능 메시지 수**: 동시에 실행 중인 스레드 수 = Concurrency 5개

메커니즘:
- concurrency=5 → Consumer 스레드 5개, 각각 별도 Channel
- 각 Channel에 basicQos(10) 적용
- 각 스레드가 최대 10개 Unacked 보유 가능
- 5개 스레드 × 10개 = 50개 Unacked

실제 처리:
- 각 스레드는 한 번에 하나씩 처리 (단일 스레드 내)
- 하지만 나머지 9개는 로컬 버퍼에 대기
- 5개 스레드 × 1개 처리 중 = 5개 동시 처리
- 5개 스레드 × 9개 대기 = 45개 버퍼에 대기

Consumer 장애 시 최대 50개 메시지가 Queue로 복귀합니다.

</details>

---

**Q2.** 메시지 처리 시간이 균일하지 않다. 어떤 메시지는 5ms, 어떤 메시지는 500ms 걸린다. 이 상황에서 최적 Prefetch는 무엇인가?

<details>
<summary>해설 보기</summary>

처리 시간이 불균일할 때는 **Prefetch를 낮게 설정**하는 것이 권장됩니다.

이유:
- Prefetch가 높으면 Consumer가 많은 메시지를 미리 가져감
- 500ms 메시지 10개를 가져갔다면 → 5초 동안 슬롯 점유
- 다른 빠른 메시지들이 이 Consumer를 우회하지 못함

권장 전략:
1. **Prefetch를 낮게 설정 (1~5)**: Consumer 간 자연스러운 부하 분산
2. **메시지 유형별 Queue 분리**: 빠른 메시지 Queue와 느린 메시지 Queue를 별도 운영
   - `order.fast.queue` (Prefetch=50): 짧은 처리
   - `order.slow.queue` (Prefetch=3): 긴 처리

다른 접근:
- 처리 시간 분산이 크면 Consumer 자체의 처리 로직을 균일하게 만드는 것도 고려
- 예: 복잡한 처리는 비동기화 (메시지 수신 → 처리 작업 큐에 등록 → 빠른 Ack)

</details>

---

**Q3.** 프로덕션에서 Consumer Utilisation이 갑자기 30%로 떨어졌다. 어떤 원인을 먼저 확인해야 하는가?

<details>
<summary>해설 보기</summary>

**Utilisation 30%** = Consumer가 70%의 시간을 대기 중. 원인 후보:

**1. Queue가 비어있음** (가장 흔한 원인)
- Publisher 속도 저하, 배포 중단 등
- 확인: `rabbitmqctl list_queues name messages` → 메시지 수 0

**2. Prefetch 부족** (두 번째 흔한 원인)
- Consumer가 Ack 후 새 메시지 수신 대기 시간이 많음
- 확인: Prefetch 값과 네트워크 RTT 비교
- 해결: Prefetch 증가

**3. Consumer 수가 많음** (초과 Scale Out)
- 메시지 유입보다 Consumer가 많음
- 확인: `list_consumers` 수 vs 메시지 발행 속도
- 해결: Consumer 인스턴스 줄이기

**4. 처리 로직 내 외부 I/O 대기** (비동기 전환 고려)
- DB 쿼리, 외부 API 호출이 대부분의 처리 시간
- 실제 CPU 처리는 적고 대기가 많음
- 해결: 처리 로직 비동기화 또는 Prefetch 증가

**확인 순서**: Queue 메시지 수 → Prefetch 설정 → Consumer 수 → 처리 로직 프로파일링

</details>

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[이전: 재시도 전략 설계 ⬅️](./05-retry-strategy.md)** | **[다음: 트랜잭션 vs Publisher Confirm ➡️](./07-transaction-vs-confirm.md)**

</div>
