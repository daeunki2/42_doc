# 42 Infrastructure Notes

> From a single Linux server to containers, Kubernetes, and GitOps.

이 문서는 42의 인프라 관련 프로젝트를 진행하며 배우고 이해한 내용을 기록하기 위한 학습 노트입니다.

프로젝트를 단순히 **완성하는 것**에 목적을 두기보다,

- 이 기술은 무엇인가?
- 왜 필요한가?
- 어떤 문제를 해결하는가?
- 이전에 배운 기술과 어떻게 연결되는가?
- 실제 시스템에서는 어떤 역할을 하는가?

를 이해하는 것을 목표로 합니다.

42의 인프라 프로젝트들은 서로 독립되어 보이지만, 실제로는 하나의 자연스러운 흐름으로 연결됩니다.

---

## Learning Path

### Born2beroot

**Virtual Machine → Linux → System Administration**

가상 머신 위에 Linux 서버를 직접 구성하며  
하나의 서버가 어떻게 만들어지고 관리되는지를 배웁니다.

사용자와 권한, SSH, 방화벽, 서비스, 디스크, 시스템 모니터링 등을 다루면서  
이후 모든 인프라 학습의 기반이 되는 **Linux와 Server Administration**을 이해합니다.

---

### Inception

**Container → Docker → Docker Compose**

하나의 서버 안에서 여러 애플리케이션을  
어떻게 독립적인 환경으로 나누어 실행할 수 있는지 배웁니다.

Docker Image와 Container를 시작으로 Network, Volume, Dockerfile을 이해하고,  
Docker Compose를 이용해 여러 서비스를 하나의 시스템으로 구성합니다.

이 단계에서 관심의 대상은 하나의 서버에서  
**여러 Container와 Service를 어떻게 구성할 것인가**로 확장됩니다.

---

### Inception of Things

**Kubernetes → Container Orchestration → GitOps**

Container를 만드는 것에서 한 단계 더 나아가  
여러 Container를 **어떻게 배포하고, 연결하고, 원하는 상태로 유지할 것인지**를 배웁니다.

Vagrant와 K3s를 이용해 Kubernetes Cluster를 구성하고,  
Pod, Deployment, Service, Ingress를 이용해 Application을 배포합니다.

마지막으로 K3d와 Argo CD를 통해  
Git Repository의 상태를 기준으로 Application을 자동으로 배포하고 관리하는 **GitOps**까지 이어집니다.

---

## The Big Picture

전체 학습 흐름은 다음과 같습니다.

```text
Born2beroot
    │
    │  How do we manage a server?
    ▼
Linux / Virtual Machine
    │
    │
    ▼
Inception
    │
    │  How do we isolate and run applications?
    ▼
Docker / Container
    │
    │
    ▼
Inception of Things
    │
    │  How do we manage containers at scale?
    ▼
Kubernetes
    │
    │
    ▼
GitOps
    │
    │  How do we automate and maintain deployment state?
    ▼
Argo CD