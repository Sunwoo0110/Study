# 📚 Spring Entity

---

## 1. 주제/키워드
- Spring의 Entity에 대해 알아보자! ₍^˶ ╸𖥦  ╸˵^₎⟆

---

## 2. 핵심 요약 (Summary)
### Entity란?
- 데이터베이스 테이블과 1:1로 매핑되는 **객체**(클래스)
- JPA에서 Entity는 `@Entity` annotation이 붙은 클래스
  - 각 인스턴스(객체)가 DB의 하나의 row에 해당
  - 필드: column, 객체 = row, 클래스 = table
  - JPA가 entity 객체의 생성/조회/저장/수정/삭제 등을 직접 관리(즉, CRUD를 SQL 없이 객체로 다룸 -> 편리, 직관적)
- JPA(Java Persistence API)란?
  - 자바에서 ORM(Object-Relational Mapping) 기능을 제공하는 공식 표준 인터페이스
  - 불변식/비즈니스 규칙을 내장
  - entity 객체가 규칙을 보장하도록 코드에 명시 -> 매번 검증
    -  ```java
        // 생성자에서 불변식/규칙 보장
        public User(String email, int age) {
            if (age < 0) throw new IllegalArgumentException("나이는 0 이상이어야 합니다.");
            if (!email.contains("@")) throw new IllegalArgumentException("이메일 형식 오류");
            this.email = email;
            this.age = age;
        }
        ```
  - 상태/연관관계 관리
    - 다른 엔티티와의 관계를 코드로 표현(@ManyToOne, @OneToMany 등)
    - ```java
        @ManyToOne(fetch = FetchType.LAZY)
        private Post post; 
        ```
    - 데이터베이스 Join과 같은 복잡한 작업을 코드 수준에서 직관적이고 안전하게 처리
  - JPA 영속성 컨텍스트와 동등성 판단
    - 같은 트랜잭션/세션 내에서는, 같은 PK(Primary Key)를 가진 엔티티는 항상 “동일 객체”로 인식
    - 같은 영속성 컨텍스트 내에서
    - ``UserEntity u1 = em.find(UserEntity.class, 1L);``
    - ``UserEntity u2 = em.find(UserEntity.class, 1L);``
    - u1 == u2는 true (객체 동일성)
    - 즉, id(PK)가 같으면 같은 엔티티로 본다!
    - **따라서 Entity의 equals/hashCode도 id만 기준으로 비교해야함!!**

---

## 🚨 설계 시 주의사항

### 1. Entity에서 equals/hashCode는 id(PK)만 사용!!

#### JPA에서 "동일 엔티티"의 기준은 무엇일까?
- JPA는 **영속성 컨텍스트**(1차 캐시)라는 공간에 엔티티 객체를 보관함
- 여기서 "같은 엔티티"라고 판단하는 기준은 **PK(id)값**
    - 즉, 같은 id라면 "동일한 엔티티"로 취급

#### equals/hashCode에 **id**만 써야 하는 이유
일반 필드(이메일, 닉네임 등)를 기준으로 쓰면?
- 일반 필드는 바뀔 수 있음 (예: 사용자가 닉네임, 이메일을 변경)
- 예를 들어, 엔티티를 Set/Map 등 컬렉션에 넣어둔 뒤 이메일이 바뀌면 hashCode/equals 결과가 달라짐
   - **컬렉션에서 값이 꼬이고(못 찾거나, 중복 허용), 버그 발생!**
   - ```java
        Set<User> set = new HashSet<>();
        User user = new User(1L, "a@a.com"); // equals/hashCode email 기준
        set.add(user);

        user.setEmail("b@b.com"); // 이메일 변경!

        set.contains(user); // false!! (같은 객체인데도 찾지 못함.. ㅜㅜ)
        ```
  - 이유: hashCode가 변해서, 내부적으로 저장 위치가 바뀌어버림
    
연관관계 필드(다른 엔티티 참조)를 기준으로 쓰면?
- Book.posts(List<Post>) ↔ Post.book(Book)처럼 **양방향 연관관계가 있는 경우** equals/hashCode에 양쪽 연관 필드를 모두 넣으면 ``Book -> posts(List<Post>) -> 각 Post -> book(Book) → 다시 posts``
   - **StackOverflowError(무한루프)** 발생!
  
**id(PK)만 기준으로 쓰면?**
 - id는 **엔티티 생성 후 절대 바뀌지 않음**
 - **JPA 영속성 컨텍스트 기준**(동일 id = 동일 엔티티)와 일치
 - 컬렉션에서도 값이 꼬이지 않음 (hashCode/equals가 항상 동일 -> 안전)

#### 구현 방법
- **Lombok**
  - ```java
        @EqualsAndHashCode(onlyExplicitlyIncluded = true)
        public class User {
            @EqualsAndHashCode.Include
            private Long id;
        }
    ```
  - 단점: Hibernate의 프록시 객체 문제(Lazy Loading, 클래스 다름 등)는 신경 못 씀
- **직접 구현 시**
  - id만 비교, 프록시 객체(HibernateProxy) 지원까지 고려
    ```java
    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null) return false;
        // 프록시 객체 고려
        Class<?> oClass = (o instanceof HibernateProxy)
            ? ((HibernateProxy) o).getHibernateLazyInitializer().getPersistentClass()
            : o.getClass();
        Class<?> thisClass = (this instanceof HibernateProxy)
            ? ((HibernateProxy) this).getHibernateLazyInitializer().getPersistentClass()
            : this.getClass();
        if (thisClass != oClass) return false;
        Entity other = (Entity) o;
        return id != null && id.equals(other.id);
    }
    @Override
    public int hashCode() {
        return Objects.hashCode(id);
    }
    ```

- 여기서 잠깐! ദ്ദി´ ▽ ` )
  - (Persistence Context) 영속성 컨텍스트란?
    - JPA가 **엔티티 객체**를 메모리에서 관리하는 1차 캐시(저장소)
    - DB에서 한 번 읽어온(혹은 저장한) 엔티티 객체를 ``트랜잭션`` 동안 JVM 메모리(1차 캐시)에 보관하는 공간
    - **같은 PK(id)로 여러 번 DB 조회해도 항상 동일 객체를 리턴**
    - 엔티티 상태를 추적, 값이 바뀌면 트랜잭션 끝날 때 자동으로 DB에 반영(Dirty Checking)
    - 불필요한 중복 쿼리 방지, 효율적인 성능/동일성/일관성 보장
    - ```java
        // 트랜잭션 시작
        User user1 = em.find(User.class, 1L); // DB에서 조회, 1차 캐시에 저장
        User user2 = em.find(User.class, 1L); // 1차 캐시에서 바로 꺼냄(DB 조회 X)
        System.out.println(user1 == user2); // true! (동일 객체)

        user1.setName("수정함"); // 값만 바꿔도 자동 추적
        // 트랜잭션 끝(커밋) 시점에 update 쿼리 자동 실행됨 (Dirty Checking)
        ```


---

### 2. Hibernate Proxy/지연 로딩, equals/hashCode 오버라이드
- **JPA는 Lazy Loading 구현을 위해 프록시 객체를 리턴함**
  - 근데 리턴값이 실제 클래스와 다를 수 있음 (Book vs Book\$HibernateProxy)
  - 따라서 **equals()에서 getClass()로 비교하면 프록시/실클래스 다르므로 false** -> 이를 방지하기 위해 getClass가 아니라 "실제 클래스"를 비교해야 함
  - **프록시 객체는 HibernateProxy 인터페이스 구현체**
  - 따라서 ``getHibernateLazyInitializer().getPersistentClass()``로 원본 클래스 획득
- **id가 null인 엔티티는 equals 결과 무조건 false**
  - 아직 저장 안된 객체는 영속성 컨텍스트에서 "다른 엔티티"로 취급
- **hashCode는 클래스 기준으로 고정(변경 불가 필드 기반)**
  - 컬렉션 동작이 일관적으로 유지

- 여기서 잠깐! ꜀( ꜆>ᯅ<)꜆
  - **Lazy Loading**(지연 로딩)이란?
    - 엔티티의 **연관관계 필드**(다른 엔티티 참조)를 실제로 "필요할 때까지" DB에서 조회하지 않고, 나중에 접근하는 순간에 “그때 가서” 쿼리를 실행해서 값을 불러오는 방식
    - 즉, 처음부터 모든 연관 데이터를 다 가져오지 않고, 진짜 사용되는 시점에만 추가로 불러옴
    - 따라서 실제로 필요 없는 데이터까지 미리 다 불러오지 않아서, 불필요한 쿼리/메모리 낭비 감소(**성능 최적화**)
    - 대용량 연관관계(예: 한 유저의 수천개 게시글 등)에서 최초 조회 속도 빠름
    - ```java
        Post post = postRepository.findById(1); // post만 조회, user 쿼리 X

        User user = post.getUser(); // 이 시점에서 user 쿼리 실행!
        ```
    - **Lazy Loading은 반드시 트랜잭션 안에서 써야 안전!!**
    - 트랜잭션이 끝난 뒤(영속성 컨텍스트 종료) 지연로딩이 발생하면 LazyInitializationException(지연로딩 실패) 발생
    - 무분별하게 남발하면 N+1 문제 등 성능 이슈 발생할 수 있음
      - 쿼리가 너무 많이 발생함ㅜ
  
  - **Proxy**란?
    - 진짜 객체 대신 동작하는 가짜(대리) 객체
    - JPA에선 연관관계(특히 Lazy Loading)에서 진짜 엔티티 객체를 바로 만들지 않고, **프록시 객체**라는 가짜 껍데기를 먼저 만들어서 반환함
    - 따라서 내부에 실제 데이터는 없음 -> 필드/메서드 접근 시점에만 진짜 데이터 채움
    - ```java
        // post.getUser()는 Lazy 로딩
        User user = post.getUser();
        System.out.println(user.getClass());
        // 출력: class com.example.User$HibernateProxy (프록시 객체!)
        ```
---

### 3. Entity 에서 Setter 사용 최소화
Setter 의 문제점?
- **무결성(Integrity) 보장 실패**
  - 아무 데서나 값 변경 가능하다면 객체가 “비정상적/유효하지 않은 상태”로 바뀔 수 있음
  - 예를 들면, 나이가 음수인 User, 포인트가 -100인 Account 등
  - setter로 값만 바꾸면, 검증 없이 잘못된 값이 저장될 수 있음

- 비즈니스 규칙/불변식 위반
  - 엔티티가 “항상 지켜야 할 규칙”을 못지킴
  - ex) 이메일 형식, username 소문자 변환, status 변경 불가 등
  - setter에는 이런 검증/정규화를 실수하기 쉬움
    - Lombok의 @Setter(혹은 IDE 자동 생성)로 만든 setter는 단순히 값을 대입만 함
    - 별도의 검증/정규화(예: 공백 제거, 소문자 변환, 형식 검사 등)가 자동으로 들어가지 않음
    - 즉, 비즈니스 규칙(검증/정규화)을 넣는 걸 까먹기 쉽고, 실수하기 쉬워짐
  - setter로 인해 비즈니스 규칙을 못 지키면 코드의 일관성/신뢰도가 낮아짐

- 의미 없는/불투명한 상태 변경
  - “누가, 왜, 언제 바꿨는지” 추적 어려움
  - setter는 메서드명만 보고도 의미 파악 어려움(무의미한 값 변경)
  - ex) user.setNickname("홍길동") vs user.changeNickname("홍길동")
  - 그래서 값을 추가하거나 변경하는 경우, 따로 updateEmail, increaseCount 등의 의미가 명확한 메소드를 생성해야함
  - 이는 코드 리팩토링/유지보수성 저하로 이어질 수 있음

- 테스트/디버깅 어려움
  - 예상치 못한 곳에서 값이 변경될 수 있어, 버그 추적이 힘듦
  - setter가 열려 있으면, 어디서든 값이 바뀔 수 있음

- 불변 객체(Immutable) 설계 불가
  - setter가 있으면 “생성 이후 변경 불가” 보장 불가

- 도메인 주도 설계(DDD) 원칙 위배
  - DDD에서는 ``엔티티는 스스로 비즈니스 규칙/상태 변경을 통제``해야 함
  - setter를 열면 외부에서 마음대로 값 변경 가능


---

### 4. @Builder 주의사항
- **@Builder 사용시 모든 필드가 외부로 노출**
  - PK(id), 생성/수정일, 연관관계(다른 엔티티) 등 외부에서 임의로 값 주입 가능
- 따라서 **builder에서는 꼭 필요한 필드만 받기**
  - id, 날짜 등은 builder에 포함 X, 내부에서만 관리
- **정규화/유효성(공백/소문자 변환)은 builder/생성자/세터 모두 방어 코드 필요**
  - Builder는 setter를 안 거치고 필드를 바로 채우므로, 실수 방지하려면 @PrePersist/@PreUpdate까지 써야 함
  - 이게 뭔소리냐면... 
  - 엔티티 객체는 다양한 방식으로 값이 들어올 수 있음
  - ```java
      // Setter
      user.setEmail(" TestUser@Naver.com ");

      // 생성자
      new User(" TestUser@Naver.com ");

      // Builder
      User.builder().email(" TestUser@Naver.com ").build();
      ```
  - 따라서 각각의 경우마다 정규화/유효성 검증이 제대로 적용되어야 함!!
  - 특히 Builder 패턴에는 함정이 있음! Lombok @Builder는 Setter를 거치지 않고, “필드에 직접 값 할당”.. 즉, setEmail()에 정규화를 적용하면 얘가 가진 정규화 로직이 동작하지 않음(글구 setter 자체가 Entity 에 사용하기에 위험성이 큼)
  - ```java
        public void setEmail(String email) {
            this.email = email == null ? null : email.trim().toLowerCase(); // 앞 뒤 공백제거 + 소문자
        }

        // builder는 아래처럼 값이 들어감
        User user = User.builder().email("   TestUser@Naver.com   ").build();
        // → setEmail() 안 거침! 그냥 "   TestUser@Naver.com   " 이 필드에 저장됨
        // 정규화가 누락! 실데이터 오염 위험!!
        ```
  - 따라서 ``@PrePersist / @PreUpdate``로 이중 방어가 필요
    - JPA 라이프사이클 이벤트 메서드로 엔티티가 DB에 저장(@PrePersist), 수정(@PreUpdate)될 때 자동으로 특정 메서드 실행 가능하도록 함
    - ```java
        @PrePersist
        @PreUpdate
        public void normalize() {
            if (this.email != null) this.email = this.email.trim().toLowerCase();
        }
        ```
    - 이렇게 하면 빌더/생성자/세터/어떤 방식으로 값이 들어와도, DB에 저장/수정될 때 반드시 정규화가 적용됨!
    - 따라서 나의 경우는 builder, 생성자, update 등의 메소드에 모두 적용


---

### 5. 데이터 정규화(이메일, username 등)
- **대소문자, 앞뒤 공백 정규화 자주 함(소문자+trim)**
  - "TestUser"와 " testuser "를 같은 계정으로 인식
  - 유니크 제약 & 혼란 방지
- **MySQL 등은 기본적으로 대소문자 민감/비민감 옵션 존재**
  - collation을 `utf8mb4\_bin`(완전일치)로 맞추면 대소문자 구분, `utf8mb4\_general\_ci`면 대소문자 구분 X (사실 MySQL과 같은 애플리케이션에서 일관 정규화가 가장 안전)
- **setter/생성자/빌더에 변환 코드 삽입 + @PrePersist/@PreUpdate로 이중 방어**

---

### 6. 연관관계 설계 (단방향/양방향/ManyToMany)

#### 단방향을 기본으로, 양방향은 진짜 필요할 때만

- 단방향이 훨씬 단순(코드/쿼리/구조/유지보수 용이)
- 양방향은 무한루프(StackOverflow), 순환참조, 직렬화 등 문제
- 만약, Post.user, User.posts 모두 건다고 가정해보자!!
- **무한루프(StackOverflow) 문제**
  - `@ToString`, `@EqualsAndHashCode`, 또는 Jackson 등으로 객체를 문자열/JSON으로 변환할 때
  - Post와 User가 서로를 계속 참조 -> 영원히 반복 -> StackOverflow
  - ```java
    @Entity
    public class User {
        @OneToMany(mappedBy = "user")
        private List<Post> posts = new ArrayList<>();
    }

    @Entity
    public class Post {
        @ManyToOne
        private User user;
    }
    ```

    ```java
    User user = ...; // User.posts → Post.user → User.posts ...
    System.out.println(user.toString()); // 무한루프 시작! StackOverflowError!
    ```
  - user.toString() -> posts(List<Post>) -> 각 post의 toString() → user -> 또 그 user의 posts... -> 반복!

- **순환참조(Circular Reference) 문제**
  - 직렬화(Serialization), Jackson으로 JSON 변환, API 응답 등에서 “A → B → A → B …” 순환구조가 감지됨 (Spring Boot RestAPI에서 자주 터짐)
  - ```java
        @RestController
        public class PostController {
            @GetMapping("/post/{id}")
            public Post getPost(@PathVariable Long id) {
                return postRepository.findById(id).get();
            }
        }
        // Post.user → User.posts → Post.user...
        // JSON 변환 시 무한 순환참조 발생
    ```
  - ``@JsonIgnore``, ``@JsonManagedReference``/``@JsonBackReference ``등으로 일부 필드 직렬화 제외하여 해결

- **직렬화 문제(Serializable 인터페이스 구현 등)**
  - 직렬화(Serialization)란?
    - 객체를 파일, 네트워크, DB, 캐시 등으로 “전송/저장할 수 있는 형태(연속된 바이트)”로 변환하는 과정
    - 반대로, 바이트 데이터를 다시 “원래 객체”로 만드는 과정을 역직렬화(Deserialization) 라고 함
  - 엔티티를 직렬화(예: Redis 저장, 파일 저장, 네트워크 전송 등)하려고 할 때 문제 발생
  - Post → User → Post ...
  - 순환구조로 인해 직렬화/역직렬화 불가 또는 데이터 크기 폭발
  - Redis 캐싱, 세션 저장소 등에 Entity를 저장하려다 무한 순환 참조 발생

- **@EqualsAndHashCode/Set/Map 등 컬렉션 사용 문제**
  - Lombok의 ``@EqualsAndHashCode``, `@Data`, `@ToString` 등에 연관관계 필드를 포함시켰을 때
  - Post와 User가 서로 equals/hashCode, toString 호출 -> 무한루프, StackOverflowError, set/map 동작 불가
  -  ```java
        @Data // Lombok으로 전체 필드 대상 equals/hashCode, toString 생성
        public class User { ... }
        @Data
        public class Post { ... }
        // set.add(post);, set.contains(post); 할 때 무한루프, StackOverflow 발생 가능
        ```

#### ManyToMany는 금지!
- **조인 테이블에 컬럼 추가(상태, 등록일 등) 불가능**
  - 이게 왜냐면, ManyToMany 매핑하면 JPA가 중간 테이블(조인 테이블)을 자동 생성해서 관리
  - 근데 이 조인 테이블은 각 테이블 id인 두 컬럼만 자동으로 만들어서, 추가 컬럼을 넣을 방법이 없음!
- ManyToMany는 JPA가 자동으로 insert/delete만 처리, update 불가
- 관계형 DB에서 관리 어려움, 쿼리/페이징/상태변경 비효율
- 그럼 우뜨케 하라구!
- **중간 엔티티 직접 선언하기**
  - User <-> UserRole <-> Role 구조
  - 내가 원하는 칼럼 추가 가능!

---

### 7. 컬렉션 필드 & 무한루프 주의
- @ToString, @EqualsAndHashCode, @Data 등 Lombok 어노테이션 주의!
  - 연관관계(컬렉션, 엔티티 참조) 필드 포함하면 무한루프 위험
  - 반드시 exclude 처리(@ToString.Exclude 등)

---

### 8. 평점/통계 필드와 정합성 문제
- **평균/개수 등 집계 필드는 “실시간”으로 update가 필요**
  - BookRating 생성/수정/삭제 시 Book의 ratingCount, ratingAvg update하기
- 근데 동시성/트랜잭션 꼬이면 실제 데이터와 집계 필드 불일치 가능
  - **트랜잭션 내에서 집계값을 aggregate 쿼리로 갱신 & update**
  - 평점이 바뀔 때마다, BookRating 테이블에서 해당 Book의 모든 평점을 select해서 평균/개수 aggregate 쿼리로 계산 -> Book의 ratingAvg, ratingCount 필드에 update
- DB 트리거나 이벤트로 처리 권장 X
  - JPA, 서비스 계층에서 통제/트랜잭션 보장하는 게 더 안전하다고 함
  - DB 트리거는 JPA, 서비스 계층 트랜잭션과 분리되어 트랜잭션/에러 처리/동시성 관리가 꼬일 수도 있다고 함..

---

### 9. UserDetails 구현과 인증객체
- **Spring Security는 인증된 사용자 정보를 UserDetails 구현체로 관리**
  - Entity가 UserDetails를 구현하면(상속 받으면) 바로 인증/인가에 사용 가능
  - JWT 등 커스텀 인증도 UserEntity에서 바로 접근 가능
  - **getAuthorities, getPassword, getUsername 등 필수 오버라이드**
  - isEnabled는 soft delete(삭제 여부) 등으로 컨트롤

---

### 10. 상태 변경/비즈니스 로직 메서드만 공개 (setter 남발 금지)
- **엔티티의 상태는 도메인 메서드(명확한 이름)로만 변경**
  - ex) updateReview(), changeNickname(), softDelete()
  - 무의미한 setter 남발하면 데이터 무결성, 유지보수성 저하
- 상태 변화 시, 도메인 규칙(유효성, 권한 등) 함께 캡슐화
  - ```java
    user.setAge(-10); // 무결성 깨짐

    // 따라서 항상 규칙 검증 필요       
    public void changeAge(int age) {
        if (age < 0) throw new IllegalArgumentException();
        this.age = age;
    }
    ```

---

### 11. @Transactional 어디에 붙일까?
- **메서드 단위**
  - 변경/저장/삭제(쓰기)에만 붙이고, 조회는 @Transactional(readOnly = true)
  - 클래스 전체에 붙이면 public 메서드 전부 트랜잭션, 오버헤드, 혼용 위험

---

### 12. 복합 unique 제약 걸기
- LikeEntity에서 user, targetType, targetId 복합 unique
  - 같은 유저가 같은 글/댓글에 여러 번 좋아요 불가하도록
- @Table(uniqueConstraints = ...)으로 관리

---

### 13. 필드/상태 관리 원칙
- 모든 필드는 private/protected + @Getter
  - 무결성, 외부 직접 변경 금지
  - **setter는 진짜 필요한 값만, 상태 변화 없는 객체는 불변(fianl+private+생성자 할당+setter X)**
- **생성자(특히 JPA 기본 생성자)는 protected로 제한**
  - 외부에서 new UserEntity() 호출 금지, 프레임워크만 객체 생성 허용

---

### 14. AllArgsConstructor 사용하지 않기
- 모든 필드를 파라미터로 받는 “전체 필드 생성자”를 자동으로 만들어줌
- 모든 필드를 "순서대로" 세팅해야 해서, 필드 추가/삭제/순서 변경 시 객체 생성 코드가 꼬일 수 있음
- id, createdAt, updatedAt 등은 외부에서 세팅되면 안 되는 필드인데, AllArgsConstructor는 이 필드까지 파라미터로 받아버림
- 대신 **Builder** 사용하기

---

### 15. DTO와 엔티티의 역할 분리
- 엔티티는 오직 DB 매핑용, 비즈니스/프레젠테이션 계층에는 별도 DTO 사용
  - 엔티티와 DTO 혼용(엔티티를 서비스/컨트롤러/응답에 그대로 노출)은 버그, 보안, 데이터 오염 가능성
  - DTO를 활용하면 검증, 필드 변환, 응답 필드 제한, 계층 분리, 유지보수성 모두 보장

---

### 16.  API 응답에 엔티티 직접 노출 금지
- API/Controller 응답에 엔티티를 그대로 반환하면, 연관관계 무한루프, Lazy Loading, 민감정보 유출 등 문제
  - 반드시 필요한 필드만 추린 DTO/Response 객체로 변환해서 반환
  - Jackson 등 직렬화 이슈/연관관계 주입 문제 예방
  - 실제로 Spring 첫 개발했을 때 entity 냅다 반환해서ㅠ 무한루프 걸렸음..

---

### 아직 적용안한 항목들...(◞‸◟；)

### 17. Auditing 자동화
- 엔티티의 생성일, 수정일, 삭제일(soft delete) 등은 JPA Auditing 기능 적극 활용
- @CreatedDate, @LastModifiedDate, @EntityListeners(AuditingEntityListener.class)로 구현
- 날짜 관련 코드 반복 없이, 모든 엔티티에 공통 베이스 엔티티로 자동화
- 삭제시에도 "진짜 delete" 대신 isDeleted(soft delete) + deletedAt 필드로 관리하면 복구, 로깅, 감사 추적 가능

### 18. 불변 객체(Immutable Entity) 원칙
- 상태 변화가 없는 필드는 ``final``로 선언, 생성자에서만 값 할당
  - setter 최소화, 가능하면 아예 금지
  - 사이드이펙트 및 예상치 못한 값으로의 변경 방지
  - 특히 엔티티 내 변경 불가 값(이메일, 가입일 등)에 적용

### 19. Test Fixture(엔티티 생성 헬퍼) 분리
- 테스트 코드에서 @Builder/생성자 직접 호출로 엔티티 생성 남발 금지
  - 실제 프로덕션 코드와 동일한 정규화/유효성 검증, 필수 값 세팅 등 일관성 깨질 수 있음
  - 별도의 Fixture/Factory/Helper 클래스로 표준화된 엔티티 생성 메서드를 제공
  - 테스트 코드와 실 서비스 코드가 "동일한 생성, 검증 방식"을 따르게 설계

---

## 3. 예제 코드/활용법 (Example)
```java
@Entity
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
@Table(name = "post")
public class PostEntity {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(length = 128, nullable = false)
    private String title;

    @Column(columnDefinition = "text", nullable = false)
    private String content;

    @Enumerated(EnumType.STRING)
    @Column(length = 32, nullable = false)
    private PostCategory category; // review/recommend/debate/etc

    @Column(name = "view_count", nullable = false)
    private int viewCount = 0;

    @Column(name = "like_count", nullable = false)
    private int likeCount = 0;

    @Column(name = "comment_count", nullable = false)
    private int commentCount = 0;

    @CreationTimestamp
    @Column(name = "created_at", updatable = false)
    private LocalDateTime createdAt;

    @UpdateTimestamp
    @Column(name = "updated_at")
    private LocalDateTime updatedAt;

    @Column(name = "is_deleted", nullable = false)
    private Boolean isDeleted = false;

    @ManyToOne(fetch = FetchType.LAZY, optional = false)
    @JoinColumn(name = "user_id", nullable = false)
    private UserEntity user;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "book_id")
    private BookEntity book;

    public enum PostCategory {
        REVIEW, RECOMMEND, DEBATE, ETC
    }

    private static PostCategory parseCategory(String category) {
        if (category == null || category.trim().isEmpty()) return PostCategory.ETC;
        try {
            return PostCategory.valueOf(category.trim().toUpperCase());
        } catch (IllegalArgumentException e) {
            return PostCategory.ETC;
        }
    }

    public void softDelete() { this.isDeleted = true; }

    public void increaseViewCount() { this.viewCount++; }
    public void increaseLikeCount() { this.likeCount++; }
    public void decreaseLikeCount() { if (this.likeCount > 0) this.likeCount--; }
    public void increaseCommentCount() { this.commentCount++; }
    public void decreaseCommentCount() { if (this.commentCount > 0) this.commentCount--; }

    private static String normalizeString(String str) {
        return (str == null || str.trim().isEmpty()) ? null : str.trim();
    }

    @PrePersist
    @PreUpdate
    private void normalize() {
        this.title = normalizeString(title);
        this.content = normalizeString(content);
    }

    @Builder
    public PostEntity(String title, String content, String category, UserEntity user, BookEntity book) {
        this.title = normalizeString(title);
        this.category = parseCategory(category);
        this.content = normalizeString(content);
        this.user = user;
        this.book = book;
        // 나머지 필드는 JPA가 관리
    }

    // equals/hashCode (id만)
    @Override
    public final boolean equals(Object o) {
        if (this == o) return true;
        if (o == null) return false;
        Class<?> oEffectiveClass = o instanceof org.hibernate.proxy.HibernateProxy
                ? ((org.hibernate.proxy.HibernateProxy) o).getHibernateLazyInitializer().getPersistentClass()
                : o.getClass();
        Class<?> thisEffectiveClass = this instanceof org.hibernate.proxy.HibernateProxy
                ? ((org.hibernate.proxy.HibernateProxy) this).getHibernateLazyInitializer().getPersistentClass()
                : this.getClass();
        if (thisEffectiveClass != oEffectiveClass) return false;
        PostEntity that = (PostEntity) o;
        return getId() != null && getId().equals(that.getId());
    }
    @Override
    public final int hashCode() {
        return this instanceof org.hibernate.proxy.HibernateProxy
                ? ((org.hibernate.proxy.HibernateProxy) this).getHibernateLazyInitializer().getPersistentClass().hashCode()
                : getClass().hashCode();
    }
}
```

## 4. 실전 적용/느낀 점 (Usage & Insights)
- 공부하다가 울뻔.... ｡°꒰ ՞ ´ ᗣ`°꒱°｡
- 그냥.. 여태 내가 얼마나 고민없이 설계하고 개발했는지 느낄 수 있는 시간이였삼...

---

## 5. 참고/추가 자료 (References)
- 코드 작성하고 문제 상황 고민하고 GPT 한테 물어봄

---

## 6. 내일/다음에 볼 것 (Next Steps)
- JPA / Persistence context(영속성 컨택스트)