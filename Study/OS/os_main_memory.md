# 📚 Main Memory

---

## 1. 주제/키워드
- 운영 체제의 메인 메모리에 대해 알아보자! (ง •̀_•́)ง

---

## 2. 핵심 요약 (Summary)

### Main Memory란?
- **CPU가 직접 접근하는 저장 공간**으로, 실행 중인 **프로그램의 명령어와 데이터를 저장**함
- 주소를 갖는 **바이트 단위 배열 구조**로 되어 있음.
- CPU는 **프로그램 카운터**(PC)가 가리키는 주소를 따라 메모리에서 명령어를 **fetch**(가져옴)해서 실행
- 명령어 실행 중 필요에 따라 메모리에서 데이터를 읽어오거나 저장하기도 함 (load/store)

---

### Memory Space 분리 & 보호
- 각 **프로세스는 독립적인 메모리 공간**을 가져야 함
- **Base Register + Limit Register** 구조를 통해 허용된 주소 범위를 벗어난 접근을 막음
- 사용자 모드에서 생성된 주소는 **CPU 하드웨어가 체크**함

---

### Address Binding
- 프로그램은 디스크에 바이너리 파일로 존재하다가 실행을 위해 메모리에 올라감
- 코드 내 주소는 보통 **Symbolic Address**로 존재
- 이후 컴파일러가 **Relocatable Address(재배치 주소)**로 바꿈
- 최종 실행 시 **Absolute Address**로 바인딩됨

---

### Logical Address vs Physical Address
- Logical Address(논리 주소)
  - CPU가 생성한 주소 (프로세스 입장에서 보는 주소)
- Physical Address(물리 주소)  
  - 실제 메모리에서의 주소
- logical address space와 physical address space는 분리되어 있음
- CPU가 논리 주소를 만들고, **MMU**(Memory Management Unit)가 물리 주소로 변환함

---

### MMU (Memory Management Unit)
- **논리 주소 -> 물리 주소**로 변환하는 하드웨어 장치
- 내부적으로는 **Relocation Register**를 사용해서 변환 수행
- 프로그램은 항상 0에서 시작하는 것처럼 보이지만 실제로는 물리 메모리의 아무 위치에서 실행 가능함

---

### Dynamic Loading(동적 로딩)
- 프로그램 전체를 한 번에 메모리에 올리지 않고, **필요할 때 해당 루틴만 로딩**
- 메모리 공간을 효율적으로 사용 가능
- `Relocatable Linking Loader`가 필요한 코드만 불러와 주소 테이블 갱신

---

### Dynamic Linking
- 시스템 라이브러리 등 **공통 코드가 필요한 순간에만 연결**(Linking)됨
- 메모리에 라이브러리 한 번만 올려놓고 **여러 프로세스가 공유**
- ex. Windows의 `.dll`, Linux의 `.so`

---

### 메모리 할당 전략

#### Contiguous Memory Allocation (연속 할당 방식)
- 메모리를 **연속된 블록**으로 나눠서 프로세스에 할당
- OS 영역 + 사용자 프로세스 영역

#### Variable Partition (동적 할당 방식)
- 메모리를 **필요한 크기만큼 동적으로 할당**
- 사용 가능한 빈 공간(hole) 중에서 골라 할당
- First-Fit / Best-Fit / Worst-Fit

---

### Fragmentation (메모리 단편화)
- External fragmentation (외부 단편화)
  - 메모리 빈 공간이 쪼개져서 활용 못하는 문제
- Internal fragmentation (내부 단편화)
  - 할당된 공간 내 일부가 낭비되는 문제

---

### Paging (페이징)
- 물리 메모리를 프레임(**frame**)으로, 논리 메모리를 페이지(**page**)로 나눠서 관리
- 프로세스의 주소 공간이 **물리 메모리에서 연속일 필요 없음**
- **외부 단편화 방지** + **메모리 공간 활용률 향상**
- 주소 변환 과정
  1. logical address -> page number + page offset
  2. 페이지 번호로 **page table** 검색 -> **frame number** 찾기
  3. 최종 physical address = frame number + page offset

---

### TLB (Translation Lookaside Buffer)
- 주소 변환 속도를 높이기 위한 **캐시**
- 최근 사용한 페이지 테이블 항목들을 저장
- EAT (Effective Access Time)
  - TLB hit 비율에 따라 실질적인 메모리 접근 시간 변화

---

### 메모리 보호 (Protection)
- 페이지 테이블의 각 항목에 **유효(valid) / 무효(invalid)** 비트 부여
- 프로세스가 접근 가능한 페이지인지 여부 판단

---

### 공유 페이지 (Shared Page)
- 여러 프로세스가 동일한 코드(like libc)를 동시에 공유할 수 있도록 지원
- 코드는 **재진입 가능**(non-self-modifying)해야 함

---

### Structure of the Page Table
- 주소 공간이 커질수록 페이지 테이블도 커짐
- 이를 해결하기 위한 방법
  - **Hierarchical Paging**: 다단계 테이블
  - **Hashed Page Table**: 해시 사용
  - **Inverted Page Table**

---

### Swapping
- 실제 메모리보다 더 많은 프로세스를 처리하기 위해, **일시적으로 메모리 밖으로 내보내는 기법**
- 전체 프로세스를 디스크로 내보냈다가 다시 불러오는 게 아니라, **필요한 프로세스만 메모리로 올림**
- 페이지 단위로 스와핑하면 **속도와 효율 모두 향상**
- **virtual memory와 함께 사용됨**

---

## 3. 참고/추가 자료 (References)
- [인프런 운영체제 공룡책 강의](https://www.inflearn.com/course/%EC%9A%B4%EC%98%81%EC%B2%B4%EC%A0%9C-%EA%B3%B5%EB%A3%A1%EC%B1%85-%EC%A0%84%EA%B3%B5%EA%B0%95%EC%9D%98)

---

## 4. 내일/다음에 볼 것 (Next Steps)
- 가상 메모리!
