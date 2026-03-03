# 🏗️ SSAFY 2학기 인프라 기초 정복 노트 (Day 2)

### 🗺️ 인프라 정복 여정 요약 (1~6단계)

| 단계 | 주제 | 우리가 한 일 (핵심 액션) | 인프라적 의미 |
| :--- | :--- | :--- | :--- |
| **1단계** | **Linux 기초** 🐧 | `mkdir`, `cd`, `echo`로 폴더와 파일을 만듦 | 서버라는 **'땅'**을 일구고 기초 도구를 다루는 법을 익힘 |
| **2단계** | **Docker 입문** 📦 | `docker run hello-world` 실행 | **'상자(컨테이너)'**라는 개념을 처음으로 내 컴퓨터에 띄움 |
| **3단계** | **Nginx 서버** 🌐 | `-p 8080:80`으로 포트를 연결해 Nginx 실행 | 외부 사용자가 들어올 수 있는 **'통로(Port)'**를 여는 법을 배움 |
| **4단계** | **볼륨 마운트** 🔌 | `-v` 옵션으로 내 컴퓨터의 파일을 서버에 연결 | 내 코드를 서버에 **'실시간 동기화'**하는 개발 환경 구축 |
| **5단계** | **Dockerfile** 🍳 | `FROM`, `COPY`를 써서 나만의 이미지 제작 | 남의 상자를 빌려 쓰는 게 아니라, **'우리만의 상자'**를 굽는 법을 익힘 |
| **6단계** | **Compose 자동화** 🏗️ | `docker-compose.yml`에 `build: .` 설정 | 상자를 굽고 띄우는 복잡한 과정을 **'요리책'** 하나로 자동화함 |



🌐 인프라에서의 연결: Docker Network
두 개의 컨테이너(예: Spring Boot와 MySQL)가 서로 통신하려면, 먼저 같은 **네트워크(Network)**라는 울타리 안에 있어야 합니다.

Docker Compose를 사용하면 기본적으로 모든 서비스가 하나의 네트워크에 자동으로 묶이게 되는데요. 이렇게 네트워크가 연결되면, 복잡한 IP 주소 대신 **'서비스 이름'**만으로 서로를 찾아갈 수 있게 됩니다. 🗺️

Q) 두 컨테이너가 같은 네트워크에 연결되어 있다면, 백엔드에서 DB에 접속할 때 호스트 주소(Host) 자리에 어떤 단어를 쓰면 될까?
```
services:
  backend:  # 백엔드 서비스 이름
    ...
  database: # DB 서비스 이름
    ...
```

A) 정답은 database이다.


## 백엔드와 DB 함께 띄우기

- YML 파일
```
version: '3.8'

services:
  # 1. 데이터베이스 서비스
  database:
    image: mysql:8.0
    container_name: my-mysql-db
    environment:
      MYSQL_DATABASE: my_blockchain_db
      MYSQL_ROOT_PASSWORD: ssafy
    ports:
      - "3306:3306"

  # 2. 자바 백엔드 서비스
  backend:
    image: openjdk:17-jdk-slim
    container_name: my-spring-app
    # 백엔드는 database 서비스가 켜질 때까지 기다렸다가 실행되어야 함
    depends_on:
      - database
    environment:
      SPRING_DATASOURCE_URL: jdbc:mysql://database:3306/my_blockchain_db
    ports:
      - "8080:8080"
```


🧐 여기서 인프라 담당자가 주목해야 할 부분
depends_on: "DB가 먼저 켜진 다음에 나를 켜줘!"라는 순서를 정해줍니다.

environment: 프로그램 코드 안에 비번을 적지 않고, 이렇게 인프라 설정 파일에서 값을 넘겨주는 방식을 많이 씁니다. (보안상 아주 중요해요!)



### [5단계: Dockerfile — 나만의 이미지 만들기]

🍳 Dockerfile: 상자를 만드는 '레시피'
Dockerfile은 도커 이미지를 어떻게 만들지 적어놓은 텍스트 파일입니다. "어떤 운영체제를 쓸 건지", "어떤 파일을 넣을 건지", "어떤 명령어를 실행할 건지"를 순서대로 적어줍니다.

index.html이 아예 상자 안에 박혀 있는 우리만의 이미지를 만들기


1. Dockerfile 작성하기

```

# 1. 베이스 이미지 선택 (기반이 되는 땅)
FROM nginx:latest

# 2. 내 컴퓨터의 index.html을 컨테이너 안의 특정 경로로 복사
# (아까 -v 옵션으로 연결했던 그 경로입니다!)
COPY index.html /usr/share/nginx/html/index.html

# 3. 컨테이너가 켜질 때 실행할 명령어 (Nginx는 이미 설정되어 있어 생략 가능하지만 연습삼아!)
CMD ["nginx", "-g", "daemon off;"]

```


2. 이미지 빌드(Build)하기
이제 이 레시피를 가지고 실제 '상자(Image)'를 구워낼 시간입니다.

```
docker build -t my-seohyun-nginx .
```

- -t (Tag): 이미지에 이름을 붙여줍니다. (my-seohyun-nginx)

- . (Dot): "현재 폴더에 있는 Dockerfile을 사용해!"라는 뜻입니다. (매우 중요!)


이제는 실행할 때 거추장스러운 -v (볼륨 연결) 옵션이 필요 없습니다.
```
docker run -d -p 8083:80 my-seohyun-nginx
```



2. Dockerfile의 '3대 요소'가 전부인가요?

3가지(FROM, COPY, CMD)는 이미지를 만들기 위한 가장 핵심적인 기둥이 맞습니다.
하지만 실무(SSAFY 프로젝트 등)로 가면 몇 가지가 더 추가되기도 합니다. 가장 중요한 것들만 뽑아볼게요.

- FROM (기반): "어떤 땅 위에서 시작할까?" (예: nginx, openjdk) — 필수
- COPY (복사): "내 코드를 상자 어디에 넣을까?" — 필수
- CMD (명령): "상자가 열리자마자 어떤 프로그램을 실행할까?" — 필수
- WORKDIR (작업실): "상자 안에서 내가 일할 기본 폴더는 어디야?" (마치 cd 명령어와 비슷해요.)
- EXPOSE (구멍): "이 상자는 몇 번 포트를 외부에 열어둘 거야?" (안내 역할)
- ENV (환경 변수): "상자 안에서 쓸 비밀번호나 설정값은 뭐야?"




### 🏗️ 6단계: Docker Compose와 Dockerfile의 만남

지금까지 docker-compose.yml에서는 남이 만든 이미지를 가져다 쓰기 위해 image: nginx라고 적었습니다. 하지만 이제는 **"내가 만든 레시피로 직접 구워서 써!"**라고 명령을 바꿔야 합니다.


1. docker-compose.yml 수정하기

YAML
```
version: '3.8'

services:
  my-custom-server:
    # 'image' 대신 'build'를 사용합니다!
    build: . 
    container_name: my-final-web
    ports:
      - "8084:80"
    # 이제 이미 상자 안에 index.html이 들어있으므로 volumes는 생략해도 됩니다.

```

- build: .: "현재 폴더(.)에 있는 Dockerfile을 읽어서 이미지를 직접 만들어(Build) 사용해!"라는 뜻입니다.

2. 마법의 명령어 입력 (Build + Up)
```
docker-compose up -d --build
```

- --build: 이 옵션이 핵심입니다! 요리책만 펴는 게 아니라, 재료(Dockerfile)를 보고 상자를 새로 구운 뒤에 서버를 띄우라는 뜻입니다.




### 🌐 7단계 핵심: "우리들만의 비밀 채팅방" (Docker Network)

docker-compose.yml로 여러 서비스를 띄우면, 도커는 이 서비스들을 하나의 가상 네트워크 안에 몰아넣습니다.

이름이 곧 주소다: 이 네트워크 안에서는 복잡한 IP 주소가 필요 없습니다. 요리책에 적어준 서비스 이름(backend, database 등)이 곧 그 서버의 주소가 됩니다.

외부와는 격리: 유저는 우리가 열어준 포트(8080 등)로만 들어올 수 있지만, 상자들끼리는 안에서 자유롭게 대화할 수 있습니다.


🛠️ 실습: 백엔드와 DB를 동시에 띄우는 '진짜' 요리책 작성

YAML
```
version: '3.8'

services:
  # 1. 데이터베이스 상자 (공식 창고에서 가져옴)
  database:
    image: mysql:8.0
    container_name: my-mysql
    environment:
      MYSQL_ROOT_PASSWORD: ssafy     # DB 접속 비밀번호
      MYSQL_DATABASE: my_project_db  # 미리 만들어둘 DB 이름
    # 외부(내 컴퓨터)에서 접속할 포트
    ports:
      - "3306:3306"

  # 2. 백엔드 서버 상자 (우리가 직접 구운 상자)
  backend-server:
    build: .                         # 현재 폴더의 Dockerfile로 빌드
    container_name: my-backend
    ports:
      - "8085:80"
    # 중요: DB가 먼저 켜질 때까지 기다려!
    depends_on:
      - database
    # 백엔드가 DB를 찾아갈 때 쓸 환경변수
    environment:
      DB_HOST: database              # 호스트 주소에 서비스 이름을 씁니다!
      DB_PASSWORD: ssafy

```


environment (환경 변수): DB의 비밀번호나 이름을 코드에 직접 적지 않고 이렇게 밖에서 넘겨줍니다. 나중에 보안을 위해 꼭 필요한 습관이에요.

depends_on: 백엔드가 먼저 켜졌는데 DB가 아직 로딩 중이면 에러가 나겠죠? "DB 먼저, 그 다음 나!"라는 순서를 정해주는 포스트잇 같은 거예요.

서비스 이름 대화: 이제 backend-server는 내부적으로 jdbc:mysql://database:3306/... 같은 주소를 통해 database 상자에게 말을 걸 수 있게 되었습니다.


🚀 실행 및 확인 
```
docker-compose up -d --build
```



⚠️ 여기서 잠깐! 인프라 담당자의 고민
지금 my-mysql이 아주 잘 돌아가고 있죠? 그런데 한 가지 무서운 사실이 있습니다.

지금 상태에서 **docker-compose down**을 하거나 컨테이너를 삭제하면, 그 안에 쌓였던 데이터베이스의 데이터(회원 정보, 게시글 등)가 모두 마법처럼 사라져 버립니다. 컨테이너는 삭제되면 내부 데이터를 남기지 않는 '휘발성' 특징이 있기 때문이죠.

이 문제를 해결하지 못하면 2학기 프로젝트 때 서버를 재시작할 때마다 데이터가 초기화되는 대참사가 일어날 거예요.


### 💾 8단계: 데이터의 영속성 — 볼륨(Volume)으로 데이터 박제하기

그래서 필요한 것이 바로 볼륨(Volume) 설정입니다. 아까 우리가 index.html을 연결할 때 썼던 그 기술을, 이번에는 데이터베이스의 저장소에 적용해 볼 거예요.

🛠️ 실습: DB 데이터 보호하기
docker-compose.yml 파일의 database 서비스 부분에 딱 두 줄만 추가해 봅시다.

YAML
```
database:
    image: mysql:8.0
    container_name: my-mysql
    environment:
      MYSQL_ROOT_PASSWORD: ssafy
      MYSQL_DATABASE: my_project_db
    ports:
      - "3306:3306"
    # --- 여기서부터 추가 ---
    volumes:
      - ./mysql_data:/var/lib/mysql  # 내 폴더의 mysql_data와 DB 저장소를 연결!
    # ----------------------

```


- ./mysql_data:/var/lib/mysql:
내 컴퓨터의 mysql_data 폴더와 컨테이너 안의 진짜 데이터 저장소(/var/lib/mysql)를 연결합니다.
이렇게 하면 컨테이너를 지웠다 다시 띄워도, 데이터는 내 컴퓨터의 mysql_data 폴더에 안전하게 남아있게 됩니다.



### [9단계: 환경 변수(.env) — 비밀 장부로 보안 지키기]

지금처럼 docker-compose.yml 파일에 비밀번호를 직접 적어두면, 나중에 깃허브(GitHub) 같은 곳에 코드를 올렸을 때 전 세계에 내 비밀번호가 공개되는 위험이 있습니다. 이를 방지하기 위해 사용하는 것이 바로 .env 파일입니다.

1. 비밀 장부(.env) 만들기
먼저 hello_infra 폴더 안에 이름이 정확히 .env인 파일을 하나 만듭니다. (파일 이름 앞에 점.이 붙어야 하며, 확장자는 없습니다.)

그 안에 숨기고 싶은 정보를 변수명=값 형태로 적어주세요.


.env 파일 내용:
```
DB_PASSWORD=ssafy
DB_NAME=my_project_db
```


2. 요리책(docker-compose.yml) 수정하기
이제 docker-compose.yml 파일에서 실제 비밀번호가 적혔던 부분을 **비밀 장부의 이름(변수명)**으로 교체합니다. 도커 컴포즈는 ${ } 기호를 보고 .env 파일에서 값을 찾아옵니다.

수정된 docker-compose.yml 예시:

```
services:
  database:
    image: mysql:8.0
    container_name: my-mysql
    environment:
      # 비밀번호 대신 변수명을 사용합니다
      MYSQL_ROOT_PASSWORD: ${DB_PASSWORD}
      MYSQL_DATABASE: ${DB_NAME}
    ports:
      - "3306:3306"
    volumes:
      - ./mysql_data:/var/lib/mysql

  backend-server:
    build: .
    container_name: my-backend
    ports:
      - "8085:80"
    depends_on:
      - database
    environment:
      # 백엔드 설정도 비밀 장부의 값을 활용합니다
      DB_HOST: database
      DB_PASSWORD: ${DB_PASSWORD}

```




