# Webserv

Webserv는 C++98로 직접 HTTP 서버를 구현하는 과제입니다.

브라우저가 URL을 입력했을 때 실제로 어떤 일이 일어나는지, 그리고 서버가 여러 Client의 연결을 어떻게 관리하는지 코드로 이해하는 것이 이 프로젝트의 핵심입니다.

## 이 과제의 핵심 질문

```text
브라우저는 서버에 어떤 형식으로 요청을 보내는가?
서버는 요청을 어떻게 읽고 해석하는가?
서버는 어떤 상태 코드와 Header를 응답해야 하는가?
여러 Client를 막히지 않고 동시에 어떻게 처리하는가?
파일과 CGI를 어떻게 HTTP Response로 바꾸는가?
```

## 한 문장으로 이해하기

```text
Socket으로 Client를 받고
Event Multiplexing으로 여러 연결을 관리한 뒤
HTTP Request를 파싱하고
정적 파일, Upload, CGI, Error를 HTTP Response로 제공한다.
```

## 과제의 구성

| 영역 | 배우는 내용 |
|---|---|
| HTTP | Request, Response, Method, Header, Status Code |
| Network | Socket, Bind, Listen, Accept, Non-blocking I/O |
| Event Loop | `poll`, `select`, `epoll`, `kqueue` |
| Server Design | Connection State, Read/Write Buffer, Timeout |
| Configuration | Port, Route, Root, Redirect, CGI, Upload |
| Process | CGI 실행, Pipe, Environment Variable |
| Testing | Browser, `curl`, `telnet`, NGINX 비교, Stress Test |

## 학습 순서

```text
1. HTTP 메시지 구조 이해
2. TCP Socket의 생명주기 이해
3. Non-blocking과 Event Multiplexing 이해
4. 최소 HTTP Response 보내기
5. Request Parser 만들기
6. Static File과 Error 처리
7. Configuration과 Route 적용
8. GET, POST, DELETE 구현
9. Upload와 CGI 구현
10. Browser와 Stress Test로 검증
```

## 문서 읽는 순서

1. [기초 개념](concepts.md)에서 Web, URL, Port, Socket, HTTP를 처음부터 확인합니다.
2. [Web의 역사와 Backend의 탄생](../../history-and-backend.md)에서 정적 Web, CGI, Backend의 발전을 이해합니다.
3. [Requirements](requirements.md)에서 과제의 제한과 필수 기능을 확인합니다.
4. [HTTP and Networking](http-networking.md)에서 Protocol과 Socket을 더 깊게 공부합니다.
5. [Implementation Guide](implementation.md)에서 기능을 어떤 계층으로 나눌지 이해합니다.
6. [Testing and Evaluation](testing.md)에서 동작을 어떻게 증명할지 확인합니다.

## 가장 중요한 설계 원칙

### 1. 서버는 절대로 멈추면 안 된다

한 Client의 느린 요청이나 큰 파일 때문에 다른 Client의 연결이 막히면 안 됩니다. Socket과 Pipe처럼 기다릴 수 있는 File Descriptor는 Non-blocking으로 사용하고, 하나의 Event Multiplexing 호출로 읽기와 쓰기를 관리해야 합니다.

### 2. 원하는 동작을 계층으로 나눈다

```text
OS / Socket
    ↓
Event Loop
    ↓
Connection State
    ↓
HTTP Parser
    ↓
Router / Configuration
    ↓
Response Builder
    ↓
File / Upload / CGI
```

문제가 생겼을 때 이 계층을 따라 어느 단계에서 잘못됐는지 좁히는 것이 중요합니다.

### 3. 동작보다 상태 전이가 먼저다

Client Socket 하나를 단순히 `recv`하고 `send`하는 함수로 생각하면 Non-blocking 서버를 만들기 어렵습니다. 각 연결이 현재 무엇을 기다리는지 명시해야 합니다.

```text
READING_HEADERS
      ↓
READING_BODY
      ↓
PROCESSING
      ↓
WRITING_RESPONSE
      ↓
CLOSING
```

## 완료 후 설명할 수 있어야 하는 것

- HTTP Request와 Response는 어떤 구조인가?
- TCP Socket의 `bind`, `listen`, `accept`는 각각 무엇을 하는가?
- 왜 Blocking I/O가 여러 Client를 처리하는 서버에 적합하지 않은가?
- `poll` 또는 동등한 함수는 무엇을 알려주는가?
- Request를 읽을 때 왜 한 번의 `recv`로 끝난다고 가정하면 안 되는가?
- `Content-Length`와 `Transfer-Encoding: chunked`는 Body를 어떻게 결정하는가?
- Service와 Route Configuration은 어떤 관계인가?
- CGI는 왜 별도의 Process와 Pipe를 사용하는가?
- 응답을 다 보낸 뒤 Socket을 언제 닫아야 하는가?

## 과제를 대하는 태도

이 프로젝트는 HTTP 문법을 외우는 과제가 아닙니다. 브라우저와 서버 사이에서 실제로 오가는 Byte가 어떤 의미를 가지는지 확인하고, 그 의미를 코드의 상태와 연결하는 과제입니다.

구현한 뒤에는 항상 다음 세 가지를 확인합니다.

```text
내 서버가 보낸 실제 Request/Response는 무엇인가?
상태 코드와 Header가 상황에 맞는가?
느린 Client나 비정상 Request에서도 다른 연결이 살아 있는가?
```
