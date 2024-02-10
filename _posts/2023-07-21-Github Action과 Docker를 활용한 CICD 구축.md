---
title: Github Action과 Docker를 활용한 CI/CD 구축
date: 2023-07-21 12:00:00 +09:00
categories: [SpringBoot, CI/CD]
tags: [Java, SpringBoot, Github Actions, JPA]     
---

Github Actions을 이용해 코드가 머지되었을 때 ec2서버로 자동으로 image를 만들고 docker run 하는 방법을 알아보겠습니다.

## Ec2 접속

<hr style="height: 2px; border: none; background-color: white;" />

먼저 호스팅 서버 ec2에 접속을 해줍니다.

```shell
ssh -i ec2-key.pem ec2-user@${EC2_PUBLIC_IP}
```

pem 키는 EC2 인스턴스에 접속하기 위한 비밀 키입니다. 이 비밀 키는 EC2 인스턴스에 연결할 때 사용되며, EC2 인스턴스에 연결하려면 해당 키를 가지고 있어야 합니다.

EC2 인스턴스에는 `.ssh/authorized_keys` 파일에 공개 키가 저장되어 있습니다. 따라서 SSH 연결 시에는 키 쌍을 이용하여 공개 키와 비밀 키를 서로 비교하여 인증을 수행합니다.

> **<font color='dodgerblue'>새로운 키 쌍을 생성하려면 ssh-keygen 명령어를 사용하여 공개 키와 비밀 키를 생성합니다. 그 후에 생성된 공개 키를 EC2 인스턴스의 .ssh/authorized_keys 파일에 저장하면 해당 비밀 키로도 EC2 인스턴스에 연결할 수 있습니다.</font>**

<br>

## Dockerfile

<hr style="height: 2px; border: none; background-color: white;" />

Jar 파일을 사용하여 Docker 이미지를 생성하기 위해서는 Dockerfile이 필요합니다. 

따라서 EC2 서버에서 SCP를 통해 전달받은 Jar 파일을 활용하여 Docker 이미지를 생성하기 위해 Dockerfile을 EC2에 생성해야 합니다.

```shell
FROM amazoncorretto:17-alpine3.16
ARG JAR_FILE=source/build/libs/status-page-server-0.0.1-SNAPSHOT.jar
COPY ${JAR_FILE} app.jar
ENTRYPOINT ["java","-jar","/app.jar"]
```

이 Dockerfile은 EC2 인스턴스의 JAR_FILE 경로에 있는 jar 파일을 app.jar로 복사하여 Docker 이미지 내부로 가져오고, 컨테이너가 시작되면 Java VM에서 해당 app.jar 파일이 실행됩니다.

<br>

## github 환경변수 등록

<hr style="height: 2px; border: none; background-color: white;" />

GitHub에는 데이터베이스 비밀번호와 같은 민감한 정보를 코드에 직접 넣는 것보다는 환경 변수를 사용하여 관리하는 것이 안전합니다. 아래 내용은 GitHub Secrets에 등록하여 사용한 내용입니다.

![image](https://github.com/rlatmd0829/rlatmd0829.github.io/assets/70622731/19abf86d-cf84-4035-82b8-056d230e52f3)

<br>

## Github Actions CI/CD 

<hr style="height: 2px; border: none; background-color: white;" />

GitHub Actions 워크플로우는 다음과 같습니다.

- pull Request는 테스트 코드만 실행, push는 전체 배포 실행
- JDK 설정
- gradlew 실행권한 부여 및 빌드
- scp를 이용해 로컬 `build/libs/*.jar` 에 있는 jar파일을 ec2서버 `source` 폴더로 이동
- docker 명령어를 실행
  - Dockerfile을 이용하여 docker image 생성
  - 환경변수 넣고 docker run

```yaml
name: Java CI with Gradle

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  build-and-deploy:
    if: github.event_name == 'push'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Set up JDK 17
        uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'adopt'
          cache: gradle
      - name: Grant execute permission for gradlew
        run: chmod +x gradlew
      - name: Build with Gradle
        run: ./gradlew build
      - name: Deliver file
        uses: appleboy/scp-action@master
        with:
          host: 호스팅 서버의 SSH 호스트 주소
          username: 호스팅 서버의 SSH 이름
          key: SSH 연결에 사용되는 키
          port: 호스팅 서버의 포트 번호
          source: "build/libs/*.jar"
          target: "source"
          rm: true
      - name: Deploy new version of server
        uses: appleboy/ssh-action@master
        with:
          host: 호스팅 서버의 SSH 호스트 주소
          username: 호스팅 서버의 SSH 이름
          key: SSH 연결에 사용되는 키
          port: 호스팅 서버의 포트 번호
          script: |
            export DB_HOST= 호스팅 서버의 SSH 호스트 주소
            export DB_USERNAME= 호스팅 서버의 SSH 이름
            export DB_PASSWORD= 호스팅 서버의 SSH 비밀번호
            export ACTIVE_PROFILE= active 프로필
            
            docker stop status-page-api || true
            docker rm status-page-api || true
            docker rmi status-page-image || true
            
            docker build -t status-page-image .

            docker run -d -p 8080:8080 --name status-page-api -e DB_HOST=$DB_HOST -e DB_USERNAME=$DB_USERNAME -e DB_PASSWORD=$DB_PASSWORD -e SPRING_PROFILES_ACTIVE=$ACTIVE_PROFILE status-page-image:latest

  test:
    if: github.event_name == 'pull_request'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Set up JDK 17
        uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'adopt'
          cache: gradle
      - name: Grant execute permission for gradlew
        run: chmod +x gradlew
      - name: Run tests
        run: ./gradlew test
```

> **<font color='dodgerblue'>DB_HOST, DB_USERNAME, DB_PASSWORD 등 직접 값을 넣어주기 보다는 GitHub Secrets에 환경변수를 넣어주고 ${{ secrets.QUACK_RUN_SSH_HOST }} 이런식으로 사용하는게 좋습니다.</font>**

<br>

## 개선사항

<hr style="height: 2px; border: none; background-color: white;" />

### 현재 CI/CD 로직

- 현재는 scp로 호스팅 서버로 jar 파일을 보내고 있다
- 호스팅 서버에서 jar를 받아 dockerfile을 이용해 빌드한다
- 스크립트를 사용해 호스팅 서버에서 dockerfile을 이용해 이미지 빌드하고 run을 하고있다
  - Docker run할때 환경변수가 굉장히 길다


### 개선할부분

**로컬에서 Dockerfile 빌드 및 Docker Hub에 푸시**

- 로컬에서 Dockerfile을 사용하여 이미지를 빌드합니다.
- 빌드된 이미지를 Docker Hub에 푸시합니다.

**호스팅 서버에서 Docker 이미지 Pull**

- 호스팅 서버에서는 Docker Hub에서 해당 이미지를 Pull 받습니다. 

**환경 변수 관리를 위한 Docker Compose 사용**

- 호스팅 서버에서는 Docker Compose 파일을 사용하여 환경 변수를 관리합니다.
- Docker Compose 파일에는 실행에 필요한 모든 환경 변수를 설정합니다.

