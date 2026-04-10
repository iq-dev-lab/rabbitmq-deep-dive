# 메시지 직렬화 전략 — Jackson, 타입 안전성, 커스텀 Converter

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `Jackson2JsonMessageConverter`는 어떻게 Object를 Message로 변환하는가?
- 역직렬화 시 타입 정보를 어떻게 전달하고 안전하게 사용하는가?
- `__TypeId__` 헤더가 왜 필요하고, 잘못 사용하면 어떤 문제가 생기는가?
- 서로 다른 서비스 간 메시지 형식 버전 호환성을 어떻게 유지하는가?
- 커스텀 MessageConverter는 언제 필요하고 어떻게 구현하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

서로 다른 두 서비스가 RabbitMQ로 통신할 때, 메시지를 어떻게 직렬화하고 역직렬화하는지가 실제로 가장 많은 오류를 만드는 영역이다. "ClassNotFoundException", "Cannot deserialize", "Unknown type id" 같은 오류가 배포 후 발생하면 서비스 간 메시지 계약이 깨진 것이다. 타입 정보 전달 방식을 이해하고 버전 호환 설계를 알면 이런 오류를 예방할 수 있다.

---

## 😱 흔한 실수 (Before)

```
실수 1: __TypeId__ 헤더에 구체적인 클래스 경로 의존

  // Java 클래스 이름이 헤더에 저장됨
  // __TypeId__: com.company.order.domain.OrderPlacedEvent
  
  문제:
    패키지 리팩토링 시 → 헤더 값이 구 클래스명
    → 역직렬화 ClassNotFoundException
    Producer와 Consumer의 패키지 구조가 달라야 할 때 불가

실수 2: 서로 다른 서비스에서 같은 DTO를 공유 라이브러리로 관리

  // shared-dto 라이브러리 의존
  // OrderEvent.class를 모든 서비스가 공유
  
  문제:
    OrderEvent 필드 추가 → shared-dto 버전 업
    모든 서비스 동시 배포 필요 (롤링 배포 불가)
    서비스 간 강한 결합

실수 3: SimpleMessageConverter로 복잡한 Object 전송

  // SimpleMessageConverter: String/byte[]/Serializable만 지원
  rabbitTemplate.convertAndSend(exchange, key, orderEvent);
  
  오류: MessageConversionException
  → Jackson2JsonMessageConverter 필요
```

---

## ✨ 올바른 접근 (After — Jackson + 타입 별칭 전략)

```
권장 설정:

1. Jackson2JsonMessageConverter 사용 (JSON 직렬화)
2. TypeMapper로 클래스명 대신 별칭 사용
3. Consumer에서 명시적 타입 지정

@Bean
public Jackson2JsonMessageConverter messageConverter() {
  Jackson2JsonMessageConverter converter = new Jackson2JsonMessageConverter();
  
  // 클래스명 대신 별칭 매핑
  DefaultJackson2JavaTypeMapper typeMapper = new DefaultJackson2JavaTypeMapper();
  typeMapper.setTypePrecedence(TypePrecedence.TYPE_ID);
  typeMapper.addTrustedPackages("com.mycompany.*");
  
  Map<String, Class<?>> idClassMapping = new HashMap<>();
  idClassMapping.put("order.placed", OrderPlacedEvent.class);
  idClassMapping.put("payment.completed", PaymentCompletedEvent.class);
  typeMapper.setIdClassMapping(idClassMapping);
  
  converter.setJavaTypeMapper(typeMapper);
  return converter;
}
```

---

## 🔬 내부 동작 원리

### 1. Jackson2JsonMessageConverter 직렬화 흐름

```
=== 발행 시 (Object → Message) ===

rabbitTemplate.convertAndSend("order.exchange", "order.placed", orderEvent);

1. convertAndSend → MessageConverter.toMessage() 호출
2. Jackson ObjectMapper: orderEvent → JSON bytes
   {"orderId":"ord-1","amount":50000,"customerId":"cust-1"}
3. MessageProperties 설정:
   content-type: application/json
   content-encoding: UTF-8
   __TypeId__: com.mycompany.order.OrderPlacedEvent  ← 기본값: 클래스 FQN
4. Message 생성: {headers: {...}, body: JSON bytes}
5. basicPublish 전송

=== 수신 시 (Message → Object) ===

@RabbitListener(queues = "order.queue")
public void handle(OrderPlacedEvent event) { ... }
// Spring AMQP가 자동으로 역직렬화

1. Message 수신
2. __TypeId__ 헤더 읽기: "com.mycompany.order.OrderPlacedEvent"
3. TypeMapper: 헤더 값 → Class 조회
4. Jackson ObjectMapper: JSON bytes → OrderPlacedEvent 객체
5. handle(event) 호출

=== __TypeId__ 헤더의 역할 ===

TypeId 헤더가 없으면 (기본 Jackson 설정):
  @RabbitListener(queues = "order.queue")
  public void handle(Object event) {  // Object로 받아야 함
    OrderPlacedEvent order = (OrderPlacedEvent) event;  // ClassCastException 위험!
  }

TypeId 헤더 있으면:
  @RabbitListener(queues = "order.queue")
  public void handle(OrderPlacedEvent event) {  // 자동 타입 변환
    // OrderPlacedEvent로 역직렬화됨
  }
```

### 2. 타입 별칭 — 패키지 독립성

```
=== 문제: 클래스 FQN에 의존 ===

Producer 패키지: com.order.service.event.OrderPlacedEvent
Consumer 패키지: com.payment.service.dto.OrderEvent

Producer가 발행:
  __TypeId__: com.order.service.event.OrderPlacedEvent

Consumer가 수신:
  com.order.service.event.OrderPlacedEvent 클래스를 찾음
  → ClassNotFoundException (Consumer에 해당 클래스 없음)

=== 해결: 타입 별칭 매핑 ===

Producer 설정:
  typeMapper.setIdClassMapping(Map.of(
    "order.placed", OrderPlacedEvent.class  // 별칭 → 실제 클래스
  ));
  // 발행 시 __TypeId__: "order.placed" (클래스명 아님)

Consumer 설정:
  typeMapper.setIdClassMapping(Map.of(
    "order.placed", OrderEvent.class  // 같은 별칭 → 다른 클래스 OK
  ));
  // 수신 시 "order.placed" → Consumer의 OrderEvent로 역직렬화

결과:
  Producer와 Consumer가 서로 다른 클래스 구조 가능
  패키지 변경해도 별칭이 유지되면 호환성 유지

=== 신뢰할 수 있는 패키지 설정 ===

DefaultJackson2JavaTypeMapper:
  // 기본: 모든 패키지 신뢰 (보안 위험)
  typeMapper.addTrustedPackages("*");  // 위험

  // 권장: 특정 패키지만 신뢰
  typeMapper.addTrustedPackages(
    "com.mycompany.order",
    "com.mycompany.payment"
  );
  // 다른 패키지의 클래스 역직렬화 → 거부
  
  // 또는 별칭만 허용
  typeMapper.setTypePrecedence(TypePrecedence.TYPE_ID);
  // __TypeId__ 헤더의 별칭만 사용, FQN 무시
```

### 3. 버전 호환성 설계

```
=== 이벤트 버전 관리 전략 ===

전략 1: 기존 필드 유지 + 새 필드 Optional

// v1 이벤트 (발행 중)
{"orderId":"ord-1","amount":50000}

// v2 이벤트 (새로 추가)
{"orderId":"ord-1","amount":50000,"discountCode":"SUMMER20"}

Consumer가 v1/v2 모두 처리:
  @JsonIgnoreProperties(ignoreUnknown = true)  // 모르는 필드 무시
  public class OrderPlacedEvent {
    private String orderId;
    private int amount;
    private String discountCode;  // v2에서 추가, null이면 v1 메시지
  }

→ 롤링 배포 가능:
  Consumer v2 배포 → v1 메시지도 처리 가능 (discountCode = null)
  Producer v2 배포 → v1 Consumer도 처리 가능 (모르는 필드 무시)

전략 2: 이벤트 버전 명시

public class OrderPlacedEvent {
  private String version = "2.0";
  private String orderId;
  private int amount;
  private String discountCode;  // v2에서 추가
}

Consumer:
  if ("1.0".equals(event.getVersion())) {
    handleV1(event);
  } else {
    handleV2(event);
  }

전략 3: 타입 별칭으로 버전 관리

  별칭: "order.placed.v1" → OrderPlacedEventV1
  별칭: "order.placed.v2" → OrderPlacedEventV2

  Producer: 새 버전으로 전환 시 별칭 변경
  Consumer: 두 별칭 모두 처리 (전환 기간)

=== Jackson 설정 권장 ===

ObjectMapper 설정:
  objectMapper.configure(
    DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false);
  // 모르는 필드 → 에러 대신 무시 (호환성)

  objectMapper.configure(
    SerializationFeature.WRITE_DATES_AS_TIMESTAMPS, false);
  // 날짜 → ISO 8601 문자열 (타임스탬프보다 가독성)

  objectMapper.registerModule(new JavaTimeModule());
  // LocalDateTime 직렬화 지원
```

### 4. 커스텀 MessageConverter

```
=== 커스텀 Converter 필요한 경우 ===

1. Protobuf 직렬화 (더 작은 크기, 강한 스키마)
2. Avro (Schema Registry 통합)
3. 레거시 시스템 연동 (XML, CSV 등)
4. 특수 압축 요구사항

=== Protobuf 커스텀 Converter 예시 ===

public class ProtobufMessageConverter implements MessageConverter {

  @Override
  public Message toMessage(Object object, MessageProperties messageProperties)
      throws MessageConversionException {
    if (!(object instanceof MessageLite)) {
      throw new MessageConversionException("Protobuf 메시지가 아닙니다");
    }
    byte[] bytes = ((MessageLite) object).toByteArray();
    messageProperties.setContentType("application/x-protobuf");
    messageProperties.setHeader("protobuf-type", object.getClass().getName());
    return new Message(bytes, messageProperties);
  }

  @Override
  public Object fromMessage(Message message) throws MessageConversionException {
    String typeName = (String) message.getMessageProperties()
      .getHeaders().get("protobuf-type");
    try {
      Class<? extends MessageLite> type =
        (Class<? extends MessageLite>) Class.forName(typeName);
      Method parseFrom = type.getMethod("parseFrom", byte[].class);
      return parseFrom.invoke(null, (Object) message.getBody());
    } catch (Exception e) {
      throw new MessageConversionException("역직렬화 실패", e);
    }
  }
}

// 등록:
@Bean
public RabbitTemplate rabbitTemplate(ConnectionFactory cf) {
  RabbitTemplate template = new RabbitTemplate(cf);
  template.setMessageConverter(new ProtobufMessageConverter());
  return template;
}

=== 다중 Converter 처리 ===

content-type별 다른 Converter:
@Bean
public ContentTypeDelegatingMessageConverter converter() {
  ContentTypeDelegatingMessageConverter converter =
    new ContentTypeDelegatingMessageConverter();
  converter.addDelegate("application/json", new Jackson2JsonMessageConverter());
  converter.addDelegate("application/x-protobuf", new ProtobufMessageConverter());
  return converter;
}
```

---

## 💻 실전 실험

### 실험 1: 타입 별칭 설정 완전한 예시

```java
@Configuration
public class SerializationConfig {

  @Bean
  public Jackson2JsonMessageConverter messageConverter(ObjectMapper objectMapper) {
    Jackson2JsonMessageConverter converter =
      new Jackson2JsonMessageConverter(objectMapper);

    DefaultJackson2JavaTypeMapper typeMapper = new DefaultJackson2JavaTypeMapper();
    typeMapper.setTypePrecedence(TypePrecedence.TYPE_ID);  // 별칭 우선

    // 별칭 등록 (패키지 독립)
    Map<String, Class<?>> mapping = new HashMap<>();
    mapping.put("order.placed",       OrderPlacedEvent.class);
    mapping.put("order.cancelled",    OrderCancelledEvent.class);
    mapping.put("payment.completed",  PaymentCompletedEvent.class);
    typeMapper.setIdClassMapping(mapping);

    // 신뢰 패키지
    typeMapper.addTrustedPackages("com.mycompany.*");

    converter.setJavaTypeMapper(typeMapper);
    return converter;
  }

  @Bean
  public ObjectMapper objectMapper() {
    return Jackson2ObjectMapperBuilder.json()
      .featuresToDisable(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS)
      .featuresToDisable(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES)
      .modules(new JavaTimeModule())
      .build();
  }
}
```

### 실험 2: 버전 호환 이벤트 설계 테스트

```bash
# Producer v1 메시지 발행
rabbitmqadmin publish exchange=order.exchange routing_key="order.placed" \
  properties='{"content_type":"application/json","headers":{"__TypeId__":"order.placed"}}' \
  payload='{"orderId":"ord-1","amount":50000}'

# Consumer v2 코드로 수신 (discountCode=null이면 v1 메시지)
# FAIL_ON_UNKNOWN_PROPERTIES=false → 모르는 필드 무시하므로 OK

# Producer v2 메시지 발행 (discountCode 추가)
rabbitmqadmin publish exchange=order.exchange routing_key="order.placed" \
  properties='{"content_type":"application/json","headers":{"__TypeId__":"order.placed"}}' \
  payload='{"orderId":"ord-2","amount":50000,"discountCode":"SUMMER20"}'
```

---

## 📊 Kafka vs RabbitMQ 비교

```
=== 직렬화 전략 비교 ===

RabbitMQ (Spring AMQP):
  기본: Jackson2JsonMessageConverter (JSON)
  타입 정보: __TypeId__ 헤더 (메시지마다 포함)
  Schema 관리: 없음 (개발팀이 직접 관리)

Kafka (Spring Kafka):
  기본: StringSerializer + JsonDeserializer
  Schema 관리: Confluent Schema Registry 활용 가능
  Avro/Protobuf: Schema Registry로 버전 관리

Schema Registry 차이:
  Kafka: Schema Registry → Producer/Consumer가 Registry에서 스키마 조회
         스키마 변경 시 Registry에서 호환성 검사 (forward/backward compatible)
         → 강한 계약 (Contract-first)

  RabbitMQ: @JsonIgnoreProperties + 수동 버전 관리
            → 유연하지만 계약 관리 어려움

권장:
  팀 2~3개 서비스: JSON + @JsonIgnoreProperties + 타입 별칭
  팀 10개+ 서비스: Schema Registry (Kafka) 또는 Protobuf + 버전 명시
```

---

## ⚖️ 트레이드오프

```
직렬화 전략 트레이드오프:

JSON (Jackson):
  ✅ 가독성 높음, 디버깅 쉬움
  ✅ 유연성 (필드 추가/무시)
  ❌ 파싱 비용 (텍스트 파싱)
  ❌ 메시지 크기 (바이너리보다 큼)

Protobuf:
  ✅ 작은 크기 (JSON 대비 3~10배 작음)
  ✅ 강한 스키마 (컴파일 타임 검증)
  ❌ 가독성 낮음 (바이너리)
  ❌ 설정 복잡

클래스 FQN vs 타입 별칭:
  FQN:
    ✅ 설정 간단 (기본값)
    ❌ 패키지 변경 시 호환성 깨짐
  타입 별칭:
    ✅ 패키지 독립적
    ✅ Producer/Consumer 클래스 구조 다를 수 있음
    ❌ 별칭 매핑 관리 필요
```

---

## 📌 핵심 정리

```
메시지 직렬화 핵심:

Jackson2JsonMessageConverter:
  Object → JSON bytes + __TypeId__ 헤더
  수신 시 __TypeId__로 역직렬화 타입 결정

타입 별칭 (권장):
  클래스 FQN 대신 "order.placed" 같은 별칭 사용
  Producer/Consumer 패키지 독립
  DefaultJackson2JavaTypeMapper.setIdClassMapping()

버전 호환:
  @JsonIgnoreProperties(ignoreUnknown = true): 모르는 필드 무시
  새 필드 Optional (null 허용)
  → 롤링 배포 가능

커스텀 Converter:
  Protobuf, Avro, XML 등 특수 포맷 필요 시
  MessageConverter 인터페이스 구현

신뢰 패키지:
  addTrustedPackages("com.mycompany.*")
  모든 패키지 허용(*) 은 보안 위험
```

---

## 🤔 생각해볼 문제

**Q1.** Producer가 `OrderEvent` 클래스에 `customerId` 필드를 추가하고 배포했다. Consumer는 아직 구버전 `OrderEvent`를 사용한다. Consumer에서 오류가 발생하는가?

<details>
<summary>해설 보기</summary>

**`@JsonIgnoreProperties(ignoreUnknown = true)` 설정 여부에 따라 다릅니다.**

설정 있는 경우 (권장):
```java
@JsonIgnoreProperties(ignoreUnknown = true)
public class OrderEvent { ... }
```
- 새 `customerId` 필드가 JSON에 있지만 Consumer 클래스에 없음
- `ignoreUnknown = true` → 모르는 필드 무시 → 정상 역직렬화
- **오류 없음** ✅

설정 없는 경우 (기본):
- `FAIL_ON_UNKNOWN_PROPERTIES = true` (Jackson 기본)
- `customerId` 필드를 모르는 필드로 인식 → `UnrecognizedPropertyException`
- **역직렬화 실패** ❌

**권장**: 모든 이벤트 DTO에 `@JsonIgnoreProperties(ignoreUnknown = true)` 또는 `ObjectMapper`에 `FAIL_ON_UNKNOWN_PROPERTIES = false` 설정. 롤링 배포의 필수 조건입니다.

</details>

---

**Q2.** 두 서비스(OrderService, PaymentService)가 메시지를 주고받는다. 이 두 서비스가 같은 DTO 클래스를 공유 라이브러리로 사용하는 것이 좋은가, 아니면 각자 독립적인 DTO를 사용하는 것이 좋은가?

<details>
<summary>해설 보기</summary>

**마이크로서비스 원칙에서는 독립 DTO가 권장됩니다.** 하지만 실용적으로는 규모에 따라 다릅니다.

**공유 라이브러리의 문제:**
- OrderEvent 필드 추가 → 공유 라이브러리 버전 업
- **모든 서비스 동시 배포 필요** (롤링 배포 불가)
- 강한 결합 → 마이크로서비스 독립성 훼손
- "이 필드는 OrderService만 쓰는데 PaymentService도 의존"

**독립 DTO의 장점:**
- OrderService의 `OrderPlacedEvent` / PaymentService의 `OrderReceivedEvent`
- 각 서비스가 자신에게 필요한 필드만 정의
- 타입 별칭으로 연결 (같은 별칭, 다른 클래스)
- 서비스 독립 배포 가능

**실용적 절충:**
- 소규모 팀 (2~3개 서비스): 공유 라이브러리도 무방 (관리 단순)
- 중규모 이상 (5개+ 서비스): 독립 DTO + 타입 별칭 권장

</details>

---

**Q3.** `content-type: application/json` 대신 `content-type: text/plain`으로 발행된 메시지를 `@RabbitListener`에서 `OrderEvent`로 역직렬화하려 한다. 어떻게 해야 하는가?

<details>
<summary>해설 보기</summary>

**`ContentTypeDelegatingMessageConverter`를 사용하거나 직접 변환 로직을 추가합니다.**

**방법 1: ContentTypeDelegatingMessageConverter**
```java
@Bean
public ContentTypeDelegatingMessageConverter converter() {
  ContentTypeDelegatingMessageConverter converter =
    new ContentTypeDelegatingMessageConverter();
  converter.addDelegate("application/json", new Jackson2JsonMessageConverter());
  converter.addDelegate("text/plain", customTextToJsonConverter());
  return converter;
}
```

**방법 2: Consumer에서 직접 처리**
```java
@RabbitListener(queues = "order.queue")
public void handle(Message message) {
  String contentType = message.getMessageProperties().getContentType();
  if ("text/plain".equals(contentType)) {
    String text = new String(message.getBody());
    // 텍스트를 파싱하여 OrderEvent 변환
    OrderEvent event = parseFromText(text);
    processOrder(event);
  }
}
```

**방법 3: 발행 측에서 content-type 수정 (근본 해결)**
발행 시 `content-type: application/json`으로 올바르게 설정하는 것이 가장 깔끔합니다. Consumer에서 content-type을 처리하는 것은 임시 방편입니다.

</details>

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[이전: Spring AMQP 아키텍처 ⬅️](./01-spring-amqp-architecture.md)** | **[다음: 에러 처리와 재시도 ➡️](./03-error-handling-retry.md)**

</div>
