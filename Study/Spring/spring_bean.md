# 📚 Spring Bean & Container

---

## 1. 주제/키워드
- Spring container와 bean 개념, 생명주기, @Transactional과 영속성 컨텍스트 관계에 대해 알아보자~ (っ- ‸ – ς)

---

## 2. 핵심 요약 (Summary)

### Spring Container (스프링 컨테이너)
- 스프링에서 **객체(Bean)를 관리하는 박스** 같은 존재
- 역할
  - 객체 생성 (new 대신 해줌)
  - 의존성 주입
  - 생명주기 관리 (초기화, 종료 시점까지 관리)
- 왜 쓰냐?
  - 개발자가 `new`로 만들고 연결하는 과정이 없어짐
  - 객체 간 결합도를 낮추고, 테스트, 확장성을 높임
- 여기서 잠깐!
- 의존성 주입이란?
  - 어떤 객체가 필요로 하는 **다른 객체**(의존성)를 개발자가 직접 new 해서 만드는 대신, **스프링 컨테이너가 대신 만들어서 넣어주는 것**
  - 객체 간 결합도를 낮추고, 코드 재사용성과 테스트 용이성을 높임

---

### Spring Bean (스프링 빈)
- 컨테이너가 관리하는 **객체 인스턴스**
- 개발자가 직접 만든 클래스(@Service, @Repository 등)나, 설정(@Bean)으로 등록한 애들이 빈이 됨
- **스프링이 대신 관리하는 객체**
- 개발자가 직접 객체를 생성하고 연결하면 결합도 높고 테스트 어려움
- 따라서 스프링이 대신 관리하면
  - 필요한 시점에만 생성해서 메모리 절약 (싱글턴)
  - 다른 구현체로 쉽게 교체 가능
  - 단위 테스트 쉬움

---

### Bean 등록 방법
- **컴포넌트 스캔** (`@ComponentScan`)
  - `@Component`, `@Service`, `@Repository`, `@Controller` 붙은 클래스를 자동 탐지
  - 가장 흔한 방식, 보통 스프링 부트 메인 클래스 기준 하위 패키지를 싹 스캔
- **자바 설정 메서드** (`@Bean`)
  - 메서드 안에서 객체를 직접 생성해서 컨테이너에 등록
  - 주로 외부 라이브러리 객체나 생성 과정이 복잡한 애들
- **설정 클래스** (`@Configuration`)
  - `@Bean` 여러 개 묶어서 관리하는 설정 파일
  - 내부적으로 CGLIB 프록시가 개입해서 같은 Bean을 여러 번 호출해도 싱글턴 보장

---

### Bean LifeCycle (빈 생명주기)

```
컨테이너 시작
-> Bean 생성 (객체 new)
-> 의존성 주입 (DI)
-> 초기화 콜백 (@PostConstruct 같은 초기화 메서드 실행)
-> 애플리케이션 동작
-> 종료 직전 소멸 콜백 (@PreDestroy 같은 cleanup 실행)
컨테이너 종료
```
- 자주 쓰는 콜백
  - `@PostConstruct`: 초기 세팅 (ex. 캐시 로딩)
  - `@PreDestroy`: 종료 전 정리 (ex. 커넥션 닫기)

---

### @Transactional과 영속성 컨텍스트
- 여기서 중요한 건 **Bean 생명주기와 트랜잭션/영속성 컨텍스트 생명주기**가 다르다는 거!
- `@Service` 같은 Bean: 애플리케이션 시작할 때 딱 1번 만들어짐 (싱글턴)
- `@Transactional`
  - 메서드 호출 순간 프록시가 트랜잭션 열고 닫음
  - 트랜잭션 범위 안에서 **영속성 컨텍스트**(EntityManager)가 작동 -> DB 엔티티 관리
  - 커밋 시점에 flush -> DB 반영
- 따라서 **Bean 자체는 계속 살아있음**, **트랜잭션/영속성 컨텍스트는 메서드 실행할 때마다 열렸다 닫힘**
- 주의 사항: 같은 클래스 안에서 `@Transactional` 메서드끼리 서로 호출하면 프록시를 안 거쳐서 트랜잭션이 안 걸릴 수 있음 (self-invocation 문제)
- 여기서 잠깐!
- Self-invocation 문제란?
  - `@Transactional`은 **프록시**(proxy)를 통해 동작함
  - 즉, 스프링이 실제 객체 앞에 프록시 객체를 하나 더 씌워서 트랜잭션 시작/커밋을 처리함
  - 그런데 같은 클래스 안에서 자기 자신의 다른 메서드를 호출하면 프록시를 거치지 않음!!
    - 그냥 this.메서드()로 직접 실행되니까 트랜잭션이 적용되지 않음
  - ```java
        @Service
        public class testService {

            @Transactional
            public void test1() {
                // 트랜잭션 시작됨
                test2(); // 같은 클래스 메서드 호출!!
            }

            @Transactional
            public void test2() {
                // 여기서는 트랜잭션 안 걸림! (프록시를 안 거쳐서)
            }
        }
        ```
  - **클래스를 분리**하거나, **자기 자신을 주입**해서 해결
    - ```java
        @Autowired
        private testService self; // 프록시가 주입됨

        @Transactional
        public void test1() {
            self.test2(); // 프록시를 거치므로 트랜잭션 적용됨
        }
        ```

---

## 3. 참고/추가 자료
- [Spring Docs - Bean LifeCycle](https://docs.spring.io/spring-framework/reference/core/beans/factory-nature.html)
- [Spring Docs - Component Scanning](https://docs.spring.io/spring-framework/reference/core/beans/classpath-scanning.html)
- [Spring Docs - @Bean & @Configuration](https://docs.spring.io/spring-framework/reference/core/beans/java/basic-concepts.html)
- [Spring Docs - @Transactional](https://docs.spring.io/spring-framework/reference/data-access/transaction/declarative/annotations.html)

---

## 4. 다음에 볼 것
- 솔직히 반도 이해 못한 것 같은데..ㅜㅠ 스프링 왜이리 어렵니
- 스프링 개발 더 해보면서 여러번 공부하자~,,

