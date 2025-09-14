
# 📚 Mysql vs MongoDB

## 1. 주제/키워드
- MySQL (RDBMS) vs MongoDB (NoSQL) 에 대해 비교해보자~~ ヾ(＠⌒ー⌒＠)ノ 

## 📑 목차
- [📚 Mysql vs MongoDB](#-mysql-vs-mongodb)
  - [1. 주제/키워드](#1-주제키워드)
  - [📑 목차](#-목차)
  - [2. 핵심 요약 (Summary)](#2-핵심-요약-summary)
    - [MySQL](#mysql)
    - [MongoDB](#mongodb)
    - [MongoDB가 MySQL보다 나은 점](#mongodb가-mysql보다-나은-점)
    - [다양한 부분에서 비교해보기~](#다양한-부분에서-비교해보기)
      - [가용성 (Availability)](#가용성-availability)
      - [데이터 모델 \& 개발 효율](#데이터-모델--개발-효율)
      - [확장성 (Scalability)](#확장성-scalability)
      - [쿼리 언어 \& 표준](#쿼리-언어--표준)
      - [성능](#성능)
      - [데이터 정합성 \& 트랜잭션](#데이터-정합성--트랜잭션)
      - [스토리지 \& 메모리 효율](#스토리지--메모리-효율)
      - [운영 \& 안정성](#운영--안정성)
    - [정리](#정리)
  - [3. 참고/추가 자료 (References)](#3-참고추가-자료-references)
  - [4. 내일/다음에 볼 것 (Next Steps)](#4-내일다음에-볼-것-next-steps)

---

## 2. 핵심 요약 (Summary)

### MySQL
  - 오라클(Oracle)에서 제공하는 **관계형 데이터베이스 관리 시스템**(RDBMS)
  - 데이터를 테이블/행/열로 저장
  - **SQL**을 사용해 접근
  - 여러 테이블 데이터를 합치려면 **JOIN 필요**
  - 데이터베이스 스키마를 사전에 정의해야 하며, 테이블 간 관계 규칙도 명시

### MongoDB
  - **NoSQL** 문서 기반 데이터베이스
  - 데이터를 **JSON 유사 문서**(Document)로 저장
  - 문서는 자체적으로 구조를 설명(Self-Describing)
  - MQL (MongoDB Query Language) 사용
  - 필요 시 스키마 검증(schema validation) 기능으로 데이터 일관성 유지

---

### MongoDB가 MySQL보다 나은 점
- **빠른 개발 속도**
  - 객체지향 언어와 문서(Document)가 자연스럽게 매핑
  - ORM(Object Relational Mapping) 계층 불필요
  - 스키마가 유연 -> 요구사항 변화에 맞춰 데이터 모델 쉽게 확장 가능
- **확장성과 가용성**
  - 다중 데이터센터에 걸친 확장 가능
  - 무중단으로 용량 및 처리량 증가 대응
  - MySQL은 확장을 위해 별도 커스텀 엔지니어링 필요

---

### 다양한 부분에서 비교해보기~

#### 가용성 (Availability)
- MongoDB
  - Replica Set 기반 고가용성
  - 장애 시 자동으로 Primary 선출(수 초 내) -> **무중단 운영**에 강점
  - readConcern, writeConcern 으로 일관성 수준 조정 가능
- MySQL
  - 전통적으로 수동 Failover가 많아 복구까지 수분 이상 걸릴 수 있음
  - 최근에는 HAProxy, Orchestrator 같은 도구로 자동화 가능하지만, MongoDB보단 느림

#### 데이터 모델 & 개발 효율
- MongoDB
  - JSON/BSON 문서 모델 -> 앱 데이터 구조와 유사, 직관적
  - 배열, 서브 document로 한 엔티티 관련 데이터를 한 문서에 저장 -> JOIN 필요 줄어듦
  - 각 문서가 다른 스키마를 가질 수 있어 유연성이 높음
- MySQL
  - 엄격한 스키마 기반 -> 데이터 무결성 보장에 유리
  - JSON 컬럼을 지원하나, 함수/처리가 필요해 MongoDB보단 번거로움

#### 확장성 (Scalability)
- MongoDB
  - 기본적으로 Sharding 지원 -> 손쉽게 Scale-Out 가능
  - Atlas Global Cluster로 글로벌 분산 배치 용이
- MySQL
  - 전통적으로 Scale-Up(고성능 서버) 중심
  - 샤딩 시 JOIN 제약, 분산 설계는 추가적인 노력 필요

#### 쿼리 언어 & 표준
- MySQL
  - SQL 기반 -> 30년 이상 축적된 표준, 툴, 레퍼런스, BI 연동 풍부
- MongoDB
  - MQL(Mongo Query Language) 사용 -> JSON 친화적이지만 SQL만큼 표준화 안 됨
  - SQL 기반 BI/분석 도구와의 호환성은 상대적으로 약함

#### 성능
- MongoDB
  - 문서 단위로 연관 데이터가 모여 있어 단순 조회, 쓰기 성능 강함
  - 수평 확장 덕분에 대규모 트래픽 처리에 유리
- MySQL
  - 인덱스, JOIN 최적화로 정형 데이터 조회/집계 성능 뛰어남
  - OLAP/OLTP(리포트, 대시보드) 환경에서 안정적

#### 데이터 정합성 & 트랜잭션
- MySQL
  - ACID 트랜잭션 강력 지원
  - JOIN, Foreign Key 제약조건을 통해 데이터 무결성을 데이터베이스 레벨에서 강제
  - 은행, 결제, 주문 같은 정합성이 최우선인 서비스에 적합
  - 트랜잭션 격리 수준(Isolation Level)까지 세밀하게 조정 가능
- MongoDB
  - 문서 단위 트랜잭션은 빠름
  - 멀티 도큐먼트 트랜잭션도 지원하지만 성능 오버헤드가 큼
  - 대신 스키마 유연성과 단일 문서 내 중첩 구조 덕분에, 하나의 문서 안에서 정합성 보장이 쉬움
  - 금융보다는 소셜, 로그, IoT 같이 정합성보다 속도, 확장성이 중요한 서비스에 더 적합

#### 스토리지 & 메모리 효율
- MySQL
  - 고정 스키마 기반 -> 저장 공간 효율적
  - 압축, 파티셔닝 등 최적화 기능이 예전부터 사용됨
- MongoDB
  - BSON 저장 구조 -> 키 이름 중복으로 공간 낭비 가능
  - 대규모 데이터일수록 스토리지 효율이 떨어질 수 있음

#### 운영 & 안정성
- MySQL
  - 오래된 기술 -> 운영 경험, 모니터링 툴, 커뮤니티 풍부
  - 안정성이 검증되어 공공/금융 등에서 표준
- MongoDB
  - 클라우드 네이티브, 분산 환경에서 강점
  - 그러나 소규모/단일 서버에서는 오히려 과한 스펙 + 운영 복잡도 높아짐

---

### 정리
- MySQL 강점
  - 정형 데이터 / 금융, 결제 같은 트랜잭션 무결성 필수 서비스
  - SQL 표준 기반
  - 안정성, 운영 경험 풍부
- MongoDB 강점
  - 빠른 개발 속도, 유연한 스키마, 다양한 데이터 타입
  - 수평 확장(Sharding) & 글로벌 분산 클러스터
  - 현대 웹/모바일, 스타트업에서 빠른 제품 개발에 최적
- 실무에서는 **혼합 사용** 많음 (ex. 주문 내역은 MySQL, 로그/이벤트는 MongoDB)

---

## 3. 참고/추가 자료 (References)
- [MongoDB vs MySQL 비교](https://www.mongodb.com/compare/mongodb-mysql)

---

## 4. 내일/다음에 볼 것 (Next Steps)
- DB 샤딩이란... 
