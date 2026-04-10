# Spring Boot Auto-configuration — 설정, 테스트, 통합 전략

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `RabbitAutoConfiguration`은 어떤 Bean을 자동으로 생성하는가?
- `application.yml`만으로 Exchange/Queue/Binding을 선언할 수 있는가?
- `EmbeddedRabbitMQ` vs `Testcontainers` 중 테스트에서 어떤 것을 선택하는가?
- `@SpringBootTest` 통합 테스트에서 RabbitMQ를 어떻게 설정하는가?
- `@RabbitListenerTest`는 무엇이고 어떻게 활용하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

Spring Boot Auto-configuration 덕분에 RabbitMQ 연동이 매우 간단해졌지만, 자동 생성되는 Bean의 종류와 설정 방법을 모르면 "왜 내 설정이 적용 안 되는가"를 알 수 없다. 또 RabbitMQ 없이 로컬 개발과 테스트를 어떻게 할지, Testcontainers로 실제 브로커를 테스트에서 어떻게 활용하는지를 알아야 안정적인 테스트 전략을 설계할 수 있다.

---

## 😱 흔한 실수 (Before)

```
실수 1: Auto-configuration 위에 같은 Bean 중복 선언

  // Spring Boot가 이미 RabbitTemplate Bean을 자동 생성하는데
  @Bean
  public RabbitTemplate rabbitTemplate() {
    return new RabbitTemplate(connectionFactory);  // 충돌!
  }
  
  결과:
    ConflictingBeanDefinitionException 또는 예상치 못한 Bean 사용
    Auto-configured RabbitTemplate의 Converter/ConfirmCallback이 무시됨

실수 2: 단위 테스트에서 실제 RabbitMQ 연결 시도

  @SpringBootTest
  class OrderServiceTest {
    @Autowired RabbitTemplate rabbitTemplate;  // 실제 RabbitMQ 연결!
    
    // 로컬 환경에서 RabbitMQ가 없으면 테스트 실패
    // CI 환경에서 RabbitMQ 없으면 항상 실패
  }

실수 3: EmbeddedRabbitMQ 의존성 없이 @EmbeddedAmqp 사용

  @EmbeddedAmqp  // spring-rabbit-test 의존성 필요
  class Test { }
  
  의존성 없으면 NoSuchBeanDefinitionException
```

---

## ✨ 올바른 접근 (After — 환경별 설정 전략)

```
환경별 RabbitMQ 설정:

개발 환경:
  docker-compose로 로컬 RabbitMQ 실행
  application-dev.yml에 연결 정보

단위 테스트:
  @MockBean RabbitTemplate 또는 @RabbitListenerTest
  실제 브로커 없이 로직 테스트

통합 테스트:
  Testcontainers로 RabbitMQ 컨테이너 실행
  실제 메시지 흐름 테스트

CI/CD:
  Testcontainers 또는 GitHub Actions의 서비스 컨테이너
  실제 RabbitMQ로 E2E 테스트
```

---

## 🔬 내부 동작 원리

### 1. RabbitAutoConfiguration 자동 생성 Bean

```
=== RabbitAutoConfiguration이 생성하는 Bean ===

spring-boot-autoconfigure의 RabbitAutoConfiguration:

1. RabbitConnectionFactoryBean
   → CachingConnectionFactory Bean 생성
   → spring.rabbitmq.* 설정 적용

2. RabbitTemplateConfiguration
   → RabbitTemplate Bean 생성
   → MessageConverter Bean 있으면 자동 설정

3. RabbitMessagingTemplate
   → RabbitTemplate 래퍼 (Spring Messaging 호환)

4. RabbitAnnotationDrivenConfiguration
   → @RabbitListener 처리를 위한 RabbitListenerAnnotationBeanPostProcessor
   → SimpleRabbitListenerContainerFactory Bean 생성

5. RabbitAdminConfiguration (선택적)
   → RabbitAdmin Bean 생성 (spring.rabbitmq.dynamic=true 시)

=== application.yml 주요 설정 ===

spring:
  rabbitmq:
    host: localhost                        # 브로커 호스트
    port: 5672                             # AMQP 포트
    username: admin
    password: password
    virtual-host: /                        # vHost
    connection-timeout: 5000              # 연결 타임아웃 (ms)
    
    # Publisher Confirm
    publisher-confirm-type: correlated    # NONE, SIMPLE, CORRELATED
    publisher-returns: true               # Return 콜백 활성화
    
    # Connection Pool
    cache:
      channel:
        size: 25                          # Channel Cache 크기
      connection:
        mode: channel                     # CHANNEL or CONNECTION
        size: 1                           # CONNECTION 모드 시 Pool 크기
    
    # Consumer 기본 설정
    listener:
      simple:
        acknowledge-mode: manual          # NONE, AUTO, MANUAL
        prefetch: 10
        concurrency: 3                    # 최소 Consumer 스레드
        max-concurrency: 10              # 최대 Consumer 스레드
        default-requeue-rejected: false  # 기본 requeue=false
        retry:
          enabled: true
          max-attempts: 3
          initial-interval: 1000
          multiplier: 2.0
          max-interval: 10000

=== Auto-configuration 커스터마이징 ===

방법 1: @Bean으로 재정의 (이미 있는 Bean 타입 재선언)
  @Bean
  public RabbitTemplate rabbitTemplate(ConnectionFactory connectionFactory,
      MessageConverter converter) {
    RabbitTemplate template = new RabbitTemplate(connectionFactory);
    template.setMessageConverter(converter);
    template.setMandatory(true);
    // Auto-configuration의 RabbitTemplate을 이 Bean이 대체
    return template;
  }

방법 2: RabbitTemplateCustomizer (Auto-configured Template 수정)
  @Bean
  public RabbitTemplateCustomizer confirmCallbackCustomizer() {
    return template -> {
      template.setConfirmCallback((correlation, ack, cause) -> {
        if (!ack) log.error("Publisher Nack: {}", cause);
      });
    };
  }
  // Auto-configured RabbitTemplate에 커스터마이저만 적용
  // Bean 재선언 없이 추가 설정

방법 3: Properties만으로 설정 (가능한 범위 내)
  spring.rabbitmq.listener.simple.prefetch=10
  spring.rabbitmq.publisher-confirm-type=correlated
```

### 2. 테스트 전략

```
=== 단위 테스트: @MockBean 활용 ===

@ExtendWith(SpringExtension.class)
@SpringBootTest
class OrderServiceTest {

  @MockBean
  private RabbitTemplate rabbitTemplate;
  
  @Autowired
  private OrderService orderService;
  
  @Test
  void placeOrder_shouldPublishEvent() {
    // 실제 RabbitMQ 없이 동작
    Order order = orderService.placeOrder(new OrderRequest("item-1", 5));
    
    // 발행 호출 검증
    verify(rabbitTemplate).convertAndSend(
      eq("order.exchange"),
      eq("order.placed"),
      argThat((OrderPlacedEvent e) -> e.getOrderId().equals(order.getId()))
    );
  }
}

// @MockBean 대신 Mockito만 사용 (Spring 없이):
class OrderServiceUnitTest {
  private RabbitTemplate rabbitTemplate = mock(RabbitTemplate.class);
  private OrderService orderService = new OrderService(rabbitTemplate, orderRepository);
  
  @Test
  void placeOrder_withMockRabbit() {
    orderService.placeOrder(...);
    verify(rabbitTemplate).convertAndSend(anyString(), anyString(), any());
  }
}

=== @RabbitListenerTest — Consumer 테스트 ===

spring-rabbit-test 의존성:
  <dependency>
    <groupId>org.springframework.amqp</groupId>
    <artifactId>spring-rabbit-test</artifactId>
    <scope>test</scope>
  </dependency>

@SpringBootTest
@RabbitListenerTest(capture = true)  // 메시지 캡처 모드
class OrderConsumerTest {

  @Autowired
  private RabbitListenerTestHarness harness;
  
  @Autowired
  private RabbitTemplate rabbitTemplate;
  
  @Test
  void consumer_shouldProcessOrderEvent() throws Exception {
    // Consumer 가져오기
    OrderConsumer consumer = harness.getSpy("orderConsumerMethod");
    
    // 메시지 발행
    rabbitTemplate.convertAndSend("order.exchange", "order.placed",
      new OrderPlacedEvent("ord-1", 50000));
    
    // Consumer가 호출됐는지 검증 (비동기 대기)
    verify(consumer, timeout(5000)).handleOrder(any(OrderPlacedEvent.class));
  }
}

=== Testcontainers — 실제 RabbitMQ 통합 테스트 ===

의존성:
  <dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>rabbitmq</artifactId>
    <scope>test</scope>
  </dependency>

@SpringBootTest
@Testcontainers
class RabbitMQIntegrationTest {

  @Container
  static RabbitMQContainer rabbitMQ = new RabbitMQContainer("rabbitmq:3.12-management")
    .withVhost("/")
    .withUser("admin", "password")
    .withExchange("order.exchange", "topic")
    .withQueue("order.queue")
    .withBinding("order.exchange", "order.queue", Map.of(), "order.#", "queue");

  @DynamicPropertySource
  static void configureRabbitMQ(DynamicPropertyRegistry registry) {
    registry.add("spring.rabbitmq.host", rabbitMQ::getHost);
    registry.add("spring.rabbitmq.port", rabbitMQ::getAmqpPort);
    registry.add("spring.rabbitmq.username", () -> "admin");
    registry.add("spring.rabbitmq.password", () -> "password");
  }

  @Autowired private RabbitTemplate rabbitTemplate;
  @Autowired private OrderConsumer orderConsumer;

  @Test
  void fullFlow_publishAndConsume() throws Exception {
    CountDownLatch latch = new CountDownLatch(1);
    // Consumer가 처리하면 latch 해제 (실제 Consumer에 latch 주입 필요)
    
    rabbitTemplate.convertAndSend("order.exchange", "order.placed",
      new OrderPlacedEvent("ord-1", 50000));
    
    assertTrue(latch.await(5, TimeUnit.SECONDS));
    // 실제 메시지가 실제 RabbitMQ를 통해 Consumer에 도달
  }
}
```

### 3. 환경별 설정 분리

```
=== Spring Profiles로 환경별 RabbitMQ 설정 ===

application.yml (공통):
  spring:
    rabbitmq:
      listener:
        simple:
          acknowledge-mode: manual
          prefetch: 10

application-local.yml (로컬 개발):
  spring:
    rabbitmq:
      host: localhost
      port: 5672
      username: admin
      password: password

application-test.yml (테스트 환경):
  spring:
    rabbitmq:
      host: ${RABBITMQ_HOST:localhost}
      port: ${RABBITMQ_PORT:5672}
      # Testcontainers가 동적으로 설정
      
application-prod.yml (프로덕션):
  spring:
    rabbitmq:
      addresses: rabbitmq-1:5672,rabbitmq-2:5672,rabbitmq-3:5672
      username: ${RABBITMQ_USER}
      password: ${RABBITMQ_PASSWORD}
      ssl:
        enabled: true
        key-store: classpath:client.p12
        trust-store: classpath:ca.p12

=== Docker Compose 로컬 개발 환경 ===

docker-compose.yml:
  services:
    rabbitmq:
      image: rabbitmq:3.12-management
      ports:
        - "5672:5672"    # AMQP
        - "15672:15672"  # Management UI
      environment:
        RABBITMQ_DEFAULT_USER: admin
        RABBITMQ_DEFAULT_PASS: password
      volumes:
        - rabbitmq-data:/var/lib/rabbitmq
      healthcheck:
        test: ["CMD", "rabbitmq-diagnostics", "ping"]
        interval: 10s
        timeout: 5s
        retries: 5
  volumes:
    rabbitmq-data:
```

### 4. 프로덕션 설정 체크리스트

```
=== Spring Boot 프로덕션 설정 체크리스트 ===

연결 설정:
  ✅ addresses: 클러스터 주소 (3노드)
  ✅ connection-timeout: 5000 (5초)
  ✅ channel-cache-size: 예상 동시 Publisher 수에 맞게
  ✅ SSL 설정 (프로덕션)

발행 설정:
  ✅ publisher-confirm-type: correlated
  ✅ publisher-returns: true
  ✅ ConfirmCallback + ReturnsCallback 등록

소비 설정:
  ✅ acknowledge-mode: manual (또는 auto + defaultRequeueRejected=false)
  ✅ prefetch: 처리 시간 기반으로 설정
  ✅ concurrency / max-concurrency: 처리량 기반
  ✅ default-requeue-rejected: false (무한 루프 방지)

재시도 설정:
  ✅ retry.enabled: true
  ✅ max-attempts: 3~5
  ✅ multiplier: 2.0 (지수 백오프)
  ✅ DeadLetterPublishingRecoverer 또는 RabbitMQ DLX

테스트 설정:
  ✅ Testcontainers 의존성 (통합 테스트)
  ✅ @MockBean RabbitTemplate (단위 테스트)
  ✅ @RabbitListenerTest (Consumer 테스트)

모니터링:
  ✅ rabbitmq_prometheus Plugin 활성화
  ✅ Grafana 대시보드 연결
  ✅ DLQ 메시지 알림 설정
```

---

## 💻 실전 실험

### 실험 1: Auto-configuration 확인

```bash
# Spring Boot 앱에서 자동 생성된 Bean 확인
curl http://localhost:8080/actuator/beans | jq '.contexts.*.beans | to_entries | 
  map(select(.key | contains("rabbit") or contains("Rabbit"))) | .[].key'

# 출력 예:
# "rabbitTemplate"
# "amqpAdmin"
# "rabbitListenerContainerFactory"
# "simpleMessageListenerContainer"
# "org.springframework.boot.autoconfigure.amqp.RabbitAutoConfiguration"
```

### 실험 2: Testcontainers 통합 테스트

```java
// build.gradle
dependencies {
  testImplementation 'org.springframework.amqp:spring-rabbit-test'
  testImplementation 'org.testcontainers:rabbitmq:1.19.0'
  testImplementation 'org.testcontainers:junit-jupiter:1.19.0'
}

// 통합 테스트 실행
@SpringBootTest(webEnvironment = WebEnvironment.NONE)
@Testcontainers
@ActiveProfiles("test")
class OrderFlowIntegrationTest {

  @Container
  static RabbitMQContainer rmq = new RabbitMQContainer("rabbitmq:3.12-management");
  
  @DynamicPropertySource
  static void props(DynamicPropertyRegistry registry) {
    registry.add("spring.rabbitmq.host", rmq::getHost);
    registry.add("spring.rabbitmq.port", rmq::getAmqpPort);
  }
  
  @Autowired OrderService orderService;
  @Autowired RabbitTemplate rabbitTemplate;
  
  @Test
  @Timeout(10)
  void placeOrder_shouldTriggerPaymentProcessing() throws Exception {
    CountDownLatch processed = new CountDownLatch(1);
    // PaymentService Consumer에 latch 주입 후
    
    orderService.placeOrder(new OrderRequest("item-1", 3));
    assertTrue(processed.await(5, TimeUnit.SECONDS));
  }
}
```

---

## 📊 Kafka vs RabbitMQ 비교

```
=== Spring Integration 편의성 비교 ===

Spring AMQP (RabbitMQ):
  Auto-configuration: RabbitAutoConfiguration
  @RabbitListener로 Consumer 선언 (Spring 완벽 통합)
  EmbeddedRabbitMQ 테스트 지원 (spring-rabbit-test)
  Testcontainers RabbitMQContainer

Spring Kafka:
  Auto-configuration: KafkaAutoConfiguration
  @KafkaListener로 Consumer 선언
  EmbeddedKafka 테스트 지원 (@EmbeddedKafka)
  Testcontainers KafkaContainer

테스트 편의성:
  둘 다 유사한 수준 (Testcontainers + Mock 지원)
  EmbeddedRabbitMQ는 Erlang 없이 순수 Java로 동작 (더 가벼움)
  EmbeddedKafka는 실제 Kafka를 인메모리로 실행

Auto-configuration 범위:
  Spring AMQP: ConnectionFactory, RabbitTemplate, ContainerFactory, RabbitAdmin
  Spring Kafka: ProducerFactory, ConsumerFactory, KafkaTemplate, ListenerContainerFactory
```

---

## ⚖️ 트레이드오프

```
테스트 전략 트레이드오프:

@MockBean RabbitTemplate (단위 테스트):
  ✅ 빠름 (실제 브로커 없음)
  ✅ CI 환경 의존성 없음
  ❌ 실제 메시지 흐름 테스트 불가
  ❌ 직렬화/역직렬화 검증 불가

@EmbeddedAmqp (경량 통합 테스트):
  ✅ Erlang 없이 Java만으로 동작
  ✅ 단위 테스트보다 실제에 가까움
  ❌ 실제 RabbitMQ와 완전히 동일하지 않음
  ❌ 일부 기능 제한 (Quorum Queue 등)

Testcontainers (통합 테스트):
  ✅ 실제 RabbitMQ 동작 (Docker)
  ✅ 프로덕션과 동일한 환경
  ❌ Docker 필요 (CI에서 Docker-in-Docker 설정)
  ❌ 테스트 시작 느림 (컨테이너 시작 시간)

권장 조합:
  단위 테스트: @MockBean (빠른 피드백)
  통합 테스트: Testcontainers (신뢰성)
  테스트 피라미드: 단위 70% + 통합 20% + E2E 10%
```

---

## 📌 핵심 정리

```
Spring Boot Auto-configuration 핵심:

자동 생성 Bean:
  CachingConnectionFactory, RabbitTemplate
  SimpleRabbitListenerContainerFactory, RabbitAdmin
  → 별도 Bean 없이 @RabbitListener 바로 사용 가능

설정 커스터마이징:
  application.yml: 대부분 설정 가능
  @Bean 재선언: Auto-configured Bean 대체
  RabbitTemplateCustomizer: 일부 속성만 추가

테스트 전략:
  단위: @MockBean RabbitTemplate
  Consumer: @RabbitListenerTest + RabbitListenerTestHarness
  통합: Testcontainers RabbitMQContainer
  @DynamicPropertySource: Testcontainers 포트 동적 설정

환경별 설정:
  Spring Profiles (local/test/prod)
  prod: addresses(클러스터), SSL, 환경변수 비밀번호

프로덕션 필수:
  publisher-confirm-type: correlated
  acknowledge-mode: manual
  default-requeue-rejected: false
  DLX + DLQ 설정
  Prometheus 모니터링
```

---

## 🤔 생각해볼 문제

**Q1.** `spring.rabbitmq.listener.simple.retry.enabled=true`로 설정하면 RetryTemplate이 자동 생성된다. 이 설정과 `DeadLetterPublishingRecoverer`를 직접 Bean으로 등록하면 어떻게 상호작용하는가?

<details>
<summary>해설 보기</summary>

**Auto-configuration의 RetryTemplate은 `RejectAndDontRequeueRecoverer`를 기본 Recoverer로 사용합니다.** 

`DeadLetterPublishingRecoverer` Bean을 직접 등록하면 Auto-configuration이 이를 감지하여 기본 Recoverer 대신 사용할 수 있지만, 이는 Spring Boot 버전에 따라 다릅니다.

확실한 방법:
```java
@Bean
public SimpleRabbitListenerContainerFactory rabbitListenerContainerFactory(
    ConnectionFactory cf, RabbitTemplate rabbitTemplate) {
  SimpleRabbitListenerContainerFactory factory = new SimpleRabbitListenerContainerFactory();
  factory.setConnectionFactory(cf);
  
  // 직접 RetryTemplate 설정 (Auto-configuration 재정의)
  factory.setAdviceChain(
    RetryInterceptorBuilder.stateless()
      .maxAttempts(3)
      .backOffOptions(1000, 2.0, 10000)
      .recoverer(new DeadLetterPublishingRecoverer(rabbitTemplate,
        (m, e) -> new Address("dlx.exchange", "order.dlq")))
      .build()
  );
  return factory;
}
```

이렇게 `rabbitListenerContainerFactory` Bean을 직접 선언하면 Auto-configuration의 ContainerFactory를 완전히 대체합니다. `application.yml`의 `retry.*` 설정은 이 경우 적용되지 않습니다.

</details>

---

**Q2.** Testcontainers를 사용하는 통합 테스트에서 `@DirtiesContext` 없이 여러 테스트 클래스가 같은 RabbitMQ 컨테이너를 공유하려면 어떻게 해야 하는가?

<details>
<summary>해설 보기</summary>

**`static @Container`와 `@DynamicPropertySource`를 사용하면 Testcontainers가 JVM당 하나의 컨테이너를 공유할 수 있습니다.**

```java
// 공통 추상 클래스로 컨테이너 공유
@Testcontainers
abstract class AbstractRabbitMQTest {

  @Container
  static RabbitMQContainer rabbitMQ = new RabbitMQContainer("rabbitmq:3.12")
    .withReuse(true);  // 컨테이너 재사용 활성화

  @DynamicPropertySource
  static void registerProperties(DynamicPropertyRegistry registry) {
    registry.add("spring.rabbitmq.host", rabbitMQ::getHost);
    registry.add("spring.rabbitmq.port", rabbitMQ::getAmqpPort);
  }
}

@SpringBootTest
class OrderServiceTest extends AbstractRabbitMQTest {
  // 같은 컨테이너 재사용
}

@SpringBootTest
class PaymentServiceTest extends AbstractRabbitMQTest {
  // 같은 컨테이너 재사용
}
```

`withReuse(true)` 설정 시 `~/.testcontainers.properties`에 `testcontainers.reuse.enable=true`도 필요합니다.

각 테스트 간 Queue 상태 격리:
```java
@BeforeEach
void cleanUp() {
  rabbitAdmin.purgeQueue("order.queue");
  rabbitAdmin.purgeQueue("order.dlq");
}
```

</details>

---

**Q3.** `@RabbitListenerTest`로 Consumer 테스트를 작성하려 한다. Consumer가 메시지 처리 후 다른 서비스에 HTTP 호출을 한다. 이 HTTP 호출을 테스트에서 어떻게 처리해야 하는가?

<details>
<summary>해설 보기</summary>

**`@MockBean`으로 HTTP Client를 Mock 처리합니다.**

```java
@SpringBootTest
@RabbitListenerTest(capture = true)
class OrderConsumerTest {

  @MockBean
  private NotificationClient notificationClient;  // HTTP Client Mock
  
  @MockBean
  private InventoryClient inventoryClient;  // 또 다른 HTTP Client
  
  @Autowired
  private RabbitListenerTestHarness harness;
  
  @Autowired
  private RabbitTemplate rabbitTemplate;
  
  @Test
  void handleOrderPlaced_shouldCallNotification() throws Exception {
    // Mock 설정
    when(inventoryClient.reserve(any())).thenReturn(new ReservationResult(true));
    
    // 메시지 발행
    rabbitTemplate.convertAndSend("order.exchange", "order.placed",
      new OrderPlacedEvent("ord-1", 50000));
    
    // Consumer 호출 대기
    OrderConsumer spy = harness.getSpy("handleOrderPlaced");
    verify(spy, timeout(3000)).handleOrderPlaced(any(OrderPlacedEvent.class));
    
    // HTTP 호출 검증
    verify(notificationClient, timeout(3000))
      .sendOrderConfirmation("ord-1");
    verify(inventoryClient, timeout(3000))
      .reserve(argThat(r -> r.getQuantity() == 3));
  }
}
```

이 방식으로:
- RabbitMQ 메시지 수신 및 처리 로직은 실제로 테스트
- 외부 HTTP 호출은 Mock으로 격리 (외부 의존성 없음)
- 실제 RabbitMQ는 EmbeddedRabbitMQ 또는 Testcontainers 사용

</details>

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[이전: 에러 처리와 재시도 ⬅️](./03-error-handling-retry.md)**

</div>
