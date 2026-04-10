# RPC(Remote Procedure Call) — Reply-To Queue로 요청-응답 구현

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Reply-To Queue와 Correlation ID로 RPC를 어떻게 구현하는가?
- RabbitMQ RPC의 타임아웃은 어떻게 처리하는가?
- 동기 HTTP 호출 대신 RabbitMQ RPC를 써야 하는 경우는 언제인가?
- 여러 클라이언트가 동시에 RPC를 호출할 때 응답이 뒤섞이지 않는 이유는?
- RabbitMQ RPC 패턴의 성능 한계와 대안은?

---

## 🔍 왜 이 개념이 실무에서 중요한가

비동기 메시지 시스템에서도 "요청 → 응답"이 필요한 경우가 있다. 재고 조회, 가격 계산, 인증 검증처럼 즉각적인 결과가 필요한 작업이다. HTTP REST는 이미 요청-응답이므로 RabbitMQ RPC는 언제 필요한가? 서비스 간 통신에 이미 RabbitMQ 인프라만 있거나, 브로커의 라우팅/재처리 기능을 RPC에서도 활용하고 싶을 때다. 그러나 대부분의 경우 HTTP가 더 단순하고 빠르다. 이 문서는 RPC 구현 원리와 함께 "쓰지 말아야 할 경우"를 명확히 한다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: Correlation ID 없이 RPC → 응답 뒤섞임

  Client A: 요청 발행 → reply.queue 대기
  Client B: 요청 발행 → reply.queue 대기

  서버: 두 요청 처리 → reply.queue에 응답 2개 발행
  Client A: 응답 1 수신 (B의 응답일 수도 있음!)
  Client B: 응답 2 수신 (A의 응답일 수도 있음!)
  → 응답 뒤섞임 → 데이터 오염

실수 2: 타임아웃 없는 RPC 대기 → 스레드 무한 점유

  future.get();  // 서버가 응답 안 하면 영원히 대기
  // 서버 장애 → 클라이언트 스레드 무한 대기
  // 스레드 풀 고갈 → 전체 서비스 마비

실수 3: 성능이 필요한 곳에 RabbitMQ RPC 사용

  초당 1000건 재고 조회를 RabbitMQ RPC로:
    요청 발행 → 서버 Queue 수신 → 처리 → 응답 발행
    → 클라이언트 응답 Queue 수신

  레이턴시: HTTP 직접 호출 ~5ms vs RabbitMQ RPC ~20~50ms
  브로커가 중간에 있으므로 항상 추가 지연 발생
  → 고처리량/저지연 RPC는 HTTP/gRPC가 적합
```

---

## ✨ 올바른 접근 (After — Correlation ID + 타임아웃)

```
RPC 완전한 구현:

Client:
  1. replyTo Queue 생성 (임시, exclusive)
  2. Correlation ID 생성 (UUID)
  3. replyTo, correlationId를 메시지 헤더에 포함하여 요청 발행
  4. replyTo Queue에서 응답 대기 (타임아웃 설정)
  5. 수신 응답의 correlationId 확인 → 내 요청 응답인지 검증

Server:
  1. 요청 Queue에서 수신
  2. 비즈니스 로직 처리
  3. replyTo Queue로 응답 발행 (correlationId 포함)

Spring AMQP RabbitTemplate:
  // DirectReplyTo 사용 (가장 간단)
  Object response = rabbitTemplate.convertSendAndReceive(
    "rpc.exchange",     // Exchange
    "inventory.check",  // Routing Key
    request,            // 요청 페이로드
    message -> {
      message.getMessageProperties().setReplyTo("amq.rabbitmq.reply-to");
      return message;
    }
  );
  // 기본 타임아웃: 5000ms (설정 가능)
  rabbitTemplate.setReplyTimeout(3000);  // 3초
```

---

## 🔬 내부 동작 원리

### 1. Reply-To Queue와 Correlation ID 메커니즘

```
=== RPC 전체 흐름 ===

[Client]                    [RabbitMQ]              [Server]
    │                           │                       │
    ├─ replyTo Queue 생성 ──────►│ amq.gen-ABC           │
    │                           │                       │
    ├─ 요청 발행 ───────────────►│ rpc.request.queue     │
    │   headers:                │                       │
    │   replyTo=amq.gen-ABC    ├─ 요청 전달 ────────────►│
    │   correlationId=uuid-1   │                       │
    │                           │                       ├─ 처리
    │                           │                       │
    │                           │◄── 응답 발행 ──────────┤
    │                           │    to=amq.gen-ABC     │
    │                           │    correlationId=uuid-1│
    │◄─ 응답 수신 ───────────────┤                       │
    │   correlationId=uuid-1   │                       │
    │   확인 → 내 요청 응답 ✅   │                       │

=== Correlation ID의 역할 ===

동시에 여러 클라이언트가 RPC 호출:

Client A: correlationId=uuid-A → rpc.queue
Client B: correlationId=uuid-B → rpc.queue

Server가 두 요청 처리:
  응답 A: replyTo=A.queue, correlationId=uuid-A 발행
  응답 B: replyTo=B.queue, correlationId=uuid-B 발행

(replyTo Queue가 클라이언트별로 다르면 라우팅으로 자동 분리)
(공유 replyTo Queue라면 correlationId로 구분)

Client A: replyTo=A.queue에서 응답 수신 → correlationId=uuid-A 확인 → 내 응답 ✅
Client B: replyTo=B.queue에서 응답 수신 → correlationId=uuid-B 확인 → 내 응답 ✅

=== DirectReplyTo (pseudo-queue) ===

RabbitMQ의 "amq.rabbitmq.reply-to":
  실제 Queue를 만들지 않고도 RPC 응답을 받는 특수 기능
  클라이언트 Connection의 채널에 직접 응답을 주입
  
  장점:
    Queue 생성/삭제 오버헤드 없음
    임시 Queue 관리 불필요
    응답 속도 빠름

  Spring AMQP 설정:
    rabbitTemplate.setUseDirectReplyToContainer(true);  // 기본값
    // DirectReplyTo를 사용하면 replyTo Queue 명시 불필요
```

### 2. 타임아웃 처리

```
=== RPC 타임아웃 시나리오 ===

정상:
  Client: 요청 발행 → CompletableFuture.get(5, SECONDS) 대기
  Server: 처리 → 응답 (3초 내)
  Client: 응답 수신 → 반환

타임아웃 (서버 장애):
  Client: 요청 발행 → 5초 대기
  Server: 응답 없음 (다운)
  Client: TimeoutException 발생

  처리:
    catch (TimeoutException e) {
      // 응답 없음 처리
      // 1. 기본값 반환
      // 2. 재시도 (한 번 더)
      // 3. 에러 응답
      // 4. 폴백 서비스 호출
    }

=== 타임아웃 후 늦은 응답 처리 ===

Client: 5초 후 TimeoutException → 폴백 처리
Server: 6초 후 응답 발행 (늦게 처리 완료)

문제:
  replyTo Queue에 늦은 응답이 쌓임
  다음 RPC 호출에서 이 응답을 수신할 수 있음 (correlationId 불일치)

해결:
  1. replyTo Queue에 TTL 설정 (예: x-message-ttl=10000)
     → 늦은 응답이 10초 후 자동 삭제
  2. correlationId 검증 → 불일치면 무시
     if (!expectedCorrelationId.equals(received.getCorrelationId())) {
       // 무시하고 새 메시지 대기
     }
  3. DirectReplyTo 사용 → 연결별 독립 채널로 뒤섞임 방지
```

### 3. RabbitMQ RPC vs HTTP: 선택 기준

```
=== RabbitMQ RPC가 유리한 경우 ===

1. 이미 RabbitMQ가 인프라에 있고 HTTP 엔드포인트 추가가 어려운 경우
2. 서버가 여러 Worker로 확장되어야 하고 로드밸런싱을 자동화하고 싶을 때
   (RabbitMQ가 Worker Queue처럼 자동 분배)
3. 요청 처리가 오래 걸려 HTTP 타임아웃 문제가 있을 때
   (RabbitMQ는 메시지 보관 → Worker 처리 → 응답까지 느슨한 결합)
4. RPC 요청의 재시도/DLX 등 신뢰성 메커니즘이 필요할 때

=== HTTP/gRPC가 더 적합한 경우 ===

1. 저지연이 필요한 경우:
   HTTP 직접 호출: ~5ms
   RabbitMQ RPC: ~20~50ms (브로커 경유 오버헤드)
   → 재고 조회, 가격 계산 등 즉각 응답 필요 시

2. 단순한 동기 요청-응답:
   브로커 없이 HTTP 직접 통신이 더 단순

3. 표준 프로토콜 필요 시:
   외부 파트너사와 통신, REST API 문서화 필요 시
   
4. 처리량 우선 시:
   HTTP/2 + gRPC가 RabbitMQ RPC보다 처리량 높음

결론:
  "RabbitMQ가 이미 있고, Worker 확장이 필요하고,
   결과를 바로 안 받아도 되면(약간의 지연 허용)" → RabbitMQ RPC
  "빠른 응답이 최우선" → HTTP/gRPC
```

---

## 💻 실전 실험

### 실험 1: Spring AMQP RPC 구현

```java
// Server (RPC 처리 서버)
@Component
public class InventoryServer {

  @RabbitListener(queues = "inventory.rpc.queue")
  public InventoryResponse checkInventory(InventoryRequest request) {
    // @RabbitListener의 return값이 자동으로 replyTo Queue로 전송
    int quantity = inventoryRepository.getQuantity(request.getItemId());
    return new InventoryResponse(request.getItemId(), quantity);
  }
}

// Client
@Service
public class OrderService {

  @Autowired private RabbitTemplate rabbitTemplate;

  public boolean isItemAvailable(String itemId, int required) {
    InventoryRequest request = new InventoryRequest(itemId, required);

    try {
      InventoryResponse response = (InventoryResponse)
        rabbitTemplate.convertSendAndReceive(
          "inventory.rpc.exchange",
          "inventory.check",
          request
        );

      if (response == null) {
        throw new ServiceUnavailableException("Inventory service timeout");
      }
      return response.getQuantity() >= required;

    } catch (AmqpException e) {
      log.error("RPC 실패", e);
      throw new ServiceUnavailableException("Inventory service unavailable");
    }
  }
}

// 설정
@Bean
public RabbitTemplate rabbitTemplate(ConnectionFactory cf) {
  RabbitTemplate template = new RabbitTemplate(cf);
  template.setReplyTimeout(3000);           // 3초 타임아웃
  template.setUseDirectReplyToContainer(true); // DirectReplyTo 사용
  return template;
}
```

### 실험 2: RPC 레이턴시 측정

```bash
# 기준 측정: HTTP 직접 호출 vs RabbitMQ RPC

# HTTP (ab benchmark):
ab -n 1000 -c 10 http://inventory-service/api/check?item=123
# Requests per second: 2000 rps, Mean: 5ms

# RabbitMQ RPC (간접 측정):
# Management UI → Queues → inventory.rpc.queue
# Message rates: publish + deliver + ack를 순환 측정
# 일반적으로 RabbitMQ RPC는 HTTP보다 3~10배 지연 높음
```

---

## 📊 Kafka vs RabbitMQ 비교

```
=== 요청-응답 패턴 비교 ===

RabbitMQ RPC:
  Reply-To Queue + Correlation ID
  브로커를 통한 요청-응답 (추가 지연)
  Worker 확장 자동 (Queue 공유)
  타임아웃 직접 구현 필요

Kafka Request-Reply (드물게 사용):
  요청 Topic + 응답 Topic + Correlation ID
  레이턴시가 더 높음 (Kafka는 스트리밍 최적화)
  실무에서 Kafka RPC는 거의 사용 안 함
  → Kafka 환경에서도 RPC는 HTTP/gRPC 사용

결론:
  메시지 기반 시스템에서 RPC: RabbitMQ가 상대적으로 나음
  하지만 대부분 HTTP/gRPC가 더 적합
```

---

## ⚖️ 트레이드오프

```
RabbitMQ RPC 장단점:

장점:
  ① Worker 자동 확장 (Queue가 로드밸런서 역할)
  ② 요청 유실 방지 (durable Queue + Persistent)
  ③ 이미 RabbitMQ 인프라 활용

단점:
  ① HTTP/gRPC보다 3~10배 높은 레이턴시
  ② 구현 복잡 (Correlation ID, Reply Queue 관리)
  ③ 타임아웃 처리 직접 구현 필요
  ④ 브로커 장애 시 RPC 전체 실패

사용 판단 기준:
  레이턴시 < 100ms 요구: HTTP/gRPC 사용
  레이턴시 100ms+ 허용: RabbitMQ RPC 고려
  Worker 확장이 핵심: RabbitMQ RPC 고려
```

---

## 📌 핵심 정리

```
RabbitMQ RPC 핵심:

구현 요소:
  Reply-To Queue: 응답을 받을 임시 Queue
  Correlation ID: 요청-응답 매핑 (동시 다중 RPC 구분)
  타임아웃: convertSendAndReceive에 replyTimeout 설정

DirectReplyTo:
  amq.rabbitmq.reply-to pseudo-queue
  실제 Queue 없이 Connection 채널로 직접 응답
  Spring AMQP 기본값 (권장)

선택 기준:
  저지연(< 100ms): HTTP/gRPC
  Worker 자동 확장 + 지연 허용: RabbitMQ RPC
  실무: 대부분 HTTP가 더 단순하고 빠름

타임아웃 후 처리:
  늦은 응답: correlationId 검증으로 무시
  replyTo Queue TTL로 오래된 응답 자동 삭제
```

---

## 🤔 생각해볼 문제

**Q1.** 10개 Worker가 RPC 요청 Queue를 처리한다. 응답 시간이 들쭉날쭉하다. 클라이언트가 받는 응답 순서는 요청 순서와 같은가?

<details>
<summary>해설 보기</summary>

**같지 않습니다.** RabbitMQ RPC는 응답 순서를 보장하지 않습니다.

시나리오:
- Client: 요청 A, 요청 B, 요청 C 순서로 발행
- Worker 1: 요청 B를 빠르게 처리 → 응답 B 먼저 발행
- Worker 2: 요청 A를 늦게 처리 → 응답 A 나중 발행

Client Reply Queue: [응답 B][응답 A][응답 C] (순서 뒤섞임)

해결 방법:
- **Correlation ID로 각 응답을 해당 요청에 매핑** (순서 무관, 각자 처리)
- CompletableFuture Map에 `correlationId → CompletableFuture` 저장

```java
Map<String, CompletableFuture<Response>> pending = new ConcurrentHashMap<>();

// 요청 발행 시
String corrId = UUID.randomUUID().toString();
pending.put(corrId, new CompletableFuture<>());
rabbitTemplate.convertAndSend(exchange, key, request, m -> {
  m.getMessageProperties().setCorrelationId(corrId);
  return m;
});

// 응답 수신 시
String corrId = props.getCorrelationId();
pending.get(corrId).complete(response);
```

RPC에서는 "순서"보다 "매핑"이 중요합니다.

</details>

---

**Q2.** RPC 서버가 처리 중 예외가 발생했다. Client는 어떻게 이 에러를 받아야 하는가?

<details>
<summary>해설 보기</summary>

RabbitMQ는 에러 응답 표준이 없으므로 **애플리케이션 레벨에서 에러 응답 형식**을 정의해야 합니다.

**방법 1: 에러 필드 포함 응답 객체**
```java
public class RpcResponse<T> {
  private T data;
  private String errorCode;
  private String errorMessage;
  private boolean success;
}

// 서버
@RabbitListener(queues = "rpc.queue")
public RpcResponse<InventoryData> handle(InventoryRequest req) {
  try {
    return RpcResponse.success(processRequest(req));
  } catch (Exception e) {
    return RpcResponse.error("INVENTORY_ERROR", e.getMessage());
  }
}

// 클라이언트
RpcResponse<InventoryData> response = (RpcResponse) rabbitTemplate
  .convertSendAndReceive(exchange, key, request);
if (!response.isSuccess()) {
  throw new InventoryException(response.getErrorCode());
}
```

**방법 2: 에러 시 응답 없음 (타임아웃으로 처리)**
- 서버 예외 → 응답 발행 안 함
- 클라이언트: 타임아웃 → 에러로 처리
- 단점: 타임아웃 시간만큼 대기

**방법 1이 권장**됩니다. 타임아웃과 비즈니스 에러를 구분할 수 있습니다.

</details>

---

**Q3.** RabbitMQ RPC와 HTTP REST 중 어떤 상황에서 RabbitMQ를 선택하는 것이 정당화되는가?

<details>
<summary>해설 보기</summary>

**RabbitMQ RPC가 정당화되는 구체적 상황:**

1. **Worker 풀 자동 확장이 핵심**
   - 처리 요청이 폭발적으로 증가 → Worker를 동적으로 추가
   - 로드밸런서 설정 없이 Queue가 자동 분배

2. **처리 시간이 길고 재처리가 필요**
   - 요청 처리가 수십 초 이상 → HTTP 타임아웃 회피
   - DLX로 처리 실패 재시도

3. **요청을 잃어서는 안 되는 경우**
   - HTTP: 클라이언트 재시도 없으면 유실
   - RabbitMQ: durable Queue에 보관 → Worker 재시작 후 처리

4. **이미 RabbitMQ 인프라가 있고 추가 서비스 엔드포인트가 부담**
   - HTTP 서버 추가 없이 Worker 형태로 기능 추가

**HTTP가 항상 더 나은 경우:**
- 낮은 레이턴시 요구 (< 50ms)
- 외부 공개 API
- 단순한 1:1 서비스 통신

결론: RabbitMQ RPC는 특수한 상황에서만 정당화됩니다. 기본 선택은 HTTP/gRPC.

</details>

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[이전: Publish/Subscribe ⬅️](./02-pubsub.md)** | **[다음: Priority Queue ➡️](./04-priority-queue.md)**

</div>
