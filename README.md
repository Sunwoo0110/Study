# Study

개발하면서 발생한 문제 상황 및 공부 내용을 기록합니다~  `\_へ(▭-▭)✨`

---
## 📑 목차

- [Study](#study)
  - [📑 목차](#-목차)
  - [📚 Study](#-study)
    - [⚒️ Backend](#️-backend)
    - [💾 DB](#-db)
    - [🌱 Spring](#-spring)
    - [🖥️ OS](#️-os)
  - [🐞 Troubleshooting](#-troubleshooting)

---

## 📚 Study

### ⚒️ Backend
| 파일 | 설명 |
| --- | --- |
| [backend_cache.md](Study/Backend/backend_cache.md) | cache의 개념과 전략, eviction 방법, 개발 시 고민 사항, redis의 영속성, 원자성 보장, 분산락 정리 |
| [backend_message_queue.md](Study/Backend/backend_message_queue.md) | 메세지 큐의 개념과 메세지 전송 보장 방식, Kafka의 메세지 처리 흐름, 개발 시 고민 사항, 배치 + 스케줄러와의 사용 방법 |
| [backend_scaleout.md](Study/Backend/backend_scaleout.md) | 시스템 규모 확장을 위한 기법 정리 |
| [backend_zero_downtime_deployment.md](Study/Backend/backend_zero_downtime_deployment.md) | 무중단 배포의 필요성과 기법, 실제 도입 사례 정리 |
| [backend_incident_perf_handling.md](Study/Backend/backend_incident_perf_handling.md) | 서비스 장애 원인 분석 및 해결, 장애 모니터링 사례 정리 |
| [backend_mvc_pattern.md](Study/Backend/backend_mvc_pattern.md) | MVC 패턴의 개념 및 장단점과 Spring에서의 작동 방식 |


### 💾 DB

| 파일 | 설명 |
| --- | --- |
| [db_transaction.md](Study/DB/db_transaction.md) | MySQL의 Lock, Transaction, Isolation level, Propagation, Spring의 `Transactional`, 분산 환경에서의 보상 트랜잭션 정리 |
| [db_index.md](Study/DB/db_index.md) | 인덱스 개념과 분류, B-Tree 성능 요인과 주요 스캔 방식, 멀티 컬럼 인덱스, 클러스터링 인덱스, 유니크 인덱스, 외래 키 |
| [db_mysql_vs_mongodb.md](Study/DB/db_mysql_vs_mongodb.md) | MySQL 과 MongoDB를 성능, 확장성, 속도, 데이터 정합성 등의 측면에서 비교 분석 정리 |
| [db_normalization.md](Study/DB/db_normalization.md) | 정규화 개념, 과정, 필요성, 비정규화 장점 |

### 🌱 Spring

| 파일 | 설명 |
| --- | --------- |
| [spring_entity.md](Study/Spring/spring_entity.md) | Entity 개념 및 설계 시 주의점 |
| [spring_jpa.md](Study/Spring/spring_jpa.md) | JPA 의 개념, 동작 원리, 주의 사항 |
| [spring_dto.md](Study/Spring/spring_dto.md) | DTO 개념 및 설계 시 주의점 |
| [spring_persistence_context.md](Study/Spring/spring_persistence_context.md) | 스프링 영속성 컨텍스트의 개념과 역할 |
| [spring_java_class.md](Study/Spring/spring_java_class.md) | Java Class 구조 및 특징 |
| [spring_security_jwt.md](Study/Spring/spring_security_jwt.md) | Spring Security + JWT 인증 구현/이론 |
| [spring_bean.md](Study/Spring/spring_bean.md) | Spring container와 bean 개념, 생명주기 |
| [spring_annotation.md](Study/Spring/spring_annotation.md) | Spring 주요 Annotation 개념/예제 정리 |

### 🖥️ OS

| 파일 | 설명 |
| --- | --- |
| [os_operating_system.md](Study/OS/os_operating_system.md) | 운영체제 기본 개념 |
| [os_process.md](Study/OS/os_process.md) | 프로세스 개념 및 프로세스 간의 통신 방법 |
| [os_thread.md](Study/OS/os_thread.md) | 스레드 개념 및 멀티스레드 장단점 |
| [os_cpu_scheduling.md](Study/OS/os_cpu_scheduling.md) | CPU 스케줄링 개념과 종류 |
| [os_synchronization.md](Study/OS/os_synchronization.md) | 경쟁 상태와 동기화 기법 종류 |
| [os_deadlock.md](Study/OS/os_deadlock.md) | 데드락의 발생 조건과 방지, 회피, 감지 방법  |
| [os_main_memory.md](Study/OS/os_main_memory.md) | 메모리 할당 방식과 페이징 기반 주소 변환 |
| [os_virtual_memory.md](Study/OS/os_virtual_memory.md) | 가상 메모리의 page fault와 페이지 교체 알고리즘 |

---

## 🐞 Troubleshooting

| 파일 | 설명 | 종류 |
| --- | ------------- | --- |
|[Java에서_Integer와_int_차이.md](Troubleshooting/Java에서_Integer와_int_차이.md)| Java에서 기본형과 참조형의 차이 | Java |
|[Spring_type_unboxing.md](Troubleshooting/Spring_type_unboxing.md)| NPE 발생: 타입 언박싱 오류 | Spring |
|[Long_type.md](Troubleshooting/Long_type.md)| Long 타입 vs Int 타입| Algorithm |
|[Flyway_도입_문제.md](Troubleshooting/Flyway_도입_문제.md)| Flyway 도입 시 발생한 문제 해결 | Spring |
