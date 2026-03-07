# 전체 통합 실습 - Jenkins + Nginx + Spring Boot + HTTPS

## 최종 목표

```
git push
  → GitLab Webhook
  → Jenkins 자동 빌드
  → Docker 이미지 빌드
  → Nginx 리버스 프록시 (HTTPS)
  → https://https-test.duckdns.org 접속
```

---

## 전체 아키텍처

```
[개발자] git push
    ↓ Webhook (HTTP POST)
[Jenkins :8080]
    ↓ Jenkinsfile 실행
    ├── Checkout (GitLab에서 코드 pull)
    ├── Build   (docker-compose build)
    └── Deploy  (docker-compose up)
         ↓
[EC2 - Docker Compose]
    ├── Nginx :443 (SSL + 리버스 프록시)
    │     ├── /      → frontend (nginx 정적 서버)
    │     └── /api/  → backend (Spring Boot :8080)
    ├── frontend 컨테이너
    └── backend 컨테이너
         ↓
[https://https-test.duckdns.org]
```

---

## 파일 구조

```
integrated-practice/
├── backend/
│   ├── src/main/java/com/example/demo/
│   │   ├── DemoApplication.java
│   │   └── HelloController.java
│   ├── build.gradle
│   ├── settings.gradle
│   └── Dockerfile
├── frontend/
│   ├── index.html
│   └── Dockerfile
├── nginx/
│   ├── nginx.conf
│   └── Dockerfile
├── docker-compose.yml
└── Jenkinsfile
```

---

## 개념 정리

### Java 관련 용어

**JVM / JRE / JDK:**

```
JDK (개발 도구 전체)
├── JRE (실행 환경)
│    └── JVM (실제 실행기)
└── 컴파일러, 디버거 등 개발 도구
```

| 용어 | 역할 | 비유 |
|---|---|---|
| JVM | .class 파일을 실제로 실행 | 엔진 |
| JRE | JVM + 실행에 필요한 라이브러리 | 자동차 |
| JDK | JRE + 개발 도구(컴파일러 등) | 자동차 + 공장 |

- 개발할 때 → JDK 필요
- 서버에서 실행만 할 때 → JRE면 충분

**JAR 파일:**

```
src/*.java → [Gradle 빌드] → app.jar
```

모든 코드 + 라이브러리를 하나로 묶은 실행 파일. `java -jar app.jar` 로 실행.

**Gradle:**

```
build.gradle에 의존성 명시
    ↓
Gradle이 자동으로 라이브러리 다운로드
    ↓
소스코드 컴파일 + jar 파일 생성
```

Maven(`pom.xml`)과 비슷한 빌드 자동화 도구.

---

### 멀티 스테이지 Docker 빌드

```dockerfile
# 1단계: 빌드 (gradle + JDK)
FROM gradle:8-jdk17 AS build
WORKDIR /app
COPY . .
RUN gradle bootJar --no-daemon

# 2단계: 실행 (JRE만)
FROM eclipse-temurin:17-jre
WORKDIR /app
COPY --from=build /app/build/libs/*.jar app.jar
ENTRYPOINT ["java", "-jar", "app.jar"]
```

| 단계 | 이미지 크기 |
|---|---|
| 1단계만 사용 | ~800MB (gradle + JDK 포함) |
| 2단계 분리 | ~200MB (JRE + jar만) |

빌드 도구는 실행 시 필요 없으므로 최종 이미지에서 제거.

---

### nginx.conf 상세 설명

```nginx
# 80번 포트 블록 (HTTP → HTTPS 리다이렉트)
server {
    listen 80;                          # 80번 포트에서 요청 수신
    server_name https-test.duckdns.org; # 이 도메인으로 온 요청만 처리

    location /.well-known/acme-challenge/ {
        root /var/www/certbot;          # Let's Encrypt 인증용 경로
    }

    location / {
        return 301 https://$host$request_uri;  # HTTP → HTTPS 영구 리다이렉트
        # $host = 요청한 도메인명
        # $request_uri = 원래 요청 경로 (/about, /login 등)
    }
}

# 443번 포트 블록 (HTTPS 실제 처리)
server {
    listen 443 ssl;
    server_name https-test.duckdns.org;

    ssl_certificate /etc/letsencrypt/live/https-test.duckdns.org/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/https-test.duckdns.org/privkey.pem;

    location / {
        proxy_pass http://frontend:80;  # / 요청 → frontend 컨테이너로 전달
    }

    location /api/ {
        proxy_pass http://backend:8080/api/;  # /api/ 요청 → backend 컨테이너로 전달
    }
}
```

**location 매칭 우선순위:** 더 구체적인 경로가 먼저 매칭.

```
요청: /api/hello → /api/ 매칭 (더 구체적)
요청: /about    → / 매칭
```

**proxy_pass에서 서비스명 사용 이유:**

```nginx
proxy_pass http://frontend:80;   # 서비스 이름 (O)
proxy_pass http://172.18.0.3:80; # IP 직접 사용 (X)
```

Docker Compose로 실행된 컨테이너들은 같은 네트워크 안에서 **서비스 이름이 DNS 역할**을 해. IP는 재시작마다 바뀔 수 있으므로 서비스 이름 사용.

---

### docker-compose.yml volumes 상세 설명

```yaml
volumes:
  - /home/ubuntu/certbot/conf:/etc/letsencrypt  # 호스트경로:컨테이너경로
  - /home/ubuntu/certbot/www:/var/www/certbot
```

| 볼륨 | 역할 |
|---|---|
| `/home/ubuntu/certbot/conf` | certbot이 인증서 저장, nginx가 읽음 (공유) |
| `/home/ubuntu/certbot/www` | certbot이 인증 파일 생성, nginx가 서빙 (공유) |

> **주의**: Jenkins 컨테이너 안에서 docker-compose 실행 시 **절대 경로** 사용 필수.
> 상대 경로(`./nginx/nginx.conf`)는 Jenkins 컨테이너 내부 경로로 해석되어 호스트 Docker 데몬이 찾지 못함.

---

### Jenkins docker.sock 마운트

```
호스트 EC2
├── Docker 데몬 (dockerd)
│     └── /var/run/docker.sock  ← 소켓 파일
└── Jenkins 컨테이너
      └── /var/run/docker.sock  ← 호스트 소켓을 마운트
```

Jenkins 컨테이너 안에서 실행하는 `docker` 명령이 호스트 Docker를 제어함.
→ Jenkins가 빌드한 이미지, 실행한 컨테이너가 호스트에 그대로 반영.

**EC2 재시작 시 주의:** 재시작하면 docker.sock 권한이 초기화됨.
```bash
docker exec -u root jenkins bash -c "chmod 666 /var/run/docker.sock"
```

---

### Webhook + Jenkins 보안 설정

**왜 Anonymous 권한이 필요한가:**

```
GitLab Webhook → Jenkins (인증 없이 HTTP POST 요청)
Jenkins: "이 요청은 Anonymous(익명) 사용자의 요청"
Anonymous에게 Job/Build 권한 없으면 → 403 Forbidden
```

**설정 경로:**
- Manage Jenkins → Security → Authorization → Matrix-based security
- Anonymous 행 → Job → Build 체크

**CSRF Protection:**

Jenkins는 기본적으로 외부 POST 요청을 CSRF 공격으로 간주해 차단.
- Manage Jenkins → Security → CSRF Protection → Enable proxy compatibility 체크

---

## 설정 파일 전체

### backend/HelloController.java

```java
package com.example.demo;

import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
@RequestMapping("/api")
public class HelloController {

    @GetMapping("/hello")
    public String hello() {
        return "Hello from Spring Boot Backend!";
    }
}
```

### backend/build.gradle

```groovy
plugins {
    id 'java'
    id 'org.springframework.boot' version '3.5.11'
    id 'io.spring.dependency-management' version '1.1.7'
}

group = 'com.example'
version = '0.0.1-SNAPSHOT'

java {
    toolchain {
        languageVersion = JavaLanguageVersion.of(17)
    }
}

repositories {
    mavenCentral()
}

dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-web'  // Spring Web 필수!
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
    testRuntimeOnly 'org.junit.platform:junit-platform-launcher'
}
```

### backend/Dockerfile

```dockerfile
# 1단계: 빌드
FROM gradle:8-jdk17 AS build
WORKDIR /app
COPY . .
RUN gradle bootJar --no-daemon

# 2단계: 실행
FROM eclipse-temurin:17-jre
WORKDIR /app
COPY --from=build /app/build/libs/*.jar app.jar
ENTRYPOINT ["java", "-jar", "app.jar"]
```

### frontend/index.html

```html
<!DOCTYPE html>
<html>
<head>
  <meta charset="UTF-8">
  <title>통합 실습</title>
</head>
<body>
  <h1>Frontend</h1>
  <p>Spring Boot API 호출 테스트:</p>
  <button onclick="callApi()">API 호출</button>
  <p id="result"></p>

  <script>
    function callApi() {
      fetch('/api/hello')
        .then(res => res.text())
        .then(data => document.getElementById('result').innerText = data);
    }
  </script>
</body>
</html>
```

### frontend/Dockerfile

```dockerfile
FROM nginx:latest
COPY index.html /usr/share/nginx/html/index.html
```

### nginx/nginx.conf

```nginx
server {
    listen 80;
    server_name https-test.duckdns.org;

    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
    }

    location / {
        return 301 https://$host$request_uri;
    }
}

server {
    listen 443 ssl;
    server_name https-test.duckdns.org;

    ssl_certificate /etc/letsencrypt/live/https-test.duckdns.org/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/https-test.duckdns.org/privkey.pem;

    location / {
        proxy_pass http://frontend:80;
    }

    location /api/ {
        proxy_pass http://backend:8080/api/;
    }
}
```

### nginx/Dockerfile

```dockerfile
FROM nginx:latest
COPY nginx.conf /etc/nginx/conf.d/default.conf
```

> nginx는 볼륨 마운트 대신 Dockerfile로 설정 파일을 이미지에 포함시킴.
> 이유: Jenkins 컨테이너 내부에서 docker-compose 실행 시 상대 경로 볼륨 마운트가 동작하지 않음.

### docker-compose.yml

```yaml
services:
  nginx:
    build: ./nginx
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - /home/ubuntu/certbot/conf:/etc/letsencrypt   # 절대 경로 사용!
      - /home/ubuntu/certbot/www:/var/www/certbot     # 절대 경로 사용!
    depends_on:
      - frontend
      - backend

  frontend:
    build: ./frontend
    expose:
      - "80"

  backend:
    build: ./backend
    expose:
      - "8080"
```

### Jenkinsfile

```groovy
pipeline {
    agent any

    stages {
        stage('Checkout') {
            steps {
                echo '코드를 가져왔습니다.'
                checkout scm
            }
        }

        stage('Build') {
            steps {
                echo 'Docker 이미지를 빌드합니다.'
                sh 'docker-compose -f INFRA/integrated-practice/docker-compose.yml build'
            }
        }

        stage('Deploy') {
            steps {
                echo '배포를 시작합니다.'
                sh 'docker-compose -f INFRA/integrated-practice/docker-compose.yml up -d'
            }
        }
    }

    post {
        success {
            echo '파이프라인 성공!'
        }
        failure {
            echo '파이프라인 실패!'
        }
    }
}
```

---

## 전체 실행 순서

### Step 1: EC2 인스턴스 생성

AWS 콘솔 → EC2 → 인스턴스 시작

| 항목 | 값 |
|---|---|
| AMI | Ubuntu 22.04 LTS |
| Instance type | t3.micro |
| Key pair | last_test.pem |

**보안 그룹 인바운드 규칙:**

| 포트 | 용도 |
|---|---|
| 22 | SSH |
| 80 | HTTP |
| 443 | HTTPS |
| 8080 | Jenkins |

### Step 2: Docker 설치

```bash
sudo apt update && sudo apt install -y docker.io
sudo systemctl start docker && sudo systemctl enable docker
sudo usermod -aG docker ubuntu
# 재접속 필요
```

### Step 3: Docker Compose 설치

```bash
sudo apt install -y ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list
sudo apt update && sudo apt install -y docker-compose-plugin
```

### Step 4: Jenkins 컨테이너 실행

```bash
docker run -d \
  -p 8080:8080 \
  -p 50000:50000 \
  -v jenkins_home:/var/jenkins_home \
  -v /var/run/docker.sock:/var/run/docker.sock \
  --name jenkins \
  jenkins/jenkins:lts
```

### Step 5: Jenkins 컨테이너 안에 docker 설치

```bash
# docker 설치
docker exec -u root jenkins bash -c "apt-get update && apt-get install -y docker.io"

# docker-compose 설치 (Jenkins 컨테이너는 Debian 기반이라 별도 설치)
docker exec -u root jenkins bash -c "curl -L https://github.com/docker/compose/releases/download/v2.24.0/docker-compose-linux-x86_64 -o /usr/local/bin/docker-compose && chmod +x /usr/local/bin/docker-compose"

# docker.sock 권한 설정
docker exec -u root jenkins bash -c "chmod 666 /var/run/docker.sock"

# 설치 확인
docker exec jenkins docker ps
docker exec jenkins docker-compose version
```

> **EC2 재시작 시 주의:** 재시작 후 반드시 chmod 666 재설정 필요.

### Step 6: Swap 메모리 추가 (t3.micro 메모리 부족 대비)

```bash
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
free -h  # 확인
```

### Step 7: Jenkins 초기 설정

```bash
# 초기 비밀번호 확인
docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword
```

브라우저에서 `http://EC2_IP:8080` 접속 → 초기 비밀번호 입력 → Install suggested plugins

### Step 8: GitLab 플러그인 설치

Manage Jenkins → Plugins → Available plugins → "GitLab" 검색 → Install

### Step 9: GitLab Credential 등록

Manage Jenkins → Credentials → System → Global credentials → Add Credentials

| 항목 | 값 |
|---|---|
| Kind | Username with password |
| Username | GitLab 아이디 |
| Password | GitLab 비밀번호 |
| ID | `gitlab-credentials` |

### Step 10: Pipeline Job 생성

New Item → `integrated-pipeline` → Pipeline

| 항목 | 값 |
|---|---|
| Build Triggers | Build when a change is pushed to GitLab 체크 |
| Definition | Pipeline script from SCM |
| SCM | Git |
| Repository URL | `https://lab.ssafy.com/아이디/jenkins-practice.git` |
| Credentials | gitlab-credentials |
| Branch | `*/main` |
| Script Path | `INFRA/integrated-practice/Jenkinsfile` |

### Step 11: GitLab Webhook 설정

GitLab → Settings → Webhooks → Add new webhook

| 항목 | 값 |
|---|---|
| URL | `http://EC2_IP:8080/project/integrated-pipeline` |
| Trigger | Push events |
| SSL verification | 비활성화 |

### Step 12: Jenkins 보안 설정

**Matrix-based security:**
Manage Jenkins → Security → Authorization → Matrix-based security
→ Anonymous 행 → Job → Build 체크

**CSRF Protection:**
Manage Jenkins → Security → CSRF Protection → Enable proxy compatibility 체크

### Step 13: certbot으로 HTTPS 인증서 발급

```bash
# EC2에 certbot 디렉토리 생성
mkdir -p /home/ubuntu/certbot/conf /home/ubuntu/certbot/www

# 인증서 발급 (nginx가 80 포트로 실행 중인 상태에서)
docker run --rm \
  -v /home/ubuntu/certbot/conf:/etc/letsencrypt \
  -v /home/ubuntu/certbot/www:/var/www/certbot \
  certbot/certbot certonly \
  --webroot \
  --webroot-path=/var/www/certbot \
  --email 이메일@gmail.com \
  --agree-tos \
  --no-eff-email \
  -d https-test.duckdns.org
```

인증서 저장 위치:
- 공개 인증서: `/home/ubuntu/certbot/conf/live/https-test.duckdns.org/fullchain.pem`
- 개인 키: `/home/ubuntu/certbot/conf/live/https-test.duckdns.org/privkey.pem`
- 만료일: 발급 후 90일

---

## 발생했던 오류 & 해결

### 오류 1: Jenkinsfile 대소문자 오류

```
ERROR: Unable to find INFRA/integrated-practice/jenkinsfile
```

**원인**: Script Path에 `jenkinsfile` (소문자 j) 입력
**해결**: `Jenkinsfile` (대문자 J)로 수정

---

### 오류 2: docker compose 명령어 오류

```
unknown shorthand flag: 'f' in -f
```

**원인**: Jenkins 컨테이너 안에 docker-compose 플러그인 미설치
**해결**: docker-compose 바이너리 직접 설치

```bash
docker exec -u root jenkins bash -c "curl -L https://github.com/docker/compose/releases/download/v2.24.0/docker-compose-linux-x86_64 -o /usr/local/bin/docker-compose && chmod +x /usr/local/bin/docker-compose"
```

> EC2 호스트의 docker compose와 Jenkins 컨테이너 안의 docker-compose는 별개!
> Jenkins 파이프라인은 컨테이너 안에서 실행되므로 컨테이너 안에도 설치 필요.

---

### 오류 3: docker.sock permission denied

```
permission denied while trying to connect to the Docker daemon socket
```

**원인**: EC2 재시작 후 docker.sock 권한 초기화
**해결**:

```bash
docker exec -u root jenkins bash -c "chmod 666 /var/run/docker.sock"
```

---

### 오류 4: Spring Web 의존성 누락

```
error: package org.springframework.web.bind.annotation does not exist
```

**원인**: `build.gradle`에 `spring-boot-starter-web` 미포함 (starter만 있음)
**해결**: `build.gradle` 수정

```groovy
// 변경 전
implementation 'org.springframework.boot:spring-boot-starter'

// 변경 후
implementation 'org.springframework.boot:spring-boot-starter-web'
```

---

### 오류 5: nginx.conf 볼륨 마운트 오류 (DinD 문제)

```
error mounting "...nginx.conf": not a directory
```

**원인**: Jenkins 컨테이너 안에서 docker-compose 실행 시, 볼륨의 상대 경로가 호스트 기준으로 해석됨. Jenkins 컨테이너 내부의 파일을 호스트 Docker 데몬이 찾지 못함.

**해결**: nginx도 Dockerfile로 이미지 빌드 (볼륨 마운트 제거)

```dockerfile
# nginx/Dockerfile
FROM nginx:latest
COPY nginx.conf /etc/nginx/conf.d/default.conf
```

SSL 인증서는 **절대 경로** 볼륨 사용:
```yaml
volumes:
  - /home/ubuntu/certbot/conf:/etc/letsencrypt  # 절대 경로
```

---

### 오류 6: EC2 IP 변경 문제

**원인**: EC2 Stop → Start 시 퍼블릭 IP 변경
**해결 체크리스트:**
1. MobaXterm 세션 IP 업데이트
2. 보안 그룹 SSH 소스 IP 업데이트 (`curl -4 ifconfig.me` 로 현재 IP 확인)
3. Jenkins Webhook URL 업데이트 (`http://새IP:8080/project/integrated-pipeline`)
4. DuckDNS IP 업데이트
5. `docker exec -u root jenkins bash -c "chmod 666 /var/run/docker.sock"` 재실행

---

### 오류 7: certbot 인증 실패

```
3.143.235.78: Timeout during connect (likely firewall problem)
```

**원인**: DuckDNS가 이전 EC2 IP를 가리키고 있음
**해결**: DuckDNS에서 도메인 IP를 새 EC2 IP로 업데이트 후 재시도

---

## HTTPS 2단계 배포 전략

### 왜 2번 push가 필요한가?

```
Push 1: HTTP + /.well-known/acme-challenge/ 경로 추가
    ↓ 배포 완료
certbot 실행 (인증서 발급)
    ↓ 인증서 생성됨
Push 2: HTTPS 설정 (443 블록 추가)
```

인증서가 없는 상태에서 HTTPS 설정을 하면 nginx가 시작 실패하기 때문.

---

## 핵심 개념 요약

### Jenkins CI/CD 흐름

```
1. 개발자: 코드 수정 → git push
2. GitLab: Webhook으로 Jenkins에 HTTP POST
3. Jenkins: Webhook 수신 → Jenkinsfile 실행
4. Jenkins: GitLab에서 최신 코드 pull
5. Jenkins: docker-compose build (이미지 빌드)
6. Jenkins: docker-compose up -d (컨테이너 실행)
7. 완료: 새 버전 자동 배포!
```

### Pipeline as Code

Jenkinsfile을 코드와 함께 Git으로 관리.
배포 방식 변경 = Jenkinsfile 수정 후 push.

### Docker-in-Docker (DinD)

```
Jenkins 컨테이너
└── docker-compose 실행
      ↓ /var/run/docker.sock 통해
호스트 Docker 데몬
└── 실제 컨테이너 생성/실행
```

Jenkins가 빌드한 컨테이너는 Jenkins 컨테이너 안이 아닌 **호스트에 생성**됨.
볼륨 경로도 **호스트 기준**으로 해석됨 → 절대 경로 사용 필수.
