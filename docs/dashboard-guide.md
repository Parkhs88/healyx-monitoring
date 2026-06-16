# 대시보드 가이드

HEALYX 모니터링 Grafana 대시보드 탭별 설명 및 설계 포인트입니다.

---

## 데이터 소스 구성

| 소스 | 용도 |
|------|------|
| Prometheus | Spring Boot Actuator 메트릭 수집 (API 요청/응답/에러) |
| AWS CloudWatch | EC2, RDS, ALB 인프라 메트릭 및 Billing 비용 수집 |
| CloudWatch Logs | 에러 로그 수집 (`LoggingFilter.java` → Docker awslogs → CloudWatch) |

---

## 쿼리 설계 원칙

Prometheus 패널은 목적에 따라 쿼리 타입을 구분해서 적용했습니다.

| 패널 유형 | 쿼리 타입 | 적용 이유 |
|-----------|-----------|-----------|
| Stat / Bar / Pie (단일 수치) | `instant=true` | 현재 시점의 값 하나만 필요하기 때문 |
| Time Series (시계열 추이) | `range=true` | 시간대별 변화 흐름을 표현하기 위함 |

---

## 1. Overview

전체 시스템 상태를 한눈에 파악할 수 있는 요약 대시보드입니다.

| 패널 | 데이터 소스 | 설명 |
|------|-------------|------|
| 서버 상태 | CloudWatch (ALB HealthyHostCount) | 백엔드 EC2 서버 정상 여부 |
| 운영 서버 수 | CloudWatch (ALB HealthyHostCount) | 현재 운영 중인 서버 수 |
| EC2 CPU 사용률 추이 | CloudWatch (EC2 CPUUtilization) | CPU 사용률 시계열 그래프 |
| API 평균 응답시간 | Prometheus | 전체 API 평균 응답시간 |
| API 성공률 추이 | Prometheus | 시간대별 API 성공률 변화 |
| RDS DB 연결 수 | CloudWatch (RDS DatabaseConnections) | 현재 RDS 연결 수 |
| API별 에러 수 TOP 5 | Prometheus | 에러 발생 상위 5개 API |
| 오류 수 | Prometheus | 전체 4XX/5XX 오류 건수 |
| 월 누적 비용 | CloudWatch (Billing EstimatedCharges) | 이번 달 누적 AWS 비용 |

---

## 2. Infra (인프라)

EC2, RDS, ALB 인프라 리소스를 CloudWatch 메트릭으로 모니터링합니다.
인프라 지표는 모두 CloudWatch에서 수집하므로 Prometheus 없이도 동작합니다.

| 패널 | CloudWatch 메트릭 | 설명 |
|------|-------------------|------|
| EC2 상태 | EC2/StatusCheckFailed | 인스턴스 상태 체크 실패 여부 |
| 운영 서버 수 | ALB/HealthyHostCount | 현재 정상 운영 중인 서버 수 |
| CPU 사용률 | EC2/CPUUtilization | 현재 CPU 사용률 |
| EC2 CPU 사용률 추이 | EC2/CPUUtilization | 시간대별 CPU 사용률 변화 |
| Network In / Out | EC2/NetworkIn, NetworkOut | 네트워크 인바운드/아웃바운드 트래픽 |
| ALB 요청 수 | ALB/RequestCount | Application Load Balancer 요청 수 |
| RDS 연결 수 | RDS/DatabaseConnections | MySQL RDS 현재 연결 수 |

---

## 3. Service (서비스)

Spring Boot Actuator의 `http_server_requests_seconds` 메트릭을 기반으로
API 요청 현황 및 응답 성능을 모니터링합니다.

**설계 포인트:** URI를 한국어 기능명으로 매핑하여 가독성을 높였습니다.
예) `/api/auth/login` → 로그인, `/api/hospital` → 병원 검색

| 패널 | 쿼리 타입 | 설명 |
|------|-----------|------|
| API 성공률 | instant | 전체 요청 중 2XX 응답 비율 |
| 평균 응답시간 | instant | 전체 API 평균 응답시간 (ms) |
| 분당 요청 수 | instant | 현재 분당 API 요청 건수 |
| 활성 API 수 | instant | 현재 호출되고 있는 고유 API 수 |
| 최근 5분 API 요청 수 추이 | range | 최근 5분간 요청 수 변화 |
| API 성공률 추이 | range | 시간대별 성공률 변화 |
| 평균 응답시간 추이 | range | 시간대별 응답시간 변화 |
| API 호출 빈도 Top 10 | instant | 가장 많이 호출된 API 상위 10개 |
| API 사용 비율 | instant | API별 호출 비율 파이 차트 |

---

## 4. Errors (에러 로그)

`LoggingFilter.java`가 모든 API 요청의 상태코드를 기록하고,
Docker `awslogs` 드라이버를 통해 CloudWatch Logs로 전송된 데이터를 시각화합니다.

**설계 포인트:** 5XX(서버 에러)와 4XX(클라이언트 에러)를 분리해서 모니터링합니다.
로그 레벨도 5XX → ERROR, 4XX → WARN으로 구분하여 심각도를 구별했습니다.

| 패널 | 쿼리 타입 | 설명 |
|------|-----------|------|
| 4XX 에러 수 | instant | 클라이언트 에러 발생 건수 |
| 5XX 에러 수 | instant | 서버 에러 발생 건수 |
| 에러 비율 | instant | 전체 요청 중 에러 비율 |
| 최다 에러 건수 | instant | 에러가 가장 많이 발생한 API |
| 미분류 요청 에러 | range | 분류되지 않은 에러 요청 추이 |
| 4XX / 5XX 에러 추이 | range | 시간대별 에러 발생 추이 |
| API별 에러 수 TOP 10 | instant | 에러 발생 상위 10개 API |
| API별 에러 상세 | - | API별 상세 에러 로그 조회 |

---

## 5. Cost (비용)

AWS 서비스별 운영 비용을 모니터링합니다.

**설계 포인트:** AWS Billing 메트릭은 `us-east-1` 리전에서만 수집 가능합니다.
나머지 인프라 메트릭(`ap-northeast-2`)과 리전을 분리하여 datasource를 구성했습니다.

| 패널 | 설명 |
|------|------|
| 월 누적 비용 | 이번 달 누적 총 비용 |
| 예상 월 비용 | 현재 추세 기준 월말 예상 비용 |
| 비용 상태 | 비용 정상/주의 여부 |
| 예산 사용률 | 설정 예산 대비 사용 비율 |
| 월 누적 비용 추이 | 시간대별 비용 변화 |
| 서비스별 비용 비율 | EC2, ALB 등 서비스별 비용 파이 차트 |
