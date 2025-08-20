# 📚 Zero-Downtime Deployment (무중단 배포)

---

## 1. 주제/키워드
- 무중단 배포에 대해 알아보자! (੭*ˊᵕˋ)੭*  

## 📑 목차

- [📚 Zero-Downtime Deployment (무중단 배포)](#-zero-downtime-deployment-무중단-배포)
  - [1. 주제/키워드](#1-주제키워드)
  - [📑 목차](#-목차)
  - [2. 핵심 요약 (Summary)](#2-핵심-요약-summary)
    - [무중단 배포가 필요한 이유](#무중단-배포가-필요한-이유)
    - [무중단 배포 전략의 종류](#무중단-배포-전략의-종류)
      - [Blue-Green Deployment](#blue-green-deployment)
      - [Rolling Update](#rolling-update)
      - [Canary Release](#canary-release)
      - [Traffic Mirroring(Shadow Traffic)](#traffic-mirroringshadow-traffic)
      - [Canary + Mirroring](#canary--mirroring)
    - [무중단 배포 도입 시 고려사항](#무중단-배포-도입-시-고려사항)
    - [Nginx + Uvicorn + PM2](#nginx--uvicorn--pm2)
      - [Nginx](#nginx)
      - [Uvicorn (ASGI Server)](#uvicorn-asgi-server)
      - [PM2 (Process Manager 2)](#pm2-process-manager-2)
    - [Canary 배포 예시 (토스페이먼츠 SDK 사례)](#canary-배포-예시-토스페이먼츠-sdk-사례)
      - [카나리 배포 요구사항](#카나리-배포-요구사항)
      - [초기 설계 (CloudFront + Lambda@Edge + S3)](#초기-설계-cloudfront--lambdaedge--s3)
      - [User-Based Canary (개선된 설계)](#user-based-canary-개선된-설계)
      - [아키텍처 요약](#아키텍처-요약)
      - [효과](#효과)
      - [정리](#정리)
    - [TOSS Bank 무중단 배포 사례](#toss-bank-무중단-배포-사례)
      - [배포 전략](#배포-전략)
      - [문제 상황 1  - 버전 스위칭 문제](#문제-상황-1----버전-스위칭-문제)
      - [해결책 1 – Sticky Canary](#해결책-1--sticky-canary)
      - [문제 상황 2 – 모니터링/롤백 한계](#문제-상황-2--모니터링롤백-한계)
      - [해결책 2 – Auto Canary (자동 롤백)](#해결책-2--auto-canary-자동-롤백)
      - [정리](#정리-1)
  - [3. 참고/추가 자료 (References)](#3-참고추가-자료-references)
  - [4. 내일/다음에 볼 것 (Next Steps)](#4-내일다음에-볼-것-next-steps)

---

## 2. 핵심 요약 (Summary)

### 무중단 배포가 필요한 이유
- 사용자는 언제나 사용 가능한 서비스 이용을 기대 -> 서비스 중단 시 **신뢰도 하락 + 이탈**로 이어짐
- 트래픽이 많은 서비스일수록 **다운타임이 곧 금전적 손실**
- 기존 배포 방식(서버 재시작 -> 새 버전 반영)은 짧게라도 장애 발생
- 따라서, 배포 중에도 서비스가 계속 응답할 수 있도록 **무중단 배포 전략** 필요

---

### 무중단 배포 전략의 종류

#### Blue-Green Deployment
- **Blue**(현재 운영 버전), **Green**(새 버전) 두 개의 환경을 준비
- Green에 새 버전 배포 -> 테스트 트래픽으로 정상 동작 확인 후 실 트래픽 스위칭
- 장점
  - 롤백 쉬움 (트래픽만 다시 Blue로 돌리면 됨)
  - 완전히 분리된 환경에서 안전하게 테스트 가능
- 단점
  - 인프라 2배 필요 (비용 높음)
  - DB 스키마 변경 시 양쪽 버전 호환성 고려 필요

#### Rolling Update
- 여러 서버 중 일부만 순차적으로 새 버전으로 교체
- ex. 10대 서버 -> 2대씩 새 버전으로 교체 -> 전체 교체 완료
- 장점
  - 인프라 비용 절감 (추가 서버 불필요)
  - 점진적 교체로 안정성 확보
- 단점
  - 배포 중 장애 발생 시 롤백 복잡
  - 새 버전/구버전이 동시에 운영되므로 호환성 이슈 발생 가능

#### Canary Release
- 전체 트래픽 중 일부만 새로운 버전에 전달
- ex. 1% 사용자만 Canary 버전 사용 -> 점진적으로 확대
- 장점
  - 실제 사용자 트래픽 기반 테스트 가능
  - 문제 발생 시 영향 최소화
- 단점
  - **일부 트래픽만 커버 -> 전체 이슈 탐지 어려움**
  - 트래픽 분산/모니터링 구조 필요

#### Traffic Mirroring(Shadow Traffic)
- **실제 사용자 요청을 복제**해서 새 버전에도 전달 (응답은 discard)
- 실제 응답에는 영향 X, 새 버전의 동작만 관찰 가능
- 장점
  - 실제 트래픽 기반으로 테스트 가능
  - 문제 발생해도 사용자에게 영향 안감
- 단점
  - **리소스 2배 사용**
  - 쓰기 요청 복제 시 데이터 무결성 깨질 위험
    - 중복 요청으로 인한 문제(ex. 주문 두번)
  - DB, 외부 API 호출 같은 side effect 방어 필요
    - 같은 DB를 사용하면 원래 횟수보다 많은 커밋으로 데이터 훼손

#### Canary + Mirroring
- 새로운 버전은 **Mirroring**으로 안정성 검증
- 이후 **Canary Release**로 점진적 실 사용자 적용
- **실제 트래픽 기반 검증 + 제한적 적용**을 함께 사용

---

### 무중단 배포 도입 시 고려사항
- **비용**
  - Blue-Green, Mirroring은 서버 리소스를 거의 2배로 요구
- **DB 호환성**
  - DB 스키마 변경 시 구버전/신버전 동시 지원 필요
  - Backward Compatibility (하위 호환성) 확보 -> Feature Flag 활용
    - Backward Compatibility: 새 버전의 소프트웨어가 이전 버전에서 사용되던 기능/데이터/프로토콜을 그대로 지원하는 것 
    - Feature Flag: 코드에 새로운 기능을 넣더라도, 플래그(스위치) 값에 따라 기능 on/off 제어할 수 있는 기법
- **모니터링**
  - Latency, Error Rate, CPU, Memory 등 지표 기반 자동 롤백
- **트래픽 특성**
  - Canary는 소량 트래픽에서 발생하지 않는 edge case를 놓칠 수 있음
  - Mirroring은 read-heavy 서비스에 적합, write-heavy는 주의 필요
- **배포 자동화**
  - Kubernetes, Argo Rollouts, Spinnaker 같은 툴 활용

### Nginx + Uvicorn + PM2
- 내가 사용했던 배포 방식에 대해 공부해보자~

#### Nginx
- **Reverse Proxy + Load Balancer**
- 여러 애플리케이션 인스턴스 앞단에서 트래픽 분산
- 무중단 배포 시 새 버전 서버로 트래픽 라우팅을 점진적으로 변경 가능 
- 무중단 배포 방법
  - `upstream` 블록 내 서버 그룹 교체
  - Canary: 일부 트래픽만 신규 서버에 전달
  - Mirroring: `mirror` 모듈 활용해 요청 복제 가능
- Reverse Proxy (리버스 프록시)
  - 클라이언트가 직접 백엔드 서버에 붙는 게 아니라, 중간에 서버(프록시 서버)가 대신 요청을 받아주는 구조
  - 클라이언트 입장에서는 프록시 서버가 최종 서버처럼 보이지만, 실제로는 내부 서버들로 요청을 역으로 전달(reverse) 해주기 때문에 Reverse Proxy라고 함

#### Uvicorn (ASGI Server)
- Python FastAPI 같은 ASGI 서버 실행용
- **멀티 프로세스 실행**
  - `--workers` 옵션으로 여러 프로세스 띄워서 병렬 처리
  - systemd나 gunicorn과 조합하면 **graceful reload** 지원
  - Graceful reload: 서버 프로세스를 강제 종료(kill)하지 않고, 기존 연결은 다 처리한 뒤에만 종료하고 새 프로세스로 교체하는 방식
- **Nginx와 조합**
  - Nginx가 연결 관리, 헬스 체크 -> Uvicorn은 비동기 처리에 집중
  - 배포 시 Uvicorn 프로세스만 순차적으로 재시작하면 **Rolling Update 효과**
- Health Check
  - 서버나 애플리케이션이 정상 동작 중인지 주기적으로 확인하는 과정
  - 보통 로드밸런서, 오케스트레이터(K8s 등), 모니터링 툴에서 사용

#### PM2 (Process Manager 2)
- Node.js 앱용 프로세스 매니저
- **무중단 배포 지원**
  - `pm2 reload app` -> 기존 프로세스는 현재 처리 중인 요청을 끝까지 처리하고, 새로운 프로세스가 다음 요청을 이어서 처리
  - 유저는 다운타임 없이 계속 서비스 이용 가능
- **클러스터 모드**
  - `pm2 start app.js -i max` -> CPU 코어 수만큼 프로세스 실행, 내부 라운드로빈 분산 처리
  - 라운드 로빈 (Round Robin): 여러 서버(또는 프로세스)가 있을 때, 요청을 순서대로 골고루 분배하는 방식
- 장점
  - 모니터링/로그 관리 내장
  - 장애 시 자동 재시작 (self-healing)
  - Rolling Update, Blue-Green 없이도 소규모 서비스에서 무중단 배포 쉽게 구현 가능
- 한계
  - Node.js 한정 (Python/Java는 gunicorn, systemd, Kubernetes 등 다른 툴 필요)
  - DB 스키마 변경 같은 대규모 배포 시엔 다른 방식이 필요

---

### Canary 배포 예시 (토스페이먼츠 SDK 사례)
- JavaScript SDK는 결제 시작점이므로 안정성이 매우 중요
- 기존 방식은 변경 사항이 **모든 가맹점에 동시에 반영** -> 작은 버그도 전면 장애로 확대
- 이에 따라 배포 후 개발자들의 불안감, 모니터링 부담이 컸음
- 해결 방법: **Canary Release 도입**
  - 변경 사항을 일부 사용자에게만 적용 -> 문제 발견 시 영향 최소화
  - 실제 사용 환경에서 안정성 검증 가능

#### 카나리 배포 요구사항
- **균일한 비율**로 모든 환경(브라우저, 언어, 지역, 기기)에 카나리 버전 제공
  - 특정 유저 그룹만 카나리 적용되면 문제 탐지가 어렵기 때문
- **요청에 대한 응답이 CDN 캐시** (적중률 99% 이상) -> JS SDK는 초기 로딩 속도가 중요
- **실시간 롤백 + 비율 조정 가능해야함** -> 문제가 생기면 즉시 안정화 버전으로 되돌리기

#### 초기 설계 (CloudFront + Lambda@Edge + S3)
- S3: stable/canary 버전 SDK 파일 저장
- CloudFront + Lambda@Edge: 요청 시 weight 기반으로 버전 분기
- 문제점
  - **캐시 활성화 시**: 캐시에 한 번 canary가 들어가면 캐시 만료 시간 동안 모든 요청이 같은 버전 제공 -> 비율 제어 불가능
  - **캐시 비활성화 시**: weight 기반 랜덤 분배는 가능하지만 모든 요청이 Lambda@Edge 실행 -> 비용 폭증 + 응답 지연

#### User-Based Canary (개선된 설계)
- **JA3 Fingerprint** 기반 사용자 구분 (브라우저/지역 등 특정 환경에 치우치지 않음)
- JA3를 단순화해 X-Toss-Cohort 헤더(0\~9)로 변환 -> 캐시 키로 사용
- 이 Cohort 값과 Canary weight를 비교해 stable/canary 버전 제공
- 장점
  - 캐싱 적중률 유지 + 카나리 비율 제어 가능
  - 사용자별로 **일관된 버전 제공** (쿠키 저장 활용)
  - Canary weight 실시간 변경 가능 -> 롤백 즉시 반영

---

#### 아키텍처 요약
1. **요청** -> CloudFront
2. CloudFront Function이 JA3 Fingerprint -> X-Toss-Cohort 헤더로 변환
3. Lambda@Edge가 Cohort와 Canary weight 비교 -> stable/canary 선택
4. 결과는 CDN 캐시에 저장되어 빠르게 응답
5. Cohort 값은 쿠키에 기록해 사용자가 항상 같은 버전 경험

---

#### 효과
- **개발자 배포 경험 개선**: 전체 사용자에게 바로 노출되지 않으므로 불안감 감소
- **안전한 롤백**: Canary에서 문제 발견 시 즉시 stable로 회귀 가능
- **비용 효율성**: 캐시 활용으로 Lambda@Edge 실행 최소화, 요청당 지연 거의 없음
- 1천만 건 SDK 요청 시 추가 비용 약 **$0.6** 수준

---

#### 정리
- **기존**: 모든 사용자에 한 번에 배포 -> 리스크 큼
- **Canary 적용 후**: 일부 사용자에게만 점진 적용 -> 문제 조기 발견 + 안전한 배포 가능
- User-based Cohort + CDN 캐시 전략으로 **성능, 비용, 안정성 모두 고려**

---

### TOSS Bank 무중단 배포 사례
- CI/CD 도입으로 배포 주기 단축, 배포 전략 중요성 확대
- 토스뱅크는 주로 **Rolling Update**와 **Canary Release**를 활용

#### 배포 전략
- **Rolling Update**
  - 쿠버네티스 기본 배포 방식
  - 서버를 하나씩 교체 (신규 서버 -> 트래픽 할당 -> 구버전 서버 제거)
  - 장점: 단순, 기본 기능으로 가능
  - 단점: 배포 중간에 **멈출 수 없음**, 장애 발생 시 롤백 과정이 길고, 모든 유저가 장애 경험
- **Canary Release**
  - Istio 등 네트워크 레벨 트래픽 제어 도구 필요
  - 신규 워크로드에 소량 트래픽(예: 1%)부터 할당 -> 점진적으로 확대
  - 장점: 장애 발생 시 **즉시 롤백 가능**
- 토스뱅크: **대고객 서비스에는 Canary, 내부 서비스에는 Rolling Update** 적용

---

#### 문제 상황 1  - 버전 스위칭 문제
- 신규 기능(예: 금융거래 종합보고서) 배포 시 **버전 스위칭 문제** 발생
  - 첫 요청은 신규 버전(탭 노출) -> 이후 요청은 구버전(탭 없음) -> **Page Not Found 오류**
- 원인: 요청 단위로 트래픽이 분기되어 같은 유저가 **구버전/신버전을 오가며 문제 발생**

---

#### 해결책 1 – Sticky Canary
- **Sticky Session 개념 차용**: 동일 유저의 요청은 항상 동일 버전으로 고정
- 구현 방식
  - Gateway에서 유저 고유 키(hash) 기반으로 cohort 값 생성
  - Cohort 값과 Canary 비율을 비교해 **헤더에 버전 정보 부여**
  - Istio VirtualService가 헤더 기반 라우팅 (v1/v2 고정)
- 효과
  - 한 번 신규 버전에 진입한 유저는 계속 신규 버전만 사용
  - 배포 중에도 기능 불일치 문제 방지

---

#### 문제 상황 2 – 모니터링/롤백 한계
- Canary는 모니터링이 핵심인데, 사람이 직접 로그/메트릭을 체크해야 해서 **탐지 실패 가능성**
- MSA 환경(토스뱅크는 400+ 서비스)에서는 **한 서비스의 문제가 연관 서비스 장애로 전파**될 수 있음
- A 서비스 배포 시 A는 정상, B/C/D 서비스는 장애 -> 원인 추적 어려움

---

#### 해결책 2 – Auto Canary (자동 롤백)
= 오픈소스(Spinnaker, Flagger 등) 대신 자체 시스템 도입
- 핵심 기능: **Auto Rollback**
- Viva System(Web 관리도구)에 Auto Rollback 스레드를 두어 조건 체크
- 조건
  - 에러 로그가 임계치 이상 증가
  - 로그 유실/지연 발생 (ElasticSearch 문제 감지)
- 조건 만족 시 -> **해당 버전 트래픽 즉시 차단, 롤백**
- 보수적 운영: 연관 없는 문제라도 롤백 -> 고객 신뢰 우선

---

#### 정리
- **Canary 자체보다 중요한 건 모니터링 + 롤백 체계**
- 장애 최소화를 위해 복잡한 툴 도입보다 **간단하고 빠른 MVP 해결책**이 효과적
- Sticky Canary, Auto Rollback을 통해 배포 신뢰성과 안정성 강화

--- 

## 3. 참고/추가 자료 (References)
- [Canary Release based on Ingress-Nginx](https://kubernetes.github.io/ingress-nginx/examples/canary/)
- [토스 기술 블로그 - 프론트엔드 배포 시스템의 진화 (1) - 결제 SDK에 카나리 배포 적용하기](https://toss.tech/article/engineering-note-9)
- [토스ㅣSLASH 24 - 생산성과 안정성 모두 잡는 마스터키, Canary 배포 개선기](https://www.youtube.com/watch?v=ApNj9MZU7Ak)

---

## 4. 내일/다음에 볼 것 (Next Steps)
- 사실 pm2 기반 무중단 배포 이외에는 사용해본 적이 없어서.. 공부하면서 굉장히 새롭고 재밌었다~
- 다음에는 실제 배포 시 발생하는 문제점에 대해 더 많이 고민해보고 싶다