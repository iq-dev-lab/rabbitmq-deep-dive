# AMQP 프로토콜 완전 분해 — Channel 멀티플렉싱과 3단계 라우팅

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- AMQP 0-9-1은 TCP 위에서 어떻게 동작하고, 프레임은 어떤 구조인가?
- Connection과 Channel의 차이는 무엇이고, 왜 하나의 Connection에 여러 Channel을 쓰는가?
- Channel 멀티플렉싱은 어떻게 하나의 TCP 연결에서 여러 논리 채널을 동시에 처리하는가?
- Exchange → Binding → Queue 3단계 라우팅은 왜 이렇게 설계되었는가?
- Connection/Channel을 잘못 관리하면 어떤 문제가 발생하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

Spring AMQP를 쓰면 Connection과 Channel은 자동으로 관리된다. 하지만 "Connection 풀이 고갈됐다"는 에러, "Channel이 닫혔다"는 에러, "Max Channel 수 초과"라는 에러는 AMQP 프로토콜을 모르면 원인을 파악할 수 없다. 프레임 구조와 Channel 멀티플렉싱을 이해해야 Connection Pool 설정, Channel 캐싱 전략, 스레드와 Channel의 관계를 올바르게 설정할 수 있다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: 요청마다 새 Connection 생성
  
  // 잘못된 패턴 — 매 메시지 발행마다 연결
  public void sendMessage(String message) {
    ConnectionFactory factory = new ConnectionFactory();
    Connection connection = factory.newConnection();  // TCP 연결 생성
    Channel channel = connection.createChannel();      // 채널 생성
    channel.basicPublish("exchange", "key", null, message.getBytes());
    channel.close();
    connection.close();  // TCP 연결 종료
  }
  
  결과:
    TCP 3-way handshake: 매 요청마다 발생
    AMQP Connection 협상: 매 요청마다 발생
    → 오버헤드가 메시지 자체보다 큼
    → 고부하 시 서버 연결 한계 도달 → 연결 거절

실수 2: Channel을 스레드 간 공유

  // 잘못된 패턴 — 여러 스레드가 하나의 Channel 공유
  private Channel sharedChannel;  // 싱글톤으로 관리
  
  public void send(String msg) {
    sharedChannel.basicPublish(...);  // 여러 스레드에서 동시 호출
  }
  
  결과:
    AMQP Channel은 스레드 안전하지 않음
    동시에 여러 스레드가 basicPublish 호출
    → 프레임이 뒤섞여 전송 → Channel 오류로 닫힘
    → "Channel is already closed" 에러

실수 3: Channel 누수

  // 잘못된 패턴 — Channel을 닫지 않음
  try {
    Channel channel = connection.createChannel();
    channel.basicPublish(...);
    // close() 누락
  } catch (Exception e) {
    // 예외 시 channel.close() 없음
  }
  
  결과:
    서버의 채널 한도 초과 (기본: 2047개)
    → "NOT_ALLOWED - channel_max exceeded" 에러
    → 새 Channel 생성 불가 → 서비스 장애
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설정)

```
올바른 패턴:
  1. Connection: 애플리케이션당 하나 (Connection Pool)
  2. Channel: 스레드당 하나 (Channel Caching)
  3. Channel은 반드시 try-with-resources 또는 finally에서 닫기

Spring AMQP 설정:
  @Bean
  public CachingConnectionFactory connectionFactory() {
    CachingConnectionFactory factory = new CachingConnectionFactory("localhost");
    factory.setUsername("admin");
    factory.setPassword("admin");
    
    // Channel 캐싱 설정 (기본값: CHANNEL)
    factory.setCacheMode(CachingConnectionFactory.CacheMode.CHANNEL);
    factory.setChannelCacheSize(25);     // 캐시할 Channel 수
    factory.setChannelCheckoutTimeout(1000); // Channel 대기 타임아웃 ms
    
    return factory;
  }

Connection vs Channel 비용:
  Connection 생성: TCP 연결 + AMQP 협상 (수십~수백 ms)
  → 재사용 필수, Connection Pool 운영
  
  Channel 생성: 기존 Connection 위에 논리 채널만 추가 (< 1ms)
  → 스레드마다 생성해도 상대적으로 저렴
  → 그래도 캐싱 권장

진단 명령어:
  rabbitmqctl list_connections  # 연결된 Connection 목록
  rabbitmqctl list_channels     # 현재 열린 Channel 목록
  rabbitmqctl list_channels connection_details  # Channel별 Connection 확인
```

---

## 🔬 내부 동작 원리

### 1. AMQP 0-9-1 프레임 구조

```
AMQP 메시지는 TCP 스트림을 통해 프레임(Frame) 단위로 전송됩니다.
하나의 메시지는 여러 프레임으로 나뉘어 전송됩니다.

=== AMQP 프레임 기본 구조 ===

┌──────────┬─────────────┬──────────────────────────────┬────────┐
│ Type (1B)│ Channel (2B)│       Payload                │ End(1B)│
│          │             │                              │ 0xCE   │
└──────────┴─────────────┴──────────────────────────────┴────────┘

Type (1바이트):
  1 = METHOD  : AMQP 명령어 (Queue.Declare, Basic.Publish 등)
  2 = HEADER  : 메시지 메타데이터 (Content-Type, Delivery-Mode 등)
  3 = BODY    : 메시지 실제 내용
  8 = HEARTBEAT: 연결 상태 확인 (Payload 없음)

Channel (2바이트):
  어느 논리 채널의 프레임인지 식별
  0 = Connection 레벨 프레임 (Connection.Open, Connection.Close 등)
  1~N = 각 Channel의 프레임

End (1바이트):
  항상 0xCE (프레임 경계 검증용)

=== 메시지 발행 프레임 시퀀스 ===

Producer → RabbitMQ:

  Frame 1: METHOD (Basic.Publish)
  ┌────────────────────────────────────────────┐
  │ Type: 1 (METHOD)  Channel: 1              │
  │ Payload:                                  │
  │   class-id: 60 (basic)                   │
  │   method-id: 40 (publish)                │
  │   exchange: "order.exchange"              │
  │   routing-key: "order.placed"            │
  │   mandatory: false                        │
  │   immediate: false                        │
  └────────────────────────────────────────────┘

  Frame 2: HEADER (Content Header)
  ┌────────────────────────────────────────────┐
  │ Type: 2 (HEADER)  Channel: 1              │
  │ Payload:                                  │
  │   class-id: 60 (basic)                   │
  │   body-size: 45 (바이트 수)               │
  │   content-type: "application/json"        │
  │   delivery-mode: 2 (persistent)          │
  │   message-id: "uuid-1234"                │
  │   timestamp: 1711234567                   │
  └────────────────────────────────────────────┘

  Frame 3: BODY (실제 메시지 내용)
  ┌────────────────────────────────────────────┐
  │ Type: 3 (BODY)    Channel: 1              │
  │ Payload:                                  │
  │   {"orderId": 1, "userId": 100, ...}      │
  └────────────────────────────────────────────┘

  큰 메시지는 여러 BODY 프레임으로 분할 전송:
  max-frame-size (기본 131072 = 128KB) 초과 시
  → Frame 3, Frame 3, Frame 3... 여러 개로 분할

=== AMQP 핸드셰이크 과정 ===

TCP 연결 수립 후:

Client → Server: "AMQP" + protocol header (버전 협상)
Server → Client: Connection.Start (서버 기능 안내)
Client → Server: Connection.StartOk (인증 정보 전달)
Server → Client: Connection.Tune (frame-max, channel-max, heartbeat 협상)
Client → Server: Connection.TuneOk (협상 수락)
Client → Server: Connection.Open (vHost 지정)
Server → Client: Connection.OpenOk

Channel 열기:
Client → Server: Channel.Open (channel-id: 1)
Server → Client: Channel.OpenOk

Queue 선언:
Client → Server: Queue.Declare (name, durable, exclusive, auto-delete)
Server → Client: Queue.DeclareOk (queue, message-count, consumer-count)
```

### 2. Channel 멀티플렉싱 — 하나의 TCP 연결에서 다수의 논리 채널

```
=== TCP 연결 vs AMQP 연결 vs Channel ===

TCP 연결 (물리):
  소켓 레벨 연결
  3-way handshake: SYN → SYN-ACK → ACK
  TLS 사용 시 추가 handshake
  생성 비용: 수십~수백 ms
  → 1개만 유지 (Connection Pool로 재사용)

AMQP Connection (논리, TCP 1:1 대응):
  AMQP 핸드셰이크로 확립
  인증, vHost, 프레임 크기 협상
  생성 비용: 수십 ms
  → 애플리케이션당 1개 권장

AMQP Channel (논리, Connection 1:N 대응):
  기존 Connection 위에 번호로 구분되는 가상 채널
  생성 비용: < 1ms (단순히 Channel.Open/OpenOk 교환)
  독립적인 상태 관리 (QoS, 트랜잭션, Confirm 등)
  → 스레드당 1개 권장

=== Channel 멀티플렉싱 구조 ===

  애플리케이션 (10개 스레드)
  │
  ├── Thread 1 → Channel 1 ─────────┐
  ├── Thread 2 → Channel 2 ──────── │
  ├── Thread 3 → Channel 3 ──────── │
  ├── Thread 4 → Channel 4 ──────── │   하나의 TCP 연결
  ├── Thread 5 → Channel 5 ──────── ├── ════════════════ → RabbitMQ
  ├── Thread 6 → Channel 6 ──────── │   (물리 연결 1개)
  ├── Thread 7 → Channel 7 ──────── │
  ├── Thread 8 → Channel 8 ──────── │
  ├── Thread 9 → Channel 9 ──────── │
  └── Thread 10 → Channel 10 ──────┘

TCP 스트림에서 프레임이 섞이는 방식:
  [CH1: PUBLISH] [CH3: ACK] [CH2: PUBLISH] [CH5: CONSUME] [CH1: CONFIRM]
  
  각 프레임의 Channel 필드로 어느 논리 채널의 메시지인지 식별
  RabbitMQ는 Channel별로 독립적인 상태를 유지

=== Channel이 TCP 연결을 절약하는 규모 ===

  멀티스레드 서버 (100 스레드):
    채널 없이 스레드당 TCP 연결: 100개 TCP 연결
    → 서버 포트/fd 자원 소비
    → RabbitMQ의 TCP Accept 부하
  
  Channel 멀티플렉싱 사용 시:
    TCP 연결: 1개
    Channel: 100개
    → TCP 연결 99개 절약
    → Channel 생성/소멸 비용 < 1ms vs TCP 연결 수십 ms

=== Channel은 왜 스레드 안전하지 않은가 ===

  Channel은 내부적으로 순서 있는 프레임 전송을 가정
  
  Thread A: basicPublish → Frame [Method][Header][Body] 전송 중
  Thread B: basicPublish 동시 호출 → Frame 끼어들기
  
  결과: [Method-A][Method-B][Header-A][Body-B][Header-B][Body-A]
  → 프레임 순서 깨짐 → 프로토콜 오류 → Channel 강제 종료
  
  해결: 스레드당 별도 Channel 사용
  Spring AMQP CachingConnectionFactory: 스레드별 Channel 캐싱
```

### 3. Exchange → Binding → Queue 3단계 라우팅 설계 이유

```
=== 왜 Producer가 Queue에 직접 발행하지 않는가 ===

직접 Queue 발행 방식 (AMQP가 택하지 않은 설계):
  Producer → Queue A 직접 발행
  Producer → Queue B 직접 발행
  
  문제:
    Producer가 Queue 이름을 알아야 함 → 결합도
    새 Queue 추가 시 Producer 코드 수정 필요
    하나의 메시지를 여러 Queue에 복사하려면 Producer가 N번 발행

3단계 라우팅 (AMQP 방식):
  Producer → Exchange (라우팅 결정) → Queue A
                                    → Queue B
                                    → Queue C
  
  Producer는 Exchange 이름과 Routing Key만 알면 됨
  Queue 추가/제거 → Exchange에 Binding만 변경
  Producer 코드 변경 없음

=== 3단계의 각 역할 ===

1단계: Exchange
  메시지를 받아 라우팅 규칙을 평가하는 컴포넌트
  자체로는 메시지를 저장하지 않음 (라우터 역할)
  유형: Direct, Topic, Fanout, Headers, Default
  
2단계: Binding
  Exchange와 Queue를 연결하는 규칙
  Binding Key (또는 헤더 매처)를 포함
  Exchange가 메시지의 Routing Key를 Binding Key와 비교하여
  어느 Queue로 보낼지 결정
  
3단계: Queue
  실제 메시지를 저장하는 컨테이너
  Consumer가 Queue에서 메시지를 꺼내감
  메시지 내구성, 만료, 우선순위 설정 가능

=== 라우팅 흐름 ===

Producer
  │ exchange="order.exchange"
  │ routingKey="order.payment.failed"
  ▼
Exchange (order.exchange, type=topic)
  │
  Binding 평가:
  ├── Binding Key: "order.#"           → payment-queue    ✅ 일치
  ├── Binding Key: "order.payment.*"   → audit-queue      ✅ 일치
  ├── Binding Key: "order.shipped"     → ship-queue       ❌ 불일치
  └── Binding Key: "#.failed"          → alert-queue      ✅ 일치
  │
  메시지를 일치하는 Queue들로 복사 전달
  ▼
payment-queue → Consumer A (결제 서비스)
audit-queue   → Consumer B (감사 서비스)
alert-queue   → Consumer C (알림 서비스)

핵심: Producer는 1번 발행했지만 3개 Queue에 전달됨
     Producer는 Queue 이름을 전혀 모름
```

### 4. AMQP Heartbeat — 연결 상태 감지

```
=== Heartbeat 필요성 ===

TCP 연결은 네트워크 장비(방화벽, NAT)에 의해 끊어질 수 있음
방화벽은 일정 시간 통신이 없는 TCP 연결을 강제로 끊기도 함
→ 애플리케이션은 연결이 끊긴 줄 모른 채 계속 메시지 발행 시도
→ "Connection reset by peer" 에러 또는 타임아웃 대기

=== Heartbeat 동작 ===

협상 (Connection.Tune):
  Server → Client: heartbeat: 60 (60초 간격 제안)
  Client → Server: Connection.TuneOk heartbeat: 60 (수락)

주기적 Heartbeat 프레임:
  매 60/2 = 30초마다 양방향으로 Heartbeat 프레임 전송
  (heartbeat 간격의 절반마다 전송)
  
  Frame: Type=8 (HEARTBEAT), Channel=0, Payload 없음

감지:
  2 * heartbeat 시간(120초) 동안 Heartbeat 미수신 시 연결 끊김 판단
  → Connection 재연결 시도

Spring AMQP 설정:
  factory.setRequestedHeartBeat(60);  // 60초 간격 (0이면 비활성화)
  
  주의: 60초 Heartbeat인데 방화벽 idle timeout이 30초이면 여전히 끊김
  → Heartbeat 간격 < 방화벽 idle timeout 으로 설정 필요
```

---

## 💻 실전 실험

### 실험 1: Connection과 Channel 수 모니터링

```bash
# RabbitMQ 기동 후 Connection/Channel 확인

# 현재 Connection 목록
rabbitmqctl list_connections \
  name peer_host peer_port state channels

# 현재 Channel 목록
rabbitmqctl list_channels \
  connection_details number consumer_count messages_unacknowledged

# Spring AMQP 앱 기동 후 확인
# CachingConnectionFactory 기본 설정:
# - Connection 1개
# - Channel 캐시 크기: 25개

# Management UI에서 확인:
# http://localhost:15672/#/connections
# http://localhost:15672/#/channels
```

### 실험 2: Channel 상태 확인 (Management UI)

```
Management UI → Channels 탭:
  Channel 목록에서 확인할 수 있는 정보:
  
  - Connection: 어느 Connection의 Channel인지
  - Channel: Channel 번호 (1, 2, 3...)
  - User: 인증 사용자
  - Mode: 일반 / Publisher Confirm / Transaction
  - State: running / flow (Publisher가 빠름) / blocked
  - Unacked: 미확인 메시지 수
  - Prefetch: Prefetch Count 설정값

  State별 의미:
    running → 정상
    flow    → Consumer보다 Publisher가 빠름 (flow control 발동)
    blocked → 메모리/디스크 임계값 초과로 Publisher 차단
```

### 실험 3: AMQP 프레임 직접 관찰 (Wireshark)

```bash
# Wireshark로 AMQP 프레임 캡처 (TLS 없는 개발 환경)

# AMQP 포트 5672 캡처 필터
# Wireshark 필터: tcp.port == 5672

# 또는 tcpdump로 텍스트 출력
sudo tcpdump -i lo -A -s0 port 5672 2>/dev/null | \
  grep -E "AMQP|exchange|routing|queue"

# 메시지 발행 시 다음 프레임 시퀀스 관찰:
# 1. AMQP 헤더 교환 (Connection 최초 연결)
# 2. Basic.Publish Method 프레임
# 3. Content Header 프레임 (메타데이터)
# 4. Content Body 프레임 (실제 내용)
# 5. Basic.Ack (Publisher Confirm 모드 시)
```

### 실험 4: Spring AMQP Channel 캐싱 동작 확인

```java
// CachingConnectionFactory 내부 동작 로그 활성화
// application.yml:
// logging:
//   level:
//     org.springframework.amqp.rabbit.connection: DEBUG

// 출력 로그 예시:
// Creating cached Rabbit Channel from AMQPChannel(1)
// Returning cached Channel: (AMQPChannel(1))
// Channel(1) acquired from cache
// Channel(1) returning to cache

// 캐시 통계 확인
@Autowired
CachingConnectionFactory connectionFactory;

// 캐시된 Channel 수 확인
ConnectionFactory.getCacheProperties()
// {channelCacheSize=25, idleChannelsInCache=3, activeChannels=1, ...}
```

---

## 📊 Kafka vs RabbitMQ 비교

```
=== 프로토콜 레벨 비교 ===

                | RabbitMQ (AMQP 0-9-1)     | Kafka (자체 프로토콜)
────────────────┼───────────────────────────┼──────────────────────────
전송 프로토콜    | TCP + AMQP                | TCP + Kafka Binary Protocol
연결 단위       | Connection + Channel       | 단순 TCP 연결
라우팅 레이어   | Exchange → Binding → Queue | Topic → Partition
메시지 단위     | Method + Header + Body     | Record (Key + Value + Header)
연결 비용       | Connection: 높음, Channel: 낮음 | 단순 TCP (중간)

=== 멀티플렉싱 방식 차이 ===

RabbitMQ:
  하나의 TCP 연결 위에 여러 Channel
  Channel별로 독립적인 상태 (Ack, Confirm, QoS)
  → 연결 수 절약, 프로토콜 복잡성 증가

Kafka:
  Topic/Partition별로 별도 TCP 연결을 맺는 경우 많음
  Producer는 각 Partition Leader 브로커에 직접 연결
  → 더 단순한 프로토콜, 브로커 간 직접 통신
```

---

## ⚖️ 트레이드오프

```
AMQP Channel 멀티플렉싱의 장단점:

장점:
  ① TCP 연결 수 최소화 — 서버 자원 절약
  ② Channel별 독립 상태 — 스레드별 다른 QoS/Confirm 설정 가능
  ③ Channel 생성 비용 낮음 — 기존 Connection 재사용

단점:
  ① Channel 스레드 안전 아님 — 스레드당 별도 Channel 필요
  ② Channel 수 제한 — channel_max 설정값 초과 불가
  ③ 하나의 Connection 장애 — 모든 Channel 동시 장애

Connection 1개 vs 여러 개:
  Connection 1개: TCP 연결 절약, 단일 실패 지점
  Connection 여러 개: TCP 연결 증가, 실패 격리 (발행/소비 분리)
  
  Spring AMQP 권장:
    발행용 Connection 1개 + 소비용 Connection 1개 (격리)
    factory.setPublisherConnectionFactory(publisherFactory)
```

---

## 📌 핵심 정리

```
AMQP 프로토콜 핵심:

프레임 구조:
  Type(1B) + Channel(2B) + Payload + End(0xCE)
  Method 프레임: AMQP 명령어 (Publish, Consume 등)
  Header 프레임: 메시지 메타데이터 (delivery-mode, content-type)
  Body 프레임: 실제 메시지 내용 (128KB 초과 시 분할)

Connection vs Channel:
  Connection: TCP + AMQP 협상 (수십 ms, 재사용 필수)
  Channel: 논리 채널 (< 1ms, 스레드당 하나)
  스레드당 Channel 1개 → 스레드 안전

3단계 라우팅:
  Producer → Exchange (라우팅 결정)
           → Binding (규칙 평가)
           → Queue (메시지 저장)
  Producer는 Exchange + Routing Key만 알면 됨
  → Queue 추가/제거 시 Producer 코드 변경 없음

주요 설정:
  channel_max: Channel 수 상한 (기본 2047)
  frame_max: 최대 프레임 크기 (기본 128KB)
  heartbeat: 연결 상태 확인 간격 (기본 60초)
```

---

## 🤔 생각해볼 문제

**Q1.** `channel_max=2047`인 RabbitMQ에 스레드 100개짜리 Spring 앱이 10개 연결되어 있다. 각 스레드가 Channel을 1개씩 사용하면 총 몇 개의 Channel이 필요하고, 이것이 한계를 초과하는가?

<details>
<summary>해설 보기</summary>

필요한 Channel 수: 10개 앱 × 100개 스레드 = 1,000개 Channel

channel_max=2047이므로 이론상 초과하지 않습니다. 그러나 이것은 **하나의 Connection**에서의 Channel 수 제한입니다.

실제 계산:
- 앱 10개가 각각 **별도 Connection** 사용 시: 각 Connection당 100개 Channel = 1,000개/Connection → 제한 내
- 앱 10개가 **하나의 Connection 공유** 시: 1,000개 Channel = 1,000개/Connection → 2047 이내지만 단일 실패 지점

Spring AMQP `CachingConnectionFactory`의 `channelCacheSize` 설정:
- 기본값 25 → 동시 발행 스레드가 25개 초과 시 Channel 대기 발생
- 스레드 수에 맞게 조정 필요

`channelCheckoutTimeout` 설정 없으면 Channel 무한 대기 가능 → 타임아웃 설정 권장

</details>

---

**Q2.** Heartbeat를 0으로 설정하면 어떤 장점과 단점이 있는가? 언제 0으로 설정하는 것이 합리적인가?

<details>
<summary>해설 보기</summary>

Heartbeat=0 설정:
- 장점: Heartbeat 프레임 오버헤드 없음, 방화벽 idle timeout보다 짧게 설정하지 않아도 됨
- 단점: 네트워크 장애나 방화벽 timeout으로 연결이 끊겨도 감지 못함 → "좀비 Connection" 발생

좀비 Connection:
- TCP 스택에서는 연결이 끊겼지만 AMQP 레이어에서는 살아있다고 인식
- basicPublish 시도 → 타임아웃까지 대기 → 뒤늦게 에러
- 서비스 장애 복구 시간 크게 증가

합리적인 0 설정:
- 같은 서버 내 loopback 연결 (방화벽 없음, 네트워크 장애 없음)
- 로드 테스트 환경 (Heartbeat 오버헤드 최소화)
- 연결 시간이 매우 짧은 경우 (배치 작업 후 즉시 종료)

운영 환경에서는 반드시 Heartbeat > 0, 방화벽 idle timeout보다 작게 설정 권장

</details>

---

**Q3.** 하나의 AMQP 메시지가 여러 BODY 프레임으로 분할되어 전송될 때, 중간에 Channel이 닫히면 어떤 일이 발생하는가?

<details>
<summary>해설 보기</summary>

AMQP 프로토콜의 프레임 순서는 `[Method][Header][Body...]` 입니다. 분할 전송 중 Channel이 닫히면:

1. RabbitMQ가 불완전한 메시지를 수신 중임을 인식
2. Channel.Close 또는 연결 끊김 수신 시 해당 Channel의 모든 미완성 메시지 폐기
3. **Publisher Confirm 모드** 사용 시: 미완성 메시지에 대한 Nack(또는 Return) 발송
4. **Publisher Confirm 모드 미사용 시**: 메시지 유실 — Producer는 메시지가 전달됐는지 알 수 없음

이것이 Publisher Confirm 모드를 사용해야 하는 이유 중 하나입니다. Confirm 없이는 대용량 메시지 전송 중 네트워크 장애로 인한 유실을 감지할 수 없습니다.

max-frame-size를 줄이면 각 Body 프레임이 작아지고 더 많은 프레임으로 분할됩니다. 프레임 수가 늘어나지만 각 프레임이 완전히 전달되는 확률이 높아지는 것은 아닙니다. Publisher Confirm이 유일한 확실한 해결책입니다.

</details>

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[이전: RabbitMQ가 해결하는 문제 ⬅️](./01-why-rabbitmq.md)** | **[다음: Exchange 완전 분해 ➡️](./03-exchange-internals.md)**

</div>
