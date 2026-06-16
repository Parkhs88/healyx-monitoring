# Error Log 수집 구성

Grafana Error Log 대시보드에서 실시간 에러 로그를 조회하기 위한 백엔드 설정 파일들입니다.

---

## 데이터 흐름

```
Spring Boot API 요청
        ↓
LoggingFilter.java (상태코드별 로그 기록)
        ↓
logback-spring.xml (JSON 형식으로 변환)
        ↓
Docker awslogs 드라이버
        ↓
AWS CloudWatch Logs (healyx-backend 로그 그룹)
        ↓
Grafana Error Log 대시보드
```

---

## 파일 설명

### LoggingFilter.java
모든 API 요청/응답을 가로채서 상태코드별로 로그를 자동 기록하는 필터입니다.

- 5XX 서버 에러 → `ERROR` 레벨
- 4XX 클라이언트 에러 → `WARN` 레벨
- 2XX 정상 응답 → `INFO` 레벨

기록 항목: `api_path`, `method`, `status_code`, `response_time_ms`

기존 Controller, Service 코드 수정 없이 필터만 추가하여 모든 API에 자동 적용됩니다.

### logback-spring.xml
운영 환경(`prod`)에서 로그를 JSON 형식으로 출력하는 설정 파일입니다.

CloudWatch Logs Insights가 `api_path`, `status_code` 등의 항목을 개별 필드로 파싱하려면 JSON 형식이 필요합니다.

- `prod` 환경 → JSON 형식 출력 (CloudWatch 분석 가능)
- 로컬 환경 → 기존 텍스트 형식 유지 (개발 편의성 유지)
