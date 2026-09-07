---
tags:
  - 네트워크
  - 소켓
---
## 1. 소켓 함수별 역할

|함수|역할|서버/클라|
|---|---|---|
|`socket()`|통신용 소켓 핸들을 커널에 생성 요청. 아직 IP/포트/연결 없음|공통|
|`bind()`|소켓에 로컬 IP + 포트 결합|서버 필수 (클라는 보통 생략, connect 시 OS가 임시 포트 자동 할당)|
|`listen()`|소켓을 연결 요청을 받을 수 있는 passive 상태로 전환. `backlog`로 대기 큐 크기 지정|서버 전용|
|`accept()`|ACCEPT 큐에서 완료된 연결 하나를 꺼내 **새 소켓** 반환|서버 전용|
|`connect()`|서버로 3-way handshake 시도|클라이언트|
|`closesocket()`|소켓 리소스 해제 + TCP 연결 종료(FIN) 절차 시작|공통 (winsock은 `close()`가 아니라 `closesocket()`)|

### 전체 호출 흐름

```
        SERVER                          CLIENT
      socket()                        socket()
         │                                │
         ▼                                │
       bind()                             │
         │                                │
         ▼                                ▼
      listen()  ◄─────연결 요청───── connect()
         │                                │
         ▼                                │
      accept()                            │
         │                                │
         ▼                                ▼
    read() & write()  ◄──데이터 송수신──► read() & write()
         │                                │
         ▼                                ▼
      close()      ◄────연결 종료────►  close()
```

**서버**

```
socket() → bind() → listen() → accept() (블로킹) → recv()/send() 반복 → closesocket()
```

**클라이언트**

```
socket() → connect() (블로킹, handshake) → send()/recv() 반복 → closesocket()
```

---

## 2. 서버 측 호출 순서별 상세

### 2-1. socket()

- 통신에 사용할 소켓 객체를 커널에 생성 요청.
- `SOCKET socket(int af, int type, int protocol)` — 주소체계(AF_INET), 타입(SOCK_STREAM), 프로토콜(IPPROTO_TCP) 지정.
- 이 시점엔 아직 IP/포트도, 연결도 없는 "빈 소켓" 상태.

### 2-2. bind()

- 생성한 소켓에 로컬 IP + 포트 번호를 결합.
- `sockaddr_in` 구조체에 주소/포트 채워서 넘김.
- 서버는 반드시 호출 (클라이언트가 접속할 고정 포트 필요).

### 2-3. listen()

- 소켓을 "연결 요청을 받을 수 있는 상태(passive/listening socket)"로 전환.
- 두 번째 인자 `backlog`로 accept() 되기 전 대기 가능한 연결 큐 크기 지정.
- 이 호출 이후부터 클라이언트의 SYN을 받아줄 수 있음.

#### SYN 큐 (SYN Backlog / Incomplete Connection Queue)

- SYN만 받고 아직 handshake 안 끝난 연결 대기 (상태: SYN_RECEIVED)
- 흐름: SYN 수신 → 큐 생성 → SYN-ACK 응답 → 클라 ACK 대기
- SYN Flood 공격이 노리는 큐

#### ACCEPT 큐 (Accept Queue / Completed Connection Queue)

- handshake 완료(ACK까지 수신)된 연결 대기 (상태: ESTABLISHED)
- `accept()` 호출 시 여기서 하나씩 꺼냄
- 애플리케이션이 accept()를 늦게 부르면 여기 계속 쌓임

#### 두 큐를 거치는 전체 흐름

```
클라이언트 SYN 전송
   ↓
서버: SYN 큐 진입 (SYN_RECEIVED)
   ↓
서버: SYN-ACK 응답
   ↓
클라이언트: ACK 전송
   ↓
서버: ACK 수신 → SYN 큐 제거, ACCEPT 큐 이동 (ESTABLISHED)
   ↓
서버 accept() 호출 → ACCEPT 큐에서 소켓 하나 반환
```

#### 왜 두 큐를 분리하는가

- TCP handshake는 네트워크 왕복시간(RTT)이 걸리는 **비동기 이벤트** — 커널이 자체 관리, 애플리케이션 코드 관여 없음.
- `accept()` 호출 타이밍은 **애플리케이션 스케줄링에 의존** — 다른 작업 때문에 늦게 호출될 수 있음.
- 두 속도 차이 때문에 큐 분리 필요:
    - SYN 큐: handshake 진행 중인 것들, 커널이 자동 관리
    - ACCEPT 큐: handshake 완료됐지만 애플리케이션이 아직 안 가져간 것들의 대기실 → 애플리케이션이 늦어도 완료된 연결이 유실되지 않음
- 큐가 하나였다면 "진행 중"과 "완료돼서 넘길 준비된 것"이 뒤섞여 커널이 accept() 요청에 뭘 줘야 할지 구분 불가능해짐.

#### backlog 파라미터와 큐의 관계

- `listen(listenSocket, backlog)`의 backlog 의미는 OS마다 다름
    - 리눅스: SYN 큐/ACCEPT 큐 분리, backlog는 주로 ACCEPT 큐에 영향. SYN 큐는 `tcp_max_syn_backlog` 커널 파라미터로 별도 관리
    - Windows(winsock): 두 큐를 명확히 분리 노출하지 않고 backlog가 대기 가능 연결 전체(주로 accept 대기)에 대한 힌트로 동작
- ACCEPT 큐가 가득 차면 handshake 끝난 연결도 서버가 못 받아서 RST/타임아웃 발생 가능 → accept()를 빨리 호출해 큐 비우는 게 중요

### 2-4. accept()

- listen 큐에 쌓인 완료된 연결(3-way handshake 끝난 것) 중 하나를 꺼내서 **새로운 소켓**을 반환.
- 원래 listening 소켓은 계속 살아있고, accept()가 반환한 새 소켓이 그 클라이언트와의 실제 통신용 소켓.
- 블로킹 소켓이면 연결 요청 들어올 때까지 여기서 대기.

#### 왜 listening 소켓과 accept 소켓을 분리하는가

- **listening 소켓**: 특정 상대와 묶이지 않고 "이 포트로 오는 새 연결 요청"만 계속 받는 역할. 상태는 계속 LISTEN.
- **accept()가 반환하는 소켓**: 특정 클라이언트 1개와 1:1로 묶인 통신 채널. (로컬IP, 로컬포트, 원격IP, 원격포트) 4-tuple로 유일 식별.
- 분리 안 하면: 클라이언트 A와 통신 중인 소켓이 새 클라이언트 B의 연결 요청을 동시에 받을 수 없음 (TCP 소켓 = 특정 상대와의 통신 채널이라는 모델과 충돌).
- 같은 로컬 포트를 여러 클라이언트가 동시에 물어도 문제없는 이유: 로컬 포트가 같아도 원격 IP/포트가 다르면 커널은 다른 소켓(다른 4-tuple)으로 구분.

> **요약**: listen/accept 소켓 분리는 "통신 채널의 독립성" 문제, SYN/ACCEPT 큐 분리는 "비동기 진행 상태와 소비 속도의 불일치" 문제. 둘 다 서버가 여러 연결을 안정적으로 동시에 다루기 위한 설계로 수렴.

---

## 3. 클라이언트 측 호출 순서별 상세

### 3-1. socket()

- 서버와 동일하게 소켓 핸들 생성.

### 3-2. connect()

- 서버의 IP+포트로 3-way handshake 시도.

#### 반환 시점

**블로킹 소켓 (기본값)**

- `connect()`는 **TCP 3-way handshake가 완료될 때까지** 호출 스레드를 블로킹.
- SYN 전송 → SYN-ACK 수신 → ACK 전송까지 끝나야 반환.
- 상대가 응답 없으면 시스템 기본 타임아웃(보통 21초 전후, 재전송 횟수 기반)까지 블로킹 후 실패 반환.

**논블로킹 소켓** (`ioctlsocket`으로 `FIONBIO` 설정)

- `connect()`는 즉시 반환 (handshake 완료를 기다리지 않음).
- 거의 항상 `SOCKET_ERROR` 반환 + `WSAGetLastError() == WSAEWOULDBLOCK` — 에러가 아니라 "연결 진행 중".
- 실제 완료/실패 여부는 `select()`로 writable set 확인 또는 `WSAEventSelect`/`WSAAsyncSelect`로 `FD_CONNECT` 이벤트 수신.
- `select()`가 writable 신호 시 `getsockopt()` + `SO_ERROR` 옵션으로 실제 성공 여부(0=성공) 재확인.

#### 반환값 표

|상황|반환값|
|---|---|
|연결 성공|`0`|
|실패 (블로킹)|`SOCKET_ERROR` (-1), `WSAGetLastError()`로 원인 확인|
|논블로킹 진행중|`SOCKET_ERROR`, `WSAGetLastError() == WSAEWOULDBLOCK`|

주요 에러코드:

- `WSAECONNREFUSED` — 대상 포트 리스닝 없음 (RST 받음), 재전송 없이 즉시 실패
- `WSAETIMEDOUT` — SYN 재전송 다 소진, 계속 무응답
- `WSAENETUNREACH` / `WSAEHOSTUNREACH` — 라우팅 불가
- `WSAEISCONN` — 이미 연결된 소켓에 재호출

#### 블로킹 성공/실패 흐름

- SYN-ACK 정상 수신 → handshake 완료 → `0` 반환
- SYN을 여러 번 재전송(Windows 기본 2~3회, 지수 백오프)해도 무응답 → `WSAETIMEDOUT`
- 상대가 즉시 RST 응답 → 재전송 없이 바로 `WSAECONNREFUSED`

### 3-3. send()/recv() 반복

- handshake 완료 후 데이터 송수신.

### 3-4. closesocket()

- 서버/클라 공통. 소켓 리소스 해제 + FIN 절차 시작.

---

## 4. 실무 주의점 (winsock-chat 프로젝트 관점)

1. **backlog 값을 너무 작게 주지 말 것** — 작은 값(5, 10 등) 대신 `SOMAXCONN` 사용 권장. 작으면 클라이언트 몰릴 때 ACCEPT 큐 꽉 차서 새 연결 RST로 거부됨.
2. **accept() 루프가 다른 작업에 막히면 안 됨** — accept()와 각 클라이언트 처리(recv/send)를 같은 루프에서 순차 처리하면 안 되고, accept()는 별도 스레드/루프에서 계속 돌고 클라이언트 처리는 별도 스레드로 분리.
3. **listening 소켓과 accept 반환 소켓 혼동 금지** — 통신은 항상 accept()가 리턴한 새 소켓으로. N개 클라이언트 관리 시 벡터/맵으로 잘 추적해야 나중에 브로드캐스트할 때 안 꼬임.
4. **closesocket() 순서와 타이밍** — 클라이언트 스레드 종료 시 소켓을 닫기 전에 컨테이너에서 먼저 제거해야 함. 안 그러면 다른 스레드(브로드캐스트)가 이미 닫힌 소켓에 send() 시도해서 에러. 멀티스레드 환경에서 클라이언트 목록 접근 시 **mutex로 보호 필수**.
5. **recv()가 0을 반환하는 경우 처리 누락 주의** — 상대가 정상 종료(graceful close)하면 `recv()`가 `0` 반환 (에러 아님). 이 분기를 놓치면 좀비 스레드/소켓 발생 가능.
6. **WSAStartup/WSACleanup 페어링** — `WSAStartup()`과 `WSACleanup()`을 반드시 짝 맞춰 호출. RAII(`WinsockInit`)로 감싸면 자동 처리됨.
7. **SYN Flood는 애플리케이션 레벨에서 방어 불가** — SYN 큐는 OS 커널 관리 영역. 포트폴리오 수준에서 크게 신경 안 써도 되지만, 인터뷰에서 "이건 OS가 처리하는 영역"이라고 구분해 설명할 수 있으면 좋음.