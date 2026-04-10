# 에러 처리와 재시도 — RetryTemplate, DLX, DeadLetterPublishingRecoverer

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `RetryTemplate`은 어떻게 Consumer 재시도를 구현하는가?
- `RejectAndDontRequeueRecoverer`와 `DeadLetterPublishingRecoverer`의 차이는?
- `StatefulRetryOperationsInterceptor`가 필요한 이유는 무엇인가?
- Spring AMQP의 에러 처리 계층 구조는 어떻게 구성되는가?
- `@RabbitListener concurrency` 설정이 에러 처리에 어떤 영향을 주는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

Consumer 예외 처리는 단순해 보이지만 실제로는 여러 선택지가 있다. Spring AMQP가 기본으로 제공하는 재시도 메커니즘을 모르면, 직접 try-catch를 남발하거나 무한 루프를 만든다. RetryTemplate의 한계(스레드 블로킹, 재시작 후 카운트 초기화)를 이해해야 TTL+DLX 방식을 언제 선택할지 알 수 있다.

---

## 😱 흔한 실수 (Before)

```
실수 1: 예외를 throw하면 requeue=true로 무한 루프

  @RabbitListener(queues = "order.queue")
  public void handle(OrderEvent event) {
    orderService.process(event);  // 예외 발생
    // 예외가 Container로 전파됨
  }

  Spring AMQP AUTO 모드 기본:
    예외 발생 → Container: basicNack(requeue=true)
    → 메시지 즉시 재큐 → 같은 예외 → 무한 루프

  해결: AmqpRejectAndDontRequeueException으로 throw
        또는 defaultRequeueRejected=false 설정

실수 2: RetryTemplate을 Stateless로 사용하다 재시작 후 카운트 초기화

  RetryTemplate retry = new RetryTemplate();
  retry.setRetryPolicy(new SimpleRetryPolicy(3));  // 최대 3회

  재시작 시:
    이전 2회 실패 후 재시작 → 카운트 0으로 초기화
    → 다시 3회 재시도 가능 → 실제 최대 6회 이상 재시도 가능

실수 3: DeadLetterPublishingRecoverer와 DLX를 혼용 설정

  // Spring Retry가 3회 실패 → DeadLetterPublishingRecoverer 호출
  // 동시에 RabbitMQ DLX도 설정됨
  // → 두 경로 모두 동작 → 메시지가 두 번 DLQ에 들어갈 수 있음
```

---

## ✨ 올바른 접근 (After — 계층별 에러 처리)

```
Spring AMQP 에러 처리 계층:

계층 1: 비즈니스 예외 분류 (Consumer 코드)
  try { ... }
  catch (BusinessException e) {
    throw new AmqpRejectAndDontRequeueException(e);  // DLQ로
  }
  catch (TransientException e) {
    throw e;  // Container가 requeue=true 처리 (RetryTemplate 있으면 재시도)
  }

계층 2: RetryTemplate (Container 레벨)
  일시적 오류 → 지수 백오프로 N회 재시도
  최종 실패 → MessageRecoverer 호출

계층 3: MessageRecoverer (최종 실패 처리)
  RejectAndDontRequeueRecoverer: 폐기 (DLX 있으면 DLQ로)
  DeadLetterPublishingRecoverer: 명시적으로 DLQ에 발행

계층 4: RabbitMQ DLX (브로커 레벨)
  basicNack(requeue=false) 수신 → DLX Exchange로 이동
```

---

## 🔬 내부 동작 원리

### 1. Container의 에러 처리 흐름

```
=== Spring AMQP 에러 처리 흐름 ===

메시지 수신
    │
    ▼
Container: @RabbitListener 메서드 호출
    │
    ├── 정상 완료 → Ack (AUTO 모드 시 자동)
    │
    └── 예외 발생
            │
            ├── AmqpRejectAndDontRequeueException
            │     → Nack(requeue=false)
            │     → DLX 있으면 DLQ로, 없으면 폐기
            │
            ├── ListenerExecutionFailedException (그 외 예외)
            │     → ErrorHandler 호출
            │     │
            │     └── RetryTemplate 없는 경우:
            │           AUTO 모드: Nack(requeue=true) [기본]
            │           defaultRequeueRejected=false: Nack(requeue=false)
            │
            └── RetryTemplate 있는 경우:
                  BackOffPolicy에 따라 N회 재시도
                  N회 초과 → MessageRecoverer 호출
                  Recoverer: RejectAndDontRequeueRecoverer 또는
                             DeadLetterPublishingRecoverer

=== AcknowledgeMode별 동작 ===

NONE (autoAck):
  메시지 수신 즉시 Ack → 예외 발생 시 이미 삭제됨 → 유실

AUTO (기본):
  정상 완료 → Ack
  AmqpRejectAndDontRequeueException → Nack(requeue=false)
  그 외 예외 → Nack(requeue=defaultRequeueRejected)
               defaultRequeueRejected=true (기본): requeue=true → 무한 루프 위험!
               defaultRequeueRejected=false: requeue=false → DLQ

MANUAL:
  개발자가 직접 basicAck/basicNack 호출
  예외 시 Ack/Nack 코드가 없으면 → Unacked 상태 지속 (무한 대기)
```

### 2. RetryTemplate 설정

```
=== Stateless RetryTemplate (기본, 권장하지 않음) ===

@Bean
public SimpleRabbitListenerContainerFactory containerFactory(ConnectionFactory cf) {
  SimpleRabbitListenerContainerFactory factory = new SimpleRabbitListenerContainerFactory();
  factory.setConnectionFactory(cf);
  
  RetryTemplate retryTemplate = RetryTemplate.builder()
    .maxAttempts(3)                       // 최대 3회 시도
    .fixedBackoff(2000)                   // 2초 고정 대기
    .build();
  
  factory.setRetryTemplate(retryTemplate);
  factory.setRecoveryCallback(context -> {
    // 3회 실패 후 호출 → 폐기 또는 DLQ 발행
    Message message = (Message) context.getAttribute(RetryContext.MESSAGE);
    // 처리: DLQ 발행 또는 로그
    return null;
  });
  
  return factory;
}

Stateless 문제:
  재시도 간 Thread.sleep() → Consumer 스레드 블로킹
  Prefetch 슬롯 점유 지속 → 다른 메시지 처리 차단
  재시작 시 재시도 카운트 초기화

=== Stateful RetryTemplate (재시작 후 카운트 유지) ===

Map<Object, RetryContext> retryContextCache = new ConcurrentHashMap<>();

RetryTemplate statefulRetry = RetryTemplate.builder()
  .maxAttempts(3)
  .exponentialBackoff(1000, 2.0, 30000)  // 1초, 2배, 최대 30초
  .build();

// StatefulRetryOperationsInterceptor: 메시지 ID로 재시도 상태 유지
StatefulRetryOperationsInterceptor interceptor =
  RetryInterceptorBuilder.stateful()
    .retryOperations(statefulRetry)
    .messageKeyGenerator(message -> {
      // 메시지 ID를 key로 사용
      return message.getMessageProperties().getMessageId();
    })
    .recoverer(new RejectAndDontRequeueRecoverer())
    .build();

factory.setAdviceChain(interceptor);

=== 언제 RetryTemplate, 언제 TTL+DLX ===

RetryTemplate 적합:
  빠른 재시도 (수 초 이내)
  동기 처리에서 간단한 재시도
  스레드 블로킹이 허용되는 낮은 처리량

TTL+DLX 적합 (Ch3-05 참고):
  긴 재시도 대기 (수십 초~분 단위)
  스레드 블로킹 허용 안 됨 (고처리량)
  재시작 후에도 재시도 상태 보존 필요
  → 실무에서 고처리량 시스템은 TTL+DLX 권장
```

### 3. MessageRecoverer — 최종 실패 처리

```
=== RejectAndDontRequeueRecoverer ===

동작:
  RetryTemplate 최대 재시도 초과
  → AmqpRejectAndDontRequeueException throw
  → Container: Nack(requeue=false)
  → RabbitMQ DLX 있으면 → DLQ로 이동
  → DLX 없으면 → 폐기

설정:
  factory.setAdviceChain(
    RetryInterceptorBuilder.stateless()
      .maxAttempts(3)
      .recoverer(new RejectAndDontRequeueRecoverer())
      .build()
  );

적합한 경우:
  RabbitMQ DLX가 설정된 Queue에서 사용
  DLX가 알아서 DLQ로 이동

=== DeadLetterPublishingRecoverer ===

동작:
  RetryTemplate 최대 재시도 초과
  → Recoverer: 직접 DLQ(또는 다른 Queue)에 메시지 발행
  → 원본 메시지 Ack (원본 Queue에서 삭제)

설정:
  @Bean
  public DeadLetterPublishingRecoverer recoverer(RabbitTemplate rabbitTemplate) {
    // 기본: 원본 Exchange + ".DLT" Queue로 발행
    return new DeadLetterPublishingRecoverer(rabbitTemplate,
      (message, exception) -> {
        // 실패 원인별 다른 DLQ로 라우팅
        if (exception instanceof BusinessException) {
          return new Address("dlx.exchange", "business-error");
        }
        return new Address("dlx.exchange", "system-error");
      }
    );
  }

  factory.setAdviceChain(
    RetryInterceptorBuilder.stateless()
      .maxAttempts(3)
      .recoverer(recoverer(rabbitTemplate))
      .build()
  );

차이:
  RejectAndDontRequeueRecoverer: DLX 설정에 의존 (브로커 라우팅)
  DeadLetterPublishingRecoverer: 직접 발행 (유연한 라우팅)

실패 원인 추적:
  DeadLetterPublishingRecoverer는 예외 정보를 헤더에 추가:
    x-exception-stacktrace: 스택 트레이스
    x-exception-message: 예외 메시지
    x-original-exchange: 원본 Exchange
    x-original-routing-key: 원본 Routing Key
```

### 4. 완전한 에러 처리 설정

```java
=== 프로덕션 권장 설정 ===

@Configuration
public class ErrorHandlingConfig {

  @Bean
  public SimpleRabbitListenerContainerFactory containerFactory(
      ConnectionFactory cf,
      Jackson2JsonMessageConverter converter,
      RabbitTemplate rabbitTemplate) {

    SimpleRabbitListenerContainerFactory factory = new SimpleRabbitListenerContainerFactory();
    factory.setConnectionFactory(cf);
    factory.setMessageConverter(converter);
    factory.setAcknowledgeMode(AcknowledgeMode.AUTO);
    factory.setDefaultRequeueRejected(false);  // 기본 requeue=false

    // Exponential Backoff 재시도 (3회, 1초→2초→4초)
    RetryTemplate retryTemplate = RetryTemplate.builder()
      .maxAttempts(3)
      .exponentialBackoff(1000, 2.0, 8000)
      .build();

    // 최종 실패 → DLQ 직접 발행 + 예외 정보 헤더
    DeadLetterPublishingRecoverer recoverer =
      new DeadLetterPublishingRecoverer(rabbitTemplate,
        (message, ex) -> new Address("dlx.exchange", "order.dlq")
      );

    factory.setAdviceChain(
      RetryInterceptorBuilder.stateless()
        .retryOperations(retryTemplate)
        .recoverer(recoverer)
        .build()
    );

    return factory;
  }
}

=== Consumer 코드 ===

@RabbitListener(queues = "order.queue",
  containerFactory = "containerFactory")
public void handleOrder(OrderEvent event) {

  // 분류: 일시적 vs 영구적 예외
  try {
    orderService.process(event);

  } catch (BusinessException e) {
    // 재처리 의미 없음 → 즉시 DLQ (RetryTemplate 거치지 않음)
    throw new AmqpRejectAndDontRequeueException("비즈니스 예외", e);

  } catch (DataAccessException e) {
    // 일시적 DB 오류 → RetryTemplate이 재시도
    throw e;

  }
  // 정상 완료 → AUTO 모드가 자동 Ack
}
```

---

## 💻 실전 실험

### 실험 1: RetryTemplate 동작 확인 로그

```java
// 재시도 로그 추가
RetryTemplate retryTemplate = RetryTemplate.builder()
  .maxAttempts(3)
  .exponentialBackoff(1000, 2.0, 8000)
  .withListener(new RetryListenerSupport() {
    @Override
    public <T, E extends Throwable> void onError(
        RetryContext context, RetryCallback<T, E> callback, Throwable throwable) {
      log.warn("재시도 {} / {}: {}", 
        context.getRetryCount(), 
        3,
        throwable.getMessage());
    }
    
    @Override
    public <T, E extends Throwable> void close(
        RetryContext context, RetryCallback<T, E> callback, Throwable throwable) {
      if (throwable != null) {
        log.error("최대 재시도 초과: {}", throwable.getMessage());
      }
    }
  })
  .build();

// 예상 로그:
// WARN: 재시도 1 / 3: DataAccessException
// (1초 대기)
// WARN: 재시도 2 / 3: DataAccessException
// (2초 대기)
// WARN: 재시도 3 / 3: DataAccessException
// ERROR: 최대 재시도 초과: DataAccessException
// → DeadLetterPublishingRecoverer 호출 → DLQ 발행
```

### 실험 2: DLQ 확인

```bash
# 에러 처리 후 DLQ 메시지 확인
rabbitmqadmin get queue=order.dlq ackmode=ack_requeue_true count=1

# 헤더 확인 (DeadLetterPublishingRecoverer 추가 헤더)
# x-exception-message: DataAccessException: Connection refused
# x-exception-stacktrace: com.mycompany...
# x-original-exchange: order.exchange
# x-original-routing-key: order.placed
# x-death 헤더는 없음 (RabbitMQ DLX를 거치지 않고 직접 발행했으므로)
```

---

## 📊 Kafka vs RabbitMQ 비교

```
=== 에러 처리 비교 ===

Spring AMQP RetryTemplate:
  Consumer 스레드 내 블로킹 재시도
  최종 실패 → DeadLetterPublishingRecoverer (DLQ 발행)
  재시작 시 카운트 초기화 (Stateless)

Spring Kafka DefaultErrorHandler:
  Consumer 스레드 내 재시도 (BackOff 설정)
  최종 실패 → DeadLetterPublishingRecoverer (Dead Letter Topic)
  재시작 시: 오프셋 커밋 안 하면 자동 재처리

Kafka @RetryableTopic:
  @RetryableTopic(attempts="4", backoff=@Backoff(delay=1000, multiplier=2))
  → 자동으로 retry Topic 생성 (order-events-retry-0, retry-1, ...)
  → Consumer 스레드 블로킹 없음 (메시지를 retry Topic에 발행)
  → TTL+DLX와 개념적으로 유사

RabbitMQ TTL+DLX (Ch3-05):
  Consumer 스레드 블로킹 없음
  메시지가 대기 Queue에 있는 동안 다른 메시지 처리 가능
  재시작 후에도 상태 보존 (메시지가 Queue에 있으므로)
  → 고처리량 환경에서 Kafka @RetryableTopic과 유사한 패턴
```

---

## ⚖️ 트레이드오프

```
에러 처리 전략 트레이드오프:

AUTO 모드 + defaultRequeueRejected=false:
  ✅ 설정 단순
  ❌ 재시도 없음 → 일시적 오류도 즉시 DLQ

RetryTemplate (Stateless):
  ✅ 재시도 횟수 제한
  ❌ 스레드 블로킹 (처리량 저하)
  ❌ 재시작 시 카운트 초기화

RetryTemplate (Stateful):
  ✅ 재시작 후 카운트 유지
  ❌ 상태 저장소 필요 (메모리 또는 Redis)
  ❌ 설정 복잡

TTL+DLX (RabbitMQ 브로커 레벨):
  ✅ 스레드 블로킹 없음
  ✅ 재시작 후 상태 보존
  ❌ 구성 복잡 (여러 Queue/Exchange 필요)

권장:
  낮은 처리량 + 단순 재시도: RetryTemplate (Stateless)
  높은 처리량 + 장기 재시도: TTL+DLX
  재시작 후 카운트 유지: Stateful RetryTemplate 또는 TTL+DLX
```

---

## 📌 핵심 정리

```
에러 처리 핵심:

예외 분류:
  AmqpRejectAndDontRequeueException: 즉시 DLQ (재시도 없음)
  그 외 예외: RetryTemplate 재시도 또는 defaultRequeueRejected 설정

defaultRequeueRejected:
  true (기본): 예외 → requeue=true → 무한 루프 위험!
  false (권장): 예외 → requeue=false → DLX로

RetryTemplate:
  Stateless: 스레드 블로킹, 재시작 후 초기화
  Stateful: 카운트 유지, 설정 복잡

MessageRecoverer:
  RejectAndDontRequeueRecoverer: DLX에 위임
  DeadLetterPublishingRecoverer: 직접 발행 (예외 정보 헤더 포함)

고처리량 재시도:
  TTL+DLX 방식 (Ch3-05) 권장
  스레드 블로킹 없음, 재시작 후 보존
```

---

## 🤔 생각해볼 문제

**Q1.** `@RabbitListener`에서 `RuntimeException`을 throw했다. `defaultRequeueRejected=true`(기본)와 `defaultRequeueRejected=false` 각각에서 메시지는 어떻게 처리되는가?

<details>
<summary>해설 보기</summary>

**`defaultRequeueRejected=true`(기본):**
- RuntimeException → Container: `basicNack(requeue=true)`
- 메시지 즉시 Queue 앞으로 복귀 → 같은 메시지 즉시 재처리 시도
- 무한 루프 가능성 (CPU 폭발, 다른 메시지 처리 차단)

**`defaultRequeueRejected=false`(권장):**
- RuntimeException → Container: `basicNack(requeue=false)`
- DLX 있으면: 메시지 DLQ로 이동
- DLX 없으면: 메시지 폐기

**실무 권장 설정:**
```java
factory.setDefaultRequeueRejected(false);
```
이 설정으로 의도치 않은 무한 루프를 방지하고, 일시적 오류는 RetryTemplate으로 처리.

일시적 오류를 재시도하려면:
- RetryTemplate을 설정 (`DataAccessException` 등에만 재시도)
- 또는 Consumer 코드에서 `throw e`(일시적) vs `throw new AmqpRejectAndDontRequeueException(e)`(영구) 구분

</details>

---

**Q2.** RetryTemplate을 3회로 설정했다. 1회 실패 후 브로커 연결이 끊겼다가 복구됐다. 재시도 카운트는 어떻게 되는가?

<details>
<summary>해설 보기</summary>

**Stateless RetryTemplate의 경우: 카운트가 초기화됩니다.**

동작:
1. 메시지 수신 → 1회 실패 (RetryContext에 count=1 저장, 메모리에만)
2. 브로커 연결 끊김 → Connection 재연결
3. Unacked 메시지 → Ready 복귀 → 재전달
4. 새 RetryContext 생성 (count=0) → 다시 3회 시도 가능

결과: 이론적으로 무한 재시도 가능 (연결 끊김마다 초기화)

**Stateful RetryTemplate의 경우:**
- messageKeyGenerator로 메시지 ID를 key로 사용
- RetryContext를 메모리 Map에 저장
- 재연결 후 같은 메시지 → 같은 key → 기존 count 유지
- 단, 애플리케이션 재시작 시에는 메모리 초기화 → count 리셋

**완전한 해결:** TTL+DLX 방식 — 메시지 자체가 Queue에 있으므로 재시작/재연결에 무관하게 재시도 상태 보존.

</details>

---

**Q3.** `DeadLetterPublishingRecoverer`로 DLQ에 발행했는데, DLQ Consumer가 또 실패했다. DLQ에도 RetryTemplate을 적용해야 하는가?

<details>
<summary>해설 보기</summary>

**상황에 따라 다르지만, 일반적으로 DLQ에는 재시도를 최소화하는 것이 권장됩니다.**

DLQ의 목적:
- 실패 메시지를 보존하고 원인을 분석하기 위한 장소
- 자동 재처리가 아닌 수동 검토/재처리 목적

DLQ Consumer가 실패하는 이유:
1. 원인이 해결되지 않은 상태 (코드 버그 등) → 재시도해도 계속 실패
2. DLQ 처리 로직 자체 오류 → 재시도 의미 없음
3. 외부 시스템 일시 오류 → 짧은 재시도는 의미 있음

권장 전략:
```java
// DLQ Consumer: 간단한 재시도 + 실패 시 영구 보관
@RabbitListener(queues = "order.dlq")
public void handleDlq(Message message, Channel channel, long tag) {
  long retryCount = getDlqRetryCount(message);
  
  if (retryCount >= 2) {
    // 2회 이상 DLQ 처리 실패 → 영구 보관 Queue 또는 알림
    channel.basicNack(tag, false, false);  // DLQ에서도 제거 (2차 DLX)
    alertService.sendCriticalAlert(message);
    return;
  }
  
  try {
    reprocessFromDlq(message);
    channel.basicAck(tag, false);
  } catch (Exception e) {
    channel.basicNack(tag, false, true);  // DLQ에 재큐 (재시도)
  }
}
```

"DLQ의 DLQ"는 설계 복잡도를 높이므로 알림 + 수동 처리 방식이 더 실용적입니다.

</details>

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[이전: 메시지 직렬화 전략 ⬅️](./02-message-serialization.md)** | **[다음: Spring Boot Auto-configuration ➡️](./04-spring-boot-autoconfiguration.md)**

</div>
