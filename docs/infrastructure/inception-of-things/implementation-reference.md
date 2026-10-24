# 구현 참고: 설정 예시와 검증 명령

이 페이지는 개념을 읽은 뒤 필요할 때 찾아보는 참고 자료입니다. 처음 읽을 때는 [네트워크 입문](network-journey.md)과 [VM·Docker의 네트워크](foundations.md), [Part 1](part1.md), [Part 2](part2.md), [Part 3](part3.md)를 먼저 읽으세요. 예시는 환경에 맞게 완성하고 검증해야 하며, 여기의 명령이 실제 실행 검증을 마쳤다는 뜻은 아닙니다.


## Part 1: K3s and Vagrant

목표는 **Vagrant로 VM 두 대를 만들고, 두 VM을 하나의 K3s Cluster로 연결하는 것**입니다. [네트워크 입문](network-journey.md)과 [VM·Docker의 네트워크](foundations.md)를 먼저 읽고, IP와 localhost를 구분한 상태에서 시작하세요.

### 1. 과제 원문에서 요구하는 것

기준은 저장소의 `inception_of_things.pdf` v4.0, 본문 6–8쪽(PDF 7–9페이지)입니다.

| 항목 | 요구사항 |
|---|---|
| 실행 방식 | Vagrantfile로 VM 두 대 실행 |
| 운영체제 | 선택한 배포판의 최신 안정 버전 |
| 이름 | 팀원 login 뒤에 `S`, `SW`를 붙임 |
| Server IP | `192.168.56.110` |
| Worker IP | `192.168.56.111` |
| 네트워크 | 원문은 primary network interface의 전용 IP를 요구 |
| SSH | 두 VM 모두 비밀번호 입력 없이 접속 |
| K3s | Server는 controller/server, Worker는 agent |
| 관리 도구 | kubectl 설치 및 사용 |
| 자원 | 원문은 1 CPU, RAM 512MB 또는 1024MB의 최소 구성을 강하게 권장 |

원문의 자원 권장과 현재 K3s 실행 조건은 구분해야 합니다. 현재 공식 문서의 기본 최소치는 Server 2 CPU/2GB, Agent 1 CPU/512MB이며 Workload 자원은 별도입니다. 설치 버전과 실제 사용량을 확인하고, 원문과의 차이를 기록하세요. [K3s Requirements](https://docs.k3s.io/installation/requirements)

### 2. 이번에 만들 구조

```text
실습 Linux VM: Vagrant 실행
    │
    ├── <login>S  / 192.168.56.110
    │   └── K3s Server: Cluster 상태를 관리하는 API와 제어 구성요소
    │
    └── <login>SW / 192.168.56.111
        └── K3s Agent: Server에 등록하고 Pod를 실행할 수 있는 Node
                │
                └── https://192.168.56.110:6443 에 연결
```

Node는 Kubernetes에 참여한 머신입니다. 여기서는 VM 한 대가 Node 하나가 됩니다. Cluster는 함께 관리되는 Node들의 집합입니다.

Server도 기본 설정에서는 Pod를 실행할 수 있습니다. “Server는 관리만 하고 Agent만 앱을 실행한다”는 규칙은 아닙니다. 그래서 다음 Part에서는 Server 한 대로 앱까지 실행할 수 있습니다.

### 3. Vagrant, Provider, Box를 구분하기

| 이름 | 역할 |
|---|---|
| Vagrant | VM의 생성과 설정 과정을 명령어로 관리 |
| Provider | 실제 가상화를 수행하는 기반. 예: VirtualBox, libvirt |
| Box | VM을 만들 때 사용하는 운영체제 기본 이미지 |
| Vagrantfile | VM 이름, Box, Network, 자원, 초기 설정을 선언한 파일 |
| Provisioning Script | 생성된 VM에 패키지를 설치하고 K3s 등을 설정하는 절차 |

먼저 Provider와 CPU 아키텍처에 맞는 안정판 Box를 선택합니다. 아래는 **구조 설명용 뼈대**입니다. `REPLACE_WITH_COMPATIBLE_STABLE_BOX`를 실제 값으로 바꾸고, Provider별 자원·인터페이스 설정과 Script를 완성해야 합니다.

```ruby
Vagrant.configure("2") do |config|
  config.vm.box = "REPLACE_WITH_COMPATIBLE_STABLE_BOX"

  config.vm.define "server" do |server|
    server.vm.hostname = "yourloginS"
    server.vm.network "private_network", ip: "192.168.56.110"
    server.vm.provision "shell", path: "scripts/server.sh"
  end

  config.vm.define "worker" do |worker|
    worker.vm.hostname = "yourloginSW"
    worker.vm.network "private_network", ip: "192.168.56.111"
    worker.vm.provision "shell", path: "scripts/agent.sh"
  end
end
```

`server`와 `worker`는 Vagrant 명령에서 선택할 이름이고, `vm.hostname`은 Guest OS의 hostname입니다. Provider 화면에 표시되는 VM 이름도 원문에 맞게 별도로 지정할 수 있습니다. 여기의 `yourlogin`은 실제 팀원의 login으로 바꿉니다.

Private Network 설정만으로 원문의 primary interface 조건까지 충족했다고 판단하면 안 됩니다. Provider는 NAT를 첫 번째, Private Network를 추가 인터페이스로 만들 수 있습니다. 실제 NIC 배치와 주소를 확인하고 Provider 설정에서 요구사항을 충족시켜야 합니다.

### 4. IP가 두 개 이상이면 어떤 주소를 써야 할까?

Vagrant의 NAT 주소는 다운로드 등에 사용될 수 있습니다. Node끼리 찾아갈 주소는 과제에서 지정한 `.110`, `.111`로 명확히 정합니다.

```text
Agent -> Server: 192.168.56.110:6443
kubectl -> API: Server VM 안에서는 로컬 API 주소 사용 가능
Pod 사이 통신: K3s가 구성하는 별도 Pod Network 사용
```

VM의 IP와 Pod의 IP는 다릅니다. K3s 기본 Pod 대역과 Service 대역도 Node 대역과 겹치지 않아야 합니다.

```bash
# 각 Guest VM에서
hostname
ip -br address
ip route
```

멀티 NIC 환경에서는 K3s의 `node-ip`와 필요 시 `flannel-iface`를 명시해 실습용 인터페이스를 사용하게 합니다. `flannel-iface`에는 실제 확인한 이름을 넣습니다. IP 지정만으로 모든 Overlay 트래픽의 인터페이스 선택이 해결된다고 가정하지 마세요.

### 5. Server를 먼저 준비하는 이유

Agent는 Server의 주소와 Join Token을 알아야 등록할 수 있습니다. 따라서 다음 순서가 필요합니다.

```text
Server VM 준비
  -> Server K3s 설치
  -> API 준비 완료 대기
  -> Join Token 확보
  -> Agent VM 준비
  -> Server 주소와 Token으로 등록
  -> 두 Node의 Ready 확인
```

Server의 `/etc/rancher/k3s/config.yaml`에는 다음과 같은 설정을 둘 수 있습니다.

```yaml
node-ip: 192.168.56.110
# 멀티 NIC 환경에서 필요한 경우 실제 인터페이스 이름으로 추가:
# flannel-iface: enp0s8
```

설정 디렉터리와 파일은 Provisioning Script가 root 권한으로 준비합니다. 이후 공식 설치 Script를 내려받아 내용을 확인한 뒤 실행하는 흐름입니다.

```bash
# Server VM에서 실행하는 예시
curl -sfL https://get.k3s.io -o /tmp/install-k3s.sh
sudo sh /tmp/install-k3s.sh server
sudo systemctl status k3s --no-pager
sudo k3s kubectl get nodes -o wide
```

재현 가능한 제출 Script에서는 검증한 K3s 버전을 `INSTALL_K3S_VERSION`으로 고정하고 두 Node가 같은 버전을 사용하게 합니다. 배포판 최신 안정판 조건과 K3s 버전 선택은 별도 항목입니다.

설치 Script는 K3s 서비스와 관리 도구를 설치하고 `/etc/rancher/k3s/k3s.yaml`에 kubeconfig를 만듭니다. [K3s Quick-Start](https://docs.k3s.io/quick-start)

### 6. Agent는 어떻게 같은 Cluster에 들어오는가?

Join Token은 Server의 `/var/lib/rancher/k3s/server/node-token`에 있습니다. Agent 등록에 필요한 인증 정보이므로 Git에 넣지 않습니다.

아래는 Agent의 `/etc/rancher/k3s/config.yaml` 구성 예시입니다. Token은 Provisioning 과정에서 실제 값으로 전달하고 파일 접근 권한을 제한합니다.

```yaml
server: https://192.168.56.110:6443
token: REPLACE_WITH_SERVER_JOIN_TOKEN
node-ip: 192.168.56.111
# flannel-iface: 실제_실습용_인터페이스
```

```bash
# Agent VM에서 실행하는 예시
curl -sfL https://get.k3s.io -o /tmp/install-k3s.sh
sudo sh /tmp/install-k3s.sh agent
sudo systemctl status k3s-agent --no-pager
```

최종 `scripts/agent.sh`에서는 위 placeholder를 직접 고치게 두는 대신 Token을 자동으로 전달받도록 만듭니다. 예를 들어 인증된 SSH 경로로 읽거나, 접근을 제한한 임시 파일을 사용할 수 있습니다. 공유 폴더를 이용한다면 해당 Provider에서 지원되는지 확인하고 Token 전달 후 임시 파일을 제거합니다.

자동화에서는 “몇 초 잔다”보다 API 응답과 Token 파일 생성을 검사하고 제한 시간 안에 준비되지 않으면 명확히 실패하도록 만드는 편이 원인 파악에 좋습니다.

### 7. SSH와 Kubernetes 인증은 다르다

```text
SSH Key   -> Linux 사용자로 VM에 로그인
Join Token -> Agent가 Cluster에 참여
kubeconfig -> kubectl이 어느 API에 어떤 자격으로 요청할지 결정
```

비밀번호 없는 SSH는 인증이 없는 접속이 아닙니다. SSH 공개키·개인키를 사용한다는 뜻입니다. Vagrant는 Provider와 Box 설정에 맞는 SSH 정보를 관리합니다.

```bash
# 실습 VM의 p1 디렉터리에서
vagrant status
vagrant ssh-config server
vagrant ssh server
```

`ssh-config`에서 HostName, Port, IdentityFile을 확인할 수 있습니다. SSH 경로가 NAT 포트 전달을 사용하면 Vagrant SSH가 된다는 사실만으로 `.110`과 `.111` 사이 통신까지 검증되지는 않습니다.

`sudo k3s kubectl`은 Server 안에서 시작하기 쉬운 방식입니다. 다른 머신에서 kubectl을 사용하려면 API 주소, 인증서, kubeconfig와 파일 권한을 별도로 준비해야 합니다. kubeconfig를 무조건 전체 사용자에게 읽기 가능하게 바꾸는 방식으로 해결하지 마세요.

### 8. 실제 구현 순서와 파일 역할

아래는 **과제 제출 저장소**의 예시 구조입니다. 현재 학습 문서 저장소의 `docs` 구조와는 별개입니다.

```text
p1/
├── Vagrantfile
├── scripts/
│   ├── server.sh
│   └── agent.sh
└── confs/
    └── 필요한 K3s 설정 템플릿
```

1. Vagrantfile로 두 VM의 이름, 자원, IP를 먼저 맞춥니다.
2. 양쪽 SSH와 실제 인터페이스를 확인합니다.
3. Server Script로 K3s를 설치하고 API 준비를 확인합니다.
4. Agent Script에 Server 주소와 Token 전달을 연결합니다.
5. 두 Node가 Ready가 될 때까지 확인합니다.
6. 수동으로 알아낸 절차를 Script에 반영하고 새 환경에서도 재현되는지 검증합니다.

초기 학습에서는 순서를 눈으로 보기 위해 다음처럼 실행할 수 있습니다.

```bash
# 실습 VM의 p1 디렉터리에서
vagrant up server
vagrant up worker
```

최종 자동화에서도 Server 준비 전에 Worker가 등록을 시도하는 상황을 처리해야 합니다. Script 재실행 시 기존 설정과 Token을 어떻게 다룰지도 정합니다.

### 9. 완료를 어떻게 판단할까?

```bash
# Server VM에서
sudo k3s kubectl get nodes -o wide
sudo k3s kubectl wait --for=condition=Ready nodes --all --timeout=180s
sudo k3s kubectl get pods -A
```

판단 기준은 Node **두 개**, 두 Node 모두 `Ready`, Internal IP가 각각 `.110`, `.111`인 것입니다. Agent의 `ROLES`가 `<none>`이라고 해서 자동으로 오류는 아닙니다. 역할 표시용 Label과 실제 Agent 동작은 구분합니다.

`kubectl get pods -A`의 `-A`는 모든 Namespace를 뜻합니다. 시스템 Pod는 실행 준비 상태를 확인하고, 일회성 설치 Job은 `Completed`일 수 있습니다.

| 증상 | 먼저 확인할 것 |
|---|---|
| VM이 안 만들어짐 | Provider, Box 아키텍처, 중첩 가상화, 자원 |
| SSH 실패 | `vagrant status`, `vagrant ssh-config`, Guest 부팅 상태 |
| Agent 등록 실패 | `.110:6443` 경로, Token, Server 준비 여부 |
| Node IP가 예상과 다름 | K3s node-ip, 실제 NIC, 라우팅 |
| Node NotReady | `kubectl describe node`, Runtime/CNI 상태와 로그 |

```bash
# Server VM
sudo journalctl -u k3s -n 100 --no-pager
# Agent VM
sudo journalctl -u k3s-agent -n 100 --no-pager
```

API 연결의 TCP 6443 외에도 Node 간 통신이 필요합니다. 기본 Flannel VXLAN은 UDP 8472, kubelet 관련 통신은 TCP 10250 등을 사용합니다. 네트워크 모드별 요구 포트는 [K3s Requirements](https://docs.k3s.io/installation/requirements)에서 확인합니다.

### 10. 스스로 설명할 질문

1. VM을 만드는 도구와 Cluster를 운영하는 도구는 각각 무엇인가?
2. Agent의 Server 주소에 localhost를 쓰면 왜 실패하는가?
3. SSH Key, Join Token, kubeconfig의 역할은 어떻게 다른가?
4. Server도 Pod를 실행할 수 있다면 Part 2의 Node는 몇 개면 되는가?
5. SSH는 되는데 Agent 등록이 실패할 때 무엇부터 확인할 것인가?

다음: [Part 2 — 요청을 세 개의 앱으로 나누기](part2.md)

## Part 2: K3s and Three Applications

이번 목표는 **하나의 IP로 들어온 HTTP 요청을 Host에 따라 세 개의 앱으로 나누는 것**입니다. IP·Port·HTTP Host가 헷갈리면 [네트워크 입문](network-journey.md)를 먼저 확인하세요.

### 1. 과제 요구사항과 최종 모습

기준은 `inception_of_things.pdf` v4.0, 본문 9–11쪽(PDF 10–12페이지)입니다.

| 항목 | 요구사항 |
|---|---|
| VM | 최신 안정 배포판, 이름은 `<login>S`, IP는 `192.168.56.110` |
| K3s | Server 모드, VM 한 대 |
| 앱 | 구분 가능한 웹 앱 세 개 |
| Host `app1.com` | app1 응답 |
| Host `app2.com` | app2 응답 |
| 그 외 Host | app3 기본 응답 |
| app2 | Replica 3개 |
| 평가 | Ingress 구성을 직접 보여 주기 |

Part 1의 두 Node를 그대로 유지하는 구성이 아닙니다. Part 1을 종료하고 `p2`에서 별도의 Server 한 대를 만듭니다. 기존 `.110` VM과 동시에 실행해 IP가 충돌하지 않게 합니다.

### 2. 관리 흐름과 요청 흐름을 분리하기

먼저 **누가 Pod를 만드는지** 봅니다.

```text
Deployment: app2를 3개 유지하라는 선언
    -> ReplicaSet: 필요한 Pod 개수 유지
    -> Pod 3개
        -> 각각 웹 앱 Container 실행
```

다음은 **Client의 요청이 어디로 가는지**입니다.

```text
Client -> 192.168.56.110:80
       -> Ingress Controller (예: Traefik)
       -> Ingress 규칙으로 선택한 Service의 backend
       -> Ready Pod 중 하나의 웹 앱
```

Deployment는 요청을 통과시키는 Proxy가 아닙니다. Service는 Pod를 묶는 안정적인 진입점을 제공하며, Controller 구현에 따라 Service의 Endpoint를 찾아 Pod로 직접 연결하기도 합니다. 그림은 논리적인 연결을 표현합니다.

### 3. Pod가 바뀌어도 앱을 찾으려면?

Pod는 교체될 수 있으므로 이름과 IP가 고정된 서버처럼 취급하면 안 됩니다. Pod에는 Label을 붙이고, Service는 Selector로 그 Label을 찾습니다.

```text
Service selector: app=app2
    ├── Pod A: app=app2
    ├── Pod B: app=app2
    └── Pod C: app=app2
```

Pod B가 교체되어도 새 Pod에 같은 Label이 있으면 Service의 대상이 될 수 있습니다. Service 이름을 바꿀 필요가 없습니다.

Replica가 세 개여도 물리 VM은 한 대입니다. Pod 하나의 종료에 대응할 수는 있지만, VM 자체가 꺼지는 장애까지 견디는 구성은 아닙니다. 또한 요청 세 번이 반드시 서로 다른 Pod로 한 번씩 가는 것도 아닙니다.

### 4. Ingress와 Ingress Controller

Ingress는 HTTP Host·Path와 backend Service의 관계를 적은 **설정 리소스**입니다. 실제 요청을 받아 처리하는 프로그램이 Ingress Controller입니다.

```text
Ingress YAML: app1.com이면 app1 Service로 보내라
                         │ 설정을 읽음
                         ▼
Traefik Controller: 실제 HTTP 요청을 받아 전달
```

기본 K3s 설치에는 Traefik이 포함되지만 설치 옵션으로 비활성화할 수도 있습니다. YAML 작성 전에 Controller와 IngressClass를 확인합니다.

기본 구성에서 Traefik은 LoadBalancer Service를 만들고 K3s의 ServiceLB가 Node의 80·443 포트를 연결합니다. 따라서 이 실습의 `.110:80` 진입점이 어디서 생기는지도 함께 확인할 수 있습니다. [K3s Networking Services](https://docs.k3s.io/networking/networking-services)

```bash
# Part 2 Server VM에서. 이후 kubectl 예시도 이 위치입니다.
sudo k3s kubectl get pods -n kube-system
sudo k3s kubectl get ingressclass
```

이 문서의 예시는 `traefik` IngressClass가 준비되어 있다고 가정합니다. Ingress 파일만 만들고 Controller가 없으면 라우팅이 동작하지 않습니다. [Kubernetes Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/)

### 5. 앱 하나의 YAML을 먼저 이해하기

다음은 `p2/confs/app2.yaml` 예시입니다. 앱 구분을 위해 NGINX의 첫 페이지에 앱 이름과 Pod hostname을 씁니다. Image Tag는 설명용으로 사용했으며 제출 시 검증한 버전으로 고정하세요.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app2
spec:
  replicas: 3
  selector:
    matchLabels:
      app: app2
  template:
    metadata:
      labels:
        app: app2
    spec:
      containers:
        - name: web
          image: nginx:stable-alpine
          command: ["/bin/sh", "-c"]
          args:
            - |
              printf 'app2 - %s\n' "$HOSTNAME" > /usr/share/nginx/html/index.html
              exec nginx -g 'daemon off;'
          ports:
            - name: http
              containerPort: 80
          readinessProbe:
            httpGet:
              path: /
              port: http
            initialDelaySeconds: 2
            periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: app2
spec:
  selector:
    app: app2
  ports:
    - name: http
      port: 80
      targetPort: http
```

`---`는 한 파일에서 여러 YAML 문서를 구분합니다. 위 파일에는 Deployment와 Service 두 리소스가 있습니다.

| 위치 | 읽는 방법 |
|---|---|
| `replicas: 3` | Pod 세 개를 유지한다 |
| `selector.matchLabels` | 이 Deployment가 관리할 Pod Label |
| `template` | 새 Pod를 만들 때 사용할 설계도 |
| `template.metadata.labels` | 실제 Pod에 붙는 Label |
| `Service.spec.selector` | Service가 연결할 Pod Label |
| `containerPort: 80` | 앱 포트 정보를 선언한다. 자체적으로 서버를 실행하거나 외부 포트를 열지는 않는다 |
| `port: 80` | Service 쪽 포트 |
| `targetPort: http` | Pod의 이름 붙은 `http` 포트, 여기서는 80 |
| `readinessProbe` | HTTP 응답을 검사하여 요청을 받을 준비가 됐는지 판단 |

앱의 실제 프로세스는 NGINX가 실행합니다. `exec`는 Shell 대신 NGINX가 주 프로세스가 되게 하고, `daemon off;`는 NGINX가 Foreground에서 실행되게 합니다.

같은 구조로 `app1.yaml`과 `app3.yaml`을 만듭니다. 각 파일에서 이름·Label·Selector·출력 문자열을 해당 앱으로 바꾸고 `replicas`는 1로 둡니다. 서로 다른 파일이 실수로 같은 Label을 쓰면 여러 Service가 같은 Pod 그룹을 선택할 수 있습니다.

### 6. Ingress로 세 앱 연결하기

`p2/confs/ingress.yaml` 예시입니다. 앱 리소스들과 같은 `default` Namespace에 배포합니다.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: applications
spec:
  ingressClassName: traefik
  rules:
    - host: app1.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: app1
                port:
                  number: 80
    - host: app2.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: app2
                port:
                  number: 80
    - http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: app3
                port:
                  number: 80
```

마지막 규칙은 Host 조건이 없는 catch-all입니다. Traefik의 기본 규칙 우선순위에서는 구체적인 Host 규칙이 먼저 선택되고 나머지가 app3로 연결됩니다. 다른 Controller를 쓰거나 우선순위를 수정했다면 동작을 따로 검증합니다.

Ingress backend의 `port.number: 80`은 Service의 `port`를 가리킵니다. Pod의 `targetPort`와 직접 연결하는 필드가 아닙니다.

### 7. 파일에서 실행 상태까지

```text
p2/
├── Vagrantfile
├── scripts/
│   └── provision.sh
└── confs/
    ├── app1.yaml
    ├── app2.yaml
    ├── app3.yaml
    └── ingress.yaml
```

Provisioning Script는 K3s 설치, Node 준비 대기, Controller 준비 확인, YAML 적용을 차례로 수행합니다. 아래 명령은 `confs`가 Server VM의 `/vagrant/confs`에 공유되어 있다고 가정합니다. 공유 폴더를 쓰지 않는 Provider에서는 파일 복사 위치를 정하고 경로를 바꿉니다.

```bash
# Server VM에서
sudo k3s kubectl wait --for=condition=Ready nodes --all --timeout=180s
sudo k3s kubectl apply -n default -f /vagrant/confs/
sudo k3s kubectl rollout status deployment/app1 --timeout=180s
sudo k3s kubectl rollout status deployment/app2 --timeout=180s
sudo k3s kubectl rollout status deployment/app3 --timeout=180s
sudo k3s kubectl get deployments,pods,services,ingress
```

`apply`는 원하는 상태를 API에 등록합니다. 성공 메시지가 출력돼도 Image 다운로드나 Pod 준비까지 끝났다는 뜻은 아닙니다. `rollout status`로 배포 진행을 확인합니다.

### 8. 먼저 내부 연결, 그다음 외부 요청

Server VM에서 다음을 확인합니다.

```bash
sudo k3s kubectl get pods -l app=app2 -o wide
sudo k3s kubectl get endpointslices -l kubernetes.io/service-name=app2
sudo k3s kubectl describe service app2
sudo k3s kubectl describe ingress applications
```

EndpointSlice에는 Service의 연결 대상 주소와 준비 상태가 들어갑니다. 대상이 없다면 Service Selector와 Pod Label부터 비교합니다. [Kubernetes Service](https://kubernetes.io/docs/concepts/services-networking/service/)

그다음 **실습 VM**에서 요청합니다.

```bash
curl -i -H 'Host: app1.com' http://192.168.56.110/
curl -i -H 'Host: app2.com' http://192.168.56.110/
curl -i -H 'Host: unknown.example' http://192.168.56.110/
curl -i http://192.168.56.110/
```

응답 본문이 순서대로 app1, app2, app3, app3인지 확인합니다. 마지막 요청에도 curl이 IP 기반 Host 헤더를 보냅니다. “Host가 없는 요청”이 아니라 “app1/app2와 일치하지 않는 요청”입니다.

브라우저에서 `http://app1.com/`을 쓰고 싶다면 **브라우저를 실행하는 머신**의 이름 해석을 `.110`으로 연결하고 그 머신에서 VM까지의 네트워크 경로도 확보해야 합니다. 첫 검증은 hosts 파일 수정 없이 curl로 하는 편이 원인을 좁히기 쉽습니다.

### 9. Replica가 유지되는지 실험하기

```bash
# Server VM에서
sudo k3s kubectl get pods -l app=app2
sudo k3s kubectl delete pod <위에서_확인한_app2_Pod_이름>
sudo k3s kubectl get pods -l app=app2 -w
```

실습용 app2 Pod 하나를 삭제하면 ReplicaSet이 새 Pod를 만들어 다시 세 개를 유지합니다. `Ctrl+C`는 상태 관찰만 끝냅니다. Pod 교체 중에는 종료 중인 것까지 보여 일시적으로 줄 수가 세 개보다 많을 수 있으므로 Ready 상태를 함께 봅니다.

이 실험은 “내가 세 번 실행했다”와 “시스템에 세 개를 유지하라고 선언했다”의 차이를 보여 줍니다.

### 10. 증상별로 범위를 좁히기

| 증상 | 확인 순서 |
|---|---|
| `.110:80` 연결 실패 | VM 실행, 경로, 포트 노출, Traefik/ServiceLB 상태 |
| app1 대신 다른 앱 응답 | curl Host, Ingress 규칙, Service 이름과 Selector |
| 알 수 없는 Host에서 404 | catch-all 등록 여부, Controller 선택, 우선순위 |
| 502/503 등 backend 오류 | EndpointSlice, Pod Ready, 앱 리스닝 포트 |
| `ImagePullBackOff` | Image 이름·Tag·아키텍처, Registry 연결 |
| app2가 세 개 준비되지 않음 | Deployment replicas, Pod Events, VM 자원 |

```bash
# Server VM에서 실제 Pod 이름으로 바꾸어 실행
sudo k3s kubectl describe pod <pod-name>
sudo k3s kubectl logs <pod-name>
sudo k3s kubectl get events --sort-by=.metadata.creationTimestamp
```

### 11. 완료 후 설명할 질문

1. HTTP Host와 목적지 IP는 어떻게 다른가?
2. 요청 경로에 Deployment가 들어가지 않는 이유는 무엇인가?
3. Service가 Pod를 찾는 기준은 무엇인가?
4. Ingress 리소스만 있고 Controller가 없으면 어떻게 되는가?
5. app2 Pod 하나를 삭제하면 누가 복구하는가?
6. 이 구성이 VM 장애에도 안전한가?

다음: [Part 3 — Git 변경을 자동 배포로 연결하기](part3.md)

## Part 3: K3d, Argo CD and GitOps

이번 목표는 **GitHub의 배포 설정을 바꾸면 Argo CD가 감지하여 실행 중인 앱을 자동으로 바꾸게 만드는 것**입니다. Part 2의 Deployment와 Service 개념을 그대로 사용하되, 배포 설정을 적용하는 주체가 달라집니다.

### 1. 과제 원문에서 요구하는 것

기준은 `inception_of_things.pdf` v4.0, 본문 12–15쪽(PDF 13–16페이지)입니다.

| 항목 | 요구사항 |
|---|---|
| 환경 | VM에서 진행하며 이번 Part는 Vagrant 없이 구성 |
| Cluster | Docker와 K3d 사용 |
| 설치 자동화 | 평가 중 필요한 패키지와 도구를 설치할 Script 작성 |
| Namespace | Argo CD용 하나, 앱용 `dev` 하나 |
| GitHub | 공개 저장소, 저장소 이름에 팀원 login 포함 |
| 배포 | GitHub 설정으로 앱이 Argo CD를 통해 자동 배포 |
| 앱 | 구분 가능한 v1, v2 두 버전 |
| 검증 | 공개 GitHub에서 버전 변경 후 실제 앱 업데이트 확인 |

원문은 `wil42/playground` 앱을 제공하며 이 앱은 8888 포트를 사용합니다. 직접 만든 앱을 써도 되지만 공개 Docker Hub 저장소와 `v1`, `v2` Tag, 구분 가능한 두 버전이 필요합니다.

이 문서는 Argo CD Namespace 이름을 `argocd`로 사용합니다. 이는 원문 예시와 일치하는 선택이며, 원문에서 명시적으로 고정한 앱 Namespace 이름은 `dev`입니다.

### 2. K3s와 K3d는 무엇이 다른가?

```text
Part 1·2
실습 VM -> Vagrant Guest VM -> K3s -> Pod -> 앱 Container

Part 3
실습 VM -> Docker -> K3d가 만든 Node Container
                        -> K3s -> Pod -> 앱 Container
```

K3s는 Kubernetes 배포판입니다. K3d는 K3s Node를 Docker Container로 만들고 연결해 주는 도구입니다. K3d가 만든 Node 안에서는 K3s와 Container Runtime이 Pod를 관리합니다.

따라서 `docker ps`에는 K3d의 Node와 Load Balancer Container가 보이고, `kubectl get pods`에는 Kubernetes가 관리하는 앱이 보입니다. 두 명령이 같은 목록을 보여 줘야 하는 것은 아닙니다.

### 3. 먼저 도구와 실행 위치를 정하기

이 페이지의 명령은 별도 표시가 없으면 **실습 Linux VM**에서 실행합니다. Part 1의 Server VM이나 물리 Mac에서 실행하는 명령이 아닙니다.

설치 Script는 다음을 준비해야 합니다.

| 도구 | 쓰는 이유 |
|---|---|
| Docker Engine | K3d Node Container 실행 |
| K3d | Cluster 생성 및 관리 |
| kubectl | Kubernetes API 조회와 초기 설정 |
| Git, curl | 설정 저장소와 다운로드·HTTP 검증 |
| Argo CD | Cluster에 설치하는 배포 관리 구성요소 |

Argo CD CLI는 편리하지만 아래 흐름은 kubectl과 UI로 진행할 수 있습니다. 설치 Script는 대상 Linux 배포판과 CPU 아키텍처를 확인하고, 검증한 버전·다운로드 경로를 사용해야 합니다. 배포판이 정해지지 않은 상태에서 서로 다른 OS의 패키지 명령을 섞지 마세요.

```bash
docker version
k3d version
kubectl version --client
git --version
```

`docker version`은 Client뿐 아니라 Server에 연결되는지도 확인합니다. Docker 서비스가 실행되지 않거나 현재 사용자에게 접근 권한이 없으면 K3d 생성부터 실패합니다.

### 4. Cluster를 만들고 context 확인하기

```bash
k3d cluster create iot --servers 1 --agents 1
kubectl config current-context
kubectl get nodes -o wide
kubectl wait --for=condition=Ready nodes --all --timeout=180s
```

Server 1개와 Agent 1개는 학습용 선택이며 원문의 고정 Node 수 요구사항은 아닙니다. 작은 환경에서는 자원을 확인해 조절합니다.

Context는 kubectl이 사용할 Cluster와 인증 정보 등의 묶음입니다. 다른 과제의 Cluster에 명령을 보내지 않도록 `k3d-iot` context인지 확인합니다.

이 예시는 앱 검증에 `port-forward`를 사용하므로 Cluster 생성 때 앱 포트를 공개하지 않습니다. 별도 터미널 없이 계속 접근할 구성을 원한다면 뒤의 포트 공개 절을 참고합니다.

### 5. Namespace와 Argo CD 준비하기

Namespace는 같은 Cluster 안에서 리소스를 이름과 관리 범위로 나누는 단위입니다. VM을 추가하는 동작이 아니며, Namespace를 만들기만 했다고 네트워크 통신이 자동 차단되는 것도 아닙니다.

```bash
kubectl create namespace argocd --dry-run=client -o yaml | kubectl apply -f -
kubectl create namespace dev --dry-run=client -o yaml | kubectl apply -f -
```

`--dry-run=client -o yaml`은 만들 리소스를 YAML로 출력하고, `apply -f -`는 표준 입력에서 읽어 적용합니다. Namespace가 이미 있어도 같은 형태로 실행할 수 있습니다.

Argo CD 설치 Manifest를 `p3/confs/argocd-install.yaml`로 내려받아 검토하고 적용합니다. 아래 `stable` URL은 공식 시작 문서의 경로이며, 제출 자동화에서는 테스트한 Release Tag의 URL로 고정하세요.

```bash
curl -fL https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml \
  -o p3/confs/argocd-install.yaml
kubectl apply --server-side -n argocd -f p3/confs/argocd-install.yaml
kubectl rollout status deployment/argocd-server -n argocd --timeout=300s
kubectl get pods -n argocd
```

명령은 과제 저장소 루트에서 실행하며 `p3/confs`가 존재한다고 가정합니다. Server Deployment 준비뿐 아니라 Controller와 Repo Server 등 나머지 구성요소도 확인합니다. [Argo CD 설치 안내](https://argo-cd.readthedocs.io/en/stable/getting_started/)

`--server-side`는 큰 CRD를 설치할 때 client-side apply의 annotation 크기 제한을 피하기 위한 옵션입니다. 기존에 다른 도구로 설치한 Argo CD가 있다면 필드 관리 충돌을 확인한 뒤 처리합니다.

### 6. UI와 Application 포트를 구분하기

```bash
# 터미널 A: 실행한 채로 유지
kubectl port-forward -n argocd svc/argocd-server 8080:443
```

실습 VM에서 `https://localhost:8080`으로 관리 UI에 접근합니다. 기본 설치는 자체 서명 인증서를 사용합니다. 초기 사용자 이름은 `admin`이며 초기 비밀번호는 다음 Secret에서 읽습니다.

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath='{.data.password}' | base64 -d
```

이 값은 Git에 기록하지 않습니다. 로그인 후 초기 비밀번호를 변경합니다.

물리 컴퓨터의 브라우저를 쓰려면 그 컴퓨터의 localhost와 실습 VM의 localhost가 다르다는 점을 기억하세요. 실습 VM에 SSH로 접근 가능하다면 물리 컴퓨터에서 터널을 열 수 있습니다.

```bash
# 물리 컴퓨터에서: 사용자와 VM 주소를 실제 값으로 교체
ssh -N -L 8080:127.0.0.1:8080 <user>@<lab-vm-ip>
```

이후 물리 컴퓨터의 `https://localhost:8080` 요청이 실습 VM의 port-forward로 연결됩니다. [Argo CD 접속 안내](https://argo-cd.readthedocs.io/en/stable/getting_started/)

### 7. GitHub와 Docker Hub에는 무엇이 들어가는가?

```text
GitHub: deployment.yaml, service.yaml
    -> Argo CD가 읽음

Docker Hub: wil42/playground:v1, :v2
    -> Kubernetes Node의 Runtime이 내려받아 실행
```

Argo CD가 소스 코드를 자동으로 Image로 빌드하는 것은 아닙니다. 직접 만든 앱이라면 Image 빌드와 Registry push를 먼저 수행해야 합니다. 이 과제에서 확인하는 핵심은 이미 준비된 Image 버전을 Git으로 선택하여 배포하는 과정입니다.

PDF는 continuous integration이라는 표현을 사용하지만, 여기서 구성하는 Argo CD 중심 흐름은 배포·동기화(CD)에 해당합니다. 코드 빌드와 테스트 자동화(CI)는 별도 과정입니다.

공개 GitHub 저장소 이름은 예를 들어 `yourlogin-iot-config`처럼 정합니다. 아래에서는 이 저장소의 `apps/playground` 디렉터리만 Argo CD가 읽도록 구성합니다.

```text
yourlogin-iot-config/
└── apps/
    └── playground/
        ├── deployment.yaml
        └── service.yaml
```

과제 제출 저장소와 같은 저장소를 사용해도 되지만, 평가용 설치 Script와 Argo CD가 배포할 앱 Manifest의 경로를 명확히 구분합니다.

### 8. Git에 저장할 앱 Manifest

`apps/playground/deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: playground
  namespace: dev
spec:
  replicas: 1
  selector:
    matchLabels:
      app: playground
  template:
    metadata:
      labels:
        app: playground
    spec:
      containers:
        - name: playground
          image: wil42/playground:v1
          ports:
            - name: http
              containerPort: 8888
          readinessProbe:
            httpGet:
              path: /
              port: http
            initialDelaySeconds: 2
            periodSeconds: 5
```

`apps/playground/service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: playground
  namespace: dev
spec:
  selector:
    app: playground
  ports:
    - name: http
      port: 8888
      targetPort: http
```

Part 2와 같은 Label/Selector 관계입니다. Namespace와 앱 Image, 포트가 달라졌을 뿐입니다. 먼저 두 파일을 공개 GitHub 저장소의 `main` 브랜치에 commit·push합니다. 실제 기본 브랜치가 다르면 다음 Application의 `targetRevision`도 맞춥니다.

예시 Image를 사용할 때는 실습 VM 아키텍처와 Image 지원 아키텍처가 맞는지 확인합니다. Image가 내려받아지지 않으면 Pod Events에 원인이 표시됩니다.

### 9. Argo CD Application으로 두 세계 연결하기

다음은 **과제 제출 저장소**의 `p3/confs/application.yaml` 예시입니다. `repoURL`은 실제 공개 GitHub URL로 바꿉니다.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: playground
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/REPLACE_WITH_OWNER/REPLACE_WITH_LOGIN-iot-config.git
    targetRevision: main
    path: apps/playground
  destination:
    server: https://kubernetes.default.svc
    namespace: dev
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

| 필드 | 뜻 |
|---|---|
| `metadata.namespace: argocd` | 배포를 관리하는 Application 리소스의 위치 |
| `repoURL` | 설정을 가져올 Git 저장소 |
| `targetRevision` | 따라갈 브랜치·Tag·commit. 여기서는 main |
| `path` | 저장소 안의 앱 Manifest 디렉터리 |
| `destination.server` | Argo CD 자신이 실행되는 Cluster의 API |
| `destination.namespace: dev` | 실제 앱이 배포될 위치 |
| `automated` | Git 변경을 자동 동기화 |
| `prune: true` | Git에서 제거한 관리 리소스를 Cluster에서도 제거 |
| `selfHeal: true` | Cluster에서 수동 변경한 상태도 Git 기준으로 복구 |

원문은 자동 배포를 요구하므로 자동 동기화를 사용합니다. Prune과 Self-heal은 이 예시의 추가 정책 선택입니다. 특히 prune은 Git 파일 삭제를 실제 리소스 삭제로 연결하므로 의미를 이해하고 사용합니다. [Argo CD 자동 동기화 정책](https://argo-cd.readthedocs.io/en/stable/user-guide/auto_sync/)

```bash
kubectl apply -f p3/confs/application.yaml
kubectl get applications -n argocd
kubectl get deployments,pods,services -n dev
```

Application CRD는 Argo CD 설치 때 등록됩니다. `no matches for kind Application` 오류라면 Argo CD 설치와 현재 context부터 확인합니다.

### 10. Synced와 Healthy는 다른 질문이다

| 표시 | 답하는 질문 |
|---|---|
| `Synced` | 추적하는 Git 설정과 Cluster 리소스 설정이 일치하는가? |
| `Healthy` | 리소스가 기대한 실행 상태에 도달했는가? |
| curl 실제 응답 | 사용자가 기대하는 앱 버전이 응답하는가? |

설정이 같아도 Image를 내려받지 못하거나 앱이 시작하지 못하면 정상 서비스가 아닙니다. 세 단계 모두 확인합니다.

```bash
kubectl get application playground -n argocd \
  -o jsonpath='{.status.sync.status}{" "}{.status.health.status}{"\n"}'
kubectl rollout status deployment/playground -n dev --timeout=180s
```

자동 감지는 즉시 일어나는 것으로 가정하지 않습니다. Repository 확인 주기와 설정에 따라 시간이 걸릴 수 있습니다. Refresh는 Git 상태를 다시 확인하게 하는 동작이고 Sync는 그 상태를 반영하는 동작입니다.

### 11. v1에서 v2로 바꾸는 실험

먼저 별도 터미널에서 앱 접속 경로를 엽니다.

```bash
# 터미널 B: 실행한 채로 유지
kubectl port-forward -n dev svc/playground 8888:8888
```

```bash
# 실습 VM의 또 다른 터미널
curl http://localhost:8888/
```

응답에서 v1을 확인합니다. 그다음 **공개 설정 저장소의 로컬 checkout**에서 `deployment.yaml`의 Image를 `wil42/playground:v2`로 수정합니다.

```bash
git diff -- apps/playground/deployment.yaml
git add apps/playground/deployment.yaml
git commit -m "Deploy playground v2"
git push
```

이 단계에서 앱 YAML을 `kubectl apply`로 직접 배포하지 않습니다. Git 변경을 Argo CD가 적용하는지 검증해야 하기 때문입니다.

```text
GitHub main의 YAML이 v2로 변경
    -> Argo CD가 새 commit 발견
    -> 자동 Sync로 Deployment 변경
    -> Kubernetes가 새 Pod 생성
    -> 새 Pod Ready
    -> Service의 대상 갱신
    -> 실제 응답 v2
```

```bash
kubectl get application playground -n argocd \
  -o jsonpath='{.status.sync.revision}{"\n"}'
kubectl get deployment playground -n dev \
  -o jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'
kubectl get pods -n dev -w
```

Git commit 반영과 Deployment의 `:v2`를 확인한 뒤 `rollout status`로 완료를 기다립니다. `port-forward`는 연결했던 Pod가 교체되면 종료될 수 있으므로 터미널 B에서 다시 실행한 후 curl로 v2를 확인합니다.

### 12. port-forward와 지속적인 포트 공개

위 검증 경로는 다음과 같습니다.

```text
실습 VM localhost:8888
    -> 실행 중인 kubectl port-forward
    -> 선택된 playground Pod:8888
```

이는 Kubernetes API 연결을 이용하는 임시 검증 경로입니다. 터미널을 닫아도 접속되는 배포 구성을 원한다면 Cluster 생성 시 포트를 매핑하고 Ingress 또는 NodePort를 연결해야 합니다.

예를 들어 **처음 Cluster를 만드는 시점**에 다음 형태를 선택할 수 있습니다. 이미 `iot` Cluster가 있다면 같은 이름으로 이 명령을 추가 실행하는 절차는 아닙니다.

```bash
k3d cluster create iot --servers 1 --agents 1 -p '8888:80@loadbalancer'
```

이후 `dev` Namespace에 playground Service의 8888을 가리키는 Ingress를 추가해야 합니다. 포트 매핑만으로 앱이 연결되지는 않습니다.

```text
실습 VM :8888 -> K3d Load Balancer :80
    -> Traefik -> Ingress가 선택한 playground Service :8888
    -> Pod :8888
```

Cluster API 포트, Argo CD UI 포트, 앱 포트는 서로 다른 진입점입니다. [K3d 서비스 공개 문서](https://k3d.io/stable/usage/exposing_services/)

### 13. 제출 Script에 정리할 순서

```text
p3/
├── scripts/
│   ├── install-tools.sh
│   └── setup-cluster.sh
└── confs/
    ├── argocd-install.yaml
    └── application.yaml
```

1. 대상 OS·아키텍처·도구 버전을 확인하고 설치합니다.
2. Docker 연결을 확인합니다.
3. Cluster가 없으면 생성하고 올바른 context를 사용합니다.
4. Node Ready를 기다립니다.
5. Namespace와 Argo CD를 설치하고 준비를 기다립니다.
6. 실제 GitHub URL이 들어간 Application을 등록합니다.
7. 자동 배포 결과와 v1 응답을 확인합니다.
8. Git 변경으로 v2 응답까지 검증합니다.

공개 GitHub 저장소 준비와 Image 준비는 Script 실행 전의 선행 조건으로 적어 둡니다. 재실행 시 Cluster를 무조건 삭제하기보다 기존 상태를 검사하고 필요한 작업만 수행하도록 만듭니다.

### 14. 업데이트가 안 될 때 확인할 순서

| 증상 | 확인할 것 |
|---|---|
| Git 변경을 못 찾음 | push 여부, repoURL, targetRevision, path, 저장소 접근 |
| OutOfSync가 계속 유지 | 자동 Sync 설정, Application 오류와 Events |
| Synced지만 앱 실패 | Health, Pod Events, Image Tag·아키텍처, Ready |
| Deployment가 계속 v1 | 현재 context, Git commit, Argo CD 추적 revision |
| v2 Pod는 Ready인데 curl 실패 | port-forward 종료 여부, 실행 위치, Service 포트 |
| UI만 되고 앱이 안 됨 | 관리 UI 경로와 앱 경로를 구분하여 검사 |

```bash
kubectl describe application playground -n argocd
kubectl get events -n dev --sort-by=.metadata.creationTimestamp
kubectl describe pod -n dev <pod-name>
kubectl logs -n dev <pod-name>
```

### 15. 완료 후 설명할 질문

1. K3d Node Container와 그 안에서 관리되는 앱 Pod는 어떻게 다른가?
2. GitHub와 Docker Hub에서 각각 무엇을 가져오는가?
3. Application의 source와 destination은 무엇을 연결하는가?
4. `Synced`만 확인하면 충분하지 않은 이유는 무엇인가?
5. push 후 v2 응답까지 누가 어떤 일을 수행하는가?
6. port-forward를 종료하면 localhost 접속은 어떻게 되는가?

다음: [Bonus — Local GitLab](bonus.md)
