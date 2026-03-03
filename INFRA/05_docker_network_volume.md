# Docker 네트워크와 볼륨

## 1. Docker 네트워크

### 왜 필요한가

컨테이너는 기본적으로 독립된 환경으로 격리되어 있다. 백엔드 컨테이너가 데이터베이스 컨테이너에 접속하려면, 두 컨테이너가 같은 네트워크에 속해 있어야 한다.

### 네트워크 드라이버 종류

| 드라이버 | 설명 | 주요 사용 상황 |
|---|---|---|
| `bridge` | 같은 호스트의 컨테이너 간 통신 | 기본값. 일반적인 멀티 컨테이너 구성 |
| `host` | 컨테이너가 호스트 네트워크를 직접 사용 | 포트 포워딩 없이 직접 통신 |
| `none` | 네트워크 없음 (완전 격리) | 네트워크가 불필요한 작업 |

실무에서는 대부분 **bridge** 드라이버를 사용한다.

### 브리지 네트워크 동작 방식

```
┌──────────────────────────────────────────────┐
│              Docker Bridge Network            │
│                                              │
│  ┌─────────────┐      ┌──────────────────┐  │
│  │   backend   │ ──→  │    database      │  │
│  │ (컨테이너)   │      │  (컨테이너)       │  │
│  └─────────────┘      └──────────────────┘  │
│                                              │
└──────────────────────────────────────────────┘
         ↑ 외부에서는 포트를 통해서만 접근 가능
```

같은 네트워크 안의 컨테이너들은:
- IP 주소 대신 **컨테이너 이름 또는 서비스 이름**으로 서로를 찾을 수 있다.
- 외부에서는 접근할 수 없다 (`-p`로 개방한 포트만 접근 가능).

### Docker Compose와 네트워크

`docker-compose.yml`으로 서비스를 실행하면, Docker Compose는 자동으로 프로젝트 이름 기반의 브리지 네트워크를 생성하고 모든 서비스를 해당 네트워크에 연결한다.

```yaml
services:
  backend:
    image: openjdk:17-jdk-slim
    environment:
      # IP 주소가 아닌 서비스 이름(database)을 호스트로 사용
      SPRING_DATASOURCE_URL: jdbc:mysql://database:3306/mydb

  database:
    image: mysql:8.0
```

`database`라는 서비스 이름이 내부 DNS 주소처럼 동작한다. `backend` 컨테이너에서 `database:3306`으로 접속하면 `database` 컨테이너의 3306 포트로 연결된다.

### 네트워크 직접 관리

Docker Compose 없이 네트워크를 수동으로 관리할 때:

```bash
# 네트워크 생성
docker network create my-network

# 네트워크를 지정해서 컨테이너 실행
docker run -d --name database --network my-network mysql:8.0
docker run -d --name backend --network my-network my-backend-image

# 네트워크 목록 확인
docker network ls

# 네트워크 상세 정보 확인 (어떤 컨테이너가 연결되어 있는지)
docker network inspect my-network

# 네트워크 삭제
docker network rm my-network
```

---

## 2. Docker 볼륨

### 컨테이너 데이터의 휘발성 문제

컨테이너 내부에서 생성되거나 변경된 데이터는 컨테이너를 삭제하면 함께 사라진다.

```
docker run mysql → 데이터 쌓임 → docker rm → 데이터 모두 삭제
```

이 문제를 해결하기 위해 볼륨을 사용한다. 볼륨은 컨테이너 외부(호스트 또는 Docker 관리 영역)에 데이터를 저장하여 컨테이너가 삭제되어도 데이터가 유지된다.

### 볼륨 종류

#### 1. 바인드 마운트 (Bind Mount)

호스트의 특정 경로를 컨테이너 내부 경로에 연결한다.

```bash
docker run -v /호스트/경로:/컨테이너/경로 이미지명
```

```yaml
# docker-compose.yml
volumes:
  - ./mysql_data:/var/lib/mysql      # 호스트경로:컨테이너경로
  - ./nginx/conf:/etc/nginx/conf.d   # 설정 파일 연결
```

특징:
- 호스트의 특정 경로에 파일이 저장된다.
- 호스트에서 파일을 직접 확인하고 수정할 수 있다.
- 개발 환경에서 코드를 실시간으로 반영할 때 주로 사용한다.

#### 2. 네임드 볼륨 (Named Volume)

Docker가 직접 관리하는 볼륨이다. 경로를 직접 지정하지 않고 이름을 붙여 사용한다.

```bash
# 볼륨 생성
docker volume create db-data

# 볼륨을 마운트해서 컨테이너 실행
docker run -v db-data:/var/lib/mysql mysql:8.0
```

```yaml
# docker-compose.yml
services:
  database:
    volumes:
      - db-data:/var/lib/mysql      # 네임드 볼륨 사용

volumes:
  db-data:                          # 볼륨 선언 (하단에 필요)
```

특징:
- Docker가 `/var/lib/docker/volumes/` 아래에서 직접 관리한다.
- 컨테이너 간 볼륨 공유가 용이하다.
- 프로덕션 환경의 데이터베이스 데이터 저장에 적합하다.

### 바인드 마운트 vs 네임드 볼륨 비교

| 항목 | 바인드 마운트 | 네임드 볼륨 |
|---|---|---|
| 경로 | 호스트의 지정 경로 | Docker 내부 관리 경로 |
| 데이터 접근 | 직접 파일 시스템에서 접근 가능 | `docker volume` 명령어로 관리 |
| 주요 용도 | 개발 중 코드/설정 파일 공유 | 프로덕션 데이터 영속성 |
| 이식성 | 호스트 경로에 의존적 | Docker만 있으면 어디서든 사용 가능 |

### 볼륨 관련 명령어

```bash
# 볼륨 목록 확인
docker volume ls

# 볼륨 상세 정보 (실제 저장 경로 등)
docker volume inspect db-data

# 볼륨 삭제 (해당 볼륨을 사용하는 컨테이너가 없어야 함)
docker volume rm db-data

# 사용하지 않는 볼륨 일괄 삭제
docker volume prune
```

---

## 3. 실전: DB 데이터 영속성 설정

```yaml
version: '3.8'

services:
  database:
    image: mysql:8.0
    container_name: my-mysql
    environment:
      MYSQL_ROOT_PASSWORD: ${DB_PASSWORD}
      MYSQL_DATABASE: ${DB_NAME}
    ports:
      - "3306:3306"
    volumes:
      # 컨테이너의 MySQL 데이터 저장 경로를 호스트에 연결
      - ./mysql_data:/var/lib/mysql

  backend:
    build: .
    container_name: my-backend
    depends_on:
      - database
    environment:
      # 서비스 이름(database)을 DB 호스트 주소로 사용
      SPRING_DATASOURCE_URL: jdbc:mysql://database:3306/${DB_NAME}
      SPRING_DATASOURCE_PASSWORD: ${DB_PASSWORD}
    ports:
      - "8080:8080"
```

이 구성에서:
- `docker-compose down`을 해도 `./mysql_data` 디렉토리에 데이터가 남는다.
- `docker-compose up -d`로 다시 시작하면 이전 데이터가 그대로 유지된다.
- `docker-compose down -v`를 실행하면 볼륨까지 삭제된다 (데이터 초기화 시 사용).

> **주의**: `mysql_data` 디렉토리를 `.gitignore`에 추가해야 한다. DB 데이터 파일을 Git에 올리면 안 된다.
