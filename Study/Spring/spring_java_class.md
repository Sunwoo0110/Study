# 📚 Java Class
---

## 1. 주제/키워드
- 자바에서 Class란 무엇인가.. (~ ᵕ ̫ ᵕ)~

---

## 2. 핵심 요약 (Summary)
### 객체 지향 프로그래밍(OOP: Object Oriented Programming)
- 객체(Object)
  - 자신이 속성을 가지고 있고 다른 것과 식별 가능한 것
  - 집합 관계 / 사용 관계 / 상속 관계
- OOP 특성
  - **캡슐화(Encapsulation)**: 객체를 캡슐화하여 실제 구현 내용을 감추는 것
    - 외부의 잘못된 사용으로 객체가 손상되는 것을 방지(무결성)
    - 중요한 비즈니스 규칙(예: “나이는 0 이상이어야 한다”)은 Setter나 생성자 등에서 유효성 검증 코드 꼭 작성
  - **상속(Inheritance)**: 상위 객체가 하위 객체에게 필드와 메소드를 물려주는 것
    - 재사용을 통해 쉽고 빠르게 설계 가능
    - 반복된 코드의 중복 방지
  - **다형성(Polymorphism)**: 같은 타입이지만 실행 결과가 다양하도록 함
    - 자바는 부모 클래스, 인터페이스의 타입 변환을 허용

### Class
- 객체를 생성하기 위한 **설계도**
- new 연산자로 객체 생성 후, 리턴된 객체의 주소를 변수에 저장
  - 생성된 객체는 메모리 힙(heap), 객체의 주소는 변수에 저장(주로 스택)
- 구성요소
  - 필드(Field): 객체의 데이터가 저장되는 곳
  - 생성사(Constructor): 객체 생성 시 초기화 역할
  - 메소드(Method): 객체의 동작에 해당하는 실행 블록
- **final 필드**
  - 초기값이 저장되면 최종 값이 되어 수정이 불가능
- 접근 제한자(Access Modifier)
  - public / protected / default / private
  - **Entity에서 JPA 기본생성자는 protected**로 개발(외부에서 직접 객체 생성 막고, 프레임워크에서만 사용 가능하게)
    - JPA는 DB에서 데이터를 꺼낼 때, 리플렉션(Reflection)으로 **객체를 자동 생성**함. 이때, 파라미터 없는 기본 생성자가 꼭 필요함. ex. new User() 처럼 아무 값 없이 객체를 만들 수 있어야 JPA 내부 동작이 가능
    - protected로 두면, 외부 클래스에서는 객체를 직접 못 만들고, 오직 상속 클래스 or 같은 패키지에서만 생성 가능. 즉, 프레임워크(JPA)만 사용할 수 있게 제한(외부에서 new User() 선언 방지)
    - JPA가 DB에서 객체를 생성하는 과정
      - DB에서 데이터를 읽어옴
      - JPA 내부적으로 리플렉션(Reflection) 사용: 직접 new로 객체를 만들지 않아도, 내부적으로 Entity의 **생성자**를 찾아서 객체를 만듦. 근데 파라미터 없는 생성자(=기본 생성자)를 사용
      - 생성된 객체에 DB 값을 채워 넣음: 필드 값은 리플렉션을 이용해 직접 할당(setter 필요 없음!)
- Getter / Setter
  - OOP에서 객체의 데이터는 객체의 외부에서 직접적으로 접근하는 것을 막음(무결성)
  - 따라서 메소드를 통해 데이터를 변경(데이터는 접근을 막고, 메소드만 공개)
    - 메소드는 파라미터 값을 검증하여 유효한 값만 데이터로 저장 가능하기 때문 -> ``Setter``
    - 다만, Setter 남발 시 무결성이 깨질 가능성이 있음
      - 필수/안전하게 허용할 값만 Setter로 열어주기
    - **상태 변화 없는 객체는 무조건 불변으로 설계(final + private + Setter X + 생성자로만 값 할당)**
- 🚨 클래스 선언 시 **필드를 private 로 선언**하여 외부로부터 보호하고, 필드에 대한 Setter, Getter 메소드를 작성하여 필드값을 안정하게 변경 및 사용하는 것이 좋음
- 클래스 전체에서 “공통으로” 쓰는 값(상수 등)은 static final로 선언해서 인스턴스 생성 없이 사용

---

## 3. 예제 코드/활용법 (Example)
```java
@Entity
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED) // JPA 기본생성자 (외부 new 금지)
public class User {

    @Id @GeneratedValue
    private Long id;

    private final String name;          // 불변 필드(생성자에서만 할당, Setter X)
    private int age;                    // 변경 가능(Setter O, 검증 필수)

    public static final String DEFAULT_ROLE = "USER"; // 공통 상수(static final)

    // 객체 생성 시 반드시 name, age를 받아야 함(불변성 보장)
    public User(String name, int age) {
        validateAge(age);               // 유효성 검증(무결성)
        this.name = name;
        this.age = age;
    }

    // 나이 변경 메서드(Setter 대신 검증 추가)
    public void updateAge(int age) {
        validateAge(age);
        this.age = age;
    }

    // 나이 검증 로직(비즈니스 규칙 캡슐화)
    private void validateAge(int age) {
        if (age < 0) {
            throw new IllegalArgumentException("나이는 0 이상이어야 합니다.");
        }
    }
}

```

## 4. 실전 적용/느낀 점 (Usage & Insights)
- 클래스에서 데이터 무결성을 지키는 방법에 대해 배울 수 있었다.. 그리고 Entity 기본 annotation @NoArgsConstructor default 로 했는데, protected 로 변경해야겠다.. (new UserEntity()를 호출 방지) 코드 보니까 엉망진창이군 아놔~ ㅜㅜ
---

## 5. 참고/추가 자료 (References)
- 이것이 자바다 6강

---

## 6. 내일/다음에 볼 것 (Next Steps)
- JPA 에서 Entity 생성할 때 작동방식 궁금함
