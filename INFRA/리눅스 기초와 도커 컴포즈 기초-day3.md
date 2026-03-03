# 🏗️ SSAFY 2학기 인프라 실전 정복 노트 (Day 3 - AWS 배포편)

### 🗺️ 인프라 정복 여정 요약 (7~10단계: 클라우드 확장)

| 단계 | 주제 | 우리가 한 일 (핵심 액션) | 인프라적 의미 |
| :--- | :--- | :--- | :--- |
| **7단계** | **AWS EC2 시작** ☁️ | 인프라 전용 인스턴스(infra-test) 생성 | 내 컴퓨터를 넘어 **'클라우드 영토'**를 확보함 |
| **8단계** | **보안 그룹 설정** 🛡️ | 22(SSH), 80(HTTP), 8085(Custom) 포트 개방 | 외부 손님이 들어올 수 있는 **'대문'**을 안전하게 설계함 |
| **9단계** | **SCP 파일 전송** 🚚 | `scp` 명령어로 로컬 파일을 서버로 복사 | 개발 환경의 소스를 운영 환경으로 **'배포(Deploy)'**함 |
| **10단계** | **Cloud 가동** 🚀 | `docker-compose` 빌드 및 서비스 실행 | 전 세계 어디서든 접속 가능한 **'실제 서비스'**를 완성함 |

---

## [7단계: AWS EC2 - 클라우드 땅 일구기]

### 1. 인스턴스 생성 (Instance Start)
- **AMI:** Amazon Linux 2023 (가장 표준적인 리눅스 환경)
- **Key Pair (.pem):** 서버 접속을 위한 유일한 열쇠. 보안을 위해 상위 폴더(`../`)에 안전하게 보관.
- **Public IP:** 서버의 고유 주소 (예: `18.218.44.0`)

---

## [8단계: 보안 그룹(Security Group) - 대문 설계]

### 🛡️ 인바운드 규칙 (Inbound Rules)
서버로 들어오는 길목을 지키는 설정입니다.

- **22 (SSH):** 관리자인 서현님의 IP에서만 접속 허용 (보안 유지)
- **80 (HTTP):** 일반 웹 접속용
- **8085 (TCP):** 우리가 도커 컴포즈에서 설정한 **백엔드 서버 전용 포트**
- **443 (HTTPS):** 향후 보안 연결 대비

---

## [9단계: SCP 파일 전송 - 클라우드로 이사하기]

### 🚚 이삿짐 보내기 명령어
내 컴퓨터(Local) 터미널에서 실행합니다.

```bash
# -i: 키 파일 지정, -r: 폴더 통째로 전송
scp -i "../infra-test.pem" docker-compose.yml dockerfile .env index.html ec2-user@18.218.44.0:~/hello_infra/
```

### ⚠️ 마주했던 트러블슈팅 (Troubleshooting)
1. **경로 문제:** `scp`는 서버 안이 아니라 **내 컴퓨터**에서 실행해야 함을 확인.
2. **키 파일 인식:** `infra-test.pem` 파일이 현재 위치에 없다면 경로(`../`)를 정확히 적어줘야 함.
3. **특수 파일 거부:** `mysql.sock` 같은 소켓 파일은 복사가 안 됨. DB 데이터 폴더를 제외하고 **설정 파일(YML, Dockerfile)** 위주로 전송하는 것이 정석.

---

## [10단계: Docker Build & Run - 서비스 개시]

### 🐳 서버 내 환경 구축
EC2에 접속한 뒤 도커를 깨워야 합니다.

```bash
sudo dnf install -y docker
sudo systemctl start docker
sudo usermod -aG docker ec2-user # 로그아웃 후 재접속 필요
```

### 🛠️ 고난도 해결: Buildx 엔진 업데이트
기본 도커 엔진이 구형이라 빌드가 안 될 때(`buildx 0.17.0 or later` 요구), 직접 최신 엔진을 장착했습니다.

```bash
mkdir -p ~/.docker/cli-plugins
curl -SL https://github.com/docker/buildx/releases/download/v0.19.1/buildx-v0.19.1.linux-amd64 -o ~/.docker/cli-plugins/docker-buildx
chmod +x ~/.docker/cli-plugins/docker-buildx
```

### 🚀 마법의 명령어 실행
```bash
# 요리책(YML)을 보고 상자를 새로 구워서(Build) 배경에서 실행(-d)하라!
sudo docker-compose up -d --build
```

---

## 📚 서현님을 위한 인프라 용어 사전

- **SSH (Secure Shell):** 원격 컴퓨터를 내 것처럼 조종하는 보안 통로.
- **SCP (Secure Copy):** SSH 통로를 통해 파일을 안전하게 복사하는 방법.
- **DNF:** 리눅스에서 프로그램을 설치할 때 쓰는 '앱스토어' 같은 도구.
- **Buildx:** 도커 이미지를 구울 때 쓰는 최첨단 오븐(빌드 엔진).
- **Public IPv4:** 전 세계 인터넷에서 우리 서버를 찾아오는 고유 번호.

---

### ✅ 최종 확인
브라우저 주소창에 `http://18.218.44.0:8085/` 입력 시 서현님의 웹 페이지가 뜨면 **클라우드 엔지니어 등극 성공!** 🎊
