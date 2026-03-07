### Docker 설치

```
sudo apt update && sudo apt install -y docker.io

sudo systemctl start docker && sudo systemctl enable docker

sudo usermod -aG docker ubuntu

설치 후 movaXterm을 종료한 뒤 다시 접속하여 docker ps 확인

```


### Docker Compose 설치

```
sudo apt install -y ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list
sudo apt update && sudo apt install -y docker-compose-plugin

설치 후 docker compose version 으로 설치 성공 확인
```

### jenkins 컨테이너 실행

```
docker run -d \
  -p 8080:8080 \
  -p 50000:50000 \
  -v jenkins_home:/var/jenkins_home \
  -v /var/run/docker.sock:/var/run/docker.sock \
  --name jenkins \
  jenkins/jenkins:lts

docker ps로 jenkins 컨테이너가 보이면 성공!
```

### jenkis 컨테이너 안에 docker 설치

```
docker exec -u root jenkins bash -c "apt-get update && apt-get install -y docker.io"
```
