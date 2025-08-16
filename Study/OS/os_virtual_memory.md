# 📚 Virtual Memory

---

## 1. 주제/키워드
- 운영 체제의 **가상 메모리(Virtual Memory)** 에 대해 알아보자! (๑•̀ㅂ•́)و✧

---

## 2. 핵심 요약 (Summary)

### Virtual Memory란?
- **실제 메모리에 프로그램 전체를 올리지 않고도 실행 가능**하게 하는 기법
- 물리 메모리보다 큰 프로그램도 실행 가능
- 논리 메모리(logical memory)와 물리 메모리(physical memory)를 분리해 **추상화된 메모리 공간 제공**
- **파일 및 라이브러리 공유** 및 **프로세스 생성 효율화** 지원

---

### Virtual Address Space (가상 주소 공간)
- 프로세스 기준 **논리적 메모리 구조**
- 보통 address 0 부터 연속적으로 존재하는 것처럼 보임
- 실제 물리 메모리와는 다름
- 여러 프로세스가 같은 라이브러리를 **페이지 공유**(page sharing) 방식으로 사용 가능

---

### Demand Paging (요구 페이징)
- 프로그램 전체를 메모리에 올리지 않고, **실행 시 필요한 페이지만 적재**
- 페이지 유효 여부를 **valid-invalid bit**로 구분
  - valid: 메모리에 존재
  - invalid: 아직 메모리에 없음(디스크에 있음)

- **Page Fault 처리 절차**
  1. 접근 가능한 지 확인
  2. 유효하면 **free frame** 확보
  3. 디스크에서 해당 페이지 읽어오기
  4. 페이지 테이블 갱신
  5. 인터럽트된 명령어 재시작
- **Pure Demand Paging**: 시작 시 아무 페이지도 올리지 않고 첫 실행에서 page fault 발생하며 점진적으로 로딩
- 성능은 **Locality of Reference**(참조 지역성) 덕분에 현실적으로 양호
  - Locality of Reference: 프로그램이 메모리를 참조할 때 **특정 영역을 집중적으로 반복해서 접근하는 경향**
  - Temporal locality: 시간적으로 최근 접근한 데이터를 다시 사용
  - Spatial locality: 공간적으로 가까운 주소를 함께 참조

---

### Copy-on-Write (COW)
- `fork()` 시 부모, 자식이 페이지를 공유하다가, **쓰기 동작 발생 시점에 복사**(copy)

---

### Page Replacement (페이지 교체)
- free frame이 없을 경우, **사용하지 않는 프레임을 디스크로 내보내고 새로운 페이지 적재**
- **Page Replacement 알고리즘**
  - **FIFO**: 가장 먼저 들어온 페이지 제거. Belady’s Anomaly 발생 가능
  - **Optimal** (OPT): 앞으로 가장 오래 안 쓸 페이지 제거. 이론적 최적
  - **LRU** (Least Recently Used): 가장 오랫동안 사용되지 않은 페이지 제거
    - 과거의 참조 패턴(최근 사용 여부)을 근거로 미래 접근을 예측해서 locality를 활용(최근 사용 -> 앞으로도 사용될 가능성 높음)
    - Counter 방식, Stack 방식, Reference bit 기반 근사 알고리즘(Second-Chance, Clock 등)
- Belady’s Anomaly
  - 일반적으로는 프레임 수가 늘어나면 page fault가 줄어야 정상인데, FIFO 알고리즘에서는 프레임 수를 늘렸는데도 오히려 page fault가 증가하는 현상

| 알고리즘                                  | 특징                                            | 장점                              | 단점                               | Belady’s Anomaly |
| ------------------------------------- | --------------------------------------------- | ------------------------------- | -------------------------------- | ---------------- |
| **FIFO** (First-In First-Out)         | 가장 먼저 들어온 페이지부터 교체                            | 구현 간단, 직관적                      | 오래된 페이지가 자주 쓰이는 경우에도 교체됨         | 발생 가능          |
| **OPT** (Optimal / MIN)               | 앞으로 가장 오랫동안 사용하지 않을 페이지 교체                    | Page Fault 최소, 이론적 최적           | 미래 참조를 알아야 함 -> 실제 구현 불가 (비교/연구용) | 없음             |
| **LRU** (Least Recently Used)         | 가장 오랫동안 사용하지 않은 페이지 교체                        | locality 반영 -> 성능 우수, 실무에서 자주 사용 | 시간, 하드웨어 지원 필요 (카운터, 스택 등)        | 없음             |
| **Second-Chance** (Clock)           | FIFO + reference bit 사용, 참조된 페이지에 한 번 더 기회 부여 | LRU 근사 구현, 성능 양호                | 구현 복잡, 정확한 LRU는 아님               | 없음             |
| **LRU Approximation** (Reference Bit) | 페이지 접근 시 bit=1, 교체 시 0인 페이지 우선                | 하드웨어 부담 줄임                      | 정확성 떨어짐, 근사치                     | 없음             |


---

### Frame Allocation (프레임 할당)
- **Equal allocation**: 모든 프로세스에 동일한 수의 프레임 할당
- **Proportional allocation**: 프로세스 크기에 비례하여 할당
- Global vs Local Replacement
  - Global: 전체 시스템 프레임 중 교체
  - Local: 자기 프레임 내에서만 교체

---

### Thrashing (스래싱)
- 프로세스가 페이지 교체에만 몰두해 실제 일은 거의 못하는 상태
- 원인: 프로세스에 필요한 프레임 수 부족 -> page fault 과다 발생

---

### Working-Set Model (작업 집합 모델)
- locality 기반 접근
- 최근 n개 참조 내에 포함된 페이지 집합을 working set이라 정의
- working set 크기 = 프로세스 실행에 필요한 최소 프레임 수 추정
- 이를 바탕으로 프레임을 적절히 할당해 thrashing 방지

---

## 3. 참고/추가 자료 (References)
* [인프런 운영체제 공룡책 강의](https://www.inflearn.com/course/%EC%9A%B4%EC%98%81%EC%B2%B4%EC%A0%9C-%EA%B3%B5%EB%A3%A1%EC%B1%85-%EC%A0%84%EA%B3%B5%EA%B0%95%EC%9D%98)

---

## 4. 내일/다음에 볼 것 (Next Steps)
- 운영체제 복습~


