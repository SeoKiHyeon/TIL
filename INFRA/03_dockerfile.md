# Dockerfile

## 1. Dockerfile이란

공식 이미지(nginx, mysql 등)를 그대로 사용하는 것이 아니라, **나만의 커스텀 이미지를 만들고 싶을 때** Dockerfile을 작성한다.

Dockerfile은 이미지를 어떻게 구성할지 순서대로 적은 텍스트 파일이다.

```
Dockerfile (레시피) ──→ docker build ──→ Image ──→ docker run ──→ Container
```

기존 볼륨 마운트(`-v`) 방식과의 차이:

| 방식 | 특징 | 적합한 상황 |
|---|---|---|
| 볼륨 마운트(`-v`) | 호스트 파일을 컨테이너에 실시간 연결 | 개발 중 (코드 수정 즉시 반영) |
| Dockerfile + COPY | 파일을 이미지 안에 직접 포함 | 배포 시 (독립적으로 배포 가능한 이미지) |

---

## 2. Dockerfile 명령어

### FROM — 베이스 이미지 지정 (필수)

모든 이미지는 다른 이미지를 기반으로 만들어진다. `FROM`은 어떤 이미지를 기반으로 시작할지 지정한다.

```dockerfile
FROM nginx:latest
FROM openjdk:17-jdk-slim
FROM node:20-alpine
```

`alpine`이 붙은 이미지는 최소한의 파일만 포함한 경량 버전이다. 이미지 크기를 줄이고 싶을 때 사용한다.

### WORKDIR — 작업 디렉토리 설정

이후 명령어들이 실행될 기본 디렉토리를 지정한다. 해당 경로가 없으면 자동 생성된다.

```dockerfile
WORKDIR /app
```

이후 `COPY`, `RUN`, `CMD` 등이 모두 `/app` 기준으로 실행된다.

### COPY — 파일 복사

호스트(내 컴퓨터)의 파일을 이미지 내부로 복사한다.

```dockerfile
# COPY 호스트경로 이미지내경로
COPY index.html /usr/share/nginx/html/index.html

# 현재 디렉토리 전체를 WORKDIR로 복사
COPY . .
```

### RUN — 이미지 빌드 중 명령 실행

이미지를 만드는 과정에서 실행할 명령어를 지정한다. 패키지 설치, 디렉토리 생성 등에 사용한다.

```dockerfile
# 패키지 업데이트 및 설치
RUN apt-get update && apt-get install -y curl

# 여러 명령은 &&로 연결해서 레이어를 줄이는 것이 좋은 관행이다
RUN mkdir -p /app/logs && chmod 755 /app/logs
```

### CMD — 컨테이너 시작 시 실행할 명령어

컨테이너가 시작될 때 기본으로 실행되는 명령어다. Dockerfile에서 한 번만 작성한다.

```dockerfile
CMD ["nginx", "-g", "daemon off;"]
CMD ["java", "-jar", "app.jar"]
CMD ["node", "server.js"]
```

> **RUN vs CMD 차이**
> - `RUN`: 이미지 빌드 시점에 실행 (결과가 이미지에 반영됨)
> - `CMD`: 컨테이너 실행 시점에 실행 (이미지에 반영되지 않음)

### EXPOSE — 포트 문서화

컨테이너가 사용하는 포트를 명시한다. 실제로 포트를 여는 것이 아니라, "이 이미지는 이 포트를 사용합니다"라는 문서 역할이다.
실제 포트 연결은 `docker run -p`에서 한다.

```dockerfile
EXPOSE 80
EXPOSE 8080
```

### ENV — 환경변수 설정

이미지 내부에서 사용할 환경변수를 설정한다.

```dockerfile
ENV JAVA_OPTS="-Xms256m -Xmx512m"
ENV APP_PORT=8080
```

---

## 3. 전체 구조 예시

### Nginx 정적 파일 서버

```dockerfile
# 1. 공식 nginx 이미지를 베이스로 사용
FROM nginx:latest

# 2. 호스트의 index.html을 nginx 기본 경로로 복사
COPY index.html /usr/share/nginx/html/index.html

# 3. 컨테이너 시작 시 nginx를 포그라운드로 실행
CMD ["nginx", "-g", "daemon off;"]
```

### Spring Boot 애플리케이션

```dockerfile
# 1. JDK 17 슬림 버전을 베이스로 사용
FROM openjdk:17-jdk-slim

# 2. 컨테이너 내 작업 디렉토리 설정
WORKDIR /app

# 3. 빌드된 jar 파일을 이미지 내부로 복사
COPY build/libs/app.jar app.jar

# 4. 이미지가 사용하는 포트 문서화
EXPOSE 8080

# 5. 컨테이너 시작 시 jar 파일 실행
CMD ["java", "-jar", "app.jar"]
```

---

## 4. 이미지 빌드 명령어

```bash
# 기본 빌드 (현재 디렉토리의 Dockerfile 사용)
docker build .

# 이미지에 이름(태그) 부여
docker build -t my-nginx-image .

# 이름과 버전 태그 함께 부여
docker build -t my-nginx-image:1.0 .

# Dockerfile 경로를 직접 지정
docker build -f ./docker/Dockerfile -t my-app .
```

- `-t` (tag): 이미지 이름 지정
- `.` : Dockerfile이 위치한 디렉토리 (빌드 컨텍스트)

빌드 후 확인:

```bash
docker images
# my-nginx-image   latest   a1b2c3...   2 minutes ago   141MB
```

빌드된 이미지로 컨테이너 실행:

```bash
# 볼륨 마운트 없이 이미지만으로 실행 가능
docker run -d -p 8080:80 my-nginx-image
```

---

## 5. .dockerignore

`.gitignore`처럼 빌드 시 이미지에 포함하지 않을 파일/디렉토리를 지정한다. 이미지 크기를 줄이고 민감한 파일이 포함되는 것을 방지한다.

`.dockerignore` 파일 예시:

```
.git
.env
node_modules
*.log
target/
```

---

## 6. 이미지 레이어 구조

Dockerfile의 각 명령어는 이미지 레이어를 하나씩 생성한다.

```
FROM nginx          → 레이어 1 (nginx 기본 레이어)
COPY . /app         → 레이어 2
RUN apt install...  → 레이어 3
CMD [...]           → 레이어 4 (메타데이터)
```

레이어는 캐시된다. 이전에 빌드한 것과 동일한 명령어는 재실행하지 않고 캐시를 사용한다. 자주 변경되는 명령어(COPY 등)는 Dockerfile 아래쪽에 배치하면 빌드 속도를 높일 수 있다.
