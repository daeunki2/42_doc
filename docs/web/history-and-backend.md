# Web의 역사와 Backend의 탄생

Webserv는 사용자의 요청에 응답하는 HTTP Server를 직접 만드는 과제입니다. 처음에는 단순히 파일을 전달하는 프로그램처럼 보이지만, Web의 발전 과정을 따라가면 정적 파일, CGI, Backend, Database, API가 왜 등장했는지 자연스럽게 이해할 수 있습니다.

## 1. Web은 무엇인가

World Wide Web은 Internet이라는 네트워크 위에서 동작하는 정보 시스템입니다.

```text
Internet
  └── 여러 컴퓨터가 연결된 네트워크

Web
  └── HTTP로 Resource를 요청하고 전달하는 시스템
```

Web을 구성하는 핵심 요소는 다음과 같습니다.

- Client: Browser처럼 요청을 보내는 프로그램
- Server: 요청을 받아 Resource를 반환하는 프로그램
- HTTP: Client와 Server가 대화하는 규칙
- URL: 어떤 Resource를 요청할지 표현하는 주소
- HTML: Browser가 화면으로 표현하는 문서 형식

```text
Browser
   │  HTTP Request
   ▼
Web Server
   │  HTTP Response
   ▼
HTML / CSS / Image / Script
```

Webserv 과제는 이 구조에서 `Web Server`의 내부를 직접 구현하는 경험입니다.

## 2. 초기 Web: 정적 문서의 전달

초기 Web의 기본 모델은 매우 단순했습니다. Server의 디스크에 저장된 HTML 파일을 Client가 요청하면 Server가 파일을 읽어 그대로 전달했습니다.

```text
GET /index.html
       │
       ▼
Server Disk: index.html
       │
       ▼
HTTP Response: HTML 파일 내용
```

이런 방식을 **Static Web**이라고 합니다. 요청이 여러 번 들어와도 Server가 전달하는 내용은 파일이 바뀌지 않는 한 같습니다.

### 정적 Web의 장점

- 구조가 단순합니다.
- 응답이 빠릅니다.
- Server가 처리할 계산이 적습니다.
- 파일과 URL의 관계를 이해하기 쉽습니다.

### 정적 Web의 한계

하지만 다음과 같은 기능은 파일만 전달해서 만들기 어렵습니다.

- 사용자가 남긴 방명록 저장
- 로그인한 사용자별 화면
- 검색 결과 생성
- 현재 시간이나 날씨 표시
- Form으로 보낸 데이터 처리
- Database에서 글 목록 조회

문서는 미리 만들어져 있지만, 사용자의 입력에 따라 내용이 바뀌어야 하는 순간 새로운 실행 계층이 필요해졌습니다.

## 3. 동적 Web의 필요성

방명록을 예로 들어보겠습니다.

```text
사용자: "안녕하세요"를 입력
          │
          ▼
Server가 입력을 처리하고 저장
          │
          ▼
다음 요청에서 저장된 글을 포함한 페이지 생성
```

정적 HTML 파일 하나만으로는 사용자가 보낸 글을 저장하거나 다음 요청에 반영할 수 없습니다. Server가 프로그램을 실행하고, 그 결과를 HTML로 만들어야 합니다.

이때 Web Server와 별도의 프로그램을 연결하는 방식으로 **CGI(Common Gateway Interface)**가 사용되기 시작했습니다.

## 4. CGI의 등장

CGI는 Web Server가 요청을 외부 프로그램에 전달하고, 외부 프로그램의 출력 결과를 다시 Client에게 전달하는 규칙입니다.

```text
Client
  │ POST /guestbook
  │ name=daeunki
  ▼
Web Server
  │ 실행
  ▼
CGI Program
  │ 데이터 처리 및 HTML 출력
  ▼
Web Server
  │ HTTP Response로 포장
  ▼
Client Browser
```

CGI Program은 C, Perl, Python, PHP 등으로 작성할 수 있습니다. Web Server는 Process를 만들고, 환경 변수와 표준 입력으로 Request 정보를 전달합니다. CGI의 표준 출력을 읽어 HTTP Response의 일부로 사용합니다.

### CGI가 해결한 문제

- 사용자의 입력을 처리할 수 있습니다.
- Request마다 다른 결과를 생성할 수 있습니다.
- Web Server와 Application Logic을 분리할 수 있습니다.
- 여러 언어로 동적 페이지를 만들 수 있습니다.

### CGI의 비용

전통적인 CGI는 요청마다 새로운 Process를 실행할 수 있습니다.

```text
요청 1 → CGI Process 1 생성 → 종료
요청 2 → CGI Process 2 생성 → 종료
요청 3 → CGI Process 3 생성 → 종료
```

요청이 많아지면 Process 생성과 종료 비용이 커지고, 상태 관리도 어려워집니다. 이 문제를 해결하기 위해 Web Server와 Application을 더 효율적으로 연결하는 여러 방식이 발전했습니다.

Webserv에서는 CGI를 직접 연결해야 하므로, `fork`, `execve`, `pipe`, 환경 변수, 표준 입출력, Process 종료 처리를 직접 이해하게 됩니다.

## 5. Web Server와 Backend의 분리

Web이 복잡해지면서 하나의 프로그램이 모든 역할을 맡기보다 역할을 나누기 시작했습니다.

```text
Browser
   │
   ▼
Web Server
   ├── 정적 파일 전달
   ├── TLS 종료
   ├── 요청 전달
   └── 기본 보안 처리
         │
         ▼
Backend Application
   ├── 비즈니스 규칙
   ├── 사용자 인증
   ├── 입력 검증
   └── 다른 시스템 호출
         │
         ▼
Database
```

### Web Server의 역할

Web Server는 외부 연결을 받고 HTTP를 처리하는 입구입니다.

- Socket 연결 수락
- HTTP Request 읽기
- 정적 파일 제공
- Response Header 작성
- 연결과 Timeout 관리
- Backend 또는 CGI로 요청 전달

### Backend의 역할

Backend는 Web Server 뒤에서 Application의 규칙과 데이터를 처리합니다.

- 사용자의 입력을 검증
- 방명록이나 게시글 저장
- 로그인과 권한 확인
- Database 조회와 변경
- 조건에 따라 Response Data 생성
- 다른 서비스나 API 호출

Webserv 과제는 주로 Web Server의 역할을 구현하지만, CGI를 통해 Backend Application과 연결되는 경계를 경험하게 합니다.

## 6. Database가 등장한 이유

동적 페이지가 많아지면 데이터를 파일에 직접 저장하는 것만으로는 부족합니다.

```text
파일만 사용
  ├── 검색이 어려움
  ├── 동시 수정 충돌
  ├── 관계 표현이 어려움
  └── 권한과 일관성 관리가 어려움
```

Database는 데이터를 구조화하고 검색·수정·삭제하는 기능을 제공합니다.

```text
Client
  ▼
Web Server
  ▼
Backend
  ▼
Database
```

방명록을 예로 들면 Backend는 다음과 같은 작업을 담당합니다.

1. Client가 보낸 이름과 메시지를 받습니다.
2. 입력 형식과 길이를 검증합니다.
3. Database에 저장합니다.
4. 저장 결과를 확인합니다.
5. 최신 방명록 목록을 생성합니다.

Webserv에서는 Database 자체를 구현하지 않지만, Request가 단순한 파일 읽기를 넘어 Application 처리로 이어지는 이유를 이해하는 데 도움이 됩니다.

## 7. Server-side와 Client-side

Web Application은 Server와 Browser 양쪽에서 처리될 수 있습니다.

### Server-side

```text
Request
  ▼
Server / Backend가 계산
  ▼
완성된 HTML 또는 Data Response
  ▼
Browser
```

CGI와 Backend가 대표적입니다. Database 접근이나 비밀 키 사용처럼 Server에서 해야 하는 작업을 처리합니다.

### Client-side

```text
HTML / CSS / JavaScript
          ▼
Browser가 화면과 상호작용 처리
```

Browser에서 버튼 동작, 화면 변화, 일부 계산을 처리합니다. 하지만 Database 비밀번호처럼 공개하면 안 되는 정보는 Client-side에 둘 수 없습니다.

현대 Web은 두 방식을 함께 사용합니다.

```text
Server: 인증과 데이터 처리
Client: 화면과 사용자 상호작용
```

## 8. Web 2.0과 Interactive Web

초기의 Web이 문서를 읽는 공간이었다면, Web 2.0에서는 사용자가 직접 내용을 만들고 공유하는 서비스가 중요해졌습니다.

```text
초기 Web
  Server → 사용자에게 문서 전달

Interactive Web
  사용자 → Server로 데이터 전송
  Server → 저장·처리·개인화된 결과 반환
```

방명록, 게시판, 댓글, 파일 Upload, 로그인은 모두 사용자의 입력을 처리해야 하므로 Backend의 역할이 커졌습니다.

HTTP Method도 이 흐름과 연결됩니다.

- `GET`: Resource 조회
- `POST`: Data 제출 또는 Resource 생성
- `DELETE`: Resource 삭제

Webserv에서 이 세 Method를 구현하는 것은 Interactive Web의 가장 기본적인 동작을 직접 만드는 것입니다.

## 9. API와 Frontend/Backend 분리

Backend가 항상 HTML을 직접 만들 필요는 없습니다. Data만 JSON으로 제공하고, Browser의 JavaScript나 별도의 Mobile Application이 이를 화면으로 표현할 수도 있습니다.

```text
Browser Frontend
       │ JSON Request
       ▼
Backend API
       │
       ▼
Database / Other Services
```

이런 구조에서는 HTTP Status Code와 Header가 더욱 중요합니다. Client는 Response Body만 보는 것이 아니라 Status Code를 보고 성공·실패를 판단합니다.

Webserv의 Request Parser와 Response Builder는 이 API 구조의 가장 낮은 수준을 이해하는 기반이 됩니다.

## 10. 현대 Web의 확장

서비스 규모가 커지면서 Backend도 하나의 프로그램에서 여러 구성요소로 나뉘었습니다.

```text
Reverse Proxy / Load Balancer
          │
          ├── Web Frontend
          ├── User Service
          ├── Payment Service
          └── Search Service
```

이런 구조에서는 다음 문제가 중요해집니다.

- 여러 Client를 동시에 처리하는가?
- Connection과 Resource를 효율적으로 사용하는가?
- Service 사이의 오류를 어떻게 전달하는가?
- Timeout과 Retry를 어떻게 관리하는가?
- 로그와 상태를 어떻게 관찰하는가?

Webserv의 Non-blocking I/O, Event Multiplexing, Timeout, 정확한 Error Response는 현대 Backend와 Server의 규모가 커져도 계속 이어지는 기본 원리입니다.

## 11. Webserv와 역사 연결하기

Webserv의 요구사항을 Web의 발전 흐름과 연결하면 다음과 같습니다.

| Web의 발전 | Webserv에서 경험하는 기능 |
|---|---|
| 정적 문서 전달 | Static Website, GET, Content-Type |
| 사용자 입력 처리 | POST, Request Body |
| Resource 관리 | DELETE, Status Code |
| 동적 페이지 | CGI, Process, Pipe |
| 여러 Website 운영 | 여러 Port, Route Configuration |
| 대규모 연결 처리 | Non-blocking I/O, Event Loop |
| 오류와 신뢰성 | Error Page, Timeout, Stress Test |

따라서 Webserv는 단순한 옛날 방식의 Server를 만드는 과제가 아닙니다. HTTP Server가 어떤 문제를 해결해왔고, 그 문제들이 오늘날 Backend Architecture로 어떻게 확장되었는지를 가장 낮은 수준에서 확인하는 과제입니다.

## 12. 이 페이지를 읽은 뒤 답할 질문

1. 정적 Web과 동적 Web의 차이는 무엇인가?
2. 방명록 같은 기능을 만들 때 왜 Server-side 프로그램이 필요한가?
3. CGI는 Web Server와 Application 사이에서 어떤 역할을 하는가?
4. Web Server와 Backend를 분리하면 어떤 장점이 있는가?
5. Database는 파일 저장만으로 해결하기 어려운 어떤 문제를 해결하는가?
6. Webserv에서 구현하는 GET, POST, DELETE는 Interactive Web과 어떻게 연결되는가?
7. 왜 현대 Web에서도 Socket, HTTP, Status Code, Timeout이 중요한가?
