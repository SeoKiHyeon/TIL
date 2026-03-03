# 🏗️ SSAFY 2학기 인프라 기초 정복 노트 (Day 1)

## 1. Linux & CLI (서버와 대화하는 법) 🐧
서버의 '땅'인 리눅스 환경에서 길을 찾고 도구를 다루는 기초 명령어입니다.

| 명령어 | 설명 | 비유 |
| :--- | :--- | :--- |
| `pwd` | 현재 위치한 경로 출력 | "나 지금 어디 있어?" 📍 |
| `ls` | 현재 폴더의 파일/폴더 목록 확인 (`-al` 옵션: 상세 정보 및 숨김 파일 포함) | "내 옆에 뭐 있어?" 👀 |
| `cd` | 폴더 이동 (`cd ..`은 상위 폴더로 이동) | "저 방으로 갈래!" 🏃 |
| `mkdir` | 새로운 폴더(디렉토리) 생성 | "새로운 방 만들기" 🏠 |
| `touch` | 빈 파일 생성 | "빈 종이 놓기" 📄 |
| `echo "내용" > 파일명` | 파일에 내용을 쓰거나 생성 | "종이에 글씨 쓰기" ✍️ |
| `cat` | 파일 내용 전체 출력 | "내용 읽기" 📖 |
| `tail -f` | 파일의 뒷부분을 실시간으로 확인 (주로 로그 확인용) | "최신 뉴스 실시간 보기" 📜 |
| `rm` | 파일 삭제 (`-rf` 옵션: 폴더와 하위 내용물까지 강제 삭제) | "물건 치우기 / 방 부수기" ⚠️ |
| `vi` / `vim` | 터미널용 텍스트 에디터 | "서버 안의 메모장" ⌨️ |

---

## 2. Docker (컨테이너 기술) 📦
어디서나 동일한 환경을 실행할 수 있게 해주는 '상자' 기술입니다.

### 💡 핵심 개념
* **Docker Image**: 프로그램 실행에 필요한 모든 것이 담긴 설계도 (붕어빵 틀)
* **Docker Container**: 이미지를 실행시킨 실제 상태 (붕어빵)
* **Docker Daemon**: 실제로 명령을 받아 컨테이너를 관리하는 엔진 (엔진룸)

### 🚀 주요 명령어
* `docker run`: 이미지를 내려받고 컨테이너를 생성 및 실행
```
Ex) docker run -d -p 8080:80 --name my-web nginx
-d (Detached): 백그라운드에서 실행하라는 뜻 (터미널을 꺼도 서버가 돌아감)
-p 8080:80 (Port Forwarding): 매우 중요! 내 컴퓨터의 8080번 포트로 들어오면, 도커 컨테이너 내부의 80번 포트로 연결해주라는 뜻
--name my-web: 이 컨테이너의 이름을 my-web으로 짓겠다
```
* `docker ps`: 현재 실행 중인 컨테이너 목록 확인 (`-a` 옵션: 종료된 것 포함)
* `docker images`: 내 컴퓨터에 다운로드된 이미지 목록 확인
* `docker stop/start`: 컨테이너 일시 정지 및 재시작
* `docker rm -f`: 실행 중인 컨테이너 강제 삭제
```
Ex) 
# 1. 기존 삭제
docker rm -f my-custom-web

# 2. 경로 앞에 '/' 하나 더 추가 (//$(pwd) 로 시작)
docker run -d -p 8081:80 --name my-custom-web -v "//$(pwd)":/usr/share/nginx/html nginx
```
* `docker logs`: 컨테이너 내부에서 발생하는 로그 확인
* `docker exec -it [이름] bash`: 실행 중인 컨테이너 내부로 직접 들어가기
```
Ex) docker exec my-custom-web ls /usr/share/nginx/html
```

---

## 3. Network & Volume (통로와 저장소) 🔌
* **Port Forwarding (-p 8080:80)**: 외부 대문(8080)과 내부 방 번호(80)를 연결하는 통로입니다.
* **Volume Mount (-v)**: 내 컴퓨터의 폴더와 컨테이너 내부 폴더를 동기화하여 데이터를 영구 저장합니다.
    * *Windows Git Bash 팁*: 경로 앞에 `//$(pwd)`처럼 슬래시를 추가해 경로 인식 문제를 해결합니다.

---

## 4. Docker Compose (인프라 오케스트레이션) 🏗️
여러 개의 컨테이너(백엔드, DB 등)를 하나의 파일로 관리하는 '요리책'입니다.

* **파일**: `docker-compose.yml` (YAML 형식을 사용하며 들여쓰기가 매우 중요!)
```
version: '3.8'

services:
  # 웹 서버 (Nginx)
  web-server:
    image: nginx
    container_name: my-compose-web
    ports:
      - "8082:80"
    volumes:
      - "./:/usr/share/nginx/html" # 현재 폴더와 연결
    restart: always

  # (예시) 나중에 추가할 백엔드 자리
  # backend:
  #   image: springboot-app
  #   ...

```
-> ./ 의 마법: "이 폴더 안에 있는 거 다 가져가!"


* **실행**: `docker-compose up -d` (모든 서비스를 한꺼번에 백그라운드에서 실행)
* **장점**: 복잡한 `docker run` 옵션들을 파일 하나에 기록하여 실수 없이 관리할 수 있습니다.





- 여기까지는 docker-compose.yml 파일은 현재 Nginx 하나만 관리하고 있음. 이후에는 백엔드와 DB를 추가하여 같은 네트워크에서 작동하도록 해볼거다.
