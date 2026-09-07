# 42 Infrastructure Notes

42의 인프라 관련 프로젝트를 진행하며 배운 내용을 정리한 문서입니다.

단순히 프로젝트를 완성하는 방법보다는 **각 기술이 무엇인지, 왜 필요한지, 그리고 서로 어떻게 연결되는지** 이해하는 것을 목표로 합니다.

## 학습 흐름

### Born2beroot

**Virtual Machine → Linux → System Administration**

가상 머신에 Linux 서버를 구축하며 서버와 시스템 관리의 기본 개념을 학습합니다.

사용자와 권한, SSH, 방화벽, 서비스, 시스템 모니터링 등의 개념을 다룹니다.

---

### Inception

**Container → Docker → Docker Compose**

하나의 서버를 관리하는 것에서 나아가 애플리케이션을 여러 개의 독립적인 컨테이너로 분리합니다.

Docker Image, Container, Network, Volume과 Docker Compose를 이용해 여러 서비스를 함께 구성하는 방법을 학습합니다.

---

### Inception of Things

**Kubernetes → Container Orchestration → GitOps**

하나의 머신에서 컨테이너를 관리하는 것을 넘어 Kubernetes를 이용해 컨테이너화된 애플리케이션을 배포하고 관리합니다.

K3s, K3d를 이용해 Kubernetes 환경을 구성하고, 마지막에는 Argo CD와 GitOps를 이용한 자동화된 배포까지 학습합니다.

---

## 전체 흐름

```text id="x3v9a1"
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
   GitOps
```

각 프로젝트는 독립된 기술을 배우는 것이 아니라, 이전 프로젝트에서 배운 개념 위에 새로운 인프라 계층을 하나씩 쌓아가는 과정입니다.
