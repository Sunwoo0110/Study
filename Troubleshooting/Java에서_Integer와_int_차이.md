# 🛠️ Java에서 int vs Integer 차이 정리

---

## 1. 문제 상황 (What happened?)
- Java에서 숫자 타입을 사용할 때, 컬렉션에 담거나 null 값을 처리할 때 오류가 발생함
- 그리고 null 가능 여부 이외에 다른 차이가 있는지 궁금했음
- ex. `List<int>`는 선언 불가, `Integer x = null;`은 가능하지만 `int x = null;`은 컴파일 오류

---

## 2. 원인 분석 (Why?)
- Java에는 원시 타입(`int`)과 참조 타입(`Integer`)이 분리되어 있음
  - | 구분      | 기본 타입 (`Primitive`)                  |참조 타입 (`Reference`)                             |
    | ------- | ------------------------------------ | ----------------------------------------------- |
    | 예시      | `int`, `double`, `char`, `boolean` 등 | `String`, `Integer`, `List`, `User`, `Object` 등 |
    | 메모리 저장  | **실제 값**을 **스택**에 저장                        | **스택**에 객체가 생성된 **주소**(참조값)를 저장하고, 실제 객체는 **Heap** 메모리에 저장 |
    | 메모리 사용  | 작고 빠름 (stack 저장)                        | 상대적으로 무겁고 느림 (heap + 참조) |
    | null 허용 | ❌ 불가 (`int x = null;` → 오류)          | ✅ 가능 (`Integer x = null;`)                      |
    | 컬렉션에 사용 | ❌ 불가 (`List<int>` ← 오류)              | ✅ 가능 (`List<Integer>`)                          |
    | 객체 생성   | 사용 불가 (`new int()` ← 불가능)            | 사용 가능 (`new Integer(10)`, `new String("abc")`)  |

- 오토 박싱/언박싱 덕분에 자동 변환되지만, 성능, 메모리, Null 안정성 문제가 존재함
  - ```java
        Integer a = 10;       // auto-boxing: int -> Integer
        int b = a + 5;        // auto-unboxing: Integer -> int
        List<Integer> list = List.of(1, 2, 3); // auto-boxing
    ```
  - 박싱된 객체(int -> Integer)는 힙에 개체가 생성됨
  - 따라서 이 객체들은 heap에 쌓이고, 가비지 컬랙션 대상이 됨 -> GC 부담 증가

- 조금 다른 얘기지만, 참조 타입에서 128 ~ 127 범위는 캐싱되어 동일 객체 재사용 가능
    ```java
    Integer a = 100;
    Integer b = 100;
    System.out.println(a == b); // true (같은 캐시 객체)

    Integer c = 200;
    Integer d = 200;
    System.out.println(c == d); // false (다른 객체): new Integer() 객체 새로 생성됨
    ```
    - 따라서 == 비교 시 무조건 equals() 써야 안전

---

## 3. 최종 해결 방법 (How fixed)
### 언제 어떤 타입을 사용하면 좋을까?
- 기본 타입(int)
  - 성능, 메모리 효율이 중요할 때(연산이 빠름)
    - Entity 내부 증감 계산 로직
  - null이 절대 들어올 수 없는 필드일 때
    - 기본값이 0/false 등으로 의미가 있을 때

- 참조 타입(Integer)
  - null이 필요할 때
    - JPA **Entity** 필드(DB에 null 저장 가능해야 함)
    - **DTO**(클라이언트가 값을 보내지 않으면 null로 받기 위함)
    - 예외 처리
  - 컬렉션, 제네릭 사용
    - List<Integer>, Map<String, Boolean> 등 필요

---

## 4. 결과 및 느낀점 (Result & Learnings)
- 사실 여태 오토 박싱 기능을 굉장히 자주 편리하게 사용했는데, 이제 필요한 상황을 고려해 처음에 기본형, 참조형 중 어떤 것을 선택할 지 고민해야겠다.

---

## 5. 참고 자료 (References)
- [Baeldung - Difference between int and Integer](https://www.baeldung.com/java-integer-class-vs-type-vs-int)
- [StackOverflow - Comparing performance of int and Integer](https://stackoverflow.com/questions/4624933/comparing-performance-of-int-and-integer)
