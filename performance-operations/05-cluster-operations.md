# 클러스터 운영 — 노드 관리, 장애 복구, 무중단 배포

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 클러스터에 노드를 추가/제거하는 절차는 무엇인가?
- Queue가 특정 노드에 쏠리지 않도록 균등 분산하는 방법은?
- 노드 장애 시 Quorum Queue는 어떻게 자동으로 복구하는가?
- Rolling Upgrade로 서비스 중단 없이 RabbitMQ를 업그레이드하는 절차는?
- Classic Queue와 Quorum Queue의 장애 대응 방식 차이는?

---

## 🔍 왜 이 개념이 실무에서 중요한가

단일 노드 RabbitMQ는 SPOF(단일 장애 지점)다. 노드가 다운되면 모든 메시지 처리가 중단된다. 3노드 클러스터 + Quorum Queue를 구성하면 노드 1개가 다운되어도 자동으로 다른 노드에서 처리가 계속된다. 하지만 클러스터 운영은 노드 추가/제거, Queue 분산, 업그레이드 절차를 알아야 한다. 절차를 모르면 노드 제거 시 메시지가 유실되거나, 업그레이드 중 클러스터가 분할될 수 있다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: 노드 강제 종료 후 클러스터 제거

  # 노드 B를 kill -9로 강제 종료
  ssh node-b "kill -9 $(pidof beam.smp)"
  
  결과:
    클러스터는 노드 B가 "일시 다운"으로 간주
    오래 지나도 B가 복구 안 되면 → 클러스터 분열 위험
    Quorum Queue: B가 복귀할 것을 기다림 → 특정 Queue에서 쓰기 불가 가능

  올바른 절차:
    1. rabbitmqctl stop_app (노드 B 앱 정지)
    2. rabbitmqctl reset (클러스터에서 제거)
    3. 클러스터 상태 확인

실수 2: 모든 Queue를 하나의 노드에 생성

  Queue 100개 → 모두 node-a에서 생성
  node-a: Queue Master 100개 (모든 I/O)
  node-b, node-c: 비어있음 (트래픽 없음)

  결과:
    node-a 과부하 → 처리량 저하
    node-a 장애 → Classic Queue 전체 사용 불가

실수 3: Rolling Upgrade에서 메이저 버전 건너뜀

  현재: 3.9.x
  목표: 3.12.x (2단계 업)
  실수: 직접 3.9 → 3.12 업그레이드

  결과:
    RabbitMQ는 메이저 버전 간 클러스터 혼용 미지원
    업그레이드 중 3.9 노드와 3.12 노드가 클러스터 이탈
    → 클러스터 분열 위험

  올바른 절차: 3.9 → 3.10 → 3.11 → 3.12 (단계적)
```

---

## ✨ 올바른 접근 (After — 체계적 클러스터 관리)

```
클러스터 운영 핵심 원칙:

1. 3노드 이상 (홀수 권장):
   Quorum Queue Raft 합의: 과반수 필요
   3노드: 1노드 다운 허용 (2/3 > 1/2)
   5노드: 2노드 다운 허용 (3/5 > 1/2)

2. Quorum Queue 우선:
   Classic Queue: 노드 장애 시 Master 노드에만 Queue 존재
   Quorum Queue: 모든 노드에 복제 → 장애 자동 복구

3. 균등 분산:
   Queue가 특정 노드에 쏠리지 않도록
   Policy로 자동 분산 설정

4. Rolling Upgrade:
   한 번에 노드 하나씩 업그레이드
   마이너 버전: 한 단계씩 가능
   메이저 버전: 하나씩 순차 업그레이드 필수
```

---

## 🔬 내부 동작 원리

### 1. 노드 추가와 제거 절차

```
=== 노드 추가 ===

신규 노드(node-d) 추가 절차:

1. node-d에 RabbitMQ 설치 및 시작
   systemctl start rabbitmq-server

2. node-d에서 기존 클러스터에 참가
   rabbitmqctl stop_app
   rabbitmqctl reset           # 초기화 (기존 데이터 삭제)
   rabbitmqctl join_cluster rabbit@node-a  # node-a의 클러스터에 참가
   rabbitmqctl start_app

3. 클러스터 상태 확인 (어느 노드에서든)
   rabbitmqctl cluster_status
   # Running Nodes: rabbit@node-a, rabbit@node-b, rabbit@node-c, rabbit@node-d

4. (선택) Quorum Queue의 새 노드 포함:
   # Quorum Queue는 자동으로 새 노드에 복제하지 않음
   # 필요 시 수동으로 멤버 추가
   rabbitmqctl add_quorum_queue_member order.queue rabbit@node-d

=== 노드 안전 제거 ===

node-d 제거 절차 (안전하게):

1. 트래픽 제거 (로드밸런서에서 node-d 제외)

2. Queue를 다른 노드로 이전 (Queue가 있는 경우):
   # Quorum Queue: 자동으로 다른 노드에서 처리
   # Classic Queue: 수동 이전 필요

3. node-d 앱 정지
   rabbitmqctl stop_app   # RabbitMQ 앱만 정지 (Erlang VM 유지)

4. 클러스터에서 제거
   # node-d에서:
   rabbitmqctl reset
   
   # 또는 다른 노드에서 강제 제거 (node-d 통신 불가 시):
   rabbitmqctl forget_cluster_node rabbit@node-d

5. 클러스터 상태 확인
   rabbitmqctl cluster_status

=== forget_cluster_node vs reset의 차이 ===

forget_cluster_node (다른 노드에서):
  "저 노드는 이제 클러스터에 없다고 간주"
  해당 노드의 Classic Queue Master → 다른 노드로 이전 불가
  해당 노드가 Quorum Queue 멤버라면 → 과반수 재계산

reset (해당 노드에서):
  "나는 클러스터를 탈퇴한다"
  더 안전한 방법
  해당 노드의 데이터 삭제 후 독립 노드로

권장: 가능하면 항상 해당 노드에서 reset으로 탈퇴
      통신 불가 시에만 forget_cluster_node
```

### 2. Queue 균등 분산 전략

```
=== Classic Queue 분산 ===

Classic Queue의 문제:
  Queue 생성 시 해당 노드가 "Master" (모든 I/O가 여기서 처리)
  Binding, 발행, 소비 모두 Master 노드를 경유
  → 특정 노드에 Queue가 많으면 과부하

분산 방법 1: 라운드로빈 Queue 생성
  # 각 서비스를 다른 노드에 연결하여 Queue 생성
  spring:
    rabbitmq:
      addresses: "node-a:5672,node-b:5672,node-c:5672"
  # 연결 노드에서 Queue 생성 → 자동으로 분산

분산 방법 2: x-queue-master-locator Policy
  rabbitmqctl set_policy QueueDistribution ".*" \
    '{"queue-master-locator":"min-masters"}' --apply-to queues
  
  min-masters: 현재 Master Queue 수가 가장 적은 노드에 생성
  → 자동 균등 분산

현재 분산 상태 확인:
  rabbitmqctl list_queues name node
  # order.queue     rabbit@node-a
  # payment.queue   rabbit@node-b
  # inventory.queue rabbit@node-c
  # ...균등 분산 확인

=== Quorum Queue 분산 ===

Quorum Queue는 기본적으로 모든 노드에 복제:
  3노드 클러스터 → 각 Quorum Queue: 3노드에 복제
  Leader: 한 노드가 담당 (발행/소비 처리)
  Follower: 나머지 노드 (복제만)

Leader 분산 확인:
  rabbitmqctl list_queues name leader members
  # order.queue   rabbit@node-a  [rabbit@node-a, rabbit@node-b, rabbit@node-c]
  # payment.queue rabbit@node-b  [...]
  # 여러 Queue의 Leader가 다른 노드에 분산되어야 함

Leader 재분산 (한 노드에 쏠린 경우):
  rabbitmqctl rebalance_queue_leaders

Quorum Queue 멤버 수 제한 (큰 클러스터):
  5노드 클러스터에서 Quorum Queue default-initial-cluster-size:
  # rabbitmq.conf:
  quorum_queue.initial_cluster_size = 3  # 5노드 중 3개에만 복제
  → 불필요한 복제 오버헤드 감소
```

### 3. Quorum Queue 장애 복구 원리

```
=== Raft 기반 장애 복구 ===

3노드 클러스터 (node-a: Leader, node-b/c: Follower):

정상 동작:
  Producer → node-a (Leader) → 메시지 수신
  node-a: WAL에 기록 후 node-b, node-c에 복제
  과반수(2/3) 확인 → Producer에 Ack

node-a 장애 (Leader 다운):
  1. node-b, node-c: Heartbeat 없음 감지 (election timeout)
  2. Election 시작: node-b → "투표해줘"
  3. node-c: "투표함" → 과반수(2/3 = node-b + node-c) 달성
  4. node-b: 새 Leader 선출 완료
  5. Producer: node-b (새 Leader)에 연결 → 메시지 처리 계속

복구 시간:
  Election Timeout: 기본 5초 (설정 가능)
  Leader 선출 및 안정화: 수 초
  총 다운타임: 보통 5~15초 (자동 복구)

=== Split-Brain 방지 ===

2/2 분할 (4노드 클러스터, 2개씩 분리):
  그룹 A (node-a, node-b): 자신들이 과반수라 생각 → Leader 선출
  그룹 B (node-c, node-d): 자신들이 과반수라 생각 → 또 Leader 선출
  → 두 Leader가 동시에 쓰기 → 데이터 불일치

RabbitMQ Quorum Queue의 해결:
  과반수(> 50%) 달성 못 하면 쓰기 거부
  4노드 클러스터에서 2개 분리: 어느 쪽도 과반수 미달 → 쓰기 불가
  → 데이터 안전 (가용성 포기, 일관성 보장)

이것이 홀수 노드를 권장하는 이유:
  3노드: 분할 시 2+1 → 2가 과반수 → 쓰기 가능 (부분 가용성)
  4노드: 분할 시 2+2 → 둘 다 과반수 미달 → 전체 쓰기 불가
  5노드: 분할 시 3+2 → 3이 과반수 → 부분 가용성

=== 장애 노드 복구 ===

node-a 복구 후 클러스터 재참가:

1. node-a 시작
   systemctl start rabbitmq-server

2. node-a가 자동으로 클러스터 재참가 (Mnesia 클러스터 정보 보존)
   클러스터 상태 확인:
   rabbitmqctl cluster_status

3. Quorum Queue: node-a가 Follower로 복구
   node-b (현재 Leader): node-a에 누락된 WAL 항목 동기화
   동기화 완료 → node-a 정상 Follower

4. Leader 재분산 (선택):
   rabbitmqctl rebalance_queue_leaders
   → node-a가 일부 Queue의 Leader를 다시 맡음

=== 노드 영구 제거 후 처리 ===

node-a가 영구 제거되어 Quorum Queue 멤버 부족:

3노드 중 node-a 영구 제거:
  2노드 남음 → Quorum Queue 멤버도 2개
  과반수: 2 > 2/2 = 1.5 → 둘 다 살아있으면 OK
  하지만 하나 더 다운되면 → 1/2 → 과반수 미달 → 쓰기 불가

해결:
  새 노드 추가 후 Quorum Queue 멤버 확장:
  rabbitmqctl add_quorum_queue_member order.queue rabbit@node-new
  → 다시 3멤버 체계
```

### 4. Rolling Upgrade 절차

```
=== 마이너 버전 Rolling Upgrade (예: 3.12.4 → 3.12.6) ===

전제: 클러스터 3노드, 모두 3.12.4

1. 사전 확인
   rabbitmqctl cluster_status  # 모든 노드 running 확인
   rabbitmqctl list_queues     # Queue 상태 정상 확인

2. node-c 업그레이드 (트래픽 먼저 제거)
   # 로드밸런서에서 node-c 제외
   # node-c의 기존 Consumer 연결이 다른 노드로 이동 (자동 페일오버)
   
   # node-c에서:
   rabbitmqctl stop_app
   # 패키지 업그레이드
   apt-get install rabbitmq-server=3.12.6
   # 또는 Docker: 이미지 업데이트
   
   rabbitmqctl start_app
   rabbitmqctl cluster_status  # node-c 재합류 확인

3. node-b 업그레이드 (같은 절차)
4. node-a 업그레이드 (같은 절차)

=== 메이저 버전 Rolling Upgrade (예: 3.11 → 3.12) ===

중요: 메이저 버전 혼합 클러스터는 제한된 시간만 지원
      (RabbitMQ는 한 단계 차이 혼합 클러스터만 지원)

1. 사전 준비
   # 기능 플래그 확인 (3.11에서 사용 가능한 기능 플래그)
   rabbitmqctl list_feature_flags
   # 3.12에서 필수인 기능 플래그를 3.11에서 미리 활성화
   rabbitmqctl enable_feature_flag all

2. 노드별 순차 업그레이드 (마이너와 동일 절차)
   node-c → node-b → node-a 순서

3. 모든 노드 3.12로 업그레이드 완료 후
   # 3.12 전용 기능 플래그 활성화
   rabbitmqctl enable_feature_flag all

4. 검증
   rabbitmqctl cluster_status
   rabbitmqctl list_queues

=== Kubernetes 환경 Rolling Upgrade ===

StatefulSet Rolling Update:
  rabbitmq-statefulset.yaml:
    spec:
      updateStrategy:
        type: RollingUpdate
        rollingUpdate:
          partition: 0  # 0이면 모든 Pod 순차 업데이트
  
  kubectl set image statefulset/rabbitmq rabbitmq=rabbitmq:3.12.6
  # Pod가 하나씩 (N-1, N-2, ... 0 순서) 재시작
  
  진행 상황 모니터링:
  kubectl rollout status statefulset/rabbitmq

주의:
  Kubernetes Rolling Update는 Pod 종료 후 즉시 새 Pod 시작
  RabbitMQ 앱이 올라올 때까지 대기 (readiness probe 활용)
  readinessProbe:
    exec:
      command: ["rabbitmq-diagnostics", "ping"]
    initialDelaySeconds: 20
    periodSeconds: 10
```

---

## 💻 실전 실험

### 실험 1: 클러스터 상태 진단

```bash
# 클러스터 전체 상태
rabbitmqctl cluster_status
# Cluster name: rabbit@node-a
# Running Nodes: rabbit@node-a, rabbit@node-b, rabbit@node-c
# Disk Nodes: [rabbit@node-a, rabbit@node-b, rabbit@node-c]
# ...

# Queue별 노드 분산 확인
rabbitmqctl list_queues name node leader members --formatter pretty_table

# Quorum Queue 상태 확인
rabbitmqctl list_queues name type leader members | grep quorum

# 각 노드의 Queue 수 (분산 확인)
rabbitmqctl list_queues name node | awk '{print $2}' | sort | uniq -c
# 3 rabbit@node-a  ← node-a에 3개
# 3 rabbit@node-b
# 4 rabbit@node-c  ← 약간 불균등 → rebalance 고려
```

### 실험 2: 노드 장애 시뮬레이션 (Docker)

```bash
# 3노드 클러스터 구성 (docker-compose)
docker-compose up -d rabbitmq-node-a rabbitmq-node-b rabbitmq-node-c

# Quorum Queue 생성
rabbitmqadmin --host rabbitmq-node-a declare queue \
  name=resilient.queue durable=true \
  arguments='{"x-queue-type":"quorum"}'

# 메시지 발행
rabbitmqadmin --host rabbitmq-node-a publish exchange="" \
  routing_key=resilient.queue payload="test message"

# node-a 강제 종료 (Leader 장애 시뮬레이션)
docker stop rabbitmq-node-a

# 잠시 후 (5~15초) 자동 Leader 선출 완료
sleep 10

# node-b로 접속 → 메시지 여전히 있는지 확인
rabbitmqadmin --host rabbitmq-node-b get queue=resilient.queue ackmode=ack_requeue_true
# 메시지 존재 → 자동 복구 성공

# node-a 복구
docker start rabbitmq-node-a

# 클러스터 재합류 확인
rabbitmqctl --node rabbit@rabbitmq-node-a cluster_status
```

### 실험 3: Rolling Upgrade 진행 상황 모니터링

```bash
# 업그레이드 진행 중 클러스터 모니터링 스크립트
while true; do
  echo "=== $(date) ==="
  rabbitmqctl cluster_status | grep "Running Nodes"
  rabbitmqctl list_queues name messages | awk '{sum += $2} END {print "Total Messages:", sum}'
  echo ""
  sleep 10
done
```

---

## 📊 Kafka vs RabbitMQ 비교

```
=== 클러스터 운영 비교 ===

노드 추가:
  RabbitMQ: rabbitmqctl join_cluster → 즉시 클러스터 합류
            Quorum Queue: 수동으로 새 멤버 추가
  Kafka:    Partition 재분배 필요 (kafka-reassign-partitions.sh)
            대용량 데이터 이동 시간 소요

노드 제거:
  RabbitMQ: Quorum Queue → 자동 재선출 (과반수 유지 필요)
  Kafka:    Partition을 다른 브로커로 이동 후 제거
            데이터 이동 시간 소요

장애 복구:
  RabbitMQ Quorum Queue: 자동 Leader 선출 (5~15초)
  Kafka:    Controller 선출 + ISR 재구성 (수 초)
  두 시스템 모두 자동 복구

Rolling Upgrade:
  RabbitMQ: 마이너 버전 롤링 가능
            메이저 버전: 한 단계씩 순차 필수
  Kafka:    브로커 하나씩 업그레이드 (Protocol Version 관리)
            메이저 버전: inter.broker.protocol.version 설정 필요

=== 운영 복잡성 ===

RabbitMQ 클러스터:
  Erlang Cookie 공유 필요 (같은 Cookie = 같은 클러스터)
  Mnesia DB 동기화
  상대적으로 작은 클러스터 (3~7노드 일반적)

Kafka 클러스터:
  Zookeeper 또는 KRaft (3.x+) 필요
  대규모 클러스터 가능 (수십~수백 브로커)
  Partition 단위 세밀한 분산 제어
```

---

## ⚖️ 트레이드오프

```
클러스터 크기 트레이드오프:

3노드:
  ✅ 1노드 장애 허용 (2/3 과반수)
  ✅ 관리 단순
  ❌ 높은 처리량에 한계

5노드:
  ✅ 2노드 장애 허용 (3/5 과반수)
  ✅ 더 높은 처리량
  ❌ 관리 복잡, 비용 증가

Quorum Queue vs Classic Queue:
  Quorum Queue:
    ✅ 자동 장애 복구 (Raft)
    ✅ 강한 일관성
    ❌ 처리량 약 30% 낮음 (합의 오버헤드)
    ❌ 메모리 더 사용 (각 노드가 복제본 보유)
  
  Classic Queue:
    ✅ 높은 처리량
    ❌ 노드 장애 시 해당 노드 Queue 접근 불가
    ❌ mirroring 설정 시 추가 오버헤드

Rolling Upgrade 위험 vs 중단 업그레이드:
  Rolling Upgrade:
    ✅ 서비스 중단 없음
    ❌ 구버전/신버전 혼합 기간 존재 (기능 호환성 주의)
  
  중단 업그레이드:
    ✅ 단순 (전체 중단 → 업그레이드 → 재시작)
    ❌ 서비스 다운타임 발생
```

---

## 📌 핵심 정리

```
클러스터 운영 핵심:

노드 추가:
  join_cluster → start_app → cluster_status 확인
  Quorum Queue 멤버 확장: add_quorum_queue_member

노드 제거:
  stop_app → reset (또는 forget_cluster_node)
  Classic Queue Master 먼저 이전 필요

Queue 분산:
  min-masters Policy로 자동 균등 분산
  rebalance_queue_leaders로 Leader 재분산

Quorum Queue 장애 복구:
  Election Timeout(5초) 후 자동 Leader 선출
  과반수 살아있으면 자동 복구 (인간 개입 없음)
  노드 복구 후 자동 WAL 동기화

Rolling Upgrade:
  마이너: 노드 하나씩 stop_app → 업그레이드 → start_app
  메이저: 기능 플래그 먼저 활성화 → 단계적 업그레이드
  Kubernetes: StatefulSet RollingUpdate + readinessProbe

홀수 노드 권장:
  3, 5, 7노드: 분할 시 과반수 판별 가능
  짝수 노드: 50/50 분할 시 전체 쓰기 불가 위험
```

---

## 🤔 생각해볼 문제

**Q1.** 3노드 클러스터에서 Quorum Queue가 있다. node-a(Leader)와 node-b가 동시에 다운됐다. 이 Queue는 어떻게 되는가?

<details>
<summary>해설 보기</summary>

**Queue가 쓰기 불가 상태가 됩니다.** 과반수를 충족하지 못하기 때문입니다.

3노드 Quorum Queue 상태:
- node-a: 다운 (Leader)
- node-b: 다운 (Follower)
- node-c: 살아있음 (Follower) → 1/3 = 과반수 미달

node-c 혼자서는 과반수(2/3 이상) 달성 불가 → 새 Leader 선출 불가 → 메시지 발행/소비 중단.

RabbitMQ가 이때 취하는 동작:
- 발행 시도 → 503 (Queue not available)
- 소비 시도 → 기존 연결된 Consumer는 새 메시지 못 받음

**복구 방법:**
1. node-a 또는 node-b 중 하나 복구 → 과반수(2/3) 달성 → 자동 복구
2. 두 노드 모두 영구 장애 시 → 강제 복구 필요:
   ```bash
   # node-c에서 강제 리더 선출 (데이터 손실 가능성 있음)
   rabbitmqctl force_boot
   ```
   이 명령은 과반수 없이 강제로 시작 → 일부 메시지 유실 가능성

**이것이 5노드 클러스터가 더 안전한 이유입니다.** 5노드에서 2노드 동시 장애 시 3/5 과반수 달성으로 자동 복구.

</details>

---

**Q2.** Rolling Upgrade 중 node-b가 3.12.6으로 업그레이드됐고 node-a, node-c는 아직 3.12.4다. 이 상태에서 발행/소비가 정상 동작하는가?

<details>
<summary>해설 보기</summary>

**마이너 버전 내에서는 정상 동작합니다.**

RabbitMQ는 같은 메이저 버전 내 마이너 버전 혼합 클러스터를 지원합니다. 3.12.4와 3.12.6은 같은 3.12 메이저 버전이므로 혼합 운영이 가능합니다.

단, 주의사항:
1. **메이저 버전 혼합은 제한됨**: 3.11.x와 3.12.x 혼합은 임시만 허용 (업그레이드 전환 기간)
2. **기능 플래그**: 신버전에만 있는 기능 플래그는 모든 노드가 신버전이 된 후 활성화해야 함
3. **롤백 어려움**: 일부 노드가 새 버전으로 데이터를 썼다면, 구버전으로 롤백 시 호환성 문제 가능

**업그레이드 중 데이터 가용성:**
- Quorum Queue: node-b가 업그레이드 중 잠깐 다운 → 과반수(node-a, node-c)가 처리 → 정상
- 업그레이드 완료 후 node-b 복귀 → WAL 동기화 → 정상 복구

**실무 팁**: 업그레이드 전 항상 `rabbitmqctl list_feature_flags`로 호환성 확인.

</details>

---

**Q3.** Classic Queue의 Master 노드를 강제로 변경하는 방법이 있는가? 왜 이것이 필요한가?

<details>
<summary>해설 보기</summary>

**Classic Queue의 Master 노드를 이동하는 방법:**

**방법 1: Queue 삭제 후 재생성 (다른 노드에 연결하여)**
- 메시지 소실 위험이 있으므로 Queue가 비어있을 때만 사용

**방법 2: Queue Mirroring Policy를 활용한 이전**
```bash
# 모든 노드에 미러링 설정 (기존 노드에도 미러 생성)
rabbitmqctl set_policy HAAll ".*" '{"ha-mode":"all"}' --apply-to queues

# 구버전 노드를 클러스터에서 제거 → Master가 다른 노드로 이동
rabbitmqctl stop_app  # 구버전 노드에서
```

**방법 3: Quorum Queue로 전환**
- Classic Queue를 삭제하고 Quorum Queue로 재생성
- Quorum Queue는 Leader가 자동으로 균등 분산됨

**왜 이것이 필요한가:**
1. **노드 과부하**: 특정 노드에 Master가 몰려 CPU/메모리 불균형
2. **하드웨어 교체**: 특정 노드를 제거하기 전에 Master 이전 필요
3. **지리적 최적화**: Consumer가 주로 있는 노드 근처에 Master 배치

**권장 해결책**: Classic Queue 대신 **Quorum Queue 사용** → Leader 자동 분산 + 장애 복구 + 재분산 명령(rebalance_queue_leaders) 지원으로 이런 문제 자체가 줄어듦.

</details>

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[이전: 운영 중 발생하는 문제 패턴 ⬅️](./04-operational-issues.md)** | **[다음: Chapter 6 — RabbitMQ vs Kafka ➡️](../rabbitmq-vs-kafka/01-architecture-comparison.md)**

</div>
