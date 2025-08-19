# 📚 Transaction

---

## 1. 주제/키워드
- 데이터베이스(MySQL)의 lock, transaction, isolation level에 대해 알아보자! ✌︎('ω')✌︎

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
### Global Lock
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

### Table Lock
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

### Named Lock
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

### MetaData Lock 
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

### InnoDB 스토리지 엔진 lock
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

## Isolation Level
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

## 3. 참고/추가 자료 (References)
- [MySQL Transaction Isolation Levels](https://dev.mysql.com/doc/refman/8.4/en/innodb-transaction-isolation-levels.html)
- [MySQL InnoDB Introduction](https://dev.mysql.com/doc/refman/8.4/en/innodb-introduction.html)
- RealMySQL 8.0

---

## 4. 내일/다음에 볼 것 (Next Steps)
- 진심 너무너무너무 어려워서 반도 이해 못한 것 같음... 
- 나중에 다시 더 깊게 공부하자... :;(∩´﹏`∩);:

