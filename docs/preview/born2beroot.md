# Born2beroot

## Overview

Born2beroot는 **가상 머신(Virtual Machine)에 Linux 서버를 구축하고 관리하는 프로젝트**입니다.

이 프로젝트를 통해 가상화의 기본 개념부터 Linux 시스템의 사용자와 권한, 원격 접속, 네트워크 보안, 서비스 관리와 모니터링까지 서버 관리의 기초를 경험합니다.

이후 Inception과 Inception of Things에서 Docker와 Kubernetes를 학습하기 위한 가장 기본적인 인프라 계층에 해당합니다.

---

## 1. Virtual Machine

Virtual Machine(VM)은 물리 컴퓨터의 CPU, 메모리, 저장공간 등의 자원을 **가상화하여 독립된 컴퓨터처럼 사용할 수 있도록 만든 환경**입니다.

```text
Physical Machine
       │
       ▼
   Hypervisor
       │
       ▼
 Virtual Machine
       │
       ▼
     Debian
```

VM은 자신만의 Guest OS와 Kernel을 가질 수 있습니다.

이를 통해 하나의 물리 컴퓨터에서 서로 다른 운영체제를 실행하거나, Host 시스템과 격리된 환경에서 서버를 구축하고 테스트할 수 있습니다.

---

## 2. Linux와 Debian

Linux는 엄밀히 말하면 운영체제 전체가 아니라 **Kernel**입니다.

Kernel은 애플리케이션과 하드웨어 사이에서 CPU, 메모리, 디스크, 네트워크 등의 시스템 자원을 관리합니다.

```text
Application
     │
     ▼
Linux Kernel
     │
     ▼
  Hardware
```

Debian은 Linux Kernel에 Shell, 시스템 도구, 라이브러리, 패키지 관리자 등을 결합하여 실제 운영체제로 사용할 수 있도록 만든 **Linux Distribution(배포판)**입니다.

---

## 3. User, Group과 Permission

Linux는 여러 사용자가 하나의 시스템을 사용할 수 있는 **Multi-user System**입니다.

파일과 디렉터리의 접근 권한은 기본적으로 다음 세 범주로 구분됩니다.

```text
Owner
Group
Others
```

### 기본 명령어

현재 사용자 확인:

```bash
whoami
```

사용자의 UID, GID와 Group 확인:

```bash
id
```

시스템의 사용자 목록 확인:

```bash
cat /etc/passwd
```

특정 사용자가 속한 Group 확인:

```bash
groups username
```

사용자 생성:

```bash
sudo adduser username
```

Group 생성:

```bash
sudo groupadd groupname
```

사용자를 Group에 추가:

```bash
sudo usermod -aG groupname username
```

파일 권한 확인:

```bash
ls -l
```

파일 권한 변경:

```bash
chmod 755 file
```

핵심 원칙은 **필요한 사용자에게 필요한 권한만 부여하는 것**입니다.

---

## 4. root와 sudo

`root`는 Linux 시스템에서 매우 강력한 관리 권한을 가진 특별한 사용자입니다.

일반 사용자는 평소에는 제한된 권한으로 작업하고, 관리 권한이 필요한 경우 `sudo`를 사용합니다.

```text
Normal User
     │
     │ sudo
     ▼
Privileged Command
```

예를 들어 일반 사용자는 패키지를 설치할 권한이 없을 수 있습니다.

```bash
apt install nginx
```

이때 허용된 사용자는:

```bash
sudo apt install nginx
```

처럼 관리자 권한으로 해당 명령만 실행할 수 있습니다.

sudo 권한 확인:

```bash
sudo -l
```

`sudo`는 사용자가 아니라 **허용된 사용자가 특정 명령을 다른 사용자, 일반적으로 root의 권한으로 실행할 수 있도록 하는 도구**입니다.

---

## 5. SSH

SSH(Secure Shell)는 **네트워크를 통해 다른 컴퓨터의 Shell에 안전하게 접속하기 위한 프로토콜**입니다.

```text
Local Computer
      │
      │ SSH
      ▼
 Linux Server
```

### SSH 접속

기본 형태:

```bash
ssh username@server_ip
```

예:

```bash
ssh daeun@192.168.1.10
```

SSH의 기본 Port는 `22`입니다.

다른 Port를 사용한다면 `-p` 옵션을 사용합니다.

```bash
ssh -p 4242 daeun@192.168.1.10
```

Born2beroot에서는 SSH를 기본 Port 22가 아닌 **4242 Port**에서 실행하도록 설정합니다.

### SSH 서비스 확인

서버에서 SSH 서비스 상태 확인:

```bash
sudo systemctl status ssh
```

시작:

```bash
sudo systemctl start ssh
```

재시작:

```bash
sudo systemctl restart ssh
```

### SSH 설정

SSH 서버의 주요 설정 파일은:

```text
/etc/ssh/sshd_config
```

입니다.

예를 들어 설정 파일에서 SSH Port를 지정할 수 있습니다.

```text
Port 4242
```

설정을 변경했다면 SSH 서비스를 재시작해야 적용됩니다.

```bash
sudo systemctl restart ssh
```

### 연결 구조

```text
My Computer
     │
     │ ssh -p 4242 user@IP
     │
     ▼
   Port 4242
     │
     ▼
 SSH Server (sshd)
     │
     ▼
 Linux Shell
```

즉 SSH는 단순히 "다른 컴퓨터에 접속한다"가 아니라, **Client가 네트워크를 통해 서버에서 실행 중인 SSH Service에 연결하는 것**입니다.

---

## 6. Port

하나의 서버에서는 여러 네트워크 서비스가 동시에 실행될 수 있습니다.

Port는 **하나의 컴퓨터 안에서 어떤 네트워크 서비스와 통신할 것인지 구분하기 위한 번호**입니다.

```text
Server
│
├── 22   → SSH (default)
├── 80   → HTTP
├── 443  → HTTPS
└── 4242 → Born2beroot SSH
```

현재 열려 있거나 Listening 중인 Port를 확인할 때는 다음과 같은 명령을 사용할 수 있습니다.

```bash
ss -tuln
```

프로세스 정보까지 함께 확인하려면:

```bash
sudo ss -tulpn
```

---

## 7. Firewall

Firewall은 **네트워크 트래픽을 규칙에 따라 허용하거나 차단하는 시스템**입니다.

Born2beroot에서는 UFW(Uncomplicated Firewall)를 사용합니다.

```text
Network
   │
   ▼
  UFW
   │
   ├── Port 4242 → ALLOW
   │
   └── Others    → BLOCK
   │
   ▼
Server
```

### Firewall 상태 확인

```bash
sudo ufw status
```

좀 더 자세히 확인:

```bash
sudo ufw status verbose
```

### Firewall 활성화

```bash
sudo ufw enable
```

비활성화:

```bash
sudo ufw disable
```

### Port 허용

예를 들어 SSH가 사용하는 4242 Port를 허용하려면:

```bash
sudo ufw allow 4242
```

TCP만 명시적으로 허용하려면:

```bash
sudo ufw allow 4242/tcp
```

규칙 목록을 번호와 함께 확인:

```bash
sudo ufw status numbered
```

특정 규칙 삭제:

```bash
sudo ufw delete <rule-number>
```

### SSH와 Firewall의 관계

SSH 서버가 4242에서 정상적으로 실행되고 있어도 Firewall이 해당 Port를 막고 있다면 외부에서 접속할 수 없습니다.

```text
SSH Server
Listening :4242
       ▲
       │
     Firewall
       │
       X
     Client
```

따라서 서버를 외부에서 사용하려면 두 조건이 모두 필요합니다.

```text
Service가 Port에서 실행 중
            +
Firewall이 해당 Port 허용
            ↓
       접속 가능
```

이 관계는 이후 Inception에서 Docker의 Port와 Network를 공부할 때도 계속 등장합니다.

---

## 8. LVM

LVM(Logical Volume Manager)은 물리적인 저장장치와 실제로 사용하는 저장공간 사이에 **논리적인 관리 계층**을 추가합니다.

```text
Physical Disk
      │
      ▼
Physical Volume
      │
      ▼
Volume Group
      │
      ├── Logical Volume
      └── Logical Volume
```

LVM 상태를 간단히 확인할 수 있는 명령어:

```bash
lsblk
```

Physical Volume:

```bash
sudo pvs
```

Volume Group:

```bash
sudo vgs
```

Logical Volume:

```bash
sudo lvs
```

---

## 9. Package Manager

Debian에서는 `apt`를 이용해 패키지를 관리합니다.

패키지 목록 업데이트:

```bash
sudo apt update
```

패키지 설치:

```bash
sudo apt install <package>
```

설치된 패키지 업데이트:

```bash
sudo apt upgrade
```

패키지 삭제:

```bash
sudo apt remove <package>
```

---

## 10. Service와 Daemon

서버에는 백그라운드에서 지속적으로 실행되는 프로그램들이 있습니다.

Debian에서는 `systemd`가 많은 시스템 서비스를 관리합니다.

```text
systemd
   │
   ├── ssh.service
   ├── cron.service
   └── ...
```

서비스 상태 확인:

```bash
systemctl status <service>
```

시작:

```bash
sudo systemctl start <service>
```

중지:

```bash
sudo systemctl stop <service>
```

재시작:

```bash
sudo systemctl restart <service>
```

부팅할 때 자동으로 시작하도록 설정:

```bash
sudo systemctl enable <service>
```

---

## 11. cron과 Monitoring

`cron`은 특정 명령이나 스크립트를 **정해진 시간 또는 주기에 자동으로 실행**할 수 있도록 해주는 도구입니다.

현재 사용자의 cron 설정:

```bash
crontab -l
```

cron 설정 수정:

```bash
crontab -e
```

root의 crontab을 수정하려면:

```bash
sudo crontab -e
```

예를 들어 일정 주기로 monitoring script를 실행하는 구조입니다.

```text
cron
 │
 │ 일정 주기
 ▼
monitoring.sh
 │
 ▼
System Information
```

Monitoring의 핵심은 특정 명령어를 암기하는 것이 아니라 **서버가 정상적으로 동작하고 있는지 지속적으로 관찰하는 것**입니다.

---

## What I Learned

Born2beroot에서는 **하나의 Linux 서버를 구축하고 관리하는 기본 방법**을 학습합니다.

```text
Virtual Machine
      │
      ▼
    Linux
      │
      ├── User / Permission
      ├── Service
      ├── SSH
      ├── Port
      ├── Firewall
      ├── Storage
      └── Monitoring
```

특히 중요한 연결 관계는 다음과 같습니다.

```text
User
 ↓
Permission
 ↓
sudo / root


Client
 ↓
Network
 ↓
Firewall
 ↓
Port
 ↓
Service


Server
 ↓
Monitoring
```

다음 프로젝트인 Inception에서는 하나의 서버 위에 여러 서비스를 직접 설치하는 대신, **각 서비스를 Container라는 독립적인 실행 환경으로 분리하여 관리하는 방법**을 학습합니다.

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
```
