# 🛠️ NPE 발생: `ratingAvg` 언박싱 오류 (BookEntity → BookListResDto 변환 중)

---

## 1. 문제 상황 (What happened?)
- 도서 목록 조회(`listBooks`) 시 `BookEntity`의 double `ratingAvg` 필드를 DTO의 `double` 타입으로 언박싱하다가 `NullPointerException` 발생

---

## 2. 에러 메시지/현상 (Error/Log)

```text
java.lang.NullPointerException: Cannot invoke "java.lang.Double.doubleValue()" 
because the return value of "com.sunwoo.bookdam.domain.book.entity.BookEntity.getRatingAvg()" is null
    at com.sunwoo.bookdam.domain.book.mapper.BookMapper.toListDto(BookMapper.java:27)
    at …
    at com.sunwoo.bookdam.domain.book.service.BookServiceImpl.listBooks(BookServiceImpl.java:33)
```

---

## 3. 원인 분석 (Why?)

- 엔티티 필드에 `private double ratingAvg = 0.0;`로 초기값을 주었지만,
  JPA가 DB에서 로드할 때는 **기존 테이블의 NULL** 값을 그대로 덮어써서,
- `rating_avg` 컬럼에 NULL이 남아 있으면 `getRatingAvg()`가 `null`을 반환하고,
- primitive `double` 언박싱(`doubleValue()`) 시 NPE 발생

---

## 4. 시도해 본 해결 방법 (Tried)

- [x] DTO 필드를 `double` -> `Double`로 변경 -> 직렬화는 되나 클라이언트에 `null` 노출
- [x] MapStruct 매핑에 `constant`으로 기본 값 선언 -> 동작 확인
- [x] 엔티티에 `@ColumnDefault("0")` 추가 → DDL 자동 생성 시 기본값 설정 시도
- [x] 테이블 싹 다 드롭하고 다시 생성

---

## 5. 최종 해결 방법 (How fixed)

1. 기존 NULL 데이터 정리 & 컬럼 제약 변경(혹은 테이블 드롭 후 재생성)

   ```sql
   UPDATE book SET rating_avg = 0 WHERE rating_avg IS NULL;

   ALTER TABLE book
     ALTER COLUMN rating_avg SET DEFAULT 0,
     ALTER COLUMN rating_avg SET NOT NULL;
   ```
2. 엔티티 설정 보강

   ```java
   @Entity
   @DynamicInsert
   public class BookEntity {
       @Column(name="rating_avg", nullable=false)
       @ColumnDefault("0")
       private double ratingAvg;
       // …
   }
   ```
3. MapStruct 매핑 안전망

   ```java
   @Mapping(target = "ratingAvg", constant = "0.0")
   @Mapping(target = "ratingCount", constant = "0")
   @Mapping(target = "postCount", constant = "0")
   BookEntity toEntity(BookCreateReqDto dto);
   ```

---

## 6. 결과 및 느낀점 (Result & Learnings)

- **결과**: `ratingAvg` null이 사라지고, 모든 신규/기존 레코드가 0.0으로 정상 조회
- **배운 점**:
  - 자바 필드 초기값과 무관하게 JPA 로드 시 덮어쓰는 일이 발생
    - 자바 객체 생성 시점(`new BookEntity()` 호출하거나 Lombok `@Builder`가 내부적으로 생성자를 호출)에는 Entity에 선언한 필드 초기값(`private double ratingAvg = 0.0;`)이 적용됨
    - 근데 **JPA 영속화 컨텍스트에 등록 시점**(`EntityManager.find()`   혹은 리포지토리의 find)때는 Hibernate가 JDBC `ResultSet`에서 컬럼값을 꺼내와서 **모든 필드를 다시 채워넣음**
    - 이때 DB 컬럼에 `NULL`이 있으면 자바 필드가 초기값이든 아니든 `null` 또는 해당 JDBC 타입 그대로 덮어쓰게 됨
    - 따라서 새로 만든 엔티티(`new` 또는 빌더)는 초기값 `0.0` 그대로 유지하지만 DB에서 꺼낸 엔티티, 즉 DB에 이미 저장된 값(또는 `NULL`)으로 무조건 덮어씀 -> 기존 데이터에 `rating_avg NULL`이 남아 있으면, 자바 필드 초기값이 무시되고 `null`이 세팅
  - `@DynamicInsert`와 DB 기본값 설정이 신규 INSERT에만 적용됨을 주의(기존 값은 변동 없음)

---

## 7. 참고 자료 (References)
- [티스토리](https://eocoding.tistory.com/71)
- [baeldung](https://www.baeldung.com/jpa-default-column-values)

---

## 8. 관련 Issue/PR/Commit (Optional)
- Commit: [fix: NPE bug - add DynamicInsert in entity and constant map in mapper](https://github.com/Sunwoo0110/Bookdam-backend/commit/3a6d66bb42743c6a01351449cb39693d72a01a4c)
