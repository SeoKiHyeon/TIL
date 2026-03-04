# 🏗️ 시스템 아키텍처 설계서

> CLAUDE.md 기획서 기반으로 설계한 인프라 아키텍처
> Tech Stack: React / Spring Boot / MySQL / Redis / Polygon / OpenAI / Jenkins

---

## 전체 아키텍처 개요

```
                         ┌──────────────────────┐
                         │    사용자 (Browser)    │
                         └──────────┬───────────┘
                                    │ HTTPS
                         ┌──────────▼───────────┐
                         │   CloudFront (CDN)    │◄──── S3 (React 빌드 정적 파일)
                         └──────────┬───────────┘
                                    │
                         ┌──────────▼───────────┐
                         │  ALB (Load Balancer)  │
                         └───────┬──────┬───────┘
                                 │      │
              ┌──────────────────▼┐    ┌▼──────────────────────┐
              │   Nginx + React    │    │  Spring Boot (x2~3)   │
              │   (Frontend SPA)   │    │  (Backend API 서버)    │
              └──────────────────┘    └┬──────────┬────────────┘
                                        │          │ (공유 상태)
                           ┌────────────▼┐  ┌──────▼──────────┐
                           │    MySQL     │  │     Redis        │
                           │  (Main DB)   │  │ (캐시/분산락/타이머)│
                           └────────────┘  └────────────────┘
                                    │
         ┌──────────────────────────┼───────────────────────┐
         │                          │                       │
┌────────▼───────┐    ┌─────────────▼──────────┐  ┌────────▼──────────┐
│  OpenAI API     │    │  Polygon 네트워크       │  │  IPFS (Pinata)    │
│  (퀘스트 AI 생성) │    │  (Smart Contract)      │  │  (NFT 메타데이터) │
└────────────────┘    └────────────────────────┘  └───────────────────┘
         ┌──────────────────────────┐
         │    Jenkins (CI/CD)       │
         │    (별도 EC2 or 컨테이너) │
         └──────────────────────────┘
```

---

## 블록체인 아키텍처 상세

> 블록체인을 처음 접하는 경우를 위해 구성 요소부터 설명

### 블록체인 구성 요소

```
┌─────────────────────────────────────────────────────────────────┐
│                      블록체인 스택                               │
│                                                                 │
│  [개발 도구]       [배포 대상]       [접근 방법]      [저장소]    │
│  Hardhat       →  Polygon 네트워크  ← Web3j (Java)   IPFS       │
│  (작성/테스트/      (Amoy 테스트넷    ← ethers.js (JS)  (NFT 메타  │
│   배포 스크립트)     → Mainnet)                        데이터)    │
│                                                                 │
│  [스마트 컨트랙트 - Solidity]                                    │
│  GachaContract.sol        뽑기 결과 온체인 기록                  │
│  EnhancementContract.sol  강화 결과 온체인 기록                  │
│  SynthesisContract.sol    합성 결과 온체인 기록                  │
│  CardNFT.sol              ERC-721 (S등급 NFT 발급)              │
│  Marketplace.sol          NFT P2P 거래                          │
│                                                                 │
│  [난수 검증]                                                     │
│  Chainlink VRF            뽑기 확률의 조작 불가 증명             │
└─────────────────────────────────────────────────────────────────┘
```

### 핵심 개념: 온체인 vs 오프체인

| 처리 위치 | 대상 | 이유 |
|-----------|------|------|
| 온체인 (Polygon) | 뽑기·강화·합성 결과, NFT 소유권, P2P 거래 | 투명성 보장, 조작 불가 증명 |
| 오프체인 (MySQL) | 퀘스트 진행 상태, 골드 잔액, 랭킹, 회사 레벨 | 빠른 처리 속도, 비용 절감 |

### 백엔드-주도 방식 (권장)

> 유저가 MetaMask를 설치하지 않아도 게임 가능 (NFT 거래 시에만 필요)

```
[뽑기 흐름]
유저 클릭
  → Spring Boot: 결과 계산 (30% 확률 등 게임 로직)
  → Web3j: 서버 지갑으로 GachaContract 호출 (트랜잭션 발행)
  → Polygon: 결과값 영구 기록 (블록 확인)
  → MySQL: 카드 데이터 저장
  → 유저에게 결과 반환

[NFT 거래 흐름 - MetaMask 필요]
판매자 → MetaMask로 Marketplace.sol 호출 → NFT Lock
구매자 → MetaMask로 토큰 전송 → NFT 소유권 이전 (온체인)
Backend 이벤트 리스너 → MySQL card.user_id 업데이트
```

### 블록체인 개발 흐름

```
1. Hardhat으로 컨트랙트 작성/테스트 (로컬 네트워크)
2. Polygon Amoy 테스트넷에 배포 (무료 테스트 토큰 사용)
3. 백엔드에서 Web3j로 컨트랙트 주소 등록 & 연동
4. 프론트에서 ethers.js로 MetaMask 지갑 연결
5. 최종 검증 후 Polygon Mainnet 배포
```

---

## 동시 접속자 처리 — 아키텍처와 직결되는 문제

> 게임 서비스에서 동시 접속은 반드시 아키텍처 설계 시 고려해야 함

### 문제 상황과 해결책

**1. 서버 수평 확장 (Scale Out)**
```
ALB (로드밸런서)
  ├── Spring Boot 인스턴스 1  ─┐
  ├── Spring Boot 인스턴스 2  ─┤── Redis (공유 상태)
  └── Spring Boot 인스턴스 3  ─┘      MySQL (공유 DB)

→ JWT 사용으로 서버 자체는 무상태(Stateless) → 어느 서버로 요청이 가도 처리 가능
→ 퀘스트 타이머/세션은 Redis에 저장 → 모든 인스턴스가 공유
```

**2. 동시 뽑기 경쟁 조건 (Race Condition)**
```
문제: 유저가 골드 100 보유 중 → 동시에 2번 뽑기 요청
     → 두 요청 모두 통과 → 골드가 -100이 되는 버그

해결: DB 레벨 원자적 업데이트
  UPDATE users
  SET gold = gold - 100
  WHERE id = ? AND gold >= 100;  ← 조건부 업데이트 (0건 업데이트면 실패 처리)
```

**3. 거래소 동시 구매 (동일 NFT를 여러 명이 동시에 구매)**
```
문제: A, B 두 유저가 같은 NFT를 동시에 구매 시도

해결: Redis 분산 락 (Distributed Lock)
  SET lock:listing:{id} {buyer_id} NX EX 5  ← 5초짜리 락 획득
  → 락 획득 성공한 유저만 구매 진행
  → 실패한 유저 → "이미 판매된 아이템" 응답
```

**4. WebSocket 다중 서버 문제**
```
문제: 유저가 서버1에 WebSocket 연결 → 퀘스트 완료 이벤트는 서버2에서 발생

해결: Redis Pub/Sub
  서버2 (완료 감지) → Redis publish "quest:done:{user_id}"
  서버1 (유저 연결) → Redis subscribe → WebSocket으로 유저에게 Push
```

**5. DB 커넥션 풀 설정**
```yaml
# application.yml
spring:
  datasource:
    hikari:
      maximum-pool-size: 20      # 서버 인스턴스당 최대 20개 연결
      minimum-idle: 5
      connection-timeout: 30000  # 30초 내 연결 못 하면 예외
```

---

## 레이어별 상세 설계

### 1. Frontend — React (SPA)

| 항목 | 내용 |
|------|------|
| 프레임워크 | React 18 |
| 라우팅 | React Router v6 |
| 주요 페이지 | 대시보드 / 퀘스트 / 카드 목록 / 강화·합성·뽑기 / 거래소 / 랭킹 |
| 실시간 통신 | WebSocket (퀘스트 완료 이벤트 수신) |
| Web3 | ethers.js + MetaMask (NFT 거래 시에만 사용) |
| 상태 관리 | Zustand or Redux Toolkit |
| 배포 | `npm run build` → 정적 파일 → Nginx 서빙 |

```
퀘스트 타이머 처리
  [Spring Boot] → 퀘스트 시작 응답: { start_time, expected_end_time }
  [React]       → 두 값으로 남은 시간 클라이언트 계산 (서버 폴링 불필요)
  [WebSocket]   → 완료 이벤트만 서버 Push (불필요한 통신 최소화)
```

---

### 2. Backend — Spring Boot

**패키지 구조**
```
com.startup.game
├── auth/          JWT 인증, OAuth2 소셜 로그인
├── card/          카드 CRUD, 뽑기 로직
├── quest/         퀘스트 배정, Overflow 알고리즘, 타이머
├── enhancement/   강화 시스템
├── synthesis/     합성 시스템
├── company/       회사 레벨, 슬롯 관리
├── marketplace/   P2P 거래 목록, 분산락 구매 처리
├── ranking/       랭킹 집계 (Redis Sorted Set)
├── ai/            OpenAI API, 퀘스트 자동 생성 스케줄러
└── blockchain/    Web3j 연동, 이벤트 리스너
```

**Overflow 알고리즘 처리**
```
퀘스트 시작 요청
  → 최대 1,920가지 배치 탐색 (밀리초 이내)
  → 최적 배치 + 예상 완료 시간 계산
  → Redis에 { quest_assignment_id: expected_end_time } 저장
  → Spring Scheduler가 완료 시간 도달 감지
  → 보상 지급 (Gold + XP) → 레벨업 체크
  → Redis Pub/Sub으로 완료 이벤트 발행
```

---

### 3. 데이터베이스

#### MySQL — 주 데이터베이스

```sql
users
  id, wallet_address, gold, xp, company_level, created_at

cards
  id, user_id, grade(D/C/B/A/S), stat_be, stat_fe, stat_devops,
  stat_ai, stat_dba, stat_design, enhancement_count,
  nft_token_id(nullable), is_listed

quests
  id, title, description, difficulty(1~6),
  req_be, req_fe, req_devops, req_ai, req_dba, req_design,
  reward_gold, reward_xp, expires_at

quest_assignments
  id, quest_id, user_id, slot_number,
  card_ids(JSON), start_time, expected_end_time, status

company_slots
  id, user_id, slot_number(1~5), unlocked_at

marketplace_listings
  id, card_id, seller_id, price_token, status(판매중/판매완료/취소)

rankings
  id, season_id, user_id, total_gold, completed_quests, rank
```

#### Redis — 캐싱 & 동시성 처리

| Key 패턴 | 용도 | TTL |
|----------|------|-----|
| `quest:timer:{id}` | 퀘스트 완료 예정 시간 | 하드캡 시간 |
| `user:session:{id}` | 세션 캐시 | 1시간 |
| `ranking:season:{id}` | 랭킹 Sorted Set | 시즌 종료까지 |
| `lock:listing:{id}` | 거래소 동시 구매 분산락 | 5초 |
| `gacha:ratelimit:{id}` | 뽑기 남용 방지 | 1분 |
| `quest:available` | 현재 노출 퀘스트 목록 | 1시간 |
| `ws:channel:{user_id}` | WebSocket Pub/Sub 채널 | 세션 동안 |

---

### 4. CI/CD — Jenkins

```
[개발자 Push → GitLab/GitHub]
        │
        ▼ Webhook 트리거
[Jenkins Pipeline]
   Stage 1: 코드 체크아웃
   Stage 2: 단위 테스트 (JUnit / Jest)
   Stage 3: Docker 이미지 빌드
   Stage 4: ECR / Docker Hub에 이미지 Push
   Stage 5: EC2에 SSH 접속 → docker-compose pull & up -d
        │
        ▼
[EC2 운영 서버에 자동 배포]
```

```groovy
// Jenkinsfile 핵심 구조
pipeline {
    agent any
    stages {
        stage('Test') {
            steps {
                sh './gradlew test'        // 백엔드 테스트
                sh 'npm test -- --watchAll=false'  // 프론트 테스트
            }
        }
        stage('Build & Push') {
            steps {
                sh 'docker build -t $REGISTRY/backend:$BUILD_NUMBER ./backend'
                sh 'docker push $REGISTRY/backend:$BUILD_NUMBER'
            }
        }
        stage('Deploy') {
            steps {
                sshagent(['ec2-key']) {
                    sh 'ssh ec2-user@$EC2_HOST "cd /app && docker-compose pull && docker-compose up -d"'
                }
            }
        }
    }
}
```

---

### 5. Docker Compose (개발 환경)

```yaml
version: '3.8'
services:
  frontend:
    build: ./frontend
    ports: ["3000:3000"]

  backend:
    build: ./backend
    ports: ["8080:8080"]
    environment:
      - DB_URL=jdbc:mysql://db:3306/startup
      - REDIS_HOST=cache
      - OPENAI_API_KEY=${OPENAI_API_KEY}
      - POLYGON_RPC_URL=${POLYGON_RPC_URL}
      - BACKEND_WALLET_KEY=${WALLET_PRIVATE_KEY}  # 블록체인 서명용
    depends_on: [db, cache]

  db:
    image: mysql:8
    environment:
      MYSQL_DATABASE: startup
      MYSQL_USER: ${DB_USER}
      MYSQL_PASSWORD: ${DB_PASSWORD}
      MYSQL_ROOT_PASSWORD: ${DB_ROOT_PASSWORD}
    volumes:
      - mysql_data:/var/lib/mysql

  cache:
    image: redis:7-alpine

  nginx:
    image: nginx:alpine
    ports: ["80:80", "443:443"]
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf
      - ./frontend/build:/usr/share/nginx/html  # React 빌드 파일
    depends_on: [backend]

volumes:
  mysql_data:
```

---

### 6. AWS 운영 환경

```
AWS ap-northeast-2 (서울)

VPC
├── Public Subnet
│   ├── ALB (Application Load Balancer)  ← Spring Boot 다중 인스턴스로 트래픽 분산
│   └── NAT Gateway
│
└── Private Subnet
    ├── EC2 (React + Nginx)      ← t3.small  (정적 파일 서빙)
    ├── EC2 (Spring Boot) × 2   ← t3.medium (동시 접속자 수에 따라 Scale Out)
    ├── EC2 (Jenkins)            ← t3.small  (CI/CD 전용)
    ├── RDS MySQL                ← db.t3.micro
    └── ElastiCache Redis        ← cache.t3.micro

S3
├── React 빌드 파일
└── NFT 메타데이터 JSON (백업)

CloudFront ← CDN (정적 파일 전 세계 캐싱)
ECR        ← Docker 이미지 저장소
```

---

### 7. Nginx 설정

```nginx
server {
    listen 443 ssl;

    # React SPA 서빙
    location / {
        root /usr/share/nginx/html;
        try_files $uri $uri/ /index.html;  # SPA 라우팅 처리 핵심
    }

    # Spring Boot API
    location /api/ {
        proxy_pass http://backend:8080;
        proxy_set_header X-Real-IP $remote_addr;
    }

    # WebSocket
    location /ws/ {
        proxy_pass http://backend:8080;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}
```

---

## 추가로 고려해야 할 사항

### ① 블록체인 트랜잭션 실패 처리 (데이터 정합성)

```
문제: BC 트랜잭션은 성공했는데 MySQL UPDATE가 실패하면?
     → 온체인에는 뽑기 결과가 기록됐는데 카드가 지급 안 된 상태

해결: Outbox 패턴
  1. MySQL에 먼저 저장 (gacha_result 임시 테이블)
  2. 블록체인 트랜잭션 발행
  3. 이벤트 리스너가 BC 확인 → MySQL 정식 반영
  4. 실패 시 재시도 큐 (Redis)에 넣어 자동 복구
```

### ② 보안

| 항목 | 방법 |
|------|------|
| 서버 지갑 개인키 | AWS Secrets Manager 또는 환경변수 (절대 코드에 하드코딩 금지) |
| Smart Contract 취약점 | Reentrancy 공격 방지 (OpenZeppelin 라이브러리 사용) |
| API 남용 방지 | Redis 기반 Rate Limiting (뽑기, 강화 등) |
| JWT 탈취 | Refresh Token + Blacklist (Redis) |

### ③ 모니터링

```
Spring Boot Actuator → Prometheus (메트릭 수집) → Grafana (대시보드)

주요 알람 설정:
  - API 응답 시간 > 2초
  - 에러율 > 1%
  - Redis 메모리 사용률 > 80%
  - 퀘스트 완료 처리 실패
```

### ④ 개발 단계별 블록체인 환경

| 단계 | 환경 | 비용 |
|------|------|------|
| 로컬 개발 | Hardhat 로컬 네트워크 | 무료 |
| 통합 테스트 | Polygon Amoy 테스트넷 | 무료 (테스트 MATIC 사용) |
| 운영 | Polygon Mainnet | 실제 가스비 발생 |

---

## 미결 결정사항

| 항목 | 옵션 A | 옵션 B | 상태 |
|------|--------|--------|------|
| 블록체인 네트워크 | Polygon | Base (L2) | 미정 |
| NFT 메타데이터 | IPFS (탈중앙화) | S3 (단순) | 미정 |
| 실시간 방식 | WebSocket | SSE (단방향 단순) | 미정 |
| 스케일링 | Docker Compose (단일) | 다중 EC2 + ALB | 트래픽 보고 결정 |
