### curl 설치
`sudo apt-get install curl`

<br>

### docker 스크립트로 설치
`curl -s https://get.docker.com | sudo sh`

### docker 버전 확인
`sudo docker version`

### docker 그룹에 계정 추가
`sudo usermod -aG docker nobletuna`

### 계정 전환 (전환해야 그룹 반영됨)
`sudo su - nobletuna`

<br>

### docker compose 설치 (1.28.2)
`sudo curl -L https://github.com/docker/compose/releases/download/1.28.2/docker-compose-$(uname -s)-$(uname -m) -o /usr/local/bin/docker-compose`

### docker compose 실행권한 추가
`sudo chmod +x /usr/local/bin/docker-compose`

### docker compose 버전 확인
`docker-compose version`

<br>