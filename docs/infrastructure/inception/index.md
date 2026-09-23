# Inception

> Linux Server 위에서 Application을 격리하고, 연결하고, 지속적으로 운영하는 방법을 배운다.

---

# 0. 이 문서는 무엇을 배우기 위한 문서인가?

Born2beroot에서 우리는 **Linux Server 한 대를 관리하는 방법**을 배웠습니다.

예를 들어 다음과 같은 개념들을 만났습니다.

```text
Server
  ↓
Operating System
  ↓
Linux
  ↓
Process
  ↓
User / Permission
  ↓
Service
  ↓
Network
  ↓
Port
  ↓
Storage
```

이제 Server 자체는 어느 정도 준비할 수 있습니다.

그렇다면 다음 질문은 자연스럽습니다.

> 이 Server 위에서 실제 Application은 어떻게 운영할까?

예를 들어 WordPress Website를 운영한다고 생각해봅시다.

WordPress 하나를 운영하기 위해서도 여러 Software가 필요합니다.

```text
Browser
   ↓
Web Server
   ↓
WordPress / PHP
   ↓
Database
```

Inception에서는 이것을 대략 다음과 같은 구조로 만듭니다.

```text
Internet
   │
   │ HTTPS
   ▼
┌───────────────┐
│     NGINX     │
└───────┬───────┘
        │
        │ FastCGI
        ▼
┌───────────────┐
│   WordPress   │
│   + PHP-FPM   │
└───────┬───────┘
        │
        │ Database Connection
        ▼
┌───────────────┐
│    MariaDB    │
└───────────────┘
```

그런데 Inception의 진짜 핵심은 단순히 WordPress Website를 만드는 것이 아닙니다.

핵심 질문은 이것입니다.

> **여러 Application을 하나의 Server에서 서로 격리하면서도 함께 동작하게 만들려면 어떻게 해야 할까?**

이 질문에서 Container가 등장합니다.

그리고 Container를 관리하기 위해 Docker를 사용합니다.

이 문서에서는 다음 흐름으로 개념을 연결합니다.

```text
Born2beroot
Linux Server
     ↓
Application을 설치한다
     ↓
Application 환경이 뒤섞인다
     ↓
환경을 격리하고 싶다
     ↓
Process
     ↓
Container
     ↓
Namespace / cgroup
     ↓
Docker
     ↓
Image
     ↓
Dockerfile
     ↓
Container Filesystem
     ↓
Volume
     ↓
Container Network
     ↓
NGINX
     ↓
HTTP / HTTPS / TLS
     ↓
PHP-FPM / FastCGI
     ↓
WordPress
     ↓
MariaDB
     ↓
Docker Compose
```

---

# 1. Server에 Application을 직접 설치하면 안 될까?

물론 가능합니다.

Linux Server가 하나 있다고 생각해봅시다.

그 안에 필요한 Software를 직접 설치할 수 있습니다.

```text
Linux Server

├── NGINX
├── PHP
├── WordPress
└── MariaDB
```

예를 들어 Debian이라면 Package Manager를 이용해서 Software를 설치할 수도 있습니다.

```bash
apt install nginx
```

Born2beroot에서 배운 방식과 크게 다르지 않습니다.

그런데 Server에서 운영하는 Application이 많아지기 시작하면 문제가 생길 수 있습니다.

---

## 1.1 Dependency

Application은 혼자 존재하지 않는 경우가 많습니다.

다른 Software나 Library에 의존합니다.

이것을 **Dependency**라고 합니다.

예를 들어:

```text
Application A
└── Library X version 1 필요

Application B
└── Library X version 2 필요
```

두 Application이 서로 다른 Version을 요구할 수 있습니다.

그러면 하나의 Operating System 안에서 환경을 관리하기가 복잡해집니다.

---

## 1.2 Configuration

Application마다 설정도 필요합니다.

```text
NGINX
→ nginx.conf

PHP
→ php.ini

MariaDB
→ Database 설정

Application
→ Environment Variable
```

모든 Application을 Host OS에 직접 설치하면 이런 설정들이 하나의 System 안에 쌓이게 됩니다.

---

## 1.3 재현성

더 큰 문제는 다른 Server에서 같은 환경을 다시 만드는 것입니다.

Server A에서 Application이 잘 동작한다고 해봅시다.

그런데 Server B에서도 똑같이 실행하려면:

```text
같은 OS Version
같은 Package
같은 Library Version
같은 Configuration
같은 Directory
같은 Permission
```

등을 다시 맞춰야 합니다.

사람이 직접 설정하면 작은 차이가 생길 수 있습니다.

```text
Server A
→ 잘 동작함

Server B
→ "왜 여기서는 안 되지?"
```

그래서 이런 생각을 할 수 있습니다.

> Application뿐 아니라 **Application이 실행되는 환경 자체를 하나의 단위로 만들 수 없을까?**

Container는 이 문제를 해결하기 위한 방법 중 하나입니다.

---

# 2. Container를 이해하기 전에: Process

Container를 제대로 이해하려면 먼저 **Process**를 알아야 합니다.

Container는 결국 Linux Process와 매우 밀접한 기술이기 때문입니다.

---

# 3. Program과 Process

Program과 Process는 같은 것이 아닙니다.

Program은 Disk에 저장되어 있는 실행 가능한 Code입니다.

예를 들어:

```text
/usr/sbin/nginx
/usr/bin/bash
/usr/bin/python3
```

이것들은 File입니다.

아직 실행되고 있지 않을 수도 있습니다.

Program을 실행하면 Operating System은 그 Program을 실행하기 위한 상태를 Memory에 만들고 CPU 시간을 할당합니다.

이렇게 **현재 실행 중인 Program의 Instance**를 Process라고 합니다.

```text
Program
Disk에 저장된 Code
      │
      │ 실행
      ▼
Process
Memory에서 실행 중인 상태
```

같은 Program을 여러 번 실행하면 여러 Process가 존재할 수도 있습니다.

```text
bash Program

├── bash Process
├── bash Process
└── bash Process
```

---

# 4. PID란?

Operating System은 여러 Process를 구분해야 합니다.

그래서 각각의 Process에 번호를 부여합니다.

이 번호가:

**PID — Process ID**

입니다.

현재 Process를 확인하는 대표적인 명령어는:

```bash
ps
```

조금 더 많은 Process를 보고 싶다면:

```bash
ps aux
```

예를 들어:

```text
PID    COMMAND

1      systemd
742    sshd
810    dockerd
1250   nginx
```

처럼 볼 수 있습니다.

PID를 통해 Operating System은 어떤 Process를 종료하거나 관찰할지 구분할 수 있습니다.

예를 들어:

```bash
kill 1250
```

은 PID 1250 Process에 Signal을 보냅니다.

---

# 5. PID 1은 왜 특별할까?

Linux가 시작되면 Kernel이 준비된 뒤 User Space의 첫 Process가 시작됩니다.

일반적인 Linux Distribution에서는 이것이 `systemd`인 경우가 많습니다.

```text
Computer Boot
     ↓
Linux Kernel
     ↓
PID 1
systemd
     ↓
여러 Service / Process
```

그래서 일반적인 Linux Host에서:

```bash
ps -p 1
```

을 실행하면 PID 1 Process를 확인할 수 있습니다.

PID 1은 단순히 첫 번째 번호라는 것 이상의 의미가 있습니다.

System의 다른 Process들을 시작하고 관리하는 init 역할을 담당할 수 있고,
고아가 된 Process를 처리하는 역할도 가집니다.

그리고 Signal 처리 측면에서도 일반 Process와 조금 다른 특성이 있습니다.

이 개념은 조금 뒤 Container에서 다시 등장합니다.

---

# 6. Application마다 Process를 분리하면 충분하지 않을까?

Linux에서는 원래 여러 Process가 동시에 실행됩니다.

```text
Linux

├── nginx process
├── mariadb process
├── sshd process
└── bash process
```

그렇다면 그냥 Process만 따로 실행하면 Application이 분리되는 것 아닐까요?

문제는 Process들이 기본적으로 같은 System을 바라본다는 것입니다.

예를 들어 같은 Host에서 실행되는 Process들은 상황에 따라:

```text
같은 Process 목록
같은 Network 환경
같은 Filesystem
같은 Hostname
같은 System Resource
```

등을 공유하거나 볼 수 있습니다.

우리가 원하는 것은 단순히 Process를 여러 개 실행하는 것이 아닙니다.

```text
Application A
→ 자기 Process만 보이는 것처럼

Application B
→ 자기 Network가 있는 것처럼

Application C
→ 자기 Filesystem이 있는 것처럼
```

**Process마다 독립된 환경을 가진 것처럼 보이게 하는 것**입니다.

여기서 Linux의 **Namespace**가 등장합니다.

---

# 7. Namespace

Namespace는 Linux Kernel이 제공하는 격리 기능입니다.

쉽게 생각하면:

> **Process가 System의 어떤 부분을 볼 수 있는지를 분리하는 기능**

입니다.

실제 Computer는 하나지만 Process마다 서로 다른 System을 보고 있는 것처럼 만들 수 있습니다.

```text
                Linux Kernel
                     │
          ┌──────────┴──────────┐
          │                     │
     Namespace A           Namespace B
          │                     │
     Process A             Process B
```

Process A와 Process B는 같은 Kernel 위에서 실행되고 있지만 서로 다른 환경에 있다고 느낄 수 있습니다.

---

# 8. PID Namespace

Namespace에는 여러 종류가 있습니다.

그중 하나가 **PID Namespace**입니다.

PID Namespace는 Process가 볼 수 있는 Process ID 공간을 분리합니다.

Host에서는 다음처럼 보이는 Process가 있다고 생각해봅시다.

```text
Host

PID 1      systemd
PID 800    dockerd
PID 1427   nginx
```

그런데 nginx Process를 별도의 PID Namespace 안에서 실행하면 그 Namespace 내부에서는:

```text
Container

PID 1      nginx
```

처럼 보일 수 있습니다.

즉 **같은 Process가 어디에서 관찰되느냐에 따라 다른 PID로 보일 수 있습니다.**

```text
Host에서 바라봄
nginx → PID 1427

Container에서 바라봄
nginx → PID 1
```

Host의 PID 1이 nginx로 바뀐 것이 아닙니다.

Container가 자기만의 PID Namespace를 가지고 있기 때문에 그 안에서 번호 체계가 다시 보이는 것입니다.

---

# 9. Container의 PID 1

이제 Docker Container에서 자주 보게 되는 현상을 이해할 수 있습니다.

예를 들어 NGINX Container를 실행하면 Container 안에서는 NGINX Main Process가 PID 1로 보일 수 있습니다.

```text
Container

PID 1
nginx
```

Container Runtime은 이 Main Process의 생명주기와 Container의 생명주기를 밀접하게 연결합니다.

```text
Main Process 시작
      ↓
Container Running

Main Process 종료
      ↓
Container Stopped
```

그래서 중요한 문장이 하나 나옵니다.

> **Container가 실행된다는 것은 결국 격리된 환경 안에서 Process가 실행된다는 뜻이다.**

Container를 작은 VM이라고만 생각하면 이 부분을 놓치기 쉽습니다.

---

# 10. Foreground와 Background

Linux Program은 Foreground 또는 Background에서 실행될 수 있습니다.

Foreground Process는 현재 Terminal과 직접 연결되어 실행됩니다.

```text
Terminal
   │
   ▼
Process
```

반대로 Server Program은 전통적으로 Background에서 실행되는 경우가 많습니다.

이런 Background Service Process를 **daemon**이라고 부르기도 합니다.

```text
Terminal
   │
   │ Program 시작
   ▼
Server Process
   │
   └── Background에서 계속 실행
```

일반 Linux Server에서는 systemd 같은 Service Manager가 이런 Service를 관리합니다.

---

# 11. 왜 Docker의 NGINX에서 `daemon off;`를 사용할까?

Docker Container에서는 Main Process가 계속 실행되고 있어야 합니다.

그런데 NGINX가 기존 Server 방식처럼 스스로 Background daemon으로 이동하면 Container의 Main Foreground Process가 종료될 수 있습니다.

그래서 Container 환경에서는 NGINX를 Foreground에서 실행하는 방식을 사용합니다.

```bash
nginx -g "daemon off;"
```

Dockerfile에서는 다음처럼 볼 수 있습니다.

```dockerfile
CMD ["nginx", "-g", "daemon off;"]
```

흐름을 보면:

```text
Container Start
      ↓
nginx 실행
      ↓
Foreground 유지
      ↓
nginx Main Process가 살아 있음
      ↓
Container Running
```

NGINX가 종료되면:

```text
nginx 종료
    ↓
Main Process 종료
    ↓
Container 종료
```

즉 `daemon off;`는 단순히 Inception에서 외워야 하는 이상한 옵션이 아닙니다.

앞에서 배운:

```text
Process
  ↓
PID
  ↓
PID Namespace
  ↓
PID 1
  ↓
Container Lifecycle
```

이 모두 연결되어 만들어지는 설정입니다.

---

# 12. 다른 Namespace들

PID만 분리한다고 완전한 Container 환경이 되는 것은 아닙니다.

Linux에는 여러 Namespace가 있습니다.

대표적으로 다음과 같은 것들이 있습니다.

```text
PID Namespace
→ Process ID 공간 분리

Network Namespace
→ Network 환경 분리

Mount Namespace
→ Mount / Filesystem 관점 분리

UTS Namespace
→ Hostname 등의 System Identifier 분리

IPC Namespace
→ Process 간 통신 Resource 분리

User Namespace
→ User / UID 관점 분리
```

모든 세부 구현을 외울 필요는 없습니다.

핵심은:

> **Container는 Linux Kernel의 여러 격리 기능을 조합해 Process에게 독립된 System처럼 보이는 환경을 제공한다.**

입니다.

---

# 13. Network Namespace

Network Namespace가 분리되면 Process는 자기만의 Network 환경을 가진 것처럼 볼 수 있습니다.

예를 들어:

```text
Container A

eth0
IP A
Route A


Container B

eth0
IP B
Route B
```

실제로는 하나의 Host 위에 있지만 각 Container는 서로 다른 Network 환경을 가진 것처럼 동작합니다.

이 개념 때문에 나중에 아주 중요한 현상이 생깁니다.

> Container 안의 `localhost`는 Host가 아니다.

이것은 Network 부분에서 자세히 살펴봅니다.

---

# 14. Namespace만 있으면 충분할까?

Namespace는 **무엇을 볼 수 있는가**를 분리하는 데 강합니다.

하지만 다른 문제가 있습니다.

Container A가 CPU를 전부 사용한다면 어떻게 될까요?

```text
Container A
CPU 100%

Container B
CPU를 거의 사용하지 못함
```

또 Container 하나가 Memory를 무제한 사용하면 Host 전체에 영향을 줄 수 있습니다.

따라서:

> **Process가 얼마나 많은 Resource를 사용할 수 있는지도 관리해야 합니다.**

이 문제를 해결하는 Linux 기능이 **cgroup**입니다.

---

# 15. cgroup

cgroup은 **control group**의 줄임말입니다.

Process가 사용할 수 있는 System Resource를 관리하고 계측하는 Linux Kernel 기능입니다.

예를 들어:

```text
CPU
Memory
I/O
Process 수
```

등을 관리하는 데 사용될 수 있습니다.

개념적으로:

```text
Namespace
→ 무엇을 볼 수 있는가?

cgroup
→ 얼마나 사용할 수 있는가?
```

라고 기억하면 이해하기 쉽습니다.

Container 기술은 이런 Linux 기능들을 함께 이용합니다.

```text
Linux Process
      +
Namespaces
      +
cgroups
      +
Filesystem Isolation
      +
Network
      ↓
Container Environment
```

---

# 16. 그래서 Container란 무엇인가?

이제 처음보다 조금 더 정확하게 정의할 수 있습니다.

Container는 별도의 작은 Computer가 아닙니다.

Container는:

> **Host Kernel을 공유하면서, Linux의 격리 기능을 이용해 독립된 환경처럼 실행되는 Process 또는 Process Group**

이라고 이해할 수 있습니다.

```text
Physical Computer
       ↓
Linux Kernel
       │
       ├── Container A
       │      └── Process
       │
       ├── Container B
       │      └── Process
       │
       └── Container C
              └── Process
```

---

# 17. VM과 Container의 차이

Born2beroot에서는 Virtual Machine을 사용했습니다.

VM과 Container는 둘 다 **격리된 환경**을 만드는 데 사용할 수 있지만 방법이 다릅니다.

## Virtual Machine

```text
Physical Hardware
       ↓
Hypervisor
       │
 ┌─────┴─────┐
 │           │
VM A        VM B
 │           │
Guest OS    Guest OS
 │           │
Kernel      Kernel
 │           │
App         App
```

각 VM은 자신의 Operating System과 Kernel을 가집니다.

---

## Container

```text
Physical Hardware
       ↓
Host Operating System
       ↓
Linux Kernel
       │
 ┌─────┼─────┐
 │     │     │
 C1    C2    C3
 │     │     │
App   App   App
```

Container들은 Host Kernel을 공유합니다.

---

## 핵심 차이

| Virtual Machine | Container |
|---|---|
| Virtual Hardware를 제공 | Process 실행 환경을 격리 |
| Guest OS 존재 | Host Kernel 공유 |
| 각 VM이 Kernel을 가짐 | 별도 Kernel 없음 |
| 일반적으로 더 무거움 | 일반적으로 더 가벼움 |
| Machine 단위 격리 | Process/Application 단위 격리 |

Container가 VM보다 무조건 좋다는 뜻은 아닙니다.

둘은 목적과 격리 방식이 다릅니다.

---

# 18. Container를 직접 만들어야 할까?

여기까지 보면 Container를 만들려면 많은 Linux 기능이 필요합니다.

```text
Namespace
cgroup
Filesystem
Network
Process
```

이 모든 것을 사람이 직접 설정하는 것은 상당히 복잡합니다.

예를 들어 우리는 단순히:

> "NGINX를 격리된 환경에서 실행하고 싶다."

라고 생각했을 뿐인데 실제로는:

```text
Filesystem 준비
Namespace 생성
Network 구성
Resource 관리
Process 실행
Lifecycle 관리
```

등을 처리해야 합니다.

그래서 이 작업을 편리하게 처리할 도구가 필요합니다.

여기서 **Docker**가 등장합니다.

---

# 19. Docker

Docker는 Container를 만들고 실행하고 관리하기 위한 Platform입니다.

Docker를 사용하면 복잡한 Linux Container 기능을 직접 하나하나 조작하지 않고 높은 수준의 명령으로 Container를 관리할 수 있습니다.

예를 들어:

```bash
docker run nginx
```

이라는 명령 하나로 NGINX Container를 실행할 수 있습니다.

내부적으로는 훨씬 많은 일이 필요하지만 사용자는 Docker를 통해 이를 쉽게 관리합니다.

---

# 20. Container와 Docker는 같은 것인가?

아닙니다.

이 구분은 중요합니다.

```text
Container
→ 격리된 Process 실행 방식 / 개념

Docker
→ Container를 만들고 관리하는 Platform
```

Container라는 개념 자체가 Docker와 동일한 것은 아닙니다.

Docker가 Container 기술을 매우 쉽게 사용할 수 있게 만든 것입니다.

---

# 21. Docker의 기본 구조

우리가 Terminal에서:

```bash
docker ps
```

라고 입력하면 `docker` Command가 직접 모든 Container를 관리하는 것은 아닙니다.

큰 구조는 다음처럼 생각할 수 있습니다.

```text
User
 │
 │ docker ps
 ▼
Docker CLI
 │
 │ API
 ▼
Docker daemon
 │
 ├── Image 관리
 ├── Container 관리
 ├── Network 관리
 └── Volume 관리
```

---

# 22. Docker CLI

CLI는 **Command Line Interface**입니다.

우리가 Terminal에서 사용하는:

```bash
docker
```

Command가 여기에 해당합니다.

예를 들어:

```bash
docker ps
```

```bash
docker run nginx
```

```bash
docker logs nginx
```

같은 명령을 입력합니다.

CLI는 사용자의 요청을 Docker daemon에 전달합니다.

---

# 23. Docker daemon

Docker daemon은 Background에서 실행되며 Docker Resource를 관리합니다.

Daemon Process의 이름은 일반적으로:

```text
dockerd
```

입니다.

Linux에서 Docker Service를 확인한다면:

```bash
systemctl status docker
```

를 사용할 수 있습니다.

여기서 Born2beroot에서 배웠던 개념이 다시 등장합니다.

```text
systemd
   ↓
docker.service
   ↓
dockerd
```

즉 Docker도 결국 Linux 위에서 실행되는 Software입니다.

---

# 24. Docker를 간단히 확인해보자

Docker가 설치되어 있다면:

```bash
docker version
```

으로 Client와 Server 정보를 확인할 수 있습니다.

좀 더 자세한 Docker 환경 정보는:

```bash
docker info
```

로 볼 수 있습니다.

지금은 모든 출력 내용을 이해할 필요는 없습니다.

목적은 단순합니다.

```text
docker CLI
    ↓
Docker daemon과 통신
    ↓
Docker 환경 정보 반환
```

이라는 구조를 확인하는 것입니다.

---

# 25. 첫 Container 실행

이제 실제 Container를 하나 실행해봅시다.

```bash
docker run -it debian:bookworm bash
```

하나씩 보면:

```text
docker
→ Docker CLI

run
→ 새로운 Container를 생성하고 실행

-it
→ 현재 Terminal에서 직접 조작할 수 있게 연결

debian:bookworm
→ 사용할 Image

bash
→ Container에서 실행할 Process
```

성공하면 Debian 환경 안에서 Shell을 사용할 수 있습니다.

---

# 26. Container 안을 관찰해보자

Container 안에서:

```bash
hostname
```

을 실행해봅니다.

그리고:

```bash
ps
```

도 실행해봅니다.

Host와 다른 환경처럼 보일 것입니다.

이것은 실제 Computer가 새로 생겼기 때문이 아닙니다.

앞에서 배운 Namespace 등의 격리 기능 때문에 **Process가 제한된 System View를 보고 있기 때문**입니다.

Container에서 나오려면:

```bash
exit
```

을 입력합니다.

Main Process였던 `bash`가 종료됩니다.

그러면 Container도 종료됩니다.

왜 그런지 이제 설명할 수 있습니다.

```text
bash
→ Container의 Main Process

exit
→ bash 종료

Main Process 종료
→ Container 종료
```

---

# 27. Container 상태 확인

실행 중인 Container:

```bash
docker ps
```

종료된 Container까지 모두 보고 싶다면:

```bash
docker ps -a
```

`-a`는 `all`이라고 생각하면 됩니다.

방금 `exit`한 Debian Container가 `Exited` 상태로 남아 있는 것을 볼 수 있습니다.

이것은 중요한 구분입니다.

```text
Image
Container
Running Container
Stopped Container
```

이 네 개는 같은 것이 아닙니다.

---

# 28. 그런데 `debian:bookworm`은 어디에서 왔을까?

우리는:

```bash
docker run -it debian:bookworm bash
```

라고 했습니다.

그런데 Container를 만들려면 어떤 Filesystem과 Program을 사용할지 알아야 합니다.

예를 들어 Debian Container라면 최소한 Debian User Space 환경이 필요합니다.

Docker는 이 실행 환경의 원본을 **Image**라는 형태로 관리합니다.

---

# 29. Image

Image는 Container를 만들기 위한 **Read-only Template**이라고 이해할 수 있습니다.

예를 들어 NGINX를 실행하려면:

```text
Filesystem
NGINX Program
필요한 Library
기본 Configuration
```

등이 필요합니다.

이 환경을 미리 Packaging한 것이 Image입니다.

```text
NGINX Image

├── Filesystem
├── nginx
├── Libraries
└── 기본 File들
```

그리고 Image에서 Container를 만듭니다.

```text
              ┌── Container A
              │
NGINX Image ──┼── Container B
              │
              └── Container C
```

같은 Image에서 여러 Container를 만들 수 있습니다.

---

# 30. Image와 Container

가장 중요한 구분 중 하나입니다.

```text
Image
→ 실행 환경의 원본

Container
→ Image를 기반으로 실제 실행된 Instance
```

흐름으로 보면:

```text
Image
  │
  │ docker run
  ▼
Container
  │
  ▼
Process 실행
```

Image 자체가 실행되는 것은 아닙니다.

Image를 기반으로 Container를 만들고 그 안에서 Process가 실행됩니다.

---

# 31. Image 확인

현재 Local Machine에 있는 Image를 확인합니다.

```bash
docker image ls
```

또는:

```bash
docker images
```

예:

```text
REPOSITORY   TAG        IMAGE ID
debian       bookworm   ...
nginx        latest     ...
```

여기서:

```text
debian
→ Repository / Image 이름

bookworm
→ Tag
```

라고 볼 수 있습니다.

---

# 32. Registry

Image를 모든 사람이 자기 Computer에서 처음부터 만들 필요는 없습니다.

Image를 저장하고 배포하는 Server가 있습니다.

이를 **Container Registry**라고 합니다.

```text
Developer
    │
    │ push
    ▼
Registry
    │
    │ pull
    ▼
Server
```

대표적인 Public Registry 중 하나가 Docker Hub입니다.

Image를 내려받을 때:

```bash
docker pull debian:bookworm
```

을 사용할 수 있습니다.

그리고:

```bash
docker run debian:bookworm
```

을 실행했는데 Local에 Image가 없다면 Docker가 필요한 Image를 Registry에서 가져올 수도 있습니다.

---

# 33. 그런데 Image는 하나의 거대한 File일까?

개념적으로 Image를 하나의 Template이라고 설명했지만 실제 구조를 이해하려면 **Layer**를 알아야 합니다.

Docker Image는 여러 Layer가 쌓인 형태로 구성됩니다.

예를 들어:

```text
Layer 4
NGINX Config

Layer 3
NGINX 설치

Layer 2
Package 변경

Layer 1
Debian Base
```

이것을 겹쳐서 하나의 Filesystem처럼 사용합니다.

```text
┌──────────────────┐
│ NGINX Config     │
├──────────────────┤
│ NGINX            │
├──────────────────┤
│ Packages         │
├──────────────────┤
│ Debian Base      │
└──────────────────┘

       Image
```

---

# 34. 왜 Layer를 사용할까?

여러 Image가 공통 부분을 재사용할 수 있기 때문입니다.

예를 들어:

```text
Image A

App A
NGINX
Debian


Image B

App B
NGINX
Debian
```

모든 내용을 매번 완전히 복사하는 대신 공통 Layer를 재사용할 수 있습니다.

```text
        App A Layer
             │
NGINX Layer ─┤
             │
Debian Layer ┤
             │
        App B Layer
```

이런 Layer 구조는 Image Build와 배포를 효율적으로 만드는 데 도움을 줍니다.

---

# 35. Image는 왜 Read-only일까?

Image는 Container를 만들기 위한 **원본**입니다.

Container 하나에서 File을 수정했다고 원본 Image까지 바뀌면 문제가 생깁니다.

예를 들어 같은 Image에서 두 Container를 만들었다고 생각해봅시다.

```text
Image
 │
 ├── Container A
 └── Container B
```

Container A의 변경 때문에 Image 자체가 바뀌어 Container B까지 영향을 받으면 독립적인 실행 환경을 만들기 어렵습니다.

그래서 Image Layer는 기본적으로 Read-only로 취급됩니다.

---

# 36. 그럼 Container 안에서는 File을 어떻게 수정할까?

Container는 실행될 때 Image 위에 **Writable Layer**를 추가합니다.

```text
┌────────────────────┐
│ Container Writable │ ← 쓰기 가능
├────────────────────┤
│ Image Layer        │
├────────────────────┤
│ Image Layer        │
├────────────────────┤
│ Base Layer         │
└────────────────────┘
```

Container에서 새로운 File을 만들거나 기존 File을 변경하면 그 변경사항은 Container의 Writable Layer에 기록됩니다.

그래서 같은 Image에서 만든 Container라도 서로 다른 변경사항을 가질 수 있습니다.

```text
            Image
              │
      ┌───────┴───────┐
      │               │
Container A       Container B
Writable A        Writable B
```

---

# 37. Container를 삭제하면 왜 Data가 사라질까?

이제 이유를 이해할 수 있습니다.

```text
Image
  +
Container Writable Layer
```

에서 Container를 삭제하면 Container에 속해 있던 Writable Layer도 함께 제거됩니다.

```text
Container 삭제
      ↓
Writable Layer 삭제
      ↓
그 안에만 있던 Data도 삭제
```

단순한 Web Server라면 다시 만들면 될 수도 있습니다.

하지만 Database라면 문제가 큽니다.

```text
MariaDB Container

Users
Posts
Passwords
Application Data
```

이 Data가 Container 삭제와 함께 사라져서는 안 됩니다.

그래서 새로운 문제가 생깁니다.

> **Application의 생명주기와 Data의 생명주기를 분리할 수 없을까?**

이 문제를 해결하기 위해 **Volume**을 사용합니다.

---

# 38. Volume

Volume은 Container와 독립적으로 Data를 보존하기 위한 Storage입니다.

기본적인 그림은:

```text
Container
    │
    │ Mount
    ▼
 Volume
    │
    ▼
Persistent Data
```

입니다.

Container가 Volume의 특정 위치를 자신의 Filesystem 안의 Directory처럼 사용합니다.

---

# 39. MariaDB와 Volume

MariaDB는 Database File을 Disk에 저장해야 합니다.

개념적으로:

```text
MariaDB Container
       │
       │ Database File 저장
       ▼
/var/lib/mysql
```

이라고 생각해봅시다.

이 Directory가 Container Writable Layer에만 있다면 Container를 삭제할 때 Data도 잃을 수 있습니다.

그래서 Volume을 연결합니다.

```text
MariaDB Container
       │
       │ /var/lib/mysql
       ▼
┌─────────────────┐
│ MariaDB Volume  │
└─────────────────┘
```

이제 Container를 새로 만들어도 같은 Volume을 다시 연결할 수 있습니다.

```text
Old MariaDB Container
          X
          │
          │ 삭제
          │
     MariaDB Volume
          │
          │ 연결
          ▼
New MariaDB Container
```

Application은 교체할 수 있지만 Data는 유지할 수 있습니다.

---

# 40. Volume을 간단히 조작해보자

Volume 생성:

```bash
docker volume create my-data
```

현재 Volume 확인:

```bash
docker volume ls
```

특정 Volume 정보:

```bash
docker volume inspect my-data
```

삭제:

```bash
docker volume rm my-data
```

Volume에는 실제 중요한 Data가 들어 있을 수 있으므로 삭제할 때는 Container보다 더 주의해야 합니다.

---

# 41. Bind Mount와 Volume

Host의 특정 Directory를 Container에 직접 연결하는 방법도 있습니다.

예:

```text
Host Directory
/home/user/data
       │
       │ Mount
       ▼
Container
/data
```

이것을 **Bind Mount**라고 합니다.

반면 Named Volume은 Docker가 관리하는 Storage입니다.

```text
Bind Mount
→ Host의 특정 Path를 직접 연결

Named Volume
→ Docker가 관리하는 Storage 영역 사용
```

둘 다 Container 밖에 Data를 둘 수 있지만 관리 방식이 다릅니다.

Inception에서는 과제의 Storage 요구사항을 이해하고 그에 맞는 방식으로 Persistent Data를 구성해야 합니다.

---

# 42. Image를 직접 만들고 싶다

지금까지는 이미 존재하는 Image를 사용했습니다.

예를 들어:

```bash
docker run debian:bookworm
```

하지만 Inception에서는 우리가 원하는 Software와 Configuration이 들어 있는 Image를 직접 만들어야 합니다.

예를 들어 NGINX Image가 필요하다면:

```text
Debian
  ↓
NGINX 설치
  ↓
TLS Certificate 준비
  ↓
NGINX Configuration
  ↓
우리의 NGINX Image
```

가 필요합니다.

이 과정을 사람이 매번 직접 수행하면 다시 재현성 문제가 생깁니다.

그래서:

> **Image를 만드는 방법 자체를 File로 기록하자.**

라는 아이디어가 등장합니다.

그 File이 **Dockerfile**입니다.

---

# 43. Dockerfile

Dockerfile은 **Docker Image를 어떻게 만들 것인지 설명하는 Build Recipe**입니다.

예를 들어:

```dockerfile
FROM debian:bookworm

RUN apt-get update \
    && apt-get install -y nginx

COPY conf/nginx.conf /etc/nginx/nginx.conf

CMD ["nginx", "-g", "daemon off;"]
```

라고 작성할 수 있습니다.

흐름은:

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

입니다.

---

# 44. FROM

```dockerfile
FROM debian:bookworm
```

어떤 Base Image에서 시작할지 지정합니다.

```text
Debian Image
     ↓
우리의 변경사항
     ↓
새 Image
```

처음부터 Filesystem 전체를 만드는 대신 기존 Base Image 위에 필요한 것을 추가할 수 있습니다.

---

# 45. RUN

```dockerfile
RUN apt-get update \
    && apt-get install -y nginx
```

Image를 **Build하는 동안** Command를 실행합니다.

여기서 중요한 구분이 있습니다.

```text
RUN
→ Image Build 시 실행

CMD
→ Container 시작 시 기본 실행
```

예를 들어:

```dockerfile
RUN apt-get install -y nginx
```

는 Image에 NGINX를 설치하기 위한 것입니다.

Container를 시작할 때마다 NGINX를 다시 설치하는 것이 아닙니다.

---

# 46. COPY

```dockerfile
COPY conf/nginx.conf /etc/nginx/nginx.conf
```

Build Context의 File을 Image 안으로 복사합니다.

```text
Project

conf/nginx.conf
      │
      │ COPY
      ▼
Image

/etc/nginx/nginx.conf
```

이렇게 우리가 만든 Configuration을 Image에 포함할 수 있습니다.

---

# 47. CMD

```dockerfile
CMD ["nginx", "-g", "daemon off;"]
```

Container가 시작될 때 기본으로 실행할 Command를 지정합니다.

앞에서 배운 내용을 연결해봅시다.

```text
docker run
    ↓
Container 생성
    ↓
CMD 실행
    ↓
nginx 실행
    ↓
Main Process 유지
    ↓
Container Running
```

그래서 Dockerfile의 `CMD`는 Container Lifecycle과 직접 연결됩니다.

---

# 48. Image Build

Dockerfile을 작성했다면 Image를 만들 수 있습니다.

```bash
docker build -t my-nginx .
```

하나씩 보면:

```text
docker build
→ Dockerfile을 이용해 Image Build

-t my-nginx
→ 만들어진 Image에 이름 지정

.
→ 현재 Directory를 Build Context로 사용
```

완성된 Image는:

```bash
docker image ls
```

로 확인할 수 있습니다.

---

# 49. Build Context

마지막의 `.`은 단순히 문법 장식이 아닙니다.

```bash
docker build .
```

Docker에게:

> 현재 Directory를 Build에 사용할 Context로 제공하겠다.

라는 의미입니다.

예를 들어:

```text
project/
├── Dockerfile
└── conf/
    └── nginx.conf
```

가 있고 Dockerfile에서:

```dockerfile
COPY conf/nginx.conf /etc/nginx/nginx.conf
```

를 사용한다면 Docker는 Build Context 안에서 `conf/nginx.conf`를 찾습니다.

---

# 50. 여기까지의 흐름

지금까지 배운 것을 연결해봅시다.

```text
Linux Server
     ↓
Application을 분리하고 싶다
     ↓
Process
     ↓
Namespace
     ↓
cgroup
     ↓
Container
     ↓
Container를 쉽게 관리하고 싶다
     ↓
Docker
     ↓
Container 실행 환경의 원본이 필요하다
     ↓
Image
     ↓
Image를 재현 가능하게 만들고 싶다
     ↓
Dockerfile
     ↓
Container Data를 영구 보존하고 싶다
     ↓
Volume
```

이제 각각의 Application을 Container로 분리할 수 있습니다.

```text
┌─────────────┐
│    NGINX    │
└─────────────┘

┌─────────────┐
│  WordPress  │
└─────────────┘

┌─────────────┐
│   MariaDB   │
└─────────────┘
```

그런데 새로운 문제가 생깁니다.

서로 완전히 분리해놓았는데...

> **NGINX, WordPress, MariaDB는 어떻게 서로 통신할까?**

다음은 **Container Network**입니다.

# 51. Container Network

앞에서 우리는 Application을 Container로 분리했습니다.

```text
┌─────────────┐
│    NGINX    │
└─────────────┘

┌─────────────┐
│  WordPress  │
└─────────────┘

┌─────────────┐
│   MariaDB   │
└─────────────┘
```

격리는 잘 되었습니다.

하지만 Application은 서로 통신해야 합니다.

```text
NGINX
  ↓
WordPress / PHP-FPM
  ↓
MariaDB
```

즉 새로운 문제가 생깁니다.

> **서로 분리된 Container는 어떻게 통신할까?**

이 문제를 해결하기 위해 Docker Network를 사용합니다.

---

# 52. Network를 이해하기 전에: IP와 Port

Network Communication을 이해하려면 먼저 IP와 Port를 구분해야 합니다.

IP Address는 Network에서 **어떤 Host인가**를 식별하는 데 사용됩니다.

Port는 그 Host 안에서 **어떤 Service인가**를 구분하는 데 사용됩니다.

```text
192.168.1.10:443
│             │
│             └── Port
└──────────────── IP Address
```

예를 들어 한 Server에서:

```text
22
→ SSH

80
→ HTTP

443
→ HTTPS

3306
→ MariaDB
```

처럼 여러 Service가 서로 다른 Port를 사용할 수 있습니다.

Born2beroot에서 Firewall과 SSH를 공부할 때 보았던 Port 개념이 여기서 다시 등장합니다.

---

# 53. Network Namespace

앞에서 Namespace는 Process가 보는 System 환경을 분리한다고 했습니다.

Network Namespace는 Process가 보는 Network 환경을 분리합니다.

예를 들어 Container마다:

```text
Network Interface
IP Address
Routing Table
Port
```

등이 독립적으로 존재하는 것처럼 보일 수 있습니다.

```text
Container A
eth0
172.18.0.2

Container B
eth0
172.18.0.3
```

같은 Host 위에 있지만 각 Container는 자기 Network 환경을 갖는 것처럼 동작합니다.

---

# 54. localhost란?

`localhost`는 특별한 Hostname입니다.

일반적으로 현재 Machine 자신을 의미합니다.

대표적인 Loopback Address는:

```text
127.0.0.1
```

입니다.

예를 들어 Host Computer에서:

```bash
curl http://localhost
```

라고 하면 현재 Host 안에서 실행 중인 Web Service를 찾습니다.

---

# 55. Container 안의 localhost는 누구인가?

여기서 아주 중요한 개념이 나옵니다.

WordPress Container 안에서:

```text
localhost
```

는 Host Computer가 아닙니다.

MariaDB Container도 아닙니다.

**현재 WordPress Container 자신**입니다.

```text
Host

├── NGINX Container
│   └── localhost = NGINX Container
│
├── WordPress Container
│   └── localhost = WordPress Container
│
└── MariaDB Container
    └── localhost = MariaDB Container
```

즉 WordPress Container에서 Database에 연결할 때:

```text
localhost:3306
```

이라고 하면 MariaDB를 찾는 것이 아닙니다.

WordPress Container 자기 자신 안의 3306 Port를 찾게 됩니다.

---

# 56. 서로 다른 Container를 어떻게 찾을까?

Docker는 여러 Container를 하나의 Docker Network에 연결할 수 있습니다.

```text
Docker Network

┌───────────┐
│   NGINX   │
└─────┬─────┘
      │
      │
┌─────▼─────┐
│ WordPress │
└─────┬─────┘
      │
      │
┌─────▼─────┐
│  MariaDB  │
└───────────┘
```

같은 Network에 연결되면 서로 통신할 수 있습니다.

---

# 57. Docker Network 생성

간단한 Network를 직접 만들어봅시다.

```bash
docker network create inception-net
```

현재 Docker Network 확인:

```bash
docker network ls
```

특정 Network 정보:

```bash
docker network inspect inception-net
```

지금은 출력의 모든 내용을 이해할 필요는 없습니다.

핵심은:

```text
Docker Network
→ Container를 서로 연결하는 Logical Network
```

라는 점입니다.

---

# 58. Container를 Network에 연결

Container를 실행할 때 Network를 지정할 수 있습니다.

```bash
docker run \
  --name database \
  --network inception-net \
  mariadb
```

다른 Container도 같은 Network에 연결할 수 있습니다.

```bash
docker run \
  --name app \
  --network inception-net \
  some-image
```

그러면 두 Container는 같은 Docker Network 안에 존재합니다.

---

# 59. Container IP를 직접 사용해야 할까?

Container도 내부적으로 IP Address를 가질 수 있습니다.

예를 들어:

```text
WordPress
172.18.0.2

MariaDB
172.18.0.3
```

하지만 이 IP를 Configuration에 직접 Hard Coding하는 것은 좋지 않습니다.

Container는 재생성될 수 있기 때문입니다.

```text
Old MariaDB
172.18.0.3

삭제

New MariaDB
172.18.0.7
```

IP가 바뀔 수 있습니다.

그래서 Docker에서는 보통 **이름을 이용해서 서로를 찾습니다.**

---

# 60. Docker DNS

같은 Docker Network 안에서는 Container 또는 Compose Service 이름을 Hostname처럼 사용할 수 있습니다.

예를 들어 MariaDB Service 이름이:

```text
mariadb
```

라면 WordPress에서:

```text
mariadb:3306
```

으로 접근할 수 있습니다.

Docker가 내부적으로 이름을 IP로 해석합니다.

```text
wordpress
   │
   │ "mariadb가 어디 있지?"
   ▼
Docker DNS
   │
   ▼
MariaDB Container IP
```

이것은 매우 중요한 개념입니다.

그래서 Inception Configuration에서는 흔히:

```text
DB_HOST=mariadb
```

같은 구조를 볼 수 있습니다.

---

# 61. Service Name이 중요한 이유

Container IP를 기억할 필요 없이 Logical Name으로 Service를 찾을 수 있습니다.

```text
nginx
wordpress
mariadb
```

이 이름들이 Application Architecture 자체를 표현하게 됩니다.

```text
nginx
  ↓
wordpress:9000
  ↓
mariadb:3306
```

Container가 재생성되어 IP가 바뀌어도 Service Name을 통해 다시 찾을 수 있습니다.

---

# 62. Bridge Network

Docker에서 흔히 사용하는 Network 방식 중 하나가 Bridge Network입니다.

개념적으로 Host 안에 작은 Virtual Network를 하나 만든다고 생각하면 됩니다.

```text
Host

        Docker Bridge Network
     ┌────────────────────────┐
     │                        │
     │  nginx      wordpress  │
     │    │             │     │
     │    └──────┬──────┘     │
     │           │            │
     │        mariadb         │
     │                        │
     └────────────────────────┘
```

Container들은 이 Network를 통해 서로 통신합니다.

중요한 것은 Bridge의 내부 구현을 모두 외우는 것이 아니라:

> **Container들이 Host 내부의 별도 Network를 통해 서로 연결될 수 있다.**

라고 이해하는 것입니다.

---

# 63. 내부 Network와 외부 Network

Container끼리 통신하는 것과 외부 Client가 Container에 접근하는 것은 다른 문제입니다.

```text
Container 내부 통신

NGINX
  ↓
WordPress
  ↓
MariaDB
```

이것은 Docker Network 안에서 해결할 수 있습니다.

하지만 Browser는 Docker Network 밖에 있습니다.

```text
Browser
   │
   │ ???
   ▼
Docker Host
   │
   ▼
NGINX Container
```

외부 Traffic을 Container 안으로 전달해야 합니다.

여기서 **Port Mapping**이 등장합니다.

---

# 64. Port Mapping

Docker Host의 Port와 Container의 Port를 연결할 수 있습니다.

예:

```bash
docker run -p 8080:80 nginx
```

의미:

```text
Host Port 8080
      │
      │ Mapping
      ▼
Container Port 80
```

Browser에서:

```text
http://HOST_IP:8080
```

으로 접근하면 Container 안의 80 Port로 Traffic이 전달됩니다.

---

# 65. EXPOSE와 Port Mapping은 같은 것인가?

아닙니다.

Dockerfile에서:

```dockerfile
EXPOSE 443
```

을 볼 수 있습니다.

`EXPOSE`는:

> 이 Image의 Application이 이 Port를 사용할 예정이라고 표현하는 정보

에 가깝습니다.

Host에 실제 Port를 공개하려면:

```bash
docker run -p ...
```

또는 Docker Compose의:

```yaml
ports:
```

가 필요합니다.

---

# 66. 왜 NGINX만 외부에 공개할까?

Inception의 구조를 다시 봅시다.

```text
Browser
   ↓
NGINX
   ↓
WordPress
   ↓
MariaDB
```

외부 사용자가 직접 MariaDB에 접근할 이유는 없습니다.

WordPress도 일반적으로 Browser와 직접 통신하는 Front Door가 아닙니다.

외부에서 필요한 것은 NGINX입니다.

그래서 구조적으로:

```text
Internet
   │
   │ 443
   ▼
NGINX
   │
   │ internal network
   ▼
WordPress
   │
   │ internal network
   ▼
MariaDB
```

처럼 만들 수 있습니다.

이것은 Security 관점에서도 중요합니다.

필요한 Service만 외부에 노출하는 것이 좋습니다.

---

# 67. 이제 Inception의 Application들을 이해해보자

Container와 Docker의 기본 구조를 이해했습니다.

이제 실제 Inception에서 사용하는 세 Application을 살펴봅시다.

```text
NGINX
WordPress + PHP-FPM
MariaDB
```

이 세 가지가 왜 필요한지 이해해야 전체 Architecture가 보입니다.

---

# 68. Website는 어떻게 동작할까?

사용자가 Browser에 URL을 입력한다고 생각해봅시다.

```text
https://example.com
```

Browser는 해당 Server에 Request를 보냅니다.

```text
Browser
   │
   │ Request
   ▼
Web Server
   │
   │ Response
   ▼
Browser
```

예:

```text
Request
GET /

Response
HTML
```

이 Request와 Response를 위한 대표적인 Protocol이 HTTP입니다.

---

# 69. HTTP

HTTP는 **Hypertext Transfer Protocol**입니다.

Web에서 Client와 Server가 Request와 Response를 주고받는 규칙입니다.

예를 들어 Browser가:

```text
GET /index.html
```

을 요청하면 Server는:

```text
200 OK
HTML Content
```

를 반환할 수 있습니다.

구조:

```text
Client
  │
  │ HTTP Request
  ▼
Server
  │
  │ HTTP Response
  ▼
Client
```

---

# 70. Web Server

Web Server는 HTTP Request를 받아 Response를 반환하는 Server Software입니다.

대표적인 예:

```text
NGINX
Apache HTTP Server
```

Inception에서는 NGINX를 사용합니다.

---

# 71. NGINX

NGINX는 Web Server이자 Reverse Proxy 등 여러 역할을 수행할 수 있는 Server Software입니다.

Inception에서는 특히 다음 역할이 중요합니다.

```text
HTTPS Connection 수신
        ↓
TLS 처리
        ↓
HTTP Request 확인
        ↓
Static File 처리 또는
PHP Request를 PHP-FPM에 전달
```

즉 NGINX는 Application 앞에 서 있는 **Entry Point**입니다.

---

# 72. Static Content와 Dynamic Content

Web Content는 아주 단순하게 두 종류로 생각할 수 있습니다.

## Static Content

이미 존재하는 File을 그대로 반환합니다.

예:

```text
HTML
CSS
Image
JavaScript File
```

```text
Browser
   ↓
NGINX
   ↓
File
   ↓
Browser
```

---

## Dynamic Content

Request에 따라 Server에서 Code를 실행해서 결과를 만듭니다.

WordPress가 대표적입니다.

예:

```text
사용자 로그인
글 목록 조회
댓글 작성
관리자 페이지
```

이런 기능은 Database를 조회하고 PHP Code를 실행해야 합니다.

---

# 73. WordPress

WordPress는 Website를 만들고 관리하기 위한 Web Application입니다.

다음과 같은 Data를 다룹니다.

```text
Users
Posts
Pages
Comments
Settings
Plugins
Metadata
```

WordPress 자체는 PHP로 작성되어 있습니다.

즉 `.php` File을 실행할 수 있는 환경이 필요합니다.

---

# 74. PHP란?

PHP는 Server-side Programming Language입니다.

Browser가 PHP Source Code를 직접 실행하는 것이 아닙니다.

예를 들어 Server에:

```php
<?php
echo "Hello";
?>
```

가 있어도 Browser가 이것을 직접 실행하는 것이 아니라 Server 쪽에서 PHP Interpreter가 실행합니다.

```text
PHP Code
   ↓
PHP Runtime
   ↓
실행 결과
   ↓
HTML 등
```

---

# 75. NGINX는 PHP를 실행할 수 있을까?

NGINX는 기본적으로 PHP Interpreter가 아닙니다.

NGINX의 역할은 Web Request를 처리하는 것입니다.

PHP Code 실행은 PHP Runtime이 담당합니다.

그래서 역할을 분리합니다.

```text
NGINX
→ HTTP 요청 처리

PHP
→ PHP Code 실행
```

이 둘을 연결하기 위한 방법이 필요합니다.

---

# 76. PHP-FPM

PHP-FPM은:

**PHP FastCGI Process Manager**

입니다.

PHP 요청을 처리하기 위한 Process를 관리합니다.

개념적으로:

```text
NGINX
   │
   │ PHP 실행 요청
   ▼
PHP-FPM
   │
   ▼
PHP Script 실행
   │
   ▼
Result
```

WordPress Container 안에서 PHP-FPM이 실행되는 구조를 생각할 수 있습니다.

---

# 77. 왜 그냥 PHP를 실행하지 않고 PHP-FPM을 사용할까?

Web Server는 동시에 많은 Request를 받을 수 있습니다.

그때마다 PHP 실행 환경을 효율적으로 관리해야 합니다.

PHP-FPM은 PHP Worker Process들을 관리하는 역할을 합니다.

개념적으로:

```text
PHP-FPM
   │
   ├── PHP Worker
   ├── PHP Worker
   └── PHP Worker
```

Web Server는 PHP-FPM에 요청을 전달하고,
PHP-FPM이 적절한 Process를 통해 PHP Code를 실행합니다.

---

# 78. FastCGI

NGINX와 PHP-FPM은 HTTP로 통신하는 것이 아니라 FastCGI Protocol을 사용할 수 있습니다.

```text
Browser
   │
   │ HTTP / HTTPS
   ▼
NGINX
   │
   │ FastCGI
   ▼
PHP-FPM
```

FastCGI는 Web Server와 Application Runtime 사이에서 Request 정보를 전달하기 위한 Protocol입니다.

즉:

```text
HTTP
→ Browser ↔ NGINX

FastCGI
→ NGINX ↔ PHP-FPM
```

라고 구분하면 됩니다.

---

# 79. 왜 NGINX가 WordPress에 바로 보내는 게 아닐까?

WordPress는 PHP File들의 집합인 Application입니다.

실제로 Code를 실행하는 것은 PHP Runtime입니다.

따라서 더 정확한 흐름은:

```text
NGINX
   ↓
PHP-FPM
   ↓
WordPress PHP Code
```

입니다.

보통 설명할 때:

```text
NGINX
→ WordPress
```

라고 단순화하지만 실제로는 PHP-FPM이 중간에서 PHP Code를 실행합니다.

---

# 80. NGINX Configuration에서 fastcgi_pass

NGINX Configuration에서는 다음과 같은 설정을 볼 수 있습니다.

```nginx
fastcgi_pass wordpress:9000;
```

이것을 앞에서 배운 개념과 연결해봅시다.

```text
wordpress
→ Docker Network에서 WordPress Service 이름

9000
→ PHP-FPM이 Listen하는 Port
```

즉:

```text
NGINX
   │
   │ wordpress:9000
   ▼
WordPress Container
   │
   ▼
PHP-FPM
```

입니다.

---

# 81. MariaDB

WordPress는 Data를 영구적으로 저장해야 합니다.

예를 들어:

```text
User Account
Post
Comment
WordPress Setting
Plugin Data
```

이런 Data를 File만으로 직접 관리하는 대신 Database Management System을 사용합니다.

Inception에서는 MariaDB를 사용합니다.

---

# 82. Database란?

Database는 구조화된 Data를 저장하고 조회하기 위한 System입니다.

예를 들어 User 정보를 생각해봅시다.

```text
users

id | username | email
---|----------|------
1  | kim      | ...
2  | lee      | ...
```

Application은 Database에 Query를 보내 Data를 저장하거나 가져옵니다.

---

# 83. DBMS

Database를 실제로 관리하는 Software를 DBMS라고 합니다.

**Database Management System**

대표적인 예:

```text
MariaDB
MySQL
PostgreSQL
```

Inception에서는 MariaDB Server가 실행됩니다.

---

# 84. WordPress와 MariaDB

WordPress는 MariaDB에 연결해서 Data를 읽고 씁니다.

```text
WordPress
   │
   │ SQL
   ▼
MariaDB
```

예를 들어 사용자가 글을 열면:

```text
Browser
   ↓
NGINX
   ↓
PHP-FPM
   ↓
WordPress
   ↓
MariaDB에서 Post 조회
   ↓
WordPress가 HTML 생성
   ↓
NGINX
   ↓
Browser
```

와 같은 흐름이 만들어질 수 있습니다.

---

# 85. SQL

SQL은 Relational Database와 Communication하기 위해 사용하는 Query Language입니다.

예를 들어:

```sql
SELECT * FROM users;
```

는 `users` Table의 Data를 조회하는 Query입니다.

Inception의 목표는 SQL 자체를 깊게 배우는 것이 아니지만:

```text
WordPress
→ SQL Query
→ MariaDB
```

라는 관계는 이해해야 합니다.

---

# 86. MariaDB Port 3306

MariaDB는 일반적으로 TCP Port 3306을 사용할 수 있습니다.

```text
WordPress
     │
     │ mariadb:3306
     ▼
MariaDB
```

여기서 다시 Docker DNS 개념이 연결됩니다.

```text
mariadb
→ Service Name

3306
→ Database Service Port
```

---

# 87. MariaDB를 Internet에 공개해야 할까?

일반적으로 Inception Architecture에서는 그럴 필요가 없습니다.

```text
Internet
   │
   ▼
NGINX
```

외부 Request는 NGINX만 받습니다.

MariaDB는 Docker Internal Network에서 WordPress와 통신하면 됩니다.

```text
Internet
   │
   ▼
NGINX
   │
   ▼
WordPress
   │
   ▼
MariaDB
```

따라서 Database Port를 Host에 불필요하게 Publish하지 않아도 됩니다.

---

# 88. 전체 Application 구조

이제 세 Service의 역할을 연결할 수 있습니다.

```text
NGINX
→ Web Server / Entry Point / TLS

WordPress
→ Web Application

PHP-FPM
→ PHP 실행 환경

MariaDB
→ Persistent Application Data
```

전체 Flow:

```text
Browser
   │
   │ HTTPS
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
   │
   │ SQL
   ▼
MariaDB
```

---

# 89. 그런데 HTTPS는 무엇인가?

지금까지 HTTP Request를 이야기했습니다.

하지만 Inception에서는 HTTPS를 사용합니다.

HTTPS는 HTTP를 TLS 위에서 사용하는 방식입니다.

```text
HTTP
→ Web Request / Response Protocol

TLS
→ Communication을 암호화하고 상대 Identity를 확인하는 Protocol

HTTPS
→ HTTP over TLS
```

---

# 90. HTTP의 문제

Plain HTTP Traffic은 암호화되지 않을 수 있습니다.

개념적으로:

```text
Client
   │
   │ Plain HTTP
   ▼
Server
```

Network 중간에서 Traffic을 관찰할 수 있는 환경이라면 Content가 노출될 위험이 있습니다.

예를 들어:

```text
Password
Cookie
Form Data
```

같은 정보가 보호되어야 합니다.

그래서 TLS를 사용합니다.

---

# 91. TLS

TLS는 **Transport Layer Security**입니다.

Client와 Server 사이의 Communication을 보호합니다.

주요 목적은 크게 다음과 같이 생각할 수 있습니다.

```text
Confidentiality
→ 내용을 암호화

Integrity
→ 내용이 중간에서 변경되지 않았는지 보호

Authentication
→ 상대 Server의 Identity 확인
```

---

# 92. HTTPS Connection

HTTPS Request는 대략 다음 흐름을 가집니다.

```text
Browser
   │
   │ TLS Connection
   ▼
NGINX
   │
   │ TLS 처리
   ▼
HTTP Request
```

즉 NGINX가 TLS를 종료하고 내부 Request를 처리하는 Entry Point 역할을 할 수 있습니다.

---

# 93. Certificate

TLS Server는 Certificate를 사용합니다.

Certificate에는 Server의 Public Identity와 관련된 정보가 들어 있습니다.

개념적으로:

```text
Certificate
→ "이 Public Key는 이 Server와 관련 있다."
```

라고 생각할 수 있습니다.

Browser는 Certificate를 이용해 Server Identity를 검증합니다.

---

# 94. Private Key

Certificate와 함께 Private Key가 사용됩니다.

```text
Certificate
→ 공개 가능

Private Key
→ Server만 비밀로 보관
```

Private Key가 유출되면 Security에 큰 문제가 생길 수 있으므로 안전하게 관리해야 합니다.

---

# 95. Self-signed Certificate

실제 Public Website에서는 일반적으로 신뢰할 수 있는 Certificate Authority가 발급한 Certificate를 사용합니다.

하지만 학습 환경에서는 Self-signed Certificate를 만들 수도 있습니다.

예:

```bash
openssl req -x509 -nodes \
  -newkey rsa:2048 \
  -keyout server.key \
  -out server.crt \
  -days 365
```

이 Command를 외울 필요는 없습니다.

핵심은:

```text
server.key
→ Private Key

server.crt
→ Certificate
```

라는 점입니다.

---

# 96. 왜 Browser가 Self-signed Certificate를 경고할까?

Self-signed Certificate는 우리가 직접 서명한 Certificate입니다.

Browser가 기본적으로 신뢰하는 Certificate Authority가 서명한 것이 아닙니다.

따라서 Browser 입장에서는:

```text
"암호화는 가능하지만,
이 Certificate를 누가 믿을 수 있다고 보증하지?"
```

라는 문제가 생깁니다.

그래서 경고가 표시될 수 있습니다.

---

# 97. NGINX에서 TLS 설정

예:

```nginx
server {
    listen 443 ssl;

    ssl_certificate     /etc/nginx/ssl/server.crt;
    ssl_certificate_key /etc/nginx/ssl/server.key;
}
```

개념적으로:

```text
listen 443 ssl
→ 443 Port에서 TLS Connection 수신

ssl_certificate
→ Certificate 위치

ssl_certificate_key
→ Private Key 위치
```

입니다.

---

# 98. 이제 요청 하나를 끝까지 따라가보자

사용자가 Browser에:

```text
https://example.com
```

을 입력합니다.

---

## Step 1. Client가 Server에 연결

```text
Browser
   │
   │ TCP / 443
   ▼
Server
```

NGINX Container가 Host의 443 Port를 통해 Request를 받도록 구성되어 있습니다.

---

## Step 2. TLS

```text
Browser
   │
   │ TLS
   ▼
NGINX
```

NGINX는 Certificate와 Private Key를 사용하여 HTTPS Connection을 처리합니다.

---

## Step 3. HTTP Request

TLS Connection 안에서 HTTP Request가 전달됩니다.

예:

```text
GET /
```

---

## Step 4. NGINX

NGINX는 Request를 확인합니다.

Static File이라면 직접 반환할 수도 있습니다.

PHP Request라면 PHP-FPM에 전달합니다.

```text
NGINX
   │
   │ FastCGI
   ▼
wordpress:9000
```

---

## Step 5. PHP-FPM

PHP-FPM은 WordPress의 PHP Code를 실행합니다.

```text
PHP-FPM
   ↓
WordPress PHP
```

---

## Step 6. Database

WordPress가 Data를 필요로 한다면 MariaDB에 Query를 보냅니다.

```text
WordPress
   │
   │ mariadb:3306
   ▼
MariaDB
```

---

## Step 7. 결과 생성

MariaDB가 Data를 반환합니다.

```text
MariaDB
   ↓
WordPress
   ↓
PHP-FPM
```

WordPress는 필요한 HTML 등을 생성합니다.

---

## Step 8. Response

```text
PHP-FPM
   ↓
NGINX
   ↓
HTTPS Response
   ↓
Browser
```

사용자는 최종 Website를 보게 됩니다.

---

# 99. 이 구조에서 Container는 무엇을 분리했을까?

각 역할은 서로 다른 Container에 있습니다.

```text
NGINX Container
→ HTTP / TLS

WordPress Container
→ Application / PHP-FPM

MariaDB Container
→ Database
```

각 Container는 서로 다른 Process 환경과 Filesystem을 가집니다.

하지만 Docker Network를 통해 필요한 Communication은 허용됩니다.

---

# 100. Environment Variable

Application Configuration 중에는 Image 안에 고정하면 안 되는 값들이 있습니다.

예:

```text
Database Name
Database User
Domain Name
Environment-specific Setting
```

이런 값을 Environment Variable로 전달할 수 있습니다.

Shell에서는:

```bash
export DB_NAME=wordpress
```

확인:

```bash
echo "$DB_NAME"
```

---

# 101. Container에 Environment Variable 전달

예:

```bash
docker run \
  -e DB_NAME=wordpress \
  some-image
```

여기서:

```text
-e
→ Environment Variable 전달
```

입니다.

Application은 실행 중 이 값을 읽을 수 있습니다.

---

# 102. 왜 Configuration을 Image와 분리할까?

Image 안에 모든 값을 Hard Coding하면 같은 Image를 다른 환경에서 재사용하기 어렵습니다.

```text
Development
DB_NAME=dev

Production
DB_NAME=prod
```

같은 Image를 사용하면서 실행 Configuration만 다르게 전달할 수 있으면 더 유연합니다.

```text
Image
  │
  ├── Environment A
  └── Environment B
```

---

# 103. Password도 Environment Variable에 넣으면 끝일까?

아닙니다.

Environment Variable은 Configuration 전달 방법이지 자동으로 안전한 Secret Storage가 되는 것은 아닙니다.

Password나 Key 같은 민감한 정보는 별도로 주의해서 관리해야 합니다.

특히 Git Repository에 Plain Text Password를 Commit하는 것은 피해야 합니다.

---

# 104. .env File

Docker Compose에서는 반복되는 값을 `.env` File에서 읽어 사용할 수 있습니다.

예:

```dotenv
DOMAIN_NAME=example.local
MYSQL_DATABASE=wordpress
MYSQL_USER=wpuser
```

Compose에서는:

```yaml
environment:
  MYSQL_DATABASE: ${MYSQL_DATABASE}
  MYSQL_USER: ${MYSQL_USER}
```

처럼 사용할 수 있습니다.

핵심은:

```text
Image
→ Application 실행 환경

Environment Variable
→ 실행 시 달라질 수 있는 Configuration
```

이라는 분리입니다.

---

# 105. Container가 세 개면 명령어도 세 배가 된다

지금까지 배운 것만으로도 Inception을 수동으로 실행할 수 있습니다.

예를 들어:

```text
Network 생성
Volume 생성
MariaDB 실행
WordPress 실행
NGINX 실행
Port 설정
Environment Variable 전달
Restart Policy 설정
```

해야 합니다.

CLI만 사용하면 대략 이런 느낌입니다.

```bash
docker network create inception

docker volume create mariadb-data
docker volume create wordpress-data

docker run ...
docker run ...
docker run ...
```

Service가 많아질수록 Command도 복잡해집니다.

---

# 106. 새로운 문제: Application 전체를 어떻게 재현할까?

Dockerfile 덕분에 **Image 하나**를 재현할 수 있게 되었습니다.

```text
Dockerfile
→ Image
```

그런데 Inception은 Image 하나가 아닙니다.

```text
NGINX
WordPress
MariaDB
Network
Volumes
Ports
Environment
```

모두 함께 구성되어야 합니다.

그래서 다음 질문이 생깁니다.

> **Multi-Container Application 전체 구조를 File 하나에 선언할 수 없을까?**

그 역할을 하는 것이 Docker Compose입니다.

---

# 107. Docker Compose

Docker Compose는 여러 Container로 이루어진 Application의 실행 Configuration을 YAML File에 정의할 수 있게 합니다.

일반적으로:

```text
compose.yaml
```

또는:

```text
docker-compose.yml
```

형태의 File을 사용합니다.

---

# 108. Dockerfile과 Compose의 차이

이 구분은 반드시 이해해야 합니다.

## Dockerfile

```text
Image를 어떻게 만들 것인가?
```

예:

```text
Debian에서 시작
NGINX 설치
Config 복사
```

---

## Docker Compose

```text
만들어진 Service들을 어떻게 실행하고 연결할 것인가?
```

예:

```text
NGINX 실행
WordPress 실행
MariaDB 실행
Network 연결
Volume 연결
Port 공개
```

즉:

```text
Dockerfile
→ Build

Compose
→ Runtime Architecture
```

라고 생각하면 좋습니다.

---

# 109. Compose 기본 구조

간단한 예:

```yaml
services:
  nginx:
    build: ./nginx

  wordpress:
    build: ./wordpress

  mariadb:
    build: ./mariadb
```

`services` 아래에 Application을 구성하는 Service들을 정의합니다.

---

# 110. Service란?

Compose에서 Service는 Application을 구성하는 Logical Component입니다.

예:

```text
nginx
wordpress
mariadb
```

각 Service는 Image, Network, Volume, Environment 등을 정의할 수 있습니다.

개념적으로:

```text
Service Definition
      ↓
Container 실행
```

이라고 볼 수 있습니다.

---

# 111. Compose의 build

예:

```yaml
services:
  nginx:
    build: ./requirements/nginx
```

의미:

```text
./requirements/nginx
       ↓
Dockerfile
       ↓
Image Build
       ↓
NGINX Service 실행
```

입니다.

---

# 112. Compose의 ports

예:

```yaml
services:
  nginx:
    ports:
      - "443:443"
```

의미:

```text
Host 443
   ↓
NGINX Container 443
```

외부 Browser가 NGINX에 접근할 수 있습니다.

---

# 113. Compose의 networks

예:

```yaml
services:
  nginx:
    networks:
      - inception

  wordpress:
    networks:
      - inception

  mariadb:
    networks:
      - inception
```

그리고:

```yaml
networks:
  inception:
```

를 정의할 수 있습니다.

세 Service는 같은 Network를 공유합니다.

```text
inception Network

nginx
  │
wordpress
  │
mariadb
```

---

# 114. Compose의 volumes

예:

```yaml
services:
  mariadb:
    volumes:
      - mariadb-data:/var/lib/mysql

volumes:
  mariadb-data:
```

의미:

```text
MariaDB Container
     │
     │ /var/lib/mysql
     ▼
mariadb-data Volume
```

Container와 Database Data의 Lifecycle을 분리합니다.

---

# 115. depends_on

Compose에서는 Service 간 시작 순서를 표현할 수 있습니다.

예:

```yaml
depends_on:
  - mariadb
```

하지만 매우 중요한 점이 있습니다.

> Container가 시작되었다고 Application이 Ready 상태라는 뜻은 아닙니다.

예를 들어 MariaDB Process가 시작된 직후 내부 Initialization이 아직 끝나지 않았을 수 있습니다.

```text
Container Running
≠
Application Ready
```

이 차이는 실제 Service 운영에서 매우 중요합니다.

---

# 116. restart policy

Server Application은 Crash할 수 있습니다.

Docker는 Restart Policy를 통해 Container를 다시 시작하도록 설정할 수 있습니다.

예:

```yaml
restart: unless-stopped
```

개념적으로:

```text
Process 종료
    ↓
Container 종료
    ↓
Restart Policy
    ↓
Container 다시 시작
```

입니다.

앞에서 배운 Container Lifecycle과 연결됩니다.

---

# 117. Compose 실행

Application 전체를 실행:

```bash
docker compose up
```

Background에서 실행:

```bash
docker compose up -d
```

여기서:

```text
up
→ Compose Application 실행

-d
→ Detached Mode
```

입니다.

---

# 118. Compose 상태 확인

```bash
docker compose ps
```

각 Service 상태를 확인할 수 있습니다.

예:

```text
nginx       running
wordpress   running
mariadb     running
```

이것은 Inception Debugging에서 가장 먼저 확인하기 좋은 명령 중 하나입니다.

---

# 119. Compose Logs

전체 Log:

```bash
docker compose logs
```

특정 Service:

```bash
docker compose logs nginx
```

실시간:

```bash
docker compose logs -f
```

문제가 생겼을 때 가장 먼저 확인해야 할 정보 중 하나입니다.

---

# 120. Compose Container 안에 들어가기

예:

```bash
docker compose exec nginx sh
```

WordPress:

```bash
docker compose exec wordpress sh
```

MariaDB:

```bash
docker compose exec mariadb sh
```

Container 내부 Filesystem, Process, Configuration 등을 확인할 때 사용합니다.

---

# 121. Compose 종료

```bash
docker compose down
```

Compose로 실행된 Container와 Network 등을 종료하고 정리합니다.

Volume까지 제거:

```bash
docker compose down -v
```

하지만 이것은 주의해야 합니다.

```text
-v
→ Volume 삭제 가능
→ Persistent Data 손실 가능
```

Database Data를 유지해야 한다면 무심코 사용하면 안 됩니다.

---

# 122. 이제 전체 구조를 Compose 관점에서 보자

```text
compose.yaml
     │
     ├── nginx service
     │      ├── Dockerfile
     │      ├── Port 443
     │      └── Network
     │
     ├── wordpress service
     │      ├── Dockerfile
     │      ├── PHP-FPM
     │      └── Network
     │
     └── mariadb service
            ├── Dockerfile
            ├── Volume
            └── Network
```

Compose는 세 Container를 단순히 동시에 실행하는 것이 아니라,
**Application 전체 Architecture를 선언**합니다.

---

# 123. Inception 전체 요청 흐름

이제 처음부터 끝까지 한 번에 연결해봅시다.

```text
User
 │
 │ Browser
 ▼
HTTPS Request
 │
 ▼
Host :443
 │
 │ Port Mapping
 ▼
NGINX Container
 │
 │ TLS Termination
 │
 │ HTTP Request 처리
 │
 │ FastCGI
 ▼
WordPress Container
 │
 │ PHP-FPM
 │
 │ WordPress PHP 실행
 │
 │ SQL
 ▼
MariaDB Container
 │
 │
 ▼
Database Volume
```

Response는 반대 방향으로 돌아옵니다.

```text
Database
   ↓
WordPress
   ↓
PHP-FPM
   ↓
NGINX
   ↓
HTTPS
   ↓
Browser
```

---

# 124. Data 흐름

Application Request와 Persistent Data는 다른 흐름을 가집니다.

## Request Flow

```text
Browser
→ NGINX
→ PHP-FPM
→ WordPress
→ MariaDB
```

## Persistent Storage

```text
WordPress Files
→ WordPress Volume

Database Files
→ MariaDB Volume
```

즉 Network와 Storage는 서로 다른 문제를 해결합니다.

```text
Network
→ Service끼리 어떻게 통신하는가?

Volume
→ Data를 어디에 지속적으로 저장하는가?
```

---

# 125. Docker의 핵심 Object 정리

Docker를 이해할 때 다음 다섯 개를 구분하면 좋습니다.

```text
Image
→ Container 실행 환경의 원본

Container
→ Image를 기반으로 실행되는 격리된 Process 환경

Network
→ Container 간 Communication

Volume
→ Persistent Data Storage

Dockerfile
→ Image Build 방법

Compose
→ 여러 Service의 실행 Architecture
```

---

# 126. Container를 다시 정확하게 정의해보자

문서 시작에서는 Container를 단순하게:

> Application을 격리된 환경에서 실행하는 방법

이라고 봤습니다.

이제 더 정확하게 설명할 수 있습니다.

```text
Container

Linux Process
+
Namespaces
+
cgroups
+
Isolated Filesystem View
+
Network Environment
```

그리고 Docker는 이것을 쉽게 만들고 관리합니다.

---

# 127. 왜 Container는 VM이 아닌가?

다시 한번 확인합니다.

```text
VM
→ Virtual Machine
→ Guest Kernel 존재

Container
→ Isolated Process Environment
→ Host Kernel 공유
```

그래서 Container 안에 Debian Filesystem이 있다고 해서 Debian Kernel이 하나 더 실행되는 것은 아닙니다.

Host Linux Kernel을 공유합니다.

이 차이를 이해하면 Container 개념의 절반 이상을 이해한 것입니다.

---

# 128. 왜 Container의 Process가 중요할까?

Container는 Application Package를 보관하는 Box가 아닙니다.

실행 관점에서는 Process가 핵심입니다.

```text
docker run
   ↓
Container Environment 준비
   ↓
Main Process 실행
   ↓
Running
```

Main Process 종료:

```text
Main Process Exit
   ↓
Container Exit
```

그래서:

```text
PID 1
foreground
daemon off
signal
restart policy
```

같은 개념들이 모두 연결됩니다.

---

# 129. Signal

Linux에서는 Process에 Signal을 보내 특정 동작을 요청할 수 있습니다.

예:

```text
SIGTERM
→ 정상 종료 요청

SIGKILL
→ 강제 종료
```

Docker에서:

```bash
docker stop nginx
```

을 실행하면 Docker는 Container의 Main Process가 정상적으로 종료할 기회를 주는 방식으로 Signal을 전달합니다.

그래서 Container Application은 Signal을 올바르게 처리하는 것이 중요합니다.

---

# 130. `docker stop`과 `docker kill`

개념적으로:

```bash
docker stop container
```

은 정상 종료를 시도합니다.

반면:

```bash
docker kill container
```

은 더 즉각적인 종료에 사용할 수 있습니다.

보통은 `stop`을 우선 사용하는 것이 자연스럽습니다.

---

# 131. Zombie Process와 PID 1

Linux에서 Child Process가 종료되면 Parent Process가 그 종료 상태를 회수해야 합니다.

그렇지 않으면 Zombie Process가 남을 수 있습니다.

일반 Linux System에서는 init 계열 Process가 고아 Process 처리에 관여합니다.

Container에서는 Application 자체가 PID 1이 될 수 있기 때문에 Process를 많이 Spawn하는 Application에서는 PID 1의 Process 관리 역할도 중요해질 수 있습니다.

Inception 수준에서는 다음 정도를 기억하면 충분합니다.

> Container의 PID 1은 단순한 숫자 1이 아니라 Container Lifecycle과 Process 관리에서 특별한 위치를 가진다.

---

# 132. Image Layer와 Cache

Dockerfile의 Instruction은 Image Layer 생성과 Build Cache에 영향을 줄 수 있습니다.

예:

```dockerfile
FROM debian:bookworm

RUN apt-get update && apt-get install -y nginx

COPY conf/nginx.conf /etc/nginx/nginx.conf
```

개념적으로:

```text
Base Layer
   ↓
Package Layer
   ↓
Config Layer
```

Docker는 변경되지 않은 Build Step을 Cache에서 재사용할 수 있습니다.

그래서 Dockerfile의 구조가 Build 효율에도 영향을 줍니다.

---

# 133. Container Writable Layer는 영구 Storage가 아니다

Container 안에서:

```bash
touch hello.txt
```

를 실행하면 File을 만들 수 있습니다.

하지만 그 File이 Container Writable Layer에만 있다면 Container 삭제 시 함께 사라질 수 있습니다.

그래서:

```text
Temporary Runtime Change
→ Container Writable Layer

Persistent Application Data
→ Volume
```

이라고 구분하는 것이 좋습니다.

---

# 134. Container를 자주 다시 만드는 이유

전통적인 Server 관리에서는 하나의 Server를 오랫동안 유지하면서 내부를 계속 수정하는 방식이 흔했습니다.

Container 환경에서는 Image를 새로 Build하고 Container를 교체하는 방식이 자연스럽습니다.

```text
Old Image
   ↓
Old Container

Dockerfile 수정

New Image
   ↓
New Container
```

이런 방식은 실행 환경을 재현 가능하게 유지하는 데 도움을 줍니다.

---

# 135. Immutable Infrastructure라는 생각

Container 환경에서는 실행 중인 Server 내부를 계속 손으로 수정하기보다:

```text
Configuration 수정
   ↓
Dockerfile 수정
   ↓
Image 다시 Build
   ↓
Container 교체
```

하는 방식을 선호할 수 있습니다.

이를 더 넓은 Infrastructure 개념에서는 Immutable Infrastructure와 연결해서 이해할 수 있습니다.

Inception에서는 이 철학을 완전히 구현하는 것이 목표라기보다,
**재현 가능한 Image와 Container 운영 방식**을 경험한다고 생각하면 됩니다.

---

# 136. 기본 Debugging: 먼저 상태를 본다

Container가 동작하지 않을 때 무작정 Configuration부터 수정하지 않습니다.

먼저:

```bash
docker compose ps
```

를 봅니다.

질문은:

> Container가 Running 상태인가?

입니다.

---

# 137. 종료되어 있다면 Logs를 본다

```bash
docker compose logs nginx
```

또는:

```bash
docker compose logs mariadb
```

를 사용합니다.

질문:

> Application이 왜 종료되었는가?

입니다.

---

# 138. 살아 있는데 이상하다면 내부를 본다

```bash
docker compose exec nginx sh
```

Container 안에서:

```bash
ps
```

```bash
ls
```

```bash
cat <configuration-file>
```

등으로 실제 상태를 확인할 수 있습니다.

---

# 139. NGINX 설정 확인

NGINX Configuration Syntax를 확인할 때:

```bash
nginx -t
```

를 사용할 수 있습니다.

예:

```text
syntax is ok
test is successful
```

같은 결과를 볼 수 있습니다.

이 Command는:

> NGINX가 안 되는데 Docker 문제인가, NGINX Configuration 문제인가?

를 구분하는 데 도움이 됩니다.

---

# 140. 어느 Port를 듣고 있는지 확인

Linux에서는 다음과 같은 명령을 사용할 수 있습니다.

```bash
ss -tlnp
```

옵션을 단순하게 보면:

```text
-t
→ TCP

-l
→ Listening

-n
→ 숫자로 표시

-p
→ Process 정보
```

Container 안에서 실제 Application이 예상 Port를 Listen하고 있는지 확인하는 데 도움을 줍니다.

---

# 141. DNS 확인

WordPress Container가 MariaDB를 찾지 못한다면 Service Name Resolution을 확인할 수 있습니다.

예:

```bash
getent hosts mariadb
```

성공하면 `mariadb` 이름이 어떤 IP로 해석되는지 확인할 수 있습니다.

질문:

> 같은 Docker Network에서 Service 이름을 찾을 수 있는가?

입니다.

---

# 142. Network 확인

```bash
docker network ls
```

특정 Network:

```bash
docker network inspect <network-name>
```

를 통해 어떤 Container들이 연결되어 있는지 확인할 수 있습니다.

---

# 143. Volume 확인

```bash
docker volume ls
```

특정 Volume:

```bash
docker volume inspect <volume-name>
```

질문:

> Persistent Data가 어떤 Volume에 연결되어 있는가?

입니다.

---

# 144. HTTP Request 확인

NGINX까지 Request가 도달하는지 확인할 때 `curl`을 사용할 수 있습니다.

예:

```bash
curl -v https://example.local
```

`-v`는 Verbose Output을 보여줍니다.

TLS Verification을 건너뛰는 학습용 Debugging에서는:

```bash
curl -k https://example.local
```

를 볼 수 있습니다.

하지만 `-k`는 Certificate Verification을 생략하므로 실제 운영에서 무심코 사용하는 습관은 좋지 않습니다.

---

# 145. `docker inspect`

Docker Object의 상세 Configuration을 보고 싶다면:

```bash
docker inspect <container>
```

를 사용할 수 있습니다.

출력이 매우 길 수 있습니다.

처음부터 모두 읽기보다 다음과 같은 질문을 가지고 보는 것이 좋습니다.

```text
어떤 Image에서 만들어졌나?
어떤 Network에 연결됐나?
어떤 Mount가 있나?
어떤 Environment Variable이 있나?
```

---

# 146. Debugging의 핵심은 계층을 나누는 것

Inception 문제가 생겼을 때 모든 것을 한 번에 보면 복잡합니다.

계층별로 나눕니다.

```text
1. Container
   ↓
살아 있는가?

2. Process
   ↓
Application이 실행 중인가?

3. Configuration
   ↓
설정이 올바른가?

4. Network
   ↓
상대 Service를 찾을 수 있는가?

5. Port
   ↓
예상 Port를 Listen하는가?

6. Storage
   ↓
Volume이 제대로 연결됐는가?

7. Application
   ↓
실제 Request를 정상 처리하는가?
```

---

# 147. 질문 → 도구

```text
"Container가 살아 있나?"
→ docker compose ps

"왜 죽었지?"
→ docker compose logs

"Container 안에서는 무슨 일이 일어나지?"
→ docker compose exec

"NGINX 설정이 맞나?"
→ nginx -t

"어느 Port를 듣고 있지?"
→ ss -tlnp

"Service 이름을 찾을 수 있나?"
→ getent hosts

"Network에 누가 연결되어 있지?"
→ docker network inspect

"Volume은 어디에 연결됐지?"
→ docker volume inspect

"HTTP/TLS 요청은 되나?"
→ curl -v
```

명령어를 외우기보다 **질문에 맞는 도구를 고르는 것**이 더 중요합니다.

---

# 148. 자주 헷갈리는 개념 1: Image vs Container

```text
Image
→ 실행 환경의 원본

Container
→ Image에서 만들어진 실행 Instance
```

예:

```text
debian:bookworm
→ Image

그 Image를 docker run으로 실행
→ Container
```

---

# 149. 자주 헷갈리는 개념 2: Dockerfile vs Compose

```text
Dockerfile
→ Image를 어떻게 만들까?

Compose
→ 여러 Service를 어떻게 실행하고 연결할까?
```

---

# 150. 자주 헷갈리는 개념 3: Container vs VM

```text
VM
→ Guest Kernel 존재

Container
→ Host Kernel 공유
```

Container를 작은 VM이라고만 외우면 Process와 Namespace 개념이 보이지 않습니다.

---

# 151. 자주 헷갈리는 개념 4: EXPOSE vs ports

```text
EXPOSE
→ Image가 사용하는 Port에 대한 정보

ports
→ Host와 Container Port를 실제로 연결
```

---

# 152. 자주 헷갈리는 개념 5: localhost

```text
Host의 localhost
→ Host 자신

NGINX Container의 localhost
→ NGINX Container 자신

WordPress Container의 localhost
→ WordPress Container 자신
```

다른 Container는 Service Name으로 찾습니다.

---

# 153. 자주 헷갈리는 개념 6: Network vs Port Mapping

```text
Docker Network
→ Container ↔ Container

Port Mapping
→ Host / External Client ↔ Container
```

---

# 154. 자주 헷갈리는 개념 7: Volume vs Container Filesystem

```text
Container Writable Layer
→ Container Lifecycle에 종속

Volume
→ Container Lifecycle과 분리된 Storage
```

---

# 155. 자주 헷갈리는 개념 8: HTTP vs FastCGI

```text
Browser
   │
   │ HTTP / HTTPS
   ▼
NGINX

NGINX
   │
   │ FastCGI
   ▼
PHP-FPM
```

FastCGI는 Browser가 사용하는 Protocol이 아닙니다.

---

# 156. 자주 헷갈리는 개념 9: WordPress와 PHP-FPM

WordPress는 Application입니다.

PHP-FPM은 PHP Code를 실행하는 Runtime/Process Manager입니다.

```text
WordPress
→ 무엇을 실행할 것인가

PHP-FPM
→ PHP Code를 어떻게 실행할 것인가
```

---

# 157. 자주 헷갈리는 개념 10: MariaDB와 Volume

MariaDB는 Database Server Process입니다.

Volume은 Database File을 저장하는 Persistent Storage입니다.

```text
MariaDB
→ Data를 관리하는 Software

Volume
→ Data File이 살아남는 Storage
```

---

# 158. Inception을 구현할 때 생각하는 순서

무작정 세 Container를 동시에 만들면 Debugging이 어렵습니다.

개념적으로 하나씩 연결하는 것이 좋습니다.

```text
1. MariaDB
   ↓
Database가 정상 실행되는가?

2. WordPress / PHP-FPM
   ↓
Database에 연결되는가?

3. NGINX
   ↓
PHP-FPM에 Request를 전달하는가?

4. TLS
   ↓
HTTPS로 접근되는가?

5. Volume
   ↓
재시작 / 재생성 후 Data가 유지되는가?

6. Compose
   ↓
전체 Application을 한 번에 재현할 수 있는가?
```

---

# 159. MariaDB 먼저

MariaDB는 Application의 가장 아래쪽 Dependency입니다.

```text
NGINX
   ↓
WordPress
   ↓
MariaDB
```

WordPress가 정상 동작하려면 Database가 필요합니다.

따라서 먼저 MariaDB Container가 정상적으로 실행되는지 확인합니다.

---

# 160. 그다음 WordPress

WordPress는 MariaDB에 연결해야 합니다.

질문:

```text
MariaDB Service Name이 맞는가?
Port가 맞는가?
Database가 존재하는가?
User가 존재하는가?
Credential이 맞는가?
```

를 확인합니다.

---

# 161. 그다음 NGINX

WordPress/PHP-FPM이 정상적이라면 NGINX를 연결합니다.

질문:

```text
NGINX가 443을 Listen하는가?
TLS Certificate를 읽을 수 있는가?
WordPress Service Name을 찾는가?
PHP-FPM 9000으로 연결되는가?
```

를 확인합니다.

---

# 162. 마지막으로 외부 Request

모든 내부 Service가 정상이라면 외부 Browser에서 요청합니다.

```text
Browser
   ↓
Host Port
   ↓
NGINX
   ↓
WordPress
   ↓
MariaDB
```

문제가 생기면 어느 구간에서 끊겼는지 찾습니다.

---

# 163. Inception에서 진짜 배우는 것

겉으로 보면 WordPress Website를 Docker로 실행하는 과제입니다.

하지만 실제로 배우는 핵심은 더 넓습니다.

```text
Application Isolation
Container Lifecycle
Image Build
Persistent Storage
Service Networking
Configuration
TLS
Multi-Service Architecture
Infrastructure Reproducibility
```

입니다.

---

# 164. Born2beroot와 Inception의 연결

Born2beroot에서는 **Server 자체**를 배웠습니다.

```text
Hardware / VM
   ↓
Linux
   ↓
User
   ↓
Permission
   ↓
Process
   ↓
Service
   ↓
Network
   ↓
Storage
```

Inception에서는 그 Server 위에서 **Application 실행 환경**을 다룹니다.

```text
Linux
   ↓
Process
   ↓
Isolation
   ↓
Container
   ↓
Application
```

즉 두 프로젝트는 분리된 이야기가 아닙니다.

---

# 165. Born2beroot의 Process가 Inception에서 Container가 된다

Born2beroot:

```text
systemd
  ↓
Service
  ↓
Process
```

Inception:

```text
Docker
  ↓
Container
  ↓
Isolated Process
```

Process 개념이 Container 이해의 기반이 됩니다.

---

# 166. Born2beroot의 Network가 Inception에서 확장된다

Born2beroot:

```text
IP
Port
SSH
Firewall
```

Inception:

```text
Network Namespace
Docker Network
Service DNS
Port Mapping
```

Host Network만 보던 단계에서 Application Network까지 확장됩니다.

---

# 167. Born2beroot의 Storage가 Inception에서 확장된다

Born2beroot:

```text
Disk
Partition
Filesystem
LVM
```

Inception:

```text
Container Writable Layer
Volume
Bind Mount
Persistent Data
```

Storage 자체에서 **Application Data Lifecycle** 문제로 확장됩니다.

---

# 168. Inception 이후에는 어떤 문제가 생길까?

Inception에서는 하나의 Server 안에서 몇 개의 Container를 관리합니다.

```text
One Server

Docker Compose
   │
   ├── NGINX
   ├── WordPress
   └── MariaDB
```

이 정도 규모에서는 Docker Compose가 매우 편리합니다.

그런데 Container가 수십 개, 수백 개라면 어떨까요?

---

# 169. Container가 많아지면

다음과 같은 문제가 생깁니다.

```text
어느 Server에 Container를 배치하지?

Container가 죽으면 누가 다시 실행하지?

Traffic이 늘어나면 Container를 몇 개 더 만들지?

여러 Server 사이 Network는 어떻게 연결하지?

새 Version을 어떻게 배포하지?

원하는 개수의 Container가 계속 살아 있는지 누가 확인하지?
```

Docker 하나만의 문제가 아니라 **Cluster 운영 문제**가 됩니다.

---

# 170. Orchestration

여러 Container의 배치와 Lifecycle을 자동으로 관리하는 것을 Container Orchestration이라고 합니다.

예를 들어 시스템에:

```text
"Web Application을 항상 3개 실행해줘."
```

라고 선언합니다.

Orchestrator는 실제 상태를 확인합니다.

```text
Desired State
3 Containers

Actual State
2 Containers
```

그러면 하나를 추가로 실행할 수 있습니다.

```text
Actual State
3 Containers
```

---

# 171. Kubernetes

대표적인 Container Orchestration Platform이 Kubernetes입니다.

```text
Docker / Container
       ↓
Container가 많아짐
       ↓
Orchestration 필요
       ↓
Kubernetes
```

Kubernetes는 여러 Machine에 걸쳐 Containerized Application을 운영하는 문제를 다룹니다.

---

# 172. Inception of Things로 연결

42의 흐름을 연결하면 아주 자연스럽습니다.

```text
Born2beroot
    ↓
Linux Server 한 대를 이해한다

Inception
    ↓
그 Server 위에서
Containerized Application을 운영한다

Inception of Things
    ↓
여러 Node에서
Containerized Application을
Orchestrate한다
```

즉:

```text
Server
   ↓
Container
   ↓
Cluster
```

로 확장됩니다.

---

# 173. 전체 Concept Map

```text
Application을 운영하고 싶다
        ↓
Linux Server
        ↓
Application 환경이 섞인다
        ↓
Isolation이 필요하다
        ↓
Process
        ↓
Namespace
        ↓
cgroup
        ↓
Container
        ↓
Container를 쉽게 관리하고 싶다
        ↓
Docker
        ↓
실행 환경을 저장하고 싶다
        ↓
Image
        ↓
Image 생성 과정을 기록한다
        ↓
Dockerfile
        ↓
Container File 변경은 임시적이다
        ↓
Persistent Data가 필요하다
        ↓
Volume
        ↓
Container를 여러 개 분리했다
        ↓
서로 통신해야 한다
        ↓
Docker Network
        ↓
Service Name / DNS
        ↓
외부에서도 접근해야 한다
        ↓
Port Mapping
        ↓
Web Request를 받는다
        ↓
NGINX
        ↓
Communication을 보호한다
        ↓
TLS / HTTPS
        ↓
PHP Code를 실행해야 한다
        ↓
PHP-FPM
        ↓
NGINX와 PHP Runtime을 연결한다
        ↓
FastCGI
        ↓
Application
        ↓
WordPress
        ↓
Persistent Business Data
        ↓
MariaDB
        ↓
여러 Container 설정을 한꺼번에 관리한다
        ↓
Docker Compose
        ↓
Container가 너무 많아진다
        ↓
Orchestration
        ↓
Kubernetes
```

---

# 174. 최소 Docker 명령어 정리

이 문서의 목표는 Docker Command를 모두 외우는 것이 아닙니다.

다음 정도를 이해하고 사용할 수 있으면 기본 조작에는 충분합니다.

---

## Container 실행

```bash
docker run <image>
```

예:

```bash
docker run nginx
```

의미:

```text
Image
→ Container 생성
→ Main Process 실행
```

---

## 실행 중 Container 확인

```bash
docker ps
```

모두 확인:

```bash
docker ps -a
```

---

## Logs 확인

```bash
docker logs <container>
```

---

## Container 내부에서 Command 실행

```bash
docker exec -it <container> sh
```

---

## Image 확인

```bash
docker image ls
```

---

## Image Build

```bash
docker build -t <name> .
```

---

## Network 확인

```bash
docker network ls
```

---

## Volume 확인

```bash
docker volume ls
```

---

## Compose 실행

```bash
docker compose up -d
```

---

## Compose 상태

```bash
docker compose ps
```

---

## Compose Logs

```bash
docker compose logs
```

---

## Compose 종료

```bash
docker compose down
```

---

# 175. 명령어를 외우지 말고 질문을 기억하자

```text
"실행 중인가?"
→ docker ps

"왜 안 되지?"
→ docker logs

"안에서는 무슨 일이 일어나지?"
→ docker exec

"Image가 있나?"
→ docker image ls

"Network가 있나?"
→ docker network ls

"Data는 어디에 있지?"
→ docker volume ls

"전체 Application 상태는?"
→ docker compose ps
```

---

# 176. 마지막으로

Inception에서 가장 중요한 것은 Docker Command를 많이 아는 것이 아닙니다.

다음 질문들에 자기 말로 답할 수 있는지가 더 중요합니다.

```text
Container는 VM과 무엇이 다른가?

Container가 왜 Process라고 하는가?

왜 Container 안에서 PID 1이 다시 보일 수 있는가?

Namespace는 무엇을 격리하는가?

cgroup은 Namespace와 무엇이 다른가?

Image와 Container의 차이는 무엇인가?

왜 Image는 Layer 구조를 사용하는가?

왜 Container를 삭제하면 Data가 사라질 수 있는가?

Volume은 어떤 문제를 해결하는가?

Container의 localhost는 왜 Host가 아닌가?

Docker Network는 왜 필요한가?

왜 Container IP 대신 Service Name을 사용하는가?

왜 NGINX만 외부 Port를 열면 되는가?

NGINX는 무엇을 하는가?

PHP-FPM은 왜 필요한가?

FastCGI는 어디에서 사용되는가?

WordPress와 MariaDB는 어떻게 연결되는가?

Dockerfile과 Docker Compose의 차이는 무엇인가?

왜 Docker Compose 다음에 Kubernetes가 등장하는가?
```

이 질문들을 설명할 수 있다면 Inception의 핵심 구조를 이해한 것입니다.

---

# 177. 한 문장으로 정리

> **Born2beroot가 Linux Server를 이해하는 프로젝트라면, Inception은 그 Server 위에서 Application을 Container라는 격리된 실행 환경으로 나누고, Network와 Storage를 통해 다시 하나의 Service로 연결하는 방법을 이해하는 프로젝트다.**

그리고 이 다음 단계에서는:

> **그 Container들을 여러 Server에 걸쳐 자동으로 운영하려면 어떻게 해야 하는가?**

라는 질문을 만나게 됩니다.

그 질문이 Kubernetes와 Inception of Things로 이어집니다.