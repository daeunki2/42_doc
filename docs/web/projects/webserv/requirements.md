# Requirements

이 문서는 Webserv 과제의 지시사항을 “무엇을 해야 하는가”와 “왜 필요한가”로 나누어 읽는 페이지입니다.

## 1. 실행 형식

실행 파일은 설정 파일을 인자로 받아 실행되어야 합니다.

```bash
./webserv [configuration file]
```

설정 파일은 서버가 어떤 Port에서 듣고, 어떤 URL을 어떤 파일과 연결하며, 어떤 기능을 허용할지 정의합니다. 설정 파일을 코드에 하드코딩하면 안 되는 이유는 같은 서버 프로그램으로 여러 사이트와 정책을 실행할 수 있어야 하기 때문입니다.

## 2. 언어와 빌드 제한

| 항목 | 요구사항 |
|---|---|
| 언어 | C++98 |
| 컴파일러 | `c++` |
| 경고 | `-Wall -Wextra -Werror` |
| 외부 라이브러리 | 금지, Boost 포함 |
| 필수 Makefile 규칙 | `NAME`, `all`, `clean`, `fclean`, `re` |
| 재링크 | 불필요한 재링크를 하지 않아야 함 |

C++98로 작성해야 하므로 최신 C++ 문법이나 외부 HTTP 라이브러리에 의존하지 않습니다. 이 제한은 Socket, Parser, Process, Resource 관리의 기본을 직접 이해하게 합니다.

## 3. 사용 가능한 시스템 함수의 의미

과제는 운영체제의 저수준 API를 사용해 서버를 구현하도록 합니다.

| 목적 | 대표 함수 |
|---|---|
| Socket 생성과 연결 | `socket`, `bind`, `listen`, `accept`, `connect` |
| 주소 변환 | `getaddrinfo`, `freeaddrinfo`, `htons`, `ntohs` |
| I/O | `read`, `write`, `recv`, `send`, `shutdown`, `close` |
| Event Multiplexing | `poll`, `select`, `epoll`, `kqueue` |
| Non-blocking | `fcntl`와 허용된 Flag |
| CGI Process | `fork`, `execve`, `pipe`, `dup2`, `waitpid` |
| 파일과 디렉토리 | `open`, `stat`, `access`, `opendir`, `readdir`, `closedir` |
| 오류 | `errno`, `strerror`, `gai_strerror` |

운영체제 API를 호출하는 것보다 중요한 것은 각 호출이 서버 상태에서 어느 시점에 필요한지 설명하는 것입니다.

## 4. Non-blocking과 단일 Event Loop

서버는 항상 Non-blocking으로 동작해야 하며, Client와 Server의 모든 I/O를 하나의 `poll` 또는 동등한 Event Multiplexing 호출로 관리해야 합니다.

```text
한 번의 Event Wait
    ├── Listen Socket 읽기 가능
    ├── Client Socket 읽기 가능
    ├── Client Socket 쓰기 가능
    └── CGI Pipe 읽기/쓰기 가능
```

다음 규칙은 특히 엄격합니다.

- `poll` 또는 동등한 호출 전에 Socket을 읽거나 쓰지 않습니다.
- 읽기와 쓰기를 둘 다 감시합니다.
- I/O가 준비되지 않았다고 `errno`를 확인해 반복하는 방식으로 해결하지 않습니다.
- 일반 디스크 파일은 Readiness 감시에서 예외지만, Socket과 Pipe는 반드시 이벤트 기반으로 처리합니다.
- Client 요청이 영원히 멈추지 않도록 Timeout 또는 종료 조건을 설계합니다.

이 규칙의 목적은 한 Client의 느린 동작이 전체 서버를 막지 못하게 하는 것입니다.

## 5. 필수 HTTP 기능

### GET

정적 파일이나 디렉토리 요청을 처리해야 합니다.

```text
GET /index.html HTTP/1.1
        ↓
파일 탐색
        ↓
200 OK + 파일 내용
```

파일이 없으면 `404 Not Found`, 접근 권한이 없으면 상황에 맞는 오류 응답을 반환해야 합니다.

### POST

Body가 포함된 요청을 처리해야 합니다. Upload나 CGI 실행에 활용할 수 있습니다.

Body를 읽을 때 `Content-Length` 또는 `Transfer-Encoding`을 확인해야 하며, 한 번의 Read가 Body 전체를 준다고 가정하면 안 됩니다.

### DELETE

Configuration과 Route 정책이 허용하는 경우 요청된 자원을 삭제해야 합니다. 삭제할 수 없는 파일, 존재하지 않는 경로, 권한 문제를 각각 적절한 상태 코드로 구분해야 합니다.

## 6. Static Website

서버는 HTML, CSS, JavaScript, 이미지 등을 포함하는 완전한 정적 Website를 제공할 수 있어야 합니다.

이를 위해 다음 흐름이 필요합니다.

```text
URL Path
   ↓
Route Root와 결합
   ↓
파일 또는 디렉토리 확인
   ↓
권한과 존재 여부 확인
   ↓
Content-Type 결정
   ↓
HTTP Response 작성
```

URL 문자열을 파일 경로로 단순히 이어붙이는 것은 위험합니다. `..`과 같은 경로가 Root 밖으로 빠져나가지 않는지 검증해야 합니다.

## 7. Error Page와 Status Code

설정 파일에서 사용자 정의 Error Page를 지정할 수 있어야 하고, 지정되지 않은 경우에도 기본 Error Page를 제공해야 합니다.

상태 코드는 “무언가 실패했다”가 아니라 실패 원인을 표현해야 합니다.

| 상황 | 예시 상태 |
|---|---|
| 정상 응답 | `200 OK` |
| 새로 생성됨 | `201 Created` |
| Redirect | `301`, `302` 등 |
| 잘못된 Request | `400 Bad Request` |
| 권한 없음 | `403 Forbidden` |
| 파일 없음 | `404 Not Found` |
| 허용되지 않은 Method | `405 Method Not Allowed` |
| Body가 너무 큼 | `413 Payload Too Large` |
| 서버 내부 문제 | `500 Internal Server Error` |
| CGI Gateway 문제 | `502 Bad Gateway` |
실제 사용할 상태 코드는 상황과 HTTP 버전에 맞게 설계하고, NGINX를 비교 기준으로 활용합니다.

## 8. Port와 여러 Website

서버는 여러 `interface:port` 쌍을 듣고 서로 다른 Content를 제공할 수 있어야 합니다.

```text
0.0.0.0:8080 -> Website A
0.0.0.0:8081 -> Website B
```

따라서 설정 Parser는 Server Block 또는 그에 해당하는 구조를 읽어야 하고, 각 Listen Socket을 Event Loop에 등록해야 합니다.

Virtual Host는 과제의 필수 범위를 벗어나지만, 여러 Port를 지원하는 구조는 필수입니다.

## 9. Configuration Route 기능

Configuration 파일에서는 URL Route마다 다음 정책을 지정할 수 있어야 합니다.

- 허용할 HTTP Method 목록
- HTTP Redirect
- 요청 파일을 찾을 Root 디렉토리
- Directory Listing 활성화 여부
- Directory 요청의 기본 파일
- Upload 허용 여부와 저장 경로
- 파일 확장자에 따른 CGI 실행

```text
Request Path: /kapouet/pouic/toto
Route Root:   /tmp/www
File Path:    /tmp/www/pouic/toto
```

Route를 찾는 순서와 가장 적합한 규칙을 선택하는 방식은 구현 전에 명확히 정해야 합니다.

## 10. CGI

CGI는 Web Server가 외부 프로그램을 실행해 동적인 Response를 얻는 방식입니다.

```text
Client
  ↓ Request
Webserv
  ├── 환경 변수 준비
  ├── Pipe 생성
  ├── CGI Process 실행
  ├── Request Body 전달
  └── CGI Output을 HTTP Response로 변환
```

필수 학습 포인트는 다음과 같습니다.

- `fork`는 CGI 실행에만 사용합니다.
- `execve`로 PHP-CGI나 Python 등 하나 이상의 CGI를 실행합니다.
- Request와 Query String 정보를 환경 변수에 전달합니다.
- Chunked Request Body는 CGI에 전달하기 전에 Un-chunk해야 합니다.
- CGI가 `Content-Length`를 출력하지 않으면 EOF가 출력 끝을 의미할 수 있습니다.
- CGI를 올바른 Working Directory에서 실행해야 상대 경로가 의도대로 동작합니다.
- CGI의 Pipe도 Non-blocking Event Loop 규칙을 따라야 합니다.

## 11. macOS 제한

macOS에서는 `write()` 동작 차이 때문에 File Descriptor를 Non-blocking으로 설정해야 합니다.

macOS에서 `fcntl()` 사용은 다음 Flag로 제한됩니다.

```text
F_SETFL
O_NONBLOCK
FD_CLOEXEC
```

플랫폼별로 허용 함수와 Flag가 다른지 항상 평가 환경에 맞춰 확인해야 합니다.

## 12. 안정성

프로그램은 어떤 상황에서도 Crash하거나 예기치 않게 종료되어서는 안 됩니다. 특히 다음 경우를 시험해야 합니다.

- Client가 Header를 한 Byte씩 천천히 보냄
- Body가 매우 큼
- Client가 연결 중간에 끊김
- 잘못된 Method나 Header가 들어옴
- 존재하지 않는 파일을 요청함
- CGI가 실패하거나 예상과 다른 Output을 냄
- 여러 Client가 동시에 접속함
- Stress Test 중 연결이 반복해서 생성되고 종료됨

## 13. README 요구사항

Repository Root의 `README.md`는 영어로 작성해야 합니다.

최소한 다음을 포함해야 합니다.

- 첫 줄: 42 과정에서 만들어진 프로젝트임을 나타내는 Italic 문구
- Description
- Instructions
- Resources
- AI를 어떤 작업에 어떻게 사용했는지에 대한 설명

README는 평가자와 채용 담당자가 프로젝트를 빠르게 이해하는 문서이므로, 단순히 명령어만 나열하지 말고 설계 선택과 테스트 방법도 설명하는 것이 좋습니다.

## 14. Bonus

Mandatory가 모든 조건에서 안정적으로 동작한 뒤에만 Bonus를 진행합니다.

- Cookie와 Session Management
- 여러 종류의 CGI 지원

Bonus는 필수 기능을 대신하지 않습니다. 필수 기능이 불완전하면 Bonus는 평가되지 않습니다.
