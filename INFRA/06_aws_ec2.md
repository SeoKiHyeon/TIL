# AWS EC2 배포

## 1. AWS EC2란

**EC2 (Elastic Compute Cloud)**는 AWS에서 제공하는 가상 서버 서비스다.
내 컴퓨터(로컬 환경) 대신 인터넷에 연결된 클라우드 서버에서 애플리케이션을 실행할 수 있게 해준다.

```
로컬 개발 환경 (내 컴퓨터)
    ↓ 배포
EC2 인스턴스 (AWS 클라우드 서버) → 전 세계에서 접속 가능
```

### 핵심 용어

| 용어 | 설명 |
|---|---|
| **인스턴스 (Instance)** | 실행 중인 가상 서버 하나 |
| **AMI (Amazon Machine Image)** | 인스턴스를 만들 때 사용하는 OS 템플릿 |
| **인스턴스 타입** | 서버의 사양 (CPU, 메모리 크기). 예: `t2.micro` |
| **키 페어 (Key Pair)** | SSH 접속 시 사용하는 공개키/개인키 쌍 |
| **퍼블릭 IP** | 외부 인터넷에서 이 서버를 찾아오는 주소 |
| **보안 그룹 (Security Group)** | 서버의 방화벽. 허용할 IP와 포트를 지정 |

---

## 2. 인스턴스 생성

AWS 콘솔에서 EC2 → 인스턴스 시작 순서로 진행한다.

### AMI 선택

운영체제를 선택하는 단계다.

- **Amazon Linux 2023**: AWS 최적화 리눅스. 프리 티어 지원. 일반적으로 권장.
- **Ubuntu 22.04 LTS**: 많은 레퍼런스와 커뮤니티 지원.

### 인스턴스 타입 선택

| 타입 | vCPU | 메모리 | 특징 |
|---|---|---|---|
| `t2.micro` | 1 | 1 GB | 프리 티어 (12개월 무료) |
| `t3.small` | 2 | 2 GB | 실습/개발 서버에 적합 |
| `t3.medium` | 2 | 4 GB | 간단한 프로덕션 서버 |

### 키 페어 생성

SSH 접속에 사용하는 인증 키다. `.pem` 파일로 다운로드된다.

- 생성 후 안전한 경로에 보관한다.
- **분실하면 서버에 접속할 수 없다**. 재생성이 불가능하므로 주의.
- 권한 설정이 필요하다:

```bash
# 키 파일 권한 설정 (읽기 전용, 소유자만)
chmod 400 my-key.pem
```

---

## 3. 보안 그룹 설정

보안 그룹은 서버의 **방화벽** 역할을 한다. 기본적으로 모든 인바운드 트래픽은 차단되어 있으며, 명시적으로 허용한 포트만 접근할 수 있다.

### 인바운드 규칙 (Inbound Rules)

외부에서 서버로 들어오는 트래픽을 제어한다.

| 포트 | 프로토콜 | 용도 | 소스(허용 IP) |
|---|---|---|---|
| 22 | TCP (SSH) | 서버 원격 접속 | 내 IP만 허용 (보안) |
| 80 | TCP (HTTP) | 웹 서비스 | 0.0.0.0/0 (전체) |
| 443 | TCP (HTTPS) | 보안 웹 서비스 | 0.0.0.0/0 (전체) |
| 8080 | TCP | 백엔드 API 서버 | 0.0.0.0/0 |

> **소스 설정 주의**:
> - SSH(22)는 `내 IP`로 제한하는 것이 보안상 올바르다. `0.0.0.0/0`으로 열면 전 세계에서 접속 시도가 들어온다.
> - 개발 중에는 필요한 포트만 열고, 불필요한 포트는 닫는 것이 원칙이다.

---

## 4. SSH로 서버 접속

SSH(Secure Shell)는 암호화된 통신으로 원격 서버를 제어하는 프로토콜이다.

```bash
# 기본 SSH 접속 명령어
ssh -i "키파일.pem" 사용자명@퍼블릭IP

# 예시 (Amazon Linux)
ssh -i "my-key.pem" ec2-user@18.218.44.0

# 예시 (Ubuntu)
ssh -i "my-key.pem" ubuntu@18.218.44.0
```

| AMI | 기본 사용자 이름 |
|---|---|
| Amazon Linux | `ec2-user` |
| Ubuntu | `ubuntu` |
| Debian | `admin` |

접속 성공 시:

```
Last login: Mon Jan 1 12:00:00 2025 from xxx.xxx.xxx.xxx

       __|  __|_  )
       _|  (     /   Amazon Linux 2023
      ___|\___|___|

[ec2-user@ip-172-31-xx-xx ~]$
```

### 접속 문제 해결

```bash
# "Permission denied" 오류 → 키 파일 권한 문제
chmod 400 my-key.pem

# "Connection refused" 오류 → 보안 그룹의 22번 포트가 열려 있는지 확인

# "Connection timed out" 오류 → 아래 3가지 확인
# 1. 퍼블릭 IP가 맞는지 확인 (프라이빗 IP와 혼동 주의)
# 2. 서버가 실행 중인지 확인 (인스턴스 상태: running)
# 3. 내 IP가 변경되었는지 확인 → 보안 그룹 SSH 소스 업데이트 필요
```

현재 내 IP 확인 방법:
```bash
curl -4 ifconfig.me
```

> **IP 변경 주의**: 보안 그룹에 SSH를 "내 IP"로 제한한 경우, 네트워크가 바뀌면(카페, 학교 등) IP가 변경되어 접속이 차단된다. 이 경우 AWS 콘솔에서 보안 그룹의 SSH 소스를 현재 IP로 업데이트해야 한다.

---

## 5. SCP로 파일 전송

SCP(Secure Copy)는 SSH를 통해 파일을 안전하게 복사하는 방법이다.
**로컬 터미널**에서 실행한다 (서버 안에서 실행하는 것이 아님).

```bash
# 기본 형식
scp -i "키파일.pem" [옵션] 출발지 목적지

# 단일 파일 전송
scp -i "my-key.pem" docker-compose.yml ec2-user@18.218.44.0:~/project/

# 여러 파일 한 번에 전송
scp -i "my-key.pem" docker-compose.yml Dockerfile .env ec2-user@18.218.44.0:~/project/

# 디렉토리 전체 전송 (-r: recursive)
scp -i "my-key.pem" -r ./project ec2-user@18.218.44.0:~/
```

전송할 파일 목록 (일반적으로 필요한 것):
- `docker-compose.yml`
- `Dockerfile`
- `.env`
- 정적 파일 (HTML, 설정 파일 등)

전송하지 않는 것:
- `mysql_data/` (데이터베이스 데이터는 서버에서 새로 생성)
- `.git/`
- `node_modules/`, `target/`, `build/` (서버에서 빌드하거나 Docker 빌드 시 처리)

---

## 6. EC2에서 Docker 설치 및 실행

### Docker 설치

```bash
# Amazon Linux 2023
sudo dnf install -y docker

# Ubuntu
sudo apt-get update && sudo apt-get install -y docker.io
```

### Docker 서비스 시작

```bash
# Docker 데몬 시작
sudo systemctl start docker

# 서버 재부팅 시 자동 시작 설정
sudo systemctl enable docker
```

### 현재 사용자에게 Docker 권한 부여

기본적으로 `docker` 명령어는 `sudo`가 필요하다. 현재 사용자를 docker 그룹에 추가하면 `sudo` 없이 사용할 수 있다.

```bash
sudo usermod -aG docker ec2-user   # Amazon Linux
sudo usermod -aG docker ubuntu     # Ubuntu

# 변경 사항 적용 (재접속 필요)
exit
# 다시 SSH로 접속
```

재접속 후 확인:

```bash
docker ps    # sudo 없이 실행되면 정상
```

### Docker Compose 설치

Docker Compose는 별도로 설치해야 한다. OS에 따라 설치 방법이 다르다.

```bash
# 설치 확인
docker compose version
```

**Amazon Linux 2023:**
```bash
sudo dnf install -y docker-compose-plugin
```

**Ubuntu (공식 Docker 저장소 추가 필요):**
```bash
# 1. 필수 패키지 설치
sudo apt install -y ca-certificates curl gnupg

# 2. Docker 공식 GPG 키 추가
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# 3. Docker 공식 저장소 추가
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list

# 4. 패키지 목록 업데이트 후 설치
sudo apt update
sudo apt install -y docker-compose-plugin
```

> **주의**: Ubuntu에서 `sudo apt install docker.io`로 설치한 경우, docker-compose-plugin이 기본 저장소에 없어 위 방법으로 Docker 공식 저장소를 추가해야 한다.

---

## 7. 서비스 배포 및 실행

파일 전송 후 서버에서 실행:

```bash
# 전송받은 디렉토리로 이동
cd ~/project

# 이미지 빌드 후 백그라운드 실행
docker-compose up -d --build

# 실행 상태 확인
docker-compose ps

# 로그 확인
docker-compose logs -f
```

브라우저에서 `http://퍼블릭IP:포트`로 접속하여 서비스 동작을 확인한다.

---

## 8. 퍼블릭 IP와 탄력적 IP

EC2 인스턴스를 중지 후 재시작하면 **퍼블릭 IP가 변경**된다. 고정 IP가 필요하다면 **탄력적 IP(Elastic IP)**를 할당해야 한다.

- AWS 콘솔 → EC2 → 탄력적 IP → IP 주소 할당 → 인스턴스에 연결
- 인스턴스에 연결된 탄력적 IP는 무료이지만, 연결하지 않고 할당만 해두면 과금된다.

---

## 9. 전체 배포 흐름 정리

```
1. 로컬에서 개발 및 테스트
        ↓
2. EC2 인스턴스 생성 (AMI, 인스턴스 타입, 키 페어, 보안 그룹 설정)
        ↓
3. SCP로 필요한 파일 서버로 전송 (docker-compose.yml, Dockerfile, .env)
        ↓
4. SSH로 서버 접속
        ↓
5. Docker 설치 및 권한 설정
        ↓
6. docker-compose up -d --build 실행
        ↓
7. 퍼블릭 IP로 접속하여 서비스 확인
```
