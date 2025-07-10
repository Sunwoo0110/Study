# 🛠️ 문제 상황 제목 ( 파일명 동일하게 )

---

## 1. 문제 상황 (What happened?)
- 어떤 작업/기능/코드에서 무슨 문제가 발생했는지 작성  
- 예시: 회원가입 시 DB 저장은 되는데, 응답이 500 에러로 옴

---

## 2. 에러 메시지/현상 (Error/Log)
- 실제 콘솔/로그/에러 메시지 붙여넣거나 사진 캡쳐
- 예시: 
``` org.hibernate.LazyInitializationException: failed to lazily initialize a collection of role...-  ```


## 3. 원인 분석 (Why?)
- 에러가 발생한 근본 원인/흐름, 추측/확인한 이유 정리  
- 예시: 트랜잭션 범위 바깥에서 Lazy 로딩된 엔티티 접근 시도

---

## 4. 시도해 본 해결 방법 (Tried)
- 문제를 해결하기 위해 시도한 방법과 그 결과  
  - [ ] fetch join 사용 → 실패  
  - [ ] @Transactional 적용  
- (시도 결과도 간단히!)

---

## 5. 최종 해결 방법 (How fixed)
- 실제로 적용해서 해결된 방법, 핵심 코드/설명  
- 예시: `@Transactional(readOnly = true)` 추가 후 정상 동작

---

## 6. 결과 및 느낀점 (Result & Learnings)
- 문제 해결 결과, 배운 점, 다음에 유의할 점 등  
- 예시: 서비스 레이어 트랜잭션 범위 중요성 이해
- 공부하면서 정리한 내용 있으면 링크 걸기

---

## 7. 참고 자료 (References)
- 공식문서, 블로그, StackOverflow 등 참고한 자료 링크  
  - [공식 JPA 트랜잭션 설명](https://docs.spring.io/...)
  - [관련 StackOverflow 답변](https://stackoverflow.com/...)

---

## 8. 관련 Issue/PR/Commit (Optional)
- 해당 깃허브 issue, PR, commit 링크/번호 등  
  - #13 Lazy 로딩 에러  
  - PR: [fix: 트랜잭션 범위 조정 (#15)](https://github.com/...)
