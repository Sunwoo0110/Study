# 📚 1. Operating System

---

## 1. 주제/키워드
  - 운영체제란 무엇인가.. ヽ(‘ ∇‘ )ノ

---

## 2. 핵심 요약 (Summary)

### Operating System
  - Computer를 운영하는 software
  - Computer란? -> 정보를 처리하는 기계
  - 폰 노이만 구조
    - **Memory(RAM)에 program을 저장**하는 stored-program computer
    - 메모리에서 명령어 fetch -> instruction register에 저장 -> decode -> execute -> 결과를 메모리에 저장
  
### Program
  - 하드웨어가 특정한 일을 수행하도록 하는 **instruction(명령어)들의 집합**
  - Program: 명령어 집합, Process: 실행 중인 프로그램 인스턴스
  - 그럼 OS도 program인가요? >> 넵! 
    - 컴퓨터에서 항상 메모리에 상주하며 실행되는 **핵심 프로그램 → Kernel(커널)**
    - Kernel만 직접적으로 하드웨어 접근/제어 가능(excute system call)
    - System Call
      - 프로그램(유저)이 커널에게 특정 작업을 요청할 때 사용하는 함수/인터페이스  
      - ex. process control:  ``fork()``, ``wait()`` .. 


---

## 3. 참고/추가 자료 (References)
- [인프런 운영체제 공룡책 강의](https://www.inflearn.com/course/%EC%9A%B4%EC%98%81%EC%B2%B4%EC%A0%9C-%EA%B3%B5%EB%A3%A1%EC%B1%85-%EC%A0%84%EA%B3%B5%EA%B0%95%EC%9D%98)

---

## 4. 내일/다음에 볼 것 (Next Steps)
- Process 공부하자~~ (;´Д`)

