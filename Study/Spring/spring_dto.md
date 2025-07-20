# 📚 Spring DTO

---

## 1. 주제/키워드
- Spring에서 DTO에 대해 알아보자~~ (๑و•̀Δ•́)و

---

## 2. 핵심 요약 (Summary)
### DTO(Data Transfer Object)란?
- 계층 간 또는 시스템 간 **데이터 전송**을 위해 사용하는 객체
- DTO는 순수해야한다!
  - DTO는 데이터 구조만 가지고, **비즈니스 로직이나 변환 및 검증 로직을 포함하지 않음**
- **유지보수성**: 변경 시 연관되는 영향이 적어 안정적으로 수정 가능
- **테스트 용이**: 로직이 없으니 DTO 단위로 직관적이고 간단한 단위 테스트 작성 가능
- **재사용성**: 다양한 컨텍스트(Request, Response, 매핑)에서 같은 DTO를 안전하게 재사용
- **결합도 감소**: Entity나 서비스 로직과의 의존성을 줄여, 서로 독립적으로 발전, 배포 가능
- 왜 사용할까?
  - **의존성 분리**: Entity와 API 스펙(요청, 응답 구조)을 분리
    - 내부 도메인 모델(Entity) 노출 방지
    - API 문서와 실제 코드가 일치하여 유지보수 편리
  - **네트워크/페이로드 최적화**: 필요한 필드만 포함해 전송량 감소
  - **보안 강화**: 민감 정보(비밀번호, 내부 상태 등) 노출 차단
- 종류
  - **Request DTO**: 클라이언트 요청 데이터를 Controller -> Service로 전달
  - **Response DTO**: Service 결과를 Controller/클라이언트로 응답

----

### Request DTO vs Response DTO
- **Request DTO**은 `Mutable`하게 설계
  - **필드 변경 허용**
  - Jackson 바인딩을 위해 `@NoArgsConstructor` + `@Setter` 사용 (`@Data` 포함 가능)
    - 다만 `@Data` 하나는 모든 Getter/Setter/equals/toString 생성하기 때문에, 의도치 않은 Setter 허용, equals/hashCode 전 필드 반영 위험이 있음
    - 따라서 필요에 따라 여러 어노테이션을 조합해서 사용하는 것이 안전(`@Getter`+`@Setter`+`@NoArgsConstructor`, `@Getter`+`@AllArgsConstructor`+`@Builder`)
  - 경우에 따라 Builder 바인딩(`@ConstructorBinding` or Lombok `@Builder`)으로 **불변 Request** 생성도 선택 가능
  - Validation 애노테이션(`@NotNull`, `@NotBlank`)으로 필수 필드 표시 및 API 문서화
  - 여기서 잠깐! Jackon이란?
    - 자바 객체와 JSON 간(또는 XML 등 다른 포맷도 지원)을 직렬화/역직렬화해 주는 라이브러리
    - 직렬화(Serialization): 자바 객체 -> JSON 문자열
    - 역직렬화(Deserialization): JSON 문자열 -> 자바 객체
  - 나의 경우에는 `@Getter @Setter @NoArgsConstructor(access = AccessLevel.PROTECTED) @AllArgsConstructor` 조합으로 사용

- **Response DTO**은 `Immutable`하게 설계
  - **처음 생성 후 내부 상태 변경 금지**
  - 모든 필드를 `private final`으로 선언, Setter 제거
  - Lombok `@Getter`, `@AllArgsConstructor(access = AccessLevel.PRIVATE)` + `@Builder` 조합으로 Builder를 통한 명시적 생성 강제
  - 혹은 간편하게 `@Value` 로 선언 가능
  - Java 16+ 프로젝트에서는 `record`로 간결하게 불변 DTO 정의 가능
  - `@JsonInclude(JsonInclude.Include.NON_NULL)` 사용해 `null` 필드를 응답 JSON에서 제외하여 안전하고 가벼운 payload 생성
  - 나의 경우에는 `@Getter @AllArgsConstructor(access = AccessLevel.PRIVATE) @Builder @JsonInclude(JsonInclude.Include.NON_NULL)` 조합으로 사용

- 여기서 잠깐!! (・Д・)ノ 

- 그러면 왜 **Response DTO는 Immutable** 해야 하나?
  1. 사이드 이펙트 방지: 중간 처리(필터나 AOP 등) 과정에서 값이 바뀌지 않아야함
  2. 스레드 안전성: 멀티스레드 환경에서 동기화 없이 공유 가능(값이 바뀌지 않아 안전하게 공유 가능)
  3. 캐싱 안정성: 캐시된 응답이 변하지 않으므로 일관된 결과 보장
  4. 디버깅, 테스트 용이: 불변 객체는 추적 및 검증이 쉬움

- 그러면 반대로 왜 **Request DTO는 Mutable** 로 설계하는 경우가 많을까?
  1. Jackson 바인딩 편의: 기본 생성자와 Setter가 필요
       - Spring MVC는 JSON을 객체로 바꿀 때 기본 생성자로 빈 DTO 인스턴스를 만들고 setXxx() 메서드로 하나씩 필드를 채움
  2. **Valid 검증**: DTO가 mutable해야 빈 DTO -> Setter로 값 채움 -> 검증 순서가 이루어짐. 즉, Setter가 없으면 검증 단계로 넘어갈 수도 없음. (Request DTO에는 검증을 위한 `@NotNull`, `@Size`, `@Pattern` 등의 annotation을 사용)
  3. 유연성 및 단계적 바인딩: API 요청에서 여러 소스(path 변수, 쿼리 파라미터, 바디 등)에서 오는 경우, 차례대로 바인딩 되는 경우가 있음. 이 때, Mutable 해야 컨트롤러가 호출될 때 프레임워크가 알아서 바인딩해줌(setXxx 를 여러 번 각각 호출)

* 어노테이션 정리

  | 목적                 | 어노테이션 조합                                                                            |
  | ------------------ | ----------------------------------------------------------------------------------- |
  | Mutable Request    | `@Getter`, `@Setter`, `@NoArgsConstructor`, (`@Data`)                               |
  | Immutable Request  | `@ConstructorBinding`, `@Builder`, final 필드, Setter 제거                              |
  | Immutable Response | `@Getter`, `@AllArgsConstructor(access = PRIVATE)`, `@Builder`, final 필드, Setter 제거 |

---

### 정규화 로직
- API로 받은 데이터가 바로 엔티티로 들어가기 전에 **일관성**과 **무결성**을 보장해 주기 위해서 사용
- ex. 공백 제거(trim), 소문자 변환(lowercase), 날짜 형식 파싱
- MapStruct: 

**MapStruct 매퍼 방식**
- 컴파일 타임에 필드 매핑 오류를 잡음(타입, 정규화 등)
- 매핑 코드와 정규화 코드를 한곳에 모아두어 관리가 편함
- DTO와 Entity가 느슨한 결합(Loosely Coupled)
  - DTO는 단순 데이터 구조만 가지고, 변환 로직은 매퍼 인터페이스 안에 둠
  - 따라서 변경이 있을 때 Entity 구조만 바뀌면 매퍼만 고치면 되고, DTO는 그대로 유지할 수 있어 유지보수가 쉬움!!
- 다만 작은 DTO 하나 처리하는데도 매퍼 클래스가 필요해 오버헤드 발생 가능
```java
@Mapper(componentModel = "spring")
public interface UserMapper {
    @Mapping(target="username", expression="java(dto.getUsername().trim().toLowerCase())")
    @Mapping(target="signupDate", dateFormat="yyyy-MM-dd")
    User toEntity(UserCreateDto dto);
}
```
**DTO `toEntity()` 내부 방식**
- DTO 안에 바로 변환 로직을 넣어 간단히 사용 가능
- DTO와 Entity가 강하게 결합
  - DTO 클래스 안에 엔티티 생성(변환) 로직이 들어 있어서 두 객체가 서로 깊이 의존
  - 따라서 DTO가 Entity 구조를 알아야 하고, Entity가 수정되면 DTO의 toEntity도 수정되어야함 -> 유지보수 부담 증가
- 여러 DTO마다 같은 코드가 중복될 수 있음

```java
public class UserCreateDto {
    private String username;
    private String signupDate;

    public User toEntity() {
        return User.builder()
            .username(username.trim().toLowerCase())
            .signupDate(LocalDate.parse(signupDate))
            .build();
    }
}
```

---

### Builder vs Setter

| 항목            | Setter 방식                                            | Builder 방식                                                                                  |
| ------------- | ---------------------------------------------------- | ------------------------------------------------------------------------------------------- |
| **초기화 명시성**   | 여러 `setXxx()` 호출로 인해 어느 필드가 빠졌는지 확인하기 어려워 실수 발생 가능 | `.field(value)` 체이닝으로 어떤 필드를 설정했는지 한눈에 파악, 필수 필드 누락 시 `Objects.requireNonNull` 등으로 오류 조기 발견 |
| **불변성 보장**    | 객체 생성 후에도 언제든 `setXxx()`로 값 변경 가능 -> 상태 추적 어려움        | `build()` 호출 후 모든 필드가 `final`로 고정되어 변경 불가 -> 상태 일관성, 안정성 확보                                   |
| **가독성**       | 설정 코드가 분산되어 있어 전체 초기화 흐름을 파악하기 어려움                   | 빌더 호출 한 줄로 객체 생성, 초기화 과정을 명확하게 보여줘 유지보수성 향상                                            |
| **테스트 안정성**   | 테스트 코드에서 잘못된 순서나 누락으로 초기화 오류 발생 가능                   | 빌더 내 기본값 설정, 필수 필드 검증 로직 활용으로 일관된 테스트 데이터 생성 -> 재사용성 향상                                         |
| **유연성**       | 런타임에 동적으로 필드를 변경해야 할 때 유리                            | 객체가 불변이므로, 변경할 때마다 Builder로 다시 구성해야 함 -> 유연성 낮음                              |
| **로직 분리**     | DTO 자체에 필드 설정 로직이 섞여 복잡해질 수 있음                       | Builder 클래스로 생성 로직 분리 가능 -> DTO 클래스는 순수 데이터 모델로 유지                                           |

---

### 기본값 처리
- **DTO Getter 오버라이드**
  - 기본값 로직을 DTO 안에 모아서 관리 -> 서비스 코드에서는 dto.getPage()만 호출하기 때문에 깔끔
  - DTO에 비즈니스 로직이 들어감 -> “DTO는 순수 데이터”라는 원칙에서 벗어날 수 있음
  - ```java
      public class PageRequestDto {
          private Integer page;
          private Integer size;

          public int getPage() {
              return (page != null && page > 0) ? page : 1;
          }

          public int getSize() {
              return (size != null && size > 0) ? size : 20;
          }
      }
      ```


- **서비스 내 Null 체크**
  - DTO는 순수하게 바인딩된 값만 보유
  - 서비스마다 반복적인 orElse() 코드가 늘어남 -> 실수로 null 체크를 빼먹기 쉬움
  - ```java
      int page = Optional.ofNullable(dto.getPage()).filter(p -> p > 0).orElse(1);
      int size = Optional.ofNullable(dto.getSize()).filter(s -> s > 0).orElse(20);
      ```


- 장단점이 명확해서 보통 팀 스타일에 맞춰 작성한다고 함
- ~~나의 경우에는 **DTO Getter 오버라이드** 로 작성 (서비스에서 까먹고 빼먹을거 같아서..)~~
- DTO 설계 원칙 지키려고 **서비스 내 Null 체크**로 변경

--- 

### 자주 일어나는 실수
1. 매번 새로운 DTO를 용도별로 만들기
   - 모든 상황마다 별도의 DTO를 만들면 클래스와 매퍼 수가 기하급수로 늘어나 관리 비용이 증가

2. 하나의 DTO를 모두에게 다 쓰려고 하기
   - 단일 클래스로 너무 많은 시나리오를 처리하려 하면, 사용하지 않는 속성이 잔뜩 포함된 거대한 계약(contract)이 만들어짐

3. DTO에 비즈니스 로직을 넣기
   - DTO는 데이터 전송과 계약 구조를 최적화하기 위한 패턴이므로, 모든 비즈니스 로직은 도메인(서비스) 레이어에만 두어야 함

4. 로컬 DTO(LocalDTO)를 남발하기
   - 도메인 간에 DTO를 계속 전달하기 위해 LocalDTO를 쓰면, 역시 매핑과 유지보수 비용이 증가

---

## 3. 예제 코드/활용법 (Example)
```java
/**
 * 책 검색 요청 DTO
 */
@Getter
@Setter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
@AllArgsConstructor
public class BookSearchReqDto {

    @Schema(
            description  = "책 제목(검색, optional)",
            example      = "해리포터",
            requiredMode = RequiredMode.NOT_REQUIRED
    )
    private String title;

    @Schema(
            description  = "저자명(검색, optional)",
            example      = "J.K.롤링",
            requiredMode = RequiredMode.NOT_REQUIRED
    )
    private String author;

    @Schema(
            description  = "정렬 기준 (예: ratingAvg,desc 또는 createdAt,asc)",
            example      = "ratingAvg,desc",
            requiredMode = RequiredMode.NOT_REQUIRED
    )
    @Pattern(
            regexp  = "^[a-zA-Z0-9]+(Avg)?,(asc|desc)$",
            message = "sort는 '필드명,asc' 또는 '필드명,desc' 형식이어야 합니다."
    )
    private String sort;


    @Schema(
            description  = "페이지 번호 (1부터 시작, 기본=1)",
            example      = "1",
            requiredMode = RequiredMode.NOT_REQUIRED
    )
    @Min(1)
    private Integer page;

    @Schema(
            description  = "페이지 크기 (기본=20, 최대=100)",
            example      = "20",
            requiredMode = RequiredMode.NOT_REQUIRED
    )
    @Min(1) @Max(100)
    private Integer size;

    // 기본값 보장
    public int getPage() {
        return page == null ? 1 : page;
    }

    public int getSize() {
        return size == null ? 20 : size;
    }

    // 입력 정규화
    public void setTitle(String title) {
        this.title = (title == null ? null : title.trim());
    }

    public void setAuthor(String author) {
        this.author = (author == null ? null : author.trim());
    }
}
```

---

## 4. 실전 적용/느낀 점 (Usage & Insights)
- [연관 이슈]()
- [연관 PR]()
- 소프트웨어공학개론에서 배웠던 재사용성, 유지보수, 테스트용이성, 안정성 이런 부분이 실제 설계에서 얼마나 중요한 지 알게 되었다... 나름대로 정말 사소한 부분 하나 하나 많이 고민해서 설계해봤는데.. 너무 힘들다!! 그리고 너무 오래 걸린다!! 하하

---

## 5. 참고/추가 자료 (References)
- [baeldung](https://www.baeldung.com/java-dto-pattern)
- [마틴 파울러](https://martinfowler.com/eaaCatalog/dataTransferObject.html)
- 수많은 티스토리 & velog.... 그리고 GPT

---

## 6. 내일/다음에 볼 것 (Next Steps)
- 더이상 미룰 수 없다.. MVC 패턴

