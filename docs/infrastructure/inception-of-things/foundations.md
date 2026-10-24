# VM·Docker로 이어지는 네트워크

[네트워크 입문](network-journey.md)에서는 `www.naver.com`을 입력한 뒤 웹 페이지가 표시되는 과정을 따라갔습니다. 이제 같은 지식을 **내 PC에서 직접 실행하는 서버**에 적용합니다.

이 페이지의 질문은 “목적지가 인터넷의 서버가 아니라 VM이나 Container라면 무엇이 달라질까?”입니다. 주소, 경로, 포트, HTTP라는 원리는 그대로입니다. 프로그램이 실행되는 위치와 연결 구성이 달라집니다.

## 1. 멀리 있던 서버를 내 PC로 옮겨 보기

서버는 요청을 받아 응답하는 프로그램입니다. 인터넷의 큰 컴퓨터에서만 실행할 수 있는 것은 아닙니다. 지금 문서를 보여 주는 MkDocs도 내 컴퓨터에서 실행하는 웹 서버입니다.

```text
앞 페이지: 내 브라우저 -> 인터넷 -> 다른 곳의 웹 서버
이번 페이지: 내 브라우저 -> 내 PC의 문서 서버
```

같은 PC의 프로그램끼리 통신할 때도 IP와 Port를 사용할 수 있습니다. 이때 자주 등장하는 이름이 localhost입니다.

## 2. localhost와 Loopback: 나에게 보내는 요청

`localhost`는 보통 IPv4 `127.0.0.1` 또는 IPv6 `::1`로 해석됩니다. **현재 프로그램이 속한 네트워크 환경 자신**을 가리킵니다.

Loopback 인터페이스는 밖으로 나가지 않고 같은 네트워크 환경 안에서 데이터를 주고받는 가상 연결 지점입니다. Linux에서는 보통 `lo`라는 이름으로 보입니다.

```text
브라우저 -> 127.0.0.1:8000 -> 같은 PC의 MkDocs
```

공유기를 통해 인터넷으로 다녀올 필요가 없습니다. 앞 페이지에서 배운 인터페이스가 반드시 Wi-Fi나 랜카드 같은 물리 장치여야 하는 것은 아니라는 예입니다.

하지만 PC 안에 VM을 만들면 “나”가 무엇인지 다시 구분해야 합니다.

## 3. VM: 내 PC 안의 별도 컴퓨터 환경

**운영체제(OS)**는 CPU, 메모리, 장치와 프로그램 실행을 관리합니다. **VM**은 가상 하드웨어 위에 별도의 운영체제를 실행하는 환경입니다. 그 가상화를 수행하는 기반을 Hypervisor라고 부릅니다.

VM을 실행하는 기반을 **Host**, VM 안의 환경을 **Guest**라고 부릅니다. 여러 겹으로 구성하면 어떤 층을 기준으로 말하는지 함께 확인해야 합니다.

```text
물리 PC
├── PC의 브라우저
└── Guest VM
    ├── Guest 운영체제
    └── Guest 안의 웹 서버
```

VM 안의 프로그램도 자신의 파일, 프로세스, 네트워크 설정을 가진 환경에서 실행됩니다. PC의 브라우저가 `localhost`를 요청하면 PC 자신을 찾습니다. VM 안의 서버를 찾으려면 **VM으로 이어지는 주소와 경로**가 필요합니다.

## 4. VM의 가상 인터페이스와 연결 방식

VM에 진짜 랜카드를 꽂지 않아도 통신할 수 있습니다. 가상화 기반이 가상 네트워크 장치와 연결을 제공하고, Guest OS는 이를 자신의 인터페이스로 사용합니다.

```text
Guest 프로그램
    -> Guest OS의 가상 인터페이스
    -> 가상 네트워크
    -> Host 또는 다른 VM 또는 외부 네트워크
```

어디까지 연결되는지는 구성에 달려 있습니다.

| 구성 | 어떤 연결을 만드는가? |
|---|---|
| NAT | Guest가 외부에 요청할 때 주소 변환을 거치는 연결 |
| Host-only | Host와 Guest들 사이의 전용 로컬 네트워크 |
| Bridged | Guest를 Host가 사용하는 외부 네트워크에 연결하는 구성 |

하나의 VM에 NAT용 인터페이스와 실습용 인터페이스를 함께 둘 수도 있습니다. 그러면 외부 다운로드에 사용하는 주소와 다른 VM이 찾아오는 주소가 달라집니다.

Vagrant는 VM 환경의 생성을 관리하는 도구이고, Provider는 실제 가상화 기반입니다. Vagrant의 Private Network 설정이 구체적으로 어떻게 구현되는지는 Provider에 따라 달라집니다. [Vagrant Private Networks](https://developer.hashicorp.com/vagrant/docs/networking/private_network)

## 5. Part 1의 IP 두 개를 이제 읽어 보기

```text
Server VM: 192.168.56.110
Agent VM:  192.168.56.111
```

앞 페이지의 사설 IP, Subnet, 인터페이스 개념을 적용할 수 있습니다. 두 VM이 실제로 같은 실습 네트워크에 연결되어 있고 적절한 Subnet과 경로가 설정되어야 합니다. 주소만 비슷하게 적는 것으로 연결이 완성되지는 않습니다.

```text
Agent VM
    -> 실습용 인터페이스
    -> 가상 네트워크
    -> Server VM의 192.168.56.110:6443
```

6443은 K3s API 연결에 사용하는 포트입니다. Agent의 localhost:6443은 Server가 아니라 Agent 자신을 가리킵니다. SSH가 되는 주소·포트와 K3s가 통신하는 주소·포트도 구분해야 합니다.

## 6. 서버는 어느 주소에서 요청을 받을까? Bind

프로그램이 실행 중이라는 사실과 외부에서 접속 가능하다는 사실은 다릅니다. 서버는 어느 로컬 주소·포트에서 받을지 정합니다. 이 설정을 bind라고 합니다.

| IPv4 바인딩 예시 | 받는 위치 |
|---|---|
| `127.0.0.1:8000` | 같은 네트워크 환경의 Loopback |
| `192.168.56.110:8000` | 해당 로컬 주소 |
| `0.0.0.0:8000` | 모든 로컬 IPv4 인터페이스 |

Guest 서버가 `127.0.0.1`에만 바인딩되어 있다면 Host에서 Guest IP로 접근해도 해당 서버가 연결을 받지 않습니다. `0.0.0.0`은 모든 로컬 주소에서 받는다는 설정이며 다른 머신이 찾아갈 목적지 주소는 실제 IP입니다.

전체 인터페이스에서 받더라도 경로와 방화벽이 접근을 허용해야 합니다. “서버 실행”, “올바른 주소에서 대기”, “네트워크 연결 허용”을 나누어 생각하세요.

## 7. NAT 뒤의 서버를 찾아가는 Port Forwarding

앞 페이지에서는 내부 PC가 시작한 연결의 응답을 NAT가 돌려주는 과정을 봤습니다. 이제 외부에서 새 요청을 시작해 내부 서버에 도달시키려면 어떻게 할까요?

**Port Forwarding**은 특정 주소·포트로 들어온 연결을 지정한 내부 목적지에 전달하는 설정입니다.

```text
Host의 8080
    -> 포트 전달 규칙
    -> Guest의 80
    -> Guest 웹 서버
```

8080과 80은 각각 다른 환경의 포트이므로 같을 필요가 없습니다. 전달 규칙이 있어도 Guest의 80에서 실제 프로그램이 요청을 받아야 합니다.

포트 전달은 외부에서 들어오는 경로를 구성하는 한 방법입니다. Host-only처럼 직접 도달 가능한 VM 주소가 있다면 그 주소로 접속하는 구성을 사용할 수도 있습니다.

## 8. VM에서 Container로: 왜 실행 환경을 또 나눌까?

프로그램을 실행한 대상을 **프로세스(Process)**라고 부릅니다. 여러 앱을 한 OS에서 실행하면 파일, 라이브러리, 설정이 서로 영향을 줄 수 있습니다. 앱별 실행 환경을 분리하고 필요한 파일을 함께 묶고 싶어서 Container를 사용합니다.

**커널(Kernel)**은 OS의 핵심으로 자원과 프로세스를 관리합니다. VM은 별도 커널을 실행하지만, Linux Container들은 기반 Linux 커널을 공유하면서 프로세스가 보는 환경을 격리합니다.

```text
VM:        가상 하드웨어 -> Guest 커널 -> 앱 프로세스
Container: 공유 커널 위의 격리된 환경 -> 앱 프로세스
```

Docker는 Image를 만들고 Container를 실행·관리하는 도구입니다. Mac에서 Linux Container를 실행할 때는 Linux VM 계층이 사용됩니다. [Docker Container 개념](https://docs.docker.com/get-started/docker-concepts/the-basics/what-is-a-container/)

“작은 VM”이라는 비유만 기억하면 커널과 네트워크 공유 관계를 놓치기 쉬우므로, 프로세스와 격리라는 관점도 함께 기억하세요.

## 9. Image, Container, Registry, Volume

| 개념 | 무엇인가? | 과제와의 연결 |
|---|---|---|
| Image | 실행 파일과 의존 파일, 기본 실행 설정을 담은 템플릿 | 어떤 앱을 실행할지 선택 |
| Container | Image를 바탕으로 만든 실행 환경 | 실제 앱 프로세스가 실행 |
| Registry | Image를 저장하고 가져오는 곳 | Docker Hub에서 앱 Image 확보 |
| Tag | Image를 가리키는 이름표 | v1, v2를 선택 |
| Volume | Container와 별도로 데이터를 보관하는 방식 | 실행 환경 교체와 데이터 지속성을 구분 |

Image 하나로 여러 Container를 만들 수 있습니다. 같은 Image를 쓴다고 실행 중인 파일 변경까지 자동 공유되는 것은 아닙니다. Container 내부에만 기록한 변경은 Container 삭제 시 사라질 수 있습니다.

Tag는 변경 가능한 이름표입니다. Part 3에서는 v1과 v2가 실제로 구분되는 Image를 가리키도록 준비하고, 어떤 버전을 사용할지 Git의 설정에 기록합니다.

## 10. Container의 localhost와 네트워크

일반적인 격리 구성에서는 Container가 사용하는 인터페이스, 주소, 경로의 환경이 분리됩니다. 이를 네트워크 Namespace라고 부릅니다. 뒤에서 배우는 Kubernetes Namespace와는 다른 개념입니다.

```text
Container A의 localhost -> A의 네트워크 환경
Container B의 localhost -> B의 네트워크 환경
```

서로 다른 네트워크 환경이면 둘 다 내부 80번 포트를 사용할 수 있습니다. 반면 Host의 같은 주소·포트에 두 서버를 동시에 공개하려고 하면 일반적으로 충돌합니다.

Docker의 사용자 정의 Bridge Network에서는 같은 네트워크에 참여한 Container들이 이름으로 서로 찾을 수 있습니다. DNS로 이름을 주소에 연결하는 원리가 내부 네트워크에도 사용되는 것입니다. 별도 설정 없이 모든 Container 이름이 어디에서나 해석되는 것은 아닙니다.

Host 네트워크를 공유하는 등 다른 구성도 있으므로, localhost를 볼 때는 “지금 어느 네트워크 환경을 공유하는가”를 확인합니다.

## 11. Docker의 포트 공개를 요청 경로로 읽기

Docker에서 `127.0.0.1:8080:80`이라는 포트 공개 설정을 보면 다음처럼 읽습니다.

```text
Docker Host의 127.0.0.1:8080
    -> Container의 80
    -> 실제 웹 앱
```

앞의 IP는 Host에서 받을 주소이고, 두 포트는 Host와 Container 각각의 포트입니다. 이 구성에서는 Host 내부에서 접근하는 경로를 엽니다. 다른 컴퓨터가 Host의 외부 IP:8080으로 직접 접속하도록 여는 설정과는 다릅니다.

Dockerfile의 `EXPOSE 80`은 포트에 대한 정보이며, 이것만으로 Host의 포트 전달이 생성되는 것은 아닙니다. [Docker 포트 공개 문서](https://docs.docker.com/get-started/docker-concepts/running-containers/publishing-ports/)

이제 VM과 Container 어느 쪽이든 “외부 진입점과 내부 앱 포트를 연결한다”는 같은 질문을 할 수 있습니다.

## 12. Kubernetes에 적용하면 주소가 왜 더 생길까?

Kubernetes는 여러 머신에서 앱의 실행 상태를 관리합니다. 참여한 머신을 **Node**, 함께 관리되는 Node들의 집합을 **Cluster**라고 합니다. 앱 Container는 **Pod**라는 단위 안에서 실행합니다.

같은 Pod의 Container들은 네트워크 환경을 공유하므로 localhost로 통신할 수 있습니다. 다른 Pod의 localhost는 그 Pod 자신입니다.

| 주소 | 찾는 대상 |
|---|---|
| Node IP | Cluster에 참여한 머신 |
| Pod IP | 앱이 실행되는 Pod |
| Service IP | 선택한 Pod 그룹으로 연결되는 안정적인 진입점 |

Pod는 교체될 수 있기 때문에 IP를 고정 서버 주소처럼 취급하기 어렵습니다. Service는 이 변화 앞에 안정적인 접점을 제공합니다. 일반적인 ClusterIP Service의 주소는 특정 물리 랜카드의 주소가 아니라 전달 규칙으로 구현되는 가상 주소입니다.

**Overlay Network**는 기존 네트워크 위에 논리적인 네트워크를 만드는 방식입니다. 기본 K3s의 Flannel VXLAN 구성에서는 Pod 통신을 Node 간 전달용 Packet 안에 담아 운반합니다. 앞 페이지에서 배운 캡슐화가 여기서 다시 사용됩니다.

## 13. DNS와 HTTP Host를 Part 2에 적용하기

인터넷의 `www.naver.com` 대신 실습용 app1과 app2를 생각해 봅시다.

```text
목적지 IP:Port는 같음: 192.168.56.110:80

Host: app1.com -> app1
Host: app2.com -> app2
그 외         -> app3
```

DNS나 로컬 이름 설정은 이름을 주소에 연결합니다. HTTP Host는 그 주소에 연결한 뒤 원하는 앱을 구분합니다. 테스트 도구에서 목적지 IP와 Host를 따로 지정하면 공개 DNS를 바꾸지 않고도 이 라우팅을 확인할 수 있습니다.

Part 2의 **Ingress**는 이 연결 규칙이고, **Ingress Controller**는 실제 요청을 받아 규칙을 적용하는 프로그램입니다. 앞 페이지의 Reverse Proxy가 과제에서는 이런 형태로 등장합니다.

```text
Client -> VM의 IP:80
    -> Ingress Controller가 Host 확인
    -> 선택한 Service의 backend
    -> 준비된 Pod의 웹 앱
```

Deployment는 Pod의 배포 상태를 관리하며 이 요청을 통과시키는 Proxy가 아닙니다. 관리 흐름과 요청 흐름은 Part 2에서 더 자세히 나눕니다.

## 14. Part 3에서는 무엇이 한 겹 더 들어갈까?

K3s는 Kubernetes 배포판이며 기본적으로 containerd라는 실행 도구를 사용해 앱 Container를 실행합니다. Part 1·2의 K3s Node에 반드시 Docker Engine을 따로 설치해야 하는 것은 아닙니다.

Part 3의 K3d는 **Node 자체를 Docker Container로 만드는 도구**입니다.

```text
실습 VM
└── Docker
    └── K3d의 Node Container
        └── K3s와 Runtime이 관리하는 앱 Pod
```

Node를 실행하는 Container와 Node가 관리하는 앱의 Container는 서로 다른 층입니다. 따라서 바깥에서 앱에 접속하려면 이 환경에 맞는 진입 경로를 만들어야 합니다.

또한 GitHub는 배포 설정을 제공하고, Image Registry는 실행할 Image를 제공합니다. 이 둘과 통신하려면 앞에서 배운 DNS, 경로, HTTPS 등이 다시 필요합니다. 네트워크 지식은 앱에 접속할 때뿐 아니라 도구를 설치하고 배포할 때도 사용됩니다.

## 15. 같은 PC 안에서 요청 위치를 구분하기

과제는 전체 작업을 VM에서 진행하도록 요구합니다. 이 문서의 환경 구분을 적용하면 다음과 같습니다.

```text
물리 PC의 localhost      -> 물리 PC
실습 VM의 localhost      -> 실습 VM
Vagrant Guest의 localhost -> 해당 Guest
Pod의 localhost          -> 해당 Pod의 네트워크 환경
```

같은 IP 문자열이라도 명령을 실행한 환경이 달라지면 다른 곳을 가리킵니다. 접속이 안 될 때 포트를 바꾸기 전에 **어디에서 어디로 연결하려는지** 먼저 그려 보세요.

## 16. 일반 지식에서 과제의 질문으로

| 이미 배운 개념 | 과제에서 더 알아볼 질문 |
|---|---|
| Interface, IP, Subnet, Route | Part 1의 Server와 Agent는 어떤 경로로 통신할까? |
| TCP, Port, TLS | K3s API 주소와 SSH 접속은 어떻게 다를까? |
| DNS, HTTP Host, Proxy | Part 2에서 같은 IP의 앱을 어떻게 구분할까? |
| Process, Container, 주소 격리 | Pod의 실행과 접속을 왜 나누어 관리할까? |
| 이름 해석과 HTTPS | Part 3에서 Git 설정과 Image를 어떻게 가져올까? |

이후 [학습 목표와 지도](subject.md)에서 Kubernetes가 필요한 이유와 목표를 먼저 확인합니다. 각 Part는 **요구사항 → PC 안의 구조 → 배울 수 있는 개념** 순서로 읽습니다. 명령어와 YAML은 이해가 된 뒤 [구현 참고](implementation-reference.md)에서 찾아볼 수 있습니다.

## 17. 이해를 확인하기

1. PC의 브라우저에서 localhost를 열면 VM 서버로 바로 연결되지 않는 이유는 무엇인가?
2. Guest 서버가 Loopback에서만 기다리면 어떤 접속이 제한되는가?
3. 외부 인터넷이 되는 VM에 외부에서 새 연결을 시작하는 것은 왜 별도 문제인가?
4. `8080:80`의 두 포트는 각각 누구의 포트인가?
5. 같은 Pod 안의 localhost와 다른 Pod의 localhost는 어떻게 다른가?
6. Node IP, Pod IP, Service IP가 각각 필요한 이유는 무엇인가?

다음: [학습 목표와 지도 — 네트워크에서 Kubernetes로](subject.md)
