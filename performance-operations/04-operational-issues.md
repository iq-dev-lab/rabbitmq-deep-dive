# 운영 중 발생하는 문제 패턴 — 진단과 즉시 해결 가이드

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Queue 메시지가 무한히 쌓이는 원인을 어떻게 빠르게 진단하는가?
- Unacknowledged 메시지가 누적되면 어떤 증상이 나타나고 어떻게 해결하는가?
- 연결 폭풍(Connection Storm)은 왜 발생하고, 어떻게 방지하는가?
- 각 문제의 임시 해결책과 근본 해결책은 무엇인가?
- 문제 발생 시 서비스 영향을 최소화하는 순서는?

---

## 🔍 왜 이 개념이 실무에서 중요한가

프로덕션에서 RabbitMQ 문제는 대부분 세 가지 패턴에서 나온다. Queue가 쌓이는 문제, Unacked 메시지가 누적되는 문제, 재시작 후 연결이 폭발하는 문제다. 각각 증상은 비슷해 보이지만 원인과 해결책이 다르다. "RabbitMQ가 느리다"는 모호한 신고가 들어왔을 때 5분 안에 원인 카테고리를 특정하고, 서비스 영향을 최소화하는 임시 처치를 적용하고, 근본 원인을 해결하는 체계를 알아야 한다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: Queue 적체에 RabbitMQ 재시작으로 대응

  증상: Queue에 메시지 100만 건 적체
  대응: "RabbitMQ 재시작하면 낫겠지"
  
  결과:
    durable=true + PERSISTENT: 재시작 후 100만 건 그대로
    Consumer 처리 속도 문제가 해결 안 됐으면 → 다시 쌓임
    재시작 중 Connection Storm 추가 발생

실수 2: Unacked 누적을 "메시지가 처리 중"으로 오해

  증상: Unacked 50,000개
  판단: "Consumer가 열심히 처리 중이구나"
  실제: Consumer가 응답 없는 외부 API를 기다리며 블로킹
         Prefetch 슬롯을 모두 점유 → 새 메시지 처리 불가
         사실상 Consumer 마비 상태

실수 3: 연결 폭풍을 개별 서비스 재시작으로 해결

  증상: RabbitMQ 재시작 후 Connection 1000개 동시 생성
  대응: 각 서비스 하나씩 재시작
  결과: 재시작된 서비스가 다시 즉시 연결 시도 → 반복
        지수 백오프 없으면 폭풍 계속
```

---

## ✨ 올바른 접근 (After — 문제 유형별 진단 트리)

```
5분 진단 플로우:

Step 1: Management UI 확인
  Queue Depth 급증? → 문제 패턴 A (적체)
  Unacked 급증?     → 문제 패턴 B (Unacked 누적)
  Connection 급증?  → 문제 패턴 C (연결 폭풍)

Step 2: 문제 패턴별 즉시 처치

  A (적체):
    임시: Consumer 인스턴스 증가 (Scale Out)
    확인: Consumer Utilisation 모니터링
    근본: Consumer 처리 속도 최적화 또는 발행 속도 제한

  B (Unacked 누적):
    임시: 해당 Consumer 재시작 (Connection 끊김 → Unacked 복귀)
    확인: 재시작 후 Unacked = 0인지 확인
    근본: Consumer 블로킹 원인 제거 (외부 의존성 타임아웃 설정)

  C (연결 폭풍):
    임시: 서비스별 순차 재시작 (동시 재시작 금지)
    확인: Connection 수가 정상 범위로 감소하는지
    근본: 클라이언트 재연결 로직에 Exponential Backoff + Jitter 추가
```

---

## 🔬 내부 동작 원리

### 문제 패턴 A — Queue 메시지 무한 적체

```
=== 발생 원인 ===

원인 1: Consumer 처리 속도 < 발행 속도

  발행: 초당 5000개
  Consumer 3개 × 처리 100개/sec = 300개/sec
  초당 4700개씩 적체 → 1시간 후 1700만 건

원인 2: Consumer 전체 다운

  배포 중 롤링 업데이트로 Consumer 0개 구간 발생
  발행 중단 없음 → Queue에 무제한 적체

원인 3: 특정 메시지에서 Consumer 무한 루프

  일부 메시지 처리 실패 → requeue=true → 즉시 재처리
  CPU 100% + 다른 메시지 차단
  Queue Depth: 해당 메시지만 계속 순환 (숫자는 작지만 처리 불가)

=== 진단 ===

rabbitmqctl list_queues name messages messages_ready messages_unacknowledged consumers
# 출력 예:
# order.queue  850000  849000  1000  3  ← Ready 많음, Consumer는 있음

# Ready가 많고 Consumer가 있으면 → 처리 속도 문제
# Ready 많고 Consumer = 0이면 → Consumer 전체 다운

# Consumer Utilisation 확인 (Management UI):
# 100% → Consumer 과부하, Scale Out 필요
# 낮음 → Prefetch 문제 또는 Consumer 블로킹

=== 임시 처치 ===

1. Consumer Scale Out:
   kubectl scale deployment order-consumer --replicas=10
   # 또는 Spring AMQP max-concurrency 동적 증가

2. 발행 속도 제한 (긴급 시):
   // Publisher에 Rate Limiter 적용
   RateLimiter limiter = RateLimiter.create(1000);  // 초당 1000개로 제한
   limiter.acquire();
   rabbitTemplate.convertAndSend(...);

3. 오래된 메시지 일괄 제거 (비즈니스 판단 필요):
   rabbitmqctl purge_queue order.queue
   // 주의: 모든 메시지 삭제 → 비즈니스 영향 확인 필수

=== 근본 해결 ===

처리 시간 프로파일링:
  Consumer 로그에 처리 시간 기록
  평균 처리 시간 × 목표 처리량 = 필요 Consumer 수

병목 제거:
  DB 쿼리 최적화 (인덱스, 쿼리 튜닝)
  외부 API 비동기 처리
  배치 처리로 전환 (개별 → 배치 Insert)

용량 계획:
  피크 발행량 × 1.5 처리량을 Consumer가 처리 가능하도록 설계
```

### 문제 패턴 B — Unacknowledged 메시지 누적

```
=== 발생 원인 ===

원인 1: Consumer 블로킹 (I/O 대기)

  Consumer: 외부 API 호출 (타임아웃 없음)
  외부 API: 응답 안 함 (다운)
  Consumer: 무한 대기 (스레드 블로킹)
  Prefetch=10이면 → 10개 Unacked 점유 → 새 메시지 수신 불가

원인 2: Prefetch 너무 크고 처리가 느림

  Prefetch=1000, 처리 시간=500ms
  Consumer가 1000개를 받아 하나씩 처리
  처음 1000개 처리하는 동안 → 1000개 Unacked
  Queue에서 새 메시지 못 받음 → 1001번째 메시지는 대기

원인 3: Consumer 크래시 (Ack 못 보냄)

  OOM Kill, Segfault 등으로 Consumer 비정상 종료
  Connection 끊김 → Unacked 자동 복귀 (Ready)
  재연결 후 재처리

=== 진단 ===

# Unacked 수 확인
rabbitmqctl list_queues name messages_ready messages_unacknowledged
# order.queue  0  50000  ← Ready는 없는데 Unacked 5만 건

# Consumer 연결 상태 확인
rabbitmqctl list_consumers
# order.queue  consumer-1  prefetch=50  active=true
# order.queue  consumer-2  prefetch=50  active=true
# 모두 active → Consumer는 살아있지만 Ack 안 보냄

# Consumer 로그에서 블로킹 확인:
# grep "Waiting\|timeout\|blocked" consumer.log
# → "Waiting for external API response..." 무한 반복

=== Consumer 블로킹 감지 ===

// Unacked 감지 알림 (Prometheus):
rabbitmq_queue_messages_unacked > 10000
→ 15분 이상 지속 시 알림

// 애플리케이션 레벨 타임아웃 강제:
@RabbitListener(queues = "order.queue")
public void handle(Message msg, Channel ch, @Header(AmqpHeaders.DELIVERY_TAG) long tag) {
  CompletableFuture<Void> future = CompletableFuture.runAsync(() -> {
    externalApi.call(msg);  // 블로킹 가능한 작업
  });
  
  try {
    future.get(5, TimeUnit.SECONDS);  // 5초 타임아웃
    ch.basicAck(tag, false);
  } catch (TimeoutException e) {
    future.cancel(true);
    ch.basicNack(tag, false, true);  // 재큐 (일시적 오류)
  }
}

=== 임시 처치 ===

1. 블로킹 Consumer 재시작:
   kubectl rollout restart deployment/order-consumer
   // Connection 끊김 → 모든 Unacked → Ready 복귀

2. Unacked 확인 후 정상인지 판단:
   재시작 후: Unacked 즉시 0이 됨 → 복구 확인
   다시 쌓이면 → 근본 원인 미해결

=== 근본 해결 ===

1. 모든 외부 I/O에 타임아웃 설정:
   HttpClient: connectTimeout=3s, readTimeout=5s
   DB: queryTimeout=10s, connectionTimeout=5s
   외부 API: 5초 타임아웃 + Circuit Breaker

2. Prefetch 감소:
   Prefetch=1000 → Prefetch=10
   Consumer 처리 속도에 맞게 조정

3. 처리 비동기화:
   메시지 수신 → 즉시 Ack → 비동기 처리
   단, 처리 실패 시 재발행 로직 추가 필요
```

### 문제 패턴 C — 연결 폭풍(Connection Storm)

```
=== 발생 원인 ===

시나리오:
  RabbitMQ 재시작 (유지보수, 장애 복구)
  서비스 100개 × 평균 5개 Connection = 500개 연결
  모든 서비스가 "연결 끊겼다" 감지 → 즉시 재연결 시도
  → 500개 Connection 동시 생성 요청

브로커 영향:
  동시 500개 Connection 생성 처리
  각 Connection: TLS Handshake, AMQP 협상, Exchange/Queue 선언
  → CPU 스파이크, 메모리 급증
  → 일부 Connection 생성 실패 → 또 재시도 → 폭풍 증폭

=== 재연결 패턴 비교 ===

나쁜 패턴 (즉시 재연결):
  while (true) {
    try {
      connect();
      break;
    } catch (Exception e) {
      Thread.sleep(1000);  // 고정 1초 → 모든 서비스가 1초 후 동시 시도
      continue;
    }
  }

좋은 패턴 (Exponential Backoff + Jitter):
  int attempt = 0;
  while (true) {
    try {
      connect();
      break;
    } catch (Exception e) {
      long delay = Math.min(
        (long)(Math.pow(2, attempt) * 1000),  // 지수 증가: 1,2,4,8,16...초
        30000                                   // 최대 30초
      );
      long jitter = (long)(Math.random() * delay * 0.3);  // ±30% 랜덤
      Thread.sleep(delay + jitter);  // 서비스마다 다른 시간에 재연결
      attempt++;
    }
  }

Spring AMQP 자동 재연결 (기본 적용):
  CachingConnectionFactory:
    setRecoveryInterval(5000)  // 5초 후 재연결 (고정)
  
  → 서비스가 많으면 5초 후 동시 재연결 = 미니 폭풍
  → Recovery Interval에 Jitter 추가 필요

=== Spring AMQP 재연결 Jitter 설정 ===

@Bean
public CachingConnectionFactory connectionFactory() {
  CachingConnectionFactory factory = new CachingConnectionFactory();
  factory.setHost(rabbitHost);
  
  // 재연결 리스너에 Jitter 추가
  factory.addConnectionListener(new ConnectionListener() {
    @Override
    public void onCreate(Connection connection) {
      log.info("RabbitMQ 연결 성공");
    }
    
    @Override
    public void onClose(Connection connection) {
      // 재연결은 Spring이 자동 처리
      // RecoveryInterval + Jitter는 AbstractConnectionFactory 내부 설정
    }
  });
  
  return factory;
}

// 또는 Resilience4j RetryConfig로 재연결 래핑:
RetryConfig config = RetryConfig.custom()
  .maxAttempts(10)
  .waitDuration(Duration.ofSeconds(2))
  .enableExponentialBackoff()
  .exponentialBackoffMultiplier(2)
  .randomizedWaitFactor(0.3)  // Jitter
  .build();

=== 브로커 측 연결 수 제한 ===

rabbitmq.conf:
  # 클라이언트당 최대 채널 수
  channel_max = 2047

  # 최대 Connection 수 (전체)
  connection_max = 1000

  # 소켓 백로그 (연결 대기열)
  tcp_listen_options.backlog = 128

Policy로 vHost별 연결 제한:
  rabbitmqctl set_policy ConnectionLimit ".*" \
    '{"max-connections":500}' --apply-to vhosts
```

---

## 💻 실전 실험

### 실험 1: Queue 적체 진단 스크립트

```bash
#!/bin/bash
# RabbitMQ 빠른 상태 진단

echo "=== Queue 상태 ==="
rabbitmqctl list_queues name messages messages_ready messages_unacknowledged consumers \
  --formatter pretty_table | grep -v "^$"

echo ""
echo "=== 문제 Queue 감지 ==="
# Ready > 10000이거나 Unacked > 1000이거나 Consumer = 0인 Queue
rabbitmqctl list_queues name messages_ready messages_unacknowledged consumers \
  | awk '$2 > 10000 || $3 > 1000 || $4 == 0 {
    if ($4 == 0) type="⚠️ NO CONSUMER"
    else if ($2 > 10000) type="⚠️ QUEUE BUILDUP"
    else if ($3 > 1000) type="⚠️ UNACKED HIGH"
    print type, $0
  }'

echo ""
echo "=== Connection 상태 ==="
rabbitmqctl list_connections name state channels | head -20
```

### 실험 2: Unacked 누적 시뮬레이션

```bash
# 메시지 발행
rabbitmqadmin declare queue name=unack.test durable=true
for i in $(seq 1 10); do
  rabbitmqadmin publish exchange="" routing_key=unack.test payload="msg$i"
done

# Ack 없이 메시지 수신 (Unacked 상태 생성)
rabbitmqadmin get queue=unack.test ackmode=ack_requeue_true count=5

# 상태 확인
rabbitmqctl list_queues name messages_ready messages_unacknowledged
# unack.test  5  5  ← Ready 5, Unacked 5

# rabbitmqadmin 종료 → Connection 끊김 → Unacked 복귀
# 재확인
rabbitmqctl list_queues name messages_ready messages_unacknowledged
# unack.test  10  0  ← 모두 Ready로 복귀
```

### 실험 3: 연결 폭풍 시뮬레이션 및 방지

```python
# 나쁜 패턴: 고정 간격 재연결
import pika, time, random

def connect_bad():
    while True:
        try:
            conn = pika.BlockingConnection(
                pika.ConnectionParameters('localhost'))
            print("연결 성공")
            return conn
        except Exception as e:
            print(f"연결 실패, 1초 후 재시도: {e}")
            time.sleep(1)  # 모든 서비스가 1초 후 동시 재시도

# 좋은 패턴: Exponential Backoff + Jitter
def connect_good():
    attempt = 0
    while True:
        try:
            conn = pika.BlockingConnection(
                pika.ConnectionParameters('localhost'))
            print(f"연결 성공 (시도 {attempt+1}회)")
            return conn
        except Exception as e:
            delay = min(2**attempt, 30)          # 지수 증가, 최대 30초
            jitter = random.uniform(0, delay*0.3) # 30% Jitter
            wait = delay + jitter
            print(f"연결 실패, {wait:.1f}초 후 재시도: {e}")
            time.sleep(wait)
            attempt += 1
```

---

## 📊 Kafka vs RabbitMQ 비교

```
=== 운영 문제 패턴 비교 ===

Queue 적체:
  RabbitMQ: Queue Depth 증가 → 메모리 → Flow Control
  Kafka:    Consumer Lag 증가 → 메모리 이슈 없음 (디스크 기반)
  
  RabbitMQ: 적체 메시지가 메모리 압박 → 추가 문제 유발
  Kafka:    적체해도 안정적 (디스크 저장, 보관 기간만큼)

Unacked 누적:
  RabbitMQ: Consumer 블로킹 → Prefetch 슬롯 고갈 → 새 메시지 불가
  Kafka:    Consumer가 poll()하지 않으면 Lag 증가 (하지만 슬롯 개념 없음)
  
  RabbitMQ: Heartbeat 타임아웃 → Connection 끊김 → Unacked 자동 복귀
  Kafka:    session.timeout.ms 초과 → Consumer Group Rebalancing

연결 폭풍:
  RabbitMQ: Connection Storm → 브로커 과부하
  Kafka:    재연결 시 Rebalancing → 처리 일시 중단 (Connection Storm보다 가벼움)

=== 안정성 비교 ===

메모리 관련 장애:
  RabbitMQ: 메모리 임계값 초과 시 Flow Control → 발행 차단
  Kafka:    디스크 기반 → 메모리 관련 장애 적음

재시작 후 복구:
  RabbitMQ: durable Queue + PERSISTENT → 메시지 보존, Connection Storm 주의
  Kafka:    오프셋 기반 → 재시작 후 마지막 커밋부터 자동 재처리
```

---

## ⚖️ 트레이드오프

```
각 처치의 트레이드오프:

Queue 적체 시 purge:
  ✅ 즉각 해소 (빠른 복구)
  ❌ 메시지 영구 삭제 → 비즈니스 영향
  → 비즈니스 판단 후 결정 (로그성 메시지 OK, 주문/결제 NG)

Consumer 재시작:
  ✅ Unacked 즉시 복귀 (간단한 처치)
  ❌ 재시작 중 처리 중단 (짧게)
  ❌ 근본 원인 미해결 → 재발
  → 임시 처치로만 사용, 근본 해결 병행 필수

연결 수 제한 (max-connections):
  ✅ 폭풍 완화 (브로커 보호)
  ❌ 초과 시 Connection 생성 실패 → 서비스 오류
  → 임계값 신중하게 설정
```

---

## 📌 핵심 정리

```
문제 패턴별 진단과 해결:

패턴 A (Queue 적체):
  진단: list_queues → Ready 급증 + Consumer 있음
  임시: Consumer Scale Out
  근본: 처리 속도 최적화 / 발행 속도 제한

패턴 B (Unacked 누적):
  진단: list_queues → Unacked 급증, Ready ≈ 0
  임시: Consumer 재시작 (Unacked → Ready 복귀)
  근본: 외부 I/O 타임아웃 설정, Prefetch 감소

패턴 C (연결 폭풍):
  진단: list_connections → 수백~수천 Connection 동시 생성
  임시: 순차 재시작 (동시 재시작 금지)
  근본: Exponential Backoff + Jitter 재연결 전략

공통 도구:
  rabbitmqctl list_queues (상태)
  rabbitmqctl list_consumers (Consumer)
  rabbitmqctl list_connections (Connection)
  Management UI → Overview (실시간 차트)
```

---

## 🤔 생각해볼 문제

**Q1.** Queue 적체가 발생했을 때 `rabbitmqctl purge_queue`를 실행하기 전에 확인해야 할 것은 무엇인가?

<details>
<summary>해설 보기</summary>

**purge_queue는 모든 메시지를 복구 불가능하게 삭제합니다.** 반드시 다음을 확인해야 합니다.

1. **메시지의 비즈니스 중요도**
   - 주문, 결제, 재고: 절대 purge 금지 (비즈니스 손실)
   - 로그, 메트릭, 알림: purge 가능 (재처리 불필요)

2. **DLX 설정 여부**
   - DLX 없이 purge: 영구 소실
   - 가능하면 Consumer를 DLQ로 라우팅하여 메시지 보존

3. **메시지 내용 샘플링**
   ```bash
   rabbitmqadmin get queue=order.queue ackmode=ack_requeue_true count=10
   # 메시지 내용 확인 후 purge 여부 결정
   ```

4. **적체 원인 해결 여부**
   - 원인 해결 없이 purge → 다시 쌓임

5. **관계자 승인**
   - 비즈니스 팀과 합의 후 실행

대안: purge 대신 임시 Consumer를 붙여 DLQ로 이동한 후 나중에 재처리.

</details>

---

**Q2.** Unacked 메시지가 갑자기 100,000개로 증가했다. Consumer는 살아있고 Queue Ready는 0이다. Consumer를 재시작하면 안 되는 이유가 있는가?

<details>
<summary>해설 보기</summary>

**Consumer 재시작이 위험할 수 있는 경우:**

1. **처리 중인 작업이 완료 직전인 경우**
   - 재시작 → 처리 중인 100,000개 Unacked 작업 중단
   - 재전달 후 처음부터 다시 처리 → 시간 낭비, 중복 부작용

2. **비멱등성 Consumer**
   - 이미 외부 API 호출 완료 (결제 청구됨), 하지만 Ack 못 보냄
   - 재시작 → 재전달 → 같은 결제 다시 청구 (이중 결제)
   - 멱등성 없는 Consumer는 재시작 전 부작용 확인 필요

3. **처리 상태 확인 먼저**
   ```bash
   # Consumer 로그 확인 (현재 무엇을 하고 있는가)
   kubectl logs -f order-consumer-pod
   
   # 처리 중인 메시지 ID 확인 (가능하면)
   # Prometheus: consumer_processing_count, consumer_last_processed_id
   ```

**재시작이 안전한 경우:**
- Consumer가 완전히 멈춤 (CPU 0%, 응답 없음)
- Consumer 처리 로직이 멱등적
- 외부 작업이 트랜잭션 안에서 Ack와 함께 처리됨

**권장 순서:**
1. Consumer 로그 확인 (무엇을 하는 중인가)
2. 처리 중 메시지 비즈니스 영향 파악
3. 안전한 경우에만 재시작

</details>

---

**Q3.** 연결 폭풍이 발생했을 때 브로커 측에서 즉시 취할 수 있는 임시 처치는 무엇인가?

<details>
<summary>해설 보기</summary>

**브로커 측 즉시 처치 (서비스 코드 변경 없이):**

1. **Connection 수 제한 (실시간 적용)**
   ```bash
   # 새 Connection 생성을 제한 (기존 Connection 영향 없음)
   rabbitmqctl set_vhost_limits / '{"max-connections": 200}'
   # 200개 초과 시 신규 Connection 거부 → 클라이언트가 대기 후 재시도
   ```

2. **소켓 백로그 확인**
   ```bash
   # rabbitmq.conf:
   tcp_listen_options.backlog = 512  # 기본 128 → 증가
   # 대기 중인 연결 요청을 더 많이 수용
   ```

3. **느린 서비스 Connection 강제 종료**
   ```bash
   # 특정 서비스의 과도한 Connection 닫기
   rabbitmqadmin -V / list connections name | grep "service-name" | \
     awk '{print $2}' | xargs -I{} rabbitmqadmin close connection name={}
   ```

4. **메모리 확보 (Flow Control 방지)**
   ```bash
   # 임시로 임계값 낮춰 Flow Control 빠르게 해제
   rabbitmqctl set_vm_memory_high_watermark 0.5
   ```

**서비스 측에서 병행:**
- 재배포 없이 가능: 환경 변수로 재연결 간격 설정
  ```yaml
  RABBITMQ_RECONNECT_DELAY: "10000"  # 10초로 강제
  ```
- 비상용: 일부 서비스 일시 중단으로 연결 수 감소

</details>

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[이전: 모니터링 핵심 지표 ⬅️](./03-monitoring.md)** | **[다음: 클러스터 운영 ➡️](./05-cluster-operations.md)**

</div>
