---
layout: post
title: 데이터 통신 - Introduction to Application Layer (2)
date: 2024-09-05 21:20:23 +0900
category: DataCommunication
---
# Chapter 25.3–25.4 Iterative Programming in C / Java — UDP와 TCP 기반 기본기

이 글은 앞에서 정리한 **Application Layer, 클라이언트–서버 프로그래밍 개요**를 바탕으로,
실제 코드 수준에서 **“iterative 서버/클라이언트”를 C와 Java로 구현하는 방법**을 정리한다.

- **25.3 Iterative Programming in C**
  - General issues
  - Iterative programming using UDP
  - Iterative programming using TCP
- **25.4 Iterative Programming in Java**
  - Addresses and ports
  - Iterative programming using UDP
  - Iterative programming using TCP

여기서 말하는 **iterative 서버**란,
> “한 번에 하나의 클라이언트 요청만 처리하고, 끝나면 그 다음 요청을 받는 구조”
를 의미한다.

실서비스에서는 보통 **concurrent 서버(멀티프로세스/멀티스레드/이벤트 기반)** 를 쓰지만,
소켓 API를 이해하기 위한 첫 단계로 iterative 구조가 가장 단순하고 직관적이다.

---

## Iterative Programming in C

### General Issues — C 네트워크 코드에서 항상 고민해야 할 것들

C로 소켓 프로그래밍을 할 때는 다음 공통 이슈들을 먼저 짚고 넘어가야 한다.

#### 1) 주소 구조와 IPv4/IPv6

- IPv4: `struct sockaddr_in` (AF_INET)
- IPv6: `struct sockaddr_in6` (AF_INET6)
- 두 프로토콜을 모두 지원하려면 `getaddrinfo()` + `struct addrinfo` + `struct sockaddr_storage`를 쓰는 것이 현대적인 방식이다.

예: IPv4만 가정한 단순 예제에서 자주 쓰는 구조체:

```c
struct sockaddr_in addr;
memset(&addr, 0, sizeof(addr));
addr.sin_family = AF_INET;
addr.sin_port   = htons(9999);
addr.sin_addr.s_addr = htonl(INADDR_ANY); // 0.0.0.0
```

- `htons`, `htonl` 등은 **host byte order → network byte order** 변환 함수다.
- network byte order는 항상 **big-endian** 이다.

IPv6까지 확장하려면 대신 `getaddrinfo()`를 사용하여
주소 해석과 구조체 세팅을 한 번에 처리하는 것이 좋다.

#### 2) blocking I/O와 iterative 서버

우리가 만들 iterative 서버는 **blocking 소켓**을 사용한다.

- `recvfrom`, `recv`, `accept` 등은 데이터가 준비될 때까지 블록(block)한다.
- iterative 구조에서는 이것이 오히려 단순해서 좋다:
  - 하나의 요청을 처리하는 동안 다른 클라이언트는 대기하게 되지만,
  - 코드 구조가 직관적이다.

#### 3) 오류 처리

C에서는 모든 시스템 호출에 대해 **반환값 검사**가 필수다.

- `socket` / `bind` / `listen` / `accept` / `recvfrom` / `sendto` / `recv` / `send` / `connect` 등
- 실패하면 `-1`을 반환하고, `errno`에 오류 코드가 설정된다.

편의상 예제에서는 에러 발생 시 `perror()` 호출 후 `exit(1)` 하는 스타일을 많이 쓴다.

```c
int fd = socket(AF_INET, SOCK_DGRAM, 0);
if (fd < 0) {
    perror("socket");
    exit(1);
}
```

실서비스에서는 오류 종류에 따라 재시도, 로그, graceful shutdown 등을 세분화해야 한다.

#### 4) 메시지 경계 vs 바이트 스트림

- UDP: **데이터그램 단위**가 유지된다. 한 번 `sendto()` 한 것이 `recvfrom()` 한 번에 대응.
- TCP: **바이트 스트림**이다.
  - 여러 `send()` 호출이 한 `recv()`에 섞여 들어가거나,
    한 `send()` 가 여러 `recv()`에 나누어질 수 있다.
  - 따라서 TCP에서 “메시지 단위”를 구현하려면,
    반드시 애플리케이션 레벨에서 길이 헤더, 구분자(예: `\n`) 등을 정의해야 한다.

이 차이는 iterative 서버든 concurrent 서버든 항상 중요한 디자인 포인트다.

---

### Iterative Programming using UDP (C)

#### 서버 구조 개요

UDP iterative 서버의 흐름:

1. `socket(AF_INET, SOCK_DGRAM, 0)` 로 소켓 생성
2. `bind()` 로 로컬 포트에 바인딩
3. 무한 루프:
   - `recvfrom()` 으로 클라이언트 데이터 수신 (발신 주소 포함)
   - 처리
   - `sendto()` 로 같은 주소에 응답 전송

연결 개념이 없으므로, **클라이언트 수만큼 소켓을 생성할 필요가 없고**,
항상 **하나의 소켓**으로 모든 클라이언트를 처리한다.

#### 예제: 대문자 변환 UDP 서버 (IPv4 단순 버전)

```c
/* udp_upper_server.c */

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <ctype.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <sys/socket.h>

#define BUF_SIZE 2048
#define SERVER_PORT 9999

int main(void) {
    int sockfd;
    struct sockaddr_in servaddr, cliaddr;
    socklen_t cliaddr_len;
    char buf[BUF_SIZE];

    /* 1. 소켓 생성 (IPv4, UDP) */
    sockfd = socket(AF_INET, SOCK_DGRAM, 0);
    if (sockfd < 0) {
        perror("socket");
        exit(EXIT_FAILURE);
    }

    /* 2. 서버 주소 설정 및 바인드 */
    memset(&servaddr, 0, sizeof(servaddr));
    servaddr.sin_family      = AF_INET;
    servaddr.sin_port        = htons(SERVER_PORT);
    servaddr.sin_addr.s_addr = htonl(INADDR_ANY);  // 0.0.0.0

    if (bind(sockfd, (struct sockaddr *)&servaddr, sizeof(servaddr)) < 0) {
        perror("bind");
        close(sockfd);
        exit(EXIT_FAILURE);
    }

    printf("[*] UDP Uppercase Server listening on port %d\n", SERVER_PORT);

    /* 3. iterative 루프 */
    while (1) {
        cliaddr_len = sizeof(cliaddr);
        ssize_t n = recvfrom(sockfd, buf, sizeof(buf) - 1, 0,
                             (struct sockaddr *)&cliaddr, &cliaddr_len);
        if (n < 0) {
            perror("recvfrom");
            continue;   // 에러 발생 시 다음 루프로
        }

        buf[n] = '\0'; // 문자열로 처리

        char addrstr[INET_ADDRSTRLEN];
        inet_ntop(AF_INET, &cliaddr.sin_addr, addrstr, sizeof(addrstr));
        printf("[+] Received from %s:%d: '%s'\n",
               addrstr, ntohs(cliaddr.sin_port), buf);

        /* 4. 대문자 변환 */
        for (ssize_t i = 0; i < n; i++) {
            buf[i] = (char)toupper((unsigned char)buf[i]);
        }

        /* 5. 응답 전송 */
        ssize_t sent = sendto(sockfd, buf, n, 0,
                              (struct sockaddr *)&cliaddr, cliaddr_len);
        if (sent < 0) {
            perror("sendto");
        }
    }

    /* 이 코드는 사실상 여기까지 오지 않음 */
    close(sockfd);
    return 0;
}
```

클라이언트 (간단 UDP 클라이언트):

```c
/* udp_upper_client.c */

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <sys/socket.h>

#define BUF_SIZE 2048

int main(int argc, char *argv[]) {
    int sockfd;
    struct sockaddr_in servaddr;
    char buf[BUF_SIZE];

    if (argc != 3) {
        fprintf(stderr, "Usage: %s <server_ip> <port>\n", argv[0]);
        exit(EXIT_FAILURE);
    }

    const char *server_ip = argv[1];
    int port = atoi(argv[2]);

    sockfd = socket(AF_INET, SOCK_DGRAM, 0);
    if (sockfd < 0) {
        perror("socket");
        exit(EXIT_FAILURE);
    }

    memset(&servaddr, 0, sizeof(servaddr));
    servaddr.sin_family = AF_INET;
    servaddr.sin_port   = htons(port);
    if (inet_pton(AF_INET, server_ip, &servaddr.sin_addr) <= 0) {
        perror("inet_pton");
        close(sockfd);
        exit(EXIT_FAILURE);
    }

    while (1) {
        printf("> ");
        fflush(stdout);

        if (!fgets(buf, sizeof(buf), stdin))
            break;

        size_t len = strlen(buf);
        if (buf[len - 1] == '\n') {
            buf[len - 1] = '\0';
            len--;
        }

        /* 서버로 전송 */
        if (sendto(sockfd, buf, len, 0,
                   (struct sockaddr *)&servaddr, sizeof(servaddr)) < 0) {
            perror("sendto");
            break;
        }

        /* 서버에서 응답 수신 */
        struct sockaddr_in from;
        socklen_t fromlen = sizeof(from);
        ssize_t n = recvfrom(sockfd, buf, sizeof(buf) - 1, 0,
                             (struct sockaddr *)&from, &fromlen);
        if (n < 0) {
            perror("recvfrom");
            break;
        }
        buf[n] = '\0';

        printf("RECV: '%s'\n", buf);
    }

    close(sockfd);
    return 0;
}
```

이 구조는 매우 전형적인 **UDP iterative 서버/클라이언트 예제**다.

---

### Iterative Programming using TCP (C)

#### 서버 구조 개요

TCP iterative 서버의 흐름:

1. `socket(AF_INET, SOCK_STREAM, 0)`
2. `bind()`
3. `listen()`
4. 반복:
   - `accept()` 로 새 연결 수락 (blocking)
   - 해당 연결에서:
     - 루프: `recv()` → 처리 → `send()`
   - 연결 종료 후 `close()`
   - 다시 `accept()` 로 돌아감

#### 예제: iterative TCP 에코 서버

```c
/* tcp_echo_server_iter.c */

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <sys/socket.h>

#define BUF_SIZE 4096
#define SERVER_PORT 8888

int main(void) {
    int listenfd, connfd;
    struct sockaddr_in servaddr, cliaddr;
    socklen_t cliaddr_len;
    char buf[BUF_SIZE];

    /* 1. listen용 소켓 생성 */
    listenfd = socket(AF_INET, SOCK_STREAM, 0);
    if (listenfd < 0) {
        perror("socket");
        exit(EXIT_FAILURE);
    }

    /* SO_REUSEADDR 옵션: 재실행 시 bind 에러 줄이기 */
    int opt = 1;
    if (setsockopt(listenfd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt)) < 0) {
        perror("setsockopt");
        close(listenfd);
        exit(EXIT_FAILURE);
    }

    /* 2. 서버 주소 설정 및 bind */
    memset(&servaddr, 0, sizeof(servaddr));
    servaddr.sin_family      = AF_INET;
    servaddr.sin_port        = htons(SERVER_PORT);
    servaddr.sin_addr.s_addr = htonl(INADDR_ANY);

    if (bind(listenfd, (struct sockaddr *)&servaddr, sizeof(servaddr)) < 0) {
        perror("bind");
        close(listenfd);
        exit(EXIT_FAILURE);
    }

    /* 3. listen */
    if (listen(listenfd, 5) < 0) {
        perror("listen");
        close(listenfd);
        exit(EXIT_FAILURE);
    }

    printf("[*] Iterative TCP Echo Server listening on port %d\n", SERVER_PORT);

    /* 4. iterative accept-serve 루프 */
    while (1) {
        cliaddr_len = sizeof(cliaddr);
        connfd = accept(listenfd, (struct sockaddr *)&cliaddr, &cliaddr_len);
        if (connfd < 0) {
            perror("accept");
            continue;
        }

        char addrstr[INET_ADDRSTRLEN];
        inet_ntop(AF_INET, &cliaddr.sin_addr, addrstr, sizeof(addrstr));
        printf("[+] New connection from %s:%d\n",
               addrstr, ntohs(cliaddr.sin_port));

        /* 이 연결에 대해 echo 처리 루프 */
        while (1) {
            ssize_t n = recv(connfd, buf, sizeof(buf), 0);
            if (n < 0) {
                perror("recv");
                break;
            }
            if (n == 0) {
                /* 클라이언트 종료 */
                printf("[-] Client closed connection: %s:%d\n",
                       addrstr, ntohs(cliaddr.sin_port));
                break;
            }

            /* 그대로 echo */
            ssize_t sent = 0;
            while (sent < n) {
                ssize_t m = send(connfd, buf + sent, n - sent, 0);
                if (m <= 0) {
                    perror("send");
                    goto done_with_conn;
                }
                sent += m;
            }
        }

    done_with_conn:
        close(connfd);
    }

    close(listenfd);
    return 0;
}
```

클라이언트는 간단한 TCP 클라이언트:

```c
/* tcp_echo_client.c */

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <sys/socket.h>

#define BUF_SIZE 4096

int main(int argc, char *argv[]) {
    int sockfd;
    struct sockaddr_in servaddr;
    char buf[BUF_SIZE];

    if (argc != 3) {
        fprintf(stderr, "Usage: %s <server_ip> <port>\n", argv[0]);
        exit(EXIT_FAILURE);
    }

    const char *server_ip = argv[1];
    int port = atoi(argv[2]);

    sockfd = socket(AF_INET, SOCK_STREAM, 0);
    if (sockfd < 0) {
        perror("socket");
        exit(EXIT_FAILURE);
    }

    memset(&servaddr, 0, sizeof(servaddr));
    servaddr.sin_family = AF_INET;
    servaddr.sin_port   = htons(port);

    if (inet_pton(AF_INET, server_ip, &servaddr.sin_addr) <= 0) {
        perror("inet_pton");
        close(sockfd);
        exit(EXIT_FAILURE);
    }

    if (connect(sockfd, (struct sockaddr *)&servaddr, sizeof(servaddr)) < 0) {
        perror("connect");
        close(sockfd);
        exit(EXIT_FAILURE);
    }

    printf("[*] Connected to %s:%d\n", server_ip, port);

    while (1) {
        printf("> ");
        fflush(stdout);

        if (!fgets(buf, sizeof(buf), stdin))
            break;

        size_t len = strlen(buf);
        if (buf[len - 1] == '\n') {
            /* echo 서버에 줄바꿈도 보내고 싶으면 이 줄 제거 */
            buf[len - 1] = '\0';
            len--;
        }

        if (send(sockfd, buf, len, 0) < 0) {
            perror("send");
            break;
        }

        ssize_t n = recv(sockfd, buf, sizeof(buf) - 1, 0);
        if (n <= 0) {
            if (n < 0) perror("recv");
            break;
        }
        buf[n] = '\0';
        printf("RECV: '%s'\n", buf);
    }

    close(sockfd);
    return 0;
}
```

이 예제는 **iterative 서버의 한계**도 잘 보여 준다:

- 한 클라이언트가 연결된 상태에서 아무 데이터도 보내지 않고 오래 버티면,
  - 서버는 `recv()`에서 막혀 있고,
  - 다음 클라이언트의 `accept()`는 그 클라이언트가 연결 종료할 때까지 지연된다.
- 그래서 실제 서비스에서는 **멀티프로세스/스레드/이벤트 기반**으로 확장해야 한다.

---

## Iterative Programming in Java

Java는 C에 비해 **네트워크 API를 객체 지향적으로 감싸 놓은 언어**다.

- UDP: `DatagramSocket`, `DatagramPacket`
- TCP:
  - 서버: `ServerSocket`
  - 클라이언트: `Socket`
- 주소/포트: `InetAddress`, `InetSocketAddress` 등

### Addresses and Ports — Java에서 주소와 포트 다루기

#### InetAddress

`InetAddress`는 IP 주소를 나타내는 클래스다.

- IPv4, IPv6 모두 지원
- DNS를 통해 이름 → IP 주소를 매핑하는 메서드 제공

예:

```java
InetAddress addr1 = InetAddress.getByName("example.com");
InetAddress addr2 = InetAddress.getByName("192.0.2.10"); // IPv4 literal
InetAddress addr3 = InetAddress.getByName("2001:db8::1"); // IPv6 literal
```

여러 RFC에서 IPv4/IPv6 주소 표기, DNS, 표준 포트가 정의되어 있으며
Java는 이를 구현한 라이브러리를 제공한다.

#### InetSocketAddress

IP 주소 + 포트를 표현하는 편리한 래퍼 클래스.
`Socket`이나 `ServerSocket` 생성 시 자주 사용한다.

```java
InetSocketAddress sockAddr =
    new InetSocketAddress("0.0.0.0", 8888); // 바인딩용 (서버)
```

또는

```java
InetSocketAddress remote =
    new InetSocketAddress("example.com", 80); // 접속용 (클라이언트)
```

### Iterative Programming using UDP (Java)

#### DatagramSocket / DatagramPacket 기본 구조

- `DatagramSocket`:
  - UDP 소켓을 나타낸다.
  - 포트에 바인드하거나, 임의 포트로 열 수도 있다.
- `DatagramPacket`:
  - 하나의 UDP 패킷(데이터그램)을 나타내는 객체.
  - 수신용: 버퍼 + 길이, 송신용: 버퍼 + 길이 + 대상 주소/포트.

Java UDP iterative 서버 흐름:

1. `DatagramSocket socket = new DatagramSocket(port);`
2. 무한 루프:
   - `DatagramPacket` 생성 (수신 버퍼)
   - `socket.receive(packet);` (blocking)
   - 처리
   - 응답을 담은 `DatagramPacket` 생성
   - `socket.send(responsePacket);`

#### 예제: Java UDP 대문자 변환 서버

```java
// UdpUpperServer.java
import java.net.DatagramPacket;
import java.net.DatagramSocket;
import java.net.InetAddress;

public class UdpUpperServer {
    private static final int SERVER_PORT = 9999;
    private static final int BUF_SIZE = 2048;

    public static void main(String[] args) throws Exception {
        DatagramSocket socket = new DatagramSocket(SERVER_PORT);
        System.out.println("[*] UDP Uppercase Server listening on port " + SERVER_PORT);

        byte[] buf = new byte[BUF_SIZE];

        while (true) {
            DatagramPacket packet = new DatagramPacket(buf, buf.length);
            socket.receive(packet); // blocking

            InetAddress clientAddr = packet.getAddress();
            int clientPort = packet.getPort();
            int len = packet.getLength();

            String received = new String(packet.getData(), 0, len, "UTF-8");
            System.out.printf("[+] Received from %s:%d -> '%s'%n",
                              clientAddr.getHostAddress(), clientPort, received);

            // 대문자 변환
            String upper = received.toUpperCase();
            byte[] respData = upper.getBytes("UTF-8");

            DatagramPacket respPacket = new DatagramPacket(
                    respData, respData.length,
                    clientAddr, clientPort);

            socket.send(respPacket);
        }
    }
}
```

클라이언트:

```java
// UdpUpperClient.java
import java.net.DatagramPacket;
import java.net.DatagramSocket;
import java.net.InetAddress;
import java.util.Scanner;

public class UdpUpperClient {
    private static final int BUF_SIZE = 2048;

    public static void main(String[] args) throws Exception {
        if (args.length != 2) {
            System.err.println("Usage: java UdpUpperClient <server_ip> <port>");
            System.exit(1);
        }

        String serverIp = args[0];
        int port = Integer.parseInt(args[1]);

        DatagramSocket socket = new DatagramSocket(); // 임의 포트
        InetAddress serverAddr = InetAddress.getByName(serverIp);

        Scanner sc = new Scanner(System.in, "UTF-8");
        byte[] recvBuf = new byte[BUF_SIZE];

        while (true) {
            System.out.print("> ");
            if (!sc.hasNextLine()) break;
            String line = sc.nextLine();

            byte[] data = line.getBytes("UTF-8");
            DatagramPacket packet = new DatagramPacket(
                    data, data.length, serverAddr, port);
            socket.send(packet);

            DatagramPacket recvPacket = new DatagramPacket(recvBuf, recvBuf.length);
            socket.receive(recvPacket);

            String resp = new String(recvPacket.getData(), 0, recvPacket.getLength(), "UTF-8");
            System.out.println("RECV: '" + resp + "'");
        }

        socket.close();
    }
}
```

이 예제 역시 **iterative** 다:

- 서버는 한 번에 하나의 패킷을 처리하고,
  그 처리(대문자 변환 + 응답)가 끝나야 다음 패킷을 받는다.

---

### Iterative Programming using TCP (Java)

#### ServerSocket / Socket 구조

Java에서 TCP 서버/클라이언트를 구현할 때 핵심 클래스:

- `ServerSocket`:
  - 특정 포트에서 **연결 요청을 대기(listen)** 한다.
  - `accept()` 메서드로 연결 수락 (blocking)
- `Socket`:
  - 하나의 TCP 연결을 나타낸다.
  - `getInputStream()` / `getOutputStream()` 으로 스트림 I/O 사용.

Iterative 서버 구조:

1. `ServerSocket server = new ServerSocket(port);`
2. 반복:
   - `Socket client = server.accept();`
   - 이 소켓에 대해 `InputStream` / `OutputStream` 으로 통신
   - 끝나면 `client.close();`
   - 다시 `accept()` 로 돌아감

#### 예제: Java iterative TCP 에코 서버

```java
// TcpEchoServerIterative.java
import java.io.BufferedReader;
import java.io.InputStreamReader;
import java.io.PrintWriter;
import java.net.ServerSocket;
import java.net.Socket;

public class TcpEchoServerIterative {
    private static final int SERVER_PORT = 8888;

    public static void main(String[] args) throws Exception {
        ServerSocket serverSocket = new ServerSocket(SERVER_PORT);
        System.out.println("[*] Iterative TCP Echo Server listening on port " + SERVER_PORT);

        while (true) {
            Socket clientSocket = serverSocket.accept();
            String clientInfo = clientSocket.getRemoteSocketAddress().toString();
            System.out.println("[+] New connection from " + clientInfo);

            try (
                BufferedReader in = new BufferedReader(
                        new InputStreamReader(clientSocket.getInputStream(), "UTF-8"));
                PrintWriter out = new PrintWriter(
                        clientSocket.getOutputStream(), true); // auto-flush
            ) {
                String line;
                while ((line = in.readLine()) != null) {
                    System.out.println("[" + clientInfo + "] " + line);
                    out.println(line); // echo
                }
            } catch (Exception e) {
                e.printStackTrace();
            } finally {
                System.out.println("[-] Connection closed: " + clientInfo);
                clientSocket.close();
            }
        }
    }
}
```

클라이언트:

```java
// TcpEchoClient.java
import java.io.BufferedReader;
import java.io.InputStreamReader;
import java.io.PrintWriter;
import java.net.Socket;

public class TcpEchoClient {
    public static void main(String[] args) throws Exception {
        if (args.length != 2) {
            System.err.println("Usage: java TcpEchoClient <server_ip> <port>");
            System.exit(1);
        }

        String serverIp = args[0];
        int port = Integer.parseInt(args[1]);

        try (
            Socket socket = new Socket(serverIp, port);
            BufferedReader in = new BufferedReader(
                    new InputStreamReader(socket.getInputStream(), "UTF-8"));
            PrintWriter out = new PrintWriter(
                    socket.getOutputStream(), true);
            BufferedReader stdin = new BufferedReader(
                    new InputStreamReader(System.in, "UTF-8"));
        ) {
            System.out.println("[*] Connected to " + serverIp + ":" + port);

            String line;
            while (true) {
                System.out.print("> ");
                line = stdin.readLine();
                if (line == null || line.isEmpty()) {
                    break;
                }
                out.println(line);        // 서버로 전송
                String resp = in.readLine(); // echo 수신
                if (resp == null) break;
                System.out.println("RECV: " + resp);
            }
        }
    }
}
```

여기서도 구조는 C와 동일한 논리를 따른다:

- 서버:
  - `accept()` → 연결 하나 처리 → 끝나면 또 `accept()`
- 클라이언트:
  - `Socket` 생성 시 `connect()` 수행
  - 줄 단위(`BufferedReader.readLine()`)로 메시지 읽고 쓰기

#### Iterative 구조의 한계와 확장 방향

Java에서 iterative 서버는 학습용으로 매우 좋지만, 실제 서비스에서는 한계가 뚜렷하다.

- 한 클라이언트 연결이 오래 유지되면, 다른 클라이언트는 연결을 못하거나,
  연결은 되더라도 처리 지연이 심해진다.
- 해결책:
  - `accept()` 이후 스레드를 생성해 처리하는 **Thread-per-connection** 패턴
  - Java NIO (`Selector`, non-blocking channel) 기반 이벤트 루프
  - Netty, Undertow 같은 고성능 프레임워크 사용

그러나 이 모든 고급 기법도 결국 **여기서 다룬 iterative 기본 구조**를 확장한 것이다:

- `ServerSocket.accept()` → `Socket` → 스트림 I/O
- 메시지 프레이밍, 에러 처리, 프로토콜 파서 등은 항상 동일한 개념 위에 있다.

---

## 전체 요약

- C와 Java 모두 **소켓 API**를 통해 전송 계층(TCP/UDP)의 서비스를 사용한다.
- **Iterative 서버**는:
  - 한 번에 하나의 요청/연결만 처리하는 가장 단순한 구조다.
  - UDP에서는 `recvfrom()/sendto()` 루프,
  - TCP에서는 `accept()` 후 `recv()/send()` 혹은 Java의 `ServerSocket/Socket` 구조로 구현된다.
- C에서는:
  - 주소 구조(`sockaddr_in`, `getaddrinfo`)와 바이트 오더, 오류 처리, 메시지 경계 관리에 특히 신경 써야 한다.
- Java에서는:
  - `InetAddress`, `InetSocketAddress`, `DatagramSocket`, `DatagramPacket`, `ServerSocket`, `Socket` 등의 클래스를 통해 같은 개념을 더 높은 수준의 API로 감싼다.
- Iterative 구조는 **학습용/테스트용**으로 이상적이지만,
  - 실제 서비스에서는 동시성 한계를 극복하기 위해
    프로세스/스레드/이벤트 기반 concurrent 서버로 확장하는 것이 일반적이다.

이 글을 기반으로 다음 단계에서는 **Java NIO 기반 비동기 서버**,
또는 **C에서 `select`/`epoll`을 사용한 I/O 다중화 서버**를 설계해 나가면
애플리케이션 계층까지 포함한 전체 네트워크 스택을 실전 수준으로 다룰 수 있게 된다.
