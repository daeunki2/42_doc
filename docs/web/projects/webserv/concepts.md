# Webserv를 위한 기초 개념

이 문서는 Web을 처음 접하는 사람을 위한 출발점입니다. 아직 Browser, URL, Server, Port가 무엇인지 몰라도 괜찮습니다. 이 페이지의 목표는 Webserv 과제를 읽을 때 등장하는 단어를 서로 연결하는 것입니다.

## 1. 우리가 만들 것은 무엇인가

Webserv는 Browser 같은 Client가 보낸 요청을 받아 적절한 응답을 보내는 프로그램입니다.

```text
사용자
  ↓ 주소 입력 또는 링크 클릭
Browser
  ↓ HTTP Request
Webserv
  ↓ HTTP Response
Browser가 화면에 표시
```

예를 들어 사용자가 Browser에 다음 주소를 입력한다고 합시다.

```text
http://localhost:8080/index.html
```

Browser는 이 문자열을 보고 Webserv에게 다음 의미의 요청을 보냅니다.

> 이 컴퓨터의 8080번 입구에서 `index.html`이라는 Resource를 달라.

Webserv는 파일을 찾아 다음과 같이 응답합니다.

> 파일을 찾았다. 이것은 HTML 문서이고, 내용은 다음과 같다.

## 2. Web이란 무엇인가

Web은 서로 연결된 문서와 Application을 HTTP로 주고받는 시스템입니다. Web은 Internet과 같은 말이 아닙니다.

```text
Internet
  = 컴퓨터들이 서로 통신할 수 있게 연결된 네트워크

Web
  = Internet 위에서 HTTP와 URL을 사용해 Resource를 주고받는 시스템
```

Internet 위에는 Web 외에도 SSH, Email, DNS, File Transfer 같은 여러 Protocol이 동작합니다. Webserv는 그중 HTTP를 사용하는 Server입니다.

## 3. Client와 Server

Client와 Server는 특정 컴퓨터의 이름이 아니라 역할입니다.

```text
Client
  = 서비스를 요청하는 쪽

Server
  = 요청을 받아 서비스를 제공하는 쪽
```

Browser는 Web을 사용할 때 Client입니다. 같은 컴퓨터에서 실행되더라도 역할은 나뉩니다.

```text
내 Mac
  ├── Browser: Client
  └── Webserv: Server
```

Server는 반드시 별도의 거대한 컴퓨터일 필요가 없습니다. 이 과제에서는 자신의 컴퓨터에서 Webserv를 실행하고 Browser로 접속합니다.

## 4. Resource와 URL

Web에서 Resource는 요청할 수 있는 대상입니다.

- HTML 문서
- CSS 파일
- JavaScript 파일
- 이미지
- 동적으로 생성된 결과
- Upload된 파일

URL은 Resource의 위치를 사람이 읽을 수 있는 형식으로 표현합니다.

```text
http://localhost:8080/docs/index.html?lang=ko
└─┬─┘ └──────┬──────┘└──┬────────┘└──┬────┘
 scheme      host       path       query
```

### Scheme

`http`는 어떤 Protocol로 통신할지 나타냅니다. `https`는 HTTP를 TLS로 보호하는 방식입니다.

### Host

`localhost`는 “현재 컴퓨터”를 의미하는 특수한 Host 이름입니다. 실제 Website에서는 `example.com` 같은 Domain Name이 들어갑니다.

### Port

`8080`은 어떤 프로그램의 입구로 연결할지 나타냅니다. 한 컴퓨터에서 여러 Server가 실행될 수 있으므로 Port로 통신 대상을 구분합니다.

### Path

`/docs/index.html`은 Server에게 어떤 Resource를 원하는지 알려주는 경로입니다. 이 문자열이 Server의 실제 디스크 경로와 반드시 같은 것은 아닙니다.

### Query

`?lang=ko` 같은 부분은 Resource에 전달하는 추가 조건입니다. CGI나 Backend가 검색어, 필터, 옵션을 읽을 때 사용합니다.

## 5. Domain Name과 IP Address

컴퓨터는 서로 통신할 때 IP Address를 사용하지만, 사람은 Domain Name을 사용하기 편합니다.

```text
example.com
     ↓ DNS 조회
93.184.216.34
```

`localhost`는 보통 `127.0.0.1`이라는 Loopback IP로 해석됩니다. Loopback은 네트워크 밖으로 나가지 않고 자신의 컴퓨터로 돌아오는 주소입니다.

Webserv를 테스트할 때 다음 두 주소는 같은 컴퓨터를 가리킬 수 있습니다.

```text
http://localhost:8080/
http://127.0.0.1:8080/
```

## 6. Port는 왜 필요한가

IP Address가 컴퓨터를 찾는 주소라면, Port는 그 컴퓨터 안에서 어떤 프로그램을 찾을지 나타내는 번호입니다.

```text
IP Address: 어느 컴퓨터인가
Port:       그 컴퓨터의 어느 서비스인가
```

```text
127.0.0.1:8080 -> Webserv
127.0.0.1:3000 -> 다른 개발 Server
```

Server가 `127.0.0.1:8080`에 `bind`한다는 것은 그 주소와 Port를 자신의 통신 입구로 등록한다는 뜻입니다.

## 7. TCP는 무엇인가

TCP는 두 프로그램 사이에서 Byte를 순서대로 전달하는 Transport Protocol입니다. HTTP는 보통 TCP 연결 위에서 전달됩니다.

```text
HTTP Message
      ↓
TCP Byte Stream
      ↓
Network
```

TCP는 “이것이 한 Request의 끝이다”라는 경계를 보장하지 않습니다. 따라서 Webserv가 받은 Byte를 보고 HTTP Header의 끝과 Body의 길이를 판단해야 합니다.

```text
한 번에 보낸 Request
      ↓ TCP
여러 번 나누어 도착할 수 있음
```

이것이 Webserv에서 Buffer와 Partial Read가 필요한 이유입니다.

## 8. Socket은 무엇인가

Socket은 프로그램이 Network 통신을 사용하기 위한 운영체제의 입구입니다. Unix에서는 Socket도 File Descriptor라는 숫자로 다뤄집니다.

```text
Webserv
  └── Socket File Descriptor 5
          ↓
      TCP 연결
          ↓
      Browser
```

### Server Socket의 단계

```text
socket()
  = 통신용 Socket 생성

bind()
  = IP와 Port 연결

listen()
  = 새로운 Client 연결 대기

accept()
  = 대기 중인 Client를 실제 통신 Socket으로 받음
```

`listen` 중인 Socket과 Client와 통신하는 Socket은 다릅니다.

```text
Listen Socket
  = 새 손님을 받는 입구

Client Socket
  = 특정 손님과 대화하는 연결
```

## 9. HTTP는 어떤 대화인가

HTTP는 Client와 Server가 정해진 형식으로 주고받는 Text 기반 Protocol입니다.

### Request

```http
GET /index.html HTTP/1.1
Host: localhost:8080
Connection: close

```

읽는 방법:

- `GET`: 원하는 동작
- `/index.html`: 대상 Resource
- `HTTP/1.1`: Protocol Version
- `Host`: 어떤 Host를 요청했는지 나타내는 Header
- 마지막 빈 줄: Header가 끝났다는 표시

### Response

```http
HTTP/1.1 200 OK
Content-Type: text/html
Content-Length: 18
Connection: close

<h1>Hello</h1>
```

읽는 방법:

- `200 OK`: 처리 결과
- `Content-Type`: Body의 종류
- `Content-Length`: Body의 Byte 길이
- 빈 줄 뒤: 실제 Body

Webserv의 첫 번째 핵심 작업은 이 형식을 정확히 읽고 만드는 것입니다.

## 10. Method는 동작의 종류다

Method는 Request가 Server에게 원하는 동작을 나타냅니다.

```text
GET    = 읽어 달라
POST   = 데이터를 보내 처리하거나 생성해 달라
DELETE = 삭제해 달라
```

Method 이름만 보고 실제 작업이 자동으로 정해지는 것은 아닙니다. Configuration이 해당 Route에서 어떤 Method를 허용하는지도 확인해야 합니다.

## 11. Status Code는 결과의 종류다

Response의 Status Code는 Server가 Request를 어떻게 처리했는지 알려줍니다.

```text
2xx = 성공
3xx = 다른 위치로 이동하거나 추가 처리 필요
4xx = Client Request가 잘못됨
5xx = Server가 처리하지 못함
```

예를 들어 파일이 없을 때 `404 Not Found`, 허용하지 않은 Method일 때 `405 Method Not Allowed`를 보내는 이유는 Client가 다음 행동을 판단할 수 있게 하기 위해서입니다.

## 12. HTTP Header는 설명 정보다

Header는 Request나 Response의 본문이 아니라, 본문을 어떻게 해석해야 하는지 설명하는 Metadata입니다.

```text
Content-Type: 무엇의 데이터인가
Content-Length: Body가 몇 Byte인가
Host: 어느 Host를 요청했는가
Connection: 연결을 어떻게 처리할 것인가
Location: Redirect할 주소는 어디인가
```

Header와 Body 사이에는 반드시 빈 줄이 있습니다.

```text
Headers

Body
```

## 13. Web Server는 무엇을 하는가

Web Server는 단순히 HTML을 출력하는 프로그램이 아니라, 외부 Client와 내부 Resource 사이를 관리하는 입구입니다.

```text
Client 연결 수락
      ↓
Request 수신
      ↓
Request 해석
      ↓
Route 선택
      ↓
파일/CGI/Upload 처리
      ↓
Response 생성
      ↓
Client에 전송
```

Webserv 과제에서는 이 모든 과정을 직접 구현합니다.

## 14. Static File과 Dynamic Content

### Static File

이미 디스크에 존재하는 파일을 그대로 반환합니다.

```text
GET /index.html
      ↓
디스크에서 index.html 읽기
      ↓
HTML Response
```

### Dynamic Content

Request에 따라 프로그램이 결과를 계산합니다.

```text
POST /guestbook
      ↓
CGI 또는 Backend 실행
      ↓
입력 저장/계산
      ↓
새로운 HTML 또는 Data 생성
```

Webserv는 Static File을 제공해야 하고, CGI를 통해 Dynamic Content와의 연결도 지원해야 합니다.

## 15. CGI는 왜 필요한가

CGI는 Web Server가 외부 프로그램에게 일을 맡기는 표준적인 연결 방식입니다.

```text
Webserv
  ├── 환경 변수로 Request 설명
  ├── stdin으로 Body 전달
  └── stdout에서 결과 수신
          ↓
      CGI Program
```

예를 들어 방명록에서 사용자가 메시지를 보내면 Webserv는 CGI Program에 데이터를 전달하고, CGI는 저장하거나 HTML을 출력할 수 있습니다.

Webserv가 CGI를 직접 실행하는 과정에서 다음 운영체제 개념을 배웁니다.

- `fork`: Process 복제
- `execve`: 다른 프로그램 실행
- `pipe`: Process 사이의 데이터 통로
- `dup2`: 표준 입력/출력을 Pipe에 연결
- Environment Variable: Request 정보를 전달하는 방법

## 16. 여러 Client를 어떻게 처리하는가

순차적인 Server는 한 Client의 응답이 끝날 때까지 다음 Client를 기다릴 수 있습니다.

```text
나쁜 흐름
Client A 처리 시작
  ↓ A가 느림
Client B는 기다림
```

Webserv는 Non-blocking I/O와 Event Multiplexing을 사용합니다.

```text
Event Loop
  ├── A가 읽을 수 있는가?
  ├── B가 쓸 수 있는가?
  ├── 새 연결이 있는가?
  └── CGI Pipe가 준비됐는가?
```

`poll`, `select`, `kqueue`, `epoll`은 여러 File Descriptor 중 지금 처리해도 Block되지 않을 것을 알려주는 도구입니다.

## 17. 이 개념들이 과제 요구사항으로 바뀌는 과정

```text
Browser가 접속해야 한다
      ↓
Socket, bind, listen, accept 필요

여러 Client를 처리해야 한다
      ↓
Non-blocking과 Event Multiplexing 필요

파일을 제공해야 한다
      ↓
Path, open, read, Content-Type 필요

사용자 입력을 처리해야 한다
      ↓
POST, Body, Upload 필요

동적 페이지가 필요하다
      ↓
CGI, fork, execve, pipe 필요

안정적으로 동작해야 한다
      ↓
Buffer, Timeout, Error, Stress Test 필요
```

## 18. 여기까지 이해했는지 확인하기

다음 질문에 답할 수 있다면 Webserv의 출발점을 잡은 것입니다.

1. Browser와 Webserv는 각각 Client와 Server 중 어느 역할인가?
2. URL의 Host, Port, Path는 각각 무엇을 가리키는가?
3. IP Address와 Port는 어떻게 다른가?
4. Socket과 File Descriptor는 어떤 관계인가?
5. TCP가 Request 경계를 보장하지 않는다는 말은 무슨 뜻인가?
6. HTTP Request의 Header와 Body는 어떻게 구분하는가?
7. Static File과 Dynamic Content의 차이는 무엇인가?
8. CGI는 Web Server와 외부 프로그램을 어떻게 연결하는가?
9. 왜 한 번의 `recv()`만으로 Request를 처리하면 안 되는가?
10. 왜 Event Multiplexing이 필요한가?
