# Spring AMQP 아키텍처 — RabbitTemplate, @RabbitListener, RabbitAdmin

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `RabbitTemplate`은 어떻게 Connection/Channel을 관리하는가?
- `@RabbitListener`는 내부적으로 어떤 Container를 사용하는가?
- `SimpleMessageListenerContainer`와 `DirectMessageListenerContainer`의 차이는?
- `RabbitAdmin`은 Exchange/Queue/Binding을 언제 어떻게 선언하는가?
- Spring AMQP의 전체 컴포넌트가 어떻게 연결되는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

Spring AMQP를 쓰면서도 내부 구조를 모르면, `@RabbitListener` 오류가 왜 나는지, 채널이 왜 고갈되는지, RabbitAdmin이 왜 Exchange를 재선언하는지 이해할 수 없다. 컴포넌트 역할과 상호작용을 이해해야 올바른 설정과 장애 원인 파악이 가능하다.

---

## 😱 흔한 실수 (Before)

```
실수 1: Connection을 매 요청마다 생성

  // 잘못된 패턴 (새 Connection 생성)
  ConnectionFactory cf = new CachingConnectionFactory("rabbitmq");
  Connection conn = cf.createConnection();
  Channel channel = conn.createChannel(false);
  channel.basicPublish(...);
  channel.close();
  conn.close();

  → Connection 생성/해제 오버헤드 (TLS Handshake 포함)
  → 올바른 방법: ConnectionFactory Bean을 싱글톤으로, Channel은 CachingConnectionFactory가 재사용

실수 2: SMLC vs DMLC 차이 모르고 선택

  @RabbitListener(queues = "order.queue")
  public void handle(OrderEvent event) {
    // SMLC와 DMLC 중 어떤 것인지 모른 채 사용
    // concurrency 설정이 두 Container에서 동작이 다름
  }

실수 3: RabbitAdmin이 앱 시작마다 Exchange 재선언

  // 이미 서버에 Exchange/Queue 있는데
  // 앱 재시작 때마다 declare 시도 → 로그에 warn 가득
  // 실제로는 idempotent하므로 기능 문제는 없지만 설계 이해 부족
```

---

## ✨ 올바른 접근 (After — 컴포넌트 역할 이해)

```
Spring AMQP 핵심 컴포넌트:

1. CachingConnectionFactory (연결 관리)
   ConnectionPool과 Channel Pool 담당
   
2. RabbitTemplate (발행)
   Exchange에 메시지 발행
   Channel Pool에서 Channel 빌려 사용 후 반납
   
3. SimpleRabbitListenerContainerFactory (소비 설정)
   @RabbitListener의 Container 설정 (Concurrency, Prefetch 등)
   
4. MessageListenerContainer (소비 실행)
   Consumer 스레드 관리, Ack 처리
   
5. RabbitAdmin (선언)
   Exchange/Queue/Binding 자동 선언
   Connection 이벤트 시 재선언
```

---

## 🔬 내부 동작 원리

### 1. CachingConnectionFactory — Connection/Channel 관리

```
=== Connection Pool 구조 ===

CachingConnectionFactory:
  connectionCacheSize: Connection 캐시 수 (기본 1)
  channelCacheSize: Channel 캐시 수 (기본 25)
  cacheMode: CHANNEL (기본) 또는 CONNECTION

CHANNEL 모드 (기본):
  하나의 Connection → 여러 Channel 재사용
  
  Connection (1개)
    └── Channel Cache (최대 25개)
          ├── Channel 1 (RabbitTemplate 사용 후 반납)
          ├── Channel 2 (사용 중)
          └── ...

CONNECTION 모드:
  여러 Connection 사용 (고처리량 환경)
  각 Connection이 Channel 캐시 보유

=== Channel 생애주기 ===

RabbitTemplate.send() 호출:
  1. Channel Cache에서 유효한 Channel 획득
     (캐시 없으면 Connection에서 새 Channel 생성)
  2. basicPublish() 실행
  3. Channel → Cache 반납 (close()가 아닌 return)

Cache에서 Channel이 없을 때:
  → 새 Channel 생성 (RabbitMQ에 channel.open 요청)
  → 사용 후 Cache에 추가 (Cache 용량 초과 시 실제 close)

설정:
  @Bean
  public CachingConnectionFactory connectionFactory() {
    CachingConnectionFactory factory = new CachingConnectionFactory("rabbitmq");
    factory.setChannelCacheSize(50);      // Channel Cache 크기
    factory.setConnectionCacheSize(5);   // CONNECTION 모드 시 Connection 수
    factory.setPublisherConfirmType(CachingConnectionFactory.ConfirmType.CORRELATED);
    factory.setPublisherReturns(true);
    return factory;
  }
```

### 2. RabbitTemplate — 발행 컴포넌트

```
=== RabbitTemplate 주요 메서드 ===

발행:
  convertAndSend(exchange, routingKey, object)
  → MessageConverter로 Object → Message 변환
  → Channel 획득 → basicPublish → Channel 반납

발행 + 변환:
  send(exchange, routingKey, message)
  → MessageConverter 없이 Message 직접 전송

요청-응답:
  convertSendAndReceive(exchange, routingKey, object)
  → DirectReplyTo Queue로 응답 대기
  → 응답 수신 후 반환 (타임아웃: replyTimeout)

Publisher Confirm:
  setConfirmCallback(ConfirmCallback)
  → Ack/Nack 비동기 통보

Return 콜백:
  setReturnsCallback(ReturnsCallback)
  setMandatory(true)
  → 라우팅 실패 메시지 반환

=== MessagePostProcessor ===

발행 전 메시지 변환:
  rabbitTemplate.convertAndSend(exchange, key, payload,
    message -> {
      message.getMessageProperties().setPriority(10);
      message.getMessageProperties().setHeader("x-delay", 5000);
      return message;
    }
  );

  사용 사례:
    - Priority 설정
    - 지연 메시지 x-delay 헤더
    - Correlation ID 추가
    - 압축 처리
```

### 3. SMLC vs DMLC — Consumer Container

```
=== SimpleMessageListenerContainer (SMLC) ===

특성:
  Consumer 스레드와 Channel이 1:1 대응
  concurrency=5 → 5개 스레드 × 5개 Channel

장점:
  - 설정 단순
  - Concurrency 동적 조정 가능 (min~max)
  
단점:
  - 스레드마다 Channel → 높은 concurrency 시 Channel 과다
  - 스레드 블로킹 시 해당 Channel도 블로킹

적합한 경우:
  - 일반적인 Consumer (기본 선택)
  - concurrency가 높지 않은 경우 (1~20)

=== DirectMessageListenerContainer (DMLC) ===

특성:
  Channel당 Consumer가 아닌 Channel 이벤트 루프 기반
  concurrency가 아닌 consumersPerQueue 설정

@Bean
public DirectRabbitListenerContainerFactory directFactory(ConnectionFactory cf) {
  DirectRabbitListenerContainerFactory factory = new DirectRabbitListenerContainerFactory();
  factory.setConnectionFactory(cf);
  factory.setConsumersPerQueue(3);  // 큐당 Consumer 수
  factory.setAcknowledgeMode(AcknowledgeMode.MANUAL);
  return factory;
}

장점:
  - Channel 이벤트 기반 → 스레드 효율적
  - 높은 concurrency에서 SMLC보다 효율적
  
단점:
  - 설정이 조금 다름 (consumersPerQueue vs concurrency)
  - 동적 concurrency 미지원

적합한 경우:
  - 높은 concurrency (> 20)
  - 스레드 효율이 중요한 경우

=== 비교 ===

특성                  | SMLC              | DMLC
─────────────────────┼───────────────────┼──────────────────
스레드 모델           | 스레드당 Channel   | 이벤트 루프
동적 Concurrency      | 지원 (min~max)     | 미지원
Concurrency 설정       | concurrency        | consumersPerQueue
스레드 수 = Channel 수 | Yes               | No (더 효율적)
Spring Boot 기본      | Yes               | 명시적 설정 필요

기본 선택: SMLC
고성능 필요: DMLC 검토
```

### 4. RabbitAdmin — Exchange/Queue/Binding 선언

```
=== RabbitAdmin 동작 원리 ===

역할:
  애플리케이션 컨텍스트의 Exchange, Queue, Binding Bean을 
  자동으로 RabbitMQ 서버에 선언

선언 시점:
  1. 앱 시작 시 (Connection 최초 생성 시)
  2. Connection 재연결 시 (네트워크 장애 후 복구)
  3. 직접 호출: rabbitAdmin.declareQueue(queue)

멱등성:
  이미 존재하는 Exchange/Queue라면 → 에러 없이 넘어감
  (서버의 기존 설정과 다른 속성이면 에러 발생)

@Bean
public RabbitAdmin rabbitAdmin(ConnectionFactory connectionFactory) {
  RabbitAdmin admin = new RabbitAdmin(connectionFactory);
  admin.setAutoStartup(true);   // 앱 시작 시 자동 선언
  return admin;
}

// 또는 Spring Boot에서 자동 설정 (별도 Bean 불필요)

=== Bean 선언과 자동 등록 ===

@Configuration
public class RabbitConfig {
  
  @Bean
  public Queue orderQueue() {
    return QueueBuilder.durable("order.queue")
      .deadLetterExchange("dlx.exchange")
      .build();
    // RabbitAdmin이 이 Bean을 감지 → 서버에 선언
  }
  
  @Bean
  public TopicExchange orderExchange() {
    return ExchangeBuilder.topicExchange("order.exchange")
      .durable(true).build();
  }
  
  @Bean
  public Binding orderBinding(Queue orderQueue, TopicExchange orderExchange) {
    return BindingBuilder.bind(orderQueue).to(orderExchange).with("order.#");
  }
}

// RabbitAdmin 선언 순서:
// Exchange → Queue → Binding (의존성 순서)

=== 선언 실패 처리 ===

기존 Queue와 속성 불일치 시:
  // 서버에 order.queue가 durable=false로 이미 존재
  // Bean으로 durable=true 선언 시
  → IOException: PRECONDITION_FAILED - inequivalent arg
  → 앱 시작 실패

해결:
  1. RabbitMQ Management UI에서 기존 Queue 삭제 후 재시작
  2. 또는 Bean 속성을 서버와 일치시킴
  3. 또는 RabbitAdmin.declareQueue() 호출 시 예외 처리
```

---

## 💻 실전 실험

### 실험 1: 컴포넌트 연결 전체 설정

```java
@Configuration
public class RabbitConfig {

  // 1. ConnectionFactory (연결 관리)
  @Bean
  public CachingConnectionFactory connectionFactory() {
    CachingConnectionFactory factory = new CachingConnectionFactory();
    factory.setHost("rabbitmq");
    factory.setPort(5672);
    factory.setUsername("admin");
    factory.setPassword("password");
    factory.setChannelCacheSize(25);  // Channel 캐시
    factory.setPublisherConfirmType(ConfirmType.CORRELATED);  // Confirm
    factory.setPublisherReturns(true);
    return factory;
  }

  // 2. RabbitTemplate (발행)
  @Bean
  public RabbitTemplate rabbitTemplate(ConnectionFactory cf, Jackson2JsonMessageConverter converter) {
    RabbitTemplate template = new RabbitTemplate(cf);
    template.setMessageConverter(converter);
    template.setMandatory(true);
    template.setConfirmCallback((correlation, ack, cause) -> {
      if (!ack) log.error("Nack: {}", cause);
    });
    template.setReturnsCallback(r -> log.error("Returned: {}", r.getRoutingKey()));
    template.setReplyTimeout(5000);  // RPC 타임아웃
    return template;
  }

  // 3. Container Factory (소비 설정)
  @Bean
  public SimpleRabbitListenerContainerFactory rabbitListenerContainerFactory(
      ConnectionFactory cf, Jackson2JsonMessageConverter converter) {
    SimpleRabbitListenerContainerFactory factory = new SimpleRabbitListenerContainerFactory();
    factory.setConnectionFactory(cf);
    factory.setMessageConverter(converter);
    factory.setAcknowledgeMode(AcknowledgeMode.MANUAL);
    factory.setPrefetchCount(10);
    factory.setConcurrentConsumers(3);
    factory.setMaxConcurrentConsumers(10);
    return factory;
  }

  // 4. MessageConverter
  @Bean
  public Jackson2JsonMessageConverter messageConverter() {
    return new Jackson2JsonMessageConverter();
  }

  // 5. RabbitAdmin (자동 선언)
  @Bean
  public RabbitAdmin rabbitAdmin(ConnectionFactory cf) {
    return new RabbitAdmin(cf);
  }
}
```

### 실험 2: Channel 캐시 동작 확인

```bash
# Channel 사용량 모니터링
rabbitmqctl list_channels
# connection              channel_number  prefetch_count
# <rabbit@node>:conn-1    1               10
# <rabbit@node>:conn-1    2               10
# ...

# Channel Cache 고갈 테스트
# channelCacheSize=5, 동시 발행 스레드 10개 → 5개 캐시 후 새 Channel 생성
# 처리 완료 후 Cache에 반납 (최대 channelCacheSize까지만)
```

---

## 📊 Kafka vs RabbitMQ 비교

```
=== Spring 라이브러리 비교 ===

Spring AMQP (RabbitMQ):
  핵심: RabbitTemplate, @RabbitListener, RabbitAdmin
  Connection: CachingConnectionFactory (Channel Pool)
  Consumer: SMLC 또는 DMLC
  
Spring Kafka:
  핵심: KafkaTemplate, @KafkaListener, AdminClient
  Connection: KafkaProducer/Consumer (Thread-safe, 별도 Pool 불필요)
  Consumer: KafkaMessageListenerContainer

차이:
  RabbitMQ: Channel 재사용이 핵심 (Connection 하나에 N개 Channel)
  Kafka: Producer/Consumer는 각자 Connection (Thread-safe)
  
  RabbitMQ Channel Pool: 경쟁 최소화 (동시 발행 시 Channel 공유)
  Kafka Producer: Thread-safe (하나의 KafkaTemplate 공유)
```

---

## ⚖️ 트레이드오프

```
SMLC vs DMLC:

SMLC:
  ✅ Spring Boot 기본, 문서 풍부
  ✅ 동적 Concurrency 지원
  ❌ 스레드 수 = Channel 수 (높은 concurrency 시 비효율)

DMLC:
  ✅ Channel 이벤트 기반 (스레드 효율적)
  ✅ 높은 concurrency에서 리소스 절약
  ❌ 동적 Concurrency 미지원
  ❌ 설정 다름 (consumersPerQueue)

Channel Cache 크기:
  크면: 재사용 높음, 메모리 사용 증가
  작으면: Channel 생성 빈번, 오버헤드
  권장: (예상 동시 Publisher 스레드 수) × 1.5
```

---

## 📌 핵심 정리

```
Spring AMQP 컴포넌트 핵심:

CachingConnectionFactory:
  Connection + Channel Pool 관리
  channelCacheSize: Channel 재사용 수

RabbitTemplate:
  발행, RPC, Publisher Confirm, Return 콜백
  Channel Pool에서 빌려 사용

Container Factory (SMLC/DMLC):
  @RabbitListener의 Consumer 실행 환경
  Prefetch, Concurrency, AcknowledgeMode 설정

RabbitAdmin:
  앱 시작/재연결 시 Exchange/Queue/Binding 자동 선언
  Bean 정의를 서버에 동기화

SMLC vs DMLC:
  일반: SMLC (기본)
  고성능: DMLC (스레드 효율)
```

---

## 🤔 생각해볼 문제

**Q1.** `channelCacheSize=25`로 설정됐는데, 동시에 50개의 RabbitTemplate.send() 호출이 발생했다. 어떻게 되는가?

<details>
<summary>해설 보기</summary>

**캐시에 없는 25개는 새 Channel을 생성합니다.**

동작:
- Cache에 25개 Channel이 있음
- 50개 동시 호출 → 25개는 Cache에서 획득, 25개는 새 Channel 생성 (Connection.createChannel())
- 사용 완료 후:
  - Cache 공간이 남으면 → Cache에 반납
  - Cache가 가득 차면 (25개 한도) → 실제 close()

결과:
- 순간적으로 50개 Channel 사용 (RabbitMQ에서 확인 가능)
- 처리 완료 후 25개는 Cache에, 25개는 닫힘
- 다음 호출 시 캐시된 25개 재사용

권장: `channelCacheSize`를 예상 동시 호출 수에 맞게 설정. 너무 작으면 Channel 생성/해제 오버헤드 반복.

</details>

---

**Q2.** RabbitAdmin이 앱 재시작 시마다 Exchange를 재선언한다. 이것이 성능 문제를 일으킬 수 있는가?

<details>
<summary>해설 보기</summary>

**일반적으로 성능 문제가 없습니다.** RabbitAdmin의 선언은 멱등적이고 빠릅니다.

동작:
- 앱 시작 → Connection 연결 → RabbitAdmin: Exchange/Queue/Binding 선언
- 이미 동일한 속성으로 존재하면 → AMQP passiveDeclare (확인만, 재생성 없음)
- 선언 수십~수백 개도 수십 ms 이내 완료

**성능 문제가 생기는 경우:**
1. 선언이 수천 개인 경우 (Queue가 매우 많음)
2. 선언 속성이 서버와 불일치 → PRECONDITION_FAILED 예외 → 재선언 반복

**최적화 (필요 시):**
```java
// 앱 시작 시만 선언 (재연결 시 재선언 안 함)
@Bean
public RabbitAdmin rabbitAdmin(ConnectionFactory cf) {
  RabbitAdmin admin = new RabbitAdmin(cf);
  admin.setAutoStartup(true);
  // reconnect 시 재선언하지 않으려면: admin.afterPropertiesSet() 후 선언만
  return admin;
}
```

하지만 재선언은 보통 안전하게 두는 것이 권장됩니다. 재연결 후 Exchange/Queue가 없으면 발행 실패하기 때문입니다.

</details>

---

**Q3.** SMLC에서 `setConcurrentConsumers(3)` 설정 시 Consumer 스레드가 3개 생성된다. 이 스레드들은 각각 별도의 Channel을 사용하는가, 아니면 Channel을 공유하는가?

<details>
<summary>해설 보기</summary>

**각 스레드가 별도의 Channel을 사용합니다.**

SMLC 내부 구조:
```
SMLC (concurrency=3)
├── Consumer Thread 1 → Channel 1 → basicConsume(order.queue)
├── Consumer Thread 2 → Channel 2 → basicConsume(order.queue)
└── Consumer Thread 3 → Channel 3 → basicConsume(order.queue)
```

각 스레드는:
- 독립적인 Channel에 `basicConsume` 등록
- Prefetch가 Channel 레벨에서 설정 (`basicQos`)
- Ack도 각 Channel을 통해 전송

Channel 공유가 안 되는 이유:
- AMQP Channel은 `deliveryTag`가 Channel 범위 내에서만 유효
- 다른 Channel의 deliveryTag로 Ack 불가
- 각 Consumer가 독립적으로 Ack/Nack를 처리하려면 독립 Channel 필요

결과:
- `concurrentConsumers=3` → RabbitMQ에서 3개 Consumer 등록 (`list_consumers` 확인)
- 각 Consumer가 Prefetch만큼 메시지 보유
- `channelCacheSize`가 Consumer Channel 수보다 작아도 무방 (Consumer Channel은 캐시와 별도)

</details>

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[이전: Chapter 6 — 함께 쓰는 경우 ⬅️](../rabbitmq-vs-kafka/04-using-both.md)** | **[다음: 메시지 직렬화 전략 ➡️](./02-message-serialization.md)**

</div>
