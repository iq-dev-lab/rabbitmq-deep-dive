# Delayed/Scheduled Message — 지연 메시지 구현의 두 가지 방법

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- TTL + DLX 조합으로 지연 메시지를 구현하는 원리는 무엇인가?
- RabbitMQ Delayed Message Plugin은 어떻게 동작하고 한계는 무엇인가?
- 두 방법 중 어떤 것을 선택해야 하는가?
- "정확한 지연 보장"이 어려운 이유는 무엇인가?
- Quartz 스케줄러와 RabbitMQ 지연 메시지는 어떻게 책임을 나누는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

"주문 후 30분이 지나도 결제가 안 됐으면 자동 취소", "회원가입 24시간 후 웰컴 이메일 발송", "실패한 작업을 5분 후 재시도". 이 모든 것이 지연 메시지 패턴이다. DB 기반 스케줄러(Quartz)로도 구현할 수 있지만, RabbitMQ의 기존 메시지 신뢰성 메커니즘과 통합하면 재시도, 장애 복구, 모니터링을 일관되게 처리할 수 있다. 두 가지 구현 방법의 원리와 정확도 한계를 알아야 상황에 맞는 방법을 선택할 수 있다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: TTL Queue에 Consumer를 붙임

  wait.30min.queue (TTL=1800000ms, DLX=main.exchange)
  ↑ Consumer 추가!

  결과:
    Consumer가 TTL 만료 전에 메시지를 즉시 소비
    TTL 지연 효과 없음
    30분을 기다리지 않고 즉시 처리됨

  핵심: TTL 대기 Queue에는 Consumer 없음 (의도적으로)

실수 2: Plugin 없이 정확한 ms 단위 지연 기대

  TTL + DLX 방식의 한계:
    TTL 만료 = 정확한 시각에 처리 시작이 아님
    RabbitMQ가 만료 메시지를 주기적으로 검사하고 이동
    → 수십 ms ~ 수 초의 추가 지연 발생 가능

  기대: "정확히 30분 후 처리"
  실제: "30분 + α초 후 처리" (α는 브로커 부하에 따라 가변)

실수 3: 메시지당 다른 지연 시간을 하나의 TTL Queue로 처리

  wait.queue (TTL=60000, DLX=main.exchange)
  
  // 5초 지연 메시지 발행
  rabbitTemplate.convertAndSend("", "wait.queue", msg5s);
  // 60초 지연 메시지 발행
  rabbitTemplate.convertAndSend("", "wait.queue", msg60s);

  문제:
    Queue는 FIFO
    5초 후 처리 원하는 msg5s가 msg60s 뒤에 들어오면
    msg60s가 만료될 때까지 msg5s도 대기
    → Queue 레벨 TTL은 앞 메시지가 만료돼야 다음 메시지 확인

  해결: 메시지 레벨 TTL (x-message-ttl 아닌 개별 메시지 expiration)
        또는 Delayed Message Plugin
```

---

## ✨ 올바른 접근 (After — 방법별 적합한 사용 사례)

```
방법 1: TTL + DLX (Plugin 없이 구현)

  설계:
    wait.5s.queue  (x-message-ttl=5000,   DLX=main.exchange)
    wait.30s.queue (x-message-ttl=30000,  DLX=main.exchange)
    wait.5m.queue  (x-message-ttl=300000, DLX=main.exchange)
    wait.30m.queue (x-message-ttl=1800000, DLX=main.exchange)

  각 Queue에 Consumer 없음
  지연 후 DLX → 원본 Queue로 라우팅
  미리 정해진 지연 시간에만 적합 (5초, 30초, 5분 등)

방법 2: Delayed Message Plugin

  Exchange에 x-delayed-type 설정
  메시지 헤더에 x-delay 값(ms) 설정
  → 임의의 지연 시간 가능
  → 메시지당 다른 지연 시간 지원

  rabbitmq-plugins enable rabbitmq_delayed_message_exchange

  @Bean
  public CustomExchange delayedExchange() {
    Map<String, Object> args = new HashMap<>();
    args.put("x-delayed-type", "direct");
    return new CustomExchange("delayed.exchange", "x-delayed-message", true, false, args);
  }

  // 발행 시 지연 시간 설정
  rabbitTemplate.convertAndSend("delayed.exchange", "order.timeout",
    message, msg -> {
      msg.getMessageProperties().setHeader("x-delay", 1800000);  // 30분
      return msg;
    }
  );
```

---

## 🔬 내부 동작 원리

### 1. TTL + DLX 방식 상세

```
=== 동작 흐름 ===

발행:
  Producer → wait.30m.queue (TTL=1800000ms, DLX=order.exchange)
  메시지: {orderId: 1, action: "auto-cancel"}

대기 (30분):
  wait.30m.queue: 메시지 저장 (Consumer 없음)
  30분 동안: Queue가 메시지 보관

TTL 만료:
  RabbitMQ: 메시지 만료 감지
  → DLX(order.exchange)로 메시지 이동
  → x-dead-letter-routing-key 설정 시 해당 Key로 라우팅

처리:
  order.exchange → order.timeout.queue → Consumer 처리
  Consumer: orderId=1 주문 아직 미결제? → 자동 취소

=== TTL Queue 구성 ===

설정:
  rabbitmqadmin declare queue name=wait.30m durable=true \
    arguments='{
      "x-message-ttl": 1800000,
      "x-dead-letter-exchange": "order.exchange",
      "x-dead-letter-routing-key": "order.timeout"
    }'

  // Consumer 없음! 의도적으로 비워둠

  rabbitmqadmin declare queue name=order.timeout.queue durable=true
  rabbitmqadmin declare binding source=order.exchange \
    destination=order.timeout.queue routing_key=order.timeout

=== TTL 정확도의 한계 ===

Queue 레벨 TTL (x-message-ttl):
  Queue 전체에 동일한 TTL 적용
  만료 확인: Queue 앞에서부터 순서대로 확인
  문제: Queue 앞에 아직 만료 안 된 메시지 있으면
        그 뒤의 만료된 메시지도 대기

메시지 레벨 TTL (MessageProperties.setExpiration):
  개별 메시지에 만료 시간 설정
  주의: Queue 앞의 메시지가 Block하면 뒤 메시지의 TTL 만료 처리 지연
  → 정확한 시간에 처리 보장 어려움

Delayed Message Plugin:
  Mnesia DB에 메시지와 만료 시각 저장
  별도 타이머 스레드가 만료 시각에 Exchange로 라우팅
  더 정확한 지연 보장 (수십 ms 이내)
  단, 여전히 완벽한 ms 정확도는 아님
```

### 2. Delayed Message Plugin 동작

```
=== Plugin 설치 및 설정 ===

설치:
  rabbitmq-plugins enable rabbitmq_delayed_message_exchange
  # 또는
  docker run -d rabbitmq:3.12-management
  # 이미지에 Plugin 포함된 경우도 있음

Exchange 선언:
  type: "x-delayed-message"
  x-delayed-type: "direct" | "topic" | "fanout"
  → 지연 후 라우팅에 사용할 Exchange 유형

=== 메시지 처리 흐름 ===

발행:
  headers: {x-delay: 1800000}  ← 30분(ms)
  exchange: delayed.exchange
  routing_key: order.timeout

Plugin 처리:
  1. 메시지 수신 → Mnesia DB에 저장
     (메시지 내용 + routing_key + 만료 시각)
  2. 타이머 스레드: 만료 시각 모니터링
  3. 만료 시각 도달 → x-delayed-type 방식으로 라우팅
     (direct라면 routing_key로 일치하는 Queue에 전달)

=== Plugin의 특성과 한계 ===

특성:
  - 메시지마다 다른 지연 시간 가능 (TTL Queue 방식 대비 장점)
  - Mnesia DB에 저장 → 재시작 후에도 지연 유지 (durable)
  - 지연 시간: 최대 약 49일 (int 범위: 2^31-1 ms)

한계:
  - 브로커 노드당 Mnesia DB 용량 제한
  - 대량의 지연 메시지 (수백만 건) → Mnesia 성능 저하
  - 클러스터 환경에서 지연 메시지는 Plugin이 설치된 노드에 저장
    → 노드 장애 시 해당 노드의 지연 메시지 영향
  - Plugin 업데이트/유지보수 필요

=== 두 방법 비교 ===

특성               | TTL + DLX              | Delayed Message Plugin
──────────────────┼───────────────────────┼──────────────────────
지연 시간 유연성   | 고정 (Queue별)          | 메시지마다 다르게 가능
정확도            | 낮음 (Queue FIFO 영향)   | 높음 (타이머 기반)
설치 필요         | 없음 (기본 기능)         | Plugin 설치 필요
대량 지연 메시지   | 여러 Queue로 분산 가능   | Mnesia 부하
재시작 후 보존    | durable Queue면 가능     | Mnesia에 보존
클러스터 지원     | 완전 지원               | 제한적 (단일 노드)
```

### 3. Quartz 스케줄러와의 책임 분리

```
=== Quartz vs RabbitMQ 지연 메시지 ===

Quartz 스케줄러:
  - "특정 시각에 특정 작업 실행"
  - 고정 스케줄: cron("0 0 * * * ?") = 매시 정각
  - DB에 Job 정보 저장 → 클러스터 환경 지원
  - Job이 실패하면 Quartz가 재시도 관리

RabbitMQ 지연 메시지:
  - "N초 후에 메시지 처리"
  - 메시지 기반: 이벤트 발생 → N초 후 처리
  - RabbitMQ의 신뢰성(Ack, DLX, Confirm) 활용
  - 처리 실패 시 DLX → 재처리 큐로 이동

=== 책임 분리 원칙 ===

Quartz로 처리:
  - 매일 자정 정산 배치 실행
  - 매 시간 데이터 집계
  - 정기적인 시스템 유지보수 작업
  - 실패 시 관리자 알림 + 수동 재실행

RabbitMQ 지연 메시지로 처리:
  - "주문 30분 후 미결제 → 취소" (이벤트 기반 지연)
  - "재시도 10초 후" (처리 실패 지연 재시도)
  - "N분 후 후속 처리" (비즈니스 이벤트 체인)
  - 실패 시 DLX → 자동 재처리

=== 조합 패턴 ===

Quartz: 매일 자정 → "미결제 주문 목록 조회"
         → 각 주문을 RabbitMQ 지연 메시지로 발행
RabbitMQ: 각 주문마다 개별 처리 (병렬, 실패 격리)
          실패 시 DLX → 재처리

→ Quartz: 트리거 역할 / RabbitMQ: 실제 처리 역할
```

---

## 💻 실전 실험

### 실험 1: TTL + DLX 지연 메시지 구성

```bash
# 대기 Queue (Consumer 없음)
rabbitmqadmin declare queue name=wait.30s durable=true \
  arguments='{"x-message-ttl":30000,"x-dead-letter-exchange":"main.exchange","x-dead-letter-routing-key":"order.timeout"}'

rabbitmqadmin declare exchange name=main.exchange type=direct durable=true
rabbitmqadmin declare queue name=order.timeout.queue durable=true
rabbitmqadmin declare binding source=main.exchange \
  destination=order.timeout.queue routing_key=order.timeout

# 지연 메시지 발행
rabbitmqadmin publish exchange="" routing_key=wait.30s \
  payload='{"orderId":1,"action":"auto-cancel"}'

# 30초 대기
echo "30초 후 처리됩니다..."
sleep 31

# order.timeout.queue에 도착 확인
rabbitmqctl list_queues name messages
# order.timeout.queue  1  ← 30초 후 도착
```

### 실험 2: Delayed Message Plugin (설치 필요)

```bash
# Plugin 활성화
rabbitmq-plugins enable rabbitmq_delayed_message_exchange

# Delayed Exchange 선언
rabbitmqadmin declare exchange name=delayed.exchange type=x-delayed-message \
  arguments='{"x-delayed-type":"direct"}'

rabbitmqadmin declare queue name=delayed.target durable=true
rabbitmqadmin declare binding source=delayed.exchange \
  destination=delayed.target routing_key=delayed

# x-delay 헤더로 메시지별 지연 시간 지정 (ms)
rabbitmqadmin publish exchange=delayed.exchange routing_key=delayed \
  properties='{"headers":{"x-delay":10000}}' \
  payload='{"action":"notify","delay":"10s"}'

rabbitmqadmin publish exchange=delayed.exchange routing_key=delayed \
  properties='{"headers":{"x-delay":5000}}' \
  payload='{"action":"check","delay":"5s"}'

# 5초 후 "check" 메시지 먼저 도착, 10초 후 "notify" 도착 확인
```

---

## 📊 Kafka vs RabbitMQ 비교

```
=== 지연 메시지 처리 비교 ===

RabbitMQ:
  TTL + DLX: 기본 기능으로 구현 (Plugin 불필요)
  Delayed Message Plugin: 메시지별 다른 지연 시간

Kafka:
  기본 지연 메시지 미지원
  구현 방법:
  1. Consumer에서 직접 지연:
     sleep(delay) 후 처리 → Consumer 스레드 블로킹
  2. 별도 지연 처리 서비스:
     DB에 지연 메시지 저장 → 스케줄러가 Kafka로 재발행
  3. Kafka Streams에서 windowing으로 유사 구현

대체 도구:
  Redis Sorted Set: 만료 시각을 Score로 저장, 스케줄러가 조회 발행
  → Kafka 환경에서 RabbitMQ 지연 메시지 역할 대체 가능

결론:
  지연 메시지 기본 지원: RabbitMQ 우위
  Kafka 환경: Redis 또는 별도 서비스 조합 필요
```

---

## ⚖️ 트레이드오프

```
TTL + DLX:
  ✅ Plugin 불필요 (기본 기능)
  ✅ 안정적 (검증된 방식)
  ❌ 지연 시간 고정 (Queue마다 하나의 TTL)
  ❌ 메시지별 다른 지연 어려움 (별도 Queue 필요)
  → 재시도 지연, 고정된 N분 후 처리에 적합

Delayed Message Plugin:
  ✅ 메시지마다 다른 지연 시간
  ✅ 정확한 지연 보장 (타이머 기반)
  ❌ Plugin 설치/관리 필요
  ❌ 대량 지연 메시지 시 Mnesia 성능 저하
  ❌ 클러스터 환경 제한적
  → 이벤트 기반 동적 지연 (주문별 타임아웃)에 적합

Quartz 스케줄러 (별도 도구):
  ✅ 정확한 시각 실행 (cron)
  ✅ 클러스터 환경 완전 지원
  ❌ 메시지 기반 처리 아님 (RabbitMQ 신뢰성 미활용)
  → 정기 배치, 정해진 시각에 실행에 적합
```

---

## 📌 핵심 정리

```
지연 메시지 핵심:

방법 1: TTL + DLX
  wait.Ns.queue (TTL=N초, DLX, Consumer 없음)
  N초 후 TTL 만료 → DLX → 원본 Queue
  고정 지연 시간에 적합, Plugin 불필요

방법 2: Delayed Message Plugin
  x-delayed-message Exchange 타입
  headers: {x-delay: N} (메시지별 ms 단위)
  메시지마다 다른 지연 가능

정확도:
  두 방법 모두 완벽한 ms 정확도 보장 안 함
  수십 ms ~ 수 초의 추가 지연 가능
  비즈니스 수준(분 단위)에서는 충분

Quartz 분리:
  정기 스케줄: Quartz
  이벤트 기반 지연: RabbitMQ 지연 메시지

선택 기준:
  고정 지연 + Plugin 없이: TTL + DLX
  동적 지연 + Plugin 허용: Delayed Message Plugin
  정확한 시각 실행: Quartz
```

---

## 🤔 생각해볼 문제

**Q1.** TTL Queue에 메시지가 10개 쌓여 있다. 처음 8개는 TTL=60초이고, 9번째는 TTL=5초이다. 9번째 메시지는 5초 후에 DLX로 이동하는가?

<details>
<summary>해설 보기</summary>

**Queue 레벨 TTL (x-message-ttl)을 쓴다면: 아닐 수 있습니다.**

Queue 레벨 TTL은 Queue 앞에서부터 순서대로 만료를 처리합니다. 앞의 8개 메시지가 만료되기 전까지 9번째 메시지의 만료도 처리되지 않을 수 있습니다.

**메시지 레벨 TTL (MessageProperties.setExpiration)을 쓴다면:**
각 메시지는 자신의 만료 시간을 가지지만, RabbitMQ의 만료 처리는 Queue 앞의 메시지를 먼저 확인합니다. 앞 메시지가 아직 만료 안 됐으면 뒤 메시지의 만료 처리가 지연될 수 있습니다.

**Delayed Message Plugin을 쓴다면:**
각 메시지가 독립적인 타이머를 가집니다. 9번째 메시지는 정확히 5초 후에 Exchange로 라우팅됩니다.

결론: 메시지별로 다른 지연 시간이 필요하다면 Delayed Message Plugin이 유일한 정확한 해결책입니다.

</details>

---

**Q2.** 30분 후 자동 취소 기능에 TTL=1800000ms Queue를 사용한다. 브로커가 재시작됐다. 지연 중인 메시지는 어떻게 되는가?

<details>
<summary>해설 보기</summary>

**Queue와 메시지 설정에 따라 다릅니다.**

**경우 1: durable Queue + PERSISTENT 메시지**
- Queue 정의: 재시작 후 복원
- 메시지: 디스크에 보존 → 재시작 후 복원
- TTL 타이머: **처음부터 다시 시작** (재시작 전 경과 시간 없어짐)
- 결과: 브로커 재시작 시 모든 지연이 초기화

이것이 TTL + DLX의 큰 한계입니다. 재시작 후 "30분 대기가 25분 남았던 메시지"가 다시 "30분 대기"로 리셋됩니다.

**경우 2: Delayed Message Plugin**
- Mnesia DB에 메시지와 만료 시각 저장
- 재시작 후 Mnesia 복구 → 남은 지연 시간으로 타이머 재설정
- **정확한 만료 시각 유지** (브로커 재시작에도)

비즈니스 영향:
- 자동 취소가 30분 후여야 하는데 재시작 때문에 60분 후가 됨
- 고객 경험에 영향을 줄 수 있음

완화 방법:
- 브로커 재시작 최소화 (Quorum Queue + Rolling Upgrade)
- 또는 Delayed Message Plugin 사용

</details>

---

**Q3.** "1시간 후 만료된 세션을 정리" 작업을 RabbitMQ 지연 메시지로 구현할 것인가, Quartz 스케줄러로 구현할 것인가?

<details>
<summary>해설 보기</summary>

**Quartz 스케줄러가 더 적합합니다.** 이유:

**Quartz가 나은 이유:**
- "1시간 후" 정리는 사용자 행동 이벤트가 아닌 시간 기반 정기 작업
- 세션이 100만 개라면 RabbitMQ에 100만 개 지연 메시지 → Mnesia 과부하
- Quartz: DB에서 만료된 세션을 배치로 조회 → 효율적

**RabbitMQ 지연 메시지가 나은 경우:**
- 세션 생성 이벤트가 메시지로 발행되는 구조이고,
- 각 세션의 만료 시각이 서로 다를 때 (ex: "로그인 후 정확히 N분 후 만료")
- 세션 수가 적고 (수천 건 이하) 이벤트 기반 처리가 필요할 때

**결론:**
- 대량 세션 정리 (주기적 배치): Quartz
- 특정 이벤트 후 N분 후 단건 처리: RabbitMQ 지연 메시지
- 두 방법 조합: Quartz로 만료 세션 목록 조회 → 각 세션을 RabbitMQ로 개별 처리

</details>

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[이전: Priority Queue ⬅️](./04-priority-queue.md)** | **[다음: Saga 패턴 구현 ➡️](./06-saga-pattern.md)**

</div>
