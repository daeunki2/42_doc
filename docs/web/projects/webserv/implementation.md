# Implementation Guide

Webserv는 기능을 하나씩 붙이는 것보다, 처음부터 **이벤트 기반 서버의 구조**를 세우는 것이 중요합니다. 아래 순서는 구현과 학습을 함께 진행하기 위한 제안입니다.

## 1. 먼저 전체 흐름을 그린다

```text
Configuration File
        ↓
Server Configuration
        ↓
Listen Socket
        ↓
Event Loop
        ↓
Client Connection State
        ↓
HTTP Request Parser
        ↓
Route Matching
        ↓
Response Builder
        ↓
File / CGI / Upload
```

각 계층은 다음 계층에 필요한 정보만 전달해야 합니다. 예를 들어 Socket 계층은 “몇 Byte를 읽었는가”를 알려주고, HTTP Parser는 그 Byte가 Header인지 Body인지 판단합니다.

## 2. Configuration Parser

처음부터 NGINX 문법 전체를 복사할 필요는 없습니다. 과제가 요구하는 데이터 모델을 먼저 만듭니다.

```text
ServerConfig
  ├── listen addresses
  ├── error pages
  ├── client max body size
  └── routes
       ├── path
       ├── methods
       ├── redirect
       ├── root
       ├── directory listing
       ├── index file
       ├── upload path
       └── CGI rules
```

Parser는 다음 오류를 구분해야 합니다.

- 문법이 잘못됨
- 필수 값이 없음
- Port나 Path 값이 유효하지 않음
- 같은 정책이 충돌함

설정 오류는 서버가 시작되기 전에 발견하고 명확한 메시지와 함께 종료하는 편이 안전합니다.

## 3. Listen Socket 만들기

각 `interface:port`에 대해 Listen Socket을 생성합니다.

```text
getaddrinfo
    ↓
socket
    ↓
setsockopt
    ↓
fcntl(O_NONBLOCK)
    ↓
bind
    ↓
listen
    ↓
Event Loop에 등록
```

Listen Socket은 Client의 Application Data를 읽는 Socket이 아니라 새로운 연결을 받아들이는 Socket입니다. Event Multiplexing에서 Readable 상태가 되면 `accept()`를 시도합니다.

## 4. 하나의 Event Loop

서버의 중심은 모든 Socket과 Pipe를 관리하는 하나의 Event Loop입니다.

```cpp
while (running)
{
    wait_for_events();
    handle_ready_descriptors();
    handle_timeouts();
}
```

실제 구현에서는 `poll()`을 사용할 수도 있고, macOS에서는 `kqueue()`를 사용할 수도 있습니다. 중요한 것은 함수 이름이 아니라 다음 규칙입니다.

1. 모든 I/O 가능한 Descriptor를 하나의 감시 목록으로 관리합니다.
2. Event가 준비됐다는 결과를 받은 뒤에만 Socket을 읽거나 씁니다.
3. Read Event와 Write Event를 상황에 맞게 등록합니다.
4. 한 Client를 처리하는 동안 다른 Client를 기다리며 멈추지 않습니다.

## 5. Connection State

Client마다 독립적인 상태 객체가 필요합니다.

```text
ClientConnection
  ├── socket fd
  ├── input buffer
  ├── output buffer
  ├── parser state
  ├── request metadata
  ├── body progress
  ├── selected route
  ├── CGI state
  └── close/keep-alive state
```

여러 Client가 동시에 Header를 보내면 각 Client의 진행 위치를 섞으면 안 됩니다.

### 예시 상태 전이

```text
NEW
 ↓ accept
READING_HEADERS
 ↓ complete headers
READING_BODY
 ↓ complete body
PROCESSING
 ↓ response ready
WRITING_RESPONSE
 ↓ all bytes sent
KEEP_ALIVE 또는 CLOSING
```

## 6. HTTP Parser

HTTP Parser는 문자열을 한 번에 완성된 Request로 받는다고 가정하면 안 됩니다.

```text
첫 번째 read:  GET /index
두 번째 read: .html HTTP/1.1\r\nHost: ...
세 번째 read: \r\n\r\n
```

따라서 Buffer에 받은 데이터를 누적하고, Header 종료 구분자 `\r\n\r\n`을 찾은 뒤 Request Line과 Header를 해석해야 합니다.

### Parser 단계

```text
1. Request Line 대기
2. Header Line 분리
3. Header 이름 정규화/검증
4. Body 길이 결정
5. Body 수신
6. 완전한 Request 생성
```

검증해야 하는 대표 항목은 Method, Target, HTTP Version, Header 형식, Body 크기입니다.

## 7. Body 처리

`Content-Length`가 있는 Request는 정해진 Byte 수를 받을 때까지 Body를 누적합니다.

`Transfer-Encoding: chunked`가 사용되면 Chunk 크기를 해석하고 Chunk Header와 끝의 `0` Chunk를 제거한 뒤 실제 Body만 만들어야 합니다. CGI는 일반적으로 Un-chunk된 Body와 EOF를 기대합니다.

```text
Chunked Body
  size\r\n
  data\r\n
  0\r\n
  \r\n
        ↓
CGI에 전달할 순수 Body
```

Client Body가 설정된 최대 크기를 넘으면 전체 Body를 불필요하게 저장하지 말고 적절한 Error Response를 준비해야 합니다.

## 8. Route Matching

Request Target과 Configuration Route를 비교해 가장 적합한 규칙을 선택합니다.

```text
Request: /kapouet/pouic/toto
Route:   /kapouet
Root:    /tmp/www
Result:  /tmp/www/pouic/toto
```

Route 선택 후 다음 순서로 처리합니다.

```text
Method 허용 여부
    ↓
Redirect 여부
    ↓
파일/디렉토리 경로 계산
    ↓
Directory Index 또는 Listing
    ↓
GET/POST/DELETE 정책
    ↓
CGI 또는 Static File
```

경로를 계산할 때 URL Decode, `..`, Root 밖 접근, 디렉토리와 파일의 구분을 신중하게 처리해야 합니다.

## 9. Response Builder

응답은 다음 세 부분으로 구성됩니다.

```text
Status Line
Headers
빈 줄
Body
```

예시:

```http
HTTP/1.1 200 OK\r\n
Content-Type: text/html\r\n
Content-Length: 42\r\n
Connection: close\r\n
\r\n
<html>...</html>
```

Output Buffer에 모두 저장한 뒤 Socket이 Writable일 때 일부씩 보내야 합니다. `send()`가 전체 Buffer를 한 번에 처리한다고 가정하면 안 됩니다.

```text
Output Buffer: [전송하지 않은 전체 Response]
       ↓ send가 일부만 전송
Output Buffer: [남은 Response]
       ↓ 반복
Buffer empty -> keep-alive 또는 close
```

## 10. Static File과 Directory

GET 요청이 파일을 가리키면 파일을 읽어 Response Body로 보냅니다. 디렉토리를 가리키면 Configuration에 따라 다음 중 하나를 수행합니다.

- Index 파일 제공
- Directory Listing 생성
- `403 Forbidden` 또는 적절한 Error 응답

Content-Type은 파일 확장자 또는 최소한의 MIME 매핑을 통해 설정합니다.

## 11. Upload와 DELETE

Upload는 Body를 받은 뒤 설정된 Upload 경로에 파일을 저장하는 기능입니다. 파일을 열고 쓰는 디스크 I/O는 Socket Readiness와는 다르지만, 파일 크기·권한·이름 충돌·경로 탈출을 검증해야 합니다.

DELETE는 Route가 허용하는지 먼저 확인하고, 파일 존재 여부와 삭제 결과를 Response로 표현합니다.

## 12. CGI Process

CGI는 서버 Process와 외부 프로그램 사이에 Pipe를 연결하는 작업입니다.

```text
Webserv Process
  ├── request body -> stdin Pipe -> CGI stdin
  └── CGI stdout -> stdout Pipe -> Webserv
```

일반적인 순서는 다음과 같습니다.

```text
1. Pipe 생성
2. CGI 실행에 필요한 환경 변수 준비
3. fork
4. Child에서 dup2로 stdin/stdout 연결
5. Child에서 execve
6. Parent에서 CGI Output을 Event Loop로 읽기
7. CGI Header와 Body를 해석
8. Client Response로 변환
```

`fork()`는 CGI 이외의 용도로 사용하지 않습니다. CGI가 끝나지 않거나 Pipe가 닫히지 않는 경우를 대비해 Timeout과 종료 처리가 필요합니다.

## 13. Resource와 종료 처리

각 단계가 실패하면 이미 생성한 File Descriptor와 Process를 정리해야 합니다.

```text
Client close
  ↓
Event 목록에서 제거
  ↓
Socket shutdown/close
  ↓
Buffer와 CGI 상태 해제
  ↓
Child Process wait 처리
```

서버가 한 번의 잘못된 Request 때문에 전체 Process를 종료하지 않도록 예외와 오류 반환 경계를 설계해야 합니다.

## 14. 구현 순서 제안

```text
1. Makefile과 빈 Server 실행
2. Listen Socket 하나 만들기
3. Client accept와 연결 종료
4. Event Loop로 여러 Client 관리
5. 고정 HTTP Response 보내기
6. Request Line과 Header Parser
7. GET 정적 파일
8. Error Page와 상태 코드
9. Configuration Parser
10. Route와 여러 Port
11. POST, Upload, DELETE
12. Directory Listing과 Redirect
13. CGI
14. Browser/Stress/비정상 입력 검증
```

각 단계에서 작동하는 작은 상태를 유지해야 합니다. 여러 기능을 한 번에 추가하면 Parser, Routing, I/O 문제를 구분하기 어렵습니다.
