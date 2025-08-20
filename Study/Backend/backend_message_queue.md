# 📚 Message Queue

---

## 1. 주제/키워드
- 메시지 큐와 Kafka에 대해 알아보자!! (ง🔥‿🔥)ง

## 📑 목차

- [📚 Message Queue](#-message-queue)
  - [1. 주제/키워드](#1-주제키워드)
  - [📑 목차](#-목차)
  - [2. 핵심 요약 (Summary)](#2-핵심-요약-summary)
    - [Message Queue (메시지 큐)](#message-queue-메시지-큐)
    - [Message Queue의 장점](#message-queue의-장점)
    - [메시지 전송 보장 방식](#메시지-전송-보장-방식)
    - [Message Queue 사용 시 고민한 점](#message-queue-사용-시-고민한-점)
      - [순서 보장 (Ordering)](#순서-보장-ordering)
      - [중복 처리 (Duplicate Handling)](#중복-처리-duplicate-handling)
      - [장애 대응 (Failure Handling)](#장애-대응-failure-handling)
      - [백프레셔 (Backpressure)](#백프레셔-backpressure)
    - [Kafka (분산 메시지 스트리밍 플랫폼)](#kafka-분산-메시지-스트리밍-플랫폼)
    - [Kafka의 구조](#kafka의-구조)
    - [Kafka의 특징](#kafka의-특징)
    - [Kafka 메세지 처리 흐름](#kafka-메세지-처리-흐름)
      - [Offset Commit 방식](#offset-commit-방식)
    - [Idempotent Producer / Idempotent Consumer](#idempotent-producer--idempotent-consumer)
    - [Kafka 사용 시 고민한 점](#kafka-사용-시-고민한-점)
      - [메시지 순서](#메시지-순서)
      - [Exactly-once (정확히 한번만)](#exactly-once-정확히-한번만)
      - [백프레셔 (Lag 문제)](#백프레셔-lag-문제)
      - [장애 대응](#장애-대응)
      - [데이터 정합성 (Commit 시점)](#데이터-정합성-commit-시점)
    - [Message Queue + Batch](#message-queue--batch)
    - [Message Queue + Batch + Scheduler](#message-queue--batch--scheduler)
  - [3. 참고/추가 자료 (References)](#3-참고추가-자료-references)
  - [4. 내일/다음에 볼 것 (Next Steps)](#4-내일다음에-볼-것-next-steps)

---

## 2. 핵심 요약 (Summary)

### Message Queue (메시지 큐)
- **비동기 통신**을 위해 사용되는 **중간 저장소**
- 프로듀서(Producer)가 메시지를 넣고, 컨슈머(Consumer)가 메시지를 꺼내 처리
- 대표 기술: RabbitMQ, Kafka, SQS, Redis Streams
- 목적: **시스템 간 결합도 낮추기 + 트래픽 분산 처리**

---

### Message Queue의 장점
- **비동기 처리**
  - 요청 즉시 응답하고, **실제 작업은 큐에 넣어 나중에 처리**
  - 예: 이메일/SMS 발송, 로그 저장
- **버퍼 역할**
  - 트래픽 급증 시, 큐가 버퍼 역할
- **서비스 간 결합도 낮춤**
  - 생산자, 소비자가 직접 연결되지 않고, 메시지 큐를 사이에 둠
- **확장성**
  - 컨슈머를 수평 확장(scale-out)해서 처리량 늘림

---

### 메시지 전송 보장 방식
- **At-most once** (최대 한 번만)
  - 메시지가 유실될 수 있음, 중복은 없음
- **At-least once** (최소 한 번만)
  - 유실은 없지만, 중복 가능 -> 보통 이 방식을 기본으로 함
- **Exactly once** (정확히 한 번)
  - 한 번만, 빠짐없이, 중복 없이 처리
  - 유실도, 중복도 없음 (구현 복잡, 성능 비용 큼)
  - Kafka + 트랜잭션 기능을 통해 지원 가능
  - 이를 보장하려면 Producer, Broker, Consumer, DB 모두에서 정합성을 맞춰야 함
- 보통 메세지 큐(kafka)를 사용할 때 이 3부분에서 문제가 발생함
  - Producer -> Broker 구간: 네트워크 지연/Retry 때문에 메시지가 중복 기록될 수 있음
  - Broker -> Consumer 구간: Consumer가 메시지 받았지만 처리 중 죽으면 -> 다시 같은 메시지를 받음 (중복)
  - Consumer -> DB 처리 구간: DB insert 중 실패해서 Retry 하면 중복 insert 가능
- 여기서 잠깐! 
- Broker란?
  - 메시지를 저장하고 전달하는 서버
  - Kafka에서는 하나의 Broker = 하나의 Kafka 서버 프로세스
  - Producer -> 메시지를 Broker에 전송 -> Broker가 디스크에 저장 -> Consumer가 Broker에서 읽어감

---

### Message Queue 사용 시 고민한 점
- 메세지큐(Kafka) 도입을 위해 고민한 부분 정리하는 중~~

#### 순서 보장 (Ordering)
- 메시지 큐는 순서대로 처리될 거야~~ 라고 생각하기 쉽지만, 실제로는 **무조건 순서 보장을 하는 것은 아님**
- 예를 들어
  - Kafka는 **파티션 단위**로만 순서를 보장
  - RabbitMQ는 **Queue 단일 consumer**일 때만 순서 보장
- 문제 상황
  - 예를 들어 **잔액 차감 -> 거래 완료 기록** 이벤트가 있는데 순서가 바뀌면? -> 데이터 정합성 깨짐
- 해결 방법
  - Kafka에서는 **Key 기반 파티셔닝** 사용 -> 같은 key(예: 같은 사용자 ID)의 이벤트는 같은 파티션으로 들어가서 순서 유지
  - 전체 순서 보장이 필요한 경우? -> 단일 파티션만 쓰기 (하지만 성능 확장성은 낮아짐)
  - 따라서 내가 순서를 얼마나 강하게 보장해야 하지? 에 따라 결정해야 함
  - 뒤에 더 자세히 설명!

---

#### 중복 처리 (Duplicate Handling)
- MQ는 보통 **At-least-once**를 보장 -> 메시지가 중복 소비될 수 있음
- 문제 상황
  - DB Insert 두 번 -> 중복 데이터 발생
  - 결제 이벤트 두 번 -> 돈 두 번 빠짐(안돼~~)
- 해결 방법
  - **Idempotent Consumer** 설계 (멱등성 고려)
    - 메시지 ID 기반으로 **이미 처리한 메시지인지 체크** (ex. Redis나 DB에 processed flag 저장)
    - DB에서 `INSERT` 대신 `UPSERT`(중복 시 update) 사용
    - 결제/송금 같은 강한 정합성이 필요한 경우 -> DB 트랜잭션, unique key constraint 활용

---

#### 장애 대응 (Failure Handling)
- 문제 상황
  - Consumer가 죽으면? -> 메시지 안 읽힘
  - Broker가 죽으면? -> 데이터 유실 위험
  - 메시지 처리 실패하면? -> 무한 재시도? Dead Letter Queue?
- 해결 방법
  - Consumer 장애
    - Consumer Group을 활용 -> 같은 그룹 내 다른 인스턴스가 이어받음 (auto rebalance)
  - Broker 장애
    - Kafka는 Replication Factor ≥ 3 권장 -> 하나 죽어도 나머지로 복구
    - 하지만 failover 중엔 잠깐의 쓰기 불가 상태 발생 가능 -> 이거 감안하고 설계 필요
  - 메시지 처리 실패
    - 재시도 횟수 초과 시 -> DLQ (Dead Letter Queue)에 넣어 분석/재처리
    - DLQ: 정상적으로 처리되지 못한 메시지를 따로 모아두는 큐(장애 처리)
  - Fallback
    - 메시지큐 장애 시 DB 임시 기록 후 나중에 batch 재처리
    - 단, DB 부하 폭증 가능 -> 중요도 낮은 기능은 **Graceful Degradation**으로 멈추는 것도 고려

---

#### 백프레셔 (Backpressure)
- 큐에 메시지가 쌓이는데 소비 속도가 못 따라가는 상황
- 문제 상황
  - Kafka에 메시지는 쌓이는데 consumer 처리 속도가 느리면 → **lag 증가**
  - Lag: Producer가 쌓은 메시지 개수 - Consumer가 처리한 메시지 개수
  - 일정 이상 쌓이면 디스크 터짐 / 처리 지연 -> 장애 전파
- 해결 방법
  - Consumer Scale-out: Consumer 인스턴스 수 늘려 병렬 처리
  - Batch 처리: 하나씩 말고 묶어서 처리
  - QoS(품질 제어)
    - 중요도 낮은 메시지는 Drop
    - 우선순위 Queue 분리
  - Circuit Breaker
    - 특정 consumer가 과부하 걸리면 요청 차단/지연
  - Kafka의 경우 **Lag 모니터링** (Prometheus + Grafana) 필수

---

### Kafka (분산 메시지 스트리밍 플랫폼)
- 단순 메시지큐가 아니라 **대규모 로그 및 이벤트 스트리밍** 처리에 최적화된 플랫폼
- LinkedIn에서 시작 -> 현재 업계 표준
- 초당 **수백만 건 메시지 처리 가능**
- 메시지를 **디스크**에 저장해서 재처리 가능 -> 단순 큐와 큰 차이점

---

### Kafka의 구조
- Producer: 메시지를 발행하는 쪽
- Consumer: 메시지를 구독하고 처리하는 쪽
- Topic: 메시지를 분류하는 단위 (ex. `order-events`) (메시지 종류별 카테고리 느낌)
- Partition: 토픽을 여러 조각으로 나눈 것 -> 병렬 처리 가능
- Broker: Kafka 서버 (메시지를 저장/전송)
- Zookeeper(or KRaft): 클러스터 메타데이터 관리 (최근엔 KRaft로 대체 추세?)
- Offset: 메시지 위치 표시 (소비자가 어디까지 읽었는지 추적)

---

### Kafka의 특징
- 고성능
  - 메시지를 하나씩 쓰는 게 아니라, **여러 개 모아서(배치 처리) 씀**
  - 디스크에 기록할 때도 순차적으로 쭉~ 쓰기
- 확장성
  - 토픽을 파티션으로 나눠서 **여러 Consumer가 병렬**로 처리 가능
  - Broker(서버) 개수를 늘리면 처리량도 같이 확장됨
- 내구성
  - 메시지를 메모리에만 두는 게 아니라 **디스크에 저장**
  - 여러 Broker에 복제(Replication) 해서 한 대가 죽어도 데이터 안전
- Consumer Group
  - Consumer 여러 개를 하나의 그룹으로 묶으면 -> 파티션을 나눠서 각각 처리
  - 한 그룹 안에서는 중복 처리 없음 (파티션 하나 = Consumer 하나 전담)
- 재처리 가능
  - 메시지를 어디까지 읽었는지 offset으로 기록
  - offset을 되돌리면, **이전 메시지를 다시 읽어서 재처리** 가능
  - 장애 복구, 데이터 분석할 때 유용

---

### Kafka 메세지 처리 흐름
- Producer
  - 메시지를 특정 Topic에 발행
  - Key 해시값으로 Partition 선택 -> 같은 Key는 같은 Partition(순서 보장)
- Broker
  - 메시지를 디스크(Log)에 순차 저장
  - 각 메시지는 Offset으로 구분
  - Leader-Follower 구조로 복제하여 장애 대비
- Consumer
  - Topic 구독 후 메시지 읽음
  - Consumer Group 단위로 파티션을 나눠 병렬 처리
  - 처리 완료 후 Offset Commit -> 다음번 이어서 처리

--- 

#### Offset Commit 방식
- **Auto Commit** (기본값)
  - 주기적으로 자동 commit (예: 5초마다)
  - 처리 완료 전 commit -> 데이터 유실 가능
- **Manual Commit**
  - 메시지 처리 완료 후에 직접 commit
  - 유실은 막지만, 장애 시 중복 발생 가능
  - 따라서 중복 방지를 위해 Manual Commit + Idempotent Consumer 조합 사용
  - Idempotent Consumer: 같은 메시지를 여러 번 받아도 결과가 한 번 처리한 것과 같게 만드는 컨슈머

---

### Idempotent Producer / Idempotent Consumer
- Idempotent Producer
  - 같은 메시지를 여러 번 보내도 브로커에는 한 번만 기록되게 함
  - 네트워크 오류/재시도로 중복 전송이 발생하여 브로커에 중복 메세지가 쌓이지 않도록 사용
  - Kafka가 PID(Producer ID) + sequence number로 중복 식별하여 중복 기록 드롭
  - 단점
    - 성능 오버헤드: Producer와 Broker 간 추가 메타데이터(ID 관리 등) 처리 필요
    - Broker 의존성 증가: Broker 장애 시, 중복 보장 무너질 수 있음
    - Kafka는 기본적으로 단일 파티션 내에서만 순서+Idempotent 보장

- Idempotent Consumer
  - 같은 메시지를 여러 번 읽어도 처리 결과가 한 번 처리한 것과 동일하게 함
  - Kafka는 기본이 at-least-once라 중복 소비 발생 위험이 있어 사용
  - **고유 메시지 키 기반 중복 방지**
    - 메시지에 eventId(또는 비즈니스 키: orderId, paymentId) 포함
    - 처리 전 processed:eventId 존재 여부 확인(캐시/DB)
    - 없으면 처리 후 체크(set) / 있으면 스킵

---

### Kafka 사용 시 고민한 점
- MQ 고민한 점 연장선으로 좀 더 기술적인 내용!
  
#### 메시지 순서
  - Kafka는 **토픽 전체 순서는 보장 불가**
  - 파티션 단위로만 순서를 보장 (정확히는 파티션 내부 메세지끼리만 순서 보장)
  - Key hashing으로 같은 키는 같은 파티션으로 가게 설계해야 함
  - 전체 순서 정렬이 필요하다면? -> 파티션 1개만 두고 순차 처리

---

#### Exactly-once (정확히 한번만)
- Kafka는 원래 at-least-once -> 중복 가능
- Exactly-once를 만들려면
  - Producer에서 **idempotent producer 설정**
    - Producer가 같은 메시지를 여러 번 보내더라도 Broker가 중복을 걸러내고 단 한 번만 저장하도록 보장하는 설정
  - Consumer에서 처리 후 -> 커밋
  - Consumer + DB를 묶는 트랜잭션 필요 -> Kafka Transaction API 활용
  - 하지만 성능 오버헤드가 심해서 서비스에 따라 at-least-once + idempotent consumer로 타협

---

#### 백프레셔 (Lag 문제)
- Consumer가 메시지를 늦게 처리하면 -> Lag이 계속 쌓임
- Lag이 커지면
  - 실시간성이 깨짐
  - Broker 디스크에 부하
- 해결 방법
  - Consumer 인스턴스 늘리기 (Group 내 파티션 분배)
  - 메시지 처리 최적화 (batch 처리, DB bulk insert)
  - 중요도 낮은 메시지는 TTL 두고 자동 폐기

---

#### 장애 대응
- **Consumer 장애**
  - 동일 그룹 내 다른 Consumer가 이어받음 (Rebalance)
  - 문제: Rebalance 동안 처리 지연 발생
- **Broker 장애**
  - Replication으로 복구 가능
  - 새로운 Leader 승격 하는 동안 메시지 전송 불가 -> 복구 후 클라이언트 Retry 필요(자체 Retry 메커니즘이 존재)
    - 각 Topic-Partition에는 Leader Broker가 있음 -> Producer/Consumer는 Leader랑만 통신함
    - Leader가 죽으면, Follower 중 하나를 Leader로 승격해야 함 -> 이때 Controller Broker가 선출 절차를 관리함
- **네트워크 장애**
  - Producer ACK 옵션 (`acks=0,1,all`)
  - `acks=all` 설정하면 안전하지만 성능 저하
  - acks=0 빠르지만 유실 위험 / acks=1 기본값, 유실 가능 / acks=all 유실 방지,성능 낮아짐

---

#### 데이터 정합성 (Commit 시점)
- Kafka는 **메시지 읽고 나서 수동 커밋/자동 커밋** 가능
  - **처리 성공 전 커밋** -> 유실 위험
  - **처리 후 커밋** -> 중복 위험
- 따라서 처리 후 커밋 + idempotent consumer로 중복 처리 방어

---

### Message Queue + Batch
- 메시지 큐(MQ)
  - 이벤트가 발생하는 즉시 → Consumer가 읽어 처리 (**실시간성**)
  - 예: 주문 발생 시 바로 알림 발송, 재고 차감
- 배치(Batch)
  - 데이터를 모아두었다가 일정 시간/주기마다 한 번에 처리 (**일괄성**)
  - 예: 하루 동안 모인 거래 내역 -> 자정에 정산
- 같이 쓰는 이유
  - **실시간 처리 + 재처리**
    - MQ: 이벤트 즉시 처리 (알림, 로그 적재)
    - Batch: 혹시 MQ에서 실패하거나 누락된 메시지 있으면 다시 재처리
  - **빠른 응답 + 무거운 작업 분리**
    - MQ: 요청 즉시 큐에 넣고 빠른 응답 -> 가용성 높음
    - Batch: 큐에 쌓인 메시지를 일정량 모아서 DB Bulk Insert -> 성능 최적화
  - **정합성 보완**
    - MQ만 쓰면 장애로 메시지 일부 유실 가능
    - Batch를 돌려서 예전 데이서를 다시 맞춰주면 정합성 지킬 수 있음
- 그럼 같이 쓰는 상황을 고민해보자~
- 온라인 주문 시스템
  - 사용자가 주문 -> MQ에 “order_created” 이벤트 발행
  - Consumer가 바로 읽고 **알림 발송, 재고 차감** (실시간 처리)
  - 동시에 이벤트 로그는 Kafka에 계속 쌓임
  - 자정마다 Batch가 Kafka 로그를 읽어서 **일별 매출 집계, 정산 처리, 누락된 이벤트 재검증** 등을 실행함
- 기술적으로 고민할 점
  - **중복 처리**
    - MQ와 Batch가 같은 데이터를 두 번 반영할 수도 있음 -> Idempotent 설계 필요
  - **데이터 정합성**
    - MQ는 즉시 반영, Batch는 나중 반영 -> 타이밍에 따라 값이 다를 수 있음

---

### Message Queue + Batch + Scheduler
- 메시지 큐(MQ)
  - 이벤트가 발생하는 즉시 → Consumer가 읽어 처리 (**실시간성**)
  - 예: 주문 발생 시 바로 알림 발송, 재고 차감
- 배치(Batch)
  - 데이터를 모아두었다가 일정 시간/주기마다 한 번에 처리 (**일괄성**)
  - 예: 하루 동안 모인 거래 내역 -> 자정에 정산
- 스케줄러(Scheduler)**
  - 언제, 얼마나 자주 실행할지 제어
  - 배치 실행 트리거, retry job 실행
- 그럼 어떻게 같이 사용할까?
  - 이벤트 -> 큐 -> 배치
    - 주문 이벤트가 Kafka에 실시간으로 쌓임
    - 배치가 새벽 1시에 해당 Kafka Topic 읽어서 **일일 정산/통계 집계** 실행
    - MQ는 실시간 반영, 배치는 정합성 맞춤
  - 스케줄러 -> 배치 -> 큐
    - 스케줄러가 매일 새벽 2시에 배치 실행 -> 고객에게 발송할 쿠폰 목록 생성
    - 배치가 결과를 Kafka Topic에 넣음
    - MQ Consumer가 쿠폰 발송 이벤트를 읽어서 이메일/SMS 전송
  - MQ 실패 -> DLQ -> 스케줄러+배치 재처리
    - MQ Consumer가 처리 실패한 메시지 -> DLQ로 이동
    - 스케줄러가 10분마다 DLQ 재처리 배치를 실행 -> 성공 시 정상 큐로 다시 Publish
- 그러면 왜 이렇게 조합할까?
  - **실시간성 + 안정성**
    - MQ: 실시간성
    - Batch: 유실/중복/정합성 문제 제어
    - Scheduler: 실행 시점/주기 제어
    - MQ만 쓰면: 실시간은 되지만, 누락/중복 발생 시 회복 어려움
    - Batch만 쓰면: 정확하지만 실시간성이 떨어짐

---

## 3. 참고/추가 자료 (References)
- 가상 면접 사례로 배우는 대규모 시스템 설계 기초
- [Kafka 공식 문서](https://kafka.apache.org/documentation/)
- [Confluent Blog – A Guide to Consumer Offsets in Kafka](https://www.confluent.io/blog/guide-to-consumer-offsets)
- [Conduktor Docs – Idempotent Kafka Producer](https://learn.conduktor.io/kafka/idempotent-kafka-producer)

---

## 4. 내일/다음에 볼 것 (Next Steps)
- Bookdam 프로젝트 적용 고민 + 설계

