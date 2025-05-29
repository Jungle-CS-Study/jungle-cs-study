# 대용량 트래픽 설계 스터디 자료

## 1. 트래픽 측정 기준

대용량 트래픽 설계를 위해선 **현재 시스템의 트래픽 수준**을 정확히 측정해야 합니다. 이를 위해 주로 사용하는 지표는 다음과 같습니다:

- **QPS (Queries Per Second) / TPS (Transactions Per Second)**
    - 초당 처리 가능한 요청(쿼리) 또는 트랜잭션 수.
    - API 서버, 웹 서버 등 **요청 처리 성능**을 파악할 때 핵심 지표.
- **RPS (Requests Per Second)**
    - 초당 처리되는 HTTP 요청 수.
    - 웹 서비스에서의 트래픽 상황을 파악할 때 주로 사용.
- **Bandwidth (대역폭)**
    - 초당 데이터 전송량 (단위: Mbps, Gbps).
    - 동영상 스트리밍, 파일 다운로드 서비스 등 **대용량 데이터 처리**에 중요한 지표.
- **Concurrent Users (동시 접속자 수)**
    - 특정 시점에 동시에 접속해 있는 사용자 수.
    - 실시간 서비스(게임, 라이브 스트리밍)에서는 특히 중요.
- **Latency (지연 시간)**
    - 요청 → 응답까지 걸리는 시간.
    - 평균, P95, P99 지표를 함께 살펴야 함.
- **Error Rate (에러율)**
    - 전체 요청 중 실패한 요청의 비율.
    - 트래픽 증가로 인한 장애 여부를 판단하는 데 필요.

> 💡 **트래픽 측정**은 단순히 요청 수뿐 아니라, **데이터 크기**와 **서비스 특성**을 함께 고려해야 함. 예를 들어, API 서비스와 스트리밍 서비스의 단위 트래픽은 다른 기준을 가져야 함.

---

## 2. 트래픽 측정을 위한 오픈 API 및 도구

트래픽 측정을 위해 사용할 수 있는 **오픈 API** 및 **도구**는 아래와 같습니다:

### 2.1. Prometheus + Grafana

- **Prometheus**: 시계열 데이터 수집 및 저장을 위한 오픈 소스 모니터링 시스템.
- **Grafana**: 데이터 시각화 도구로, Prometheus와 연동해 실시간 대시보드를 구성.
- **측정 가능한 항목**: QPS, 응답 시간, 에러율, 시스템 리소스 사용량 등.
- **사용 예시**: 서버/애플리케이션에 `/metrics` 엔드포인트 노출 → Prometheus에서 scrape → Grafana 대시보드 생성.

### 2.2. Postman API Monitoring

- **Postman**: API 테스트 도구로, 모니터링 기능을 통해 API 응답 시간, 상태 코드 등을 주기적으로 체크 가능.
- **특징**:
    - API 호출 상태 모니터링 및 테스트 결과 저장.
    - 응답 시간, 상태 코드 기준으로 알림 설정 가능.

### 2.3. Uptrace

- **Uptrace**: OpenTelemetry 기반의 분산 추적 및 메트릭 수집 도구.
- **특징**:
    - 분산 시스템의 API 호출 흐름 추적.
    - 실시간 모니터링 및 메트릭 시각화.

### 2.4. 기타 도구

| 도구        | 특징                             | 주요 활용                       |
|-------------|----------------------------------|--------------------------------|
| **OpenTelemetry** | 표준화된 분산 추적, 메트릭 수집 | Prometheus, Grafana와 통합 사용 가능 |
| **CloudWatch (AWS)** | AWS 리소스 및 애플리케이션 모니터링 | AWS 기반 시스템 트래픽 측정     |
| **Datadog**   | 클라우드 기반 통합 모니터링     | SaaS 형태의 고급 모니터링 플랫폼  |
| **New Relic**  | 어플리케이션 성능 모니터링(APM) | 대시보드, 알림, 분석 기능 제공    |

---

## 3. 트래픽 측정 단계별 실습 가이드 (예시: Prometheus + Grafana)

### 3.1. Prometheus + Grafana로 트래픽 측정하기

1. **Prometheus 설치 및 설정**
   - Prometheus를 설치 후 `prometheus.yml` 설정 파일에 측정 대상(타겟) 추가.
   - 예시 설정:
     ```yaml
     scrape_configs:
       - job_name: 'web_app'
         static_configs:
           - targets: ['localhost:8080']
     ```
2. 애플리케이션에 메트릭 엔드포인트 추가
   - 예: /metrics 엔드포인트를 통해 Prometheus가 메트릭을 수집할 수 있도록 구현.

3. Grafana 설치 및 대시보드 구성
   - Grafana를 설치 후 Prometheus를 데이터 소스로 연결.
   - QPS, TPS, 응답 시간, 에러율 등의 메트릭을 대시보드에 추가하여 시각화.

4. 알림 설정
   - Grafana에서 임계치(Threshold) 설정 → 이메일, Slack 등으로 알림 전송.

---
### 3.2. Postman으로 API 모니터링하기
1. Postman Collection 생성
   - 모니터링할 API 요청을 Collection에 추가.

2. 테스트 스크립트 작성
   - 각 요청에 대한 상태 코드, 응답 시간 검증 코드 작성.\ 
       ```javascript
       pm.test("Status code is 200", function () {
         pm.response.to.have.status(200);
       });
       ```
3. Monitor 생성 및 실행
   - Postman에서 "Monitor" 기능을 통해 Collection을 주기적으로 실행.
   - 주기(분, 시간, 일 단위) 및 알림 설정 가능.

4. 모니터링 결과 확인
   - Postman 대시보드에서 응답 시간, 상태 코드, 실패 요청 등을 확인.


## 4. 트래픽 규모별 구분 기준

| 구분                    | QPS/TPS (대략)     | Bandwidth (대역폭) | 예시                         |
| --------------------- | ---------------- | --------------- | -------------------------- |
| 저용량 (Low Traffic)     | 1 \~ 100 TPS     | \~10 Mbps 이하    | 개발/테스트 서버, 내부 시스템          |
| 일반용량 (Normal Traffic) | 100 \~ 1,000 TPS | \~100 Mbps 이하   | 중소형 서비스, 쇼핑몰, 커뮤니티         |
| 대용량 (High Traffic)    | 1,000 TPS 이상     | 1 Gbps 이상       | 대형 포털, SNS, 게임 서버, OTT 서비스 |
 
  
 
---

## 5. 트래픽 증가 단계별 고려사항
| 고려 항목      | 저용량 (Low)        | 일반용량 (Normal)              | 대용량 (High)                                   |
| ---------- | ---------------- | -------------------------- | -------------------------------------------- |
| **인프라**    | 단일 서버, 단일 DB     | Auto-Scaling, Read Replica | 분산 시스템, 글로벌 클러스터                             |
| **데이터베이스** | 단일 DB, 단일 인스턴스   | 인덱스 최적화, 쿼리 튜닝             | Sharding, 분산 DB (Cassandra, DynamoDB 등)      |
| **캐싱**     | 없음               | 메모리 캐시 (Redis, Memcached)  | CDN, Global Cache, Edge Server               |
| **트래픽 분산** | 단일 Load Balancer | Regional Load Balancer     | Global Load Balancer, Anycast, GSLB          |
| **모니터링**   | 기본 모니터링          | 알림, 대시보드 구축                | 실시간 모니터링, Chaos Testing                      |
| **배포 전략**  | 수동 배포            | CI/CD, Blue-Green          | Canary, Feature Flag, Rollback 자동화           |
| **보안**     | 기본 인증/인가         | WAF, API Key               | Rate Limit, DDoS 보호 (Cloudflare, AWS Shield) |
| **비용 관리**  | 비용 고려 X          | 리소스 최적화                    | 클라우드 비용 모니터링, 비용 예측 및 제어                     |

  
---

## 마무리
- 트래픽 측정 → 구분 → 단계별 설계 고려사항을 이해하면, 트래픽 급증 상황에서도 안정적인 서비스를 유지할 수 있음.
- 트래픽 기준은 정해진 "절대 수치"가 아니라 서비스 특성과 비즈니스 목표에 맞춰 유연하게 정의해야 함.
- 대용량 트래픽 설계는 단순히 서버를 늘리는 게 아니라, 아키텍처 설계와 비용 효율성, 성능 최적화까지 모두 고려해야 하는 종합적인 문제임.

