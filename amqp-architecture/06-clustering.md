# RabbitMQ 클러스터링 — Quorum Queue와 Split-Brain 처리

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- RabbitMQ 클러스터는 어떻게 구성되고, 노드 간에 무엇이 공유되는가?
- Classic Mirrored Queue는 왜 RabbitMQ 3.12에서 제거되었는가?
- Quorum Queue의 Raft 합의는 어떻게 노드 장애를 허용하는가?
- 네트워크 파티션(Split-Brain)이 발생하면 RabbitMQ는 어떻게 반응하는가?
- 프로덕션 클러스터를 안전하게 운영하기 위한 최소 노드 수는 몇 개인가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

단일 RabbitMQ 노드는 SPOF(Single Point of Failure)다. 노드가 재시작되는 수십 초 동안 메시지 발행과 소비가 전부 중단된다. 클러스터를 구성하면 노드 장애에도 서비스가 계속된다. 하지만 클러스터를 잘못 이해하면 더 위험하다. "클러스터를 구성했으니 안전하다"고 생각했지만, Classic Queue를 그대로 쓰면 마스터 노드 장애 시 데이터 유실과 서비스 중단이 동시에 발생한다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: 클러스터 구성 후 Classic Queue 그대로 사용

  클러스터 3노드 구성:
    Node 1, Node 2, Node 3

  Queue "order.queue" 선언 → Node 1에 마스터로 저장
  
  Node 1 장애 발생:
    "order.queue" 접근 불가 (Node 1에만 있으므로)
    → Consumer: 에러 발생, 메시지 소비 불가
    → Producer: "order.queue"로 메시지 발행 불가
    → Node 1 복구 전까지 서비스 중단

  착각:
    "클러스터를 구성했으니 하나가 죽어도 괜찮다"
    → 클러스터가 Queue를 복제하지 않는다는 것을 몰랐음

실수 2: Mirrored Queue를 모든 노드에 설정 (구식, 3.12에서 제거)

  Policy: ha-mode=all → 모든 노드에 동기 미러링
  
  문제:
    쓰기 연산: 모든 노드에 동기 복제 → 노드 수만큼 지연 증가
    노드 추가 시: 기존 메시지 전체 동기화 → 클러스터 부하 폭증
    네트워크 파티션 시: 데이터 불일치 위험
    RabbitMQ 3.12에서 공식 제거됨

실수 3: 홀수 노드 없이 클러스터 구성

  2노드 클러스터:
    Node 1, Node 2
    Node 1 장애 → Node 2 혼자 = 과반수? 아님 (2/2의 과반수 = 2 필요)
    → Quorum Queue: 쓰기 불가 (과반수 미충족)
    → 실제로 의미 있는 고가용성이 안 됨

  올바른 구성: 3노드 이상 홀수 (3, 5, 7...)
```

---

## ✨ 올바른 접근 (After — 클러스터 올바른 구성)

```
프로덕션 클러스터 기본 구성:

노드 수: 3개 (최소) 또는 5개 (높은 가용성)
Queue 유형: Quorum Queue (Classic Queue 사용 지양)
Load Balancer: HAProxy 또는 AWS NLB로 노드 앞에 배치

3노드 클러스터:
  Node 1 장애 허용 (과반수 2/3 충족)
  총 1개 노드 장애 허용

5노드 클러스터:
  Node 1, 2 동시 장애 허용 (과반수 3/5 충족)
  동시 2개 노드 장애 허용

Docker Compose 클러스터:
  rabbitmq-node1:
    image: rabbitmq:3.12-management
    environment:
      RABBITMQ_ERLANG_COOKIE: 'secret-cookie'
      RABBITMQ_NODENAME: 'rabbit@rabbitmq-node1'
    volumes:
      - ./cluster-entrypoint.sh:/usr/local/bin/cluster-entrypoint.sh
    entrypoint: cluster-entrypoint.sh
  
  rabbitmq-node2:
    image: rabbitmq:3.12-management
    environment:
      RABBITMQ_ERLANG_COOKIE: 'secret-cookie'
      RABBITMQ_NODENAME: 'rabbit@rabbitmq-node2'
  
  rabbitmq-node3:
    image: rabbitmq:3.12-management
    environment:
      RABBITMQ_ERLANG_COOKIE: 'secret-cookie'
      RABBITMQ_NODENAME: 'rabbit@rabbitmq-node3'
```

---

## 🔬 내부 동작 원리

### 1. RabbitMQ 클러스터 구성 원리

```
=== 클러스터에서 공유되는 것 ===

┌─────────────────────────────────────────────────────────────────┐
│                    RabbitMQ Cluster                             │
│                                                                 │
│  Node 1          Node 2          Node 3                        │
│  ┌───────────┐  ┌───────────┐  ┌───────────┐                  │
│  │ Exchange  │  │ Exchange  │  │ Exchange  │                   │
│  │ (복제됨)  │  │ (복제됨)  │  │ (복제됨)  │ ← 메타데이터 공유  │
│  │           │  │           │  │           │                   │
│  │ vHost     │  │ vHost     │  │ vHost     │ ← 메타데이터 공유  │
│  │ (복제됨)  │  │ (복제됨)  │  │ (복제됨)  │                   │
│  │           │  │           │  │           │                   │
│  │ Queue A   │  │           │  │           │ ← Classic Queue:  │
│  │ (데이터)  │  │           │  │           │   단일 노드에만    │
│  │           │  │           │  │           │                   │
│  │           │  │ Queue B   │  │ Queue B   │ ← Quorum Queue:  │
│  │ Queue B   │  │ (복제본)  │  │ (복제본)  │   과반수에 복제   │
│  │ (Leader)  │  │           │  │           │                   │
│  └───────────┘  └───────────┘  └───────────┘                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘

클러스터에서 공유 (모든 노드 동기화):
  ✅ Exchange 정의 (이름, 유형, 속성)
  ✅ Queue 정의 (이름, 속성) — 실제 데이터는 별개
  ✅ Binding 정의
  ✅ vHost 정의
  ✅ 사용자/권한 정의
  ✅ Policy 정의

클러스터에서 복제되지 않는 것 (Classic Queue 기준):
  ❌ Queue의 메시지 데이터 — 마스터 노드에만 저장
  ❌ Consumer 상태 (Unacked 메시지 등)

핵심:
  클러스터를 구성해도 Classic Queue의 메시지는 하나의 노드에만 있음
  Quorum Queue만이 메시지 데이터를 과반수 노드에 복제
```

### 2. Quorum Queue의 Raft 합의 상세

```
=== Raft 알고리즘 핵심 ===

Raft 역할:
  Leader: 모든 쓰기(메시지 발행, Ack) 처리
  Follower: Leader의 로그를 복제
  Candidate: Leader 선출 과정 중 상태

=== 메시지 발행 (쓰기) 과정 ===

Producer → RabbitMQ 클러스터 (Load Balancer):

  1. 메시지 수신 (Leader 노드)
     Node 1 (Leader): 메시지 받음
     Raft 로그에 항목 추가: [index=42, "order.placed", {...}]

  2. Follower에게 복제 (AppendEntries RPC)
     Node 1 → Node 2: "index=42 추가해줘"
     Node 1 → Node 3: "index=42 추가해줘"
     Node 2: 로컬에 추가, "OK" 응답
     Node 3: 로컬에 추가, "OK" 응답

  3. 과반수 확인 (Commit)
     Node 1: Node 2 응답 받음 → 과반수(2/3) 달성
     → 메시지 Commit (내구성 보장)
     → Producer에게 Basic.Ack 전송
     
     Node 3 응답 지연 중이어도 OK (과반수이므로)

  4. 소비
     Consumer → Node 1 (Leader): 메시지 요청
     Leader가 메시지 전달 → Consumer Ack
     → 모든 노드에서 해당 로그 항목 삭제

=== Leader 선출 과정 ===

Node 1 (Leader) 장애:

  1. Node 2, Node 3: Heartbeat 수신 안 됨 (election timeout, 150~300ms)
  2. Node 2: "나는 Candidate다. 투표해줘" → Node 3에 요청
  3. Node 3: Node 2의 로그 상태 확인
     - Node 2의 마지막 로그 인덱스 >= 자신의 것 → 투표 허용
  4. Node 2: 과반수(자신 + Node 3 = 2/3) 투표 받음 → Leader 선출
  5. Node 2: 새 Leader 선언, 새 term 시작
  6. Producer/Consumer가 새 Leader 감지:
     - Connection 재연결 → Node 2로 라우팅
  
  소요 시간: 수 초 이내 (election timeout + 재연결)

=== 로그 일관성 보장 ===

"왜 Leader 장애 시 메시지가 소실되지 않는가?"

Raft의 커밋 조건:
  과반수 노드에 로그 복제 완료 = 커밋
  커밋된 항목 = 어떤 새 Leader도 반드시 포함

예시:
  3노드 클러스터
  index=42 커밋됨 (Node 1, 2, 3 모두 복제)
  Node 1 장애 → Node 2가 Leader 선출
  Node 2는 index=42 보유 → 소실 없음

  index=43 미커밋 (Node 1에만 있었음, 과반수 도달 전)
  Node 1 장애 → index=43 소실 (커밋 안 됐으므로 Producer에게 Nack)
  → Producer가 재발행
```

### 3. 네트워크 파티션 — Split-Brain 처리

```
=== 네트워크 파티션이란 ===

3노드 클러스터에서 네트워크 장애:
  
  [Node 1] ← 네트워크 단절 → [Node 2] [Node 3]
  
  Node 1은 "나만 살아있다. 내가 리더다"
  Node 2, 3은 "Node 1이 죽었다. 우리끼리 리더를 선출하자"
  
  결과 (잘못된 처리 시):
    Node 1도 쓰기 처리
    Node 2도 쓰기 처리
    → 두 파티션에서 서로 다른 데이터 생성 → 불일치

=== RabbitMQ Quorum Queue의 파티션 처리 ===

Raft 기반 Quorum Queue:
  과반수 없으면 쓰기 거절 (안전 우선)
  
  [Node 1] 단독 파티션:
    과반수 = 2/3 필요
    Node 1 혼자 = 1/3 → 과반수 미달
    → 쓰기(발행) 거절 (서비스 불가하지만 데이터 손실 없음)
  
  [Node 2, Node 3] 파티션:
    과반수 = 2/3
    Node 2, 3 = 2/3 → 과반수 달성
    → 정상 운영 지속

결론:
  과반수 쪽 파티션: 계속 정상 운영
  소수 파티션: 쓰기 거절 (데이터 일관성 보장)
  Split-Brain 데이터 불일치 없음 (Raft의 핵심 보장)

=== 파티션 복구 후 ===

네트워크 복구:
  Node 1: "내가 과반수 없는 동안 뭔가 놓쳤다"
  → Node 2 (새 Leader)에게 누락된 로그 요청
  → 동기화 완료 후 Follower로 복귀

=== partition-handling 설정 (Classic Queue) ===

Classic Queue의 경우 (구식, 참고용):
rabbitmq.conf:
  cluster_partition_handling = pause_minority  # 권장
  # pause_minority: 소수 파티션이 일시 정지
  # autoheal: 파티션 복구 시 자동 병합 (데이터 유실 가능)
  # ignore: 아무것도 하지 않음 (Split-Brain 방치 위험)

Quorum Queue 사용 시 이 설정이 크게 중요하지 않음
(Raft 자체가 Split-Brain을 방지)
```

### 4. 클러스터 노드 연결과 Erlang Cookie

```
=== Erlang Cookie — 클러스터 인증 ===

RabbitMQ는 Erlang VM 위에서 동작
Erlang 노드 간 통신은 shared secret (cookie)으로 인증

클러스터 구성 조건:
  1. 모든 노드의 Erlang Cookie가 동일해야 함
  2. 노드 이름: rabbit@hostname 형식
  3. 네트워크에서 상호 접근 가능 (포트 4369: Erlang Port Mapper)

cookie 설정:
  파일: /var/lib/rabbitmq/.erlang.cookie
  환경변수: RABBITMQ_ERLANG_COOKIE=secret-cookie
  
  보안: cookie가 유출되면 클러스터에 악의적 노드 참여 가능
  → 복잡한 랜덤 문자열 사용 권장

=== 노드 추가 절차 ===

1. 새 노드(Node 3) 시작
   RABBITMQ_ERLANG_COOKIE=secret-cookie rabbitmq-server

2. 클러스터에 참여
   rabbitmqctl stop_app
   rabbitmqctl reset
   rabbitmqctl join_cluster rabbit@node1  # Node 1에 참여
   rabbitmqctl start_app

3. 클러스터 상태 확인
   rabbitmqctl cluster_status
   # Running nodes: rabbit@node1, rabbit@node2, rabbit@node3
   
4. Quorum Queue 복제 범위 자동 확장
   rabbitmqctl grow_quorum_queue order.queue rabbit@node3
   # 또는 policy로 자동 관리
```

---

## 💻 실전 실험

### 실험 1: 3노드 클러스터 구성

```bash
# Docker Compose로 3노드 클러스터

# cluster-entrypoint.sh
#!/bin/bash
rabbitmq-server &
sleep 10

if [ "$RABBITMQ_NODENAME" != "rabbit@rabbitmq-node1" ]; then
  rabbitmqctl stop_app
  rabbitmqctl reset
  rabbitmqctl join_cluster rabbit@rabbitmq-node1
  rabbitmqctl start_app
fi
wait

# 클러스터 상태 확인
docker exec rabbitmq-node1 rabbitmqctl cluster_status
# Cluster status of node rabbit@rabbitmq-node1
# Disk Nodes:
#   rabbit@rabbitmq-node1
#   rabbit@rabbitmq-node2
#   rabbit@rabbitmq-node3
# Running Nodes:
#   rabbit@rabbitmq-node1
#   rabbit@rabbitmq-node2
#   rabbit@rabbitmq-node3
# alarms: []
```

### 실험 2: Quorum Queue 노드 장애 허용 테스트

```bash
# Quorum Queue 생성
rabbitmqadmin declare queue name=resilient.queue durable=true \
  arguments='{"x-queue-type": "quorum"}'

# 메시지 100개 발행
for i in $(seq 1 100); do
  rabbitmqadmin publish exchange="" routing_key=resilient.queue \
    payload="{\"id\": $i}"
done

# Node 2 강제 종료
docker stop rabbitmq-node2

# 클러스터 상태 확인 (Node 2 장애)
rabbitmqctl cluster_status
# Running: rabbit@rabbitmq-node1, rabbit@rabbitmq-node3  (2/3 과반수)

# Queue 여전히 접근 가능 확인
rabbitmqctl list_queues name messages
# resilient.queue  100  ← 메시지 전부 보존, 접근 가능

# 메시지 발행 계속 가능
rabbitmqadmin publish exchange="" routing_key=resilient.queue \
  payload='{"id": 101}'

# Node 2 복구
docker start rabbitmq-node2
sleep 10

# Node 2 자동 동기화 확인
rabbitmqctl quorum_status resilient.queue
# 3노드 모두 최신 상태
```

### 실험 3: Management UI에서 클러스터 모니터링

```
Management UI → Overview:
  Nodes 섹션:
  ┌─────────────────┬──────────┬──────────┬──────────┐
  │ Node            │ Mem Used │ Disk Free│ Uptime   │
  ├─────────────────┼──────────┼──────────┼──────────┤
  │ rabbit@node1 ✅ │ 180MB    │ 50GB     │ 2d 3h    │
  │ rabbit@node2 ✅ │ 190MB    │ 48GB     │ 2d 3h    │
  │ rabbit@node3 ✅ │ 175MB    │ 52GB     │ 2d 3h    │
  └─────────────────┴──────────┴──────────┴──────────┘

Management UI → Queues → resilient.queue:
  Leader: rabbit@node1
  Members: rabbit@node1, rabbit@node2, rabbit@node3
  Online: rabbit@node1, rabbit@node2, rabbit@node3
  
  노드 장애 시:
  Online: rabbit@node1, rabbit@node3
  Offline: rabbit@node2 (빨간 표시)
```

---

## 📊 Kafka vs RabbitMQ 비교

```
=== 클러스터링 방식 비교 ===

RabbitMQ 클러스터링:
  Raft 기반 Quorum Queue (3.8+)
  과반수 노드 확인 후 커밋
  Exchange/Queue 메타데이터 전체 복제
  로드 밸런서가 클라이언트 연결 분산
  
  적합: 고가용성 메시지 브로커, 낮은 지연

Kafka 클러스터링:
  ISR(In-Sync Replicas) 기반 복제
  Partition Leader가 쓰기 처리, Follower가 복제
  min.insync.replicas로 최소 동기화 수 설정
  Controller 노드가 Leader 선출 관리 (KRaft 모드)
  
  적합: 대용량 스트리밍, 높은 처리량

=== 장애 허용 비교 ===

3노드 클러스터 기준:

RabbitMQ (Quorum Queue):
  허용 장애: 1노드 (과반수 2/3 필요)
  Leader 자동 선출: Raft 투표 (수 초)
  데이터 일관성: 강한 일관성 (커밋 = 과반수 확인)

Kafka (replication-factor=3, min.insync.replicas=2):
  허용 장애: 1노드 (ISR 2개 이상 필요)
  Leader 자동 선출: Controller 관리 (수 초~수십 초)
  데이터 일관성: 강한 일관성 (min.insync.replicas 설정 시)
```

---

## ⚖️ 트레이드오프

```
클러스터 크기별 트레이드오프:

3노드 클러스터:
  ✅ 1개 노드 장애 허용
  ✅ 운영 비용 최소
  ❌ 동시 2개 장애 시 서비스 중단

5노드 클러스터:
  ✅ 동시 2개 노드 장애 허용
  ✅ 더 높은 가용성
  ❌ 더 많은 서버 비용
  ❌ Raft 합의 오버헤드 증가 (5노드 확인)

Quorum Queue vs Classic Queue 성능:
  Classic (단일 노드): 처리량 높음, 지연 낮음, 노드 장애 시 접근 불가
  Quorum (3노드): 처리량 약간 낮음, 지연 약간 높음, 노드 장애 허용

클러스터 vs 단일 노드 + 빠른 재시작:
  클러스터: 고가용성, 비용/복잡성 증가
  단일 노드 + Durable Queue: 낮은 비용, 재시작 시 수십 초 중단
  → 요구사항에 따라 선택 (99.9% vs 99.99% 가용성)
```

---

## 📌 핵심 정리

```
클러스터링 핵심:

클러스터에서 공유:
  Exchange, Queue 정의, Binding, vHost, 권한 → 모든 노드 공유
  Classic Queue 메시지 데이터 → 마스터 노드에만

Quorum Queue (권장):
  Raft 합의 기반 복제
  과반수 노드 확인 후 커밋
  노드 장애 시 자동 Leader 선출 (수 초)
  Split-Brain 자동 방지

클러스터 구성 기준:
  최소 3노드 (홀수)
  3노드: 1개 장애 허용
  5노드: 2개 동시 장애 허용

네트워크 파티션:
  Quorum Queue: 과반수 없으면 쓰기 거절 (데이터 안전 우선)
  복구 후 자동 동기화

Erlang Cookie:
  모든 노드 동일 cookie 필수
  보안 관리 중요 (유출 시 악의적 노드 참여 가능)
```

---

## 🤔 생각해볼 문제

**Q1.** 5노드 Quorum Queue 클러스터에서 3노드가 동시에 장애났다. 어떤 일이 발생하고, 데이터는 안전한가?

<details>
<summary>해설 보기</summary>

5노드 클러스터의 과반수 = 3노드.

3노드 장애 → 남은 2노드 = 2/5 (과반수 미달)

발생하는 일:
- Quorum Queue: 쓰기(발행) **거절**. 과반수 없으면 아무 노드도 Leader 역할 불가
- 읽기(소비): 일부 제한 (Leader가 없으므로)
- 서비스 중단 (가용성 포기)

데이터 안전성:
- 이미 **커밋된 메시지** (장애 전 과반수 3개에 복제 완료된 것): 안전
- 남은 2개 노드 중 하나에라도 있다면 복구 후 복원 가능
- 커밋 안 된 메시지: 손실 가능 (Raft의 일관성 보장)

복구:
- 3개 노드 복구 후 자동으로 클러스터 재결합
- 리더 선출 재개
- 서비스 정상화

결론: **데이터는 커밋된 것까지 안전, 서비스는 일시 중단**. Raft의 "안전성 > 가용성" 선택.

</details>

---

**Q2.** 클러스터에서 Consumer가 Node 2에 연결되어 있는데, Queue의 Quorum Leader가 Node 1이다. 메시지 처리 경로는 어떻게 되는가?

<details>
<summary>해설 보기</summary>

Quorum Queue에서 모든 읽기/쓰기는 **Leader 노드**를 통해서만 처리됩니다.

**메시지 처리 경로:**
1. Consumer → Node 2 (연결)
2. Node 2: "이 Queue의 Leader는 Node 1이다"
3. Node 2 → Node 1 (내부 클러스터 통신)으로 메시지 요청 프록시
4. Node 1 (Leader): 메시지 꺼냄
5. Node 1 → Node 2 → Consumer 전달

**성능 영향:**
- 클라이언트가 Non-Leader 노드에 연결된 경우 추가 홉(hop) 발생
- 지연 미세하게 증가 (같은 데이터센터 내 1ms 미만)

**최적화:**
- `consumer_prefetch`와 `connection_balance` 설정으로 Leader와 동일한 노드에 Consumer가 연결되도록 유도
- `x-queue-leader-locator` 설정으로 Queue Leader 위치를 특정 노드에 고정

</details>

---

**Q3.** 개발팀이 "RabbitMQ 클러스터를 구성했으니 이제 메시지는 절대 유실되지 않는다"고 주장한다. 이 주장의 어떤 부분이 틀렸고, 완전한 메시지 보장을 위해 추가로 필요한 것은?

<details>
<summary>해설 보기</summary>

클러스터만으로는 메시지 유실을 완전히 막지 못합니다.

**클러스터가 해결하는 것:**
- 노드 장애 시 데이터 보존 (Quorum Queue 사용 시)
- 서비스 연속성 (노드 장애 후 자동 복구)

**클러스터가 해결하지 않는 것:**

1. **Publisher → Broker 전송 중 유실**
   - Publisher Confirm 없으면 브로커가 받았는지 확인 불가
   - 해결: `confirmSelect()` + Confirm Callback

2. **Consumer 처리 중 유실**
   - `autoAck=true`이면 Consumer가 받는 순간 삭제
   - Consumer가 처리 전에 죽으면 메시지 소실
   - 해결: `autoAck=false` + 수동 `basicAck`

3. **메시지 미영속 유실**
   - `DeliveryMode=TRANSIENT`이면 재시작 시 소실
   - 해결: `DeliveryMode=PERSISTENT` + `durable=true` Queue

4. **DLQ 없는 처리 실패**
   - 처리 실패 + `requeue=false` → 메시지 폐기
   - 해결: Dead Letter Exchange + Dead Letter Queue

완전한 메시지 보장 = 클러스터 + Publisher Confirm + 수동 Ack + Persistent 메시지 + DLX

</details>

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[이전: 가상 호스트(vHost) ⬅️](./05-vhost.md)** | **[다음: Chapter 2 — Direct Exchange ➡️](../exchange-routing/01-direct-exchange.md)**

</div>
