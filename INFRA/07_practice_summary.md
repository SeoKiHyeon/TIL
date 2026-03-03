# 실습 정리 - Docker 기초부터 EC2 배포까지

## 실습 순서 요약

```
1. Docker 기본 명령어 (hello-world, nginx)
2. Dockerfile 작성 → 커스텀 이미지 빌드
3. Docker Compose로 멀티 컨테이너 실행
4. Volume으로 데이터 영속성 확인
5. .env 파일로 환경변수 분리
6. AWS EC2 인스턴스 생성 및 배포
```

---

## 실행했던 명령어 전체 정리

### Docker 기본

```bash
# Docker 버전 및 상태 확인
docker --version
docker info

# 첫 번째 컨테이너 실행
docker run hello-world

# 이미지 목록 확인
docker images

# 실행 중인 컨테이너 목록
docker ps

# 종료된 것 포함 전체 컨테이너 목록
docker ps -a

# nginx 포트 포워딩 실행
docker run -d -p 8888:80 nginx

# 특정 이미지로 실행된 컨테이너 강제 삭제
docker rm -f $(docker ps -q --filter ancestor=my-nginx)

# 실행 중인 모든 컨테이너 강제 삭제
docker rm -f $(docker ps -aq)
```

### Dockerfile 빌드 및 실행

```bash
# 이미지 빌드 (-t: 이름 지정, .: 현재 폴더의 Dockerfile 사용)
docker build -t my-nginx .

# 빌드한 이미지로 컨테이너 실행
docker run -d -p 8888:80 my-nginx
```

### Docker Compose

```bash
# 컨테이너 전체 시작 (백그라운드)
docker compose up -d

# Dockerfile 변경 시 재빌드 후 시작
docker compose up -d --build

# 실행 중인 서비스 확인
docker compose ps

# 전체 중지 및 컨테이너 삭제
docker compose down

# 로그 확인
docker compose logs -f
```

### MySQL 컨테이너 접속

```bash
# 컨테이너 내부 MySQL 접속
docker exec -it docker-practice-db-1 mysql -u root -ppassword1234

# MySQL 명령어
USE mydb;
CREATE TABLE test (id INT, name VARCHAR(50));
INSERT INTO test VALUES (1, '볼륨 테스트');
SELECT * FROM test;
exit
```

### AWS EC2 - SSH 접속

```bash
# 키 파일 권한 설정 (필수)
chmod 400 ~/Desktop/infra-test-1.pem

# SSH 접속
ssh -i ~/Desktop/infra-test-1.pem ubuntu@퍼블릭IP

# 내 현재 IP 확인 (IPv4)
curl -4 ifconfig.me
```

### AWS EC2 - Docker 설치 (Ubuntu)

```bash
# 패키지 목록 업데이트
sudo apt update

# Docker 설치
sudo apt install -y docker.io

# Docker 서비스 시작 및 자동시작 설정
sudo systemctl start docker
sudo systemctl enable docker

# sudo 없이 docker 사용하도록 권한 부여
sudo usermod -aG docker ubuntu

# 재접속 후 확인
docker ps
```

### AWS EC2 - Docker Compose 설치 (Ubuntu)

```bash
sudo apt install -y ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list
sudo apt update
sudo apt install -y docker-compose-plugin
```

### AWS EC2 - 파일 전송 및 배포

```bash
# 로컬 → 서버로 파일 전송 (SCP)
scp -i ~/Desktop/infra-test-1.pem \
  .env docker-compose.yml Dockerfile index.html \
  ubuntu@퍼블릭IP:~/docker-practice/

# 서버에서 폴더 미리 생성 (SCP 전송 전)
ssh -i ~/Desktop/infra-test-1.pem ubuntu@퍼블릭IP "mkdir -p ~/docker-practice"

# 서버에서 컨테이너 실행
cd ~/docker-practice
docker compose up -d --build
```

---

## 발생했던 오류 및 해결 방법

### 1. 경로에 공백이 있을 때 cd 오류
```
bash: cd: too many arguments
```
**원인**: 경로에 공백("INFRA 공부")이 있어서 인자가 분리됨
**해결**: 따옴표로 감싸기
```bash
cd ~/Desktop/"INFRA 공부"/TIL/INFRA
```

---

### 2. Docker Desktop이 꺼져있을 때
```
ERROR: failed to connect to the docker API
```
**원인**: Docker Desktop 앱이 실행되지 않은 상태
**해결**: Docker Desktop 앱 실행 후 재시도

---

### 3. SSH Connection timed out
```
ssh: connect to host x.x.x.x port 22: Connection timed out
```
**원인 1**: 퍼블릭 IP를 잘못 입력 (프라이빗 IP와 혼동)
**원인 2**: 보안 그룹에 등록된 IP와 현재 내 IP가 다름 (네트워크 변경 시 IP 변경됨)
**해결**:
```bash
# 현재 IP 확인
curl -4 ifconfig.me
# AWS 콘솔 → 보안 그룹 → SSH 소스를 현재 IP로 업데이트
```

---

### 4. docker compose 명령어 오류
```
unknown shorthand flag: 'd' in -d
```
**원인**: Docker Compose가 설치되지 않은 상태 (docker.io만 설치됨)
**해결**: Docker 공식 저장소 추가 후 docker-compose-plugin 설치

---

### 5. .env 파일 내용 오염
**원인**: `echo ".env" >> .gitignore` 명령어가 .gitignore 대신 .env 파일에 잘못 입력됨
**해결**: .env 파일 내용을 직접 수정하여 올바른 환경변수로 복구
```
MYSQL_ROOT_PASSWORD=password1234
MYSQL_DATABASE=mydb
```

---

### 6. SCP로 mysql_data 폴더까지 전송
**원인**: `-r` 옵션으로 폴더 전체를 전송하면 mysql_data(수백 MB)도 함께 전송됨
**해결**: 필요한 파일만 개별 지정해서 전송
```bash
scp -i 키파일 .env docker-compose.yml Dockerfile index.html ubuntu@IP:~/docker-practice/
```

---

## 오늘 만든 파일 구조

```
docker-practice/
├── .env                # 민감한 환경변수 (git 제외)
├── .gitignore          # .env, mysql_data/ 제외
├── docker-compose.yml  # 멀티 컨테이너 정의
├── Dockerfile          # nginx 커스텀 이미지 정의
├── index.html          # 서빙할 HTML 파일
└── mysql_data/         # MySQL 데이터 (git 제외, 로컬 생성)
```

## 퍼블릭 IP vs 프라이빗 IP

| 종류 | 예시 | 용도 |
|---|---|---|
| 퍼블릭 IP | `3.143.235.78` | 외부 인터넷에서 접근 (SSH, 브라우저) |
| 프라이빗 IP | `172.31.1.169` | AWS 내부 네트워크 통신용 |

SSH 접속은 항상 **퍼블릭 IP** 사용.

## AMI별 기본 사용자 이름

| AMI | 사용자 이름 |
|---|---|
| Amazon Linux | `ec2-user` |
| Ubuntu | `ubuntu` |
