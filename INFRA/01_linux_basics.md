# Linux 기초

## 1. Shell과 CLI

### Shell이란

Shell은 사용자가 운영체제(OS)에 명령을 내릴 수 있도록 중간에서 해석해주는 프로그램이다.

```
사용자 → Shell → OS(Kernel) → 하드웨어
```

- **GUI (Graphic User Interface)**: 마우스로 클릭하는 방식 (Windows 탐색기, macOS Finder)
- **CLI (Command Line Interface)**: 텍스트 명령어를 입력하는 방식 (Terminal, Bash)

서버 환경에서 CLI를 사용하는 이유:
- 서버는 대부분 화면(모니터)이 없는 원격 컴퓨터다.
- GUI보다 자원 소비가 적고, 자동화(스크립트)에 유리하다.
- SSH를 통해 원격에서 텍스트만으로 서버를 제어할 수 있다.

---

## 2. 리눅스 파일시스템 구조

리눅스는 모든 것을 하나의 트리 구조로 관리한다. 최상위는 `/` (루트)다.

```
/
├── home/        # 일반 사용자 디렉토리 (예: /home/ubuntu)
├── etc/         # 시스템 설정 파일들
├── var/         # 로그, 데이터베이스 등 변동되는 파일들
├── usr/         # 설치된 프로그램과 라이브러리
├── tmp/         # 임시 파일 (재부팅 시 삭제)
└── root/        # root(관리자) 사용자의 홈 디렉토리
```

### 경로 표기 방식

| 종류 | 설명 | 예시 |
|---|---|---|
| 절대 경로 | 루트(`/`)부터 시작하는 전체 경로 | `/home/ubuntu/project` |
| 상대 경로 | 현재 위치 기준으로 표기 | `./project`, `../config` |

- `.` : 현재 디렉토리
- `..` : 상위 디렉토리
- `~` : 현재 사용자의 홈 디렉토리

---

## 3. 기본 명령어

### 탐색

```bash
# 현재 위치한 디렉토리의 절대 경로 출력
pwd

# 현재 디렉토리의 파일/폴더 목록 출력
ls
ls -l      # 상세 정보 (권한, 소유자, 크기, 날짜) 포함
ls -a      # 숨김 파일(.으로 시작하는 파일) 포함
ls -al     # 상세 정보 + 숨김 파일

# 디렉토리 이동
cd /home/ubuntu    # 절대 경로로 이동
cd ..              # 상위 디렉토리로 이동
cd ~               # 홈 디렉토리로 이동
```

### 생성 / 삭제

```bash
# 디렉토리 생성
mkdir my-folder
mkdir -p a/b/c     # 중간 디렉토리가 없어도 한 번에 생성 (-p 옵션)

# 빈 파일 생성 (또는 파일의 수정 시간을 현재로 갱신)
touch index.html

# 파일 삭제
rm file.txt
rm -r my-folder    # 디렉토리(폴더)와 내부 내용 모두 삭제 (-r: recursive)
rm -rf my-folder   # 확인 메시지 없이 강제 삭제 (-f: force) — 주의해서 사용
```

> **주의**: `rm -rf`는 복구가 불가능하다. 경로를 반드시 확인 후 실행할 것.

### 파일 내용 확인

```bash
# 파일 전체 내용 출력
cat file.txt

# 파일의 끝 부분을 실시간으로 출력 (로그 모니터링에 주로 사용)
tail -f /var/log/nginx/access.log

# 파일 내용을 페이지 단위로 출력 (q로 종료)
less file.txt
```

### 파일 내용 쓰기

```bash
# 파일에 내용 쓰기 (기존 내용 덮어쓰기)
echo "Hello World" > file.txt

# 파일에 내용 추가 (기존 내용 유지)
echo "new line" >> file.txt
```

---

## 4. 텍스트 편집기: Vim

서버에 파일을 직접 수정해야 할 때 사용하는 터미널 기반 에디터다.

```bash
vim file.txt
```

### Vim의 두 가지 모드

| 모드 | 진입 방법 | 역할 |
|---|---|---|
| **Normal 모드** | 기본 진입 상태 / `Esc` | 커서 이동, 명령 입력 |
| **Insert 모드** | `i` 키 | 텍스트 입력/수정 |

### 자주 쓰는 명령어

```
i          → Insert 모드 진입 (텍스트 입력 시작)
Esc        → Normal 모드로 돌아오기
:w         → 저장 (write)
:q         → 종료 (quit)
:wq        → 저장 후 종료
:q!        → 저장하지 않고 강제 종료
```

---

## 5. 파일 권한 (Permission)

`ls -l` 출력 결과를 읽는 법:

```
-rwxr-xr-- 1 ubuntu ubuntu 1234 Jan 1 12:00 script.sh
```

| 부분 | 의미 |
|---|---|
| `-` | 파일 타입 (`-`: 일반 파일, `d`: 디렉토리) |
| `rwx` | 소유자(owner) 권한: 읽기(r), 쓰기(w), 실행(x) |
| `r-x` | 그룹(group) 권한 |
| `r--` | 기타(others) 권한 |

```bash
# 실행 권한 추가 (스크립트 파일 실행 전에 필요)
chmod +x script.sh

# 소유자만 읽기 가능 (SSH 키 파일에 사용)
chmod 400 my-key.pem

# 소유자 변경
chown ubuntu:ubuntu file.txt
```

숫자 권한 표기법:

| 숫자 | 의미 |
|---|---|
| `4` | 읽기 (r) |
| `2` | 쓰기 (w) |
| `1` | 실행 (x) |

`chmod 400`의 의미: `소유자(4)` / `그룹(0)` / `기타(0)` → 나만 읽기 가능

> **SSH 키 파일 주의**: `.pem` 파일 권한이 너무 열려있으면 SSH가 보안상 접속을 거부한다. 반드시 `chmod 400`으로 설정해야 한다.

---

## 6. 패키지 관리자

리눅스는 운영체제 종류에 따라 프로그램 설치 방식이 다르다.

| 배포판 | 패키지 관리자 | 설치 명령 |
|---|---|---|
| Ubuntu / Debian | `apt` | `sudo apt install nginx` |
| Amazon Linux / CentOS / RHEL | `yum` / `dnf` | `sudo dnf install docker` |

```bash
# 패키지 목록 업데이트 후 설치 (Ubuntu)
sudo apt update && sudo apt install -y nginx

# 설치 (Amazon Linux 2023)
sudo dnf install -y docker
```

> `sudo`: 관리자(root) 권한으로 명령을 실행한다. 시스템 설정 변경이나 프로그램 설치 시 필요하다.
