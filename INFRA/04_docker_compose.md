# Docker Compose

## 1. Docker Compose란

실제 서비스는 단일 컨테이너로 구성되지 않는다. 예를 들어 웹 애플리케이션은 최소한 다음을 필요로 한다.

- 프론트엔드 서버 (Nginx)
- 백엔드 서버 (Spring Boot, FastAPI 등)
- 데이터베이스 (MySQL, PostgreSQL 등)

각각을 `docker run` 명령어로 따로 실행하면 명령이 복잡하고, 컨테이너 간 네트워크 연결도 수동으로 설정해야 한다.

**Docker Compose**는 이러한 여러 컨테이너를 `docker-compose.yml` 파일 하나에 정의하고, 단일 명령어로 한 번에 관리할 수 있게 해준다.

```
docker-compose.yml 파일 하나로:
  - 여러 컨테이너를 정의
  - 컨테이너 간 네트워크 자동 설정
  - 실행 순서(의존성) 지정
  - 환경변수, 볼륨 통합 관리
```

---

## 2. docker-compose.yml 기본 구조

YAML 형식을 사용한다. **들여쓰기(indentation)가 문법적으로 중요**하며, 탭 대신 공백(스페이스)을 사용해야 한다.

```yaml
services:
  서비스이름1:
    # 서비스 설정
  서비스이름2:
    # 서비스 설정

volumes:
  # 볼륨 정의 (선택)

networks:
  # 네트워크 정의 (선택, 미지정 시 자동 생성)
```

- `services`: 실행할 컨테이너 목록. 각 항목이 하나의 컨테이너가 된다.

> **`version` 필드**: Docker Compose v2 이상에서는 `version: '3.8'` 선언이 불필요하다. 최신 버전에서는 생략한다.

---

## 3. 서비스 설정 옵션

### image — 사용할 이미지 지정

Docker Hub의 공식 이미지를 사용할 때 쓴다.

```yaml
services:
  database:
    image: mysql:8.0
```

### build — Dockerfile로 직접 빌드

`image` 대신 사용한다. 해당 경로의 Dockerfile을 읽어 이미지를 빌드한다.

```yaml
services:
  backend:
    build: .          # 현재 디렉토리의 Dockerfile 사용
```

또는 경로와 Dockerfile을 명시적으로 지정할 수 있다:

```yaml
services:
  backend:
    build:
      context: ./backend    # 빌드 컨텍스트 경로
      dockerfile: Dockerfile
```

### container_name — 컨테이너 이름 지정

```yaml
services:
  database:
    image: mysql:8.0
    container_name: my-mysql
```

### ports — 포트 포워딩

`"호스트포트:컨테이너포트"` 형식으로 지정한다.

```yaml
services:
  backend:
    ports:
      - "8080:8080"
      - "8443:8443"
```

### volumes — 볼륨 마운트

```yaml
services:
  database:
    volumes:
      - ./mysql_data:/var/lib/mysql    # 바인드 마운트
      - db-data:/var/lib/mysql         # 네임드 볼륨 (하단 volumes 섹션에 선언 필요)
```

### environment — 환경변수 설정

컨테이너 내부에서 참조하는 설정값을 외부에서 주입한다.

```yaml
services:
  database:
    environment:
      MYSQL_ROOT_PASSWORD: mypassword
      MYSQL_DATABASE: mydb
```

또는 `.env` 파일의 변수를 참조할 수 있다:

```yaml
services:
  database:
    environment:
      MYSQL_ROOT_PASSWORD: ${DB_PASSWORD}
      MYSQL_DATABASE: ${DB_NAME}
```

### depends_on — 실행 순서 지정

특정 서비스가 먼저 시작된 후에 현재 서비스를 시작하도록 지정한다.

```yaml
services:
  backend:
    depends_on:
      - database
```

> **주의**: `depends_on`은 컨테이너가 **시작**되는 순서만 보장한다. 데이터베이스가 완전히 **준비**될 때까지 기다리지는 않는다. 애플리케이션 레벨에서 재시도 로직이 필요할 수 있다.

### restart — 재시작 정책

```yaml
services:
  backend:
    restart: always      # 항상 재시작 (서버 재부팅 시에도)
    # restart: on-failure  # 오류로 종료된 경우에만 재시작
    # restart: unless-stopped  # 수동으로 중지하지 않으면 재시작
```

---

## 4. .env 파일로 비밀 정보 분리

`docker-compose.yml` 파일에 비밀번호를 직접 작성하면, 파일이 Git에 올라갈 경우 노출된다.

`.env` 파일에 민감한 정보를 분리하고, `.gitignore`에 `.env`를 추가한다.

`.env` 파일:

```
DB_PASSWORD=mypassword
DB_NAME=mydb
```

`docker-compose.yml`에서 참조:

```yaml
services:
  database:
    environment:
      MYSQL_ROOT_PASSWORD: ${DB_PASSWORD}
      MYSQL_DATABASE: ${DB_NAME}
```

Docker Compose는 `docker-compose.yml`과 같은 디렉토리에 있는 `.env` 파일을 자동으로 읽는다.

`.gitignore`에 추가:

```
.env
mysql_data/
```

---

## 5. 전체 예시

```yaml
services:
  # 1. 데이터베이스 컨테이너
  database:
    image: mysql:8.0
    container_name: my-mysql
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: ${DB_PASSWORD}
      MYSQL_DATABASE: ${DB_NAME}
    ports:
      - "3306:3306"
    volumes:
      - ./mysql_data:/var/lib/mysql

  # 2. 백엔드 컨테이너 (Dockerfile로 빌드)
  backend:
    build: .
    container_name: my-backend
    restart: always
    ports:
      - "8080:8080"
    depends_on:
      - database
    environment:
      DB_HOST: database        # 서비스 이름을 호스트 주소로 사용
      DB_PORT: 3306
      DB_PASSWORD: ${DB_PASSWORD}
      DB_NAME: ${DB_NAME}
```

> `DB_HOST: database` — 같은 Compose 네트워크 안에서 컨테이너는 서비스 이름으로 서로를 찾는다. 자세한 내용은 `05_docker_network_volume.md` 참고.

---

## 6. 주요 명령어

모든 명령어는 `docker-compose.yml`이 있는 디렉토리에서 실행한다.

```bash
# 서비스 전체 시작 (백그라운드)
docker compose up -d

# Dockerfile 변경 후 이미지 재빌드 후 시작
docker compose up -d --build

# 서비스 전체 중지 및 컨테이너 삭제
docker compose down

# 컨테이너 + 볼륨까지 삭제
docker compose down -v

# 실행 중인 서비스 상태 확인
docker compose ps

# 특정 서비스 로그 확인
docker compose logs backend

# 실시간 로그 출력
docker compose logs -f backend

# 특정 서비스만 재시작
docker compose restart backend

# 특정 서비스 컨테이너 내부 접속
docker compose exec backend bash
```

---

## 7. docker run vs docker-compose 비교

같은 작업을 두 가지 방식으로 비교:

```bash
# docker run 방식 (명령이 길고 실수하기 쉬움)
docker run -d \
  --name my-mysql \
  -e MYSQL_ROOT_PASSWORD=mypassword \
  -e MYSQL_DATABASE=mydb \
  -p 3306:3306 \
  -v ./mysql_data:/var/lib/mysql \
  mysql:8.0
```

```yaml
# docker-compose.yml 방식 (파일로 관리, 재현 가능)
services:
  database:
    image: mysql:8.0
    container_name: my-mysql
    environment:
      MYSQL_ROOT_PASSWORD: mypassword
      MYSQL_DATABASE: mydb
    ports:
      - "3306:3306"
    volumes:
      - ./mysql_data:/var/lib/mysql
```

Docker Compose는 파일로 버전 관리되고, 팀원이 동일한 환경을 재현하기 쉽다.
