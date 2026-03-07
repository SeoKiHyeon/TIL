# Jenkins CI/CD 파이프라인 구축 - GitLab 자동 배포

## 전체 흐름 요약

```
GitLab Push → Webhook → Jenkins → Docker Build → 컨테이너 배포 → EC2 서버에서 실서비스
```

- 코드를 GitLab에 push하면 → Jenkins가 감지하고 → Docker 이미지를 빌드하고 → 컨테이너를 재시작하는 자동화 파이프라인

---

## CI/CD란?

| 용어 | 뜻 | 예시 |
|---|---|---|
| CI (Continuous Integration) | 코드 변경 시 자동으로 빌드/테스트 | git push → 자동 빌드 |
| CD (Continuous Delivery) | 빌드 결과를 자동으로 배포 | 빌드 성공 → 자동 서버 배포 |

Jenkins는 CI/CD 도구 중 가장 널리 사용되는 오픈소스 자동화 서버.

---

## 아키텍처

```
[로컬 개발자]
     │
     │ git push
     ▼
[GitLab 원격 저장소]
     │
     │ Webhook (HTTP POST)
     ▼
[EC2 서버 - Jenkins 컨테이너 :8080]
     │
     │ Jenkinsfile 실행
     ▼
[Docker Build → 컨테이너 배포 :80]
     │
     ▼
[브라우저에서 http://EC2_IP 접속]
```

---

## 1단계: EC2에 Jenkins 설치

### apt로 직접 설치 시도 → 실패

처음에는 EC2에 Jenkins를 직접 설치하려 했으나 GPG 키 오류 발생:

```
W: GPG error: https://pkg.jenkins.io/debian-stable binary/ Release:
   The following signatures couldn't be verified because the public key is not available:
   NO_PUBKEY 7198F4B714ABFC68
```

**원인**: Jenkins 공식 저장소의 GPG 키가 만료되거나 시스템에 없음
**해결**: apt 설치 대신 Docker 컨테이너로 Jenkins 실행으로 방향 전환

---

### Docker 컨테이너로 Jenkins 실행 (권장 방법)

```bash
docker run -d \
  -p 8080:8080 \
  -p 50000:50000 \
  -v jenkins_home:/var/jenkins_home \
  -v /var/run/docker.sock:/var/run/docker.sock \
  --name jenkins \
  jenkins/jenkins:lts
```

**각 옵션 설명:**

| 옵션 | 설명 |
|---|---|
| `-p 8080:8080` | Jenkins 웹 UI 포트 |
| `-p 50000:50000` | Jenkins 에이전트 통신 포트 |
| `-v jenkins_home:/var/jenkins_home` | Jenkins 설정/데이터 영속화 (볼륨) |
| `-v /var/run/docker.sock:/var/run/docker.sock` | 호스트의 Docker 소켓 마운트 (핵심!) |
| `--name jenkins` | 컨테이너 이름 |
| `jenkins/jenkins:lts` | 공식 Jenkins LTS 이미지 |

> **docker.sock 마운트가 중요한 이유**
> Jenkins 컨테이너 안에서 `docker build`, `docker run` 같은 명령어를 실행하려면
> 호스트의 Docker 데몬과 통신해야 함.
> `/var/run/docker.sock`를 마운트하면 컨테이너 안에서도 호스트 Docker를 제어 가능.

---

### Jenkins 컨테이너 안에 docker 설치

컨테이너 안에는 docker 바이너리가 없음. 소켓만 있어도 바이너리가 없으면 명령어 실행 불가.

```bash
# 컨테이너 안에 docker.io 설치
docker exec -u root jenkins bash -c "apt-get update && apt-get install -y docker.io"

# docker 소켓 파일 권한 설정 (jenkins 유저가 접근 가능하도록)
docker exec -u root jenkins bash -c "chmod 666 /var/run/docker.sock"
```

설치 확인:

```bash
docker exec jenkins docker ps
```

Jenkins 컨테이너 자신이 출력되면 성공.

---

### AWS 보안 그룹 설정

EC2 인스턴스 보안 그룹에서 아래 포트 허용 필요:

| 포트 | 프로토콜 | 용도 | 소스 |
|---|---|---|---|
| 22 | TCP | SSH 접속 | 내 IP |
| 8080 | TCP | Jenkins 웹 UI | 내 IP (또는 0.0.0.0/0) |
| 80 | TCP | 배포된 웹서비스 | 0.0.0.0/0 |

---

## 2단계: Jenkins 초기 설정

### 초기 비밀번호 확인

```bash
docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword
```

브라우저에서 `http://EC2_퍼블릭IP:8080` 접속 → 비밀번호 입력

### 플러그인 설치

- "Install suggested plugins" 선택 (권장 플러그인 자동 설치)
- GitLab 연동을 위해 **GitLab Plugin** 추가 설치 필요
  - Manage Jenkins → Plugins → Available Plugins → "GitLab" 검색 → 설치

---

## 3단계: GitLab Credential 등록

Jenkins에서 GitLab 저장소에 접근하려면 인증 정보 등록 필요.

**경로**: Manage Jenkins → Credentials → System → Global credentials → Add Credentials

| 항목 | 값 |
|---|---|
| Kind | Username with password |
| Username | GitLab 아이디 |
| Password | GitLab 비밀번호 또는 Access Token |
| ID | 아무 이름 (예: `gitlab-credentials`) |

---

## 4단계: Pipeline Job 생성

1. Jenkins 대시보드 → **New Item**
2. 이름 입력 (예: `gitlab-pipeline`)
3. **Pipeline** 선택 → OK
4. 설정:
   - **Build Triggers**: "Build when a change is pushed to GitLab" 체크
   - **Pipeline → Definition**: "Pipeline script from SCM" 선택
   - **SCM**: Git
   - **Repository URL**: GitLab 저장소 주소
   - **Credentials**: 위에서 등록한 Credential 선택
   - **Branch**: `*/main`
   - **Script Path**: `INFRA/docker-practice/Jenkinsfile` (저장소 루트 기준 경로)

> **Script Path 주의**
> Jenkinsfile이 저장소 루트가 아닌 하위 폴더에 있으면 경로를 정확히 입력해야 함.
> 잘못 입력하면 "Jenkinsfile not found" 오류 발생.

---

## 5단계: GitLab Webhook 설정

GitLab이 Jenkins에게 "push가 발생했다"는 신호를 보내는 설정.

**GitLab 경로**: 저장소 → Settings → Webhooks → Add new webhook

| 항목 | 값 |
|---|---|
| URL | `http://EC2_IP:8080/project/gitlab-pipeline` |
| Trigger | Push events 체크 |
| SSL verification | 비활성화 (HTTP인 경우) |

---

## 발생했던 오류들 & 해결

### 오류 1: Webhook 403 - No valid crumb

```
Hook execution failed: URL 'http://...' is blocked: Requests to the local network are not allowed
또는
403 Forbidden - No valid crumb was included in the request
```

**원인**: Jenkins의 CSRF Protection이 외부 요청(Webhook)을 막음
**해결**: Manage Jenkins → Security → CSRF Protection → **Enable proxy compatibility** 체크

---

### 오류 2: Webhook 403 - anonymous is missing Job/Build permission

```
HTTP/1.1 403 anonymous is missing the Job/Build permission
```

**원인**: GitLab Webhook 요청은 인증 없이 들어오는데, Jenkins가 익명 사용자의 Job 실행을 막음
**해결**: Manage Jenkins → Security → Authorization → **Matrix-based security** 선택
→ Anonymous 행에서 **Job → Build** 체크

> **보안 주의**: 실제 운영 환경에서는 Secret Token 방식 사용 권장
> Webhook URL에 token을 포함하거나, Jenkins API token으로 인증하는 방식이 더 안전함.

---

### 오류 3: docker: not found

```
/var/jenkins_home/workspace/gitlab-pipeline@tmp/durable-xxx/script.sh.copy: 1: docker: not found
ERROR: script returned exit code 127
```

**원인**: Jenkins 컨테이너 안에 docker 바이너리가 없음
**해결**:
1. 컨테이너를 `/var/run/docker.sock` 마운트 포함으로 재생성
2. 컨테이너 안에 docker.io 설치

```bash
# 기존 컨테이너 제거 후 소켓 마운트 포함하여 재생성
docker stop jenkins && docker rm jenkins

docker run -d \
  -p 8080:8080 \
  -p 50000:50000 \
  -v jenkins_home:/var/jenkins_home \
  -v /var/run/docker.sock:/var/run/docker.sock \
  --name jenkins \
  jenkins/jenkins:lts

# docker.io 설치 + 소켓 권한 설정
docker exec -u root jenkins bash -c "apt-get update && apt-get install -y docker.io"
docker exec -u root jenkins bash -c "chmod 666 /var/run/docker.sock"
```

---

### 오류 4: Jenkins 컨테이너 Exited (137)

```bash
docker ps -a
# STATUS: Exited (137) 2 minutes ago
```

**원인**: 종료 코드 137 = 128 + 9 (SIGKILL) = OOM 또는 강제 종료
**해결**: `docker start jenkins` 로 재시작

---

### 오류 5: SSH 접속 타임아웃

```
ssh: connect to host x.x.x.x port 22: Connection timed out
```

**원인**: 네트워크가 변경되면 내 IP가 바뀌는데, 보안 그룹에 등록된 IP가 이전 IP
**해결**:
```bash
# 현재 내 IP 확인
curl -4 ifconfig.me
```
AWS 콘솔 → EC2 → 보안 그룹 → SSH 규칙 소스 IP 업데이트

---

## 6단계: Jenkinsfile 작성

Jenkins 파이프라인의 실행 로직을 코드로 정의한 파일 (Pipeline as Code).

```groovy
pipeline {
    agent any

    stages {
        stage('Checkout') {
            steps {
                echo '코드를 가져왔습니다.'
                checkout scm  // 저장소에서 코드를 가져옴
            }
        }

        stage('Build') {
            steps {
                echo 'Docker 이미지를 빌드합니다.'
                sh 'docker build -t my-app ./INFRA/docker-practice'
            }
        }

        stage('Deploy') {
            steps {
                echo '배포를 시작합니다.'
                sh 'docker stop my-app || true'   // 실행 중이면 중지 (없어도 에러 무시)
                sh 'docker rm my-app || true'     // 컨테이너 삭제 (없어도 에러 무시)
                sh 'docker run -d -p 80:80 --name my-app my-app'  // 새 컨테이너 실행
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

**Jenkinsfile 구조 설명:**

| 키워드 | 설명 |
|---|---|
| `pipeline` | 파이프라인 전체를 감싸는 블록 |
| `agent any` | 사용 가능한 아무 에이전트(노드)에서 실행 |
| `stages` | 실행할 단계들의 묶음 |
| `stage('이름')` | 각 단계 (UI에서 진행 상황 표시) |
| `steps` | 단계 안에서 실행할 명령들 |
| `sh '...'` | 쉘 명령어 실행 |
| `post` | 파이프라인 완료 후 실행 (성공/실패 여부에 따라) |
| `checkout scm` | 파이프라인이 연결된 저장소에서 코드 체크아웃 |

---

## 전체 명령어 정리

### Jenkins 컨테이너 관리

```bash
# Jenkins 컨테이너 실행 (docker.sock 마운트 포함 - 필수!)
docker run -d \
  -p 8080:8080 \
  -p 50000:50000 \
  -v jenkins_home:/var/jenkins_home \
  -v /var/run/docker.sock:/var/run/docker.sock \
  --name jenkins \
  jenkins/jenkins:lts

# 초기 비밀번호 확인
docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword

# Jenkins 안에 docker 설치
docker exec -u root jenkins bash -c "apt-get update && apt-get install -y docker.io"

# docker 소켓 권한 설정
docker exec -u root jenkins bash -c "chmod 666 /var/run/docker.sock"

# Jenkins 컨테이너 안에서 docker 동작 확인
docker exec jenkins docker ps

# Jenkins 재시작
docker restart jenkins

# 컨테이너 상태 확인 (종료된 것 포함)
docker ps -a
```

---

## 최종 파일 구조

```
docker-practice/
├── Dockerfile          # nginx 이미지 빌드 정의
├── Jenkinsfile         # CI/CD 파이프라인 정의
└── index.html          # 배포할 HTML 파일
```

### Dockerfile

```dockerfile
FROM nginx:latest
COPY index.html /usr/share/nginx/html/index.html
```

---

## 전체 자동 배포 흐름 정리

```
1. 로컬에서 index.html 수정
2. git add . && git commit -m "update" && git push
3. GitLab → Jenkins로 Webhook 전송 (HTTP POST to :8080/project/gitlab-pipeline)
4. Jenkins가 Webhook 수신 → 파이프라인 자동 시작
5. [Checkout] GitLab에서 최신 코드 pull
6. [Build]   docker build -t my-app ./INFRA/docker-practice
7. [Deploy]  기존 my-app 컨테이너 중지/삭제 → 새 컨테이너 실행 (-p 80:80)
8. 브라우저에서 http://EC2_퍼블릭IP 접속 → 변경된 페이지 확인
```

---

## 핵심 개념 요약

### Jenkins vs. 직접 배포 차이

| 방식 | 과정 |
|---|---|
| 직접 배포 | 코드 수정 → SCP로 서버 전송 → SSH 접속 → docker 명령 직접 실행 |
| Jenkins CI/CD | 코드 수정 → git push → 끝 (나머지는 자동) |

### docker.sock 마운트의 의미

```
호스트 EC2
├── Docker 데몬 (dockerd)
│     └── /var/run/docker.sock  ← 소켓 파일
└── Jenkins 컨테이너
      └── /var/run/docker.sock  ← 호스트 소켓을 그대로 마운트

결과: Jenkins 컨테이너 안에서 실행하는 docker 명령이 호스트 Docker를 제어함
     = Jenkins가 빌드한 이미지, 실행한 컨테이너가 호스트에 그대로 반영됨
```

### Matrix-based security에서 Anonymous 권한이 필요한 이유

GitLab Webhook은 인증 없이 Jenkins에 HTTP 요청을 보냄.
Jenkins는 이 요청을 "익명 사용자(Anonymous)"로 처리함.
Anonymous에게 Job/Build 권한이 없으면 → 403 Forbidden.

> 실무에서는 보안을 위해 Webhook Secret Token을 설정하거나
> Jenkins API Token을 Webhook URL에 포함시켜 인증을 강화함.
