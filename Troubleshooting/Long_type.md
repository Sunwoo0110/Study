# 🛠️ Long type

---

## 1. 문제 상황 (What happened?)
- 각 몸무게 별 인원 수를 저장하려고 ``Map<Integer, Integer> map = new TreeMap<>();``로 구현 후, 조합 계산에 사용했으나, 일부 테케에서 실패
- [시소 짝꿍](https://school.programmers.co.kr/learn/courses/30/lessons/152996?language=java)

---

## 2. 원인 분석 (Why?)
- Map의 value 타입을 Integer로 선언하면, 큰 수의 곱셈/조합 계산에서 int 오버플로우 발생
  - 한 몸무게에 최대 100,000명까지 가능 →
  조합 수(nC2) 계산: 100,000 * 99,999 / 2 = 4,999,950,000 (int 최대값 초과)
- 다른 몸무게 쌍의 개수를 구하는 곱셈 ``(map.get(list.get(i)) * map.get(list.get(j)))``도 마찬가지
  - 두 value가 모두 100,000일 때 100,000 * 100,000 = 10,000,000,000 (int 최대값 초과)
- int 범위를 초과하는 모든 곱셈, 조합 결과는 반드시 long으로 받아야 하며, Map의 value와 answer도 long 타입 사용이 필수

---

## 3. 개념 정리 (Key Concepts)
### int
- 4바이트(32비트)
- -2,147,483,648 ~ 2,147,483,647

### long
- 8바이트(64비트)
- -9,223,372,036,854,775,808 ~ 9,223,372,036,854,775,807

---

## 4. 최종 해결 방법 (How fixed)
- int -> long 으로 타입 변경
- `` Map<Integer, Long> map = new TreeMap<>();``
- [최종 코드](https://github.com/Sunwoo0110/Algorithm/tree/main/%ED%94%84%EB%A1%9C%EA%B7%B8%EB%9E%98%EB%A8%B8%EC%8A%A4/2/152996.%E2%80%85%EC%8B%9C%EC%86%8C%E2%80%85%EC%A7%9D%EA%BF%8D)

---

## 6. 결과 및 느낀점 (Result & Learnings)
- Java에서 큰 수를 계산 할 때는 long 타입 써야 안전

