Inception

From managing a Linux server to running isolated application services.

Born2beroot에서 우리는 Linux Server 한 대를 만들고 관리하는 방법을 배웠습니다.

이제 그 Server 위에 실제 Application을 운영한다고 생각해봅시다.

Linux Server
├── NGINX
├── WordPress / PHP
└── MariaDB

처음에는 각 Program을 Server에 직접 설치하면 될 것처럼 보입니다.

하지만 Application이 늘어나면 새로운 문제가 생깁니다.

Application마다 필요한 Library가 다르다
        ↓
Version이 충돌할 수 있다
        ↓
설정과 File이 한 OS 안에 뒤섞인다
        ↓
다른 Server에서 같은 환경을 다시 만들기 어렵다
        ↓
하나의 Service 문제가 다른 Service에 영향을 줄 수 있다

그래서 Inception은 다음 질문에서 시작합니다.

Application의 실행 환경을 서로 분리하고, 그 환경 자체를 재현 가능하게 만들 수 없을까?

이 문제를 해결하기 위해 Container와 Docker를 사용합니다.

이 문서는 다음 흐름으로 진행합니다.

Born2beroot
Linux Server
    ↓
Application Isolation 문제
    ↓
Container
    ↓
Docker
    ↓
Image
    ↓
Dockerfile
    ↓
Container
    ↓
Network
    ↓
Volume
    ↓
NGINX / WordPress / MariaDB
    ↓
Docker Compose
    ↓
하나의 Multi-Container Application

1. Born2beroot에서 Inception으로

Born2beroot에서는 Server 자체가 관심 대상이었습니다.

Virtual Machine
      ↓
Linux
      ↓
User / Permission
      ↓
SSH / Firewall
      ↓
Service / Storage

이제 Server가 준비되었습니다.

그 위에 Web Application을 운영하려고 합니다.

Linux Server

NGINX
PHP
WordPress
MariaDB

모든 것을 Host OS에 직접 설치할 수도 있습니다.

하지만 예를 들어:

Application A → PHP 8.x 필요
Application B → 다른 PHP 환경 필요

Database A → 특정 MariaDB 설정
Database B → 다른 설정

처럼 서로 다른 Dependency와 Configuration이 필요해질 수 있습니다.

또 새로운 Server를 만들 때마다:

NGINX 설치
PHP 설치
MariaDB 설치
설정 File 복사
Version 맞추기
Permission 설정
Service 시작

을 다시 해야 합니다.

우리가 원하는 것은:

Application
+
Runtime
+
Dependencies
+
Configuration

을 하나의 재현 가능한 실행 단위로 묶는 것입니다.

2. Container

Container가 해결하려는 문제

VM은 독립적인 환경을 만드는 좋은 방법입니다.

그렇다면 Application마다 VM을 하나씩 만들 수도 있습니다.

Physical Server
      ↓
Hypervisor
├── VM 1 → OS + NGINX
├── VM 2 → OS + WordPress
└── VM 3 → OS + MariaDB

하지만 Application 하나를 분리하기 위해 OS 전체를 각각 실행하는 것은 상당한 Overhead가 있습니다.

여기서 질문이 생깁니다.

OS 전체가 아니라 Application 실행 환경만 격리할 수 없을까?

Container는 이 문제를 해결합니다.

Container란?

Container는 Host의 Kernel을 공유하면서 격리된 환경에서 실행되는 Process라고 이해하면 좋습니다.

Virtual Machine

App
↓
Guest OS
↓
Virtual Hardware
↓
Hypervisor
↓
Host


Container

App
↓
Isolated User Space
↓
Host Kernel
↓
Host

VM과 가장 큰 차이는 Container마다 별도의 Kernel을 실행하지 않는다는 것입니다.

VM
VM1 → Kernel
VM2 → Kernel
VM3 → Kernel

Container
Container1 ┐
Container2 ├→ Host Kernel 공유
Container3 ┘

그래서 일반적으로 VM보다 가볍고 빠르게 생성할 수 있습니다.

Isolation은 어떻게 가능한가?

Container는 단순히 Directory를 하나 나누는 것이 아닙니다.

Linux Kernel의 기능을 이용해 Process가 볼 수 있는 환경과 사용할 수 있는 Resource를 제한합니다.

대표적으로:

Namespaces
→ Process / Network / Mount 등의 관점을 격리

cgroups
→ CPU / Memory 등의 Resource 사용을 관리

Filesystem
→ Container별 File System 환경

덕분에 Container 안의 Process는 마치 자신만의 System에서 실행되는 것처럼 보일 수 있습니다.

하지만 VM과 달리 Host Kernel을 공유한다는 점은 계속 기억해야 합니다.

3. Docker

Container 자체는 Linux Kernel이 제공하는 여러 기능을 조합해서 만들 수 있습니다.

하지만 사람이 직접 Namespace, cgroup, Filesystem, Network를 매번 구성하는 것은 번거롭습니다.

그래서 Container를 만들고, 실행하고, 연결하고, 저장하고, 삭제하는 작업을 편하게 관리하는 Platform이 필요합니다.

그 대표적인 도구가 Docker입니다.

User
 │
 │ docker ...
 ▼
Docker CLI
 │
 ▼
Docker Engine / daemon
 │
 ├── Image
 ├── Container
 ├── Network
 └── Volume

Docker CLI와 Docker daemon

우리가 Terminal에서 실행하는:

docker ps

의 docker는 Client입니다.

실제 Container와 Image 등을 관리하는 핵심 Process는 Docker daemon인 dockerd입니다.

docker CLI
    │
    │ Docker API
    ▼
 dockerd
    │
    ├── build
    ├── run
    ├── stop
    ├── network
    └── volume

Docker 상태를 확인한다면:

docker info

Docker Version:

docker version

Linux에서 Docker daemon 상태:

systemctl status docker

Born2beroot에서 배운 systemctl이 여기서 그대로 다시 등장합니다.

4. Image

Container를 실행하려면 먼저 어떤 환경을 실행할 것인지 정의되어 있어야 합니다.

예를 들어 NGINX Container라면:

Linux File System
+
NGINX Binary
+
필요한 Library
+
Configuration

이 필요합니다.

이 실행 환경을 Packaging한 것이 Image입니다.

Image와 Container의 관계

가장 중요한 관계입니다.

Dockerfile
    ↓ build
  Image
    ↓ run
Container

Image는 Container를 만들기 위한 Template이고,
Container는 Image를 실제로 실행한 Instance입니다.

Programming에 비유하면 완전히 같지는 않지만:

Class     → Image
Object    → Container

처럼 생각하면 처음 이해할 때 도움이 됩니다.

Image 확인

docker image ls

또는:

docker images

예:

REPOSITORY   TAG       IMAGE ID
nginx        latest    abc123...

Image 받기

Registry에서 Image를 내려받습니다.

docker pull nginx

특정 Tag:

docker pull nginx:1.27

구조:

nginx:1.27
  │     │
  │     └── tag
  └──────── repository

Image 삭제

docker image rm <image>

예:

docker image rm nginx:1.27

짧은 형태:

docker rmi nginx:1.27

5. Registry

Image를 다른 Machine에서도 사용할 수 있으려면 저장하고 배포할 장소가 필요합니다.

그 역할을 하는 것이 Container Registry입니다.

Developer
    │ push
    ▼
Registry
    │ pull
    ▼
Server

Docker Hub는 대표적인 Public Registry입니다.

docker pull nginx

를 실행하면 필요한 Image가 Local에 없을 경우 Registry에서 Image를 가져옵니다.

6. Dockerfile

Image를 매번 손으로 만들면 재현성이 떨어집니다.

예를 들어:

Debian 설치
NGINX 설치
Config 복사
Directory 생성
Permission 변경

을 사람이 기억해서 반복하는 대신, Image를 만드는 과정을 File로 기록할 수 있습니다.

그 File이 Dockerfile입니다.

Dockerfile
    │
    │ docker build
    ▼
  Image

기본 Dockerfile

FROM debian:bookworm

RUN apt-get update \
    && apt-get install -y nginx \
    && rm -rf /var/lib/apt/lists/*

COPY conf/nginx.conf /etc/nginx/nginx.conf

EXPOSE 443

CMD ["nginx", "-g", "daemon off;"]

한 줄씩 보겠습니다.

FROM

FROM debian:bookworm

Image의 출발점을 지정합니다.

Debian Base Image
       ↓
우리의 변경사항 추가
       ↓
NGINX Image

RUN

RUN apt-get update && apt-get install -y nginx

Image를 Build하는 동안 Command를 실행합니다.

여기서 중요한 구분:

RUN
→ build time

CMD / ENTRYPOINT
→ container run time

apt-get install -y의 -y는 설치 질문에 자동으로 yes를 선택합니다.

COPY

COPY conf/nginx.conf /etc/nginx/nginx.conf

Build Context의 File을 Image 안으로 복사합니다.

Host
conf/nginx.conf
      ↓
Image
/etc/nginx/nginx.conf

WORKDIR

WORKDIR /app

이후 명령의 기본 Working Directory를 지정합니다.

Shell에서:

cd /app

한 것과 비슷한 역할을 Image Build 단계에 설정한다고 생각하면 됩니다.

EXPOSE

EXPOSE 443

Container가 어떤 Port에서 Service를 제공하도록 설계되었는지를 표현합니다.

중요: EXPOSE만 쓴다고 Host에 Port가 자동으로 공개되는 것은 아닙니다.

실제 Host Port Publishing은 docker run -p 또는 Compose의 ports에서 설정합니다.

CMD

CMD ["nginx", "-g", "daemon off;"]

Container가 시작될 때 기본으로 실행할 Command입니다.

왜 daemon off;일까요?

Container는 보통 주 Process가 살아 있는 동안 실행 상태를 유지합니다.

NGINX가 Background daemon으로 빠지고 main process가 끝나면 Container가 종료될 수 있으므로 NGINX를 Foreground에서 실행합니다.

Container start
      ↓
NGINX main process
      ↓
Process alive
      ↓
Container alive

7. docker build

Dockerfile에서 Image를 만듭니다.

docker build -t my-nginx .

분해하면:

docker build
→ Image Build

-t my-nginx
→ Image에 이름/tag 지정

.
→ Build Context는 현재 Directory

확인:

docker image ls

Build Context란?

Docker Build가 접근할 수 있는 File 범위입니다.

docker build .

의 .은 현재 Directory를 Build Context로 전달한다는 뜻입니다.

Dockerfile에서:

COPY conf/nginx.conf /etc/nginx/nginx.conf

를 실행하려면 conf/nginx.conf가 Build Context 안에 있어야 합니다.

8. Container 실행

Image를 실제 Process로 실행합니다.

docker run my-nginx

흐름:

Image
  ↓ docker run
Container 생성
  ↓
CMD / ENTRYPOINT 실행
  ↓
Process 실행

Background 실행

docker run -d my-nginx

-d = detached mode.

Terminal을 점유하지 않고 Background에서 실행합니다.

이름 지정

docker run --name web my-nginx

Container ID 대신 web이라는 이름으로 조작할 수 있습니다.

Port Publishing

docker run -p 8080:80 nginx

의미:

Host Port 8080
      │
      ▼
Container Port 80

Browser에서 Host의 8080으로 접근하면 Container의 80으로 전달됩니다.

9. Container 조작

실행 중 Container:

docker ps

모든 Container:

docker ps -a

-a = all.

중지

docker stop web

시작

docker start web

삭제

docker rm web

실행 중인 Container는 먼저 중지해야 합니다.

강제 삭제:

docker rm -f web

10. Container 안으로 들어가기

실행 중인 Container 안에서 Command를 실행할 수 있습니다.

docker exec web ls

Interactive Shell:

docker exec -it web sh

Bash가 있다면:

docker exec -it web bash

옵션:

-i → stdin을 열어둠
-t → pseudo-terminal 할당

그래서 -it를 같이 사용하면 Container 내부에서 Terminal을 사용하는 느낌으로 작업할 수 있습니다.

11. Logs

Container가 이상하게 동작하면 가장 먼저 볼 것 중 하나입니다.

docker logs web

실시간:

docker logs -f web

최근 100줄:

docker logs --tail 100 web

Container Debugging의 기본 흐름:

docker ps
   ↓
docker logs
   ↓
docker exec

즉:

살아 있나?
↓
무슨 Error를 냈나?
↓
내부 상태는 어떤가?

입니다.

12. Container가 사라지면 Data도 사라질까?

Container에는 Writable Layer가 있지만 Container 자체를 삭제하면 그 Container에만 있던 변경사항도 함께 사라집니다.

Database를 생각해봅시다.

MariaDB Container
       │
       └── Database Data

Container를 재생성할 때 Database까지 사라지면 안 됩니다.

따라서 Application의 생명주기와 Data의 생명주기를 분리해야 합니다.

이 문제를 해결하는 것이 Volume입니다.

13. Volume

Volume은 Container의 Lifecycle과 독립적으로 Data를 저장하기 위한 Docker Storage 방식입니다.

Container
    │
    │ mount
    ▼
 Volume
    │
    ▼
Persistent Data

Container를 삭제하고 새 Container를 만들어도 같은 Volume을 다시 연결하면 Data를 사용할 수 있습니다.

Volume 생성

docker volume create db-data

목록:

docker volume ls

상세:

docker volume inspect db-data

Volume 연결

docker run \
  --name db \
  -v db-data:/var/lib/mysql \
  mariadb

의미:

db-data
Docker Volume
     │
     ▼
/var/lib/mysql
Container 내부 경로

MariaDB의 Data가 Container 자체가 아니라 Volume에 유지될 수 있습니다.

Volume 삭제

docker volume rm db-data

주의: Volume은 실제 Data를 담을 수 있으므로 Container 삭제보다 훨씬 조심해야 합니다.

14. Bind Mount와 Volume

Host의 특정 Directory를 Container에 직접 연결할 수도 있습니다.

docker run -v ./data:/data image

이것은 Host Path와 Container Path를 직접 연결하는 형태입니다.

Host ./data
     ↕
Container /data

개념적으로:

Bind Mount
→ Host의 특정 Path를 직접 연결

Named Volume
→ Docker가 관리하는 Storage를 연결

이라고 구분하면 좋습니다.

15. Container끼리 어떻게 통신할까?

Inception에서는 하나의 Container만 실행하지 않습니다.

NGINX Container
WordPress Container
MariaDB Container

이들은 서로 통신해야 합니다.

NGINX
  ↓
WordPress / PHP-FPM
  ↓
MariaDB

그래서 Container 간 Network가 필요합니다.

16. Docker Network

Docker Network는 Container들이 서로 Network 통신할 수 있도록 연결합니다.

Docker Network

NGINX ───── WordPress ───── MariaDB

Network 생성

docker network create inception

목록:

docker network ls

상세:

docker network inspect inception

Network에 연결해서 실행

docker run \
  --name db \
  --network inception \
  mariadb

다른 Container:

docker run \
  --name wordpress \
  --network inception \
  wordpress

같은 Docker Network에 있는 Container는 이름을 통해 서로를 찾을 수 있습니다.

예를 들어 WordPress에서 DB Host를:

mariadb

같은 Service/Container 이름으로 사용할 수 있습니다.

17. localhost를 특히 조심하자

Container에서 가장 많이 헷갈리는 개념입니다.

WordPress Container 안에서:

localhost

는 Host Computer도 MariaDB Container도 아닙니다.

현재 WordPress Container 자기 자신입니다.

WordPress Container
    │
    ├── localhost → WordPress Container 자신
    │
    └── mariadb   → Network를 통해 MariaDB Container

따라서 DB가 다른 Container에 있다면:

DB_HOST=localhost

가 아니라 보통:

DB_HOST=mariadb

처럼 해당 Service 이름을 사용해야 합니다.

18. Inception의 Architecture

이제 과제의 큰 구조를 볼 수 있습니다.

Client
  │
  │ HTTPS :443
  ▼
NGINX Container
  │
  │ FastCGI
  ▼
WordPress / PHP-FPM Container
  │
  │ SQL
  ▼
MariaDB Container

그리고 Data는 Container 밖의 Persistent Storage에 보관합니다.

WordPress Container ─── WordPress Volume
MariaDB Container   ─── Database Volume

즉 Inception은 세 개의 독립된 Service를 Network와 Volume으로 하나의 Application처럼 연결하는 프로젝트라고 볼 수 있습니다.

19. NGINX

왜 NGINX가 필요한가?

Browser는 HTTP/HTTPS로 요청합니다.

WordPress의 PHP Code를 실행하는 PHP-FPM은 Browser와 직접 HTTPS Web Server 역할을 하는 것이 아닙니다.

따라서 Client의 Web Request를 받아 처리할 Front Door가 필요합니다.

Browser
   │ HTTPS
   ▼
 NGINX
   │
   ├── TLS 처리
   ├── Request 처리
   └── PHP Request 전달

NGINX가 이 역할을 담당합니다.

Web Server란?

Web Server는 HTTP Request를 받아 HTTP Response를 반환하는 Server Software입니다.

Static File이라면 직접 반환할 수 있습니다.

Browser
   │ GET /image.png
   ▼
NGINX
   │
   ▼
image.png

PHP처럼 실행이 필요한 요청은 다른 Process로 전달할 수 있습니다.

20. HTTPS와 TLS

HTTP Traffic을 그대로 보내면 Network 중간에서 내용을 볼 수 있는 위험이 있습니다.

TLS는 통신을 암호화하고 Server Identity를 확인할 수 있는 기반을 제공합니다.

HTTP
Client ───────── Server
       Plaintext 가능

HTTPS
Client ═TLS═════ Server
       Encrypted

HTTPS는 간단히:

HTTP over TLS

라고 이해할 수 있습니다.

Inception에서는 NGINX가 TLS Connection을 받는 Frontend 역할을 합니다.

Certificate와 Private Key

TLS Server에는 Certificate와 Private Key가 사용됩니다.

Certificate
→ Server의 Public Identity 정보

Private Key
→ Server만 안전하게 보관해야 하는 비밀 Key

Self-signed Certificate를 실습용으로 만들 때 OpenSSL을 사용할 수 있습니다.

예:

openssl req -x509 -nodes \
  -newkey rsa:2048 \
  -keyout server.key \
  -out server.crt \
  -days 365

이 명령은 학습용 예시입니다. 실제 운영 환경의 Certificate 관리 방식과는 다를 수 있습니다.

21. NGINX Configuration

기본적인 구조 예:

server {
    listen 443 ssl;

    server_name example.local;

    ssl_certificate     /etc/nginx/ssl/server.crt;
    ssl_certificate_key /etc/nginx/ssl/server.key;

    root /var/www/html;
    index index.php index.html;

    location ~ \.php$ {
        fastcgi_pass wordpress:9000;
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
    }
}

핵심만 보면:

listen 443 ssl
→ 443 Port에서 TLS Connection 수신

server_name
→ 어떤 Hostname 요청을 처리할지

ssl_certificate
→ Certificate

ssl_certificate_key
→ Private Key

root
→ Web Document Root

fastcgi_pass wordpress:9000
→ PHP 요청을 WordPress/PHP-FPM Service로 전달

22. WordPress와 PHP-FPM

WordPress는 PHP로 작성된 Web Application입니다.

Browser가 PHP Source Code를 받아서 실행하는 것이 아닙니다.

Browser
   │ HTTP Request
   ▼
NGINX
   │
   │ FastCGI
   ▼
PHP-FPM
   │
   │ PHP 실행
   ▼
WordPress

PHP-FPM

PHP-FPM은 PHP Script를 실행할 수 있는 Process Manager입니다.

NGINX는 PHP File을 직접 실행하지 않고 FastCGI Protocol을 통해 PHP-FPM에게 요청을 전달합니다.

NGINX
   │
   │ FastCGI
   ▼
PHP-FPM
   │
   ▼
PHP Code 실행
   │
   ▼
Result

Inception에서 흔히:

wordpress:9000

으로 연결하는 이유가 이것입니다.

wordpress는 Docker Network의 Service 이름이고 9000은 PHP-FPM이 Listen하는 Port입니다.

23. FastCGI

HTTP와 FastCGI를 구분해야 합니다.

Browser ──HTTP/HTTPS──> NGINX

NGINX ───FastCGI──────> PHP-FPM

Browser는 PHP-FPM과 직접 이야기하지 않습니다.

NGINX가 Client의 HTTP Request를 받아 필요한 정보를 FastCGI 형태로 PHP-FPM에 전달합니다.

24. MariaDB

WordPress에는 영구적으로 저장해야 할 Data가 있습니다.

예:

Users
Posts
Comments
Settings
Metadata

이런 구조화된 Data를 관리하기 위해 Database를 사용합니다.

Inception에서는 MariaDB가 Database Server 역할을 합니다.

WordPress
    │
    │ SQL / DB Connection
    ▼
MariaDB
    │
    ▼
Database Files

왜 MariaDB를 별도 Container로 둘까?

관심사를 분리할 수 있습니다.

NGINX      → HTTP / TLS
WordPress  → Application / PHP
MariaDB    → Database

각 Service를 독립적으로 Build, Restart, Debugging할 수 있습니다.

25. Environment Variable

Container마다 달라질 수 있는 설정을 Image 안에 Hard Coding하면 재사용하기 어렵습니다.

예를 들어:

Database Name
Database User
Database Password
Domain

등은 실행 환경에 따라 달라질 수 있습니다.

이런 값을 Environment Variable로 전달할 수 있습니다.

Shell에서:

export DB_NAME=wordpress

확인:

echo "$DB_NAME"

Container:

docker run -e DB_NAME=wordpress image

26. .env

Compose에서는 반복되는 값을 .env에 둘 수 있습니다.

예:

DOMAIN_NAME=example.local
MYSQL_DATABASE=wordpress
MYSQL_USER=wpuser

Compose:

environment:
  MYSQL_DATABASE: ${MYSQL_DATABASE}
  MYSQL_USER: ${MYSQL_USER}

주의할 점은 Environment Variable과 Secret은 완전히 같은 개념이 아니라는 것입니다.

Password 같은 민감한 값을 Git Repository에 그대로 Commit하면 안 됩니다.

27. Docker Compose가 왜 필요한가?

지금까지 세 Service를 Docker CLI만으로 실행한다고 생각해봅시다.

docker network create inception

docker volume create db-data
docker volume create wp-data

docker run ... mariadb
docker run ... wordpress
docker run ... nginx

각 Container마다:

Image
Network
Volume
Environment Variable
Port
Restart Policy
Dependency

를 긴 Command로 관리해야 합니다.

Service가 많아질수록 재현하기 어렵습니다.

그래서:

Multi-Container Application 전체 구성을 하나의 File로 선언할 수 없을까?

라는 문제가 생깁니다.

그 해결책이 Docker Compose입니다.

28. Compose File

Compose는 YAML File에 Application의 Service, Network, Volume 등을 선언합니다.

services:
  nginx:
    build: ./requirements/nginx
    ports:
      - "443:443"
    networks:
      - inception

  wordpress:
    build: ./requirements/wordpress
    networks:
      - inception

  mariadb:
    build: ./requirements/mariadb
    networks:
      - inception

networks:
  inception:

volumes:
  wordpress-data:
  mariadb-data:

큰 그림:

compose.yaml
│
├── services
│   ├── nginx
│   ├── wordpress
│   └── mariadb
│
├── networks
│   └── inception
│
└── volumes
    ├── wordpress-data
    └── mariadb-data

29. services

services:
  nginx:
    ...

Service는 Application을 구성하는 실행 단위입니다.

일반적으로 하나의 Service 정의를 기반으로 Container가 실행됩니다.

30. build

nginx:
  build: ./requirements/nginx

해당 Directory를 Build Context로 사용하여 Image를 Build합니다.

그 Directory 안의 Dockerfile을 사용합니다.

compose.yaml
    ↓
build path
    ↓
Dockerfile
    ↓
Image
    ↓
Container

31. ports

ports:
  - "443:443"

Host : Container
443  : 443

외부 Client가 Host의 443 Port로 접근하면 NGINX Container의 443 Port로 전달됩니다.

32. volumes

예:

volumes:
  - mariadb-data:/var/lib/mysql

의미:

mariadb-data Volume
        ↓ mount
/var/lib/mysql
Container

Container를 다시 만들어도 Database Data를 유지할 수 있습니다.

33. networks

networks:
  - inception

해당 Service를 inception Network에 연결합니다.

같은 Network에 연결된 Service들은 Service 이름으로 서로 통신할 수 있습니다.

nginx
  │
  │ wordpress:9000
  ▼
wordpress
  │
  │ mariadb:3306
  ▼
mariadb

여기서 Container IP를 직접 Hard Coding하지 않는 것이 중요합니다.

34. depends_on

depends_on:
  - wordpress

Service 시작 순서를 표현할 때 사용할 수 있습니다.

하지만 중요한 점:

depends_on으로 Container가 먼저 시작되었다고 해서 Application이 실제 요청을 받을 준비까지 끝났다는 뜻은 아닙니다.

Database가 완전히 Ready가 되는 시점 같은 것은 별도의 Health Check나 Application-level retry가 필요할 수 있습니다.

35. restart policy

Server Application은 예상치 못하게 종료될 수 있습니다.

Compose에서 Restart Policy를 설정할 수 있습니다.

restart: unless-stopped

이런 설정은 Container Lifecycle을 관리하는 데 사용됩니다.

정확한 정책 선택은 과제 요구사항과 운영 목적에 맞게 결정해야 합니다.

36. Docker Compose 명령어

Build

docker compose build

Compose에 정의된 Service Image를 Build합니다.

Cache 없이:

docker compose build --no-cache

실행

docker compose up

Foreground에서 실행.

Background:

docker compose up -d

Build 후 실행:

docker compose up -d --build

상태

docker compose ps

Logs

전체:

docker compose logs

특정 Service:

docker compose logs nginx

실시간:

docker compose logs -f

Container 내부 실행

docker compose exec nginx sh

MariaDB:

docker compose exec mariadb sh

종료

docker compose down

Container와 Compose Network 등을 내립니다.

Volume까지 삭제:

docker compose down -v

주의: -v는 Persistent Data를 담은 Volume까지 제거할 수 있습니다.

37. Dockerfile과 Compose의 차이

매우 자주 헷갈리는 부분입니다.

Dockerfile
→ Image를 어떻게 만들 것인가?

compose.yaml
→ 만들어진 Image들을 어떻게 실행하고 연결할 것인가?

조금 더 풀면:

Dockerfile
FROM
RUN
COPY
CMD
     ↓
   Image


compose.yaml
services
networks
volumes
ports
environment
     ↓
Running Application

Inception에서는 둘 다 필요합니다.

38. Inception 전체 Request Flow

Browser에서 Website에 접속하면:

Browser
   │
   │ HTTPS Request
   │ Port 443
   ▼
Host
   │
   │ Docker Port Publishing
   ▼
NGINX Container
   │
   │ TLS 종료
   │ Request 해석
   │
   ├── Static File → 직접 Response
   │
   └── PHP
        │
        │ FastCGI
        ▼
WordPress / PHP-FPM
        │
        │ SQL Connection
        ▼
MariaDB
        │
        ▼
Database Volume

Response는 반대 방향으로 돌아옵니다.

MariaDB
   ↓
WordPress
   ↓
NGINX
   ↓
Browser

39. Data Flow와 Storage

Web Request와 Data Persistence는 별개의 흐름입니다.

WordPress Files
     │
     ▼
WordPress Volume

MariaDB Data
     │
     ▼
MariaDB Volume

그래서 Container를 다시 만들어도 필요한 Data는 유지할 수 있습니다.

40. 기본 구현 순서

Inception을 처음부터 만든다면 한꺼번에 모든 것을 작성하지 않는 것이 좋습니다.

1. MariaDB
   ↓
Container가 실행되는지
DB가 생성되는지
Volume에 Data가 남는지

2. WordPress / PHP-FPM
   ↓
PHP-FPM이 실행되는지
MariaDB에 연결되는지

3. NGINX
   ↓
443에서 Listen하는지
TLS가 동작하는지
PHP 요청이 WordPress로 가는지

4. Compose
   ↓
세 Service를 한 번에 Build / Run

5. Persistence
   ↓
Container 재생성 후에도 Data 유지

6. Full Request
   ↓
Browser → NGINX → WordPress → MariaDB

이렇게 하면 문제가 생겼을 때 범위를 좁히기 쉽습니다.

41. 기본 Debugging 방법

먼저 전체 상태

docker compose ps

질문:

Container가 모두 Up인가?
Restarting 상태인가?
Exited인가?

Logs

docker compose logs

특정 Service:

docker compose logs mariadb
docker compose logs wordpress
docker compose logs nginx

Container 내부

docker compose exec nginx sh

내부에서:

ps
ls
cat /etc/nginx/nginx.conf

등으로 확인할 수 있습니다.

42. NGINX가 안 된다

Container 실행 중?
      ↓
NGINX Process 실행 중?
      ↓
Config Syntax 정상?
      ↓
443 Listen 중?
      ↓
Certificate 존재?
      ↓
Host Port Publishing 정상?

예:

docker compose ps
docker compose logs nginx
docker compose exec nginx nginx -t
docker compose exec nginx ss -tlnp

nginx -t는 NGINX Configuration Syntax를 검사합니다.

43. WordPress가 안 된다

PHP-FPM 실행 중?
      ↓
9000 Listen?
      ↓
NGINX와 같은 Network?
      ↓
MariaDB Hostname 맞음?
      ↓
DB Credential 맞음?

확인:

docker compose logs wordpress
docker compose exec wordpress ps

44. MariaDB가 안 된다

Container 실행?
     ↓
MariaDB Process 실행?
     ↓
Database 초기화?
     ↓
User 생성?
     ↓
Volume Permission?
     ↓
WordPress Credential과 일치?

확인:

docker compose logs mariadb
docker compose exec mariadb sh

DB Client가 설치되어 있다면 내부에서 직접 접속하여 확인할 수도 있습니다.

45. Network 문제

Network 확인:

docker network ls

상세:

docker network inspect <network>

Compose가 만든 Network 이름은 Project Prefix가 붙을 수 있으므로 실제 이름은 docker network ls로 확인합니다.

Container 내부에서 Name Resolution을 확인할 수 있는 도구가 있다면:

getent hosts mariadb

같이 확인할 수 있습니다.

핵심 질문:

같은 Network에 있는가?
Service 이름이 맞는가?
올바른 Port를 사용하고 있는가?
localhost를 잘못 사용하고 있지 않은가?

46. Volume 문제

목록:

docker volume ls

상세:

docker volume inspect <volume>

Container Mount 확인:

docker inspect <container>

문제가 생기면:

Volume이 실제로 연결됐나?
Mount Path가 맞나?
Process가 해당 Path에 쓸 Permission이 있나?
Container를 지우면서 Volume도 지우지 않았나?

를 확인합니다.

47. curl로 Web Server 확인

Browser만 사용하면 Error 원인을 보기 어려울 때가 있습니다.

Command Line에서는 curl이 매우 유용합니다.

curl https://example.local

자세히:

curl -v https://example.local

-v = verbose.

TLS Handshake, Request Header, Response Header 등을 더 자세히 볼 수 있습니다.

Self-signed Certificate를 사용하는 학습 환경에서 Certificate 검증을 건너뛰어야 한다면:

curl -k https://example.local

-k는 TLS Certificate 검증을 생략하므로 Debugging/학습용으로만 이해해야 합니다.

48. Docker inspect

Docker Object의 상세 Configuration을 확인할 때 사용합니다.

docker inspect <container>

여기서:

Network
IP
Mount
Environment
Command
State

등을 확인할 수 있습니다.

처음에는 Output이 매우 길기 때문에 docker inspect는:

"Docker가 이 Container를 실제로 어떤 설정으로 실행했지?"

를 확인하는 도구라고 기억하면 됩니다.

49. 자주 헷갈리는 개념

Image vs Container

Image
→ 실행 환경 Template

Container
→ Image를 실행한 Instance

Dockerfile vs Compose

Dockerfile
→ Image Build 방법

Compose
→ Multi-Container Application 실행 구성

EXPOSE vs ports

EXPOSE
→ Image가 사용하는 Port에 대한 정보/의도

ports
→ Host와 Container 사이 실제 Port Publishing

Volume vs Container Filesystem

Container Writable Layer
→ Container Lifecycle에 종속

Volume
→ Container와 독립적인 Persistent Storage

localhost

Container 안 localhost
→ 그 Container 자신

다른 Container를 의미하지 않습니다.

HTTP vs FastCGI

Browser → NGINX
HTTP / HTTPS

NGINX → PHP-FPM
FastCGI

50. 명령어 Cheat Sheet

목적

명령어

Docker 상태

docker info

Image 목록

docker image ls

Image Build

docker build -t name .

Image Pull

docker pull image

Container 실행

docker run image

실행 Container

docker ps

모든 Container

docker ps -a

Container 중지

docker stop name

Container 삭제

docker rm name

내부 Shell

docker exec -it name sh

Logs

docker logs name

Network 목록

docker network ls

Network 상세

docker network inspect name

Volume 목록

docker volume ls

Volume 상세

docker volume inspect name

Object 상세

docker inspect name

Compose Build

docker compose build

Compose 실행

docker compose up -d

Build + 실행

docker compose up -d --build

Compose 상태

docker compose ps

Compose Logs

docker compose logs -f

Service Shell

docker compose exec service sh

Compose 종료

docker compose down

NGINX Config 검사

nginx -t

Listening Port

ss -tlnp

HTTP 테스트

curl -v URL

51. 명령어보다 질문을 기억하자

"Image가 만들어졌나?"
→ docker image ls

"Container가 살아 있나?"
→ docker ps

"왜 죽었지?"
→ docker logs

"안에서는 무슨 일이 일어나지?"
→ docker exec

"어떤 설정으로 실행됐지?"
→ docker inspect

"Container끼리 연결됐나?"
→ docker network inspect

"Data는 어디에 저장되지?"
→ docker volume inspect

"Compose 전체 상태는?"
→ docker compose ps

"NGINX 설정이 틀렸나?"
→ nginx -t

"실제로 어느 Port를 듣고 있지?"
→ ss -tlnp

"HTTP/TLS 요청은 어디까지 가나?"
→ curl -v

이 관계를 이해하면 Command를 전부 암기하지 않아도 문제를 추적할 수 있습니다.

52. Inception의 전체 그림

Born2beroot
Linux Server를 관리할 수 있게 됨
        ↓
여러 Application을 설치하고 싶다
        ↓
Dependency / Configuration 충돌
        ↓
Application 환경을 격리하고 싶다
        ↓
Container
        ↓
Container를 쉽게 관리하고 싶다
        ↓
Docker
        ↓
실행 환경을 재현하고 싶다
        ↓
Dockerfile → Image
        ↓
Image를 실제로 실행한다
        ↓
Container
        ↓
Container끼리 통신해야 한다
        ↓
Docker Network
        ↓
Container가 없어져도 Data는 남아야 한다
        ↓
Volume
        ↓
Web Request를 받는다
        ↓
NGINX / TLS
        ↓
PHP Application을 실행한다
        ↓
WordPress / PHP-FPM
        ↓
Data를 저장한다
        ↓
MariaDB
        ↓
여러 Container를 하나의 System으로 정의한다
        ↓
Docker Compose

결국 Inception의 핵심 질문은:

하나의 Linux Server에서 여러 Service를 서로 격리하면서도 하나의 Application처럼 연결하고, 그 환경을 재현 가능하게 만들려면 어떻게 해야 하는가?

입니다.

53. 다음 단계: Inception of Things

Inception을 끝내면 여러 Container를 하나의 Server에서 운영할 수 있게 됩니다.

One Server
   │
Docker Compose
   │
├── NGINX Container
├── WordPress Container
└── MariaDB Container

그런데 시스템이 더 커지면 새로운 문제가 생깁니다.

Container가 수십 개라면?
Server가 여러 대라면?
Container가 죽으면 누가 다시 띄우지?
Replica를 여러 개 유지하려면?
어느 Server에서 Container를 실행하지?
배포 상태를 자동으로 유지하려면?

Docker Compose만으로는 관심 범위가 충분하지 않은 문제가 등장합니다.

그래서 다음 질문으로 이어집니다.

많은 Container를 여러 Machine에 걸쳐 자동으로 배치하고 원하는 상태로 유지할 수 없을까?

Inception
Containerization
      ↓
많은 Container의 운영 문제
      ↓
Inception of Things
      ↓
Kubernetes
      ↓
Container Orchestration

Born2beroot에서 Server,
Inception에서 Container,
Inception of Things에서 Orchestration으로 관심 범위가 확장됩니다.