# Testing and Evaluation

Webserv는 “브라우저에서 페이지가 보인다”만으로 완료할 수 없습니다. HTTP 규칙, Non-blocking 제약, 여러 Client, 오류 상황을 각각 확인해야 합니다.

## 1. 테스트 단계

```text
Build Test
    ↓
Socket Test
    ↓
HTTP Parser Test
    ↓
Static File Test
    ↓
Method/Route Test
    ↓
Upload/CGI Test
    ↓
Concurrent Client Test
    ↓
Stress and Invalid Input Test
```

기능을 추가할 때마다 이전 테스트를 다시 실행하는 Regression Test 습관을 만듭니다.

## 2. Build와 실행

```bash
make
./webserv ./conf/default.conf
```

확인할 것:

- `-Wall -Wextra -Werror`로 경고 없이 빌드되는가?
- `-std=c++98`을 추가해도 빌드되는가?
- 소스 파일이 바뀌지 않았을 때 불필요한 재링크를 하지 않는가?
- 잘못된 Configuration에서 설명 가능한 오류를 출력하는가?

## 3. 가장 작은 HTTP 테스트

먼저 고정 응답을 보내는 서버부터 확인합니다.

```bash
curl -v http://127.0.0.1:8080/
```

`-v`는 Request와 Response Header를 보여주므로 Browser 화면보다 학습에 유용합니다.

확인할 것:

- TCP 연결이 만들어지는가?
- Response Status Line이 올바른가?
- Header와 Body 사이에 빈 줄이 있는가?
- `Content-Length` 또는 Connection 종료 조건이 맞는가?
- 응답을 보낸 뒤 서버 전체가 종료되지 않는가?

## 4. telnet으로 직접 보기

```bash
telnet 127.0.0.1 8080
GET / HTTP/1.1
Host: localhost

```

빈 줄을 두 번 보내야 Header의 끝을 표현할 수 있습니다. telnet은 Browser가 자동으로 추가하는 정보를 숨기지 않기 때문에 Parser 학습에 좋습니다.

환경에 telnet이 없다면 `nc`를 사용할 수 있습니다.

```bash
printf 'GET / HTTP/1.1\r\nHost: localhost\r\nConnection: close\r\n\r\n' | nc 127.0.0.1 8080
```

## 5. Method 테스트

```bash
curl -i http://127.0.0.1:8080/index.html
curl -i -X POST -d 'name=value' http://127.0.0.1:8080/upload
curl -i -X DELETE http://127.0.0.1:8080/file.txt
```

각 테스트에서 단순히 Status Code만 보지 말고 다음을 기록합니다.

```text
Request Method
Request Target
Request Body
Response Status
Response Headers
Response Body
파일 시스템 변화
```

Route에서 Method를 허용하지 않으면 `405 Method Not Allowed`와 적절한 `Allow` Header를 고려해야 합니다.

## 6. Host와 여러 Port

```bash
curl -i -H 'Host: site-a.local' http://127.0.0.1:8080/
curl -i -H 'Host: site-b.local' http://127.0.0.1:8081/
```

과제의 필수 범위는 여러 `interface:port`를 듣는 것입니다. Host 기반 Virtual Host는 별도 확장으로 생각하되, 설정한 각 Port가 올바른 Content를 제공하는지 확인합니다.

## 7. Error 테스트

의도적으로 잘못된 Request를 보내야 합니다.

```bash
curl -i http://127.0.0.1:8080/not-found
curl -i -X PATCH http://127.0.0.1:8080/
curl -i http://127.0.0.1:8080/../../etc/passwd
```

가능한 테스트 목록:

- 존재하지 않는 파일
- 읽을 수 없는 파일
- 디렉토리 Listing 금지
- 허용되지 않은 Method
- 너무 큰 Body
- 잘못된 Request Line
- 잘못된 Header
- 불완전한 Request
- URL Root 밖으로 나가는 Path

각 결과가 `404`, `403`, `405`, `413`, `400` 중 어떤 것인지 이유를 설명할 수 있어야 합니다.

## 8. Upload와 파일 상태

```bash
curl -i -X POST --data-binary '@sample.txt' http://127.0.0.1:8080/upload
curl -i -F 'file=@sample.txt' http://127.0.0.1:8080/upload
```

확인할 것:

- Body가 일부만 저장되지 않았는가?
- 설정된 Upload Directory에 저장되는가?
- 파일 이름을 통해 경로를 탈출할 수 없는가?
- 최대 Body 크기를 넘으면 저장을 중단하는가?
- 성공과 실패의 상태 코드가 구분되는가?

## 9. CGI 테스트

하나의 CGI를 선택해 최소한의 동작을 확인합니다.

```text
Request
   ↓
Webserv
   ↓ 환경 변수 + stdin
CGI Program
   ↓ stdout
Webserv
   ↓
HTTP Response
```

확인할 것:

- CGI Process가 실행되는가?
- Working Directory가 올바른가?
- Query String이 전달되는가?
- POST Body가 stdin으로 전달되는가?
- CGI Header와 Body를 구분하는가?
- CGI가 종료되지 않을 때 Server 전체가 멈추지 않는가?
- CGI 종료 후 Zombie Process가 남지 않는가?

## 10. 느린 Client 테스트

Header 또는 Body를 여러 번에 나누어 천천히 보내는 Client를 준비합니다.

```text
연결
  ↓
Request Header 일부만 전송
  ↓ 잠시 대기
나머지 Header 전송
  ↓
Body를 작은 조각으로 전송
```

이 테스트에서 다른 Client의 요청이 정상 처리되는지 확인해야 합니다. 한 Client의 `recv()`가 완성된 Request를 기다리며 Block되면 설계 문제가 있는 것입니다.

## 11. 여러 Client와 Stress Test

서버가 동시에 많은 연결을 처리하는지 확인합니다.

```bash
for i in $(seq 1 100); do
  curl -sS http://127.0.0.1:8080/ > /dev/null &
done
wait
```

더 정교한 테스트는 Python이나 Go로 작성해 다음을 측정합니다.

- 성공 응답 비율
- 응답 시간
- 연결 실패 수
- 서버 Process 생존 여부
- File Descriptor 증가 여부
- Memory 증가 여부

평가 중에는 한 가지 Client만 사용하지 말고 Browser, curl, telnet/nc, 자체 Test Script를 조합합니다.

## 12. NGINX 비교

같은 파일과 비슷한 설정으로 NGINX와 Webserv에 같은 Request를 보냅니다.

```text
같은 URL
같은 Method
같은 Body
같은 Error 상황
        ↓
Status / Header / Body / Connection 동작 비교
```

NGINX와 다른 모든 동작이 틀린 것은 아닙니다. HTTP Version 차이와 과제에서 선택한 정책을 구분하면서 비교해야 합니다.

## 13. Event Loop 점검

코드 리뷰 때 다음 질문을 기준으로 확인합니다.

- 모든 Socket이 Non-blocking인가?
- Socket과 Pipe를 하나의 Event Multiplexing 호출로 관리하는가?
- Event 결과를 받기 전에 `read`, `recv`, `write`, `send`를 호출하는 곳이 없는가?
- Read Buffer와 Write Buffer가 Connection별로 분리되어 있는가?
- Partial Read와 Partial Write를 처리하는가?
- `POLLHUP`, `POLLERR`, EOF를 처리하는가?
- Client 하나가 다른 Client를 기다리게 만들지 않는가?

## 14. 평가 직전 체크리스트

### Build

- Makefile 필수 규칙 확인
- C++98 컴파일 확인
- 경고를 Error로 취급해도 성공
- 외부 Library와 Boost 미사용

### Runtime

- 기본 Configuration으로 실행
- 여러 Port 수신
- Browser 호환
- GET, POST, DELETE
- Static Website
- Upload
- Error Page
- Directory Listing
- Redirect
- CGI
- 큰 Body와 잘못된 Request
- Stress Test

### 제출

- Root README가 영어인가?
- README 첫 줄 형식이 맞는가?
- Configuration과 Test 파일이 Repository에 포함되어 있는가?
- 실행에 필요한 기본 파일이 모두 있는가?
- 과제 PDF와 Local 환경 파일은 Git에 포함되지 않는가?

## 15. 최종 자기 점검 질문

1. 왜 `poll` 또는 동등한 API를 한 번만 사용해야 하는가?
2. Socket에서 Partial Read와 Partial Write가 발생하면 어떻게 처리하는가?
3. Request Body의 끝을 어떻게 판단하는가?
4. CGI가 끝났다는 것을 어떻게 판단하는가?
5. 특정 Client가 멈춰 있을 때 다른 Client는 계속 처리되는가?
6. Route Root와 URL Path를 어떻게 안전하게 결합하는가?
7. 서버가 Crash하지 않는다는 것을 어떤 테스트로 증명할 수 있는가?
8. 평가자가 설정을 바꾸거나 작은 기능 수정을 요구해도 구조를 설명하고 수정할 수 있는가?
