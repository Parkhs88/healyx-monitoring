# 모니터링 EC2 설치 가이드

Ubuntu 22.04 기준 · Docker + Docker Compose 사용

---

## 1. AWS 사전 설정

### IAM 사용자 및 역할 생성
- IAM 사용자: `healyx-observer-role` 역할(role) 생성
- CloudWatch 읽기 권한 부여

### 보안 그룹 설정
- 보안 그룹명: `healyx-observer-sg`
- 아웃바운드: 모두 허용
- 백엔드 EC2 보안 그룹에 모니터링 EC2의 8080 포트 인바운드 허용 추가
  - `healyx-observer-sg`를 가진 EC2만 백엔드 8080 포트 접근 허용

### EC2 인스턴스 생성
- 인스턴스명: `healyx-observer`
- 인스턴스 타입: t3.micro
- OS: Ubuntu 22.04
- 키페어: `healyx-observer-key`

---

## 2. EC2 접속

```bash
ssh -i C:\key\healyx-observer-key.pem ubuntu@[EC2 IP]
```

---

## 3. Docker 설치

```bash
# 패키지 업데이트
sudo apt update && sudo apt upgrade -y

# Docker 설치
sudo apt install docker.io -y

# Docker 버전 확인
docker --version

# Docker 시작 및 부팅 시 자동 실행 등록
sudo systemctl start docker
sudo systemctl enable docker

# sudo 없이 Docker 사용 설정
sudo usermod -aG docker $USER

# 재접속 (권한 적용)
exit
ssh -i C:\key\healyx-observer-key.pem ubuntu@[EC2 IP]

# 정상 실행 확인
docker ps
```

---

## 4. Docker Compose 설치

```bash
# Docker Compose 설치
sudo apt-get install -y docker-compose

# 버전 확인
docker compose version
```

---

## 5. 모니터링 폴더 생성 및 설정 파일 작성

```bash
# 폴더 생성 및 이동
mkdir ~/monitoring
cd ~/monitoring
```

`docker-compose.yml` 과 `prometheus.yml` 파일을 작성합니다.  
내용은 본 레포지토리의 `prometheus/` 폴더를 참고하세요.

---

## 6. 실행

```bash
# 모니터링 폴더 안에서 실행
sudo docker compose up -d

# 상태 확인
sudo docker compose ps
```

---

## 7. 접속 확인

브라우저에서 아래 주소로 접속:

```
http://[EC2 IP]:3000
```

Grafana 초기 로그인: `admin` / `admin`

---

## 8. Grafana 설정

1. AWS CloudWatch datasource 연결
2. Prometheus datasource 연결
3. `grafana/dashboards/` 폴더의 JSON 파일 Import
