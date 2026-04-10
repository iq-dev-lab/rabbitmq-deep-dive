# RabbitMQ가 해결하는 문제 — 동기 호출 체인의 결합도

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 동기 HTTP 호출 체인은 왜 결합도 문제를 만들고, 메시지 큐는 어떻게 이를 해결하는가?
- Producer가 Consumer의 존재를 몰라도 되는 설계란 무엇이고, 왜 이것이 중요한가?
- RabbitMQ가 "메시지 브로커"이고 Kafka가 "분산 로그"라는 말의 의미는 무엇인가?
- 트래픽 버퍼(Buffer)로서의 메시지 큐는 어떤 문제를 막는가?
- RabbitMQ를 선택해야 하는 상황과 그렇지 않은 상황은 어떻게 구분하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

RabbitMQ를 도입하면 당장 코드가 복잡해진다. 비동기 처리, Ack, Exchange 설정 등 추가해야 할 것이 많다. 그렇다면 왜 이 복잡성을 감수하는가? 그것이 해결하는 문제를 모르면 RabbitMQ는 과잉 엔지니어링처럼 보인다.

동기 호출 체인이 만드는 세 가지 문제 — 결합도, 연쇄 장애, 트래픽 집중 — 을 정확히 이해해야, RabbitMQ가 어떤 트레이드오프를 감수하면서 무엇을 얻는지 판단할 수 있다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
상황: 주문 서비스가 결제, 재고, 알림 세 서비스를 차례로 호출

OrderService.placeOrder():
  1. PaymentService.charge()      → HTTP POST, 대기
  2. InventoryService.decrease()  → HTTP POST, 대기
  3. NotificationService.send()   → HTTP POST, 대기
  4. return "주문 완료"

문제 1: 결합도 (Coupling)
  PaymentService가 응답 느리면 → OrderService가 그대로 대기
  NotificationService가 다운되면 → 주문 전체 실패
  → 서비스 3개 모두 동시에 정상이어야만 주문 성공
  → 하나가 배포되면 전체 연쇄 확인 필요

문제 2: 연쇄 장애 (Cascading Failure)
  NotificationService: 일시적 OOM → 응답 없음 (30초 timeout)
  OrderService: 30초씩 스레드 점유 → 스레드 풀 소진
  → OrderService도 다운 → 결제/재고까지 연쇄 장애

문제 3: 트래픽 집중 (Traffic Spike)
  블랙프라이데이: 주문 100배 폭증
  → NotificationService: 처리 한계 초과 → 큐 넘침 → 요청 거절
  → 거절된 응답 받은 OrderService → 주문 실패 처리
  → 수만 명의 결제 정상인데 주문 실패 처리됨

흔한 대응:
  "재시도 로직 추가" → 동기 재시도가 오히려 부하 증폭
  "타임아웃 늘리기" → 스레드 점유 시간만 늘어남
  "NotificationService 스케일 아웃" → 근본 해결 아님
```

---

## ✨ 올바른 접근 (After — 메시지 큐 도입 후)

```
RabbitMQ 도입 후 설계:

OrderService.placeOrder():
  1. PaymentService.charge()          → HTTP POST, 대기 (결제는 동기 필요)
  2. RabbitMQ.publish("order.placed") → 메시지 발행 후 즉시 반환
  3. return "주문 완료"

InventoryService: "order.placed" 구독 → 재고 차감 (비동기)
NotificationService: "order.placed" 구독 → 알림 발송 (비동기)

해결 1: 결합도 제거
  OrderService는 InventoryService, NotificationService의 존재를 모름
  → 두 서비스가 다운되어도 OrderService는 메시지만 발행하고 완료
  → 구독자 추가/제거 시 OrderService 코드 수정 불필요

해결 2: 연쇄 장애 차단
  NotificationService 다운 → 메시지가 Queue에 쌓임
  → OrderService에는 영향 없음
  → NotificationService 복구 시 Queue에 쌓인 메시지 순서대로 처리

해결 3: 트래픽 버퍼
  주문 100배 폭증 → 메시지 100배 Queue에 적재
  → NotificationService는 자신의 처리 속도로 소비
  → 서비스가 감당 가능한 속도로 처리, 과부하 없음
  → 지연 발생하지만 유실 없음

=== 메시지 큐의 핵심 가치 ===

Before (동기):  OrderService ──→ NotificationService
                              (직접 호출, 강결합)

After (비동기):  OrderService ──→ [Queue] ──→ NotificationService
                              (간접 연결, 약결합)

변한 것:
  - OrderService는 Queue에만 의존 (Consumer 독립)
  - 처리 속도 불일치를 Queue가 흡수
  - 일시적 Consumer 장애를 Queue가 보존
```

---

## 🔬 내부 동작 원리

### 1. 메시지 큐가 결합도를 제거하는 원리

```
=== Temporal Decoupling (시간적 결합 제거) ===

동기 호출:
  Producer와 Consumer가 동시에 실행 중이어야 함
  Producer 호출 시점 = Consumer 처리 시점
  
  타임라인:
  Producer: ────[요청]────[대기 중...]────[완료]────
  Consumer: ────────────[처리 중...]──────────────
  
  Producer가 대기하는 동안 스레드 점유 → 스케일 한계

메시지 큐:
  Producer와 Consumer가 동시에 실행될 필요 없음
  Producer 발행 시점 ≠ Consumer 처리 시점
  
  타임라인:
  Producer: ──[발행]──[완료]──────────────────────
  Queue:    ──────[메시지 보관]────────────────────
  Consumer: ─────────────────[나중에 처리]──────────
  
  Producer는 발행 후 즉시 다른 작업 가능

=== Location Decoupling (위치적 결합 제거) ===

동기 HTTP:
  Producer가 Consumer의 Host, Port, 엔드포인트를 알아야 함
  Consumer IP 변경 → Producer 설정 변경 필요
  Consumer 여러 개 → Producer가 로드밸런싱 직접 처리

메시지 큐:
  Producer는 Exchange 이름과 Routing Key만 알면 됨
  Consumer가 어디 있는지, 몇 개인지 Producer는 무관
  Consumer 추가 → Exchange에 새 Queue 바인딩만 추가
  → Producer 코드 변경 없음

=== 메시지 큐가 없으면 불가능한 패턴 ===

이벤트 Fan-Out:
  "주문 발생" → 재고팀, 알림팀, 분석팀, 배송팀 동시 처리
  동기: OrderService에서 4개 서비스를 직접 호출
        → 4개 모두 성공해야 완료, 4개 모두 알아야 함
  메시지 큐: Exchange → 4개 Queue → 각 팀이 독립 소비
        → OrderService는 Exchange에만 발행
        → 새 팀 추가 시 새 Queue 바인딩만 추가
```

### 2. RabbitMQ(메시지 브로커) vs Kafka(분산 로그) — 설계 철학의 차이

```
=== RabbitMQ: Smart Broker, Dumb Consumer ===

설계 철학:
  브로커(RabbitMQ)가 메시지를 "배달"하는 책임을 짐
  Consumer는 메시지를 받아 처리하기만 하면 됨

메시지 생명주기:
  Producer → RabbitMQ → Consumer (처리) → 메시지 삭제
  
  Consumer가 Ack를 보내는 순간 메시지는 Queue에서 제거
  → "이미 처리된 메시지"는 저장하지 않음
  → Queue는 아직 처리 안 된 메시지만 보관
  
  처리된 메시지 재소비 불가:
  ✅ 이점: 처리 완료된 데이터 즉시 삭제 → 저장 공간 절약
  ❌ 한계: "어제 발행된 이벤트를 다시 처음부터 처리"하는 것 불가

라우팅 철학:
  Exchange → Binding → Queue 3단계 라우팅
  브로커가 메시지를 "어디로 보낼지" 판단
  Consumer는 Queue에서 꺼내기만 함
  → 복잡한 조건 기반 라우팅 가능

소비 방식:
  Push 방식 (Broker → Consumer로 밀어넣기)
  Consumer가 basicConsume 등록 → Broker가 메시지를 Push
  → Consumer는 메시지가 오면 처리하면 됨
  → Prefetch Count로 한 번에 받을 메시지 수 조절

=== Kafka: Dumb Broker, Smart Consumer ===

설계 철학:
  브로커(Kafka)는 메시지를 "보관"하는 책임만 짐
  Consumer가 어디까지 읽었는지(오프셋)를 직접 관리

메시지 생명주기:
  Producer → Kafka → 보관 기간(예: 7일)까지 보존
  
  Consumer가 읽어도 메시지 삭제 안 됨
  Consumer Group A가 읽어도 Consumer Group B도 독립적으로 읽기 가능
  오프셋 리셋하면 과거 메시지 재처리 가능
  
  ✅ 이점: 이벤트 소싱, 감사 로그, 재처리 필요한 경우 완벽
  ❌ 한계: 복잡한 조건 기반 라우팅 불가, 디스크 용량 관리 필요

소비 방식:
  Pull 방식 (Consumer가 Broker에서 직접 가져오기)
  Consumer가 poll() 호출 → 쌓인 메시지 배치로 가져옴
  → Consumer가 처리 속도를 직접 제어
  → 높은 처리량에 유리 (배치 처리)

=== 핵심 차이 요약 ===

특성                  | RabbitMQ              | Kafka
─────────────────────┼──────────────────────┼──────────────────────
메시지 보관           | Consumer 처리 후 삭제  | 보관 기간까지 보존
소비 방식            | Push (Broker → 소비자)  | Pull (소비자가 가져감)
라우팅               | Exchange/Binding 복잡  | Topic 직접 발행 단순
재처리               | 불가 (삭제됨)           | 가능 (오프셋 리셋)
여러 소비자 그룹      | Queue 공유 (경쟁 소비)  | Consumer Group별 독립
처리량               | 낮은 지연 (< 10ms)     | 높은 처리량 (수백만/초)
메시지 우선순위       | 지원                   | 미지원
```

### 3. 트래픽 버퍼(Buffer)로서의 메시지 큐

```
=== 버퍼 없는 시스템 (직접 호출) ===

  트래픽:  ████████████████████████████████████ (피크 100배)
           ──────────────────────────────────────────────
  서버:    ████████████████████ (용량 한계)
           ──────────────────────────────────────────────
  초과분:  XXXXXXXXXXXXXXXXXXXX (거절/오류 발생)

문제:
  평상시 트래픽에 맞춰 스케일 → 피크 시 초과
  피크에 맞춰 스케일 → 평상시 낭비 (비용 문제)

=== 메시지 큐 버퍼 ===

  발행 속도:  ████████████████████████████████████ (피크 100배)
              ─────────────────────────────────────────────────
  Queue:      점점 쌓임 ↗↗↗↗↗↗↗↗↗ 피크 후 소진 ↘↘↘↘↘↘↘
              ─────────────────────────────────────────────────
  소비 속도:  ████████████████ (일정, 서버 용량 내)

효과:
  피크 시 → Queue에 메시지 적재 (서버 과부하 없음)
  피크 후 → Queue 소진하며 처리 (지연 발생하지만 유실 없음)
  → 서버는 자신의 처리 속도로 안정적으로 운영

한계:
  지연(Latency) 증가 — Queue에 쌓인 메시지 처리까지 기다려야 함
  → 알림, 이메일, 통계 등 지연 허용 시스템에 적합
  → 결제, 재고 실시간 조회 등 즉시 결과 필요 시 부적합
```

---

## 💻 실전 실험

### 실험 1: 동기 호출 vs 메시지 큐 — 응답 시간 비교

```bash
# 실험 환경 시작
docker compose up -d rabbitmq

# RabbitMQ Management UI
open http://localhost:15672  # admin / admin

# 실험: 동기 호출 지연 시뮬레이션 (Spring 없이 Python으로 간단히)
# 메시지 발행 → 즉시 반환 확인

# rabbitmqadmin 설치 (Management Plugin CLI)
pip install rabbitmqadmin

# Exchange/Queue 생성
rabbitmqadmin declare exchange name=order.exchange type=topic durable=true
rabbitmqadmin declare queue name=order.notification durable=true
rabbitmqadmin declare binding \
  source=order.exchange \
  destination=order.notification \
  routing_key="order.#"

# 메시지 발행 (Producer)
time rabbitmqadmin publish \
  exchange=order.exchange \
  routing_key="order.placed" \
  payload='{"orderId": 1, "userId": 100}'
# → 수 ms 이내 완료 (Queue가 받는 순간 반환)

# Queue 상태 확인
rabbitmqctl list_queues name messages consumers
# order.notification  1  0  ← 메시지 1개 쌓임, Consumer 없음
```

### 실험 2: 트래픽 버퍼 효과 확인

```bash
# 100개 메시지 빠르게 발행
for i in $(seq 1 100); do
  rabbitmqadmin publish \
    exchange=order.exchange \
    routing_key="order.placed" \
    payload="{\"orderId\": $i}" &
done
wait

# Queue에 메시지 쌓임 확인
rabbitmqctl list_queues name messages
# order.notification  100  ← 100개 적재됨

# Consumer 없이 Queue가 메시지 보관하고 있음
# → 발행자는 이미 모두 완료됨 (빠른 응답)
# → Consumer가 나중에 연결되면 순서대로 처리

# Consumer 연결 (하나씩 꺼내기)
rabbitmqadmin get queue=order.notification ackmode=ack_requeue_false
# count를 늘려가며 Queue 소진 확인
```

### 실험 3: Management UI로 메시지 흐름 시각화

```
RabbitMQ Management UI 확인 항목:

1. http://localhost:15672/#/queues
   - order.notification 클릭
   - Messages: Ready / Unacked / Total
   - Message Rates: Publish rate / Deliver rate
   
2. Publish 속도 > Consume 속도 시뮬레이션:
   - 발행 스크립트 실행
   - UI에서 Queue Depth 증가 확인
   - Consumer 중단 후 Queue Depth 급증 확인

3. 핵심 지표 의미:
   Ready: Consumer에게 전달 가능한 메시지 수
   Unacked: Consumer가 받았지만 아직 Ack 안 한 메시지 수
   Total: Ready + Unacked
```

---

## 📊 Kafka vs RabbitMQ 비교

```
=== 같은 "주문 발생" 이벤트 처리 비교 ===

RabbitMQ 방식:
  Producer → order.exchange (Topic) → order.notification (Queue)
                                    → order.inventory (Queue)
                                    → order.analytics (Queue)
  
  특징:
  - 각 서비스가 자신만의 Queue 가짐
  - 메시지 처리 후 삭제
  - Queue별 독립적인 Ack 관리
  - 새 서비스 추가: 새 Queue 생성 + Binding 추가 (재발행 불필요)

Kafka 방식:
  Producer → order-events (Topic, Partition 3개)
  
  Consumer Group A (NotificationService): offset 추적하며 소비
  Consumer Group B (InventoryService):   offset 추적하며 소비
  Consumer Group C (AnalyticsService):   offset 추적하며 소비
  
  특징:
  - 모든 Consumer Group이 동일한 Topic에서 독립적으로 소비
  - 메시지 보관 기간(예: 7일)까지 보존
  - 새 서비스 추가: 새 Consumer Group 생성 (과거 메시지부터 처리 가능)

=== 선택 기준 ===

RabbitMQ 선택:
  ✅ "주문 실패 알림은 즉시 보내야 한다" → 낮은 지연 < 10ms
  ✅ "VIP 주문은 우선 처리해야 한다" → Priority Queue 지원
  ✅ "결제 실패 시 다른 서비스로 라우팅해야 한다" → 복잡한 라우팅
  ✅ "처리 완료된 메시지는 즉시 삭제되어야 한다" → 개인정보 규정
  ✅ "요청-응답(RPC) 패턴이 필요하다" → Reply-To Queue 지원

Kafka 선택:
  ✅ "어제 발생한 이벤트를 다시 처음부터 분석해야 한다" → 재처리
  ✅ "초당 100만 건 주문 로그를 수집해야 한다" → 높은 처리량
  ✅ "결제팀, 분석팀, 감사팀이 같은 이벤트를 독립적으로 소비" → Consumer Group
  ✅ "이벤트 소싱(Event Sourcing) 패턴을 구현해야 한다" → 로그 보관
```

---

## ⚖️ 트레이드오프

```
메시지 큐 도입의 장단점:

장점:
  ① 결합도 제거 — 서비스 간 직접 의존 제거
  ② 장애 격리 — Consumer 장애가 Producer에 영향 없음
  ③ 트래픽 버퍼 — 피크 트래픽 흡수, 안정적 처리
  ④ 확장성 — Consumer 수평 확장이 Producer와 무관

단점:
  ① 복잡성 증가 — Exchange/Queue/Binding 설정 관리 필요
  ② 지연 증가 — Queue를 거치므로 직접 호출보다 느림
  ③ 운영 부담 — RabbitMQ 클러스터 운영, 모니터링 필요
  ④ 메시지 유실 가능성 — 설정 없이는 메시지 보장 안 됨
  ⑤ 최종 일관성 — 즉시 일관성이 아닌 Eventually Consistent

도입 판단 기준:
  "Consumer 장애가 Producer에 영향을 주는가?" → 메시지 큐 필요
  "트래픽 피크에 Consumer가 버티지 못하는가?" → 메시지 큐 필요
  "처리 지연이 허용되지 않는 실시간 조회인가?" → 직접 호출 유지
  "처리 결과를 Producer가 즉시 알아야 하는가?" → 동기 호출 유지
```

---

## 📌 핵심 정리

```
RabbitMQ가 해결하는 문제 핵심:

동기 호출 체인의 3가지 문제:
  ① 결합도: Consumer가 다운되면 Producer도 실패
  ② 연쇄 장애: 하나의 서비스 장애가 전체로 전파
  ③ 트래픽 집중: Consumer 처리 속도 초과 시 요청 거절

메시지 큐의 3가지 해결:
  ① Temporal Decoupling: Producer와 Consumer가 동시 실행 불필요
  ② Location Decoupling: Producer는 Exchange만 알면 됨
  ③ Traffic Buffering: Queue가 속도 불일치를 흡수

RabbitMQ vs Kafka 선택:
  메시지 처리 후 삭제 + 복잡한 라우팅 + 낮은 지연 → RabbitMQ
  메시지 보관 + 높은 처리량 + 재처리 가능 → Kafka

도입 전 질문:
  "처리 결과를 즉시 알아야 하는가?" → Yes: 동기, No: 비동기 검토
  "Consumer 장애가 Producer를 멈추게 하는가?" → Yes: 메시지 큐
  "트래픽 피크와 평균의 차이가 큰가?" → Yes: 메시지 큐
```

---

## 🤔 생각해볼 문제

**Q1.** 결제 서비스에 RabbitMQ를 도입하려 한다. "결제 요청 → 결제 처리 → 결과 반환"을 비동기로 처리할 수 있는가? 어떤 조건이 충족되어야 하는가?

<details>
<summary>해설 보기</summary>

결제는 일반적으로 **동기 응답이 필요**합니다. 사용자는 "결제가 됐는지 안 됐는지" 즉시 알아야 합니다.

그러나 비동기로 처리할 수 있는 조건이 있습니다:
1. **RPC 패턴 사용**: Reply-To Queue + Correlation ID로 요청-응답을 구현. RabbitMQ가 비동기이지만 Producer가 응답을 기다림. 네트워크 격리 효과는 있지만 지연이 추가됨.
2. **낙관적 UI**: "결제 요청을 접수했습니다. 처리 후 알림 드립니다" 방식. 사용자에게 즉시 접수 확인 후, 실제 처리 완료 시 Webhook/알림으로 통보.

결론: 결제의 **접수**는 비동기 가능, **결과 통보**는 동기 또는 Push 알림. 비동기 도입 전 "사용자가 결과를 언제 알아야 하는가"를 먼저 정의해야 합니다.

</details>

---

**Q2.** 주문 서비스가 "order.placed" 이벤트를 발행했다. 재고 서비스에서 재고 부족으로 처리 실패했다. 이 실패를 주문 서비스에 어떻게 알릴 수 있는가?

<details>
<summary>해설 보기</summary>

비동기 메시징에서 실패 전파는 두 가지 패턴이 있습니다:

**패턴 1: 보상 이벤트 (Compensating Event)**
- InventoryService: "inventory.shortage" 이벤트 발행
- OrderService: "inventory.shortage" 구독 → 주문 취소 처리
- 각 서비스가 독립적으로 실패 상황에 반응

**패턴 2: Saga 패턴**
- 각 단계를 트랜잭션으로 보고, 실패 시 이전 단계를 취소하는 보상 트랜잭션 발행
- "order.cancel" 이벤트를 발행하면 결제 서비스가 환불 처리

핵심은 **동기 에러 전파가 불가능**하다는 점을 인정하고, 비동기 이벤트로 실패 상황을 표현하는 것입니다. 이로 인해 시스템은 즉시 일관성이 아닌 **최종 일관성(Eventual Consistency)**을 갖게 됩니다.

</details>

---

**Q3.** "RabbitMQ 없이 Redis Pub/Sub으로도 비동기 처리가 가능한데, 굳이 RabbitMQ를 써야 하는가?"

<details>
<summary>해설 보기</summary>

Redis Pub/Sub의 결정적 한계:

1. **메시지 보장 없음**: 구독자가 없거나 다운된 상태에서 발행된 메시지는 영구 소실. "Fire and Forget"
2. **영속성 없음**: Redis 재시작 시 처리 중인 메시지 모두 소실
3. **Ack 없음**: Consumer가 메시지를 받았는지, 처리 성공했는지 확인 불가
4. **라우팅 없음**: 패턴 기반 채널 이름 구독만 가능, 복잡한 라우팅 불가

RabbitMQ가 추가로 제공하는 것:
- **Durable Queue**: Broker 재시작해도 메시지 보존
- **Consumer Ack**: 처리 완료 확인 후 메시지 삭제
- **Dead Letter Exchange**: 처리 실패 메시지 별도 보관
- **Publisher Confirm**: Broker 수신 확인

"메시지 유실이 허용되는 실시간 이벤트(예: 실시간 알림 카운트, 라이브 채팅)"는 Redis Pub/Sub으로 충분합니다. "메시지 유실이 허용되지 않는 중요 이벤트(주문, 결제, 재고)"는 RabbitMQ가 필요합니다.

</details>

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[다음: AMQP 프로토콜 완전 분해 ➡️](./02-amqp-protocol.md)**

</div>
