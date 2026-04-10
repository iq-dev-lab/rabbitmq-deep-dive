# 메시지 유실의 세 지점 — 어디서, 왜, 어떻게 사라지는가

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 메시지가 유실될 수 있는 세 가지 지점은 정확히 어디인가?
- 각 지점에서 메시지가 사라지는 정확한 시점과 조건은 무엇인가?
- 세 지점을 모두 방어하지 않으면 어떤 시나리오에서 메시지가 사라지는가?
- 중요도에 따라 어떤 지점부터 방어해야 하는가?
- 메시지 유실을 탐지하는 방법은 무엇인가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

"RabbitMQ를 쓰면 메시지가 보장된다"는 착각이 있다. RabbitMQ는 기본 설정만으로는 메시지를 보장하지 않는다. Publisher Confirm을 켜지 않으면 발행 실패를 모른다. autoAck=true이면 Consumer가 받는 순간 삭제된다. durable=false이면 재시작 시 메시지가 증발한다. 유실 지점을 모르면 어떤 설정을 해야 하는지도 모른다. 이 문서는 "어디서, 왜, 어떻게"를 먼저 이해하고, 이후 문서에서 각 방어책을 깊이 다룬다.

---

## 😱 흔한 실수 (Before — 유실 지점을 모를 때)

```
실수: "기본 설정으로 쓰면 안전할 것 같다"

기본 설정의 실제 동작:
  Publisher Confirm: 비활성화 (기본)
    → 브로커가 메시지를 받았는지 확인 안 함
    → 네트워크 오류로 발행 실패해도 Producer는 모름

  autoAck: true (일부 프레임워크 기본)
    → Consumer가 메시지를 받는 순간 Queue에서 삭제
    → Consumer 처리 중 장애나도 메시지 복구 불가

  Queue durable: false (명시 안 하면 기본값 false인 경우)
    → 브로커 재시작 시 Queue와 메시지 전부 삭제

  DeliveryMode: TRANSIENT (기본)
    → Queue가 durable이어도 메시지 자체는 메모리에만 저장
    → 브로커 재시작 시 메시지 소실

결과:
  개발 환경에서 테스트 → 정상 동작
  프로덕션 배포 후 일부 메시지 유실 → 원인 파악 어려움
  "RabbitMQ가 이상하다" → 실제 원인은 설정 미비
```

---

## ✨ 올바른 접근 (After — 3지점 방어 체계)

```
메시지 유실 방지 완전 설정:

지점 1 (Publisher → Broker):
  Publisher Confirm 활성화
  channel.confirmSelect() + ConfirmCallback
  → "브로커가 받았다"는 확인을 받아야 발행 완료로 처리

지점 2 (Broker 저장):
  Queue durable=true + DeliveryMode=PERSISTENT
  → 재시작 후에도 메시지 보존

지점 3 (Consumer 처리):
  autoAck=false + 수동 basicAck
  → 처리 완료 후 Ack → 처리 전 장애 시 재전달

모두 방어 시 보장:
  Publisher가 Ack 받음 → 브로커 수신 확인
  브로커 재시작 → 메시지 디스크에 보존
  Consumer 장애 → 메시지 다른 Consumer로 재전달
```

---

## 🔬 내부 동작 원리

### 1. 지점 1 — Publisher → Broker 전송 중 유실

```
=== 발생 시나리오 ===

시나리오 A: 네트워크 오류
  Publisher: basicPublish() 호출
  TCP: 패킷 전송 중 네트워크 단절
  Publisher: "전송 완료" 로 착각 (TCP ACK ≠ AMQP Confirm)
  Broker: 메시지 수신 못 함
  
  결과: 메시지 유실, Publisher는 모름

시나리오 B: Exchange 없음 (mandatory=false)
  Publisher: exchange="존재않는exchange" 발행
  RabbitMQ: Exchange 없음 → 연결/채널 에러 또는
            Exchange는 있지만 Binding 없음 → 조용히 폐기

시나리오 C: Broker 과부하 (Flow Control)
  Broker 메모리 임계값 초과 → Flow Control 발동
  Publisher: 메시지 전송 차단 (timeout까지 대기)
  Timeout 설정 없으면 무한 대기
  Timeout 있으면 예외 → 재처리 필요

=== TCP ACK vs AMQP Confirm ===

TCP ACK:
  "패킷이 상대방 TCP 스택에 도달했다"는 확인
  ≠ "RabbitMQ가 메시지를 받아 처리했다"는 의미

AMQP Publisher Confirm (Basic.Ack):
  "RabbitMQ가 메시지를 받아 Queue에 저장했다"는 확인
  또는 (durable Queue + PERSISTENT) 시 "디스크에 기록했다"
  
  이것이 Confirm 없이는 유실을 탐지 못하는 이유

=== Publisher Confirm이 없을 때의 위험 ===

  Time ──────────────────────────────────────────────►
  
  Publisher: [send msg] → [send msg] → [send msg] ...
  Network:              ✅             ✅             ❌ (단절)
  Broker:   [recv msg]    [recv msg]     [? 못 받음]
  
  Publisher는 마지막 메시지가 유실됐는지 알 방법 없음
  Confirm 없으면 "전부 성공"으로 인식
```

### 2. 지점 2 — Broker 저장 중 유실

```
=== 발생 시나리오 ===

시나리오 A: Broker 재시작 (durable=false)
  Queue: durable=false
  Broker 재시작 → Queue 정의 삭제 + 메시지 전부 소실
  재시작 후 Queue 자체가 없음

시나리오 B: 메시지 비영속 (DeliveryMode=TRANSIENT)
  Queue: durable=true (Queue 정의는 유지)
  메시지: DeliveryMode=TRANSIENT (메모리에만 저장)
  Broker 재시작 → Queue 정의는 살아있지만 메시지 소실

시나리오 C: 메모리 부족 (OOM)
  Broker 메모리 초과 → Erlang VM OOM Kill
  PERSISTENT 메시지: 디스크에 있으면 복구 가능
  fsync 전 OOM: 일부 메시지 유실 가능

=== durable vs DeliveryMode 조합 ===

Queue durable | DeliveryMode  | 재시작 후 메시지 상태
──────────────┼───────────────┼──────────────────────
false         | any           | Queue 삭제 → 소실
true          | TRANSIENT     | Queue 존재, 메시지 소실
true          | PERSISTENT    | Queue + 메시지 보존 ✅

→ 반드시 두 가지 모두 설정해야 함

=== PERSISTENT의 실제 동작 ===

DeliveryMode=PERSISTENT 설정 시:
  메시지 수신 → 디스크에 쓰기 시도
  단, 즉시 fsync하지 않음 → OS 버퍼에 쓰기
  
  Lazy Queue 아닌 경우:
    메모리에도 보관 (읽기 속도 위해)
    주기적으로 fsync (또는 Publisher Confirm 시)
  
  Quorum Queue:
    과반수 노드에 WAL(Write-Ahead Log) 기록 후 Confirm
    가장 강한 내구성 보장
```

### 3. 지점 3 — Consumer 처리 중 유실

```
=== 발생 시나리오 ===

시나리오 A: autoAck=true (가장 흔한 유실 원인)

  Queue: [msg1][msg2][msg3]
  Consumer: basicConsume(queue, autoAck=true, ...)
  
  RabbitMQ 동작:
    msg1 전달 → 즉시 Queue에서 삭제 (Ack 안 해도)
    Consumer: msg1 처리 중...
    Consumer: 예외 발생 (DB 연결 오류 등)
    
  결과: msg1 처리 실패 + Queue에서 이미 삭제 = 영구 유실

시나리오 B: 수동 Ack인데 잘못된 시점에 Ack

  // 잘못된 패턴
  @RabbitListener(queues = "order.queue")
  public void process(Message message, Channel channel, 
                      @Header(AmqpHeaders.DELIVERY_TAG) long tag) throws Exception {
    channel.basicAck(tag, false);  // 처리 전에 Ack!
    orderService.process(message); // 이후 예외 발생해도 이미 Ack됨
  }
  
  결과: 처리 실패했지만 Ack 완료 → 메시지 삭제 → 유실

시나리오 C: Consumer 프로세스 비정상 종료

  autoAck=false, 수동 Ack 설정
  Consumer: msg1 받음 (Unacked 상태)
  Consumer: 처리 중 OOM Kill
  
  결과:
    Connection 끊김 → Unacked 메시지 자동으로 Ready 상태로 복귀
    → 다른 Consumer에게 재전달 ✅ (이것이 올바른 동작)

=== autoAck=true가 위험한 이유 (상세) ===

AMQP 레벨에서 autoAck=true:
  basicConsume(queue, noAck=true, ...)  ← noAck=true
  
  서버 동작:
    메시지를 Consumer에게 보내는 순간 → 서버 측 삭제
    Consumer가 실제 처리 완료를 기다리지 않음
    
  안전한 autoAck=true 사용 조건:
    처리가 멱등적이고 유실이 허용되는 경우 (로그, 메트릭)
    처리 실패 시 재처리보다 무시가 나은 경우
    처리 자체가 실패할 수 없는 경우 (로컬 메모리 저장)
```

### 4. 메시지 유실 탐지 방법

```
=== 탐지 지표 ===

Publisher 측:
  - Confirm 받지 못한 메시지 카운터
  - 발행 성공률 모니터링
  - Return된 메시지 (mandatory=true 설정 시)

Broker 측:
  Management UI 지표:
  - publish_rate: 발행 속도
  - deliver_rate: 전달 속도
  - ack_rate: Ack 속도
  - dead_letter_rate: DLQ 이동 속도
  
  발행 속도 > Ack 속도: 메시지 쌓임 (정상)
  발행 속도 > deliver_rate: 소비 안 됨 (Queue 적체)

Consumer 측:
  - DLQ 메시지 수 증가 → 처리 실패 메시지 있음
  - Unacked 메시지 수 증가 → Consumer 처리 중 문제
  - Consumer Utilisation 낮음 → Prefetch 문제

=== 메시지 추적 (Message ID 활용) ===

발행 시 고유 ID 부여:
  MessageProperties props = new MessageProperties();
  props.setMessageId(UUID.randomUUID().toString());
  props.setCorrelationId(orderId.toString());

DB에 발행 기록:
  INSERT INTO outbox (message_id, status, created_at) VALUES (?, 'PENDING', NOW())
  발행 성공 → UPDATE outbox SET status='PUBLISHED'
  일정 시간 후 PENDING 상태면 → 재발행 (Outbox Pattern)
```

---

## 💻 실전 실험

### 실험 1: autoAck=true 메시지 유실 재현

```bash
# 실험 환경: autoAck=true Consumer 시뮬레이션
# Queue 생성
rabbitmqadmin declare queue name=test.ack.queue durable=true

# 메시지 5개 발행
for i in $(seq 1 5); do
  rabbitmqadmin publish exchange="" routing_key=test.ack.queue \
    payload="{\"id\":$i}"
done

rabbitmqctl list_queues name messages
# test.ack.queue  5

# autoAck=true로 메시지 가져오기 (즉시 삭제)
rabbitmqadmin get queue=test.ack.queue ackmode=ack_requeue_false count=5
# ack_requeue_false = 받는 즉시 삭제 (autoAck 시뮬레이션)

rabbitmqctl list_queues name messages
# test.ack.queue  0  ← 처리 완료 여부 무관하게 삭제됨
```

### 실험 2: Unacked 메시지 복구 확인

```bash
# 메시지 발행
rabbitmqadmin publish exchange="" routing_key=test.ack.queue \
  payload='{"id":1}'

# autoAck=false로 가져오기 (Unacked 상태)
rabbitmqadmin get queue=test.ack.queue ackmode=ack_requeue_true count=1
# ack_requeue_true = 받되 Ack 안 함 (Unacked 상태 유지)

rabbitmqctl list_queues name messages messages_ready messages_unacknowledged
# test.ack.queue  1  0  1  ← Ready=0, Unacked=1

# Connection 종료 시뮬레이션 (rabbitmqadmin 종료)
# → Unacked 메시지 자동으로 Ready 상태로 복귀

rabbitmqctl list_queues name messages messages_ready messages_unacknowledged
# test.ack.queue  1  1  0  ← Ready=1, Unacked=0 (복구됨)
```

### 실험 3: durable + PERSISTENT 조합 테스트

```bash
# 조합 1: durable=true + PERSISTENT
rabbitmqadmin declare queue name=durable.persistent durable=true
rabbitmqadmin publish exchange="" routing_key=durable.persistent \
  properties='{"delivery_mode":2}' payload="persistent message"

# 조합 2: durable=true + TRANSIENT
rabbitmqadmin declare queue name=durable.transient durable=true
rabbitmqadmin publish exchange="" routing_key=durable.transient \
  properties='{"delivery_mode":1}' payload="transient message"

# RabbitMQ 재시작
docker restart rabbitmq-test

# 재시작 후 확인
rabbitmqctl list_queues name messages
# durable.persistent  1  ← 보존
# durable.transient   0  ← TRANSIENT 메시지 소실 (Queue는 살아있음)
```

---

## 📊 Kafka vs RabbitMQ 비교

```
=== 메시지 유실 방지 방어 체계 비교 ===

지점 1 (Producer → Broker):
  RabbitMQ: Publisher Confirm (acks=true 활성화 필요)
  Kafka:    acks=all (Leader + ISR 전체 확인)
            acks=1 (Leader만) → ISR 복제 전 Leader 장애 시 유실 가능

지점 2 (Broker 저장):
  RabbitMQ: durable=true + PERSISTENT + Quorum Queue
  Kafka:    replication-factor=3 + min.insync.replicas=2
            → 2개 이상 복제 확인 후 Producer에 Ack

지점 3 (Consumer 처리):
  RabbitMQ: autoAck=false + 수동 basicAck
  Kafka:    enable.auto.commit=false + 수동 commitSync/commitAsync
            → 오프셋 커밋 = RabbitMQ의 Ack와 동일한 역할

핵심 차이:
  RabbitMQ Ack: 처리 완료 → 메시지 삭제
  Kafka Commit: 처리 완료 → 오프셋 커밋 (메시지는 보관)
  
  → Kafka는 Commit 후에도 메시지 남음 → 재처리 가능
  → RabbitMQ Ack 후에는 메시지 없음 → 재처리 불가
```

---

## ⚖️ 트레이드오프

```
완전한 메시지 보장 vs 성능:

완전 보장 설정:
  Publisher Confirm + durable Queue + PERSISTENT + autoAck=false
  
  성능 영향:
    Publisher Confirm: 발행당 왕복 1회 (Ack 기다림)
    PERSISTENT: 디스크 fsync 비용
    수동 Ack: 처리 완료 후 Ack 전송 1회
  
  처리량 기준:
    완전 보장: 기본 대비 30~50% 처리량 감소 (측정 필요)
    단, 절대값은 서버 스펙에 따라 다름

각 설정의 독립적 비용:
  Publisher Confirm만: 처리량 ~10% 감소 (비동기 Confirm 시)
  PERSISTENT만: 처리량 ~20% 감소 (fsync 비용)
  수동 Ack만: 처리량 ~5% 감소 (추가 Ack 패킷)

유실 허용 가능한 경우 (완전 보장 불필요):
  실시간 메트릭 수집 (일부 유실 허용)
  로그 이벤트 (중복/유실 허용)
  실시간 알림 (빠른 응답 > 완전 보장)
  
  이런 경우: autoAck=true + TRANSIENT로 최고 처리량 확보
```

---

## 📌 핵심 정리

```
메시지 유실 3지점:

지점 1: Publisher → Broker 전송
  원인: 네트워크 오류, Exchange 없음, Flow Control
  방어: Publisher Confirm (confirmSelect + ConfirmCallback)
  탐지: Confirm 못 받은 메시지 카운터

지점 2: Broker 저장
  원인: durable=false (재시작 시 Queue 삭제),
        DeliveryMode=TRANSIENT (재시작 시 메시지 소실)
  방어: durable=true + DeliveryMode=PERSISTENT (두 가지 모두 필수)
  탐지: 재시작 후 Queue 메시지 수 비교

지점 3: Consumer 처리
  원인: autoAck=true (전달 즉시 삭제),
        처리 전 Ack (잘못된 시점)
  방어: autoAck=false + 처리 완료 후 basicAck
  탐지: Unacked 메시지 급증, DLQ 메시지 증가

완전 방어 = 3지점 모두 + DLX (처리 실패 메시지 보존)
```

---

## 🤔 생각해볼 문제

**Q1.** "최소한 한 번은 처리됨(At-Least-Once)"을 보장하기 위한 설정은 무엇이고, "정확히 한 번(Exactly-Once)"을 RabbitMQ에서 달성할 수 있는가?

<details>
<summary>해설 보기</summary>

**At-Least-Once 설정**: Publisher Confirm + 수동 Ack + durable Queue + PERSISTENT

이 설정으로 메시지가 유실되지 않는 것은 보장됩니다. 그러나 Consumer 장애 후 재전달로 인해 **중복 처리 가능성**이 있습니다.

**Exactly-Once**: RabbitMQ 자체적으로 Exactly-Once를 보장하지 않습니다.

Exactly-Once를 달성하는 방법:
1. **Consumer 멱등성**: 같은 메시지를 여러 번 받아도 결과가 동일하도록 구현
   - 메시지 ID를 DB에 저장, 중복 처리 시 skip
   - `INSERT ... ON CONFLICT DO NOTHING`
2. **Outbox Pattern**: DB 트랜잭션과 메시지 발행을 원자적으로 처리

완벽한 Exactly-Once는 분산 시스템의 이론적 한계 때문에 매우 어렵습니다. 실무에서는 At-Least-Once + Consumer 멱등성으로 충분한 경우가 대부분입니다.

</details>

---

**Q2.** Publisher가 메시지를 성공적으로 발행했지만, Broker가 재시작되기 직전에 fsync가 완료되지 않은 PERSISTENT 메시지가 있다. 이 메시지는 안전한가?

<details>
<summary>해설 보기</summary>

**Classic Queue + 단일 노드의 경우**: 안전하지 않을 수 있습니다.

PERSISTENT 메시지도 OS 페이지 캐시에 버퍼링됩니다. 브로커가 정상 종료(Graceful Shutdown)되면 fsync 후 종료하지만, **OOM Kill이나 전원 차단**의 경우 fsync 전 데이터가 소실될 수 있습니다.

Publisher Confirm의 미묘한 점:
- Classic Queue에서 Confirm은 "Queue에 도달했다"를 의미
- 반드시 fsync 완료를 의미하지는 않음

**완전한 내구성을 위한 방법**:
- **Quorum Queue 사용**: Raft 기반으로 과반수 노드에 WAL(Write-Ahead Log) 기록 후 Confirm. 가장 강한 내구성 보장
- 3노드 Quorum Queue: 1노드가 OOM Kill되어도 나머지 2노드에 기록됨

결론: 단일 노드 환경에서 완벽한 내구성은 어렵습니다. 프로덕션에서는 Quorum Queue + 클러스터 구성이 필수입니다.

</details>

---

**Q3.** Consumer가 처리 중 네트워크가 단절됐다. autoAck=false이고 Ack를 아직 보내지 않은 상태라면, 메시지는 어떻게 되는가?

<details>
<summary>해설 보기</summary>

**메시지는 안전하게 복구됩니다.**

동작 과정:
1. Consumer: msg1 수신 (Unacked 상태)
2. 네트워크 단절 → TCP Connection 끊김
3. RabbitMQ: Heartbeat 미수신 → 연결 장애로 판단
4. RabbitMQ: 해당 Connection의 모든 Unacked 메시지 → Ready 상태로 복귀
5. 다른 Consumer(또는 Connection 복구 후 같은 Consumer)에게 재전달

이것이 autoAck=false의 핵심 안전 메커니즘입니다.

주의사항:
- Heartbeat 시간만큼 메시지가 Unacked 상태로 묶임 (기본 60초)
- 빠른 복구를 위해 적절한 Heartbeat 설정 필요
- Consumer가 재연결하면 같은 메시지를 다시 받을 수 있으므로 멱등성 처리 권장

</details>

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[다음: Publisher Confirm ➡️](./02-publisher-confirm.md)**

</div>
