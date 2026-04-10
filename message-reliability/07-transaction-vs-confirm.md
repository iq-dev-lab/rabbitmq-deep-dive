# 트랜잭션 vs Publisher Confirm — 완전한 메시지 처리 보장

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- AMQP 트랜잭션(`txSelect`/`txCommit`)은 어떻게 동작하고, 성능 비용은 얼마인가?
- Publisher Confirm이 트랜잭션보다 선호되는 이유는 무엇인가?
- DB 트랜잭션과 메시지 발행을 원자적으로 처리하는 Outbox Pattern이란?
- Outbox + Publisher Confirm 조합으로 완전한 메시지 보장을 달성하는 방법은?
- 두 메커니즘의 성능과 보장 수준의 트레이드오프는?

---

## 🔍 왜 이 개념이 실무에서 중요한가

"DB에 주문을 저장하고, RabbitMQ에 메시지를 발행했다"는 두 작업이 원자적으로 성공해야 하는 경우가 있다. DB는 성공했지만 메시지 발행이 실패하면? 또는 메시지는 발행됐지만 DB 저장이 롤백되면? AMQP 트랜잭션은 메시지 발행의 원자성을 보장하지만 성능 비용이 크다. Outbox Pattern은 DB 트랜잭션과 메시지 발행을 조합하여 완전한 보장을 낮은 성능 비용으로 달성한다. 이 문서는 Chapter 3의 모든 신뢰성 메커니즘을 통합하여 완전한 보장 패턴을 설계한다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: AMQP 트랜잭션으로 처리량 붕괴

  channel.txSelect();
  try {
    channel.basicPublish(..., message1);
    channel.basicPublish(..., message2);
    channel.txCommit();  // 동기 대기: 브로커가 커밋 완료 확인 후 반환
  } catch (Exception e) {
    channel.txRollback();
  }
  
  측정 결과:
    트랜잭션 없음: 10,000 msg/sec
    AMQP 트랜잭션: 400~600 msg/sec (약 20~25배 감소)
  
  이유:
    txCommit 시 브로커가 동기적으로 fsync 후 응답
    각 트랜잭션마다 Round-Trip 1회 추가

실수 2: DB 저장 후 메시지 발행 (원자성 없음)

  @Transactional
  public void placeOrder(OrderRequest req) {
    orderRepository.save(order);          // DB 저장 성공
    rabbitTemplate.convertAndSend(...);   // 메시지 발행 실패 (네트워크 오류)
    // @Transactional은 DB만 롤백 (RabbitMQ는 이미 시도됨)
  }
  
  문제:
    DB는 롤백됐지만 메시지는 이미 발행됐을 수도 있음
    DB는 성공했지만 메시지 발행 실패 → Consumer 처리 없음
    두 경우 모두 데이터 불일치 발생

실수 3: @Transactional + RabbitMQ Channel Transaction 혼용 착각

  "@Transactional이 메시지 발행도 롤백해줄 것"으로 착각
  
  실제:
    Spring @Transactional은 DB 트랜잭션만 관리
    RabbitMQ Channel Transaction은 별도 (AMQP txSelect/txCommit)
    두 트랜잭션은 2PC(Two-Phase Commit) 없이 독립적
```

---

## ✨ 올바른 접근 (After — Outbox + Publisher Confirm)

```
완전한 메시지 보장 패턴: Outbox Pattern

핵심 아이디어:
  DB 저장과 메시지 발행을 같은 DB 트랜잭션 안에서 처리
  메시지를 직접 RabbitMQ에 발행하지 않고,
  먼저 DB의 outbox 테이블에 저장

흐름:
  1. @Transactional 시작
  2. orderRepository.save(order)       → orders 테이블
  3. outboxRepository.save(outboxEvent) → outbox 테이블
  4. @Transactional 커밋 (두 저장이 원자적)
  
  5. 별도 스케줄러/이벤트:
     outbox 테이블에서 미발행 메시지 조회
  6. rabbitTemplate + Publisher Confirm으로 발행
  7. Ack 수신 → outbox 상태 PUBLISHED로 업데이트
  8. Nack → 재발행 시도

보장:
  DB 저장 + outbox 저장: 원자적 (같은 트랜잭션)
  RabbitMQ 발행: Publisher Confirm으로 보장
  발행 실패: outbox에 남아있으므로 재발행 가능
```

---

## 🔬 내부 동작 원리

### 1. AMQP 트랜잭션의 동작과 성능 비용

```
=== AMQP 트랜잭션 프로토콜 ===

1. 트랜잭션 시작:
   Client → Server: Tx.Select
   Server → Client: Tx.SelectOk

2. 메시지 발행 (트랜잭션 내):
   Client → Server: Basic.Publish (msg1) ← 즉시 전송
   Client → Server: Basic.Publish (msg2) ← 즉시 전송
   (브로커는 트랜잭션 버퍼에 보관, Queue에 아직 미적용)

3. 커밋:
   Client → Server: Tx.Commit
   Server: 트랜잭션 버퍼의 메시지들을 Queue에 적용, fsync
   Server → Client: Tx.CommitOk  ← 동기 대기 포인트

4. 롤백:
   Client → Server: Tx.Rollback
   Server: 트랜잭션 버퍼 폐기
   Server → Client: Tx.RollbackOk

=== 성능 비용 분석 ===

트랜잭션 없이 발행:
  Client → Server: Publish(msg1)
  Client → Server: Publish(msg2)
  ...계속 발행...

  Client는 Server 응답을 기다리지 않음 (Fire and Forget)
  처리량: 브로커 수신 속도에만 의존 → 최대 처리량

트랜잭션 Commit:
  Client → Server: Publish들...
  Client → Server: Tx.Commit
  [동기 대기: 브로커가 fsync 완료 후 CommitOk 전송]
  Client: CommitOk 수신 후에야 다음 트랜잭션 시작 가능

  Round-Trip 1회 × 100ms (fsync 포함) = 최대 10 tx/sec
  각 트랜잭션에 10개 메시지 = 100 msg/sec

  단일 메시지 트랜잭션이면:
    Round-Trip 1회 = 최대 10~50 msg/sec
    → 대폭 처리량 감소

=== Publisher Confirm과의 비교 ===

AMQP 트랜잭션:
  Commit 시점: 동기 fsync → 완전한 원자성
  성능: 처리량 20~25배 감소
  롤백: 발행 취소 가능

Publisher Confirm (비동기):
  Ack 시점: Queue 저장 완료 (또는 Quorum Queue면 과반수 WAL)
  성능: 처리량 10% 감소 수준 (비동기이므로)
  롤백: 불가 (Nack 시 재발행으로 대응)

결론:
  완전한 원자성 필요: AMQP 트랜잭션 (성능 포기)
  신뢰성 + 성능 균형: Publisher Confirm
  실무 선택: 거의 항상 Publisher Confirm
```

### 2. DB + 메시지 발행 원자성 문제

```
=== 두 작업의 원자성 문제 ===

시나리오 A: DB 성공, 메시지 발행 실패
  @Transactional
  public void placeOrder(OrderRequest req) {
    orderRepository.save(order);   // DB 커밋됨
    rabbitTemplate.send(...);      // 예외 발생!
    // @Transactional 롤백 시도 → DB 롤백됨
  }
  
  결과:
    DB 롤백 성공 → 주문 없음
    메시지는 발행됐을 수도 있음 (예외 전 전송됐다면)
    → 존재하지 않는 주문에 대한 메시지 처리 시도

시나리오 B: DB 성공, @Transactional 커밋 후 발행 실패
  public void placeOrder(OrderRequest req) {
    order = saveInTransaction(req);  // DB 커밋됨
    rabbitTemplate.send(...);        // 네트워크 오류!
    // DB 롤백 불가 (이미 커밋됨)
  }
  
  결과:
    DB에 주문 있음, RabbitMQ에 메시지 없음
    → Consumer가 주문 처리를 하지 않음
    → 결제, 재고, 알림 등 후속 처리 누락

=== Spring의 @Transactional + RabbitMQ 동기화 시도 ===

RabbitTransactionManager:
  @Transactional(transactionManager = "rabbitTransactionManager")
  → AMQP 트랜잭션과 Spring @Transactional 연결
  
  단점:
    DB 트랜잭션과 AMQP 트랜잭션은 여전히 별개
    하나가 실패해도 다른 하나가 롤백 보장 안 됨
    2PC(2단계 커밋) 없이는 원자성 불가
    2PC: 복잡, 느림, 실무에서 거의 사용 안 함
```

### 3. Outbox Pattern — 완전한 해결책

```
=== Outbox Pattern 상세 설계 ===

DB 스키마:
  CREATE TABLE outbox_events (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    event_type  VARCHAR(100) NOT NULL,          -- "order.placed"
    payload     JSONB NOT NULL,                 -- 이벤트 내용
    exchange    VARCHAR(100) NOT NULL,           -- "order.exchange"
    routing_key VARCHAR(100) NOT NULL,           -- "order.placed"
    status      VARCHAR(20) DEFAULT 'PENDING',  -- PENDING, PUBLISHED, FAILED
    created_at  TIMESTAMP DEFAULT NOW(),
    published_at TIMESTAMP,
    retry_count INT DEFAULT 0
  );

=== 발행 흐름 ===

@Service
public class OrderService {

  @Transactional
  public Order placeOrder(OrderRequest req) {
    // 1. DB에 주문 저장
    Order order = orderRepository.save(new Order(req));
    
    // 2. 같은 트랜잭션에 outbox 이벤트 저장
    OutboxEvent event = OutboxEvent.builder()
      .eventType("order.placed")
      .payload(objectMapper.writeValueAsString(new OrderPlacedEvent(order)))
      .exchange("order.exchange")
      .routingKey("order.placed")
      .status("PENDING")
      .build();
    outboxRepository.save(event);
    
    // 3. 트랜잭션 커밋 → 두 저장이 원자적으로 완료
    return order;
  }
}

=== 발행자 (Outbox Publisher) ===

@Component
public class OutboxPublisher {

  // 방법 1: 스케줄러 기반 폴링
  @Scheduled(fixedDelay = 1000)  // 1초마다
  @Transactional
  public void publishPendingEvents() {
    List<OutboxEvent> pending = outboxRepository
      .findByStatusOrderByCreatedAt("PENDING", PageRequest.of(0, 100));
    
    for (OutboxEvent event : pending) {
      CorrelationData correlation = new CorrelationData(event.getId().toString());
      
      rabbitTemplate.convertAndSend(
        event.getExchange(),
        event.getRoutingKey(),
        event.getPayload(),
        correlation
      );
    }
  }

  // Publisher Confirm 콜백
  @Bean
  public RabbitTemplate rabbitTemplate(ConnectionFactory cf) {
    RabbitTemplate template = new RabbitTemplate(cf);
    
    template.setConfirmCallback((correlation, ack, cause) -> {
      if (correlation == null) return;
      UUID eventId = UUID.fromString(correlation.getId());
      
      if (ack) {
        // 발행 성공 → PUBLISHED 상태로 업데이트
        outboxRepository.updateStatus(eventId, "PUBLISHED", LocalDateTime.now());
      } else {
        // Nack → 재시도 카운트 증가
        outboxRepository.incrementRetryCount(eventId);
        log.error("Nack received for event: {}, cause: {}", eventId, cause);
      }
    });
    
    template.setMandatory(true);
    template.setReturnsCallback(returned -> {
      // 라우팅 실패 → 오류 기록
      log.error("Message returned: {}", returned.getMessage());
    });
    
    return template;
  }
}

=== 방법 2: DB 이벤트 리스너 (Transactional Outbox) ===

// Spring Data JPA @DomainEvents 활용
@Entity
public class Order extends AbstractAggregateRoot<Order> {

  public Order place() {
    // 도메인 이벤트 등록 (트랜잭션 커밋 후 발행)
    registerEvent(new OrderPlacedEvent(this));
    return this;
  }
}

// @TransactionalEventListener → 트랜잭션 커밋 후 실행
@Component
public class OrderEventListener {

  @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
  public void onOrderPlaced(OrderPlacedEvent event) {
    // 커밋 후 RabbitMQ 발행 시도
    // 실패 시 outbox에 저장 (별도 트랜잭션으로)
    try {
      rabbitTemplate.convertAndSend("order.exchange", "order.placed", event);
    } catch (Exception e) {
      outboxRepository.save(new OutboxEvent(event));  // 실패 시 outbox 저장
    }
  }
}
```

### 4. 완전한 메시지 보장 체크리스트

```
=== Chapter 3 전체 통합 ===

지점 1: Publisher → Broker (Publisher Confirm)
  ✅ publisher-confirm-type: correlated
  ✅ ConfirmCallback: Nack 시 outbox 재발행
  ✅ mandatory=true + ReturnsCallback: 라우팅 실패 감지

지점 2: Broker 저장 (영속성)
  ✅ Queue durable=true
  ✅ Message DeliveryMode=PERSISTENT
  ✅ Quorum Queue (클러스터 환경 권장)

지점 3: Consumer 처리 (Consumer Ack)
  ✅ acknowledge-mode: manual
  ✅ 처리 완료 후 basicAck
  ✅ 예외 분류 후 requeue 또는 DLX

DB + 메시지 원자성 (Outbox Pattern):
  ✅ outbox 테이블에 DB 트랜잭션 내 저장
  ✅ 스케줄러/이벤트로 RabbitMQ 발행
  ✅ Publisher Confirm Ack → PUBLISHED 상태 업데이트

재시도 (Ch3-05):
  ✅ TTL + DLX Exponential Backoff
  ✅ 재시도 횟수 제한 (x-death count)
  ✅ 최종 실패 → 영구 DLQ

Consumer 처리량 (Ch3-06):
  ✅ 최적 Prefetch Count 설정
  ✅ Consumer Utilisation 90%+ 목표

=== 성능과 보장 수준 조합 ===

보장 수준       | 설정                              | 처리량
───────────────┼───────────────────────────────────┼──────────
최대 처리량     | TRANSIENT + autoAck + No Confirm   | 100%
                | 유실 허용 (로그, 메트릭)             |
───────────────┼───────────────────────────────────┼──────────
중간 보장       | PERSISTENT + autoAck + No Confirm  | ~80%
                | 재시작 보존, 처리 중 유실 허용       |
───────────────┼───────────────────────────────────┼──────────
At-Least-Once  | PERSISTENT + Manual Ack + Confirm  | ~60%
                | 유실 없음, 중복 가능                |
───────────────┼───────────────────────────────────┼──────────
완전 보장       | Quorum + Outbox + Confirm + Manual | ~40%
                | 유실 없음, 원자성, 재처리             | (추정)
───────────────┼───────────────────────────────────┼──────────
AMQP 트랜잭션  | txSelect + txCommit                | ~5%
                | 완전한 원자성 (실용적으로 사용 금지)  |
```

---

## 💻 실전 실험

### 실험 1: AMQP 트랜잭션 성능 측정

```java
@Component
public class TransactionBenchmark {

  @Autowired
  private ConnectionFactory connectionFactory;

  public void benchmarkTransaction(int messageCount) throws Exception {
    try (Connection conn = connectionFactory.createConnection();
         Channel channel = conn.createChannel(false)) {

      // 트랜잭션 없음
      long start = System.currentTimeMillis();
      for (int i = 0; i < messageCount; i++) {
        channel.basicPublish("", "bench.queue",
          MessageProperties.PERSISTENT_TEXT_PLAIN,
          ("msg" + i).getBytes());
      }
      long noTxTime = System.currentTimeMillis() - start;
      System.out.printf("트랜잭션 없음: %d msg in %dms = %d msg/sec%n",
        messageCount, noTxTime, messageCount * 1000L / noTxTime);

      // 단일 메시지 트랜잭션
      channel.txSelect();
      start = System.currentTimeMillis();
      for (int i = 0; i < messageCount; i++) {
        channel.basicPublish("", "bench.queue",
          MessageProperties.PERSISTENT_TEXT_PLAIN,
          ("msg" + i).getBytes());
        channel.txCommit();  // 메시지마다 커밋
      }
      long txTime = System.currentTimeMillis() - start;
      System.out.printf("단일 메시지 트랜잭션: %d msg in %dms = %d msg/sec%n",
        messageCount, txTime, messageCount * 1000L / txTime);
    }
  }
}

// 예상 결과 (서버 환경에 따라 다름):
// 트랜잭션 없음:     10000 msg in 50ms = 200000 msg/sec
// 단일 메시지 트랜잭션: 10000 msg in 5000ms = 2000 msg/sec (100배 차이)
```

### 실험 2: Outbox Pattern 구현

```java
// outbox_events 테이블
@Entity
@Table(name = "outbox_events")
public class OutboxEvent {
  @Id
  @GeneratedValue
  private UUID id;

  private String eventType;
  private String payload;
  private String exchange;
  private String routingKey;
  private String status = "PENDING";
  private LocalDateTime createdAt = LocalDateTime.now();
  private LocalDateTime publishedAt;
  private int retryCount = 0;
}

// OutboxPublisher 스케줄러
@Slf4j
@Component
@RequiredArgsConstructor
public class OutboxPublisher {

  private final OutboxEventRepository outboxRepository;
  private final RabbitTemplate rabbitTemplate;

  @Scheduled(fixedDelay = 500)  // 0.5초마다
  public void publish() {
    List<OutboxEvent> events = outboxRepository
      .findTop100ByStatusAndRetryCountLessThan("PENDING", 5);

    for (OutboxEvent event : events) {
      try {
        CorrelationData correlation = new CorrelationData(event.getId().toString());
        rabbitTemplate.convertAndSend(
          event.getExchange(),
          event.getRoutingKey(),
          event.getPayload(),
          correlation
        );
      } catch (Exception e) {
        log.error("발행 예외: eventId={}", event.getId(), e);
        outboxRepository.incrementRetryCount(event.getId());
      }
    }
  }
}
```

### 실험 3: 전체 신뢰성 파이프라인 동작 확인

```bash
# 1. Queue 구성 확인
rabbitmqctl list_queues name messages durable
# order.queue  0  true

# 2. 발행 → Publisher Confirm 확인 (로그)
# 2024-03-15 INFO  ConfirmCallback: Ack received for eventId=uuid-1234

# 3. Consumer 처리 확인
# 2024-03-15 INFO  OrderConsumer: Processing order orderId=1
# 2024-03-15 INFO  OrderConsumer: Ack sent for deliveryTag=1

# 4. outbox 상태 확인
# SELECT status, count(*) FROM outbox_events GROUP BY status;
# PUBLISHED  100
# PENDING      0
# FAILED       0

# 5. DLQ 확인 (정상이면 비어야 함)
rabbitmqctl list_queues name messages
# order.permanent.dlq  0
```

---

## 📊 Kafka vs RabbitMQ 비교

```
=== DB + 메시지 원자성 비교 ===

RabbitMQ + Outbox Pattern:
  DB(outbox) → 스케줄러 → RabbitMQ
  outbox 테이블이 중간 저장소 역할
  스케줄러 폴링 주기만큼 지연 (수백 ms~수 초)

Kafka + Outbox Pattern (동일 패턴):
  DB(outbox) → Debezium (CDC) → Kafka
  Debezium: DB WAL을 읽어 Kafka에 자동 발행
  DB 변경 즉시 감지 → 낮은 지연 (수십 ms)
  별도 스케줄러 불필요 (CDC 기반)

Kafka Transactional Producer:
  producer.initTransactions()
  producer.beginTransaction()
  producer.send(...)
  producer.commitTransaction()  // Exactly-Once 보장 (Kafka만)
  
  RabbitMQ: Exactly-Once 미지원
  Kafka: Idempotent Producer + Transactional → Exactly-Once 가능

=== 원자성 보장 비교 ===

완전한 DB + 메시지 원자성:
  RabbitMQ: Outbox Pattern (At-Least-Once + Consumer 멱등성)
  Kafka:    Outbox Pattern 또는 Transactional Producer (Exactly-Once 가능)

실무 선택:
  RabbitMQ: Outbox + Consumer 멱등성으로 충분
  Kafka: Exactly-Once 필요 시 Transactional Producer 사용
```

---

## ⚖️ 트레이드오프

```
각 방법의 트레이드오프:

AMQP 트랜잭션:
  ✅ 완전한 원자성 (발행 롤백 가능)
  ❌ 처리량 25배 이상 감소
  ❌ 실무에서 사용 거의 불가
  → 테스트/학습 목적 외 권장하지 않음

Publisher Confirm (단독):
  ✅ 높은 처리량 (비동기 Confirm)
  ✅ Nack 시 재발행 가능
  ❌ DB + 발행 원자성 없음
  → At-Least-Once, 유실 없는 발행 필요 시

Outbox Pattern + Publisher Confirm:
  ✅ DB + 발행 원자성 보장
  ✅ 발행 실패 시 자동 재시도
  ✅ 처리량 상대적으로 높음 (비동기)
  ❌ 구현 복잡 (outbox 테이블, 스케줄러)
  ❌ 스케줄러 폴링 지연 (수백 ms)
  ❌ DB 부하 증가 (outbox 조회/업데이트)
  → 중요 비즈니스 이벤트에 권장

비교:
  처리량: AMQP 트랜잭션 << Outbox ≈ Publisher Confirm 단독
  보장 수준: AMQP 트랜잭션 > Outbox > Publisher Confirm 단독
  구현 복잡성: AMQP 트랜잭션 < Publisher Confirm < Outbox
```

---

## 📌 핵심 정리

```
트랜잭션 vs Publisher Confirm 핵심:

AMQP 트랜잭션:
  txSelect → 발행들 → txCommit (동기)
  완전한 원자성, 처리량 25배 감소
  → 실무에서 거의 사용 금지

Publisher Confirm:
  비동기 Ack/Nack
  처리량 ~10% 감소, Nack 재발행으로 대응
  → 실무 권장 (단독으로 충분한 경우)

DB + 메시지 원자성 문제:
  @Transactional은 DB만 관리 (RabbitMQ 무관)
  DB 커밋 + 메시지 발행은 원자적으로 불가 (2PC 없이)

Outbox Pattern (권장 해결책):
  1. DB 트랜잭션 내 order + outbox_events 동시 저장
  2. 스케줄러가 outbox 폴링 → RabbitMQ 발행
  3. Publisher Confirm Ack → outbox PUBLISHED 업데이트
  보장: DB + 메시지 원자성 + At-Least-Once

Chapter 3 완전 보장 조합:
  Outbox Pattern + Publisher Confirm (발행)
  + Quorum Queue + PERSISTENT (저장)
  + Manual Ack + DLX (소비)
  + TTL+DLX Backoff (재시도)
```

---

## 🤔 생각해볼 문제

**Q1.** Outbox Pattern에서 스케줄러가 같은 이벤트를 두 번 발행할 수 있는가? 어떻게 방지하는가?

<details>
<summary>해설 보기</summary>

**가능합니다.** 다음 시나리오에서 발생합니다:

1. 스케줄러가 PENDING 이벤트 조회 → 발행
2. Publisher Confirm Ack 수신 → PUBLISHED 업데이트 시도
3. DB 업데이트 실패 (네트워크 오류 등) → 상태 여전히 PENDING
4. 다음 스케줄러 실행 → 같은 이벤트 재발행

**방지 방법**:

**방법 1: 발행 전 상태 변경 (낙관적 락)**
```java
// PENDING → PROCESSING으로 먼저 변경 (발행 전)
int updated = outboxRepository.updateStatusIfPending(eventId, "PROCESSING");
if (updated == 0) return;  // 다른 인스턴스가 이미 처리 중

rabbitTemplate.convertAndSend(...);
// Ack 수신 후 → PUBLISHED
```

**방법 2: Consumer 멱등성 (근본 해결)**
```java
// Consumer에서 이미 처리된 eventId 확인
if (processedEventRepository.existsById(eventId)) {
  channel.basicAck(tag, false);
  return;
}
// 처리 후 processedEventRepository.save(eventId)
```

**방법 3: 분산 Lock**
Redis 기반 Lock으로 같은 이벤트를 한 번에 하나의 인스턴스만 처리

실무 권장: **방법 2 (Consumer 멱등성)**를 기본으로, 필요 시 방법 1 추가.

</details>

---

**Q2.** Outbox 테이블에 수십만 건의 PENDING 이벤트가 쌓였다. 스케줄러 폴링이 따라가지 못하면 어떻게 처리하는가?

<details>
<summary>해설 보기</summary>

**긴급 대응 단계:**

**1단계: 즉각 원인 파악**
- Publisher Confirm Ack가 오지 않는가? → RabbitMQ 연결/처리량 확인
- 스케줄러가 동작하지 않는가? → 스케줄러 상태 확인

**2단계: 스케줄러 처리량 증가**
```java
// 배치 크기 증가
List<OutboxEvent> events = outboxRepository
  .findTop1000ByStatus("PENDING");  // 100 → 1000으로

// 스케줄러 주기 감소
@Scheduled(fixedDelay = 100)  // 1초 → 0.1초로

// 병렬 처리
@Async
@Scheduled(fixedDelay = 500)
public void publishAsync() { ... }  // 여러 스케줄러 인스턴스
```

**3단계: 임시 증속 처리**
```bash
# 별도 스크립트로 PENDING 이벤트 일괄 재발행
# 또는 스케줄러 인스턴스 일시적 Scale Out
```

**4단계: 근본 원인 해결**
- RabbitMQ Consumer 처리량 부족 → Consumer Scale Out
- 발행 병목 → Publisher Confirm 비동기 처리 최적화
- outbox 인덱스 최적화: `CREATE INDEX ON outbox_events(status, created_at)`

**예방적 모니터링**:
```sql
SELECT count(*) FROM outbox_events WHERE status='PENDING' AND created_at < NOW() - INTERVAL '5 seconds';
```
→ 5초 이상 PENDING이 100건 초과 시 알림

</details>

---

**Q3.** 마이크로서비스 A(OrderService)와 마이크로서비스 B(PaymentService)가 각각 DB와 RabbitMQ를 가진다. A가 주문을 저장하고 B에게 결제를 요청한다. B가 결제 처리 후 A에게 결과를 돌려줘야 한다. 이 양방향 흐름을 Outbox Pattern으로 설계하면 어떤 구조인가?

<details>
<summary>해설 보기</summary>

**각 서비스가 독립적인 Outbox를 가지는 구조:**

```
OrderService:
  orders DB + outbox_events DB
  → Outbox Publisher → order.exchange → payment.svc.queue
  → PaymentService Consumer

PaymentService:
  payments DB + outbox_events DB
  → 결제 처리 완료
  → Outbox에 "order.payment.completed" 이벤트 저장
  → Outbox Publisher → payment.exchange → order.payment.queue
  → OrderService Consumer → 주문 상태 업데이트
```

**전체 흐름:**
1. OrderService: DB 트랜잭션으로 `orders` + `outbox(order.placed)` 저장
2. OrderService Outbox Publisher → `payment.svc.queue` 발행
3. PaymentService Consumer: 결제 처리
4. PaymentService: DB 트랜잭션으로 `payments` + `outbox(payment.completed)` 저장
5. PaymentService Outbox Publisher → `order.payment.queue` 발행
6. OrderService Consumer: 주문 상태를 "결제완료"로 업데이트

**핵심 원칙:**
- 각 서비스는 자신의 DB에만 트랜잭션
- 서비스 간 통신은 RabbitMQ로만
- 각 서비스는 자신의 Outbox + Publisher Confirm으로 발행 보장
- Consumer 멱등성으로 중복 처리 방지

이것이 **Saga 패턴 + Outbox Pattern**의 조합입니다.

</details>

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[이전: Prefetch Count 완전 분해 ⬅️](./06-prefetch-count.md)** | **[다음: Chapter 4 — 실무 메시징 패턴 ➡️](../messaging-patterns/01-work-queue.md)**

</div>
