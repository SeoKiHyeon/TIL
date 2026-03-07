# Nginx 리버스 프록시 + HTTPS (Let's Encrypt)

## 전체 흐름 요약

```
브라우저 → https://https-test.duckdns.org
    → Nginx (리버스 프록시, SSL 처리)
        → / 경로  → frontend 컨테이너
        → /api/   → backend 컨테이너
```

---

## 개념 정리

### 리버스 프록시란?

```
[클라이언트]
     │
     │ http(s)://도메인 (포트 80/443)
     ▼
[Nginx - 리버스 프록시]
     │              │
     │ /api/*        │ /*
     ▼              ▼
[백엔드 :80]    [프론트엔드 :80]
```

클라이언트는 Nginx 하나만 알면 됨. Nginx가 요청 경로를 보고 적절한 서버로 전달.

| 이유 | 설명 |
|---|---|
| 단일 진입점 | 포트 80/443 하나로 여러 서비스 연결 |
| 보안 | 내부 서버 IP/포트를 외부에 숨김 |
| SSL 종료 | HTTPS를 Nginx에서 처리, 내부는 HTTP |
| 로드밸런싱 | 여러 서버에 트래픽 분산 |

---

### SSL / HTTPS란?

**HTTP vs HTTPS:**

```
HTTP  (암호화 없음)
클라이언트 ──────────────── 서버
           "비밀번호: 1234"   ← 중간에서 그대로 보임

HTTPS (SSL/TLS 암호화)
클라이언트 ──────────────── 서버
           "xK9#mQ2!@..."    ← 중간에서 해독 불가
```

| 항목 | HTTP | HTTPS |
|---|---|---|
| 포트 | 80 | 443 |
| 암호화 | 없음 | SSL/TLS 암호화 |
| 주소창 | `http://` | `https://` |

**SSL 종료(SSL Termination):**

```
[클라이언트]
     │ HTTPS (443) - 암호화된 요청
     ▼
[Nginx] ← 여기서 복호화 처리 (SSL 종료)
     │ HTTP (내부 통신) - 평문
     ▼
[백엔드 서버]
```

백엔드는 HTTPS 처리 안 해도 됨. Nginx가 대신 해줌.

---

### Let's Encrypt란?

- 무료 SSL 인증서를 발급해주는 비영리 기관
- **Certbot**: Let's Encrypt 인증서를 자동으로 발급/갱신해주는 도구
- 인증서 유효기간: 90일 (자동 갱신 설정 가능)

**인증서 발급 원리 (HTTP-01 Challenge):**

```
1. Certbot이 Let's Encrypt에 인증서 요청
2. Let's Encrypt가 검증 파일을 지정
3. Certbot이 /.well-known/acme-challenge/ 경로에 파일 생성
4. Let's Encrypt가 http://도메인/.well-known/acme-challenge/파일 접근 확인
5. 도메인 소유 확인 완료 → 인증서 발급
```

> **주의**: Let's Encrypt는 `*.amazonaws.com` 같은 클라우드 제공자 기본 도메인에는 인증서 발급 불가.
> DuckDNS 같은 무료 도메인 서비스 이용 필요.

---

### DuckDNS란?

무료 동적 DNS 서비스. EC2 퍼블릭 IP를 도메인으로 연결해줌.

```
3.143.235.78 (EC2 퍼블릭 IP)
    ↕
https-test.duckdns.org (DuckDNS 무료 도메인)
```

---

## 아키텍처

```
[브라우저]
     │
     │ https://https-test.duckdns.org
     ▼
[DuckDNS DNS]
     │ → 3.143.235.78 (EC2 IP) 로 변환
     ▼
[EC2 - Nginx 컨테이너 :443]
     │ SSL 인증서 확인 (Let's Encrypt)
     │ 복호화 후 내부로 전달
     ├─ location /      → frontend 컨테이너 :80
     └─ location /api/  → backend 컨테이너 :80
```

---

## 파일 구조

```
nginx-practice/
├── docker-compose.yml
├── nginx/
│   └── nginx.conf          ← 리버스 프록시 + HTTPS 설정
├── frontend/
│   ├── Dockerfile
│   └── index.html
├── backend/
│   ├── Dockerfile
│   └── index.html
└── certbot/
    ├── conf/               ← Let's Encrypt 인증서 저장 (자동 생성)
    └── www/                ← HTTP-01 Challenge 파일 (자동 생성)
```

---

## 설정 파일

### docker-compose.yml

```yaml
services:
  nginx:
    image: nginx:latest
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/conf.d/default.conf
      - ./certbot/conf:/etc/letsencrypt
      - ./certbot/www:/var/www/certbot
    depends_on:
      - frontend
      - backend

  certbot:
    image: certbot/certbot
    volumes:
      - ./certbot/conf:/etc/letsencrypt
      - ./certbot/www:/var/www/certbot

  frontend:
    build: ./frontend
    expose:
      - "80"

  backend:
    build: ./backend
    expose:
      - "80"
```

**volumes 설명:**

| 볼륨 | 설명 |
|---|---|
| `./nginx/nginx.conf:/etc/nginx/conf.d/default.conf` | 내가 만든 nginx 설정 파일 마운트 |
| `./certbot/conf:/etc/letsencrypt` | 인증서 저장 경로 (nginx, certbot 공유) |
| `./certbot/www:/var/www/certbot` | HTTP-01 Challenge 파일 경로 (nginx, certbot 공유) |

---

### nginx.conf - 1단계 (인증서 발급용 HTTP)

```nginx
server {
    listen 80;
    server_name https-test.duckdns.org;

    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
    }

    location / {
        proxy_pass http://frontend:80;
    }

    location /api/ {
        proxy_pass http://backend:80/;
    }
}
```

---

### nginx.conf - 2단계 (HTTPS 적용 후 최종)

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
        proxy_pass http://backend:80/;
    }
}
```

**nginx.conf 핵심 설명:**

| 설정 | 의미 |
|---|---|
| `listen 80` | HTTP 요청 수신 |
| `listen 443 ssl` | HTTPS 요청 수신 |
| `return 301 https://...` | HTTP → HTTPS 영구 리다이렉트 |
| `ssl_certificate` | 공개 인증서 경로 |
| `ssl_certificate_key` | 개인 키 경로 |
| `proxy_pass http://frontend:80` | frontend 서비스명으로 내부 라우팅 |
| `location /.well-known/acme-challenge/` | certbot 도메인 인증용 경로 |

---

## 전체 실행 순서

### Step 1: 파일 구조 생성

```bash
mkdir -p ~/nginx-practice/nginx ~/nginx-practice/frontend ~/nginx-practice/backend ~/nginx-practice/certbot/conf ~/nginx-practice/certbot/www
```

### Step 2: 컨테이너 빌드 및 실행 (HTTP 단계)

```bash
cd ~/nginx-practice
docker compose up -d --build
```

### Step 3: Let's Encrypt 인증서 발급

```bash
docker compose run --rm certbot certonly \
  --webroot \
  --webroot-path=/var/www/certbot \
  --email 이메일@gmail.com \
  --agree-tos \
  --no-eff-email \
  -d https-test.duckdns.org
```

인증서 저장 경로:
- 공개 인증서: `/etc/letsencrypt/live/https-test.duckdns.org/fullchain.pem`
- 개인 키: `/etc/letsencrypt/live/https-test.duckdns.org/privkey.pem`
- 만료일: 발급 후 90일

### Step 4: nginx.conf를 HTTPS 설정으로 교체 후 재시작

```bash
# nginx.conf 수정 후
docker compose restart nginx
```

---

## 발생했던 오류 & 해결

### 오류: port is already allocated

```
Error response from daemon: failed to set up container networking:
Bind for 0.0.0.0:80 failed: port is already allocated
```

**원인**: 이전 실습의 `my-app` 컨테이너가 80 포트를 점유 중
**해결**:

```bash
# 어떤 컨테이너가 80을 쓰는지 확인
docker ps

# 해당 컨테이너 중지
docker stop my-app

# nginx 다시 시작
docker compose up -d nginx
```

---

## 핵심 개념 요약

### HTTP → HTTPS 리다이렉트 흐름

```
브라우저: http://https-test.duckdns.org 접속
    ↓
Nginx (80): return 301 https://$host$request_uri
    ↓
브라우저: https://https-test.duckdns.org 로 자동 이동
    ↓
Nginx (443 ssl): 인증서 확인 → 요청 처리
```

### certbot/conf 볼륨을 nginx와 certbot이 공유하는 이유

```
certbot 컨테이너: 인증서 발급 → /etc/letsencrypt 에 저장
                                        ↕ (같은 볼륨)
nginx 컨테이너:   /etc/letsencrypt 에서 인증서 읽어서 HTTPS 처리
```

### expose vs ports 차이

| 설정 | 의미 |
|---|---|
| `ports: "80:80"` | 호스트(외부) → 컨테이너 포트 바인딩. 외부에서 접근 가능 |
| `expose: "80"` | Docker 내부 네트워크에서만 접근 가능. 외부 노출 없음 |

frontend, backend는 외부에서 직접 접근할 필요 없고 nginx를 통해서만 접근하므로 `expose` 사용.

### Docker 서비스명이 DNS 역할을 하는 이유

Docker Compose로 실행된 컨테이너들은 같은 네트워크 안에 있어서
서비스 이름(`frontend`, `backend`)으로 서로를 찾을 수 있음.

```
proxy_pass http://frontend:80;
           ────────── ──
           서비스 이름  포트 (Docker 내부 통신)
```
