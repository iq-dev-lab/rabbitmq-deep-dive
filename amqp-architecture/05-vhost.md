# 가상 호스트(vHost) — 브로커 내 논리적 격리

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- vHost는 무엇이고, 왜 하나의 브로커 안에서 논리적 분리가 필요한가?
- vHost 사이에는 어떤 것들이 공유되고, 어떤 것들이 격리되는가?
- 팀별, 환경별(dev/staging/prod) vHost 설계 전략은 어떻게 세우는가?
- vHost 권한 관리는 어떻게 동작하고, 잘못 설정하면 어떤 보안 문제가 생기는가?
- 실무에서 vHost를 언제 나누고 언제 하나의 vHost로 유지하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

개발 환경 메시지가 프로덕션 Queue에 섞이거나, A팀의 Consumer가 B팀의 Queue를 실수로 소비하는 사고는 vHost 없이 운영하면 실제로 발생한다. vHost는 단순한 네이밍 컨벤션이 아니라, 브로커 레벨의 격리를 보장하는 메커니즘이다. 잘 설계된 vHost 구조는 운영 사고를 예방하고, 권한 관리를 단순화하며, 환경 간 격리를 보장한다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: 모든 환경이 동일한 vHost "/" 사용

  개발팀 A: rabbit@prod에 연결, vHost="/"
            order.queue 사용
  
  개발팀 B: rabbit@prod에 연결, vHost="/"
            order.queue 사용 (같은 이름!)
  
  결과:
    팀 A가 테스트 메시지를 발행 → 팀 B의 Consumer가 소비
    팀 B의 Consumer가 테스트 데이터로 잘못 처리
    "내 서비스에 이상한 메시지가 들어왔다" 디버깅 불가

실수 2: vHost 없이 이름으로만 구분 시도

  개발 환경: "dev.order.queue"
  스테이징:  "staging.order.queue"
  프로덕션:  "prod.order.queue"
  
  문제:
    개발자 실수 → dev 앱이 "prod.order.queue"에 연결
    → 프로덕션 데이터 훼손
    권한 설정 없음 → 모든 사용자가 모든 Queue 접근 가능
    이름 관리가 복잡해짐

실수 3: admin 계정으로 모든 서비스 연결

  SPRING_RABBITMQ_USERNAME=admin
  SPRING_RABBITMQ_PASSWORD=admin
  (모든 서비스에 동일한 admin 계정 사용)
  
  문제:
    한 서비스의 credential 노출 → 모든 vHost 접근 가능
    어떤 서비스가 어떤 Queue를 소비하는지 추적 불가
    audit log에서 "admin이 뭘 했다" → 어느 서비스인지 불명확
```

---

## ✨ 올바른 접근 (After — vHost로 격리)

```
vHost 설계 원칙:

1. 환경별 분리:
   /dev       → 개발 환경
   /staging   → 스테이징 환경
   /prod      → 프로덕션 환경
   /test      → 테스트 환경

2. 또는 팀/도메인별 분리:
   /order-service    → 주문 도메인
   /payment-service  → 결제 도메인
   /notification     → 알림 도메인

3. 서비스별 전용 계정:
   order-service 계정: /prod vHost 접근만 허용
   payment-service 계정: /prod vHost 접근만 허용
   → 각 서비스는 자신의 vHost만 접근 가능

Spring Boot 설정:
  spring:
    rabbitmq:
      host: rabbitmq.internal
      port: 5672
      username: order-service
      password: ${RABBIT_PASSWORD}
      virtual-host: /prod   # 명시적 vHost 지정
```

---

## 🔬 내부 동작 원리

### 1. vHost의 격리 범위

```
=== vHost가 격리하는 것 ===

하나의 RabbitMQ 브로커:
┌─────────────────────────────────────────────────────────────┐
│ RabbitMQ Broker                                             │
│                                                             │
│  ┌──────────────────────┐  ┌──────────────────────────────┐ │
│  │ vHost: /dev          │  │ vHost: /prod                 │ │
│  │                      │  │                              │ │
│  │  Exchange:           │  │  Exchange:                   │ │
│  │  - order.exchange    │  │  - order.exchange (별개!)    │ │
│  │  - payment.exchange  │  │  - payment.exchange (별개!)  │ │
│  │                      │  │                              │ │
│  │  Queue:              │  │  Queue:                      │ │
│  │  - order.queue       │  │  - order.queue (별개!)       │ │
│  │  - notify.queue      │  │  - notify.queue (별개!)      │ │
│  │                      │  │                              │ │
│  │  Binding:            │  │  Binding:                    │ │
│  │  (독립적)             │  │  (독립적)                    │ │
│  └──────────────────────┘  └──────────────────────────────┘ │
│                                                             │
│  공유: 브로커 프로세스, Erlang VM, 포트, 물리 메모리/디스크    │
│  격리: Exchange, Queue, Binding, 권한, Policy               │
│                                                             │
└─────────────────────────────────────────────────────────────┘

vHost가 격리하는 것:
  ✅ Exchange: /dev의 order.exchange와 /prod의 order.exchange는 별개
  ✅ Queue: 이름이 같아도 vHost가 다르면 완전히 다른 Queue
  ✅ Binding: vHost 내 Exchange-Queue 연결만 가능
  ✅ 권한: 사용자별 vHost 접근 권한 분리
  ✅ Policy: vHost별 정책 (HA, TTL 등) 독립 설정

vHost가 격리하지 않는 것:
  ❌ 물리 메모리: 모든 vHost가 공유 → 한 vHost의 메모리 폭증이 전체 영향
  ❌ 네트워크 포트: 5672 하나로 모든 vHost
  ❌ Erlang 프로세스: 브로커 레벨 공유
  ❌ 디스크: 물리 디스크 공유 (I/O 경합 가능)
```

### 2. vHost 권한 관리

```
=== RabbitMQ 권한 모델 ===

사용자(User)는 vHost에 대해 세 가지 권한을 가짐:
  configure: Exchange/Queue 생성, 삭제, 설정 변경 권한
  write:     메시지 발행 (basicPublish) 권한
  read:      메시지 소비, Queue 검사 권한

권한 패턴: 정규식으로 리소스 범위 지정
  ".*"  → 모든 리소스
  "^$"  → 아무 리소스도 없음
  "^order\\..*" → "order."으로 시작하는 리소스만

=== 권한 설정 예시 ===

CLI:
  # vHost 생성
  rabbitmqctl add_vhost /prod
  rabbitmqctl add_vhost /dev
  
  # 사용자 생성
  rabbitmqctl add_user order-service strong-password-123
  rabbitmqctl add_user payment-service strong-password-456
  rabbitmqctl add_user monitor-user monitor-password
  
  # 권한 부여
  # rabbitmqctl set_permissions -p <vhost> <user> <configure> <write> <read>
  
  # order-service: /prod에서 order 관련 리소스만
  rabbitmqctl set_permissions -p /prod order-service \
    "^order\\..*" \    # configure: order.로 시작하는 것만 생성/삭제
    "^order\\..*" \    # write: order.로 시작하는 Exchange에만 발행
    "^order\\..*"      # read: order.로 시작하는 Queue에서만 소비
  
  # monitor-user: /prod에서 읽기만
  rabbitmqctl set_permissions -p /prod monitor-user \
    "^$" \    # configure: 없음
    "^$" \    # write: 없음
    ".*"      # read: 모든 Queue 조회 가능 (모니터링용)
  
  # admin: 모든 vHost 전체 권한
  rabbitmqctl set_permissions -p /prod admin ".*" ".*" ".*"
  rabbitmqctl set_permissions -p /dev admin ".*" ".*" ".*"

=== 권한 확인 ===

rabbitmqctl list_permissions -p /prod
# user              configure        write            read
# order-service     ^order\..*      ^order\..*      ^order\..*
# payment-service   ^payment\..*    ^payment\..*    ^payment\..*
# admin             .*              .*              .*

rabbitmqctl list_user_permissions order-service
# vhost   configure    write        read
# /prod   ^order\..*  ^order\..*  ^order\..*
```

### 3. vHost 설계 전략

```
=== 전략 1: 환경별 분리 ===

권장 구조:
  브로커 1개 + vHost 여러 개
  /dev → 개발자 로컬 및 CI 환경
  /staging → QA, 성능 테스트
  /prod → 실제 프로덕션

장점:
  브로커 운영 비용 최소 (1개)
  환경 간 메시지 격리
  환경별 다른 Policy 설정 가능

단점:
  물리 자원 공유 → /prod 폭주 시 /dev도 영향
  완전한 격리 필요 시 별도 브로커 필요

=== 전략 2: 도메인별 분리 ===

/order → 주문 도메인 (OrderService Producer/Consumer)
/payment → 결제 도메인 (PaymentService)
/notification → 알림 도메인 (NotificationService)

장점:
  서비스 간 메시지 흐름 격리
  도메인별 Policy 설정 가능

단점:
  서비스 간 이벤트 라우팅 필요 시 복잡 (vHost 간 직접 라우팅 불가)
  → 서비스 간 이벤트 공유가 많으면 하나의 vHost가 나을 수 있음

=== 전략 3: 혼합 구조 ===

/prod/order, /prod/payment, /prod/notification → 도메인별 격리
하지만 서비스 간 이벤트는 shovel 또는 federation 사용
(또는 하나의 공용 vHost /prod에 네이밍 컨벤션으로 구분)

=== vHost 간 메시지 라우팅의 한계 ===

중요: vHost 간에는 직접 메시지 라우팅 불가!

/payment의 Consumer가 /order의 Queue를 구독하는 것 불가
→ 서비스 간 이벤트 공유가 많으면 같은 vHost 사용 권장

vHost 간 연결이 필요하면:
  Shovel Plugin: 한 vHost의 Queue → 다른 vHost의 Exchange로 전달
  Federation Plugin: 여러 브로커/vHost 간 Exchange 연결
  → 복잡성 증가, 단순한 설계 권장
```

### 4. vHost와 Policy

```
=== Policy — vHost 레벨 일괄 설정 ===

Policy: vHost 내 Exchange/Queue에 일괄 적용되는 설정
  패턴 매칭으로 대상 리소스 선택
  Exchange/Queue에 직접 설정하지 않고 Policy로 관리

예시: /prod vHost의 모든 Queue에 HA 설정
  rabbitmqctl set_policy \
    -p /prod \
    ha-all \
    ".*" \              # 모든 Queue에 적용
    '{"ha-mode":"all"}' \  # 설정 내용
    --apply-to queues   # Queue에만 적용

예시: /prod vHost의 order.* Queue에 TTL 설정
  rabbitmqctl set_policy \
    -p /prod \
    order-ttl \
    "^order\\..*" \     # order.로 시작하는 Queue
    '{"message-ttl":3600000}' \  # 1시간 TTL
    --apply-to queues

Policy 우선순위:
  여러 Policy가 같은 Queue에 적용될 때 priority로 결정
  높은 priority가 낮은 것을 Override
  
  rabbitmqctl list_policies -p /prod
```

---

## 💻 실전 실험

### 실험 1: vHost 생성과 격리 확인

```bash
# vHost 생성
rabbitmqctl add_vhost /dev
rabbitmqctl add_vhost /prod

# 사용자 생성 및 권한 부여
rabbitmqctl add_user app-dev dev-password
rabbitmqctl add_user app-prod prod-password

rabbitmqctl set_permissions -p /dev app-dev ".*" ".*" ".*"
rabbitmqctl set_permissions -p /prod app-prod ".*" ".*" ".*"

# /dev에 Queue 생성
rabbitmqadmin -u app-dev -p dev-password --vhost /dev \
  declare queue name=order.queue durable=true

# /prod에 같은 이름의 Queue 생성
rabbitmqadmin -u app-prod -p prod-password --vhost /prod \
  declare queue name=order.queue durable=true

# /dev에서 메시지 발행
rabbitmqadmin -u app-dev -p dev-password --vhost /dev \
  publish exchange="" routing_key=order.queue payload="DEV 메시지"

# /prod Queue 확인 → 메시지 없음 (격리 확인)
rabbitmqadmin -u app-prod -p prod-password --vhost /prod \
  list queues name messages
# order.queue  0  ← /dev 메시지가 없음
```

### 실험 2: 권한 위반 시 동작 확인

```bash
# app-dev 계정으로 /prod에 접근 시도
rabbitmqadmin -u app-dev -p dev-password --vhost /prod \
  list queues
# Error: Access refused for user 'app-dev' to vhost '/prod'
# ← vHost 접근 권한 없음

# /dev에서 write 권한 없는 리소스 접근 시도
rabbitmqctl set_permissions -p /dev app-dev \
  ".*" "^$" ".*"  # write 권한 제거

rabbitmqadmin -u app-dev -p dev-password --vhost /dev \
  publish exchange="" routing_key=order.queue payload="test"
# Error: ACCESS_REFUSED - access to exchange '' in vhost '/dev' refused for user 'app-dev'
```

### 실험 3: Management UI에서 vHost별 모니터링

```
Management UI → Virtual Hosts (상단 Admin 탭)
  - 각 vHost별 Message Rate, Connection 수 확인
  - vHost 클릭 → Exchange, Queue, Connection 목록
  
Management UI → Overview
  - 하단 "Virtual Hosts" 섹션에서 vHost별 통계 확인
  - /dev vs /prod 트래픽 비교

rabbitmqctl list_vhosts  # CLI에서 vHost 목록 확인
rabbitmqctl list_vhosts name messages  # vHost별 메시지 수
```

---

## 📊 Kafka vs RabbitMQ 비교

```
=== 환경 격리 방식 비교 ===

RabbitMQ vHost:
  하나의 브로커에서 논리적 격리
  vHost 간 완전한 리소스 분리 (Exchange, Queue, Binding)
  사용자별 vHost 접근 권한 세밀 제어
  
  한계: 물리 자원(메모리, CPU)은 공유

Kafka의 환경 격리:
  주로 Topic 네이밍으로 구분: dev.order-events, prod.order-events
  ACL(Access Control List)로 Consumer Group / Topic 접근 제어
  환경별 Kafka 클러스터 분리가 더 일반적 (비용 높음)
  
  vHost 같은 개념 없음 → 네임스페이스 플러그인 존재하지만 기본 미지원

=== 멀티테넌시 비교 ===

여러 팀이 하나의 브로커를 공유해야 할 때:

RabbitMQ:
  팀별 vHost 생성 → 완전 격리
  팀별 계정 + 권한 설정
  Management UI에서 팀별 사용량 확인 가능
  → 멀티테넌시 지원 우수

Kafka:
  Topic 네이밍 컨벤션 + ACL로 격리 (vHost 미지원)
  완전 격리 위해서는 별도 클러스터 권장
  → 멀티테넌시 지원 제한적
```

---

## ⚖️ 트레이드오프

```
vHost 분리의 장단점:

장점:
  ① 환경/팀 간 메시지 격리 (실수 방지)
  ② 권한 관리 단순화 (vHost 단위로 접근 제어)
  ③ 독립적인 Policy 설정 (vHost별 HA, TTL 등)
  ④ Management UI에서 vHost별 모니터링

단점:
  ① 물리 자원 공유 → 완전한 격리 불가
  ② vHost 간 직접 라우팅 불가 (Shovel/Federation 필요)
  ③ 설정 복잡성 증가 (vHost별 Exchange/Queue 별도 생성)

vHost 수가 너무 많을 때 문제:
  Erlang 프로세스 증가 → 메모리 오버헤드
  Management UI 느려짐
  → 10개 이상의 vHost는 설계 재검토 권장

완전 격리가 필요한 경우:
  별도 RabbitMQ 클러스터 운영 (비용 증가)
  Cloud 환경: 환경별 별도 인스턴스 사용
```

---

## 📌 핵심 정리

```
vHost 핵심:

정의:
  하나의 RabbitMQ 브로커 안의 논리적 격리 단위
  Exchange, Queue, Binding, Policy가 vHost별로 독립

격리 범위:
  ✅ Exchange, Queue, Binding → 완전 격리
  ❌ 물리 메모리, CPU, 디스크 → 공유

권한:
  사용자별 vHost 접근 권한 (configure/write/read)
  정규식으로 리소스 세밀 제어
  서비스별 전용 계정 + 최소 권한 원칙

설계 전략:
  환경별 분리 (/dev, /staging, /prod) → 가장 일반적
  도메인별 분리 → 서비스 간 이벤트 공유 적을 때
  vHost 간 직접 라우팅 불가 → 서비스 간 이벤트 많으면 하나의 vHost

한계:
  물리 자원 공유 → 완전 격리 필요 시 별도 클러스터
  vHost 간 메시지 전달 → Shovel/Federation 필요 (복잡)
```

---

## 🤔 생각해볼 문제

**Q1.** `/prod` vHost에서 `order-service` 계정에게 configure 권한을 `"^$"` (없음)으로 설정했다. 이 서비스가 시작 시 `@Bean`으로 선언한 Queue를 `RabbitAdmin`이 자동 생성하려고 하면 어떻게 되는가?

<details>
<summary>해설 보기</summary>

`ACCESS_REFUSED` 에러가 발생하고 Queue 생성에 실패합니다.

`RabbitAdmin`은 Spring 컨텍스트 시작 시 `Queue.Declare` AMQP 명령을 보냅니다. 이 명령은 configure 권한이 필요합니다. `"^$"` (빈 패턴)는 어떤 리소스도 매칭하지 않으므로 Queue 선언이 거부됩니다.

해결 방법:
1. **Queue를 사전에 수동 생성**: 배포 전 인프라팀이 Queue를 생성, 서비스는 read/write만
2. **configure 권한 부여**: 특정 패턴의 Queue만 생성 가능하도록 `"^order\\..*"` 권한 부여
3. **ignoreDeclarationExceptions=true**: RabbitAdmin이 선언 실패를 무시하도록 설정 (이미 존재하는 Queue 연결)

프로덕션 환경에서는 서비스가 Exchange/Queue를 직접 생성하는 것보다 인프라팀이 미리 생성하고 서비스는 read/write만 하는 패턴이 더 안전합니다.

</details>

---

**Q2.** 하나의 RabbitMQ 브로커에 `/prod` vHost와 `/dev` vHost가 있다. 프로덕션 서비스가 대량 메시지 처리로 메모리를 80% 사용하면 개발 환경에 어떤 영향이 있는가?

<details>
<summary>해설 보기</summary>

**개발 환경도 영향을 받습니다.** vHost는 논리적 격리이고, 물리 메모리는 브로커가 공유합니다.

메모리 80% 도달 시 (`vm_memory_high_watermark` 기본값 0.4이면 이미 초과):
- 브로커가 **모든 vHost**의 Publisher에 Flow Control 발동
- `/dev` vHost의 메시지 발행도 차단됨
- 개발자가 "메시지가 안 보내진다"는 문제 경험

RabbitMQ 3.12의 Per-vHost 메모리 제한:
```bash
rabbitmqctl set_vhost_limits -p /prod '{"max-connections": 100, "max-queues": 1000}'
```
하지만 메모리 자체는 분리 불가.

**완전한 격리**가 필요한 경우:
- 환경별 별도 RabbitMQ 인스턴스/클러스터 운영
- Cloud 환경: 환경별 별도 관리형 서비스

결론: 공유 브로커의 vHost는 "논리적 격리"이지 "성능 격리"가 아닙니다. 프로덕션과 개발을 같은 브로커에서 운영할 때 이 한계를 인지해야 합니다.

</details>

---

**Q3.** `configure=".*"` 권한을 가진 `dev-user`가 `/prod` vHost에서 `order.exchange`를 삭제할 수 있는가?

<details>
<summary>해설 보기</summary>

**vHost 접근 권한이 있으면 가능합니다.** 이것이 최소 권한 원칙이 중요한 이유입니다.

권한 체계:
1. **vHost 접근 권한**: 해당 vHost에 접속 가능 여부 (`set_permissions -p /prod dev-user ...`)
2. **리소스 권한**: vHost 내에서 configure/write/read 범위

`dev-user`에게 `/prod` vHost 접근 권한 + `configure=".*"` 권한이 있으면:
- `/prod`의 `order.exchange` 삭제 가능
- `/prod`의 모든 Queue 삭제 가능

올바른 설정:
```bash
# dev-user는 /dev 접근만 허용, /prod는 권한 없음
rabbitmqctl set_permissions -p /dev dev-user ".*" ".*" ".*"
# /prod에는 set_permissions 호출하지 않음

# 별도 모니터링 계정은 /prod에서 read만
rabbitmqctl set_permissions -p /prod monitor-user "^$" "^$" ".*"
```

환경별 계정 분리가 중요합니다: `dev-user`는 `/dev`만, `prod-service`는 `/prod`만 접근하도록 설계.

</details>

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[이전: Queue 내부 구조 ⬅️](./04-queue-internals.md)** | **[다음: RabbitMQ 클러스터링 ➡️](./06-clustering.md)**

</div>
