# 📚 Transaction

---

## 1. 주제/키워드
- 데이터베이스(MySQL)의 lock, transaction, isolation level, Spring의 트랜잭션, 분산 환경에서의 보상 트랜잭션에 대해 알아보자! ✌︎('ω')✌︎

## 📑 목차
- [📚 Transaction](#-transaction)
  - [1. 주제/키워드](#1-주제키워드)
  - [📑 목차](#-목차)
  - [2. 핵심 요약 (Summary)](#2-핵심-요약-summary)
    - [Transaction (트랜잭션)](#transaction-트랜잭션)
    - [MySQL의 Lock](#mysql의-lock)
      - [Global Lock](#global-lock)
      - [Table Lock](#table-lock)
      - [Named Lock](#named-lock)
      - [MetaData Lock](#metadata-lock)
      - [InnoDB 스토리지 엔진 lock](#innodb-스토리지-엔진-lock)
    - [Isolation Level](#isolation-level)
    - [InnoDB 스토리지 엔진](#innodb-스토리지-엔진)
    - [Propagation (트랜잭션 전파)](#propagation-트랜잭션-전파)
    - [Spring `@Transactional`](#spring-transactional)
    - [Self-invocation 문제](#self-invocation-문제)
    - [RollBack 규칙](#rollback-규칙)
    - [`readOnly = true`](#readonly--true)
    - [코드로 알아보는 `@Transactional`!! 와 신난다~~](#코드로-알아보는-transactional-와-신난다)
      - [트랜잭션을 이용한 Lazy Loading 문제 해결 (`readOnly = true`)](#트랜잭션을-이용한-lazy-loading-문제-해결-readonly--true)
      - [롤백 확인 (런타임 예외 시 롤백)](#롤백-확인-런타임-예외-시-롤백)
      - [REQUIRED (기본 전파 속성)](#required-기본-전파-속성)
      - [`REQUIRES_NEW` (이력 저장 분리 커밋)](#requires_new-이력-저장-분리-커밋)
      - [Self-invocation 문제 예시 \& 해결](#self-invocation-문제-예시--해결)
    - [분산 환경에서의 트랜잭션 (보상 트랜잭션)](#분산-환경에서의-트랜잭션-보상-트랜잭션)
      - [Saga 패턴 방식](#saga-패턴-방식)
      - [보상 트랜잭션 구현 사례 (토스뱅크 환전)](#보상-트랜잭션-구현-사례-토스뱅크-환전)
  - [3. 참고/추가 자료 (References)](#3-참고추가-자료-references)
  - [4. 내일/다음에 볼 것 (Next Steps)](#4-내일다음에-볼-것-next-steps)

---

## 2. 핵심 요약 (Summary)

### Transaction (트랜잭션)
- 작업 set을 모두 완벽하게 처리하거나, 처리 불가능인 경우에 원 상태로 복구하는 기능
- **작업의 완전성을 보장**(partial update 방지)
  - 부분 업데이트 현상이 발생하면, 남은 레코드를 삭제하는 재처리 작업이 필요할 수 있음 -> 다중 쿼리의 경우 매우 복잡!
- **최소한의 코드에만 적용해야 함!!**
- 사용자가 게시판에 게시글 작성 후 저장 버튼 눌렀을 때를 가정해보자
  >> 1. 처리 시작(DB connection 생성 + 트랜잭션 시작)
  >> 2. 로그인 확인
  >> 3. 글쓰기 오류 여부 확인
  >> 4. 첨부 파일 확인
  >> 5. 사용자 입력 내용 DBMS 저장
  >> 6. 첨부 파일 DBMS 저장 
  >> 7. 저장 내용 조회
  >> 8. 게시물 등록 알림 메일 발송
  >> 9. 알림 발송 이력 DBMS 저장 (트랜잭션 종료 Commit + DB connection 반납)
  >> 10. 처리 완료
  - 실제 데이터 저장은 5번에서 시작 -> 2,3,4의 과정은 트랜잭션에서 제거!
  - 8번과 같은 메일 전송, 파일 전송 등 **네트워크 작업은 위험** -> 트랜잭션에서 제거!
    - 프로그램 실행 중 메일 서버와 통신할 수 없는 상황이 발생하면 전체가 위험해짐(서버 부하 가능성)
  - 사용자 입력을 저장하는 5,6번은 하나의 트랜잭션으로 묶어야함
  - 7번은 단순 내용 조회이므로 트랜잭션에서 제외
  - 9번은 5,6번과 다른 내용이므로 별도의 트랜잭션으로 분리
  - **DB connection을 가지고 있는 범위와 트랜잭션이 활성화돼 있는 프로그램 번위를 최소화!!**
    - DB connection 수는 보통 제한적이기 때문에, 소유 시간이 길어지면 사용 가능한 여유 커넥션 수가 줄어듦 -> 커넥션을 가지기 위해 대기하는 상황 발생

---

### MySQL의 Lock
#### Global Lock
- 한 세션에서 글로벌 락을 획득하면 다른 세션에서 `SELECT`를 제외한 대부분의 DDL, DML은 대기 상태
- MySQL 서버 전체에 영향 미침
- `FLUSH TABLES WITH READ LOCK` 명령으로 획득
- InnoDB 스토리지 엔진은 트랜잭션을 지원하기 때문에, 일관된 시점의 데이터 백업에는 글로벌 락보다 **트랜잭션 기반 Consistent Snapshot 방식**이 더 일반적
- 글로벌 락은 주로 MyISAM 같은 트랜잭션 미지원 스토리지 엔진에서 백업을 위해 사용
- 글로벌 락이 걸린 상태에서는 DDL까지 포함해 대부분의 쓰기 작업이 전부 정지하므로, **운영 DB에서는 위험도가 큼**
- 대안: **백업락**
  - 테이블 스키마, 사용자 인증 관련 정보 변경은 허용하지 않음
  - 일반적인 테이블의 데이터 변경은 허용
  - 주로 **Replica 서버**에서 사용(데이터 백업)
  - Replica에서 글로벌 락을 획득하면, 백업 시간만큼 복제가 지연됨
  - 백업락은 글로벌 락과 달리 **DML(INSERT/UPDATE/DELETE)은 허용하므로 복제가 정상적으로 실행**됨
  - 하지만 스키마 변경(DDL)은 막혀 있기 때문에, **스키마 불일치로 인한 백업 실패 방지 가능**
  - MySQL 8.0 이상에서 지원 (LOCK INSTANCE FOR BACKUP)
  - 대규모 서비스에서는 Percona XtraBackup 같은 Hot Backup 툴을 백업락과 함께 많이 활용

---

#### Table Lock
- 개별 테이블 단위로 설정되는 lock
- 명시적 방법
  - `LOCK TABLES table_name [READ : WRITE]` 으로 획득
  - `UNLOCK TABLES` 으로 해제
  - 일반적인 애플리케이션에서는 **거의 사용하지 않음** -> 테이블 단위 전체가 막히므로 온라인 서비스에 영향이 큼
  - 주로 MyISAM 같은 트랜잭션 미지원 스토리지 엔진에서 데이터 일관성을 맞추거나, **단순/특수 작업**(데이터 마이그레이션, 배치 작업 등)에 사용됨
- 묵시적 방법
  - 데이터 변경 쿼리 실행 시 해당 테이블에 락 걸고, 변경 완료 후 잠금 해제
  - MyISAM 테이블
    - INSERT, UPDATE, DELETE 시 자동으로 테이블 단위 락 걸림
    - 읽기/쓰기 작업 간 충돌 발생 -> 동시성 낮음
  - InnoDB 테이블
    - **레코드 락**(Row Lock) 기반으로 동작하므로 일반적인 데이터 변경 쿼리에서는 테이블 락이 아닌 **행 단위 락이 사용**됨
    - 테이블 락은 **스키마 변경**(DDL, ALTER TABLE 등)에서만 의미가 있음
    - DDL은 내부적으로 **MDL**(Metadata Lock)이 걸려, 실행 중인 다른 DML/DDL을 막을 수 있음

---

#### Named Lock
- **특정 문자열**(Key)에 대해 잠금 설정 가능
- `GET_LOCK`(str, timeout) 함수로 획득, `RELEASE_LOCK`(str) 함수로 해제
- 서버 단위(Session 단위) 잠금이므로, 같은 MySQL 인스턴스 내에서만 유효
- 사용 이유
  - 하나의 자원을 여러 세션이 동시에 접근하지 않도록 **동기화 제어**
  - 복잡한 레코드 변경 작업, 배치 프로그램, 스케줄러 실행 시 **충돌 방지**
  - 분산 락처럼 활용 가능하지만, MySQL 인스턴스 단일 범위에 국한됨
- 예시
  - **배치 프로그램**이 대량 레코드를 변경하는 동안 **다른 세션에서 동일한 데이터를 건드리지 못하게 네임드 락 설정**
  - 특정 프로세스가 동시에 두 번 실행되지 않도록 방지 (GET_LOCK('batch_job', 10))
- 주의사항
  - Named Lock은 트랜잭션 롤백과 무관 -> 트랜잭션이 롤백돼도 락은 자동 해제되지 않음
  - 세션이 끊기면 자동 해제됨
  - 잘못 사용하면 병목 구간이 생길 수 있어 **락 범위를 최소화**해야 함

---

#### MetaData Lock 
- **DB 객체**(테이블, 뷰 등)의 **이름이나 구조를 변경**하거나 **접근**할 때 자동으로 획득하는 lock
- 예시: `RENAME TABLE tab_a TO tab_b` 실행 시 자동 획득
  - 원본 이름과 변경된 이름 모두에 대해 동시에 락 설정
  - DML/DDL이 동시에 실행될 때 충돌을 방지
- 특징
  - 모든 DDL, DML은 관련 객체에 대해 MDL을 자동 획득
  - DML(SELECT, INSERT, UPDATE 등): 공유 잠금(Shared Lock) -> 읽기/쓰기 가능
  - DDL(ALTER TABLE, RENAME 등): 배타 잠금(Exclusive Lock) -> 다른 세션의 접근 차단
  - **MDL은 트랜잭션이 끝날 때까지 유지됨** -> 긴 트랜잭션이 있으면 다른 세션의 DDL이 무기한 대기할 수 있음
  - 명시적으로 제어할 수 있는 게 아니라, MySQL이 내부적으로 걸어주는 시스템 락
- 예시
  - **배치 프로그램**에서 테이블을 동시에 여러 개 rename 하려는 경우
    - `RENAME TABLE A TO B, C TO A` 처럼 한 번에 실행해야 안전
    - 나눠서 실행하면, **MDL 충돌**로 인해 Table not found 오류 발생 가능
  - 테이블 구조 변경(DDL)은 단일 스레드 작업이라 오래 걸릴 수 있음
    - 해결 방법
      - 새로운 테이블을 만들고 **데이터를 병렬 복사**(시간 단축)
      - **최근 데이터만 잠깐의 락**을 걸고 복사 -> 다운타임 최소화
      - 마지막으로 **RENAME TABLE로 테이블 교체**
- 주의사항
  - 긴 쿼리(특히 오래 걸리는 SELECT)가 실행 중인 상태에서 DDL 실행하면, DDL이 MDL 획득 대기 상태로 멈춰버림
  - 이런 상황은 MySQL 서버 운영에서 자주 장애 원인이 되므로, 운영 중에는 DDL 실행 시각 조정이 중요

---

#### InnoDB 스토리지 엔진 lock
- Record Lock
  - **인덱스**의 레코드(=Row)를 잠금
  - 특정 레코드 단위로 **동시성 제어**
  - 인덱스가 없는 경우에도 InnoDB는 내부적으로 **숨은 클러스터 인덱스(자동 생성 PK)**를 이용해 잠금 설정
  - 따라서 WHERE 조건으로 검색된 레코드가 잠김
- Gap Lock
  - 레코드와 레코드 사이의 간격을 잠금
  - 조건에 맞는 레코드 자체뿐 아니라, 그 사이에 **새로운 레코드가 삽입되는 것까지 제어**
  - 주로 REPEATABLE READ 격리 수준에서 발생
  - 팬텀 리드(Phantom Read)를 방지하기 위한 잠금
- Next Key Lock
  - Record Lock + Gap Lock을 합친 형태
  - 특정 레코드와 그 전후 간격까지 동시에 잠금
  - MySQL 기본 격리 수준(`REPEATABLE READ`)에서 UPDATE/DELETE 시 자동 적용
  - `innodb_locks_unsafe_for_binlog`옵션이 비활성화된 경우 강제 사용
  - 목적: 바이너리 로그를 이용한 복제 시, **소스 서버와 레플리카 서버의 실행 결과를 일관되게 유지**하기 위함
- 자동 증가 락 (Auto-Inc Lock)
  - `AUTO_INCREMENT` 컬럼이 있는 테이블에 여러 트랜잭션이 동시에 INSERT 할 때, **자동 증가 값의 중복을 방지**하기 위해 테이블 단위 락을 사용
  - 기본적으로 짧은 시간 동안만 테이블 수준 잠금을 잡음 (insert 되는 동안만)
  - MySQL 8.0에서는 `innodb_autoinc_lock_mode` 설정으로 동작 방식 제어 가능
    - 0 (traditional): 모든 INSERT에 테이블 락 (안전하지만 동시성 낮음)
    - 1 (consecutive, 기본값): 단순 INSERT는 빠른 락, 복잡 INSERT(SELECT 기반)는 테이블 락
    - 2 (interleaved): 가장 동시성 높지만, binlog가 STATEMENT 모드면 위험

---

### Isolation Level
- 여러 트랜잭션이 동시에 처리될 때, 한 트랜잭션이 다른 트랜잭션의 변경 또는 조회 결과를 볼 수 있을지 결정하는 수준
- `READ UNCOMMITED`
  - 트랜잭션의 commit, rollback 여부 상관없이 다른 트랜잭션의 변경 내용도 읽을 수 있음 -> **Dirty Read 발생** 가능
  - InnoDB는 MVCC 기반이긴 하지만, 이 레벨에서는 undo 로그를 무시하고 최신 데이터를 바로 조회함
  - Dirty Read: 트랜잭션에서 처리한 작업이 완료되지 않았을 때, 다른 트랜잭션에서 볼 수 있는 현상
  - 데이터 정합성에 문제가 많이 발생
- `READ COMMITED`
  - **commit이 완료된 데이터만** 다른 트랜잭션에서 조회 가능 -> Dirty Read 없음
  - **트랜잭션 내에서 같은 쿼리라도 여러 번 실행하면 다른 결과**가 나올 수 있음 (Non-repeatable Read 발생 가능)
  - 각 SELECT 시점마다 새로운 Read View 생성
  - 이러한 문제는 하나의 트랜잭션에 동일 데이터에 여러 번 접근하는 금융 시스템에서 위험할 수 있음
  - Oracle 기본
- `REPEATABLE READ`
  - MVCC를 위해 **undo 영역 백업 데이터**를 이용해 **트랜잭션 내에는 같은 결과를 반환**할 수 있도록 보장
  - MVCC(Multi Version Concurrency Control): 트랜잭션이 rollback 될 가능성을 대비해 변경 전 레코드를 undo 공간에 백업해두고 실제 레코드 값을 변경하는 것
  - 모든 InnoDB의 트랜잭션은 고유한 순차적으로 증가하는 트랜잭션 번호를 가지고 있고, undo 백업 레코드에는 변경을 발생시킨 트랜잭션 번호를 포함함
  - `READ COMMITTED`는 매 SELECT마다 최신 커밋된 버전을 반환하고, `REPEATABLE READ`는 **트랜잭션 시작 시점의 스냅샷 버전을 끝까지 유지**
  - 팬텀 리드를 방지하기 위해 Next-Key Lock (Record Lock + Gap Lock) 사용
  - MySQL/InnoDB 기본
- `SERIALIZABLE`
  - 한 트랜잭션에서 읽고 쓰는 레코드는 다른 트랜잭션에서 무조건 접근 불가
  - SELECT도 기본적으로 공유 락을 걸어 실행되며 동시성이 매우 낮음
  - MySQL에서는 `REPEATABLE READ` 에서도 Next-Key Lock 을 활용해 팬텀 리드를 방지할 수 있기 때문에 사용 안함

| 격리 수준            | Dirty Read | Non-repeatable Read | Phantom Read | MVCC 일관성 | Lock 방식          | 설명            |
| ---------------- | ---------- | ------------------- | ------------ | -------- | ---------------- | ------------- |
| READ UNCOMMITTED | 가능         | 가능                  | 가능           | 작동 안 함   | 최신 버전만 읽음        | 가장 낮은 격리 수준   |
| READ COMMITTED   | 불가         | 가능                  | 가능           | 작동       | 매 SELECT마다 스냅샷   | 커밋된 데이터만 읽기   |
| REPEATABLE READ  | 불가         | 불가                  | 불가능          | 최고       | Next-Key Lock 사용 | 기본, 일관된 읽기 보장 |
| SERIALIZABLE     | 불가         | 불가                  | 불가능          | 작동       | Shared 락 강화      | 가장 안전하지만 느림   |


--- 

### InnoDB 스토리지 엔진
- MySQL 8.0 기준으로 기본 스토리지 엔진
- 주요 특징
  - ACID 트랜잭션 지원
    - COMMIT, ROLLBACK, Crash Recovery를 통해 데이터의 일관성 및 신뢰성 보장
  - 행 레벨 잠금(Row-Level Locking) 및 MVCC (다중 버전 동시성 제어)
    - 높은 동시성 처리 성능 제공
  - 클러스터형 인덱스(Clustered Index)
    - 기본 키 기반 데이터 물리 저장으로 효율적인 조회 및 I/O 최적화
  - 외래 키 제약(Foreign Key Support)
    - 테이블 간 참조 무결성 유지
  - 버퍼 풀(Buffer Pool)
    - 자주 접근되는 데이터를 메모리에 캐싱하여 속도 향상
  - 온라인 구조 변경(Online DDL), Full-Text 및 공간(Spatial) 인덱스 지원, 암호화, 압축 등
- InnoDB 의 순수 select, create는 아무런 lock 이 없음(Non-locking consistent lock)

| 항목             | InnoDB        | MyISAM (이전 기본 엔진) |
| -------------- | ------------- | ----------------- |
| 트랜잭션 지원        | 있음 (ACID)     | 없음                |
| 잠금 방식          | 행 레벨 (동시성 높음) | 테이블 레벨 (동시성 낮음)   |
| 외래 키           | 있음            | 없음                |
| Crash Recovery | robust        | 취약                |
| 기본 엔진          | MySQL 5.5 이후  | MySQL 5.5 이전      |

---

### Propagation (트랜잭션 전파)
- 트랜잭션 안에서 다른 `@Transactional` 메서드를 호출했을 때, 기존 트랜잭션에 참여할지, 새로 만들지, 끊을지를 결정하는 정책
- **REQUIRED (기본값)**: 기존 있으면 참여, 없으면 새로 시작 (가장 많이 씀)
- **REQUIRES NEW**: 무조건 새로 시작, 기존 트랜잭션은 잠시 중단 (로그/이력 저장에 활용)
- **SUPPORTS**: 있으면 참여, 없으면 트랜잭션 없이 실행
- **MANDATORY**: 기존 트랜잭션이 반드시 있어야 함, 없으면 예외
- **NOT SUPPORTED**: 트랜잭션 중단하고 비트랜잭션으로 실행
- **NEVER**: 트랜잭션이 있으면 예외 발생
- **NESTED**: 동일 트랜잭션 안에서 **세이브포인트**를 만들어 부분 롤백 가능 (JDBC 지원 필요)

---

### Spring `@Transactional`
- 메서드나 클래스 단위로 트랜잭션 경계(시작 -> 커밋/롤백)를 자동으로 관리해주는 스프링의 어노테이션
- 개발자가 `try-catch`로 직접 `commit/rollback` 하지 않아도 됨 -> **선언형 트랜잭션 관리**
- 실제론 **스프링 AOP 프록시**가 메서드 호출을 가로채서 트랜잭션을 시작/종료/롤백함
- 여기서 잠깐!!
- AOP란? (Aspect Oriented Programming)
  - 관점 지향 프로그래밍: 핵심 로직(비즈니스 로직)과 **부가 기능(공통 관심사)**을 분리해서 관리하는 방식
  - ex. 핵심: 주문 결제, 회원 가입, 게시글 작성 / 부가: 로깅, 보안, 트랜잭션, 성능 모니터링
  - 부가 기능을 직접 코드에 쓰지 않고, **한 곳에 모아 관리하고 실행 시 자동으로 끼워 넣음**!!
  - 장점
    - **중복 코드 제거**: 모든 서비스 메서드마다 try-catch로 트랜잭션 시작/종료 → 너무 반복+복잡
    - **유지보수 용이**: 공통 기능을 따로 관리하면 수정이 쉬움
    - **관심사의 분리**(SoC): 핵심 로직은 비즈니스에만 집중, 부가 기능은 AOP에서 관리

---

### Self-invocation 문제
- `@Transactional`은 기본적으로 **프록시 방식**으로 동작
- 즉, **외부에서 들어오는 메서드 호출**만 가로채서 트랜잭션을 적용
- **같은 클래스 내부에서 `this.method()`로 호출**하면 프록시를 안 거치므로 트랜잭션이 적용되지 않음 -> **self-invocation 문제**
- 해결 방법
  - 자기 자신을 프록시 형태로 주입받아 호출 (`this` 대신 주입받은 자기 자신 `self` 사용)
  - 트랜잭션을 분리할 메서드를 다른 클래스(Service)로 이동
  - AspectJ 모드 사용

---

### RollBack 규칙
- 기본적으로 **`RuntimeException`이나 `Error` 발생 시 롤백**
- 체크 예외(예: `IOException`)는 기본적으로 롤백 안 함
- 필요하면 `rollbackFor` / `noRollbackFor` 옵션으로 제어 가능

```java
@Transactional(rollbackFor = IOException.class)
public void test(...) { ... }
```

---

### `readOnly = true`
- **쓰기 작업이 없는 조회 전용 트랜잭션**에서 사용
- 장점
  - JPA/Hibernate는 불필요한 작업(변경 감지, flush 등)을 줄여 **조회 성능을 최적화**
  - 데이터베이스 드라이버에 읽기 전용 힌트를 전달하기도 함
- 주의: 무조건 쓰기 금지를 강제하는 건 아니고, 개발자 규율 차원에서 의미가 큼 (DB 마다 다름)

---

### 코드로 알아보는 `@Transactional`!! 와 신난다~~

#### 트랜잭션을 이용한 Lazy Loading 문제 해결 (`readOnly = true`)
  ```java
  @Override
  @Transactional(readOnly = true)
  public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
      UserEntity user = userRepository.findById(Long.valueOf(username))
                                      .orElseThrow(() -> new UsernameNotFoundException("User Not Found"));

      // 트랜잭션 시작 -> 트랜잭션 내부에서 role 접근 -> 영속성 컨텍스트가 살아있음 -> LazyInitializationException 방지
      user.setAuthorities(user.getRoles().stream()
          .map(role -> new SimpleGrantedAuthority(role.getRole()))
          .collect(Collectors.toList()));

      return user;
  }
  ```

---

#### 롤백 확인 (런타임 예외 시 롤백)

```java
@Service
public class OrderService {
    @Transactional
    public void placeOrder(Long userId, Long productId) {
        // Order 저장하는 로직
        orderRepository.save(new Order(userId, productId)); // 아직 커밋 아님(쓰기 지연)

        if (productId == 999L) { // 강제 예외(런타임 예외)
            // 트랜잭션 인터셉터가 예외를 감지해 트랜잭션을 롤백!!
            throw new IllegalStateException("재고 부족!");
        }

        // Payment 저장하는 로직
        paymentRepository.save(new Payment(userId, productId));
        // 메서드 정상 종료 -> 커밋 시도
    }
}
```
- 예외 발생 시 `Order`/`Payment` 모두 롤백됨

---

#### REQUIRED (기본 전파 속성)
- 이미 트랜잭션이 있으면 합류(join)하고, 없으면 새로 시작
```java
@Service
public class OrderService {
    @Transactional // 기본 propagation = REQUIRED
    public void placeOrder(Long userId, Long productId) {
        orderRepository.save(new Order(userId, productId));
        // 어랏! 기존에 실행중인 트랜잭션이 있구나!
        // 이미 진행 중인 같은 트랜잭션에 합류
        // 즉! 두 메서드(placeOrder + savePayment)는 한 트랜잭션으로 묶여 실행
        paymentService.savePayment(userId, productId); // REQUIRED
    }
}

@Service
public class PaymentService {
    @Transactional // propagation = REQUIRED (기본)
    public void savePayment(Long userId, Long productId) {
        paymentRepository.save(new Payment(userId, productId));
    }
}

```
- 예외 발생 시 전체 rollback (Order, Payment 둘 다 취소)

---

#### `REQUIRES_NEW` (이력 저장 분리 커밋)
- 현재 트랜잭션(있다면) 일시 중단 -> 새 물리 트랜잭션을 시작 (보통 다른 DB 커넥션 사용) -> `saveLog()` 실행 후 즉시 커밋/롤백 -> 기존 트랜잭션 재개

```java
@Service
public class LogService {
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void saveLog(String message) {
        systemLogRepository.save(new SystemLog(message));
    }
}
```

```java
@Service
public class TestService {
  private final TestService testService;
  private final LogService logService;

  @Transactional // 기본 REQUIRED
  public void test(...) {
    testService.test(...);           // 기존 트랜잭션
    logService.saveLog("주문 시도");       // REQUIRES_NEW(분리 커밋)
    // 여기서 예외가 나면 바깥 트랜잭션은 롤백되지만,
    // saveLog()는 이미 별도로 커밋 완료!!
  }
}
```
- 트랜잭션이 실패해도 로그는 **별도 트랜잭션으로 커밋**

---

#### Self-invocation 문제 예시 & 해결

```java
@Service
public class MemberService {

    private final MemberService self; // 자기 자신 프록시 주입

    public MemberService(MemberService self) {
        this.self = self;
    }

    @Transactional
    public void registerMember() {
        self.validateMember(); // 프록시를 거쳐 호출 -> 트랜잭션 적용됨
    }

    @Transactional
    public void validateMember() {
        ... 
    }
}
```

---

### 분산 환경에서의 트랜잭션 (보상 트랜잭션)
- 왜 어려울까?
  - 단일 서버/단일 DB에서는 간단 -> 트랜잭션으로 묶고 Commit or Rollback 하면 끝
  - 하지만 MSA 환경에선 **서로 다른 서버와 DB에서 하나의 논리적 작업**(예: 환전)을 수행해야 함!!
  - 이때 한쪽은 성공, 다른 한쪽은 실패하면 데이터 불일치가 발생 -> 금융 서비스에선 치명적
- 대표적인 해결 방법
  - **2PC** (Two Phase Commit)
    - 코디네이터가 각 참여자에게 “커밋 가능?” 질문 -> 모두 OK 시 커밋, 하나라도 NO면 전체 롤백
    - 장점: 원자성 보장
    - 단점: 가장 느린 참여자까지 기다려야 함 -> 낮은 가용성/확장성 (하나의 서버가 느려지면 전체가 묶임)
  - **Saga 패턴**
    - 각 서비스는 **자기 로컬 트랜잭션만 커밋**
    - 도중에 실패 시, **보상 트랜잭션**(Compensating Transaction)으로 앞의 작업들을 취소
    - 장점: 높은 가용성 및 확장성
    - 단점: 중간 상태가 노출될 수 있음 + 보상 트랜잭션을 직접 구현해야 함

--- 

#### Saga 패턴 방식
- Choreography Saga
  - 중앙 제어자 없음
  - 서비스끼리 이벤트 발행/구독으로 이어짐
  - 장점: SPOF 없음, 느슨한 결합
  - 단점: 상태 추적/디버깅 어려움
- Orchestration Saga
  - 중앙 오케스트레이터가 각 서비스에 명령
  - 장점: **상태 추적이 쉽고 모니터링 용이**
  - 단점: 오케스트레이터가 SPOF이 될 수 있음
- 토스뱅크 환전 서버는 Orchestration Saga를 채택 (환전 상태 추적 필요)

---

#### 보상 트랜잭션 구현 사례 (토스뱅크 환전)
- 정상 시나리오
  - 원화 출금 성공 -> 외화 입금 성공 -> 환전 성공
- 출금 실패
  - 그냥 실패 처리 (아직 돈 안 나갔으니 끝)
- 입금 실패
  - 이미 원화가 출금된 상태 -> **보상 트랜잭션 실행** (**출금 취소** = 재입금)
  - 유저 입장에서는 ±0원이 되어 정합성 보장!!
- 에러/네트워크 타임아웃
  - 결과 확인 재시도 -> **Kafka 지연 메시지 스케줄러**로 일정 시간 뒤 재확인
  - 그래도 실패 시 **배치 프로세스로 미완료 환전 재처리**
- 트랜잭셔널 메시징
  - 출금 취소 메시지 발행과 환전 실패 처리는 **원자적으로 같이** 이루어져야 함
  - 토스뱅크는 Dead Letter Queue(DLQ) + Transactional Outbox 유사 방식으로 보장

- 설계 시 고려 포인트
  - 항상 출금 -> 입금 순서로 진행해야 함 (입금 먼저 하면 돈이 이미 빠져나갈 수 있어 회수 불가 위험)
  - **실시간성 vs 보상 트랜잭션 지연 처리의 균형**
    - 즉시 처리(HTTP Sync): 유저가 기다리는 구간 (출금/입금)
    - 지연 처리(Messaging): 유저가 기다리지 않는 구간 (보상 트랜잭션)
  - 모니터링/스테이트 머신으로 각 환전 상태 추적 + 미완료 환전 탐지

- 결론
  - 분산 환경에서 트랜잭션을 완벽히 ACID로 보장하기는 어려움
  - 따라서 **보상 트랜잭션 + 결과적 정합성**(Eventual Consistency)을 수용
  - 토스뱅크 환전 서비스는 Saga 패턴(Orchestration) 을 도입해 높은 트래픽, 확장성 속에서도 안정적인 정합성을 유지함!

---

## 3. 참고/추가 자료 (References)
- [MySQL Transaction Isolation Levels](https://dev.mysql.com/doc/refman/8.4/en/innodb-transaction-isolation-levels.html)
- [MySQL InnoDB Introduction](https://dev.mysql.com/doc/refman/8.4/en/innodb-introduction.html)
- RealMySQL 8.0
- [Spring Framework Docs – Transaction Management](https://docs.spring.io/spring-framework/reference/data-access/transaction.html)
- [Baeldung – Spring @Transactional](https://www.baeldung.com/transaction-configuration-with-jpa-and-spring)
- [토스ㅣSLASH 24 - 보상 트랜잭션으로 분산 환경에서도 안전하게 환전하기](https://youtu.be/xpwRTu47fqY?si=7J5uEtDUvoXmDwmh)

---

## 4. 내일/다음에 볼 것 (Next Steps)
- 진심 너무너무너무 어려워서 반도 이해 못한 것 같음... 
- 나중에 다시 더 깊게 공부하자... :;(∩´﹏`∩);:
- 난 진짜 바보다

