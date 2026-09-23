# 과제 파악하기

## 1. 이 과제는 무엇을 배우는 과제인가?

Inception of Things는 **Kubernetes를 처음 경험하기 위한 System Administration 프로젝트**입니다.

과제는 Vagrant를 이용한 Virtual Machine 구성에서 시작하여 K3s를 이용한 Kubernetes Cluster 구성, Application 배포, 그리고 K3d와 Argo CD를 이용한 자동 배포까지 단계적으로 진행됩니다.

원본 과제는 이 프로젝트를 **Kubernetes에 대한 최소한의 입문(minimal introduction)**이라고 설명합니다.

따라서 이 과제의 목표는 Kubernetes의 모든 기능을 배우는 것이 아닙니다.

전체적인 학습 흐름을 먼저 이해하는 것이 중요합니다.

```text
Virtual Machine
      │
      ▼
   Vagrant
      │
      ▼
     K3s
      │
      ▼
 Kubernetes
      │
      ▼
Application Deployment
      │
      ▼
     K3d
      │
      ▼
   Argo CD
      │
      ▼
    GitOps
```

과제는 크게 세 단계로 구성됩니다.

```text
Part 1
K3s + Vagrant
→ Kubernetes Cluster 만들기

Part 2
K3s + Applications
→ Kubernetes에 Application 배포하기

Part 3
K3d + Argo CD
→ Git을 기준으로 Application 배포 자동화하기

Bonus
GitLab
→ Git 저장소까지 Local Infrastructure에 포함하기
```

---

# 2. 먼저 알아야 할 것

이 프로젝트를 시작하기 전에 모든 개념을 완벽하게 공부할 필요는 없습니다.

다만 다음 흐름은 알고 시작하는 것이 좋습니다.

## 우리가 이미 배운 것

Born2beroot와 Inception에서 배운 개념들이 그대로 이어집니다.

```text
Born2beroot

Virtual Machine
Linux
IP / Port
SSH
Service
Network
      │
      ▼
Inception

Docker
Image
Container
Network
Volume
Docker Compose
      │
      ▼
Inception of Things

Vagrant
Kubernetes
K3s / K3d
Pod
Deployment
Service
Ingress
Namespace
Argo CD
GitOps
```

즉 완전히 새로운 내용을 처음부터 배우는 것이 아니라,

> **VM과 Container에 대한 지식을 바탕으로 여러 Container를 관리하는 방법을 배우는 과정**

이라고 생각할 수 있습니다.

---

# 3. Kubernetes는 왜 필요한가?

Inception에서는 Docker Compose를 이용하여 여러 Container를 구성했습니다.

예를 들어:

```text
Docker Compose

├── NGINX
├── WordPress
└── MariaDB
```

작은 환경에서는 이런 방식으로도 충분합니다.

하지만 Container와 Server가 많아지면 새로운 문제가 발생합니다.

```text
Container가 죽으면?

Container를 3개 실행하고 싶다면?

3개 중 하나가 죽으면?

여러 Server가 있다면
어느 Server에 Container를 실행해야 할까?

Application을 업데이트하려면?

Container의 IP가 계속 바뀐다면
다른 Application은 어떻게 찾을까?
```

이러한 문제를 관리하는 시스템을 **Container Orchestrator**라고 합니다.

Kubernetes가 대표적인 Container Orchestrator입니다.

이 과제에서는 Kubernetes가 이러한 문제를 어떻게 해결하는지 아주 작은 환경에서 직접 경험하게 됩니다.

---

# 4. 이 과제에서 가장 중요한 생각

Kubernetes를 공부할 때 가장 먼저 이해해야 할 개념 중 하나는 **Desired State**입니다.

Docker를 직접 사용할 때는 보통 명령을 내립니다.

```text
이 Container를 실행해.
```

Kubernetes에서는 조금 다르게 생각합니다.

```text
이 Application의 Pod가
항상 3개 존재해야 한다.
```

우리는 원하는 상태를 선언하고 Kubernetes는 실제 상태를 그 상태에 맞추려고 합니다.

```text
Desired State

Pod = 3
      │
      ▼
Kubernetes
      │
      ▼
Actual State

Pod
Pod
Pod
```

만약 하나가 죽으면:

```text
Desired: 3

Actual: 2

      ↓

Kubernetes

      ↓

새로운 Pod 생성

      ↓

Actual: 3
```

이 **"원하는 상태를 선언하고 시스템이 그 상태를 유지한다"**는 생각은 이후 Argo CD와 GitOps를 이해하는 데도 중요합니다.

---

# 5. Part 1 — K3s and Vagrant

## 목표

Part 1에서는 **두 개의 Virtual Machine을 만들고 K3s Cluster를 구성합니다.**

```text
              Kubernetes Cluster

        ┌────────────────────┐
        │                    │
        ▼                    ▼

   VM 1                    VM 2
192.168.56.110          192.168.56.111

 K3s Server              K3s Agent
 Controller               Worker
```

VM은 직접 하나씩 만드는 것이 아니라 **Vagrant**를 이용하여 생성합니다.

---

## 과제가 요구하는 것

VM 두 대가 필요합니다.

첫 번째:

```text
hostname
→ <login>S

IP
→ 192.168.56.110

K3s
→ Server / Controller
```

두 번째:

```text
hostname
→ <login>SW

IP
→ 192.168.56.111

K3s
→ Agent / Worker
```

두 머신 모두 password 없이 SSH 접속할 수 있어야 합니다.

또한 `kubectl`을 사용할 수 있어야 합니다.

---

## 여기서 배워야 할 개념

### Vagrant

Virtual Machine 환경을 **코드로 정의하고 자동으로 생성하기 위한 도구**입니다.

Born2beroot에서는 VM을 직접 만들었다면:

```text
Born2beroot

VM 설정
→ OS 설치
→ Network 설정
→ 직접 관리
```

Vagrant에서는:

```text
Vagrantfile
      │
      ▼
vagrant up
      │
      ▼
VM 생성
      │
      ├── CPU
      ├── RAM
      ├── Network
      ├── Hostname
      └── Provisioning
```

이라는 방식으로 VM 환경을 재현할 수 있습니다.

### Provisioning

VM을 만든 뒤 필요한 프로그램 설치와 설정까지 자동으로 수행하는 과정입니다.

```text
VM 생성
   │
   ▼
Provisioning Script
   │
   ├── K3s 설치
   ├── 설정
   └── Cluster 연결
```

### Kubernetes Cluster

여러 머신을 하나의 Kubernetes 환경으로 묶은 것입니다.

### Node

Kubernetes Cluster에 참여하는 하나의 Machine입니다.

이 과제에서는:

```text
Cluster
│
├── Server Node
│
└── Agent Node
```

형태가 됩니다.

### K3s

K3s는 Kubernetes를 가볍게 구성할 수 있도록 만든 Kubernetes Distribution입니다.

Part 1에서는 K3s를 이용하여 작은 Kubernetes Cluster를 직접 구성합니다.

### kubectl

Kubernetes Cluster에 명령을 보내고 상태를 확인하기 위한 CLI입니다.

예:

```bash
kubectl get nodes
```

```bash
kubectl get pods
```

---

## Part 1에서 스스로 답할 수 있어야 하는 질문

Part 1이 끝났다면 최소한 다음 질문에 답할 수 있어야 합니다.

```text
Vagrant는 왜 사용하는가?

Vagrantfile은 무엇인가?

Provisioning은 무엇인가?

Kubernetes Cluster란 무엇인가?

Node란 무엇인가?

Server와 Agent의 차이는 무엇인가?

K3s는 Kubernetes와 어떤 관계인가?

두 VM은 어떻게 하나의 Cluster가 되는가?

kubectl은 누구에게 명령을 보내는가?
```

---

# 6. Part 2 — K3s and Applications

Part 1에서는 Kubernetes가 실행될 **환경**을 만들었습니다.

Part 2에서는 처음으로 그 Kubernetes 위에 **Application을 배포합니다.**

## 목표

이번에는 VM 하나만 사용합니다.

```text
VM
│
│ K3s Server
│
└── Kubernetes
      │
      ├── App 1
      ├── App 2 × 3
      └── App 3
```

세 개의 Web Application을 실행해야 합니다.

요청의 `HOST`에 따라 서로 다른 Application이 보여야 합니다.

```text
                  192.168.56.110
                         │
                      Ingress
                         │
          ┌──────────────┼──────────────┐
          │              │              │
       app1.com        app2.com       default
          │              │              │
          ▼              ▼              ▼
        App 1           App 2          App 3
                        × 3
```

특히 **App 2는 3개의 Replica**를 가져야 합니다.

---

# 7. 여기서 Kubernetes의 핵심 개념을 배운다

Part 2가 이 과제에서 Kubernetes 자체를 이해하는 가장 중요한 부분입니다.

## Pod

Container가 Kubernetes에서 실행되는 기본 단위입니다.

처음에는 다음 정도로 이해하면 충분합니다.

```text
Docker

Container

    ↓ Kubernetes

Pod
└── Container
```

Pod 안에는 하나 이상의 Container가 들어갈 수 있습니다.

---

## Deployment

Application의 Pod를 어떻게 실행하고 유지할지 선언합니다.

예를 들어:

```text
App2

replicas: 3
```

라고 선언하면:

```text
Deployment
    │
    ├── Pod
    ├── Pod
    └── Pod
```

형태로 유지됩니다.

---

## Replica

같은 Application의 Pod를 여러 개 실행하는 것입니다.

Part 2의 App2가 바로 이를 경험하기 위한 요구사항입니다.

```text
App2

├── Pod
├── Pod
└── Pod
```

---

## Service

Pod는 생성되고 삭제될 수 있습니다.

따라서 다른 시스템이 특정 Pod 하나의 IP에 직접 의존하면 관리하기 어렵습니다.

Service는 여러 Pod 앞에 **안정적인 접근 지점**을 제공합니다.

```text
Service
   │
   ├── Pod
   ├── Pod
   └── Pod
```

---

## Ingress

외부에서 들어온 HTTP 요청을 규칙에 따라 적절한 Service로 전달합니다.

이 과제에서는 `HOST`를 이용합니다.

```text
app1.com
    │
    ▼
Ingress
    │
    ▼
App1 Service
```

```text
app2.com
    │
    ▼
Ingress
    │
    ▼
App2 Service
```

그 외 요청은 App3로 보내야 합니다.

---

## Kubernetes YAML

Kubernetes에서는 원하는 리소스 상태를 YAML 파일로 선언하는 방식을 많이 사용합니다.

개념적으로:

```yaml
kind: Deployment

spec:
  replicas: 3
```

처럼:

> "Pod를 세 개 만들어라"

라고 명령을 반복해서 내리는 것이 아니라,

> "이 Application은 Pod가 세 개 존재하는 상태여야 한다"

라고 원하는 상태를 선언합니다.

---

## Part 2에서 반드시 연결해야 하는 구조

각 개념을 따로 외우기보다 다음 흐름을 이해하는 것이 중요합니다.

```text
External Request
       │
       ▼
    Ingress
       │
       ▼
    Service
       │
       ▼
  Deployment
       │
       ▼
      Pod
       │
       ▼
   Container
```

---

## Part 2가 끝났을 때 답할 수 있어야 하는 질문

```text
Pod와 Container의 관계는?

Deployment는 왜 필요한가?

Replica는 왜 만드는가?

Pod 하나가 죽으면 어떻게 되는가?

Service는 왜 필요한가?

Ingress와 Service는 무엇이 다른가?

HOST에 따라 다른 Application을
어떻게 보여주는가?

왜 App2를 3개 실행하도록 했을까?
```

---

# 8. Part 3 — K3d and Argo CD

Part 2까지는 우리가 직접 Kubernetes에 Application을 배포했습니다.

Part 3에서는 여기서 한 단계 더 나아갑니다.

## 목표

이번에는 Vagrant를 사용하지 않습니다.

VM 안에 Docker와 K3d 등을 설치하고 Kubernetes 환경을 구성합니다.

그리고 Argo CD를 이용합니다.

전체 구조는 다음과 같습니다.

```text
Developer
    │
    │ git push
    ▼
Public GitHub Repository
    │
    │ Desired State
    ▼
  Argo CD
    │
    │ Sync
    ▼
 Kubernetes
    │
    ▼
Application
```

---

# 9. K3s와 K3d

과제에서는 **K3s와 K3d의 차이를 이해해야 한다고 명시적으로 요구합니다.**

Part 1과 Part 2에서는 K3s를 사용했습니다.

Part 3에서는 K3d를 사용합니다.

학습할 때 다음 질문을 중심으로 비교해야 합니다.

```text
K3s란 무엇인가?

K3d란 무엇인가?

왜 K3d를 사용하려면 Docker가 필요한가?

Part 1에서는 왜 VM 안에 K3s를 설치했는가?

Part 3에서는 Cluster가 어디에서 실행되는가?
```

---

# 10. Namespace

Part 3에서는 두 개의 Namespace를 만들어야 합니다.

```text
Kubernetes Cluster
│
├── namespace: argocd
│      │
│      └── Argo CD
│
└── namespace: dev
       │
       └── Application
```

Namespace는 하나의 Kubernetes Cluster 안에서 리소스를 논리적으로 구분하는 방법입니다.

---

# 11. Argo CD

Argo CD는 Git Repository에 선언된 Kubernetes 설정을 기준으로 Cluster의 상태를 관리합니다.

예를 들어 GitHub에:

```text
deployment.yaml

image: playground:v1
```

이 있다고 가정합니다.

Cluster도:

```text
playground:v1
```

을 실행하고 있습니다.

이 상태는 서로 일치합니다.

```text
Git        Cluster

v1    ==    v1

      Synced
```

Git을:

```text
v1 → v2
```

로 변경하면:

```text
Git        Cluster

v2    !=    v1
```

차이가 발생합니다.

Argo CD가 이를 동기화하면:

```text
Git        Cluster

v2    ==    v2

      Synced
```

가 됩니다.

과제에서는 실제로 Git Repository에서 Application 버전을 변경하고 Application이 업데이트되는 것을 확인해야 합니다.

---

# 12. GitOps

Part 3에서 배우게 되는 중요한 운영 방식입니다.

GitOps에서는 Git Repository를 시스템의 원하는 상태를 기록하는 **Source of Truth**로 사용합니다.

기존 방식:

```text
Developer
    │
kubectl 명령
    │
    ▼
Kubernetes
```

GitOps:

```text
Developer
    │
 git push
    ▼
   Git
    │
    ▼
 Argo CD
    │
    ▼
Kubernetes
```

따라서:

```text
Git에 기록된 상태
        =
우리가 원하는 상태
```

가 됩니다.

이 개념은 앞에서 배운 Kubernetes의 **Desired State**와 연결됩니다.

---

# 13. Part 3에서 배워야 할 개념

```text
K3d
│
├── K3s와의 차이
├── Docker와의 관계
└── Cluster 생성

Namespace
│
└── Resource 분리

Argo CD
│
├── Git Repository 감시
├── Desired State 비교
└── Synchronization

GitOps
│
├── Git = Source of Truth
└── Declarative Deployment

Image Tag
│
├── v1
└── v2
```

---

# 14. Part 3이 끝났을 때 답할 수 있어야 하는 질문

```text
K3s와 K3d의 차이는?

왜 K3d에 Docker가 필요한가?

Namespace는 왜 사용하는가?

Argo CD는 무엇을 하는가?

Argo CD는 무엇과 무엇을 비교하는가?

Git이 Source of Truth라는 것은 무슨 뜻인가?

Git에서 v1 → v2로 변경하면
어떤 과정을 거쳐 Application이 바뀌는가?

GitOps와 직접 kubectl을 사용하는 방식은
무엇이 다른가?
```

---

# 15. Bonus — GitLab

Bonus에서는 Part 3에서 사용했던 GitHub를 **Local GitLab**으로 대체합니다.

Part 3:

```text
GitHub
   │
   ▼
Argo CD
   │
   ▼
Kubernetes
```

Bonus:

```text
Local GitLab
      │
      ▼
   Argo CD
      │
      ▼
 Kubernetes
```

GitLab 자체도 Local에서 실행되어야 하며:

```text
namespace: gitlab
```

이라는 전용 Namespace를 만들어야 합니다.

Part 3에서 구현했던 모든 기능이 Local GitLab을 이용해서도 동작해야 합니다.

Bonus에서는 필요하다면 Helm 등의 도구를 사용할 수 있습니다.

> Bonus는 Mandatory가 완벽하게 동작하는 경우에만 평가됩니다.

---

# 16. 과제 전체에서 배워야 할 개념 지도

처음부터 모든 것을 공부하려고 하면 Kubernetes 용어가 한꺼번에 등장하기 때문에 혼란스러울 수 있습니다.

다음 순서로 공부하는 것이 좋습니다.

```text
[이미 알고 있어야 할 것]

Linux
 │
 ├── Process
 ├── Service
 ├── IP / Port
 ├── SSH
 └── Network
       │
       ▼
Docker
 │
 ├── Image
 ├── Container
 └── Network

────────────────────────────

[Part 1]

Virtual Machine
       │
       ▼
    Vagrant
       │
       ▼
  Provisioning
       │
       ▼
      K3s
       │
       ▼
Kubernetes Cluster
       │
       ├── Server
       └── Agent
       │
       ▼
     kubectl

────────────────────────────

[Part 2]

    Container
        │
        ▼
       Pod
        │
        ▼
   Deployment
        │
        ▼
     Replica
        │
        ▼
     Service
        │
        ▼
     Ingress

────────────────────────────

[Part 3]

      Docker
        │
        ▼
       K3d
        │
        ▼
 Kubernetes Cluster
        │
        ├── Namespace
        │
        ▼
     Argo CD
        │
        ▼
       Git
        │
        ▼
      GitOps

────────────────────────────

[Bonus]

     GitLab
        │
        ▼
Local Git Repository
        │
        ▼
     Argo CD
        │
        ▼
    Kubernetes
```

---

# 17. 과제의 진짜 학습 흐름

이 프로젝트를 단순히:

```text
Vagrant 설치
K3s 설치
YAML 작성
Argo CD 설치
```

라는 설치 과제로 접근하면 각각의 기술이 왜 필요한지 이해하기 어렵습니다.

대신 각 Part가 **하나의 질문에 답하는 과정**이라고 생각하는 것이 좋습니다.

### Part 1

> Kubernetes가 동작할 환경은 어떻게 만드는가?

```text
Vagrant
→ VM
→ K3s
→ Cluster
```

### Part 2

> Kubernetes는 Application을 어떻게 실행하고 외부에 제공하는가?

```text
Deployment
→ Pod
→ Service
→ Ingress
```

### Part 3

> Application의 배포 상태를 Git을 기준으로 자동 관리할 수 있는가?

```text
Git
→ Argo CD
→ Kubernetes
```

### Bonus

> Git Repository까지 직접 운영한다면?

```text
GitLab
→ Argo CD
→ Kubernetes
```

---

# 18. 구현 전에 세울 학습 목표

이 프로젝트를 완료했을 때 다음 문장을 자신의 말로 설명할 수 있다면 과제의 핵심을 이해했다고 볼 수 있습니다.

### Infrastructure

> Virtual Machine과 Container의 차이를 설명할 수 있다.

> Vagrant가 VM 환경을 어떻게 재현하는지 설명할 수 있다.

### Kubernetes

> Kubernetes가 왜 필요한지 설명할 수 있다.

> Cluster와 Node의 관계를 설명할 수 있다.

> Pod, Deployment, Service, Ingress의 역할과 연결 관계를 설명할 수 있다.

### Deployment

> Kubernetes의 Desired State 개념을 설명할 수 있다.

> Replica를 선언했을 때 Kubernetes가 무엇을 하는지 설명할 수 있다.

### GitOps

> K3s와 K3d의 차이를 설명할 수 있다.

> Argo CD가 무엇을 비교하고 동기화하는지 설명할 수 있다.

> Git을 Source of Truth로 사용한다는 의미를 설명할 수 있다.

> Git에서 Application의 Image Tag를 변경했을 때 실제 Application이 업데이트되는 흐름을 설명할 수 있다.

---

# 19. 추천 학습 순서

처음부터 Kubernetes 전체를 공부할 필요는 없습니다.

과제의 진행 순서에 맞춰 필요한 만큼 공부합니다.

```text
1. 과제 전체 구조 파악

        ↓

2. Vagrant 이해

        ↓

3. Part 1 구현
   VM → K3s Cluster

        ↓

4. Kubernetes 기본 구조 이해
   Cluster / Node / Pod

        ↓

5. Part 2 개념
   Deployment / Service / Ingress

        ↓

6. Part 2 구현

        ↓

7. K3s vs K3d

        ↓

8. Argo CD / GitOps 이해

        ↓

9. Part 3 구현

        ↓

10. 전체 구조 다시 연결

        ↓

11. Mandatory 완성

        ↓

12. Bonus GitLab
```

즉,

> **필요한 개념을 먼저 이해하고 → 바로 구현하고 → 실제 동작을 확인하고 → 다시 개념과 연결한다.**

이 과정을 반복합니다.

---

# 20. 최종적으로 만들어지는 것

Mandatory를 모두 완료하면 전체적으로 다음과 같은 경험을 하게 됩니다.

```text
Part 1

Vagrant
   │
   ├── VM : Server
   └── VM : Worker
          │
          ▼
     K3s Cluster


Part 2

Client
   │
   ▼
Ingress
   │
   ▼
Service
   │
   ▼
Deployment
   │
   ▼
Pods


Part 3

Developer
   │
git push
   ▼
GitHub
   │
   ▼
Argo CD
   │
   ▼
K3d / Kubernetes
   │
   ▼
Application


Bonus

Developer
   │
git push
   ▼
Local GitLab
   │
   ▼
Argo CD
   │
   ▼
Kubernetes
   │
   ▼
Application
```

---

# 21. 제출 구조

Mandatory는 Repository Root에 다음 세 폴더로 제출합니다.

```text
.
├── p1/
├── p2/
└── p3/
```

Bonus를 진행한다면:

```text
.
├── p1/
├── p2/
├── p3/
└── bonus/
```

형태가 됩니다.

필요한 Script는 `scripts` 폴더에, Configuration File은 `confs` 폴더에 위치해야 합니다.

예를 들어:

```text
p1/
├── Vagrantfile
├── scripts/
└── confs/

p2/
├── Vagrantfile
├── scripts/
└── confs/

p3/
├── scripts/
└── confs/

bonus/
├── ...
├── scripts/
└── confs/
```

평가는 **평가받는 팀의 컴퓨터에서 진행**되며, Repository 안에 제출된 내용만 평가 대상이 됩니다.

---

# 정리

Inception of Things의 핵심은 단순히 Kubernetes 명령어를 배우는 것이 아닙니다.

이 프로젝트는 지금까지 배운 인프라 개념을 다음 단계로 연결합니다.

```text
Born2beroot
│
│ "Server를 어떻게 관리하는가?"
│
▼
Linux / VM

        ↓

Inception
│
│ "Application을 어떻게 격리하는가?"
│
▼
Docker / Container

        ↓

Inception of Things — Part 1
│
│ "Container를 운영할 Cluster를 어떻게 만드는가?"
│
▼
Vagrant / K3s / Kubernetes

        ↓

Part 2
│
│ "Kubernetes가 Application을 어떻게 관리하는가?"
│
▼
Pod / Deployment / Service / Ingress

        ↓

Part 3
│
│ "배포 상태를 어떻게 자동으로 관리하는가?"
│
▼
K3d / Argo CD / GitOps

        ↓

Bonus
│
│ "Git Infrastructure까지 직접 운영한다면?"
│
▼
GitLab
```

따라서 이 과제의 학습 방향은 **VM → Container → Orchestration → GitOps**라는 인프라 기술의 발전 흐름을 이해하는 것입니다.
