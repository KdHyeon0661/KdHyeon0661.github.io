---
layout: post
title: 소켓프로그래밍 - Windows Winsock & IOCP
date: 2025-09-25 23:25:23 +0900
category: 소켓프로그래밍
---
## 19. Windows Winsock & IOCP

> 목표: 리눅스에서 배운 저수준 소켓 감각을 **Windows Winsock + IOCP**에 이식한다.  
> (1) **WSAStartup/WSACleanup, closesocket, 에러 체계**를 정확히 이해하고,  
> (2) **Overlapped I/O + IOCP**를 개념/수명/스레딩 관점에서 정리(리눅스 `epoll`과 비교),  
> (3) **실전 예제: IOCP 에코 서버**를 끝까지 구현,  
> (4) **포팅 체크리스트**로 실무 이전의 함정을 피한다.

---

### 19.1 Winsock 생명주기 & 기본 규칙

#### 19.1.1 시작·종료: `WSAStartup` / `WSACleanup`
- Winsock은 **프로세스당 초기화**가 필요하다.
- 보통 앱 시작 시 한 번 `WSAStartup(MAKEWORD(2,2), &data)`, 종료 시 `WSACleanup()`.

```cpp
// win_init.cpp (요지)
#define WIN32_LEAN_AND_MEAN
#include <winsock2.h>
#include <ws2tcpip.h>
#include <print>
#pragma comment(lib, "Ws2_32.lib")

int main() {
    WSADATA w{}; 
    int rc = WSAStartup(MAKEWORD(2,2), &w);
    if (rc != 0) { std::print(stderr, "WSAStartup: {}\n", rc); return 1; }
    // ... 소켓 사용 ...
    WSACleanup();
}
```

> 주의: **Thread** 종료 시 `WSACleanup()`를 자동 호출하지 않는다(프로세스 스코프). 중복 `WSAStartup()` 호출 시 **참조 카운트**가 증가하며, 카운트가 0이 되어야 진짜 종료된다.

#### 19.1.2 닫기와 반쯤 닫기: `closesocket`, `shutdown`
- Windows 소켓 타입은 `SOCKET`(정수형이지만 POSIX `fd`와 별개).
- 닫기: `closesocket(s)` (POSIX의 `close(fd)` **사용 금지**).
- 반쯤 닫기: `shutdown(s, SD_SEND)` / `SD_RECEIVE` / `SD_BOTH`.

#### 19.1.3 에러 체계: `WSAGetLastError` vs `errno`
- 대부분의 Winsock API는 실패 시 `SOCKET_ERROR`(-1) 또는 `FALSE` 반환, 에러는 **스레드 지역**으로 `WSAGetLastError()`에서 읽는다.
- 비동기(Overlapped) 경로에서 **즉시 완료가 아니면** `WSA_IO_PENDING`(= `ERROR_IO_PENDING`)이 정상이다.
- 비차단 리턴 값 비교표(대표):
  - 리눅스: `EINPROGRESS`, `EWOULDBLOCK`  
  - 윈도우: `WSAEINPROGRESS`, `WSAEWOULDBLOCK`, (Overlapped는) `WSA_IO_PENDING`

> 참고: **성공**이어도 **즉시 완료가 아니면** *바로 I/O가 끝난 게 아니다*. **완료 통지(IOCP)**를 기다려야 한다.

#### 19.1.4 주소 해석: `getaddrinfo/getnameinfo`
- 함수 시그니처/의미는 POSIX와 동일(헤더/링크만 다름).
- 링크: `Ws2_32.lib`, 헤더: `<ws2tcpip.h>`.

#### 19.1.5 옵션 차이(핵심 몇 가지)
- `SO_REUSEADDR` 의미가 리눅스와 다르다(Windows는 TIME_WAIT와 공유 의미가 약함).  
  **권장**: 서버 바인드에는 `SO_EXCLUSIVEADDRUSE`(또는 `SIO_EXCLUSIVEADDRUSE`)로 **독점**을 명시.
- Keepalive 세부값: 리눅스는 `TCP_KEEPIDLE/INTVL/KEEPCNT`,  
  윈도우는 `WSAIoctl(SIO_KEEPALIVE_VALS, ...)` 사용(아래 19.6 참조).

---

### 19.2 Overlapped I/O & IOCP 개념 — epoll과의 차이

#### 19.2.1 모델 요약
- **Overlapped I/O**: I/O 호출을 **비동기**로 시작 → 즉시 반환. I/O 완료 시 **완료 패킷** 생성.
- **IOCP(Completion Port)**: 커널이 완료 패킷을 한 큐에 모아둠. 워커 스레드는 `GetQueuedCompletionStatus(Ex)`로 **완료 이벤트**를 가져와 처리.
- **준비(readiness)** vs **완료(completion)**:
  - 리눅스 `epoll`은 “읽을 준비됨/쓸 준비됨”을 알려준다(레벨/엣지 트리거).
  - IOCP는 “**이(연산)가 끝났다**”를 알려준다. 그래서 **`WSARecv/WSASend`를 먼저 걸어놓고**(Overlapped), **완료**를 받는다.

#### 19.2.2 스레딩/동시성 직관
- IOCP는 큐 하나에 대해 **스레드 풀**을 매달아 소비한다.  
  `CreateIoCompletionPort(h, NULL, key, concurrency)`의 `concurrency`는 **실행 중인 워커 상한 힌트**(보통 CPU 코어 수).
- 워커는 **블로킹**으로 `GetQueuedCompletionStatus`를 기다리다, 완료 패킷을 꺼내 **후속 I/O**를 즉시 재무장(post)한다.

#### 19.2.3 수명 규칙(중요!)
- Overlapped 호출 시 넘긴 `OVERLAPPED*`와 **버퍼**는 **완료가 돌아올 때까지** 살아 있어야 한다(힙/풀에 두기).
- 소켓을 닫으면(pending I/O 포함) 해당 I/O는 `WSA_OPERATION_ABORTED`로 **완료 패킷이 떨어진다**. 워커가 그걸 수거하며 정리해야 **누수/유출**이 없다.
- **제로바이트 읽기**: `WSARecv`에 길이 0 버퍼를 올려두면 **데이터 도착 통지** 용으로 사용할 수 있다. (여기서는 일반 버퍼로 구현)

---

### 19.3 IOCP 에코 서버 — 단일 파일 실전 예제 (C++23, Win10+)

> 기능:  
> - IPv4/IPv6(듀얼) 리스닝  
> - `AcceptEx`로 비동기 accept N개 선공급  
> - 연결별 **수신→송신→다시 수신** 순환  
> - 워커 스레드 풀을 IOCP에 연결  
> - 안전한 종료(PostQueuedCompletionStatus로 종료 신호)

#### 19.3.1 빌드 & 실행
- 컴파일(MSVC):
  ```bat
  cl /EHsc /O2 /std:c++20 iocp_echo.cpp /link Ws2_32.lib Mswsock.lib
  ```
  > MSVC의 `/std:c++latest`를 쓰면 C++23 일부도 가능. 예제는 C++20에서 동작.
- 실행:
  ```bat
  iocp_echo.exe 0.0.0.0 9000
  ```
  테스트(다른 콘솔):
  ```bat
  # Windows
  ncat 127.0.0.1 9000
  # 또는 Linux/mac에서 telnet/nc로 접속
  ```

#### 19.3.2 전체 코드

```cpp
// iocp_echo.cpp — IOCP 기반 에코 서버(교육용, 단일 파일)
// 빌드: cl /EHsc /O2 /std:c++20 iocp_echo.cpp /link Ws2_32.lib Mswsock.lib
#define WIN32_LEAN_AND_MEAN
#define _WIN32_WINNT 0x0A00
#include <winsock2.h>
#include <mswsock.h>
#include <ws2tcpip.h>
#include <windows.h>
#include <cstdio>
#include <cstdlib>
#include <cstdint>
#include <string>
#include <vector>
#include <thread>
#include <atomic>
#include <print>

#pragma comment(lib, "Ws2_32.lib")
#pragma comment(lib, "Mswsock.lib")

// ---- AcceptEx / GetAcceptExSockaddrs 함수 포인터 로드 ----
LPFN_ACCEPTEX              pAcceptEx              = nullptr;
LPFN_GETACCEPTEXSOCKADDRS  pGetAcceptExSockaddrs  = nullptr;

bool load_mswsock_funcs(SOCKET s) {
    DWORD bytes = 0;
    GUID g1 = WSAID_ACCEPTEX;
    if (WSAIoctl(s, SIO_GET_EXTENSION_FUNCTION_POINTER, &g1, sizeof(g1),
                 &pAcceptEx, sizeof(pAcceptEx), &bytes, NULL, NULL) != 0) return false;
    GUID g2 = WSAID_GETACCEPTEXSOCKADDRS;
    if (WSAIoctl(s, SIO_GET_EXTENSION_FUNCTION_POINTER, &g2, sizeof(g2),
                 &pGetAcceptExSockaddrs, sizeof(pGetAcceptExSockaddrs), &bytes, NULL, NULL) != 0) return false;
    return true;
}

// ---- 컨텍스트/오퍼레이션 타입 ----
enum class OpType { OP_ACCEPT, OP_RECV, OP_SEND, OP_EXIT };

struct PerIo {
    OVERLAPPED ol{};
    OpType     op{OpType::OP_RECV};
    WSABUF     buf{};
    std::vector<char> storage; // 실제 데이터 저장소
    SOCKET     sock{INVALID_SOCKET}; // 대상 소켓(accept 이후)
    // accept에서 주소 파싱용 버퍼: remote/local sockaddr 보관
    std::vector<char> addrbuf;
};

struct PerConn {
    SOCKET sock{INVALID_SOCKET};
    bool   alive{false};
};

static HANDLE g_iocp = NULL;
static std::atomic<bool> g_running{true};

// ---- 유틸 ----
static void die(const char* where, int wsa = WSAGetLastError()) {
    std::print(stderr, "{}: WSA={}\n", where, wsa);
    std::exit(1);
}

static void set_no_delay(SOCKET s) {
    BOOL on = TRUE; setsockopt(s, IPPROTO_TCP, TCP_NODELAY, (char*)&on, sizeof(on));
}

static void set_exclusive_addr(SOCKET s) {
    // Windows 권장: EXCLUSIVEADDRUSE (재기동 충돌 완화)
    BOOL on = TRUE; setsockopt(s, SOL_SOCKET, SO_EXCLUSIVEADDRUSE, (char*)&on, sizeof(on));
}

// ---- 리스너 생성(듀얼스택) ----
static SOCKET make_listener(const char* host, const char* port) {
    addrinfo hints{}, *res=nullptr;
    hints.ai_family = AF_UNSPEC;
    hints.ai_socktype = SOCK_STREAM;
    hints.ai_flags = AI_PASSIVE | AI_ADDRCONFIG;

    if (getaddrinfo(host[0]?host:nullptr, port, &hints, &res) != 0) return INVALID_SOCKET;

    SOCKET listen_sock = INVALID_SOCKET;
    for (auto* ai = res; ai; ai = ai->ai_next) {
        SOCKET s = WSASocket(ai->ai_family, ai->ai_socktype, ai->ai_protocol, NULL, 0, WSA_FLAG_OVERLAPPED);
        if (s == INVALID_SOCKET) continue;

        set_exclusive_addr(s);

        // Windows는 기본 dual-stack 허용(대체로 IPV6_V6ONLY=0). 필요 시 명시:
        if (ai->ai_family == AF_INET6) {
            DWORD off = 0; setsockopt(s, IPPROTO_IPV6, IPV6_V6ONLY, (char*)&off, sizeof(off));
        }

        if (bind(s, ai->ai_addr, (int)ai->ai_addrlen) == 0 && listen(s, SOMAXCONN) == 0) {
            listen_sock = s; break;
        }
        closesocket(s);
    }
    freeaddrinfo(res);
    return listen_sock;
}

// ---- IOCP 연결 함수 ----
static void attach_to_iocp(HANDLE iocp, SOCKET s, ULONG_PTR key) {
    HANDLE r = CreateIoCompletionPort((HANDLE)s, iocp, key, 0);
    if (!r) die("CreateIoCompletionPort(sock)");
}

// ---- AcceptEx 사전투입 ----
static const DWORD ADDRBUF = (DWORD)(sizeof(SOCKADDR_IN6) + 16) * 2;

static PerIo* post_accept(SOCKET listen_sock) {
    SOCKET as = WSASocket(AF_INET6, SOCK_STREAM, 0, NULL, 0, WSA_FLAG_OVERLAPPED);
    if (as == INVALID_SOCKET) as = WSASocket(AF_INET, SOCK_STREAM, 0, NULL, 0, WSA_FLAG_OVERLAPPED);
    if (as == INVALID_SOCKET) return nullptr;

    auto* io = new PerIo{};
    io->op = OpType::OP_ACCEPT;
    io->sock = as;
    io->addrbuf.resize(ADDRBUF);
    io->storage.resize(0); // AcceptEx는 데이터 버퍼 옵션도 있으나 여기선 주소만

    DWORD bytes = 0;
    ZeroMemory(&io->ol, sizeof(io->ol));

    BOOL ok = pAcceptEx(listen_sock, as,
                        io->addrbuf.data(), 0,
                        ADDRBUF/2, ADDRBUF/2,
                        &bytes, &io->ol);
    if (!ok) {
        int e = WSAGetLastError();
        if (e != ERROR_IO_PENDING) {
            closesocket(as);
            delete io;
            return nullptr;
        }
    }
    return io;
}

// ---- 수신/송신 I/O 재무장 ----
static bool post_recv(SOCKET s, PerIo* io, size_t cap = 8192) {
    io->op = OpType::OP_RECV;
    io->storage.resize(cap);
    io->buf.buf = io->storage.data();
    io->buf.len = (ULONG)io->storage.size();
    ZeroMemory(&io->ol, sizeof(io->ol));
    DWORD flags = 0, recvd = 0;
    int rc = WSARecv(s, &io->buf, 1, &recvd, &flags, &io->ol, NULL);
    if (rc == 0) {
        // 즉시 완료 (드묾). IOCP로도 패킷이 올 수 있으니 그대로 둔다.
        return true;
    }
    int e = WSAGetLastError();
    if (e == WSA_IO_PENDING) return true;
    return false;
}

static bool post_send(SOCKET s, PerIo* io, size_t n) {
    io->op = OpType::OP_SEND;
    io->buf.buf = io->storage.data();
    io->buf.len = (ULONG)n;
    ZeroMemory(&io->ol, sizeof(io->ol));
    DWORD sent = 0;
    int rc = WSASend(s, &io->buf, 1, &sent, 0, &io->ol, NULL);
    if (rc == 0) {
        // 즉시 완료 가능
        return true;
    }
    int e = WSAGetLastError();
    if (e == WSA_IO_PENDING) return true;
    return false;
}

// ---- 워커 스레드 ----
static void worker(HANDLE iocp, SOCKET listen_sock, int id) {
    while (g_running.load(std::memory_order_relaxed)) {
        DWORD bytes = 0;
        ULONG_PTR key = 0;
        LPOVERLAPPED pol = nullptr;
        BOOL ok = GetQueuedCompletionStatus(iocp, &bytes, &key, &pol, INFINITE);
        if (!ok && pol==nullptr) {
            // 큐 에러
            continue;
        }
        if (key == (ULONG_PTR)OpType::OP_EXIT) {
            // 종료 신호
            break;
        }
        PerIo* io = reinterpret_cast<PerIo*>(pol); // 우리가 넣은 OVERLAPPED 컨테이너
        if (!io) continue;

        switch (io->op) {
            case OpType::OP_ACCEPT: {
                // 새 연결
                SOCKET as = io->sock;
                // 필수: Accept된 소켓에 컨텍스트 업데이트(로컬 옵션/getsockname 가능케)
                setsockopt(as, SOL_SOCKET, SO_UPDATE_ACCEPT_CONTEXT, (char*)&listen_sock, sizeof(listen_sock));
                set_no_delay(as);
                attach_to_iocp(iocp, as, (ULONG_PTR)as);

                // 클라이언트/서버 주소 파싱(원하면 로그)
                SOCKADDR *la=nullptr,*ra=nullptr; int ll=0, rl=0;
                pGetAcceptExSockaddrs(io->addrbuf.data(), 0, ADDRBUF/2, ADDRBUF/2,
                                      &la, &ll, &ra, &rl);

                // 첫 수신을 올린다
                auto* rio = new PerIo{}; rio->sock = as;
                if (!post_recv(as, rio)) {
                    closesocket(as);
                    delete rio;
                }

                // 다음 AcceptEx를 즉시 투입(accept 파이프라인 유지)
                auto* a2 = post_accept(listen_sock);
                if (!a2) {
                    // 실패 시 약간 쉬어가도 된다(여기선 생략)
                }
                delete io; // AcceptEx용 IO 타기종료
            } break;
            case OpType::OP_RECV: {
                SOCKET s = io->sock;
                if (bytes == 0) {
                    // peer closed
                    closesocket(s);
                    delete io;
                    break;
                }
                // 받은 만큼 에코 송신
                if (!post_send(s, io, bytes)) {
                    closesocket(s);
                    delete io;
                }
                // 주의: 같은 io 객체를 send로 재사용했다.
                // 별도 io 객체로 분리해도 된다(교육용 간소화).
            } break;
            case OpType::OP_SEND: {
                SOCKET s = io->sock;
                // 다시 수신을 걸어 순환 유지
                if (!post_recv(s, io)) {
                    closesocket(s);
                    delete io;
                }
            } break;
            default: {
                delete io;
            } break;
        }
    }
}

// ---- 메인 ----
int main(int argc, char** argv) {
    const char* host = (argc>1? argv[1] : "0.0.0.0");
    const char* port = (argc>2? argv[2] : "9000");
    SYSTEM_INFO si{}; GetSystemInfo(&si);
    DWORD ncpu = si.dwNumberOfProcessors;

    WSADATA w{}; if (WSAStartup(MAKEWORD(2,2), &w)!=0) die("WSAStartup");

    SOCKET listen_sock = make_listener(host, port);
    if (listen_sock == INVALID_SOCKET) die("listen");

    // 확장 함수 로드
    if (!load_mswsock_funcs(listen_sock)) die("load_mswsock_funcs");

    // IOCP 생성 & 리스너 연결
    g_iocp = CreateIoCompletionPort(INVALID_HANDLE_VALUE, NULL, 0, ncpu);
    if (!g_iocp) die("CreateIoCompletionPort(main)");
    attach_to_iocp(g_iocp, listen_sock, (ULONG_PTR)listen_sock);

    std::print("[iocp-echo] listen {}:{}  (workers = {})\n", host, port, ncpu*2);

    // AcceptEx 선공급(동시 N개)
    const int ACCEPT_PIPELINE = 64;
    std::vector<PerIo*> accepts; accepts.reserve(ACCEPT_PIPELINE);
    for (int i=0;i<ACCEPT_PIPELINE;i++) {
        if (auto* a = post_accept(listen_sock)) accepts.push_back(a);
    }

    // 워커 스레드 시작
    std::vector<std::jthread> th;
    int workers = (int)(ncpu * 2);
    for (int i=0;i<workers;i++) {
        th.emplace_back([=]{ worker(g_iocp, listen_sock, i); });
    }

    // 간단한 종료 대기(CTRL+C에서 강종해도 됨). 여기선 입력 대기해서 종료.
    std::print("press ENTER to stop...\n");
    (void)getchar();
    g_running.store(false);
    // 종료 패킷을 워커 수만큼 post
    for (int i=0;i<workers;i++) {
        PostQueuedCompletionStatus(g_iocp, 0, (ULONG_PTR)OpType::OP_EXIT, NULL);
    }

    // 정리
    for (auto& t : th) { if (t.joinable()) t.join(); }
    closesocket(listen_sock);
    CloseHandle(g_iocp);
    WSACleanup();
    return 0;
}
```

> 설명 포인트
> - **Accept 파이프라인**: `post_accept`를 다수 걸어두면 커널이 큐에서 받아 즉시 완료시킨다(고성능).
> - **수신→송신 순환**: RECV 완료 패킷에서 같은 `PerIo`를 SEND로 전환(교육용 간소화). 실무는 **수신/송신 별도 오브젝트**가 관리/튜닝에 유리.
> - **종료 신호**: `PostQueuedCompletionStatus`로 **유저 정의 완료 패킷**을 보낼 수 있다(여기선 `key=OP_EXIT`).

---

### 19.4 epoll vs IOCP — 사고방식 차이 한 장표

| 항목 | epoll(리눅스) | IOCP(윈도우) |
|---|---|---|
| 모델 | **준비(ready)** 통지 | **완료(completion)** 통지 |
| API 핵심 | `epoll_wait`, `EPOLLIN/OUT` | `GetQueuedCompletionStatus(Ex)` |
| I/O 호출 | 보통 논블로킹 `read/write` 반복 | **Overlapped** `WSARecv/WSASend`를 먼저 걸고 완료 큐 수거 |
| 스레딩 | 이벤트 루프(1~N) + 워커 | **완료 큐**에 워커 스레드 직접 대기 |
| 수명 | 버퍼는 호출/반환 스코프 | Overlapped 버퍼는 **완료 시점까지 유지** 필요 |
| accept | `accept4` + non-blocking | `AcceptEx`(확장 함수)로 **비동기 대량** |
| 타임아웃 | `epoll_wait` 타임아웃 or timerfd | `GQCS(Ex)` 타임아웃 + timer queue / `WaitableTimer` |
| 캔슬 | `close(fd)`로 `EPOLLHUP` 등 | `closesocket()` → pending I/O는 `WSA_OPERATION_ABORTED`로 **완료**됨 |

---

### 19.5 포팅 전략 요약 체크리스트

#### 19.5.1 API 치환·주의점
- 종료/반쯤 종료: `close` → `closesocket`, `shutdown(SHUT_WR)` → `shutdown(SD_SEND)`
- 논블로킹: `fcntl(F_SETFL,O_NONBLOCK)` → `ioctlsocket(FIONBIO, ...)` (Overlapped면 필요 없음)
- 에러 코드: `errno` → `WSAGetLastError()` / `GetLastError()`
- 타임아웃: `setsockopt(SO_RCVTIMEO/SO_SNDTIMEO)` 가능하지만, Overlapped+IOCP라면 **타이머**를 별도 운용(완료 대기 제한 or `CancelIoEx`)
- 멀티프로세스: `fork` 없음. 대신 프로세스보다는 **멀티스레드**가 일반적.
- 파일 디스크립터 혼용 금지: `SOCKET`은 POSIX `fd`와 호환 아님(단, `HANDLE`로 캐스팅해 IOCP에 연결).

#### 19.5.2 소켓 옵션 대응
- `TCP_NODELAY` 동일.
- **주소 재사용**: 리눅스의 `SO_REUSEADDR`=TIME_WAIT 공유와 의미 다름 → Windows는 **`SO_EXCLUSIVEADDRUSE` 권장**.
- Keepalive:  
  - `setsockopt(SO_KEEPALIVE, ...)`로 on/off  
  - **간격/카운트는** `WSAIoctl(SIO_KEEPALIVE_VALS, ...)`:
    ```cpp
    struct tcp_keepalive { u_long onoff, keepalivetime, keepaliveinterval; };
    tcp_keepalive ka{1, 60000, 5000}; // 60s idle, 5s interval
    DWORD bytes=0;
    WSAIoctl(s, SIO_KEEPALIVE_VALS, &ka, sizeof(ka), NULL, 0, &bytes, NULL, NULL);
    ```
- UDP 특이점: `SIO_UDP_CONNRESET`을 **꺼야** ICMP Port Unreachable 시 `WSAECONNRESET`이 계속 터지는 걸 막는다.
  ```cpp
  BOOL bNewBehavior = FALSE;
  DWORD dwBytesReturned = 0;
  WSAIoctl(udp, SIO_UDP_CONNRESET, &bNewBehavior, sizeof(bNewBehavior),
           NULL, 0, &dwBytesReturned, NULL, NULL);
  ```

#### 19.5.3 IPv6/듀얼스택
- Windows는 기본적으로 **듀얼스택 허용**(v4-mapped).  
  필요 시 `IPV6_V6ONLY=1`로 IPv6 전용으로 고정.
- 링크-로컬 통신 시 `scope id`(인터페이스 인덱스)는 리눅스와 동일하게 `sin6_scope_id`에 지정.

#### 19.5.4 IOCP로의 구조 변환(핵심 4단계)
1) **소켓 생성 시 Overlapped**: `WSASocket(..., WSA_FLAG_OVERLAPPED)`
2) **IOCP 생성/연결**: `CreateIoCompletionPort` + `attach_to_iocp(sock)`
3) **I/O를 먼저 건다**: `WSARecv`/`WSASend`/`AcceptEx` 호출 → `WSA_IO_PENDING`이면 OK
4) **워커에서 완료 수거**: `GetQueuedCompletionStatus` 루프에서 후속 I/O 재무장

---

### 19.6 실무 팁(운영/디버깅)

- **완료가 0바이트**인 `WSARecv` → peer가 **정상 종료**(FIN)했을 가능성 큼(리눅스와 동일).
- **취소/종료 시나리오**: `closesocket()` 하면 pending I/O에 대해 **반드시** 완료 패킷이 떨어진다(오류 `WSA_OPERATION_ABORTED`). 워커는 이를 받아 **힙 객체 해제**.
- **대역폭/RTT 지표**: 리눅스 `TCP_INFO`와 같은 통합 API는 없지만, **PerfMon/ETW**(Event Tracing for Windows)와 `netsh trace`, `wpr/wpa`로 네트워크/TCP 레벨을 추적 가능.
- **Accept 스로틀링**: 과도한 허용을 피하려면 AcceptEx 파이프라인 깊이를 **동적**으로 조절(큐 길이/메모리 기반).
- **메모리 풀**: Overlapped/버퍼를 **풀링**하면 할당/해제 비용과 힙 단편화를 크게 줄일 수 있다.
- **IOCP + TLS**: Windows SChannel(네이티브) 또는 OpenSSL(소켓 Overlapped 그대로) 중 택. OpenSSL은 `WSARecv/WSASend`를 SSL BIO와 잘 이어주면 된다.

---

### 19.7 (보너스) IOCP + UDP 한눈에
- UDP도 IOCP로 처리 가능: `WSARecvFrom`/`WSASendTo`를 Overlapped로 올려두고 완료 수거.
- **메시지 경계 보존**(UDP 특성). 필요한 만큼 **버퍼를 넉넉히** 두고, 짧으면 잘라 쓰기/길면 폐기 정책.
- **SIO_UDP_CONNRESET** 비활성은 사실상 필수(위 19.5.2).

---

### 19.8 미니 수식 — 워커 수와 병렬성 감 잡기
- 워커 수를 코어 수 \(C\)라 할 때, I/O 대기/처리 비율 \(\alpha\)에 대한 직관:
  $$
  N_{\text{workers}} \approx C \cdot \left(1 + \frac{\text{I/O wait}}{\text{CPU work}}\right)
  $$
  CPU 작업이 얕고 대부분 I/O 대기라면 \(2C\) 내외가 흔한 출발점이다(예제는 \(2\times C\)).

---

### 19.9 마무리

- Windows의 네트워킹은 **Overlapped + IOCP**가 정석이다.  
- 핵심 전환은 **“준비(readiness) 루프” → “완료(completion) 소비”**로의 사고 변경.  
- 이 장의 **IOCP 에코 서버**를 뼈대로, 10~11장의 **프레이밍/상태기계**를 얹고, 14장의 **TLS**를 연결하면  
  리눅스에서 구축한 구조를 **Windows 네이티브 성능**으로 그대로 가져올 수 있다.
