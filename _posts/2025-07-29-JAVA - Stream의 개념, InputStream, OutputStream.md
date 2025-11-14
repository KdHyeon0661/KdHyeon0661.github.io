---
layout: post
title: Java - Stream의 개념, InputStream, OutputStream
date: 2025-07-29 16:20:23 +0900
category: Java
---
# 스트림(Stream) 개념과 `InputStream`/`OutputStream`

## 스트림 기본 개념

### 단방향·순차 처리

- **스트림(Stream)**: 데이터가 흐르는 **단방향 통로**.
  - **입력 스트림**: 소스 → 프로그램 (`InputStream`)
  - **출력 스트림**: 프로그램 → 싱크/목적지 (`OutputStream`)
- **순차 접근**이 기본: 임의 접근(시크)은 `RandomAccessFile` 또는 NIO의 `SeekableByteChannel`/`FileChannel` 사용.

### 추상화 계층과 데코레이터(필터) 패턴

- 기반 자원: 파일, 소켓, 메모리, 파이프, 압축 파일 등.
- **데코레이터 체인**으로 기능을 합성:
  ```
  [FileInputStream] -> [BufferedInputStream] -> [DataInputStream]
  [Socket.getOutputStream()] -> [BufferedOutputStream] -> [GZIPOutputStream]
  ```
- 장점: **관심사 분리**(소스/버퍼링/형식/압축/암호화 등), **조립식 확장**.

### 바이트 vs 문자

- 본 문서: **바이트 스트림**(`InputStream`/`OutputStream`).
- 사람이 읽는 텍스트는 **문자 스트림**(`Reader`/`Writer`) + **인코딩 지정** 권장(UTF-8).
  바이트↔문자 다리는 `InputStreamReader`/`OutputStreamWriter`.

---

## `InputStream` — 핵심 API와 올바른 읽기

### 주요 메서드

| 메서드 | 의미/주의 |
|---|---|
| `int read()` | 1바이트 읽기(0~255), EOF 시 **-1**. 느림(시演용) |
| `int read(byte[] b)` | 최대 `b.length` 바이트 읽고 **읽은 바이트 수** 반환(0 가능, EOF 시 -1) |
| `int read(byte[] b, int off, int len)` | 부분 버퍼 채움 |
| `long skip(long n)` | 최대 n바이트 건너뜀(보장 아님) |
| `int available()` | **블로킹 없이 즉시 읽을 수 있는** 바이트 수(총 파일 길이 아님!) |
| `void close()` | 자원 반납 |
| `mark/reset/markSupported` | 일부 구현(예: `BufferedInputStream`)만 지원 |

> **반복문 패턴(정석)**
> `read(...)`는 요청한 만큼 **항상** 채워주지 않습니다. 반환값을 **반드시** 확인하고 누적하세요.

```java
byte[] buf = new byte[8192];
int n;
try (InputStream in = new BufferedInputStream(new FileInputStream("in.bin"))) {
  while ((n = in.read(buf)) != -1) {
    // buf[0..n) 처리
  }
}
```

#### “정확히 N바이트” 읽기

```java
static int readFully(InputStream in, byte[] b, int off, int len) throws IOException {
  int read = 0;
  while (read < len) {
    int n = in.read(b, off + read, len - read);
    if (n == -1) break;
    read += n;
  }
  return read; // len보다 적으면 EOF 도달
}
```

### 구현 클래스

- `FileInputStream`(파일), `ByteArrayInputStream`(메모리), `BufferedInputStream`(버퍼링),
  `DataInputStream`(원시형 읽기), `GZIPInputStream`/`ZipInputStream`(압축),
  `ObjectInputStream`(객체 복원), `DigestInputStream`/`CheckedInputStream`(해시/체크섬),
  `SequenceInputStream`(연결), `PipedInputStream`(스레드 간 파이프), `PushbackInputStream`(되밀기).

---

## `OutputStream` — 핵심 API와 안전한 쓰기

### 주요 메서드

| 메서드 | 의미/주의 |
|---|---|
| `void write(int b)` | 1바이트 쓰기(느림) |
| `void write(byte[] b)` / `write(b,off,len)` | 다건 쓰기 |
| `void flush()` | 내부 버퍼 강제 배출(네트워크/버퍼에 중요) |
| `void close()` | 닫으며 flush |

> **쓰기 루프 패턴**
> 보통 한 번의 `write`로 모두 전송되지만, 상위 레이어에서 부분 쓰기가 발생할 수 있음을 염두에 두고 **API 보장**을 확인하세요(채널 계층에서는 부분 쓰기 흔함).

### 구현 클래스

- `FileOutputStream`, `ByteArrayOutputStream`, `BufferedOutputStream`,
  `DataOutputStream`, `GZIPOutputStream`/`ZipOutputStream`,
  `ObjectOutputStream`, `CipherOutputStream`, `DigestOutputStream`,
  `PipedOutputStream`, `PrintStream`(표준 출력 `System.out` 포함).

---

## 버퍼링과 `flush()` — 성능과 데이터 무결성

- **버퍼 스트림**(`BufferedInputStream`/`BufferedOutputStream`)을 **항상** 고려: I/O 호출 횟수 감소 → 성능 향상.
- **`flush()` 필요 시점**
  - 네트워크/파이프/프로세스 파이프라인 등 **상대가 지금까지의 데이터를 즉시 읽어야 하는 경우**.
  - 장시간 열려 있는 로그 파일에 즉시 기록해야 할 때.
  - 파일은 `close()`에서 자동 flush되지만, **프로그램 비정상 종료** 대비와 **중간 확인**에는 명시 flush 유용.

```java
try (OutputStream out = new BufferedOutputStream(new FileOutputStream("out.bin"))) {
  out.write(payload);
  out.flush(); // 네트워크라면 특히 중요
}
```

---

## 실전: 파일 복사·이동·원자적 갱신

### 간단 파일 복사(스트림)

```java
static void copy(Path src, Path dst) throws IOException {
  try (InputStream in = new BufferedInputStream(Files.newInputStream(src));
       OutputStream out = new BufferedOutputStream(Files.newOutputStream(dst))) {
    byte[] buf = new byte[64 * 1024];
    int n;
    while ((n = in.read(buf)) != -1) out.write(buf, 0, n);
  }
}
```

### NIO 유틸로 복사(권장)

```java
Files.copy(src, dst, StandardCopyOption.REPLACE_EXISTING);
```

### 안전한 덮어쓰기(원자적 교체)

```java
Path temp = Files.createTempFile(dst.getParent(), "tmp-", ".bin");
try (OutputStream out = Files.newOutputStream(temp)) {
  out.write(bytes);
}
Files.move(temp, dst, StandardCopyOption.ATOMIC_MOVE, StandardCopyOption.REPLACE_EXISTING);
```

---

## 데이터 형식: `DataInput/OutputStream`과 바이너리 프로토콜

- **Java 원시형**을 **이식 가능한 바이너리 형식**으로 읽고/쓸 수 있음(빅엔디안).
- 텍스트보다 작고 빠른 **바이너리 프로토콜** 구현에 유용.

```java
// 쓰기
try (DataOutputStream dos = new DataOutputStream(
       new BufferedOutputStream(new FileOutputStream("pkt.bin")))) {
  dos.writeInt(0xCAFE_BABE);  // magic
  dos.writeShort(1);          // version
  dos.writeUTF("hello");      // 길이+UTF 수정형
  dos.flush();
}

// 읽기
try (DataInputStream dis = new DataInputStream(
       new BufferedInputStream(new FileInputStream("pkt.bin")))) {
  int magic = dis.readInt();
  short ver = dis.readShort();
  String msg = dis.readUTF();
}
```

> **엔디안**: `Data*`는 **빅엔디안**. 리틀엔디안 프로토콜이면 `ByteBuffer.order(LITTLE_ENDIAN)` 사용 또는 직접 변환.

---

## 압축/아카이브: GZIP/ZIP 스트림

### GZIP으로 압축/해제(단일 파일)

```java
// 압축
try (InputStream in = Files.newInputStream(Path.of("raw.log"));
     OutputStream out = new GZIPOutputStream(Files.newOutputStream(Path.of("raw.log.gz")))) {
  in.transferTo(out); // Java 9+: 내부 루프 대체
}

// 해제
try (InputStream in = new GZIPInputStream(Files.newInputStream(Path.of("raw.log.gz")));
     OutputStream out = Files.newOutputStream(Path.of("raw.log"))) {
  in.transferTo(out);
}
```

### ZIP(여러 엔트리)

```java
// 생성
try (ZipOutputStream zos = new ZipOutputStream(
        new BufferedOutputStream(Files.newOutputStream(Path.of("docs.zip"))))) {
  for (Path p : List.of(Path.of("a.txt"), Path.of("b.txt"))) {
    zos.putNextEntry(new ZipEntry(p.getFileName().toString()));
    Files.copy(p, zos);
    zos.closeEntry();
  }
}

// 해제
try (ZipInputStream zis = new ZipInputStream(
        new BufferedInputStream(Files.newInputStream(Path.of("docs.zip"))))) {
  for (ZipEntry e; (e = zis.getNextEntry()) != null; ) {
    Path out = Path.of(e.getName());
    Files.copy(zis, out, StandardCopyOption.REPLACE_EXISTING);
    zis.closeEntry();
  }
}
```

---

## 체크섬·해시: `Checked*`/`Digest*` 스트림

### 체크섬(CRC/Adler)

```java
import java.util.zip.*;

try (CheckedInputStream cin = new CheckedInputStream(
        Files.newInputStream(Path.of("file.bin")), new CRC32())) {
  cin.transferTo(OutputStream.nullOutputStream()); // 읽기만
  long crc = cin.getChecksum().getValue();
  System.out.printf("CRC32=0x%08X%n", crc);
}
```

### 메시지 다이제스트(SHA-256)

```java
import java.security.*;
import java.io.*;
import java.nio.file.*;

MessageDigest md = MessageDigest.getInstance("SHA-256");
try (DigestInputStream din = new DigestInputStream(
        Files.newInputStream(Path.of("file.bin")), md)) {
  din.transferTo(OutputStream.nullOutputStream());
}
byte[] hash = md.digest();
```

---

## 네트워크 소켓 I/O — 부분 읽기/타임아웃/버퍼링

```java
try (Socket sock = new Socket("example.com", 80)) {
  sock.setSoTimeout(5_000); // read 타임아웃
  InputStream in = new BufferedInputStream(sock.getInputStream());
  OutputStream out = new BufferedOutputStream(sock.getOutputStream());

  out.write(("GET / HTTP/1.1\r\nHost: example.com\r\n\r\n").getBytes());
  out.flush();

  byte[] buf = new byte[8192];
  int n;
  while ((n = in.read(buf)) != -1) {
    // 수신 처리(헤더 파싱 등)
  }
}
```

- **부분 읽기/부분 쓰기** 가정: 항상 루프로 누적.
- **라인 기반 프로토콜**(HTTP 등)을 바이트로 직접 다루면 복잡 → 보통 `InputStreamReader` + `BufferedReader`로 헤더/텍스트 구간 처리(인코딩 주의).

---

## 파이프/스레드 간 연결: `PipedInput/OutputStream`

```java
PipedInputStream pin = new PipedInputStream();
PipedOutputStream pout = new PipedOutputStream(pin);

// 생산자
new Thread(() -> {
  try (pout) { pout.write("hello".getBytes()); }
  catch (IOException ignored) {}
}).start();

// 소비자
new Thread(() -> {
  try (pin) {
    byte[] b = pin.readAllBytes(); // Java 9+
    System.out.println(new String(b));
  } catch (IOException ignored) {}
}).start();
```

> 파이프는 **스레드 간** 스트리밍에 유용(동일 스레드에서 연결하면 데드락 위험).

---

## 객체 직렬화 스트림 — 주의와 예제

> **보안 주의**: `ObjectInputStream`은 임의 클래스 인스턴스화를 유발 → **신뢰할 수 없는 입력**에 절대 사용 금지. 안전 대안: JSON/CBOR + 명시적 매핑.

```java
// 쓰기
try (ObjectOutputStream oos = new ObjectOutputStream(
        Files.newOutputStream(Path.of("user.ser")))) {
  oos.writeObject(new User("kim", 42));
}

// 읽기
try (ObjectInputStream ois = new ObjectInputStream(
        Files.newInputStream(Path.of("user.ser")))) {
  User u = (User) ois.readObject();
}
```

```java
// 직렬화 가능한 클래스
class User implements Serializable {
  private static final long serialVersionUID = 1L;
  String name; int age;
  User(String n, int a){ this.name=n; this.age=a; }
}
```

---

## NIO와의 연동 — 채널·버퍼·제로카피

### `Files.newInputStream/OutputStream`

- 기존 스트림과 동일하게 쓰되 NIO `Path` 기반으로 간결.

### `FileChannel` + `ByteBuffer`

```java
try (FileChannel ch = FileChannel.open(Path.of("big.bin"),
        StandardOpenOption.READ)) {
  ByteBuffer buf = ByteBuffer.allocateDirect(1024 * 1024); // 오프힙
  while (ch.read(buf) != -1) {
    buf.flip();
    // 처리
    buf.clear();
  }
}
```

### 제로카피 전송 — `transferTo/transferFrom`

```java
try (FileChannel src = FileChannel.open(srcPath, StandardOpenOption.READ);
     FileChannel dst = FileChannel.open(dstPath, StandardOpenOption.WRITE, StandardOpenOption.CREATE, StandardOpenOption.TRUNCATE_EXISTING)) {
  long pos = 0, size = src.size();
  while (pos < size) pos += src.transferTo(pos, size - pos, dst);
}
```

### 메모리 매핑 — 초대형 파일 랜덤 접근

```java
try (FileChannel ch = FileChannel.open(p, StandardOpenOption.READ)) {
  MappedByteBuffer map = ch.map(FileChannel.MapMode.READ_ONLY, 0, ch.size());
  // map.get(), map.position(...) 등으로 빠른 랜덤 읽기
}
```
- 장점: OS 페이지 캐시 활용, 매우 빠른 임의 접근.
- 주의: 매핑 해제 시점은 GC에 의존(최근 JVM은 자동 해제 개선). 사용 후 채널 닫기 필수.

---

## 필터 스트림 빠른 인덱스(무엇을 언제 쓰나)

| 목적 | 입력(읽기) | 출력(쓰기) |
|---|---|---|
| 버퍼링 | `BufferedInputStream` | `BufferedOutputStream` |
| 원시형 | `DataInputStream` | `DataOutputStream` |
| 문자 변환 | `InputStreamReader` | `OutputStreamWriter` |
| 압축 | `GZIPInputStream`, `ZipInputStream` | `GZIPOutputStream`, `ZipOutputStream` |
| 암호화 | `CipherInputStream` | `CipherOutputStream` |
| 체크섬/해시 | `CheckedInputStream`, `DigestInputStream` | `CheckedOutputStream`, `DigestOutputStream` |
| 연결/되밀기 | `SequenceInputStream`, `PushbackInputStream` | — |
| 파이프 | `PipedInputStream` | `PipedOutputStream` |

---

## 클래스패스 리소스 읽기(내장 자원)

```java
try (InputStream in = MyApp.class.getResourceAsStream("/templates/welcome.txt")) {
  if (in == null) throw new FileNotFoundException("resource not found");
  String text = new String(in.readAllBytes()); // JDK 9+
}
```

- JAR 내부 파일도 동일 방식으로 로드. 경로는 **앞에 슬래시**(`/`)로 클래스패스 루트 기준.

---

## 올바른 패턴과 안티패턴

### 반드시 지킬 것

- **`try-with-resources`**로 닫기 보장.
- **버퍼링** 기본 적용.
- **읽기 루프**에서 반환값 체크 및 누적.
- **네트워크**는 `flush()`/타임아웃/부분 읽기 대비.
- **에러 문맥 로깅**(경로, 바이트 오프셋, 엔트리명 등).

### 피해야 할 것

- `available()`를 **파일 크기**로 사용(오개념).
  → 파일 크기는 `Files.size(path)` 또는 채널 `size()` 사용.
- 바이트를 **곧바로 `char` 캐스팅**(텍스트 인코딩 무시).
  → 반드시 `InputStreamReader(UTF-8)` 사용.
- 동일 파일을 여러 스트림으로 **동시 쓰기**(데이터 손상).
- `ObjectInputStream`을 **불신 입력**에 사용(취약).
- 영구 실행 서비스에서 **`close()` 누락**(FD 누수).

---

## 벤치마크 힌트(간단)

- 충분한 크기의 버퍼(예: 64KiB 이상)로 **시스템 호출 수**를 줄이면 대개 유리.
- **`transferTo/transferFrom`**는 커널 제로카피에 의해 매우 빠른 경향.
- GC 부담을 줄이려면 **직접 버퍼**(`allocateDirect`)와 **버퍼 재사용** 검토.
- 정량 검증은 **JMH** 권장(마이크로벤치마크 프레임워크).

---

## 레시피 모음

### 바이너리 헤더 + 가변 페이로드 프레이밍

```java
// [len:int][type:byte][payload:len-1 bytes]
static void writeFrame(OutputStream out, byte type, byte[] payload) throws IOException {
  try (DataOutputStream dos = new DataOutputStream(new BufferedOutputStream(out))) {
    dos.writeInt(1 + payload.length);
    dos.writeByte(type);
    dos.write(payload);
    dos.flush();
  }
}

static Frame readFrame(InputStream in) throws IOException {
  DataInputStream dis = new DataInputStream(new BufferedInputStream(in));
  int len = dis.readInt();
  byte type = dis.readByte();
  byte[] payload = dis.readNBytes(len - 1);
  return new Frame(type, payload);
}
```

### 바이트 단위 탐색(버퍼 슬라이딩)

```java
static long countOccurrences(Path path, byte[] needle) throws IOException {
  try (InputStream in = new BufferedInputStream(Files.newInputStream(path))) {
    byte[] buf = new byte[64 * 1024];
    int n, match = 0; long cnt = 0;
    while ((n = in.read(buf)) != -1) {
      for (int i = 0; i < n; i++) {
        if (buf[i] == needle[match]) {
          if (++match == needle.length) { cnt++; match = 0; }
        } else match = 0;
      }
    }
    return cnt;
  }
}
```

### 파일 구간 추출(오프셋/길이)

```java
static void slice(Path src, Path dst, long offset, long length) throws IOException {
  try (var in = FileChannel.open(src, StandardOpenOption.READ);
       var out = FileChannel.open(dst, StandardOpenOption.WRITE, StandardOpenOption.CREATE, StandardOpenOption.TRUNCATE_EXISTING)) {
    long pos = offset, remain = length;
    while (remain > 0) {
      long x = in.transferTo(pos, remain, out);
      if (x <= 0) break;
      pos += x; remain -= x;
    }
  }
}
```

---

## 요약 표 — 무엇을 선택할까?

| 요구사항 | 권장 조합 |
|---|---|
| 일반 파일 읽기/쓰기 | `Files.newInputStream/OutputStream` + `Buffered*Stream` |
| 빠른 일괄 복사 | `Files.copy` 또는 `FileChannel.transferTo` |
| 네트워크 I/O | `Socket.getInput/OutputStream` + `Buffered*` + 타임아웃 + `flush()` |
| 텍스트 변환 | `InputStreamReader/OutputStreamWriter(UTF-8)` |
| 원시형 바이너리 | `DataInput/OutputStream` |
| 압축 | `GZIP*Stream`, `Zip*Stream` |
| 체크섬/해시 | `Checked*Stream`, `Digest*Stream` |
| 스레드 간 파이프 | `PipedInput/OutputStream` |
| 초대용량 랜덤 접근 | `FileChannel.map(MappedByteBuffer)` |

---

## 결론

- `InputStream`/`OutputStream`은 **바이트 I/O의 표준 축**입니다.
- **버퍼링 + 올바른 읽기/쓰기 루프 + `try-with-resources` + 상황별 필터 스트림**을 조합하면, 파일/네트워크/압축/체크섬/직렬화 등 대부분의 I/O 니즈를 **안전하고 빠르게** 충족할 수 있습니다.
- 임의 접근·고성능이 필요하면 **NIO 채널/버퍼/제로카피**를 검토하세요.
- 텍스트라면 **반드시 문자 스트림과 인코딩**을 사용해 데이터 무결성을 지키십시오.

```java
// 핵심 패턴: 안전하고 빠른 바이트 복사 (JDK 9+)
try (InputStream in = new BufferedInputStream(Files.newInputStream(src));
     OutputStream out = new BufferedOutputStream(Files.newOutputStream(dst))) {
  in.transferTo(out);    // 내부가 최적화된 루프 수행
  out.flush();           // 네트워크/파이프라면 명시 flush 권장
}
```
