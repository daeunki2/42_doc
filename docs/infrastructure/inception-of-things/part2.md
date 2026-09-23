# Part 2: K3s and Applications

Part 1에서 Kubernetes가 실행될 Cluster를 만들었다면, Part 2에서는 그 Cluster 위에 Web Application을 배포합니다.

이 단계의 핵심은 Application을 하나 실행하는 것이 아니라, **외부 요청이 Ingress, Service, Deployment, Pod를 거쳐 Container에 도달하는 전체 경로**를 이해하는 것입니다.

## 1. 요구사항을 구조로 바꾸기

Part 2에서는 하나의 VM에서 K3s Server를 실행하고, 세 개의 Web Application을 배포합니다.

```text
VM
└── K3s Server
    └── Kubernetes Cluster
	├── App 1
	├── App 2 x 3 replicas
	└── App 3
```

외부 요청의 `HOST` 값에 따라 서로 다른 Application이 응답해야 합니다.

```text
app1.example.com -> App 1
app2.example.com -> App 2
other host       -> App 3
```

여기서 App 2를 세 개 실행하는 이유는 같은 Application의 복제본을 여러 개 유지하는 Kubernetes의 동작을 경험하기 위해서입니다.

## 2. Container가 바로 노출되지 않는 이유

Docker를 처음 배울 때는 Port Mapping으로 Container를 외부에 노출할 수 있었습니다. Kubernetes에서는 Application을 직접 외부에 연결하기보다 여러 리소스를 역할별로 나눕니다.

```text
외부 요청
    ↓
Ingress: HTTP 요청의 규칙을 판단
    ↓
Service: Pod 그룹에 안정적으로 연결
    ↓
Deployment: Pod의 원하는 개수와 상태를 관리
    ↓
Pod: Container를 실행하는 단위
    ↓
Container: 실제 Web Application
```

이 분리는 복잡해 보이지만 각 리소스가 하나의 문제만 해결하게 해줍니다.

## 3. Pod

Pod는 Kubernetes가 실행하는 가장 작은 배포 단위입니다. 일반적인 경우 Pod 안에 Application Container가 하나 들어갑니다.

```text
Pod
└── Container
    └── Web Application
```

Pod는 영구적인 Server가 아닙니다. 문제가 생기거나 재배포되면 삭제되고 새로 생성될 수 있습니다. 따라서 다른 리소스가 특정 Pod의 IP를 직접 기억하게 만들면 안 됩니다.

## 4. Deployment

Deployment는 Application을 어떤 상태로 유지할지 선언합니다.

```yaml
kind: Deployment
spec:
  replicas: 3
```

이 선언의 의미는 “지금 당장 Pod를 세 번 만들어라”보다 다음에 가깝습니다.

> 이 Application에 해당하는 Pod가 항상 세 개 존재하도록 유지하라.

Deployment는 Pod를 직접 한 번 만드는 명령이 아니라, Pod를 생성하고 상태를 계속 관리하는 Controller입니다.

## 5. Replica와 Desired State

App 2의 Replica가 3이라는 것은 같은 Application을 실행하는 Pod가 세 개 있어야 한다는 뜻입니다.

```text
Desired State: App 2 Pod = 3

Actual State:
  Pod A
  Pod B
  Pod C
```

Pod 하나가 종료되면 실제 상태가 바뀝니다.

```text
Desired State: 3
Actual State:  2
       ↓
Deployment Controller가 차이를 발견
       ↓
새 Pod 생성
       ↓
Actual State: 3
```

이것이 Kubernetes의 핵심인 **선언적 관리**입니다. 우리는 매번 새 Pod를 직접 실행하기보다 원하는 상태를 선언하고, Kubernetes가 실제 상태를 맞추도록 합니다.

## 6. Label과 Selector

Deployment와 Service가 특정 Pod를 찾으려면 Pod의 이름보다 Label을 사용합니다.

```text
Pod
labels:
  app: app2

Service
selector:
  app: app2
```

Service는 `app: app2` Label을 가진 Pod들을 찾아 트래픽을 전달합니다. Replica Pod의 이름과 IP는 바뀔 수 있지만 Label이라는 관계는 유지됩니다.

따라서 YAML을 작성할 때 다음 두 부분이 일치하는지 확인해야 합니다.

```text
Pod template의 labels
	==
Deployment selector
	==
Service selector
```

## 7. Service

Pod는 교체될 수 있으므로 IP가 안정적이지 않습니다. Service는 여러 Pod를 하나의 안정적인 Network Endpoint로 묶습니다.

```text
App 2 Service
       |
       +-- Pod A
       +-- Pod B
       +-- Pod C
```

Service가 해결하는 문제는 “이 Pod의 IP가 무엇인가?”가 아니라 “이 Application 그룹에 어떻게 접근하는가?”입니다.

Service와 Deployment의 역할을 구분해야 합니다.

| 리소스 | 해결하는 문제 |
|---|---|
| Deployment | 어떤 Pod를 몇 개 유지할 것인가 |
| Service | 해당 Pod 그룹에 어떤 안정적인 주소로 접근할 것인가 |

## 8. Ingress

Ingress는 외부 HTTP 요청을 Host나 Path 규칙에 따라 적절한 Service로 전달합니다.

```text
Client Request
Host: app2.example.com
	↓
Ingress Rule
	↓
App 2 Service
	↓
App 2 Pods
```

App 1, App 2, App 3를 하나의 IP에서 제공하면서 Host에 따라 나누는 것이 Ingress의 역할입니다.

Ingress는 Application Container 자체가 아닙니다. 요청을 어느 Service로 보낼지 결정하는 HTTP 진입점입니다.

## 9. 기본 경로가 필요한 이유

과제에서는 App 1과 App 2의 Host 규칙에 맞지 않는 요청을 App 3로 보내야 합니다.

```text
Host = app1.example.com -> App 1 Service
Host = app2.example.com -> App 2 Service
그 외                -> App 3 Service
```

따라서 특정 Host 규칙만 만들고 끝내면 안 됩니다. 일치하지 않는 요청의 Default Backend 또는 기본 라우팅도 요구사항에 포함되는지 확인해야 합니다.

## 10. YAML을 읽는 순서

Kubernetes YAML은 파일 순서보다 리소스 간 연결이 중요합니다. 다음 순서로 읽으면 이해하기 쉽습니다.

```text
1. Deployment가 사용할 Image 확인
2. replicas와 Pod template 확인
3. Pod labels와 Service selector 비교
4. Service port와 targetPort 비교
5. Ingress가 가리키는 Service와 port 확인
6. Host rule과 기본 rule 확인
```

`port`는 Service가 받는 포트이고 `targetPort`는 Pod Container가 실제로 듣는 포트입니다. 두 포트의 의미를 섞으면 Service는 존재하지만 Application에 연결되지 않을 수 있습니다.

## 11. 요청이 실패할 때 계층별 확인

```text
Ingress가 존재하는가?
	↓
Host Rule이 요청과 일치하는가?
	↓
Ingress가 올바른 Service를 가리키는가?
	↓
Service에 Endpoint가 있는가?
	↓
Selector가 Pod Label과 일치하는가?
	↓
Pod가 Running/Ready 상태인가?
	↓
Container가 targetPort에서 응답하는가?
```

문제가 생겼을 때 모든 YAML을 한꺼번에 고치지 말고, 요청이 어느 계층에서 멈추는지 좁혀야 합니다.

## 12. 검증 명령의 의미

```bash
kubectl get deployments
```

Deployment가 원하는 Replica 수를 관리하고 있는지 확인합니다.

```bash
kubectl get pods -o wide
```

Pod의 개수, 상태, IP, 어느 Node에서 실행 중인지 확인합니다.

```bash
kubectl get services
```

Service의 ClusterIP와 포트 구성을 확인합니다.

```bash
kubectl get endpoints
```

Service의 Selector에 실제로 연결된 Pod Endpoint가 있는지 확인합니다.

```bash
kubectl get ingress
```

Host 기반 라우팅 규칙이 Cluster에 등록됐는지 확인합니다.

```bash
kubectl describe pod <pod-name>
kubectl describe ingress <ingress-name>
```

리소스의 이벤트와 설정을 자세히 확인합니다. “생성됐다”와 “정상적으로 동작한다”는 다르므로 이벤트도 함께 봐야 합니다.

## Part 2를 끝낸 뒤 답할 질문

1. Pod IP를 직접 사용하지 않고 Service를 사용하는 이유는 무엇인가?
2. Deployment와 Pod는 어떤 관계인가?
3. Replica가 3이면 Kubernetes는 무엇을 유지하려고 하는가?
4. Service의 Selector와 Pod Label이 다르면 어떤 일이 생기는가?
5. Ingress와 Service는 각각 어느 계층의 문제를 해결하는가?
6. App 2를 세 개 실행하는 것이 실제로 어떤 장점을 주는가?
7. Host가 App 1, App 2와 일치하지 않을 때 요청은 어디로 가야 하는가?
