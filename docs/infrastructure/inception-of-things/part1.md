# Part 1: K3s and Vagrant

Part 1의 목표는 **두 개의 Virtual Machine을 만들고 하나의 K3s Kubernetes Cluster로 연결하는 것**입니다.

아직 Application을 배포하는 단계가 아닙니다. 먼저 Kubernetes가 실행될 인프라와 Cluster의 기본 구조를 만듭니다.

## 1. 문제를 먼저 이해하기

Kubernetes는 여러 Machine을 하나의 Cluster로 묶어 Container Application을 관리합니다. 그러므로 먼저 다음 환경이 필요합니다.

```text
Host Computer
    |
    +-- VM 1: K3s Server
    |
    +-- VM 2: K3s Agent
```

과제에서는 이 VM들을 직접 클릭해서 만드는 대신 Vagrant로 정의하고 생성합니다.

## 2. Vagrant는 무엇인가

Vagrant는 Virtual Machine 환경을 코드로 정의하고 반복해서 만들 수 있게 해주는 도구입니다.

직접 VM을 만들면 다음 작업을 매번 수동으로 해야 합니다.

```text
VM 생성 -> CPU/RAM 설정 -> Network 설정 -> OS 설정 -> 프로그램 설치
```

Vagrant를 사용하면 `Vagrantfile`에 이 설정을 기록하고 명령어로 같은 환경을 다시 만들 수 있습니다.

```text
Vagrantfile
	|
	v
  vagrant up
	|
	v
Virtual Machine 생성 및 설정
```

여기서 중요한 점은 Vagrant 자체가 Kubernetes가 아니라는 것입니다. Vagrant는 Kubernetes를 실행할 **환경을 만드는 도구**입니다.

## 3. Provisioning이 필요한 이유

VM만 만든다고 Kubernetes가 실행되는 것은 아닙니다. VM 안에 K3s를 설치하고, Hostname과 Network를 설정하고, 필요한 파일을 배치해야 합니다.

VM 생성 후 초기 설정을 자동으로 수행하는 과정을 Provisioning이라고 합니다.

```text
VM 생성
   ↓
Provisioning Script
   ├── 패키지 설치
   ├── Hostname 설정
   ├── K3s 설치
   └── Cluster 연결에 필요한 설정
```

따라서 Part 1의 Script는 단순한 설치 명령 모음이 아니라, VM을 **Kubernetes Node로 바꾸는 절차**입니다.

## 4. K3s와 Kubernetes

Kubernetes는 Container를 여러 Machine에서 관리하기 위한 플랫폼입니다. K3s는 Kubernetes의 핵심 기능을 유지하면서 작은 환경에서도 실행하기 쉽게 만든 경량 Distribution입니다.

관계를 다음처럼 이해하면 됩니다.

```text
Kubernetes = 플랫폼과 개념
K3s        = Kubernetes를 배포하는 한 가지 방식
```

Part 1에서는 K3s를 각 VM에 설치하여 Kubernetes Cluster를 구성합니다.

## 5. Server와 Agent

K3s Cluster에는 역할이 나뉜 Node가 있습니다.

### Server

Server는 Cluster의 제어를 담당합니다. 어떤 Node가 참여하고 있는지, 어떤 리소스가 원하는 상태인지, 어떤 작업을 수행해야 하는지 관리하는 중심 Node입니다.

### Agent

Agent는 Cluster에 참여하여 실제 Workload를 실행하는 Node입니다. Server가 내린 상태에 따라 Pod와 Container가 실행될 수 있습니다.

```text
K3s Cluster
    |
    +-- Server: Cluster 제어
    |
    +-- Agent: Workload 실행
```

과제의 두 VM은 다음과 같은 관계를 가집니다.

```text
VM 1: K3s Server
  - Cluster의 중심 역할
  - kubectl을 통해 관리

VM 2: K3s Agent
  - Server가 관리하는 Worker
  - Cluster에 참여
```

## 6. VM들이 Cluster가 되는 과정

Agent는 혼자 Kubernetes Cluster가 되지 않습니다. Server의 주소와 인증 정보를 사용해 Server에 등록되어야 합니다.

```text
1. Server VM에서 K3s Server 시작
2. Server의 주소와 Join Token 확인
3. Agent VM이 Server 주소에 접속
4. Agent가 Token으로 자신을 인증
5. Server가 Agent를 Cluster Node로 등록
```

결과적으로 Host 입장에서는 VM 두 대가 따로 보이지만, Kubernetes 입장에서는 하나의 Cluster에 속한 두 Node로 보입니다.

## 7. kubectl은 무엇을 하는가

`kubectl`은 Kubernetes API Server와 통신하는 CLI입니다. 직접 Agent의 프로세스에 명령을 보내는 도구가 아닙니다.

```text
kubectl
   |
   v
Kubernetes API Server
   |
   +-- Cluster 상태 조회
   +-- 리소스 생성/변경
   +-- Node와 Pod 상태 확인
```

예를 들어 `kubectl get nodes`는 현재 Cluster에 등록된 Node 목록을 API Server에 요청합니다.

## 8. 네트워크와 SSH를 따로 확인하기

Part 1에서 문제가 생기면 Kubernetes 문제라고 단정하지 말고 계층을 나누어 확인해야 합니다.

```text
SSH 접속 가능?
	↓
VM 간 IP 통신 가능?
	↓
K3s Process 실행 중?
	↓
Agent가 Server에 등록됐나?
	↓
kubectl이 API Server에 연결되나?
```

SSH가 되지 않으면 K3s보다 먼저 Vagrant Network와 VM 상태를 확인해야 합니다. Agent가 등록되지 않으면 Server 주소, Join Token, 방화벽, K3s 로그를 확인해야 합니다.

## 9. 구현 전 체크리스트

- VM 두 대의 이름과 IP가 과제 요구사항과 일치하는가?
- 두 VM 모두 SSH Key 기반 접속이 가능한가?
- Server와 Agent의 역할이 뒤바뀌지 않았는가?
- Agent가 접속할 Server의 고정 IP를 알고 있는가?
- Join Token을 안전하게 전달할 수 있는가?
- `kubectl`이 올바른 kubeconfig를 사용하고 있는가?
- 재실행했을 때 Provisioning이 실패하지 않거나 상태를 확인할 수 있는가?

## 10. 검증 명령의 의미

```bash
vagrant status
```

Vagrant가 관리하는 VM의 상태를 확인합니다.

```bash
vagrant ssh <server-vm>
```

Server VM에 SSH로 접속합니다.

```bash
kubectl get nodes -o wide
```

Cluster에 등록된 Node와 각 Node의 주소, 역할, 상태를 확인합니다.

```bash
kubectl get pods -A
```

모든 Namespace의 Pod를 확인합니다. Part 1에서는 Application보다 K3s 자체 구성요소가 정상인지 보는 데 사용합니다.

## Part 1을 끝낸 뒤 답할 질문

1. Vagrant와 K3s는 각각 어떤 문제를 해결하는가?
2. VM 두 대는 어떤 과정을 통해 하나의 Cluster가 되는가?
3. Server와 Agent의 역할은 어떻게 다른가?
4. `kubectl`은 어떤 경로로 Cluster에 명령을 전달하는가?
5. SSH가 되지만 Node가 등록되지 않을 때 어느 부분을 확인해야 하는가?
