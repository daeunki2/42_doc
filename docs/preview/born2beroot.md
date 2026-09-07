orn2beroot

From a computer to a Linux server.

Born2beroot는 Virtual Machine 위에 Linux Server를 구축하고 직접 관리하면서 서버와 System Administration의 기본 개념을 학습하는 프로젝트입니다.

이 문서에서는 명령어부터 시작하지 않습니다. 서버란 무엇인가? → 왜 VM이 필요한가? → OS/Linux는 무슨 일을 하는가? → 서버를 어떻게 안전하게 운영하는가?라는 흐름으로 각 기술이 등장한 이유를 먼저 이해합니다.

1. Server

서버가 필요한 이유

우리가 사용하는 많은 프로그램은 혼자서 모든 일을 처리하지 않습니다. 예를 들어 Browser는 인터넷의 모든 Web Page를 가지고 있지 않습니다. 사용자가 페이지를 요청하면 어딘가의 시스템이 요청을 받아 데이터를 돌려줘야 합니다.

Browser (Client)
      │ Request
      ▼
    Server
      │ Response
      ▼
Browser

요청하는 쪽을 Client, 요청을 받아 기능이나 데이터를 제공하는 쪽을 Server라고 합니다.

Server란?

Server는 다른 프로그램이나 컴퓨터의 요청을 받아 Service를 제공하는 컴퓨터 또는 프로그램입니다.

Web Server       → Web Page 제공
Database Server  → Data 저장 / 조회
File Server      → File 제공
SSH Server       → Remote Shell 제공

Server는 반드시 특별한 종류의 거대한 컴퓨터를 뜻하지 않습니다. 일반 Computer도 필요한 프로그램을 실행하고 Network를 통해 Service를 제공하도록 구성하면 Server 역할을 할 수 있습니다.

Server를 운영한다는 것은 단순히 컴퓨터를 켜두는 것이 아닙니다. User와 Permission, Network 접근, Software, Storage, Service 상태 등을 지속적으로 관리해야 합니다. 이러한 작업을 System Administration이라고 합니다.

2. 서버를 어떻게 연습할까?

Linux Server를 공부하려면 직접 관리할 Server 환경이 필요합니다. 별도의 물리 Computer를 준비할 수도 있지만 학습할 때마다 Hardware를 준비하는 것은 비효율적이고, OS나 Network 설정을 실험하다 Host 환경을 망가뜨릴 수도 있습니다.

우리가 원하는 것은 다음과 같습니다.

내 실제 Computer
      │
      ├── 기존 환경 유지
      │
      └── 독립적인 Linux Server 실습 환경

이 문제를 해결할 수 있는 대표적인 방법이 Virtualization입니다.

3. Virtualization

독립적인 System마다 Physical Machine을 하나씩 사용하면 격리는 쉽지만 Hardware가 계속 필요하고, 각 Machine에서 사용하지 않는 CPU와 RAM도 생깁니다.

Physical Machine A → System A
Physical Machine B → System B
Physical Machine C → System C

그래서 이런 질문이 생깁니다.

하나의 물리 Computer를 여러 개의 독립적인 Computer처럼 사용할 수 없을까?

Virtualization은 물리 Computing Resource를 추상화하여 하나의 물리 System 위에 여러 독립적인 실행 환경을 만들 수 있게 합니다.

             Physical Machine
          CPU / RAM / Disk / Network
                     │
                     ▼
               Virtualization
                     │
          ┌──────────┼──────────┐
          ▼          ▼          ▼
        VM 1       VM 2       VM 3

Hypervisor

물리 Hardware를 Virtual Machine에 나누어 제공하고 VM을 관리하는 Software 계층을 Hypervisor라고 합니다.

       VM 1          VM 2
        │             │
        └──────┬──────┘
               ▼
          Hypervisor
               │
               ▼
     Physical Hardware

Born2beroot에서 사용하는 VirtualBox도 이러한 Virtualization 환경을 제공합니다.

4. Virtual Machine

**Virtual Machine(VM)**은 Virtualization을 통해 만들어진 가상의 Computer 환경입니다.

Virtual Machine
├── Virtual CPU
├── Virtual RAM
├── Virtual Disk
├── Virtual Network Interface
└── Operating System

각 VM은 독립적인 Operating System을 실행할 수 있습니다.

Physical Computer
        │
        ▼
    Hypervisor
        │
   ┌────┴────┐
   ▼         ▼
 VM 1       VM 2
   │         │
Debian     Ubuntu

VM은 Isolation, 물리 Resource의 효율적인 사용, 서로 다른 OS 환경 구성, 안전한 Test Environment라는 장점을 줍니다. 반면 각 VM이 Guest OS 전체를 실행하므로 Container에 비해 일반적으로 더 많은 Resource를 사용합니다.

Born2beroot에서는 왜 VM을 사용할까?

Born2beroot의 목적은 Linux Server를 직접 구축하고 관리하는 것입니다.

My Computer
    ↓
VirtualBox
    ↓
Virtual Machine
    ↓
Linux
    ↓
Server Administration

VM 덕분에 Host Computer를 크게 변경하지 않고 독립적인 Linux Server를 만들 수 있습니다.

그런데 Computer만 있다고 Program이 저절로 실행되는 것은 아닙니다. Hardware와 Application 사이에서 System 전체를 관리할 무언가가 필요합니다.

5. Operating System

여러 Program이 CPU, Memory, Disk, Network Device를 동시에 사용하려고 합니다. 각각이 Hardware를 직접 제어한다면 자원 충돌과 보안 문제가 생깁니다.

Applications
     │
     ▼
Operating System
     │
     ▼
Hardware

**Operating System(OS)**은 Hardware Resource를 관리하고 Application이 Computer를 사용할 수 있는 환경과 Interface를 제공합니다.

CPU     → Process 실행 관리
Memory  → Memory 할당과 보호
Disk    → File과 Storage 관리
Device  → Hardware Device 관리
User    → 사용자와 권한 관리
Network → Network Resource 관리

6. Linux와 Debian

엄밀히 말하면 Linux는 Kernel입니다. Kernel은 OS의 핵심 부분으로 CPU, Process, Memory, Device 등 Hardware Resource를 관리합니다.

Applications
     ↓
System Tools / Libraries
     ↓
Linux Kernel
     ↓
Hardware

Linux Kernel에 System Tool, Library, Package Manager 등을 묶어 실제 사용할 수 있는 OS 형태로 제공하는 것을 Linux Distribution이라고 합니다.

Linux
├── Debian
├── Ubuntu
├── Rocky Linux
└── ...

Born2beroot에서는 Linux Distribution을 설치하고 직접 Server 환경으로 구성합니다.

이제 Server와 OS가 준비되었습니다. 다음 문제는 **여러 사용자가 같은 System을 사용한다면 누가 무엇을 할 수 있는가?**입니다.

7. User, Group, Permission

모든 사용자가 모든 File과 System 설정을 자유롭게 수정할 수 있다면 Server를 안전하게 운영할 수 없습니다.

Linux는 User라는 Identity로 사용자를 구분하고, 여러 User를 Group으로 묶어 관리할 수 있습니다.

File에는 대표적으로 다음 Permission이 있습니다.

r → read
w → write
x → execute

권한의 대상은 user / group / others로 나뉩니다.

-rwxr-x---

이를 통해 누가 어떤 Resource를 읽고, 수정하고, 실행할 수 있는가를 통제합니다.

확인:

id
groups
ls -l

8. root와 sudo

System File 수정, Package 설치, User 관리, Network 설정 같은 작업은 System 전체에 영향을 줄 수 있습니다.

Linux의 강력한 관리자 계정이 root입니다. 하지만 모든 작업을 root로 하면 작은 실수도 System 전체에 영향을 줄 수 있습니다.

Normal User
     │
     │ 필요한 순간에만
     ▼
    sudo
     │
     ▼
Privileged Command

sudo는 허가된 User가 필요한 Command에 대해 관리자 권한을 사용하도록 해줍니다.

즉 Born2beroot의 sudo 설정은 단순한 명령어 학습이 아니라 관리자 권한을 어떻게 제한하고 위임할 것인가를 배우는 과정입니다.

9. Authentication과 Password Policy

Permission이 잘 설정되어 있어도 다른 사람이 User의 계정으로 쉽게 로그인할 수 있다면 의미가 없습니다.

Authorization
"이 User는 무엇을 할 수 있는가?"

Authentication
"지금 접속한 사람이 정말 그 User인가?"

Password는 Authentication 방법 중 하나입니다. Password Policy는 길이, 복잡성, 변경 주기 등의 규칙을 적용하여 너무 약한 인증 정보를 사용하는 위험을 줄입니다.

Born2beroot에서는 개별 Password뿐 아니라 System 차원의 인증 정책을 다룹니다.

10. Remote Administration과 SSH

실제 Server는 관리자 바로 옆에 있지 않을 수 있습니다. Data Center나 다른 장소에 있는 Server를 관리하려면 Network를 통해 명령을 실행할 방법이 필요합니다.

**SSH(Secure Shell)**는 Network를 통해 다른 Computer에 안전하게 접속하고 명령을 실행하기 위한 Protocol입니다.

Administrator
     │
     │ SSH
     ▼
  Network
     │
     ▼
Linux Server
     │
     ▼
   Shell

SSH는 원격 관리뿐 아니라 통신 내용을 암호화한다는 점도 중요합니다.

Born2beroot에서는 VM에서 SSH Server를 실행하고 Host에서 VM으로 접속합니다.

ssh -p 4242 user@host
systemctl status ssh

11. IP Address와 Port

Network에서 Server에 접속하려면 먼저 어느 Host인지 알아야 합니다. 이를 식별하는 데 IP Address가 사용됩니다.

하지만 하나의 Server에서는 여러 Network Service가 동시에 실행될 수 있습니다.

One Server
├── SSH
├── Web Server
└── Database

그래서 하나의 Host 안에서 Service를 구분하기 위해 Port를 사용합니다.

IP Address
    │
    ├── Port 22  → SSH의 일반적인 Port
    ├── Port 80  → HTTP
    └── Port 443 → HTTPS

즉 간단히 보면:

IP Address → 어느 Computer?
Port       → 그 Computer의 어느 Network Service?

Born2beroot에서 SSH Port를 설정하면서 이 관계를 직접 확인합니다.

12. Firewall

Server가 Network에 연결되면 외부에서 들어오는 Connection도 생깁니다. 모든 접근을 허용할 필요는 없습니다.

Firewall은 Network Traffic에 규칙을 적용하여 어떤 접근을 허용하고 차단할지 결정합니다.

Network
   │
   ▼
Firewall
   │
   ├── Allowed → Server
   └── Denied  → Block

Born2beroot에서는 필요한 Port만 허용하여 Service는 제공하면서 불필요한 Network 노출은 줄이는 것을 연습합니다.

Debian에서 UFW를 사용한다면 상태를 다음처럼 확인할 수 있습니다.

sudo ufw status

13. Storage, Partition, LVM

Server는 OS, Application, Log, User Data 등을 Disk에 저장합니다.

Storage를 목적에 따라 분리할 수 있지만 고정된 Partition만으로 관리하면 나중에 공간을 조정하기 불편할 수 있습니다.

**LVM(Logical Volume Manager)**은 Physical Storage와 실제 사용하는 Volume 사이에 논리적인 관리 계층을 둡니다.

Physical Disk
     ↓
Physical Volume
     ↓
Volume Group
     │
     ├── Logical Volume A
     ├── Logical Volume B
     └── Logical Volume C

이 구조를 통해 Storage를 더 유연하게 구성하고 확장할 수 있습니다.

확인:

lsblk
sudo pvs
sudo vgs
sudo lvs

Born2beroot에서 LVM을 사용하는 목적은 Partition 이름을 외우는 것이 아니라 Server Storage가 어떻게 구조화되고 관리되는지 이해하는 데 있습니다.

14. Package Manager

Server를 운영하면서 새로운 Software를 설치하고 Update해야 합니다.

Program을 인터넷에서 하나씩 직접 찾아 설치한다면 Version, Dependency, Update, 삭제를 모두 직접 관리해야 합니다.

Package Repository
        ↓
Package Manager
        │
        ├── Install
        ├── Update
        ├── Upgrade
        └── Remove

Debian 계열에서는 APT를 사용합니다.

sudo apt update
sudo apt install <package>

apt update는 설치된 Package를 모두 새 Version으로 바꾸는 명령이 아니라 Repository의 Package 목록 정보를 갱신하는 명령입니다.

15. Process, Daemon, Service, systemd

Program을 실행하면 OS는 실행 중인 작업을 Process로 관리합니다.

Program
   │ execute
   ▼
Process

하지만 SSH Server 같은 Program은 사용자가 Terminal을 열어둔 동안만 실행되어서는 안 됩니다. Background에서 계속 요청을 기다려야 합니다.

Unix/Linux 환경에서 이런 형태로 Background에서 지속적으로 동작하는 Program을 흔히 Daemon이라고 합니다.

그리고 Service를 시작하고 중지하고, Boot 시 자동 실행하고, 상태를 확인하는 관리 체계가 필요합니다. 많은 Linux Distribution에서는 systemd가 이 역할을 담당합니다.

systemd
├── start
├── stop
├── restart
├── enable
└── status

예:

systemctl status ssh
sudo systemctl restart ssh

16. Automation과 cron

Server Administration에는 반복 작업이 많습니다.

Backup
Status Check
Maintenance
Script 실행

사람이 정해진 시간마다 직접 실행하면 비효율적이고 실수하기 쉽습니다.

cron은 정해진 시간이나 주기에 Command 또는 Script를 자동 실행할 수 있게 합니다.

Schedule
   ↓
 cron
   ↓
Command / Script

Born2beroot에서는 Monitoring Script를 주기적으로 실행하면서 기본적인 System Automation을 경험합니다.

17. Monitoring

Server가 켜져 있다는 것과 Server가 정상적으로 동작한다는 것은 다릅니다.

관리자는 CPU, Memory, Disk, Process, User, Network 등 System 상태를 파악할 수 있어야 합니다.

Server
├── CPU
├── Memory
├── Disk
├── Process
├── Network
└── Users
     │
     ▼
 Monitoring
     │
     ▼
Administrator

Born2beroot에서는 monitoring.sh 같은 Script로 여러 System 정보를 수집하고 주기적으로 출력하면서 Monitoring + Shell Script + Automation이 어떻게 연결되는지 경험합니다.

18. Born2beroot의 전체 그림

각 기술은 서로 떨어져 있는 것이 아닙니다.

Service를 제공하는 Computer가 필요하다
                ↓
              Server
                ↓
독립적인 실습 환경이 필요하다
                ↓
       Virtualization / VM
                ↓
Hardware와 Program을 관리해야 한다
                ↓
       Operating System
                ↓
          Linux / Debian
                ↓
사용자와 Resource 접근을 구분해야 한다
                ↓
    User / Group / Permission
                ↓
관리자 권한을 통제해야 한다
                ↓
          root / sudo
                ↓
사용자의 신원을 확인해야 한다
                ↓
        Password Policy
                ↓
원격으로 Server를 관리해야 한다
                ↓
               SSH
                ↓
Network에서 Host와 Service를 찾아야 한다
                ↓
        IP Address / Port
                ↓
Network 접근을 통제해야 한다
                ↓
            Firewall
                ↓
Storage를 유연하게 관리해야 한다
                ↓
         Partition / LVM
                ↓
Software를 체계적으로 관리해야 한다
                ↓
        Package Manager
                ↓
Server Program을 지속적으로 운영해야 한다
                ↓
   Process / Service / systemd
                ↓
반복 작업을 자동화해야 한다
                ↓
              cron
                ↓
Server 상태를 파악해야 한다
                ↓
           Monitoring

결국 Born2beroot가 던지는 큰 질문은 하나입니다.

Linux Server 한 대를 안전하고 지속적으로 운영하려면 무엇이 필요한가?

19. 다음 단계: Inception

Born2beroot에서는 Linux Server 한 대를 구성하고 관리하는 기본을 배웠습니다.

그런데 하나의 Server에 여러 Application을 직접 설치하기 시작하면 새로운 문제가 생깁니다.

Linux Server
├── Web Server
├── WordPress
├── Database
└── Other Applications

각 Application은 서로 다른 Version, Library, Configuration을 요구할 수 있고 서로의 환경에 영향을 줄 수 있습니다.

그래서 다음 질문으로 이어집니다.

여러 Application을 하나의 Server에서 서로 분리된 환경으로 실행할 수 없을까?

Born2beroot
Linux Server Administration
        ↓
"Application 환경을 어떻게 분리하지?"
        ↓
Inception
        ↓
Docker / Container

이 질문이 다음 프로젝트인 Inception으로 이어집니다.