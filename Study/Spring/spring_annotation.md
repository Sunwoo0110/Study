# 📚 Spring Annotation

---

## 1. 주제/키워드
- Spring의 Annotation에 대해 알아보자~! *⸜( •ᴗ• )⸝*

---

## 2. 핵심 요약 (Summary)
### Annotation이란?
- 자바 코드에 **MetaData**를 추가하는 문법
- 종류
  - 컴파일러 지시(예: @Override)
  - 코드 자동 생성/도구 지원(예: Lombok @Getter, @Setter)
  - 프레임워크에서 동작 지정(예: Spring의 @Component, @Service, @Autowired 등)
- 사용하는 이유?
  - 코드 가독성 향상
  - 반복적인 설정/구현 최소화
  - 프레임워크(특히 Spring)에서 객체 생성, 의존성 주입, 트랜잭션 처리 등 자동화에 필수적

### Spring에서 자주 사용하는 Annotation
### Bean/Component 등록 (객체 생성/관리)
| Annotation             | 용도                                        |
| ----------------- | ----------------------------------------- |
| `@Component`      | 스프링이 자동으로 빈(bean)으로 등록 (가장 기본)            |
| `@Service`        | “비즈니스 로직” 서비스 계층 컴포넌트 (실제는 @Component 역할) |
| `@Repository`     | “데이터 접근 계층” (DAO/DB), 예외 변환 기능 포함         |
| `@Controller`     | “웹 요청-응답” 담당(Controller 계층)               |
| `@RestController` | @Controller + @ResponseBody (JSON 등 반환)   |
| `@Configuration`  | 설정(설정 클래스 등록, Bean 등록 메서드)                |
| `@Bean`           | 메서드에 붙여서 수동으로 빈 등록                        |

### 의존성 주입/DI
| Annotation                     | 용도                                 |
| -------------------------- | ---------------------------------- |
| `@Autowired`               | 타입/이름 기반 자동 주입                     |
| `@Qualifier`               | 동일 타입 여러 개 빈 있을 때 이름으로 구분          |
| `@Inject`                  | @Autowired랑 거의 같음(Jakarta/Java 표준) |
| `@Value`                   | application.properties 값 주입        |
| `@RequiredArgsConstructor` | final 필드 자동 생성자 생성(Lombok)         |

### 요청 및 응답
| Annotation             | 용도                           |
| ----------------- | ---------------------------- |
| `@RequestMapping` | URL, HTTP 메서드 매핑(가장 기본)|
| `@GetMapping`     | GET 방식 요청 매핑                 |
| `@PostMapping`    | POST 방식 요청 매핑                |
| `@PutMapping`     | PUT 방식 요청 매핑                 |
| `@DeleteMapping`  | DELETE 방식 요청 매핑              |
| `@PathVariable`   | URL 경로 변수 받기                 |
| `@RequestParam`   | 파라미터 받기                |
| `@RequestBody`    | HTTP Body(JSON 등)로 객체 받기     |
| `@ResponseBody`   | 반환값을 그대로 응답 바디로 매핑(주로 JSON)     |
| `@ModelAttribute` | 파라미터 → 객체로 바인딩               |

### 기타
| Annotation                    | 용도                      |
| ------------------------ | ----------------------- |
| `@Transactional`         | 트랜잭션 처리(주로 서비스 계층에 붙임)  |
| `@Slf4j`                 | 로그 자동 생성(Lombok)        |
| `@ExceptionHandler`      | 예외 핸들러 메서드              |
| `@Valid`, `@Validated`   | 유효성 검사(입력값 검증, DTO에 붙임) |
| `@Data`                  | getter/setter/생성자/비교/문자열 변환 생성 |
| `@Scheduled`             | 스케줄링(정해진 시간마다 실행)       |
| `@EnableScheduling`      | 스케줄러 기능 켜기              |
| `@EnableJpaRepositories` | JPA 레포지토리 스캔            |

### Annotation 사용 시 주의 사항
- Entity에는 @Data 잘 안 씀
  - setter 사용 시 객체 상태가 예측 불가, 영속성 컨텍스트 꼬임 문제
  - Entity에는 @Getter만 쓰는 게 관례 (필요한 곳만 setter 직접 추가)
  - DTO에는 @Data 자주 씀(데이터 전달용이기 때문)
- @Transactional은 **Service 계층**에 붙임
  - Controller, Repository에 직접 붙이지 않음! (구조/역할 분리)
  - 읽기 전용은 @Transactional(readOnly = true)로 성능 최적화
- 입력값 검증은 @Valid, @Validated로 명확히!
  ```java
    @PostMapping("/user")
    public ResponseEntity<?> create(@Valid @RequestBody UserDto userDto) { ... }
  ```
- 예외/에러는 전역에서 관리
  - @ControllerAdvice/@ExceptionHandler
- 필드는 final + 생성자(@RequiredArgsConstructor) 주입이 표준
  - setter 주입: 테스트나 개발 중에만 가끔 쓰고, 운영/실무에는 지양
  - @Autowired 필드 주입: 비추천(순환참조, 테스트 어려움)
  - @RequiredArgsConstructor: final 필드만 생성자로 주입 → 불변성, 테스트 용이
    - final로 선언하면 한 번만 값이 들어갈 수 있음(불변)
      - final을 사용하면 값이 반드시 “있는” 상태로만 객체가 생성
    - 생성자 주입은 객체가 만들어질 때 딱 한 번 값을 넣어줌


---

## 3. 예제 코드/활용법 (Example)
```java
@Service
@Transactional
public class UserService { ... }


@RestController
@RequestMapping("/api/user")
public class UserController { ... }
```

## 4. 실전 적용/느낀 점 (Usage & Insights)
- 산학 때 코드 리뷰 받으면서 가장 많이 수정한 부분 중에 하나가 Annotation이였는데, 당시에는 바꾸자~ 하고 넘어갔던 것 같다. 지금 공부하니까 왜 이 class에 이 annotation이 필요한 지 이해할 수 있었다.

---

## 5. 참고/추가 자료 (References)
- GPT 와 다수의 velog... 감사합니다

---

## 6. 내일/다음에 볼 것 (Next Steps)
- 자주 사용하거나 중요한 Annotation들 (@Transactional, @Repository 등) 내부 동작 파보기 ~

