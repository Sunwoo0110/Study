# 📚 Spring JPA

---

## 1. 주제/키워드
- JPA 의 개념, 동작 원리, 주의 사항에 대해 공부해보자~ (つ•̀ω•́)つ

## 📑 목차

- [📚 Spring JPA](#-spring-jpa)
  - [1. 주제/키워드](#1-주제키워드)
  - [📑 목차](#-목차)
  - [2. 핵심 요약 (Summary)](#2-핵심-요약-summary)
    - [JPA (Java Persistence API)](#jpa-java-persistence-api)
    - [Hibernate](#hibernate)
    - [Spring Data JPA](#spring-data-jpa)
    - [JPA 동작 원리](#jpa-동작-원리)
      - [코드로 파보는 JPA 동작](#코드로-파보는-jpa-동작)
      - [JPA 작동 원리 + Spring @Transactional 흐름 정리](#jpa-작동-원리--spring-transactional-흐름-정리)
    - [JPA 문제: N+1 문제](#jpa-문제-n1-문제)
      - [N+1 문제 상황 예시](#n1-문제-상황-예시)
      - [해결 방법](#해결-방법)
      - [Batch Fetch (`@BatchSize`, `hibernate.default_batch_fetch_size`)](#batch-fetch-batchsize-hibernatedefault_batch_fetch_size)
      - [DTO 전용 조회](#dto-전용-조회)
      - [`Fetch Join`](#fetch-join)
      - [`@EntityGraph`](#entitygraph)
    - [JPA 문제: 동시성 제어](#jpa-문제-동시성-제어)
      - [낙관적 락 (Optimistic Lock)](#낙관적-락-optimistic-lock)
      - [비관적 락 (Pessimistic Lock)](#비관적-락-pessimistic-lock)
      - [격리 수준(Isolation)과의 관계](#격리-수준isolation과의-관계)
    - [MyBatis vs JPA](#mybatis-vs-jpa)
    - [JPA 성능 최적화를 위한 고려 사항](#jpa-성능-최적화를-위한-고려-사항)
    - [자주 발생하는 실수 + 실제 발생했던 문제들 정리](#자주-발생하는-실수--실제-발생했던-문제들-정리)
    - [더 고민해볼 문제들](#더-고민해볼-문제들)
  - [3. 참고/추가 자료 (References)](#3-참고추가-자료-references)
  - [4. 내일/다음에 볼 것 (Next Steps)](#4-내일다음에-볼-것-next-steps)

---

## 2. 핵심 요약 (Summary)

### JPA (Java Persistence API)
- 자바에서 ORM(Object Relational Mapping) 표준 인터페이스
- **자바 객체(Entity)와 DB 테이블을 매핑**해주는 역할
- ```java
    // SQL 직접 작성 X
    User user = entityManager.find(User.class, 1L);
    ```
- 이렇게 객체 조회하면 내부적으로는 SELECT * FROM user WHERE id=1 실행됨
- 즉, SQL 대신 객체 지향적으로 데이터 접근 가능
- 장점
  - **생산성**이 높아짐
    - CRUD 코드 자동화 -> save(), findById() 등 기본 제공
    - SQL 직접 작성할 일이 줄어듦 -> 개발 속도 높아짐
  - **유지보수성**
    - 엔티티(클래스) 중심 설계 -> 비즈니스 로직 위주로 코드 작성
    - DB 변경 시에도 엔티티만 수정하면 됨 -> SQL 수정 최소화
  - **객체 지향적 개발**
    - 연관관계 매핑(@OneToMany, @ManyToOne) 객체 그래프 탐색 편리
    - 영속성 컨텍스트 + Dirty Checking 으로 객체 변경만 해도 DB 반영
  - **부가기능**
    - 캐시(1차 캐시), 지연 로딩(Lazy Loading), 배치 처리, Auditing 등 기본 기능
- 단점
  - **성능 이슈**
    - N+1 문제, 불필요한 쿼리 발생 가능
    - 잘못 쓰면 SQL 튜닝보다 상황이 더 꼬일 수 있음
  - **추상화의 한계**
    - 복잡한 SQL(예: 통계, 다중 조인, 윈도우 함수)은 JPQL로 한계 존재
    - Native Query 쓰는 상황 발생 가능 -> DB 종속성 증가
  - **디버깅 어려움**
    - SQL이 자동 생성되므로, 실제로 어떤 쿼리가 날아가는지 추적 필요 (Hibernate show_sql 등)
    - 문제 생겼을 때 JPA 내부 동작까지 파악해야 함
  - **동시성 처리**
    - Optimistic / Pessimistic Lock 따로 고려해야 함
    - 단순 트랜잭션만으로는 동시성 이슈 해결 어려움

### Hibernate
- JPA 구현체(Implementation) 중 하나
- JPA는 인터페이스이므로 직접 쓸 수 없음 -> Hibernate가 실제 동작(SQL 생성/캐시/더티체킹) 제공
- Spring에서는 거의 Hibernate를 JPA 구현체로 사용

### Spring Data JPA
- Spring에서 JPA를 더 쉽게 쓰게 해주는 모듈
- 반복적인 코드 (DAO, Repository) 줄여줌
- ```java
    public interface UserRepository extends JpaRepository<User, Long> {
        List<User> findByEmail(String email);
    }
    ```
- 위 코드 한 줄이면 -> SELECT * FROM user WHERE email = ? 자동 실행
- 이를 통해 개발자는 직접 쿼리 안 짜고, 메서드 네이밍 규칙만 지켜도 SQL 실행됨!

---

### JPA 동작 원리
- EntityManager로 명령을 받음 -> 영속성 컨텍스트(PC)에서 캐싱/변경감지/쓰기지연 관리 -> 트랜잭션 commit 시 flush -> SQL 실행

#### 코드로 파보는 JPA 동작
- `em.persist(entity)`
  - 엔티티를 PC(1차 캐시)에 저장
  - SQL 안 보냄 -> INSERT는 **쓰기 지연 SQL 저장소**에 쌓임
  - ```java
        em.persist(new User("sunwoo")); 
        // -> 1차 캐시 등록 + 쓰기 지연 저장소에 INSERT 준비
    ```

- `em.find(User.class, id)`
  - PC(1차 캐시) 먼저 확인
  - 있으면 DB 조회 안하고 바로 그대로 반환 (**동일성 보장**)
  - 없으면 DB 조회 후 PC에 저장하고 반환
  - ```java
        User u1 = em.find(User.class, 1L); // DB 조회 -> PC에 캐싱
        User u2 = em.find(User.class, 1L); // 1차 캐시 히트, 같은 객체 반환
        System.out.println(u1 == u2); // true
    ```

- `entity.setName("new")`
  - 엔티티는 단순히 자바 객체라서 setter 호출 시 값만 바뀜
  - 근데 PC는 처음 저장 시 스냅샷을 복사해둠
  - flush 시점에 스냅샷 vs 현재값 비교 -> 다르면 UPDATE SQL 생성
  - 이게 **Dirty Checking** (변경 감지)

- `transaction.commit()`
  - 커밋 시점에 JPA가 자동으로 `flush()` 호출
    - 쓰기 지연 SQL 저장소의 INSERT/UPDATE/DELETE 한꺼번에 DB로 전송
    - 트랜잭션 commit

#### JPA 작동 원리 + Spring @Transactional 흐름 정리

```java
@Transactional
public void run(EntityManager em) {
    // 1. 영속화
    User u1 = new User("sunwoo");
    em.persist(u1); 
    // -> 1차 캐시 + 쓰기 지연 저장소에 INSERT 대기

    // 2. 조회
    User u2 = em.find(User.class, u1.getId()); 
    // -> 같은 트랜잭션이므로 1차 캐시에서 바로 반환

    // 3. 변경
    u1.setName("newSunwoo"); 
    // -> 스냅샷과 비교 예정 (Dirty checking 준비)
} 
// 4. 트랜잭션 종료 시점
// flush: INSERT, UPDATE(더티 체팅으로 생성) SQL 전송
// commit: DB 반영
```

- **동일성 보장**: 같은 PK면 같은 객체(`==`) 반환
- **DB Hit 최소화**: 같은 트랜잭션에서 반복 조회 시 DB 재조회 안 함
- **쓰기 지연**: 성능 최적화, batch insert 가능
- **더티체킹**: `save()` 같은 명시적 호출 없이 변경 감지로 자동 UPDATE

---

### JPA 문제: N+1 문제
- JPA에서 엔티티는 보통 연관관계(`@ManyToOne`, `@OneToMany`)를 가짐
- 이 연관 엔티티를 언제 로딩할지가 **LAZY** / **EAGER**
- **LAZY** (지연 로딩)
  - 필요할 때 가져오는 방식
  - 실제 SQL을 바로 안 날리고 프록시 객체를 두었다가 getter 호출 시 쿼리 실행
  - 보통 지연 로딩 사용(안 그러면 터짐..)
- **EAGER** (즉시 로딩)
  - 주 테이블 불러올 때 연관 관계 데이터도 다 가져오는 방식
  - 여러 관계가 겹치면 쿼리 예측 어려움 -> N+1 문제 발생 가능
- 사용 방법
  - ManyToOne, OneToOne: fetch join (row 수 안 늘어나서 안전)
  - OneToMany, ManyToMany: batch fetch (row 중복/페이징 문제 때문에)
  - 복잡 화면: DTO 조회 (처음부터 화면에 맞는 데이터 뽑기)

#### N+1 문제 상황 예시
- 게시글(Post)과 작성자(User) 관계 (`Post N : 1 User`)

```java
List<Post> posts = postRepo.findAll();   // 여기서 1번 쿼리
for (Post p : posts) {
    System.out.println(p.getAuthor().getName()); // User 로딩할 때 N번 쿼리
}
```
- 첫 줄: `post` 전체 조회
- 루프 돌며 `getAuthor()` 호출 -> 매번 `select * from user where id = ?` 실행 -> **N번 쿼리**
- 총 **1+N 쿼리** 발생 -> N+1 문제
- 데이터 많아질수록 SELECT 많이 발생 (1000개 post면? 1001번 쿼리)
- 즉, 컬렉션 수(N)에 비례해서 추가 쿼리가 N번 더 나감!

#### 해결 방법

#### Batch Fetch (`@BatchSize`, `hibernate.default_batch_fetch_size`)
- JPA가 LAZY 프록시를 한 번에 IN 쿼리로 묶어 로딩
- ```java
    @OneToMany(mappedBy = "post")
    @BatchSize(size = 100)
    private List<Comment> comments;
    ```
- 배치 사이즈만큼 뭉탱이로 쿼리 -> IN 으로 묶어서 조회
- 장점: 다대일/컬렉션 모두 사용 가능, 실무에서 자주 씀
- 단점: fetch join만큼 깔끔하진 않고, SQL IN 사이즈 한계 고려

#### DTO 전용 조회
- 처음부터 필요한 데이터만 SELECT (JPQL, QueryDSL 등으로 DTO 직접 매핑)
- ```java
    @Query("select new com.example.PostDto(p.id, p.title, u.name) " +
        "from Post p join p.author u")
    List<PostDto> findPostDtos();
    ```
- 장점: 최적화 최고, N+1 문제 발생 안함
- 단점: 화면 종속적인 코드 많아짐(UI 요구사항이 변할 때마다 DTO + JPQL/QueryDSL 코드를 변경해야함)
  
#### `Fetch Join`
- JPQL에서 JOIN + FETCH로 한 방에 가져오기
- ```java
    @Query("select p from Post p join fetch p.author")
    List<Post> findAllWithAuthor();
    ```
- 장점: 단순
- 단점: 컬렉션(fetch join) + 페이징 불가 -> 메모리 터짐

#### `@EntityGraph`
- Spring Data JPA 기능, 메서드 호출 시 자동 fetch join 적용
- ```java
    @EntityGraph(attributePaths = {"author"})
    List<Post> findByStatus(Status status);
    ```
- 장점: 코드 간결, 쿼리 자동 생성
- 단점: 복잡한 fetch 전략 조합은 어려움

---

### JPA 문제: 동시성 제어
- JPA는 엔티티를 메모리(영속성 컨텍스트) 안에서 관리하다가 트랜잭션 커밋 시점에 한 번에 DB에 반영(flush) 함
- 그런데 만약 여러 트랜잭션이 같은 엔티티에 접근한다면??
- **여러 트랜잭션이 같은 엔티티를 동시에 읽고, 수정하고, 커밋**하며 충돌이 발생할 수 있음!!
- 가장 흔한 문제 -> **Lost Update**(갱신 손실)
  - 두 사용자가 같은 데이터를 읽음 -> 각각 수정 -> 마지막에 커밋한 값만 남음
- JPA는 기본 트랜잭션 격리로는 Lost Update를 막지 못하기 때문에, **추가로 Optimistic/Pessimistic Lock**을 걸어야 함

#### 낙관적 락 (Optimistic Lock)
- **동시에 충돌 날 일은 드물다~** -> 일단 다 읽고 쓰게 두고, **커밋 시점에 충돌 감지**
- 방법: `@Version` 필드 사용 (숫자 or 타임스탬프)
- 엔티티에 `@Version` 필드를 둠 (int, long, timestamp 등 가능)
- 데이터를 읽어올 때 **현재 버전 값**을 같이 들고 옴
- 수정 후 커밋 시 `where id=? and version=?` 조건으로 update 실행
  - 수정이 성공하면 version +1
  - 실패하면(업데이트된 row 없음) -> 누군가 먼저 고쳤다는 뜻 -> OptimisticLockException
- 장점
  - DB 락을 걸지 않아 성능, 동시성 처리량이 좋음
  - 충돌 빈도가 낮은 시나리오(예: 게시글 수정, 유저 프로필 변경)에 적합
- 단점
  - 충돌 시 예외 -> 반드시 비즈니스 레벨에서 재시도 전략 필요

#### 비관적 락 (Pessimistic Lock)
- **어차피 충돌 자주 날 거면 미리 막자~~** -> **DB 차원에서 Lock**
- JPA에서는 `@Lock` 어노테이션이나 `EntityManager.lock()`으로 사용
- SQL `select ... for update` 실행 -> 다른 트랜잭션이 해당 row 접근 시 막음
- 장점
  - 충돌이 많아도 안전
  - Lost Update 방지 확실
- 단점
  - 데드락, 타임아웃 발생 위험
  - 락 잡고 있는 동안 다른 트랜잭션 대기 -> 동시성 처리량 낮아짐
  - 따라서 **트랜잭션을 짧게 유지**해야 함!

#### 격리 수준(Isolation)과의 관계
- DB의 **트랜잭션 격리 수준**도 동시성에 영향을 미침

| Isolation Level     | 특징                                       | JPA에서 보완 방법   |
| ------------------- | ---------------------------------------- | ------------- |
| Read Committed (기본) | Dirty Read 방지, 하지만 **Lost Update 발생 가능** | 낙관적 락으로 보완  |
| Repeatable Read     | 같은 row는 항상 같은 값 보장, 그러나 갱신 충돌은 여전히 발생    | 낙관락 or 비관락 필요 |
| Serializable        | 모든 충돌 방지 (완벽), 대신 성능 매우 안좋음                | 실무에서 거의 안 씀   |

---

### MyBatis vs JPA 

| 항목       | JPA(Hibernate)         | MyBatis            |
| -------- | ---------------------- | ------------------ |
| 개발 속도    | CRUD/페이징 빠름            | SQL 직접 -> 초반 느림   |
| 유지보수     | 객체 모델 중심, 변경에 강함       | SQL이 명확, 튜닝/디버깅 직관 |
| 성능 제어    | 추상화 높아 의도치 않은 쿼리 문제 발생 가능 | 쿼리 100% 통제         |
| 복잡 쿼리/집계 | DTO/네이티브 필요            | 복잡 조인/리포트에 용이  |
| 학습 곡선    | 영속성 컨텍스트/fetch 이해 필요      | SQL 알면 바로          |
| 대규모 목록   | DTO 조회 설계 필요           | 필요한 컬럼만 깔끔히        |

- 트랜잭션이 중요하고 CRUD 중심 -> **JPA**
- 리포트/집계/튜닝 중심, SQL이 중요 -> **MyBatis**
- 같이 쓰기도 함

---

### JPA 성능 최적화를 위한 고려 사항
- **N+1 문제 고려**
  - 실행되는 SQL을 꼭 로깅하거나 테스트에서 재현해봐야 함
  - 발견되면 fetch join / EntityGraph / 배치 로 쿼리 수를 줄이기!
- **배치 INSERT/UPDATE**
  - `Hibernate jdbc.batch_size`와 `order_inserts/updates=tru`e를 켜면 여러 INSERT/UPDATE를 묶어서 날림 -> 네트워크 round trip 줄어 성능 크게 개선!
  - Round trip: 한 번 DB에 갔다 오는 왕복
- **DTO 조회**
  - 목록 화면이나 리스트+상세 같이 전송량이 많은 API는 엔티티 전체 불러오지 말고 DTO로 필요한 필드만 select
  - SQL/네트워크/직렬화 비용 절약
- **OSIV OFF**
  - Open Session In View 끄고, 서비스 계층에서 **필요한 연관관계까지 미리 로딩 후 DTO로 반환**
  - Open Session In View: 서비스/트랜잭션 끝나고도 컨트롤러나 뷰(JSP, JSON 직렬화 등) 에서 LAZY 필드를 접근 가능도록 함
  - LazyInitializationException 방지 + 트랜잭션 범위 명확
- **벌크 연산 후 PC 정리**
  - `@Modifying(clearAutomatically=true)` 또는 `em.clear()` 호출
  - 벌크 연산은 1차 캐시 무시하기 때문에, PC와 DB 데이터가 불일치하는 문제를 막아야 함
  - DB는 최신 상태인데 PC 안의 엔티티는 옛날 값 그대로 남아 있음
  - 이후 동일 엔티티를 다시 조회하면 DB 값이 아니라 1차 캐시에 있던 구버전이 반환되는 문제!
- **인덱스 설계**
  - 조회/정렬/조인 키에 맞는 복합 인덱스 꼭 고려
  - DB 성능 튜닝의 절반은 인덱스 설계라고 할 정도... JPA만 튜닝해도 인덱스 없으면 무의미
- **읽기 전용 트랜잭션**
  - `@Transactional(readOnly = true)`를 쓰면 flush/더티체킹이 비활성화
  - 성능 최적화 + DB 드라이버에 읽기 전용 힌트도 전달 가능

---

### 자주 발생하는 실수 + 실제 발생했던 문제들 정리
- **LazyInitializationException**: 트랜잭션 밖에서 LAZY 접근
  - OSIV OFF라면 **서비스 계층에서 미리 로딩(fetch/EntityGraph)** 후 DTO로 반환
- **N+1 폭탄**: EAGER/무분별한 LAZY 접근
  - Batch Fetch, DTO 조회 등
- **save()로 업데이트 남발**: merge 경로로 원치 않는 UPDATE
  - Spring Data JPA의 save()는 **새로 저장**(persist)도 하고 **병합**(merge)도 함
  - 항상 **조회(영속화) -> 변경(더티체킹) -> 커밋**
- **equals/hashCode**를 ID로 생성 전부터 사용: ID null인 상태로 조회 시 문제
  - **불변 비즈니스 키**(이메일 등) 사용
- **ManyToMany**: 중간 테이블 숨김
  - **중간 연결 엔티티**를 사용해야 확장 가능
- **DDL 자동 반영(prod)**: `ddl-auto`는 개발용
  - 운영은 **Flyway/Liquibase**

---

### 더 고민해볼 문제들
- **트랜잭션 경계**
  - 항상 말하지만 트랜잭션은 최소 범위로!
  - 조회+외부 API+저장 처럼 한번에 많은 작업은 하지 말자~
- **API 응답 모델**
  - **엔티티 직접 노출 금지**, DTO로 반환
- **Aggregate 설계**
  - 동일 트랜잭션으로 묶일 객체 범위 정의 -> Cascade/orphanRemoval 기준
  - Aggregate: 도메인 모델에서의 일관성 경계
  - Transaction: DB에서의 일관성 경계
  - 트랜잭션 범위 = Aggregate 범위 로 설계
- **데이터 정합성**
  - 낙관락 재시도/Unique Key/Upsert/멱등 키 등등
- **소프트 삭제**
  - `deleted_at` 컬럼

---

## 3. 참고/추가 자료 (References)

- [Hibernate](https://docs.jboss.org/hibernate/orm/7.1/repositories/html_single/Hibernate_Data_Repositories.html)
- [Spring Data JPA](https://docs.spring.io/spring-data/jpa/docs/current/reference/html/)
- [Hibernate Performance Tuning — Secrets To Lightning-Fast Database Access](https://medium.com/@noel.benji/hibernate-performance-tuning-secrets-to-lightning-fast-database-access-4456124c80b4)
- [Vlad Mihalcea Blog - High-Performance Java Persistence](https://vladmihalcea.com/tutorials/hibernate/)
- [JPA Locking (Baeldung)](https://www.baeldung.com/jpa-optimistic-locking)

---

## 4. 내일/다음에 볼 것 (Next Steps)
- DB 인덱스... 
