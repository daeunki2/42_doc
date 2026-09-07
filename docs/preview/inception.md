# Inception

## Overview

Inception은 **Docker를 이용하여 여러 서비스를 독립적인 Container로 분리하고 하나의 애플리케이션으로 구성하는 프로젝트**입니다.

Born2beroot에서는 하나의 Linux 서버 자체를 구축하고 관리했다면, Inception에서는 그 서버 위에서 실행되는 애플리케이션들을 Container라는 단위로 분리하여 관리합니다.

Inception의 기본 구성은 다음과 같습니다.

```text
                        Client
                          │
                        HTTPS
                          │
                          ▼
                   ┌─────────────┐
                   │    NGINX    │
                   │  Container  │
                   └──────┬──────┘
                          │
                       FastCGI
                          │
                          ▼
                   ┌─────────────┐
                   │  WordPress  │
                   │   PHP-FPM   │
                   │  Container  │
                   └──────┬──────┘
                          │
                         SQL
                          │
                          ▼
                   ┌─────────────┐
                   │   MariaDB   │
                   │  Container  │
                   └─────────────┘
```

이 세 Container는 Docker Network를 통해 서로 통신하며, 영속적으로 보관해야 하는 데이터는 Volume에 저장합니다.

---

## 1. VM vs Container

Born2beroot에서는 Virtual Machine을 사용했습니다.

VM은 Hypervisor를 이용하여 하드웨어 자원을 가상화하고, 각 VM이 자신의 Guest OS와 Kernel을 가질 수 있습니다.

```text
Physical Machine
       │
     Host OS
       │
   Hypervisor
       │
 ┌─────┴─────┐
 ▼           ▼
VM           VM
│            │
Guest OS     Guest OS
│            │
Kernel       Kernel
```

Container는 일반적으로 Host의 Kernel을 공유하면서 프로세스, 파일 시스템, 네트워크 등의 실행 환경을 격리합니다.

```text
Physical Machine
       │
     Host OS
       │
     Kernel
       │
     Docker
       │
 ┌─────┼─────┐
 ▼     ▼     ▼
 C1    C2    C3
```

따라서 기본적인 차이는 다음과 같이 이해할 수 있습니다.

```text
VM
→ 하나의 독립적인 컴퓨터 환경을 가상화

Container
→ 애플리케이션의 실행 환경을 격리
```

Container는 일반적으로 VM보다 가볍고 빠르게 생성할 수 있습니다.

---

## 2. Docker

Docker는 **Container Image를 만들고, 이를 기반으로 Container를 실행하고 관리하기 위한 플랫폼 및 도구 생태계**입니다.

Docker가 정상적으로 설치되어 있는지 확인:

```bash
docker --version
```

Docker의 전체적인 시스템 정보 확인:

```bash
docker info
```

현재 Docker에서 실행 중인 Container 확인:

```bash
docker ps
```

실행 여부와 관계없이 모든 Container 확인:

```bash
docker ps -a
```

---

## 3. Image와 Container

### Image

Image는 **Container를 생성하기 위한 템플릿**입니다.

```text
Dockerfile
    │
    │ build
    ▼
  Image
    │
    │ run
    ▼
Container
```

현재 가지고 있는 Image 확인:

```bash
docker images
```

또는:

```bash
docker image ls
```

Image 삭제:

```bash
docker image rm <image>
```

### Container

Container는 Image를 기반으로 실제 실행된 인스턴스입니다.

간단한 Container 실행:

```bash
docker run <image>
```

예를 들어:

```bash
docker run nginx
```

Container 중지:

```bash
docker stop <container>
```

다시 시작:

```bash
docker start <container>
```

Container 삭제:

```bash
docker rm <container>
```

---

## 4. Dockerfile

Dockerfile은 **Docker Image를 어떻게 만들 것인지 정의하는 파일**입니다.

예를 들어:

```dockerfile
FROM debian:bookworm

RUN apt-get update
RUN apt-get install -y nginx

COPY nginx.conf /etc/nginx/nginx.conf

CMD ["nginx", "-g", "daemon off;"]
```

전체 흐름은 다음과 같습니다.

```text
Dockerfile
    │
    │ docker build
    ▼
  Image
    │
    │ docker run
    ▼
Container
```

현재 디렉터리의 Dockerfile을 이용하여 Image 생성:

```bash
docker build -t my-nginx .
```

생성 확인:

```bash
docker images
```

실행:

```bash
docker run my-nginx
```

### FROM

```dockerfile
FROM debian:bookworm
```

Image를 만들 때 기반으로 사용할 Base Image를 지정합니다.

### RUN

```dockerfile
RUN apt-get update
RUN apt-get install -y nginx
```

Image를 **빌드하는 과정에서** 명령을 실행합니다.

### COPY

```dockerfile
COPY nginx.conf /etc/nginx/nginx.conf
```

Host의 파일을 Image 안으로 복사합니다.

### CMD

```dockerfile
CMD ["nginx", "-g", "daemon off;"]
```

Container가 실행될 때 기본적으로 실행할 명령을 지정합니다.

`RUN`과 `CMD`의 차이는 중요합니다.

```text
RUN
→ Image build 과정

CMD
→ Container 실행 시점
```

---

## 5. Container 내부 확인

실행 중인 Container 안에서 직접 명령을 실행할 수도 있습니다.

```bash
docker exec <container> <command>
```

예:

```bash
docker exec nginx ls
```

Container 안에 Shell로 들어가려면:

```bash
docker exec -it <container> /bin/bash
```

Container가 가벼운 Image라 `bash`가 없다면 `/bin/sh`를 사용할 수도 있습니다.

```bash
docker exec -it <container> /bin/sh
```

이 명령은 Inception을 디버깅할 때 매우 유용합니다.

---

## 6. Log

Container가 정상적으로 동작하지 않는다면 가장 먼저 확인할 것 중 하나가 Log입니다.

```bash
docker logs <container>
```

실시간으로 계속 확인:

```bash
docker logs -f <container>
```

예:

```bash
docker logs wordpress
```

문제가 발생했을 때 무작정 Container 안으로 들어가기 전에 **먼저 Log를 확인하는 습관**이 중요합니다.

---

## 7. Docker Network

Container는 서로 격리되어 있기 때문에 서로 통신하기 위한 Network가 필요합니다.

```text
             Docker Network
                   │
        ┌──────────┼──────────┐
        ▼          ▼          ▼
      NGINX    WordPress   MariaDB
```

Docker Network 목록 확인:

```bash
docker network ls
```

특정 Network의 상세 정보 확인:

```bash
docker network inspect <network>
```

Network 생성:

```bash
docker network create my-network
```

Docker Compose를 사용하는 경우 필요한 Network를 Compose가 생성하고 관리하도록 구성할 수 있습니다.

### Container 이름을 이용한 통신

같은 Docker Network에 연결된 Container는 Docker의 DNS를 이용하여 서비스 또는 Container 이름으로 서로를 찾을 수 있습니다.

예를 들어 WordPress에서 MariaDB에 접근할 때:

```text
mariadb:3306
```

처럼 사용할 수 있습니다.

즉 항상 Container의 IP 주소를 직접 알아낼 필요가 없습니다.

```text
WordPress
    │
    │ mariadb:3306
    ▼
Docker DNS
    │
    ▼
MariaDB Container
```

---

## 8. Port

Container 내부에서 서비스가 실행되고 있다고 해서 반드시 Host 외부에서도 접근할 수 있는 것은 아닙니다.

예를 들어 NGINX가 Container 내부의 443 Port를 사용한다고 해봅시다.

외부에서 접근할 수 있도록 Host Port와 연결할 수 있습니다.

```text
Host :443
   │
   ▼
Container :443
```

Docker에서는 다음과 같이 실행할 수 있습니다.

```bash
docker run -p 443:443 my-nginx
```

형식은:

```text
-p HOST_PORT:CONTAINER_PORT
```

입니다.

Inception에서는 외부 사용자가 접근해야 하는 NGINX를 통해 요청을 받고, WordPress와 MariaDB는 내부 Docker Network를 통해 통신하도록 구성합니다.

```text
Internet
   │
   │ 443
   ▼
 NGINX
   │
   │ Docker Network
   ▼
WordPress
   │
   │ Docker Network
   ▼
MariaDB
```

모든 서비스를 외부에 공개할 필요가 없습니다.

---

## 9. Volume

Container는 교체 가능한 실행 환경으로 생각하는 것이 좋습니다.

Container 내부에 중요한 데이터를 직접 저장하고 Container를 삭제하면 해당 데이터도 함께 잃을 수 있습니다.

따라서 영속적으로 보관해야 하는 데이터는 Volume을 사용합니다.

```text
MariaDB Container
       │
       ▼
     Volume
       │
       ▼
  Database Data
```

Container가 삭제되어도 Volume을 유지하면 새 Container가 기존 데이터를 다시 사용할 수 있습니다.

```text
Old Container
      X
      │
   Volume
      │
      ▼
New Container
```

Volume 목록 확인:

```bash
docker volume ls
```

Volume 상세 정보 확인:

```bash
docker volume inspect <volume>
```

Volume 생성:

```bash
docker volume create my-volume
```

Volume 삭제:

```bash
docker volume rm <volume>
```

핵심은:

> **Container의 생명주기와 Data의 생명주기를 분리한다.**

---

## 10. NGINX

NGINX는 Inception에서 외부 요청을 가장 먼저 받는 **Web Server이자 진입점** 역할을 합니다.

```text
Client
   │
 HTTPS
   ▼
 NGINX
```

Inception에서는 NGINX가 HTTPS 연결을 처리하고 PHP 요청을 WordPress의 PHP-FPM으로 전달합니다.

```text
Client
   │
 HTTPS
   ▼
NGINX
   │
 FastCGI
   ▼
PHP-FPM
```

NGINX Container의 로그 확인:

```bash
docker logs nginx
```

Container 내부에서 NGINX 설정 문법 확인:

```bash
docker exec nginx nginx -t
```

---

## 11. TLS / HTTPS

HTTP 통신을 암호화하기 위해 TLS를 사용합니다.

```text
HTTP
→ 암호화되지 않은 웹 통신

HTTPS
→ HTTP + TLS
```

Inception에서는 NGINX가 TLS 연결을 처리합니다.

```text
Browser
   │
 HTTPS
   │
   ▼
NGINX
   │
Certificate
```

따라서 외부에서 애플리케이션으로 들어오는 보안 연결의 시작점이 NGINX입니다.

---

## 12. WordPress와 PHP-FPM

WordPress는 PHP로 만들어진 Web Application입니다.

NGINX는 PHP 코드를 직접 실행하지 않습니다.

따라서 PHP 실행을 담당하는 PHP-FPM이 필요합니다.

```text
NGINX
   │
   │ FastCGI
   ▼
PHP-FPM
   │
   ▼
WordPress PHP Code
```

PHP-FPM은 **PHP FastCGI Process Manager**입니다.

NGINX가 PHP 요청을 PHP-FPM에게 전달하면 PHP-FPM이 PHP 코드를 실행하고 결과를 반환합니다.

---

## 13. MariaDB

MariaDB는 관계형 데이터베이스 관리 시스템(RDBMS)입니다.

WordPress는 사용자, 게시글, 설정 등의 데이터를 MariaDB에 저장합니다.

```text
WordPress
    │
    │ SQL
    ▼
 MariaDB
    │
    ▼
  Volume
```

MariaDB Container가 실행 중인지 확인:

```bash
docker ps
```

로그 확인:

```bash
docker logs mariadb
```

Container 내부의 MariaDB Client를 이용하는 구성이라면 다음과 같은 방식으로 DB에 접속할 수도 있습니다.

```bash
docker exec -it mariadb mariadb -u <user> -p
```

실제 실행 파일 이름은 사용한 MariaDB 버전과 Image 구성에 따라 달라질 수 있습니다.

---

## 14. Docker Compose

지금까지 필요한 것들을 보면 상당히 많습니다.

```text
NGINX
WordPress
MariaDB

Images
Containers
Network
Volumes
Ports
Environment Variables
```

이 모든 것을 `docker run` 명령으로 하나씩 관리하면 복잡해집니다.

Docker Compose를 이용하면 여러 Container와 관련 리소스의 구성을 하나의 YAML 파일에 선언할 수 있습니다.

예:

```yaml
services:

  nginx:
    build: ./nginx
    ports:
      - "443:443"

  wordpress:
    build: ./wordpress

  mariadb:
    build: ./mariadb

networks:
  inception:

volumes:
  wordpress:
  mariadb:
```

즉 Compose 파일은 단순히 명령어를 줄여주는 파일이라기보다:

> **애플리케이션 전체의 실행 구성을 선언하는 파일**

이라고 이해하는 것이 좋습니다.

---

## 15. Docker Compose 기본 명령어

Compose 파일에 정의된 서비스 실행:

```bash
docker compose up
```

백그라운드에서 실행:

```bash
docker compose up -d
```

Image를 다시 Build하면서 실행:

```bash
docker compose up --build
```

서비스 상태 확인:

```bash
docker compose ps
```

로그 확인:

```bash
docker compose logs
```

특정 서비스 로그:

```bash
docker compose logs nginx
```

실시간 로그:

```bash
docker compose logs -f
```

서비스 중지 및 Container/Network 정리:

```bash
docker compose down
```

Image Build:

```bash
docker compose build
```

특정 서비스 안에서 명령 실행:

```bash
docker compose exec <service> <command>
```

예:

```bash
docker compose exec nginx nginx -t
```

### 매우 자주 쓰는 흐름

개발하면서 가장 자주 보게 되는 흐름은 대략 다음과 같습니다.

```bash
docker compose up -d --build
docker compose ps
docker compose logs
docker compose down
```

---

## 16. Environment Variables

Container의 동작에는 환경에 따라 달라지는 설정값이 필요할 수 있습니다.

예:

```text
Database Name
Database User
Database Password
Domain
```

이러한 값을 애플리케이션 코드나 Dockerfile에 직접 하드코딩하기보다는 실행 환경과 분리하여 관리할 수 있습니다.

```text
Image
   +
Environment
   ↓
Container
```

Shell에서 환경 변수 확인:

```bash
env
```

Container 내부 환경 변수 확인:

```bash
docker exec <container> env
```

Docker Compose에서는 `environment`, `env_file`, secrets 등의 방식으로 설정을 전달할 수 있습니다.

비밀번호 같은 민감한 정보는 Git Repository에 그대로 commit하지 않는 것이 중요합니다.

---

## 17. Container Lifecycle

Container는 실행되고, 중지되고, 삭제되고, 다시 생성될 수 있습니다.

```text
Image
  │
  ▼
Create
  │
  ▼
Running
  │
  ▼
Stopped
  │
  ▼
Removed
```

현재 상태 확인:

```bash
docker ps -a
```

Docker Compose를 사용하면 여러 서비스의 lifecycle을 함께 관리할 수 있습니다.

```bash
docker compose up -d
```

↓

```bash
docker compose down
```

중요한 데이터는 Container가 아니라 Volume에 저장하기 때문에 Container를 다시 생성해도 데이터를 유지할 수 있습니다.

---

## 18. Inception 전체 구조

지금까지의 개념을 하나로 합치면 다음과 같습니다.

```text
                         Client
                           │
                         HTTPS
                           │
                           ▼
                  ┌────────────────┐
                  │     NGINX      │
                  │   Container    │
                  └───────┬────────┘
                          │
                       FastCGI
                          │
                          ▼
                  ┌────────────────┐
                  │   WordPress    │
                  │    PHP-FPM     │
                  │   Container    │
                  └───────┬────────┘
                          │
                         SQL
                          │
                          ▼
                  ┌────────────────┐
                  │    MariaDB     │
                  │   Container    │
                  └────────────────┘

                       Docker Network

                    ┌──────┴──────┐
                    ▼             ▼
             WordPress Volume  MariaDB Volume
```

그리고 이 전체 구성을 다음 파일들을 통해 정의합니다.

```text
Dockerfile
     │
     └── Image를 어떻게 만들 것인가?

compose.yaml
     │
     └── Container들을 어떻게 실행하고 연결할 것인가?

configuration
     │
     └── 각 Service를 어떻게 설정할 것인가?

environment / secrets
     │
     └── 환경별 설정값은 무엇인가?
```

---

## 19. 자주 사용하는 Docker 명령어

Inception을 다시 확인하거나 디버깅할 때 최소한 다음 명령어들은 알고 있으면 좋습니다.

| 목적                 | 명령어                                     |
| ------------------ | --------------------------------------- |
| 실행 중인 Container    | `docker ps`                             |
| 모든 Container       | `docker ps -a`                          |
| Image 목록           | `docker images`                         |
| Network 목록         | `docker network ls`                     |
| Volume 목록          | `docker volume ls`                      |
| Container 로그       | `docker logs <container>`               |
| Container 내부 Shell | `docker exec -it <container> /bin/bash` |
| Compose 실행         | `docker compose up -d`                  |
| 다시 Build 후 실행      | `docker compose up -d --build`          |
| Compose 상태         | `docker compose ps`                     |
| Compose 로그         | `docker compose logs`                   |
| Compose 종료         | `docker compose down`                   |

명령어를 전부 암기하는 것보다:

```text
상태 확인
    ↓
docker ps

문제 발견
    ↓
docker logs

내부 확인 필요
    ↓
docker exec

구성 전체 관리
    ↓
docker compose
```

라는 흐름을 이해하는 것이 더 중요합니다.

---

## What I Learned

Born2beroot에서는 **하나의 Linux 서버를 관리하는 방법**을 학습했습니다.

Inception에서는 그 위에서 실행되는 애플리케이션을 **Container라는 독립적인 실행 단위로 분리하고 관리하는 방법**을 학습합니다.

```text
Born2beroot

Linux Server
    │
    └── Services

          ↓

Inception

Linux Server
    │
   Docker
    │
    ├── NGINX Container
    ├── WordPress Container
    └── MariaDB Container
```

여기서 중요한 개념은 다음과 같습니다.

```text
Dockerfile
    ↓
  Image
    ↓
Container
    │
    ├── Network → Container 간 통신
    │
    └── Volume  → Data 영속화

Docker Compose
    ↓
전체 구성 관리
```

하지만 Docker Compose를 이용한 구성은 기본적으로 하나의 Docker 환경에서 여러 Container를 관리하는 데 적합합니다.

서비스의 규모가 커져 여러 머신에 걸쳐 많은 Container를 운영해야 한다면 새로운 문제가 발생합니다.

```text
어떤 서버에서 Container를 실행할 것인가?

Container가 죽으면 누가 복구할 것인가?

필요한 Container 수를 어떻게 유지할 것인가?

여러 Container에 트래픽을 어떻게 전달할 것인가?

여러 서버와 Container의 상태를 어떻게 관리할 것인가?
```

이러한 문제를 해결하기 위해 **Container Orchestration**이 필요해집니다.

그리고 다음 프로젝트인 Inception of Things에서는 Kubernetes를 사용하여 이 문제를 다룹니다.

```text
Born2beroot
     │
     │ Linux / VM
     ▼
 Inception
     │
     │ Docker / Container
     ▼
Inception of Things
     │
     │ Kubernetes
     ▼
Container Orchestration
```
