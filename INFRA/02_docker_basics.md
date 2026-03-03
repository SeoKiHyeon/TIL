# Docker 기초

## 1. 컨테이너란 무엇인가

### 배경: "내 컴퓨터에서는 됩니다" 문제

개발자의 컴퓨터와 서버의 환경(OS, 설치된 프로그램 버전, 설정 등)이 다르면, 개발 환경에서 잘 동작하던 프로그램이 서버에서 동작하지 않는 문제가 발생한다.

컨테이너는 이 문제를 해결하기 위해 **프로그램과 그 실행에 필요한 모든 환경을 하나의 독립된 패키지로 묶어서** 어느 환경에서든 동일하게 실행될 수 있도록 한다.

### VM(가상 머신) vs 컨테이너

```
[VM]
┌─────────────────────────────────┐
│  App A   │  App B   │  App C   │
├──────────┼──────────┼──────────┤
│  Guest   │  Guest   │  Guest   │  <- 각각의 OS가 통째로 올라감
│   OS     │   OS     │   OS     │
├──────────┴──────────┴──────────┤
│         Hypervisor              │
├─────────────────────────────────┤
│           Host OS               │
└─────────────────────────────────┘

[컨테이너]
┌─────────────────────────────────┐
│  App A   │  App B   │  App C   │
├──────────┴──────────┴──────────┤
│         Docker Engine           │  <- OS는 공유, 프로세스만 격리
├─────────────────────────────────┤
│           Host OS               │
└─────────────────────────────────┘
```

| 비교 항목 | VM | 컨테이너 |
|---|---|---|
| 크기 | 수 GB (OS 포함) | 수십~수백 MB |
| 시작 시간 | 수 분 | 수 초 이내 |
| 격리 수준 | 강함 (OS 단위) | 프로세스 단위 |
| 용도 | 완전한 OS 환경이 필요할 때 | 애플리케이션 배포 |

---

## 2. Docker의 핵심 구성요소

### Docker Image

프로그램 실행에 필요한 모든 것(코드, 런타임, 라이브러리, 설정 파일)을 담은 **읽기 전용 템플릿**이다.

- 직접 실행되지 않는다. 이미지를 기반으로 컨테이너를 생성해야 실행된다.
- Docker Hub(`hub.docker.com`)에서 공식 이미지를 받아 사용할 수 있다.
- 예: `nginx`, `mysql:8.0`, `openjdk:17-jdk-slim`

### Docker Container

이미지를 실행한 **인스턴스(실제로 동작하는 프로세스)**다.

- 하나의 이미지로 여러 컨테이너를 동시에 실행할 수 있다.
- 컨테이너를 삭제하면 컨테이너 내부에 생성된 데이터는 사라진다 (볼륨으로 해결).

```
Image (nginx) ─── 실행 ──→ Container 1 (my-web)
              └── 실행 ──→ Container 2 (my-api)
```

### Docker Daemon

백그라운드에서 실행되며, 이미지와 컨테이너를 실제로 관리하는 서버 프로세스다.
`docker` CLI 명령어를 입력하면 Daemon이 받아서 처리한다.

### Docker Hub

공식 Docker 이미지 레지스트리(저장소)다. `docker pull` 명령어로 이미지를 가져온다.

---

## 3. 기본 명령어

### 이미지 관련

```bash
# 이미지 다운로드 (태그 미지정 시 latest 사용)
docker pull nginx
docker pull mysql:8.0

# 로컬에 저장된 이미지 목록 확인
docker images

# 이미지 삭제
docker rmi nginx
```

### 컨테이너 실행

```bash
docker run [옵션] 이미지명 [명령어]
```

| 옵션 | 의미 |
|---|---|
| `-d` | 백그라운드(detached) 모드로 실행 |
| `-p 호스트포트:컨테이너포트` | 포트 포워딩 |
| `--name 이름` | 컨테이너 이름 지정 |
| `-v 호스트경로:컨테이너경로` | 볼륨 마운트 |
| `-e KEY=VALUE` | 환경변수 설정 |
| `--rm` | 컨테이너 종료 시 자동 삭제 |
| `-it` | 터미널 입력 가능 (인터랙티브 모드) |

```bash
# 예시: nginx를 백그라운드로 실행, 호스트 8080 → 컨테이너 80 포트 연결
docker run -d -p 8080:80 --name my-web nginx
```

### 컨테이너 상태 확인

```bash
# 실행 중인 컨테이너 목록
docker ps

# 종료된 것 포함 모든 컨테이너 목록
docker ps -a
```

출력 예시:
```
CONTAINER ID   IMAGE   COMMAND   CREATED   STATUS   PORTS                  NAMES
a1b2c3d4e5f6   nginx   ...       ...       Up       0.0.0.0:8080->80/tcp   my-web
```

### 컨테이너 제어

```bash
# 컨테이너 중지
docker stop my-web

# 중지된 컨테이너 재시작
docker start my-web

# 컨테이너 삭제 (먼저 중지해야 함)
docker rm my-web

# 실행 중인 컨테이너 강제 삭제
docker rm -f my-web
```

### 로그 및 내부 접속

```bash
# 컨테이너 로그 확인
docker logs my-web

# 로그 실시간 출력
docker logs -f my-web

# 실행 중인 컨테이너 내부 셸 접속
docker exec -it my-web bash

# 셸이 없는 경우 sh 사용
docker exec -it my-web sh

# 셸 접속 없이 단일 명령 실행
docker exec my-web ls /usr/share/nginx/html
```

---

## 4. 포트 포워딩

컨테이너는 기본적으로 외부 네트워크와 격리되어 있다. 외부에서 컨테이너의 서비스에 접근하려면 호스트의 포트와 컨테이너의 포트를 연결해야 한다.

```
외부 요청 → 호스트 8080 포트 → 컨테이너 80 포트
```

```bash
# -p 호스트포트:컨테이너포트
docker run -d -p 8080:80 nginx

# 여러 포트 동시 연결
docker run -d -p 8080:80 -p 8443:443 nginx
```

접속 확인: 브라우저에서 `http://localhost:8080` 접속

---

## 5. 볼륨 마운트

컨테이너를 삭제하면 내부 데이터가 모두 사라진다. 데이터를 호스트에 저장하거나, 호스트의 파일을 컨테이너에 반영하려면 볼륨 마운트를 사용한다.

```bash
# -v 호스트경로:컨테이너경로
docker run -d -p 8080:80 -v /home/ubuntu/html:/usr/share/nginx/html nginx
```

위 명령은 호스트의 `/home/ubuntu/html` 디렉토리를 컨테이너의 `/usr/share/nginx/html`과 동기화한다.
호스트에서 파일을 수정하면 컨테이너에도 즉시 반영된다.

> **Windows Git Bash 경로 주의**: Windows 환경의 Git Bash에서는 경로 앞에 슬래시를 하나 더 추가해야 한다.
> ```bash
> docker run -d -p 8080:80 -v "//$(pwd)":/usr/share/nginx/html nginx
> ```

---

## 6. 컨테이너 생명주기 정리

```
docker pull (이미지 다운로드)
    ↓
docker run (컨테이너 생성 + 실행)
    ↓
docker stop (실행 중지)
    ↓
docker start (재실행)
    ↓
docker rm (컨테이너 삭제)
```

이미지를 삭제하려면 해당 이미지로 만든 컨테이너를 먼저 삭제해야 한다.

```bash
docker rm -f my-web      # 컨테이너 삭제
docker rmi nginx         # 이미지 삭제
```
