# 🛠️ Flyway 도입 문제 해결

---

## 1. 문제 상황 (What happened?)
- Flyway 적용 후 애플리케이션 실행 시 에러 발생

  ```
  Caused by: org.flywaydb.core.api.FlywayException:
  Found non-empty schema(s) `bookdam` but no schema history table.
  Use baseline() or set baselineOnMigrate to true to initialize the schema history table.
  ```
- DB에는 이미 테이블이 존재했지만, Flyway의 `flyway_schema_history` 테이블이 없어 마이그레이션 이력 관리가 불가
- 또, JPA 옵션을 `ddl-auto=validate` 로 두었을 때 Hibernate가 엔티티와 DB 스키마를 비교하다 `missing table [like]` 등의 오류가 발생

---

## 2. 원인 분석 (Why?)
1. 스키마 상태와 Flyway의 이력 불일치
   - Flyway는 DB 스키마 변경 내역을 `flyway_schema_history` 테이블에서 추적
   - 하지만 기존에 수동 생성된 테이블이 있었고, Flyway 입장에서는 비어 있지 않은 DB인데 이력 없음 -> 에러 발생

2. Hibernate validate 모드 문제
   - `ddl-auto=validate`는 엔티티와 DB 스키마가 정확히 일치하지 않으면 오류를 발생시킴
   - 예약어(`like`, `view`) 사용 등으로 엔티티 테이블명과 실제 DB 테이블명이 불일치 -> missing table 오류

---

## 3. 개념 정리 (Key Concepts)
### Flyway 도입 이유
   - 스키마를 코드처럼 관리
     - DB 스키마도 Git으로 버전 관리 가능 -> 언제, 누가, 어떤 변경을 했는지 추적
   - 모든 환경 동기화
     - 로컬/CI/운영 DB가 항상 동일한 마이그레이션 이력을 공유

### Flyway Schema History
- Flyway는 `flyway_schema_history` 테이블을 생성해, 적용된 마이그레이션 파일(`V1__init.sql`, `V2__add_index.sql` 등등 )의 버전과 체크섬을 기록
- 한 번 적용된 버전 파일은 수정 불가 -> 변경이 필요하면 반드시 새로운 버전(`V3__...sql`) 추가

### 마이그레이션 파일 규칙
- `V1__init.sql` : 최초 스키마 정의
- `V2__xxx.sql`, `V3__xxx.sql` : 변경 사항 누적
- `R__xxx.sql` : 반복 실행 (뷰/프로시저/시드 데이터)

### 적용 흐름
1. 애플리케이션 부팅 시 Flyway가 `db/migration` 폴더 스캔
2. `flyway_schema_history` 확인 후, 아직 적용 안 된 버전만 실행
3. 실행 성공 -> 해당 버전 기록 저장
4. 이후 JPA가 `ddl-auto=validate` 모드에서 엔티티와 DB 스키마 정합성을 검증

### JPA ddl-auto 옵션 차이
- validate
  - 엔티티 와 DB 스키마를 비교만 함
  - 불일치 시 예외 발생 -> 애플리케이션 실행 실패
  - Flyway 같은 마이그레이션 툴과 함께 쓰면 안전
- update
  - 엔티티 기준으로 DB 스키마를 자동 수정(테이블/컬럼 생성, 변경)
  - 개발 초기 편리하지만, 변경 이력이 남지 않아 운영/협업 환경에서는 위험

---

## 4. 최종 해결 방법 (How fixed)
1. 데이터 날려도 되는 경우
   - `docker compose down -v` 로 볼륨 삭제 -> DB 초기화
   - 재시작 시 Flyway가 `V1__init.sql`부터 적용

2. 데이터 유지해야 하는 경우
   - `spring.flyway.baseline-on-migrate=true` 설정 후 첫 migrate 실행
   - 현재 스키마를 baseline으로 등록 -> 이후부터 `V2+` 파일만 적용

3. Hibernate validate -> update로 변경
   - `ddl-auto=validate`는 스키마가 조금이라도 다르면 오류 발생
   - 개발 단계에서는 `ddl-auto=update`로 바꿔 Hibernate가 자동으로 부족한 테이블/컬럼 생성 -> 부팅 성공
   - 단, 실전 운영에서는 update가 아니라 안전하게 validate + Flyway 사용

---

## 5. 결과 및 느낀점 (Result & Learnings)
- 호기롭게 flyway 도입을 도전했지만,,, 쉽지 않았다
- 사실 이 프로젝트에서는 굳이 도입하지 않아도 되지만 flyway 개념 공부 겸 한번 도전해봤다.

