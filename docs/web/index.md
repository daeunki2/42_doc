# Web

웹 분야에서는 Client와 Server가 Network를 통해 데이터를 주고받는 원리와, 그 위에서 동작하는 Protocol 및 Application을 학습합니다.

현재는 HTTP Server를 직접 구현하는 `Webserv` 과제를 정리하고 있습니다.

## 학습 흐름

```text
HTTP Protocol
	↓
TCP Socket
	↓
Event-driven Server
	↓
Request / Response
	↓
Static File / CGI / Upload
```

## 먼저 읽어볼 문서

- [Web의 역사와 Backend의 탄생](history-and-backend.md): 정적 Web, CGI, Backend, Database, API의 발전 흐름

## Projects

- [Webserv](projects/webserv/index.md): C++98로 HTTP Server 구현
