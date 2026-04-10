# 모니터링 핵심 지표 — 문제를 사전에 감지하는 대시보드 설계

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Queue Depth가 증가한다는 것은 어떤 의미인가?
- Publish Rate vs Deliver Rate 비교로 무엇을 알 수 있는가?
- Consumer Utilisation이 낮다는 것은 어떤 문제를 의미하는가?
- Prometheus Exporter를 설치하고 핵심 알림을 어떻게 설정하는가?
- Management UI만으로 충분한가, 언제 외부 모니터링이 필요한가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

"RabbitMQ가 갑자기 느려졌다"는 장애 보고가 들어온다. 원인 파악에 1시간이 걸린다. 사전에 모니터링했다면 30분 전에 Queue Depth 증가를 감지하고 대응했을 것이다. 올바른 지표를 알고, 알림 임계값을 설정하고, 대시보드를 구성해야 장애를 예방할 수 있다.

---

## 😱 흔한 실수 (Before)

```
실수 1: 모니터링 없이 운영

  이상 감지: 고객 신고 → 개발자 확인 → 로그 분석 → 원인 파악
  소요 시간: 1~2시간
  
  올바른 순서: 지표 임계값 알림 → 개발자 즉시 인지 → 5분 내 대응

실수 2: 연결 수만 모니터링 (중요 지표 누락)

  "연결 수 OK, Consumers 수 OK → 문제없다"
  실제: Queue Depth 50만 건 → Consumer가 따라가지 못함
  → Queue Depth, Unacked 메시지 수를 반드시 모니터링

실수 3: 너무 낮은 알림 임계값 → 알림 피로

  Queue Depth > 1이면 알림 발송
  → 매 분 수백 건 알림
  → 알림 무시 → 실제 장애 알림도 무시
  → 의미 있는 임계값 설정 중요
```

---

## ✨ 올바른 접근 (After)

```
핵심 지표 계층:

1. 즉각 알림 (P1 - 즉시 대응):
   Queue Depth > 10,000 이상 (또는 서비스별 임계값)
   Consumer 수 = 0 (Active Consumer 없음)
   Memory Alarm 발동
   DLQ 메시지 발생

2. 경고 알림 (P2 - 30분 내 대응):
   Queue Depth 증가 속도 이상 (전 5분 대비 10배)
   Consumer Utilisation < 50% (처리량 낭비)
   Unacked 메시지 급증

3. 정보성 (P3 - 일일 검토):
   Connection/Channel 수 증가 추세
   처리량 일별 비교
```

---

## 🔬 내부 동작 원리

### 1. 핵심 지표와 의미

```
=== Queue Depth (메시지 적체) ===

지표: rabbitmq_queue_messages (total = Ready + Unacked)

Queue Depth = 0:
  Consumer가 발행 속도를 따라잡고 있음 (정상)

Queue Depth 증가 (Ready 증가):
  Consumer 처리 속도 < 발행 속도
  원인 후보:
    1. Consumer 수 부족
    2. Consumer 처리 로직 느림 (DB 병목 등)
    3. Consumer 다운

Queue Depth 증가 (Unacked 증가):
  Consumer가 메시지를 받았지만 처리 완료 못 함
  원인 후보:
    1. Consumer 처리 중 블로킹
    2. Prefetch 너무 크고 처리 느림
    3. Consumer 크래시 (Ack 못 보냄)

=== Publish Rate vs Deliver Rate ===

  Publish Rate > Deliver Rate:
    메시지가 소비보다 빠르게 쌓임
    → Queue Depth 증가 → 장기 지속 시 알림

  Publish Rate < Deliver Rate:
    Consumer가 Queue를 소진 중 (Queue Depth 감소)
    → 정상 복구 중

  Publish Rate ≈ Deliver Rate:
    균형 상태 (이상적)

=== Consumer Utilisation ===

  Utilisation = 처리시간 / (처리시간 + 대기시간)

  100%: Consumer가 항상 처리 중 (과부하 또는 Prefetch 최적)
  70~90%: 적정 (여유 있음)
  < 50%: Consumer가 많이 놀고 있음 → Prefetch 증가 또는 Consumer 감소

=== Connection/Channel 수 ===

  Connection 수 급증:
    서비스 재시작 폭풍 (Connection Storm)
    → 수백~수천 연결 동시 생성 → 브로커 과부하

  Channel 수 증가:
    Channel 누수 (close() 없음)
    → 채널 한도 초과 → 새 채널 생성 불가

=== DLQ 메시지 수 ===

  DLQ 메시지 > 0: 처리 실패 메시지 발생
  DLQ 메시지 증가 속도: 실패 빈도
  
  즉각 알림 대상:
    - DLQ에 처음 메시지가 쌓이는 경우
    - DLQ 메시지가 임계값 초과 (예: 100건)
```

### 2. Prometheus Exporter 설정

```
=== rabbitmq_prometheus Plugin ===

활성화:
  rabbitmq-plugins enable rabbitmq_prometheus

설정 (rabbitmq.conf):
  prometheus.path = /metrics
  prometheus.tcp.port = 15692

Prometheus 수집 설정:
  scrape_configs:
    - job_name: 'rabbitmq'
      static_configs:
        - targets: ['rabbitmq:15692']
      scrape_interval: 15s

=== 핵심 메트릭 ===

Queue 지표:
  rabbitmq_queue_messages                      # 총 메시지 수 (Ready + Unacked)
  rabbitmq_queue_messages_ready                # Ready 메시지 수
  rabbitmq_queue_messages_unacked             # Unacked 메시지 수
  rabbitmq_queue_messages_published_total     # 누적 발행 수
  rabbitmq_queue_messages_delivered_ack_total # 누적 처리 완료 수

Consumer 지표:
  rabbitmq_queue_consumers                    # Consumer 수
  rabbitmq_queue_consumer_utilisation        # Consumer Utilisation (0~1)

Connection 지표:
  rabbitmq_connections                        # Connection 수
  rabbitmq_channels                           # Channel 수

시스템 지표:
  rabbitmq_resident_memory_limit_bytes        # 메모리 임계값 (절대값)
  rabbitmq_process_resident_memory_bytes      # 현재 메모리 사용량
  rabbitmq_disk_space_available_bytes         # 디스크 여유 공간

=== Grafana 대시보드 패널 ===

패널 1: Queue Depth (주요 Queue별)
  쿼리: rabbitmq_queue_messages{queue=~"order.queue|payment.queue"}
  경보: > 10000

패널 2: Publish vs Deliver Rate (차이)
  쿼리: rate(rabbitmq_queue_messages_published_total[1m])
        - rate(rabbitmq_queue_messages_delivered_ack_total[1m])
  해석: 양수 = 쌓임, 음수 = 소진

패널 3: Consumer Utilisation
  쿼리: rabbitmq_queue_consumer_utilisation
  범위: 0~1 (100%까지)
  경보: < 0.5 (50% 미만)

패널 4: DLQ 메시지 수
  쿼리: rabbitmq_queue_messages{queue=~".*dlq.*"}
  경보: > 0

패널 5: Memory Usage
  쿼리: rabbitmq_process_resident_memory_bytes / rabbitmq_resident_memory_limit_bytes
  경보: > 0.7 (70% 초과)
```

### 3. 알림 규칙 (Prometheus Alert Rules)

```yaml
=== 핵심 알림 규칙 ===

groups:
  - name: rabbitmq.rules
    rules:
      # P1: Consumer 없는 Queue
      - alert: RabbitMQNoConsumers
        expr: rabbitmq_queue_consumers{queue!~".*dlq.*"} == 0
        for: 5m
        labels: { severity: critical }
        annotations:
          summary: "Consumer 없음: {{ $labels.queue }}"

      # P1: Queue Depth 임계값 초과
      - alert: RabbitMQQueueDepthHigh
        expr: rabbitmq_queue_messages > 50000
        for: 2m
        labels: { severity: critical }
        annotations:
          summary: "Queue 적체: {{ $labels.queue }} = {{ $value }}"

      # P1: DLQ 메시지 발생
      - alert: RabbitMQDLQMessages
        expr: rabbitmq_queue_messages{queue=~".*dlq.*"} > 0
        for: 1m
        labels: { severity: warning }
        annotations:
          summary: "DLQ 메시지 발생: {{ $labels.queue }} = {{ $value }}"

      # P1: Memory Alarm
      - alert: RabbitMQMemoryAlarm
        expr: rabbitmq_alarms_memory_used_watermark > 0
        for: 0m
        labels: { severity: critical }
        annotations:
          summary: "RabbitMQ Memory Alarm - Flow Control 발동"

      # P2: Queue Depth 급증 (5분 전 대비 10배)
      - alert: RabbitMQQueueDepthSurge
        expr: rabbitmq_queue_messages / rabbitmq_queue_messages offset 5m > 10
        for: 2m
        labels: { severity: warning }
        annotations:
          summary: "Queue Depth 급증: {{ $labels.queue }}"

      # P2: Consumer Utilisation 낮음
      - alert: RabbitMQLowConsumerUtilisation
        expr: rabbitmq_queue_consumer_utilisation < 0.5
              and rabbitmq_queue_consumers > 0
        for: 10m
        labels: { severity: warning }
        annotations:
          summary: "Consumer Utilisation 낮음: {{ $labels.queue }}"

      # P3: Connection 수 급증
      - alert: RabbitMQConnectionStorm
        expr: increase(rabbitmq_connections[5m]) > 100
        for: 1m
        labels: { severity: warning }
        annotations:
          summary: "Connection 수 급증 (5분 내 100+ 연결)"
```

---

## 💻 실전 실험

### 실험 1: Management UI 핵심 화면 순서

```
1. Overview 탭:
   - Queued messages: Ready + Unacked 총합
   - Message rates: Publish/Deliver/Ack 속도
   - Nodes: 노드별 메모리/디스크 사용량
   
2. Queues 탭:
   - Messages: Ready/Unacked/Total
   - Consumer utilisation: 각 Queue별
   - 이름 클릭 → 상세 (Bindings, Consumer 목록, 메트릭)

3. Connections 탭:
   - 서비스별 Connection 수
   - 채널 수 확인

4. Admin 탭:
   - Limits: Connection/Channel 제한
   - Policies: 현재 적용된 Policy
```

### 실험 2: Queue Depth 임계값 알림 테스트

```bash
# Consumer 없이 메시지 발행 → Queue Depth 증가
rabbitmqadmin declare queue name=alert.test durable=true
for i in $(seq 1 100); do
  rabbitmqadmin publish exchange="" routing_key=alert.test payload="msg$i"
done

# Prometheus 지표 확인
curl http://rabbitmq:15692/metrics | grep alert_test
# rabbitmq_queue_messages{queue="alert.test",...} 100

# Grafana 알림: 임계값 50 초과 → alert.test 알림 발동 확인
```

---

## 📊 Kafka vs RabbitMQ 비교

```
=== 모니터링 지표 비교 ===

RabbitMQ 핵심 지표:
  Queue Depth (메시지 적체)
  Consumer Utilisation (소비 효율)
  Publish vs Deliver Rate
  DLQ 메시지 수
  Memory/Disk alarm

Kafka 핵심 지표:
  Consumer Lag (오프셋 차이 = 처리 대기 메시지 수)
  Producer 처리량 (bytes/sec)
  Partition 크기
  ISR (In-Sync Replicas) 상태

유사 개념:
  RabbitMQ Queue Depth ≈ Kafka Consumer Lag
  RabbitMQ Consumer Utilisation ≈ Kafka Consumer Throughput
  RabbitMQ DLQ ≈ Kafka Dead Letter Topic

RabbitMQ 모니터링 도구:
  내장: Management UI (HTTP API)
  외부: rabbitmq_prometheus Plugin + Grafana

Kafka 모니터링 도구:
  내장: JMX, Kafka 자체 지표
  외부: Confluent Control Center, Grafana (JMX Exporter)
```

---

## ⚖️ 트레이드오프

```
Management UI vs Prometheus + Grafana:

Management UI:
  ✅ 기본 내장, 설치 불필요
  ✅ 실시간 Queue/Connection 상태
  ✅ 메시지 직접 조회 가능
  ❌ 알림 기능 없음
  ❌ 이력 데이터 저장 없음 (새로고침 시 사라짐)
  ❌ 여러 브로커 통합 불가

Prometheus + Grafana:
  ✅ 시계열 이력 저장
  ✅ 알림 규칙 설정
  ✅ 여러 브로커 통합 대시보드
  ✅ Kubernetes/Cloud 환경 통합
  ❌ 초기 설정 필요

권장:
  개발/소규모: Management UI로 충분
  프로덕션: Prometheus + Grafana 필수
  알림: PagerDuty/Slack 연동
```

---

## 📌 핵심 정리

```
모니터링 핵심 지표:

Queue Depth (P1):
  Ready > 임계값 → Consumer 처리 부족
  Unacked > 임계값 → Consumer 처리 중 문제

Consumer Utilisation:
  < 50% → Prefetch 증가 또는 Consumer 과잉
  = 100% → Consumer Scale Out 필요

Publish vs Deliver Rate:
  차이 지속 → Queue 적체 진행 중

DLQ 메시지:
  > 0 → 처리 실패 발생, 원인 분석 필요

Memory Alarm:
  발동 → Flow Control 즉시 해결 필요

도구:
  내장: Management UI (개발)
  외부: Prometheus + Grafana (프로덕션)
  알림: 즉각(P1) / 경고(P2) / 정보(P3) 계층화
```

---

## 🤔 생각해볼 문제

**Q1.** Queue Depth는 0인데 Consumer Utilisation도 0%다. 어떤 상황인가?

<details>
<summary>해설 보기</summary>

**Queue에 메시지가 없어서 Consumer가 놀고 있는 정상 상태입니다.**

Queue Depth = 0 + Consumer Utilisation = 0:
- 발행 속도가 낮아 Queue가 비어있음
- Consumer가 처리할 메시지가 없어 대기 중
- **정상 상태** (문제 아님)

걱정해야 하는 경우:
- Queue Depth 증가 + Consumer Utilisation 100%: 과부하
- Queue Depth 증가 + Consumer Utilisation 0%: Consumer 없음 (장애)
- Queue Depth 0 + Consumer Utilisation 100%: Consumer가 처리하자마자 바로 소비 (정상이지만 조금이라도 더 오면 바로 적체)

</details>

---

**Q2.** DLQ 메시지 수가 0에서 갑자기 1000으로 증가했다. 어떤 순서로 원인을 파악해야 하는가?

<details>
<summary>해설 보기</summary>

**진단 체크리스트 (순서대로):**

1. **DLQ 메시지 확인** (Management UI → DLQ → Get Messages)
   - x-death 헤더의 `reason`: rejected/expired/maxlen
   - `queue`: 어느 Queue에서 왔는가
   - 메시지 내용: 어떤 데이터인가

2. **원인 Queue의 Consumer 로그 확인**
   - 어떤 예외로 basicNack(requeue=false)가 호출됐는가
   - 특정 시점(배포, DB 점검 등)과 일치하는가

3. **reason별 원인 추적**
   - `rejected`: Consumer 예외 → 예외 내용 확인
   - `expired`: TTL 만료 → Consumer가 너무 느림 (처리 시간 vs TTL 확인)
   - `maxlen`: Queue 가득 참 → 발행 속도 급증 또는 Consumer 중단

4. **패턴 파악**
   - 특정 messageId/orderId 패턴이 있는가 (특정 데이터 문제)
   - 전체가 실패인가, 일부인가 (간헐적 오류)

5. **재처리 결정**
   - 원인 수정 후 DLQ Consumer로 재발행
   - 또는 수동 재처리

</details>

---

**Q3.** "Publish Rate = 1000/sec, Deliver Rate = 1000/sec, Queue Depth = 50000"인 상태는 정상인가 비정상인가?

<details>
<summary>해설 보기</summary>

**비정상 상태지만 회복 중입니다.**

해석:
- Publish Rate = Deliver Rate: 현재 유입량과 처리량이 균형
- Queue Depth = 50000: 과거에 적체된 메시지가 아직 남아있음

타임라인:
1. (과거) 발행 속도가 처리 속도보다 빨랐음 → 50000건 적체
2. (현재) 균형 상태 → Queue Depth 유지 (더 이상 쌓이지 않음)
3. (미래) Deliver Rate > Publish Rate가 되면 → 점진적으로 소진

조치:
- **즉각 조치 필요**: Queue Depth가 계속 높게 유지되면 Consumer 일시 증가하여 소진
- **패턴 파악**: 언제 적체됐는가 (피크 시간, 특정 이벤트)
- **용량 계획**: 다음 피크 전에 Consumer 처리량 증가

Publish Rate와 Deliver Rate만 보면 "균형"처럼 보이지만 Queue Depth를 함께 봐야 실제 상태를 파악할 수 있습니다. 세 지표를 함께 모니터링해야 합니다.

</details>

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[이전: 메모리 관리 ⬅️](./02-memory-management.md)** | **[다음: 운영 중 발생하는 문제 패턴 ➡️](./04-operational-issues.md)**

</div>
