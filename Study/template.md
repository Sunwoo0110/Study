# 📚 공부 내용 (파일명 동일)

---

## 1. 주제/키워드
- (공부한 내용의 핵심 주제, 키워드)
- 예시: Spring @Transactional, JPA N+1 문제, RESTful API 설계 등

---

## 2. 핵심 요약 (Summary)
- 오늘 배운 것, 개념/원리 요점 정리  
- 한두 문장/리스트로 핵심만 압축
- 패드로 정리했을 때는 사진 첨부 
- 예시:
  - @Transactional은 트랜잭션 범위를 설정하는 어노테이션
  - JPA에서 Lazy Loading은 지연 로딩, 필요할 때 DB 조회

---

## 3. 예제 코드/활용법 (Example)
```java
// 대표 예제 코드나 자주 쓰는 패턴
@Transactional(readOnly = true)
public User getUser(Long id) {
    return userRepository.findById(id).orElseThrow();
}
```

## 4. 실전 적용/느낀 점 (Usage & Insights)
- 실제 프로젝트/문제에 어떻게 적용했는지  
- 공부하다가 느낀 점, 헷갈렸던 부분, 개선 아이디어 등

---

## 5. 참고/추가 자료 (References)
- 공식문서, 블로그, 영상, 강의 등  
  - [공식 스프링 트랜잭션 문서](https://docs.spring.io/...)
  - [인프런 강의](https://inflearn.com/...)

---

## 6. 내일/다음에 볼 것 (Next Steps)
- 더 파보고 싶은 개념, 오늘 남은 의문점, 내일 목표 등  
- 예시: Proxy와 @Transactional의 내부 동작, 트랜잭션 전파 옵션

