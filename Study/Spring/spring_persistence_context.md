# 📚 Spring Persistence Context (영속성 컨텍스트)

---

## 1. 주제/키워드
- JPA에서 **영속성 컨텍스트**(Persistence Context) 개념과 역할에 대해 공부하자~ (´；д；)

---

## 2. 핵심 요약 (Summary)

### Persistence Context (영속성 컨텍스트)
- **엔티티를 저장하고 관리하는 1차 캐시 같은 공간**
- JPA에서 `EntityManager`가 DB랑 직접 통신하지 않고, 먼저 **영속성 컨텍스트에 엔티티를 올려서 관리**하다가 트랜잭션 시점(Commit/Flush)에 DB에 반영함
- 즉, **엔티티의 상태를 추적하고, DB 반영을 최적화하는 중간 계층**
- 여기서 잠깐!
- JPA 란?
  - 자바 표준 ORM (Object Relational Mapping) 인터페이스
    - ORM: 객체(클래스)와 관계형 DB의 테이블을 매핑해서 SQL을 직접 쓰지 않고 데이터베이스를 다루게 해주는 기술
  - JPA 자체는 구현체가 없고, Hibernate, EclipseLink 같은 구현체가 실제로 동작
  - 개발자는 persist(), find(), remove() 같은 메서드로 엔티티를 다루면, JPA가 알아서 SQL을 생성하고 실행! -> 편리
  - @Entity, @Id, @OneToMany 같은 어노테이션을 통해 매핑 설정
  - Spring Data JPA -> Repository 인터페이스만 정의하면 CRUD가 자동 제공
  - 장점: 반복적인 SQL 작성이 줄어들고, 객체 지향적인 코드로 DB를 다룰 수 있음
  - 단점: 복잡한 SQL 튜닝이나 DB 특정 기능 활용이 어려움
    - 해결
      - Spring Data JPA + Querydsl 이용: 동적/복잡 쿼리 작성 편리
      - DB 특정 기능에 대해서는 네이티브 쿼리 작성

---

### 엔티티의 4가지 상태
1. **비영속** (new / transient)
   - 아직 컨텍스트에 올라가지 않은 상태 (`new`로 만든 객체)
   ```java
   Member m = new Member("Yushi"); // 비영속
   ```

2. **영속** (managed)
   - `persist()`로 컨텍스트에 저장된 상태
   - 이제부터 JPA가 상태 변화를 추적 (Dirty Checking)
    ```java
    em.persist(m); // 영속 상태
    ```

3. **준영속** (detached)
   - 컨텍스트에서 분리된 상태 (`em.detach()` / `em.clear()` 등)
   - 변경해도 DB에 반영되지 않음

4. **삭제** (removed)
   - 컨텍스트에서 삭제된 상태 (`em.remove()` 호출)

---

### 영속성 컨텍스트 역할

1. **1차 캐시**
   - 같은 엔티티를 여러 번 조회하면 DB에서 다시 안 물어보고 **캐시에서 꺼내줌** -> 성능 최적화

2. **동일성 보장** (Identity Guarantee)
   - 같은 트랜잭션 안에서 같은 PK를 가진 엔티티는 **항상 같은 인스턴스** 반환
    ```java
    Member m1 = em.find(Member.class, 1L);
    Member m2 = em.find(Member.class, 1L);
    System.out.println(m1 == m2); // true
    ```

3. **쓰기 지연** (Write-behind)
   - `persist()` 호출 시 바로 INSERT 쿼리 날리지 않고, **SQL 저장소에 쌓아뒀다가** 트랜잭션 커밋 시 한꺼번에 DB 반영

4. **Dirty Checking** (변경 감지)
   - 엔티티 값만 바꿔도, 스냅샷과 비교해서 **UPDATE 쿼리 자동 생성**
   - 개발자가 `save()` 같은 거 안 불러도 됨

5. **Flush** (동기화)
   - 영속성 컨텍스트 값과 DB를 맞추는 시점
   - 보통 트랜잭션 **commit 직전**에 flush가 자동 호출됨

---

### 영속성 컨텍스트와 @Transactional
- `@Transactional`이 붙은 메서드 실행 시
  - 트랜잭션 시작 -> 영속성 컨텍스트도 같이 시작
  - 엔티티 변경사항 추적
  - commit 시점에 flush -> DB에 반영
- 즉, **트랜잭션 단위 = 영속성 컨텍스트 단위**

---

## 3. 코드 예시

```java
@Transactional
public void updateUser(Long id) {
    Member member = em.find(Member.class, id); // 1차 캐시 or DB 조회
    member.setName("새로운 이름"); // Dirty Checking으로 변경 감지
} 
// 메서드 종료 시점 = commit -> flush ->  UPDATE 쿼리 실행
```
- `setName()`만 해도 자동으로 UPDATE 쿼리를 날림!! (Dirty Checking)

---

## 4. 참고/추가 자료 (References)

- [Spring Docs - JPA](https://docs.spring.io/spring-framework/reference/data-access/orm/jpa.html)
- [Hibernate Reference - Persistence Context](https://docs.jboss.org/hibernate/orm/5.4/userguide/html_single/Hibernate_User_Guide.html#pc-overview)

---

## 5. 내일/다음에 볼 것 (Next Steps)
- 스프링 트랜잭션 파보기..... 옛날에 공부 했었는데 기억이 안난다ㅜㅠ