# Inception of Things: 환경 안에 환경을 만들며 배우는 것

이 과제는 **내 PC 안에 다른 컴퓨터 환경을 만들고, 그 환경에서 앱을 관리하고, 마지막으로 배포까지 자동화하는 과정**입니다. 도구 이름을 먼저 외우기보다 “과제가 무엇을 요구하며, 그 요구를 만족시키는 과정에서 무엇을 배우는가”로 읽어 봅시다.

## 왜 Inception of Things라는 이름일까?

환경 안에 또 다른 환경이 들어가는 모습을 떠올리면 제목을 이해하기 쉽습니다. 내 PC 안에서 VM을 실행하고, 그 안에서 Kubernetes를 운영하고, Kubernetes가 관리하는 Pod 안에서 앱이 실행됩니다. Part 3에서는 Kubernetes Node 자체도 Docker Container로 만듭니다.

여기서 제목을 겹겹이 들어가는 환경에 연결하는 설명은 **과제 구조를 이해하기 위한 비유**입니다. 과제 제작자의 공식 작명 이유라고 단정하는 것은 아닙니다.

```text
내 PC
└── 가상 머신
    └── Kubernetes가 관리하는 환경
        └── Pod
            └── Container
                └── 실행 중인 앱
```

처음에는 이 그림으로 위치 관계를 잡으면 됩니다. 다만 각 상자가 모두 작은 컴퓨터라는 뜻은 아닙니다. VM은 별도 운영체제를 실행하고, Kubernetes는 앱을 관리하는 소프트웨어이며, Pod와 Container는 앱의 실행 단위입니다.

## 큰 그림부터: 각 Part에서 무엇이 달라질까?

| 단계 | 과제가 요구하는 것 | 이를 통해 배울 수 있는 것 |
|---|---|---|
| Part 1 | VM 두 대에 K3s를 설치하고 Server와 Agent로 연결 | 독립된 머신 사이의 네트워크, 인증, Cluster 구성 |
| Part 2 | VM 한 대에서 앱 세 개를 실행하고 Host별로 연결 | 앱의 실행·복제·복구와 요청 전달의 역할 구분 |
| Part 3 | K3d와 Argo CD로 Git의 변경을 자동 배포 | 실행 환경의 격리, 원하는 상태, 배포 자동화와 GitOps |
| Bonus | 외부 GitHub 대신 Local GitLab으로 같은 흐름 구성 | 배포 기준이 되는 저장소 자체를 운영하고 연결하는 방법 |

각 Part는 별도의 환경을 만듭니다. Part 1의 VM 두 대에 Part 2와 Part 3를 계속 덧붙이는 방식은 아닙니다. **구현 환경은 바꾸지만, 앞에서 배운 개념을 다음 단계에서 다시 사용합니다.**

## Part 1: PC 안에 머신을 만들고 서로 연결한다

과제는 Vagrant로 VM 두 대를 만들고, 하나는 K3s Server, 다른 하나는 Agent로 구성하라고 요구합니다. 주소는 각각 `192.168.56.110`, `192.168.56.111`입니다.

먼저 작업용 환경의 바깥 단계를 생략한 개념 그림으로 보면 다음과 같습니다.

```text
내 PC
├── VM 1: K3s Server
│   └── Cluster의 상태를 관리하는 역할
└── VM 2: K3s Agent
    └── 같은 Cluster에 참여하여 실행을 맡는 역할

두 VM은 네트워크로 연결되어 하나의 Cluster에 참여
```

여기서 배우는 것은 단순히 “VM 두 대 켜기”가 아닙니다. 두 환경은 각각의 IP와 프로세스를 가지므로 서로 찾아갈 주소가 필요합니다. 주소로 연결한 뒤에는 같은 Cluster에 참여할 자격도 확인해야 합니다.

**VM 두 대를 요구하기 때문에 독립된 머신의 연결을 배우고, Server와 Agent를 나누기 때문에 관리와 실행의 역할을 배웁니다.**

자세히 읽기: [Part 1](part1.md)

## Part 2: 그 환경 안에서 앱을 실행하고 찾아간다

이번에는 VM 한 대를 사용합니다. 그 안의 K3s가 웹 앱 세 개를 관리합니다. app2는 복제본 세 개를 유지하고, 요청의 Host에 따라 서로 다른 앱이 응답해야 합니다.

```text
내 PC
└── VM 1대
    └── K3s가 관리하는 환경
        ├── app1: Pod 안의 앱
        ├── app2: Pod A, Pod B, Pod C 안의 같은 앱
        └── app3: Pod 안의 앱
```

이제 앱이 “어디에 있는가”에 더해, 바깥의 Client가 “어떻게 찾아가는가”를 배웁니다.

```text
Client -> VM의 IP:80 -> Host에 따라 선택 -> 해당 앱의 Pod
```

하나의 IP로 세 앱을 제공하라는 요구는 HTTP Host와 라우팅을 배우게 합니다. app2를 세 개 유지하라는 요구는 원하는 개수와 실제 상태를 맞추는 관리 방식을 배우게 합니다. Pod가 교체될 수 있기 때문에 Service라는 안정적인 접점이 필요한 이유도 드러납니다.

**Part 1에서 실행할 기반을 이해했다면, Part 2에서는 그 기반 위에 앱을 유지하고 사용자 요청을 연결하는 방법을 배웁니다.**

자세히 읽기: [Part 2](part2.md)

## Part 3: Node도 Container로 만들고, Git을 배포 기준으로 삼는다

이번에는 Vagrant 없이 실습 VM에서 Docker와 K3d를 사용합니다. K3d가 Kubernetes Node를 Docker Container로 만들고 연결합니다.

```text
내 PC
└── 실습 Linux VM
    └── Docker
        └── K3d가 만든 Node Container들
            └── K3s Cluster
                ├── argocd 영역: 배포를 관리하는 프로그램
                └── dev 영역: Pod 안의 사용자 앱
```

Cluster는 Node들에 걸쳐 구성됩니다. 위 그림은 그 실행 기반과 관리 대상을 함께 보여 주는 개념도입니다.

여기에 GitHub는 또 하나의 내부 상자로 들어가는 것이 아니라, **배포 설정을 제공하는 외부 저장소**로 연결됩니다.

```text
GitHub: v2를 실행하라는 설정
    -> Argo CD가 확인하고 반영
    -> Cluster에서 v2 앱 실행
```

Node를 Docker Container로 만들면서 VM과 Container의 차이를 다시 생각하게 됩니다. Git에서 v1을 v2로 바꾸도록 요구하기 때문에, 변경 이력과 배포 상태를 연결하는 GitOps를 배웁니다.

**Part 2가 “앱을 어떻게 실행하고 연결할까?”였다면, Part 3는 “실행할 버전의 기준을 어디에 두고 누가 반영할까?”를 배웁니다.**

자세히 읽기: [Part 3](part3.md)

## 단순한 그림과 실제 과제 환경 구분하기

앞의 Part 1·2 그림은 처음 위치 관계를 이해하기 위해 작업용 바깥 VM을 생략했습니다. 기준 원문인 `inception_of_things.pdf` v4.0은 전체 작업을 VM에서 진행하도록 요구하므로, 이 문서에서 사용하는 실제 환경 구분은 다음과 같습니다.

```text
물리 PC
└── 작업을 진행하는 실습 Linux VM
    ├── Part 1: Vagrant -> Guest VM 2대 -> K3s Cluster
    ├── Part 2: Vagrant -> Guest VM 1대 -> K3s -> 앱 Pod
    └── Part 3: Docker -> K3d Node Container -> K3s -> 앱 Pod
```

이 세 환경을 모두 동시에 실행하라는 뜻은 아닙니다. VM 안에서 다시 VM을 실행하는 구성은 가상화 기반과 CPU의 지원도 확인해야 합니다. 처음 공부할 때는 각 단계가 무엇을 맡는지 먼저 이해하고, 환경 선택과 명령어는 [구현 참고](implementation-reference.md)에서 확인하세요.

제출 저장소에는 `p1`, `p2`, `p3`를 두고 Script와 설정을 각각 `scripts`, `confs`에 모읍니다. 현재 읽는 학습 문서의 폴더 구조와 제출 구조는 별개입니다.

## 선수지식은 두 단계로 읽기

먼저 과제와 독립적으로 네트워크의 전체 흐름을 배운 뒤, 그 지식을 가상 환경에 적용합니다.

1. [네트워크 입문 — 웹 접속의 여정](network-journey.md): `www.naver.com`을 입력하는 순간부터 화면이 보일 때까지 따라가며 DNS, 인터페이스, IP, 경로, TCP, TLS, HTTP를 연결합니다.
2. [VM·Docker로 이어지는 네트워크](foundations.md): 목적지가 내 PC 안의 서버일 때 localhost, 가상 인터페이스, Container Network와 포트 전달이 어떻게 쓰이는지 배웁니다.

첫 페이지에서는 Kubernetes를 몰라도 됩니다. 두 번째 페이지 다음에는 [학습 목표와 지도](subject.md)에서 Kubernetes가 필요한 이유와 Part별 목표를 확인하고, 각 Part로 넘어갑니다.

## 공부하는 방법

각 Part는 개념을 중심으로 읽습니다. 터미널을 열지 않고도 다음 질문에 답할 수 있는 것이 첫 목표입니다.

```text
과제가 무엇을 요구하는가?
    ↓
PC 안에 어떤 구조를 만들게 되는가?
    ↓
이 요구사항을 통해 무엇을 배울 수 있는가?
    ↓
각 구성요소는 무엇을 맡는가?
    ↓
요청이나 상태 변경은 어떤 순서로 이동하는가?
    ↓
하나가 없거나 잘못되면 어떤 일이 생기는가?
```

설정과 실행은 이 관계가 이해된 뒤 [구현 참고](implementation-reference.md)에서 찾아볼 수 있습니다.

## 문서 읽는 순서

1. [네트워크 입문](network-journey.md): 일상적인 웹 접속으로 네트워크 전반을 이해합니다.
2. [VM·Docker와 네트워크](foundations.md): 같은 원리를 내 PC의 가상 환경에 적용합니다.
3. [학습 목표와 지도](subject.md): Kubernetes가 무엇이고 왜 필요한지, 무엇을 만들고 요구사항별로 무엇을 배울지 파악합니다.
4. [Part 1](part1.md): 독립된 머신이 같은 Cluster에 참여하는 과정을 배웁니다.
5. [Part 2](part2.md): 앱을 관리하는 흐름과 사용자 요청을 전달하는 흐름을 나누어 봅니다.
6. [Part 3](part3.md): Git의 변경이 앱의 배포 상태로 이어지는 과정을 배웁니다.
7. [구현 참고](implementation-reference.md): 이해한 내용을 설정과 검증 명령에 적용합니다.
8. [Bonus](bonus.md): GitHub를 Local GitLab으로 바꾸는 의미를 확인합니다.

## 최종적으로 설명할 수 있어야 하는 것

```text
VM과 Container는 무엇이 다른가?
K3s와 K3d는 어떤 관계인가?
Pod, Deployment, Service, Ingress는 어떻게 연결되는가?
Argo CD는 무엇과 무엇의 상태를 비교하는가?
GitOps에서 Git이 Source of Truth라는 말은 무슨 뜻인가?
```

이 질문에 자신의 말로 답할 수 있다면, 명령어를 외운 것보다 과제의 구조를 제대로 이해한 것입니다.
