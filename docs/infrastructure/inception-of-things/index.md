# Inception of Things

Inception of Things는 Container를 실행하는 것에서 한 단계 더 나아가, 여러 Container를 **Cluster에서 배포하고 관리하는 방법**을 배우는 과제입니다.

이 과제는 Kubernetes의 모든 기능을 다루는 프로젝트가 아닙니다. 작은 환경을 직접 만들고, Application을 배포하고, Git의 상태를 기준으로 배포를 자동화하면서 Kubernetes와 GitOps의 핵심 흐름을 경험하는 입문 프로젝트입니다.

## 한 문장으로 이해하기

```text
Vagrant로 환경을 만들고
K3s/K3d로 Kubernetes를 실행한 뒤
Application을 배포하고
Argo CD로 Git과 Cluster를 동기화한다.
```

## 과제의 질문

각 Part는 서로 다른 질문에 답합니다.

| Part | 핵심 질문 | 주요 기술 |
|---|---|---|
| Part 1 | Kubernetes가 실행될 환경을 어떻게 만드는가? | Vagrant, VM, K3s |
| Part 2 | Kubernetes는 Application을 어떻게 실행하고 외부에 제공하는가? | Pod, Deployment, Service, Ingress |
| Part 3 | Application의 배포 상태를 어떻게 Git 기준으로 관리하는가? | K3d, Namespace, Argo CD, GitOps |
| Bonus | Git 저장소까지 직접 운영한다면 어떻게 되는가? | GitLab, Argo CD |

## 전체 구조

```text
Part 1
Vagrant -> Virtual Machines -> K3s Cluster
						  |
Part 2                                v
Application -> Deployment -> Pods -> Service -> Ingress
						  ^
Part 3                                |
Git Repository -> Argo CD -> K3d/Kubernetes Cluster
```

## 선행 지식

처음부터 Kubernetes를 모두 공부할 필요는 없습니다. 다음 개념을 알고 있으면 과제의 설명을 따라가기 쉽습니다.

- Linux: Process, Service, IP, Port, SSH
- Virtual Machine: 하나의 컴퓨터 안에서 독립적인 컴퓨터 환경을 실행하는 개념
- Docker: Image와 Container, Network
- Git: Repository, commit, push, remote
- YAML: 설정을 계층적인 데이터로 표현하는 형식

앞선 `Born2beroot`와 `Inception`에서 배운 VM, Linux, Network, Container 개념은 이 과제의 기반이 됩니다.

## 공부하는 방법

각 Part를 다음 순서로 진행합니다.

```text
요구사항 읽기
	↓
등장한 개념의 역할 이해하기
	↓
구성요소 사이의 요청 흐름 그리기
	↓
설정 파일 작성하기
	↓
명령어로 실제 상태 확인하기
	↓
문제가 생겼을 때 어느 계층인지 좁히기
```

설정 파일을 먼저 복사하는 것보다, 그 파일이 선언하는 **원하는 상태**를 먼저 말로 설명할 수 있어야 합니다.

## 문서 읽는 순서

1. [Project Guide](subject.md)에서 전체 요구사항과 용어를 확인합니다.
2. [Part 1](part1.md)에서 Cluster가 만들어지는 과정을 이해합니다.
3. [Part 2](part2.md)에서 Application 요청이 Pod까지 도달하는 흐름을 이해합니다.
4. [Part 3](part3.md)에서 Git이 배포의 기준이 되는 과정을 이해합니다.
5. [Bonus](bonus.md)에서 GitHub를 Local GitLab으로 바꾸는 의미를 확인합니다.

## 최종적으로 설명할 수 있어야 하는 것

```text
VM과 Container는 무엇이 다른가?
K3s와 K3d는 어떤 관계인가?
Pod, Deployment, Service, Ingress는 어떻게 연결되는가?
Argo CD는 무엇과 무엇의 상태를 비교하는가?
GitOps에서 Git이 Source of Truth라는 말은 무슨 뜻인가?
```

이 질문에 자신의 말로 답할 수 있다면, 명령어를 외운 것보다 과제의 구조를 제대로 이해한 것입니다.
