---
layout: post
title: Java - Reader, Writer
date: 2025-07-29 17:20:23 +0900
category: Java
---
# Reader/Writer & 파일 읽기/쓰기

## 개념 정리 — 언제 Reader/Writer인가?

| 구분 | 바이트 스트림 | 문자 스트림 |
|---|---|---|
| 추상 타입 | `InputStream` / `OutputStream` | `Reader` / `Writer` |
| 대상 | 이진 데이터(이미지, 동영상, 압축 등) | 텍스트(사람이 읽는 문자열) |
| 인코딩 고려 | 직접 바이트↔문자 변환 필요 | **내장된 문자 변환**(인코딩 지정 가능) |
| 대표 클래스 | `FileInputStream`, `FileOutputStream` | `FileReader`, `FileWriter`, `BufferedReader`, `BufferedWriter`, `InputStreamReader`, `OutputStreamWriter` |
| 권장 방식 | NIO: `Files.copy`, `FileChannel` | NIO: `Files.newBufferedReader/Writer`, `Files.readString/writeString` |

> **핵심 규칙**: **텍스트**면 Reader/Writer(인코딩 명시), **바이너리**면 Stream/Channel.

---

## Reader / Writer 주요 API

### Reader 핵심 메서드

| 메서드 | 설명 |
|---|---|
| `int read()` | 한 글자를 `int`로(EOF=-1). `char`가 아닌 **코드 유닛** 기반에 유의 |
| `int read(char[] cbuf)` / `read(cbuf,off,len)` | 버퍼에 다건 읽기 |
| `boolean ready()` | 블록 없이 읽기 가능 여부(보장 X) |
| `long skip(long n)` | 문자 단위 스킵 |
| `void mark(int readAheadLimit)` / `reset()` | 일부 구현만 지원(`BufferedReader` 등) |
| `void close()` | 자원 해제 |

### Writer 핵심 메서드

| 메서드 | 설명 |
|---|---|
| `void write(int c)` | 문자 1개 쓰기 |
| `void write(char[] cbuf)` / `write(String s)` | 다건 쓰기 |
| `void write(String s, int off, int len)` | 부분 쓰기 |
| `void flush()` | 내부 버퍼 강제 배출 |
| `void close()` | 닫으며 flush |

> **라인 구분**: `BufferedWriter.newLine()`, 또는 `System.lineSeparator()` 사용. 직접 `\n` 하드코딩은 플랫폼 간 차이 초래.

---

## 필수 클래스와 올바른 조합

| 클래스 | 용도/특징 | 인코딩 |
|---|---|---|
| `FileReader`/`FileWriter` | 간단 파일 문자 I/O | **플랫폼 기본 문자셋 사용**(권장 X) |
| `InputStreamReader` / `OutputStreamWriter` | 바이트↔문자 다리(브리지) | **명시적 `Charset`** 지정 가능 |
| `BufferedReader` / `BufferedWriter` | 버퍼링 + `readLine()` / `newLine()` | 상위에 `InputStreamReader`/`Files.*Reader` 얹기 |
| `PrintWriter` | `println`/`printf`/오토플러시 | 포매팅/로깅에 편리 |
| NIO `Files` | `newBufferedReader/Writer`, `readString`, `lines`, `writeString` | **가장 안전·간결** |

> **권장 조합**: `Files.newBufferedReader(Path, UTF_8)` / `Files.newBufferedWriter(Path, UTF_8, 옵션...)`

---

## 인코딩(Encoding) — 문제의 8할은 여기서

- 항상 **`StandardCharsets.UTF_8`**을 **명시**하세요.
- `FileReader`/`FileWriter`는 **OS 기본 문자셋**을 사용 → 이식성/재현성 낮음.
- BOM(UTF-8 BOM)은 표준상 불필요하며, 존재 시 파일 첫 글자로 `\uFEFF`가 들어올 수 있음(직접 제거 필요).

### 안전한 Reader 생성(UTF-8)

```java
import java.io.*;
import java.nio.charset.StandardCharsets;

Reader r = new BufferedReader(
              new InputStreamReader(
                new FileInputStream("data.txt"), StandardCharsets.UTF_8));
```

### NIO 스타일(간결 권장)

```java
import java.nio.file.*;
import java.nio.charset.StandardCharsets;

BufferedReader br = Files.newBufferedReader(Path.of("data.txt"), StandardCharsets.UTF_8);
BufferedWriter bw = Files.newBufferedWriter(Path.of("out.txt"), StandardCharsets.UTF_8);
```

---

## 기본 예제 — 실전 패턴

### 라인 단위 읽기(UTF-8, try-with-resources)

```java
import java.nio.file.*;
import java.nio.charset.StandardCharsets;
import java.io.*;

public class ReadLines {
  public static void main(String[] args) {
    Path path = Path.of("input.txt");
    try (BufferedReader br = Files.newBufferedReader(path, StandardCharsets.UTF_8)) {
      String line;
      while ((line = br.readLine()) != null) {
        // 처리
        System.out.println(line);
      }
    } catch (IOException e) {
      // 로깅/대응
      e.printStackTrace();
    }
  }
}
```

### 라인 쓰기(추가/덧붙이기)

```java
import java.nio.file.*;
import java.nio.charset.StandardCharsets;
import java.io.*;
import static java.nio.file.StandardOpenOption.*;

public class AppendLines {
  public static void main(String[] args) {
    Path p = Path.of("output.txt");
    try (BufferedWriter bw = Files.newBufferedWriter(p, StandardCharsets.UTF_8, CREATE, APPEND)) {
      bw.write("첫 번째 라인"); bw.newLine();
      bw.write("두 번째 라인"); bw.newLine();
    } catch (IOException e) {
      e.printStackTrace();
    }
  }
}
```

### 전체 문자열 읽기/쓰기 (작은 파일, Java 11+)

```java
import java.nio.file.*;
import java.nio.charset.StandardCharsets;
import java.io.IOException;

public class ReadWriteString {
  public static void main(String[] args) throws IOException {
    Path p = Path.of("small.txt");
    String content = Files.readString(p, StandardCharsets.UTF_8);
    Files.writeString(p, content + System.lineSeparator() + "추가", StandardCharsets.UTF_8);
  }
}
```

### 스트림으로 필터링 처리

```java
import java.nio.file.*;
import java.nio.charset.StandardCharsets;
import java.util.stream.Stream;
import java.io.IOException;

public class FilterErrors {
  public static void main(String[] args) throws IOException {
    Path p = Path.of("app.log");
    try (Stream<String> lines = Files.lines(p, StandardCharsets.UTF_8)) {
      lines.filter(s -> s.contains("ERROR")).forEach(System.out::println);
    }
  }
}
```

### mark/reset (lookahead)

```java
import java.nio.file.*;
import java.nio.charset.StandardCharsets;
import java.io.*;

public class MarkReset {
  public static void main(String[] args) throws IOException {
    try (BufferedReader br = Files.newBufferedReader(Path.of("input.txt"), StandardCharsets.UTF_8)) {
      if (br.markSupported()) {
        br.mark(1024);
        String first = br.readLine();
        System.out.println("HEAD: " + first);
        br.reset(); // 다시 첫 줄로
        System.out.println("AGAIN: " + br.readLine());
      }
    }
  }
}
```

---

## 실무 자주 쓰는 고급 기법

### PushbackReader — 토큰 되밀어 넣기

```java
import java.io.*;
import java.nio.charset.StandardCharsets;

try (PushbackReader pr = new PushbackReader(
        new InputStreamReader(new FileInputStream("data.txt"), StandardCharsets.UTF_8), 8)) {
  int ch;
  while ((ch = pr.read()) != -1) {
    if (ch == '#') { // 주석 시작으로 판단 → 라인 전체 스킵 시 롤백/재해석 가능
      // 필요시 pushBack
      // pr.unread(ch);
    }
  }
}
```

### LineNumberReader — 라인 번호 관리

```java
import java.io.*;
import java.nio.charset.StandardCharsets;

try (LineNumberReader lr = new LineNumberReader(
        new InputStreamReader(new FileInputStream("src.txt"), StandardCharsets.UTF_8))) {
  String line;
  while ((line = lr.readLine()) != null) {
    System.out.printf("%6d | %s%n", lr.getLineNumber(), line);
  }
}
```

### PrintWriter — 포매팅 출력

```java
import java.io.*;
import java.nio.charset.StandardCharsets;
import java.util.Locale;

try (PrintWriter pw = new PrintWriter(
        new OutputStreamWriter(new FileOutputStream("report.txt"), StandardCharsets.UTF_8), true)) { // autoFlush
  pw.printf(Locale.US, "Value: %.2f%n", 12.3456);
  pw.println("완료");
}
```

---

## 인코딩·BOM·에러 정책

### BOM 제거(UTF-8)

Java의 `InputStreamReader`는 BOM을 자동 제거하지 않을 수 있습니다. 간단 제거 예:

```java
import java.io.*;
import java.nio.charset.StandardCharsets;

static Reader utf8ReaderStrippingBOM(InputStream in) throws IOException {
  PushbackInputStream pis = new PushbackInputStream(in, 3);
  byte[] bom = new byte[3];
  int n = pis.read(bom, 0, 3);
  if (n == 3 && (bom[0] & 0xFF) == 0xEF && (bom[1] & 0xFF) == 0xBB && (bom[2] & 0xFF) == 0xBF) {
    // BOM consumed
  } else {
    if (n > 0) pis.unread(bom, 0, n);
  }
  return new InputStreamReader(pis, StandardCharsets.UTF_8);
}
```

### 손상된 바이트 처리(Decoder 에러 정책)

`CharsetDecoder`로 변환 에러 정책 제어(보고/무시/치환):

```java
import java.nio.*;
import java.nio.charset.*;
import java.nio.file.*;

ByteBuffer bytes = ByteBuffer.wrap(Files.readAllBytes(Path.of("text.dat")));
CharsetDecoder dec = StandardCharsets.UTF_8
    .newDecoder()
    .onMalformedInput(CodingErrorAction.REPORT)   // REPORT / REPLACE / IGNORE
    .onUnmappableCharacter(CodingErrorAction.REPORT);
CharBuffer chars = dec.decode(bytes);
String s = chars.toString();
```

---

## 성능·메모리·대용량 파일 전략

- **버퍼링 필수**: `BufferedReader/Writer` 기본 버퍼(8KB)로 충분한 경우가 많지만, 대용량이면 버퍼를 키워볼 수 있음.
- **스트리밍 처리**: `Files.lines()` 또는 `readLine()`로 **순차 처리**. `readString()/readAllLines()`는 작은 파일에만.
- **문자/바이트 길이 차이**: `char`는 UTF-16 코드 유닛. **문자 수 ≠ 바이트 수**. 코드포인트(이모지 등)는 2개 `char`를 쓸 수 있음.
- **라인 끝(EOL) 통일**: OS 혼재(\r\n vs \n)를 다룰 땐 `readLine()` + `newLine()`으로 **정규화**.
- **동시 쓰기 금지**: 복수 스레드가 **같은 파일**에 동시에 Writer를 열지 않기(파편화/경합). 필요 시 **파일 잠금(FileChannel.lock)**.
- **안전한 덮어쓰기**: 임시 파일에 기록 → `Files.move(temp, target, ATOMIC_MOVE, REPLACE_EXISTING)`로 **원자적 교체**.

### 빠른 텍스트 복사(스트리밍)

```java
static void copyText(Path src, Path dst) throws IOException {
  try (BufferedReader br = Files.newBufferedReader(src, StandardCharsets.UTF_8);
       BufferedWriter bw = Files.newBufferedWriter(dst, StandardCharsets.UTF_8)) {
    char[] buf = new char[64 * 1024];
    int n;
    while ((n = br.read(buf)) != -1) {
      bw.write(buf, 0, n);
    }
  }
}
```

### 마지막 N라인 tail 구현(메모리 제한)

```java
import java.util.ArrayDeque;

static void tail(Path p, int n) throws IOException {
  try (BufferedReader br = Files.newBufferedReader(p, StandardCharsets.UTF_8)) {
    ArrayDeque<String> dq = new ArrayDeque<>(n);
    String line;
    while ((line = br.readLine()) != null) {
      if (dq.size() == n) dq.removeFirst();
      dq.addLast(line);
    }
    dq.forEach(System.out::println);
  }
}
```

---

## 스캐너 vs 버퍼드리더

| 항목 | `Scanner` | `BufferedReader` |
|---|---|---|
| 장점 | 토큰화/숫자 파싱 간단 | 빠름, 제로-오버헤드 라인 처리 |
| 인코딩 | 생성자에 `Charset` 가능 | 가능 |
| 용도 | 간단 입력/파싱 | 성능/대용량 라인 처리 |

> 성능 요구가 높으면 `BufferedReader` + 직접 파싱이 일반적으로 더 빠릅니다.

---

## 텍스트 ↔ 바이너리 브리지 패턴

### GZIP 텍스트 읽기

```java
import java.util.zip.GZIPInputStream;
import java.nio.charset.StandardCharsets;
import java.io.*;

try (BufferedReader br = new BufferedReader(
       new InputStreamReader(
         new GZIPInputStream(new FileInputStream("log.txt.gz")),
         StandardCharsets.UTF_8))) {
  for (String line; (line = br.readLine()) != null;) {
    // ...
  }
}
```

### CSV 간단 변환(탭→CSV)

```java
import java.nio.file.*;
import java.nio.charset.StandardCharsets;
import java.io.*;

static void tsvToCsv(Path in, Path out) throws IOException {
  try (BufferedReader br = Files.newBufferedReader(in, StandardCharsets.UTF_8);
       BufferedWriter bw = Files.newBufferedWriter(out, StandardCharsets.UTF_8)) {
    for (String line; (line = br.readLine()) != null;) {
      String csv = String.join(",", line.split("\t", -1));
      bw.write(csv); bw.newLine();
    }
  }
}
```

---

## 안전한 쓰기 — 임시 파일, 원자적 교체

```java
import java.nio.file.*;
import java.nio.charset.StandardCharsets;
import java.io.*;
import static java.nio.file.StandardCopyOption.*;

static void rewriteSafe(Path target, Iterable<String> lines) throws IOException {
  Path temp = Files.createTempFile(target.getParent(), "tmp-", ".txt");
  try (BufferedWriter bw = Files.newBufferedWriter(temp, StandardCharsets.UTF_8)) {
    for (String s : lines) { bw.write(s); bw.newLine(); }
  }
  Files.move(temp, target, ATOMIC_MOVE, REPLACE_EXISTING);
}
```

- 부분 실패로 인해 **깨진 파일**이 남는 일을 방지.

---

## 파일 잠금(File Lock)으로 단독 쓰기 보장

```java
import java.nio.channels.FileChannel;
import java.nio.file.*;
import java.nio.charset.StandardCharsets;
import java.io.*;

try (FileChannel ch = FileChannel.open(Path.of("shared.txt"), StandardOpenOption.WRITE, StandardOpenOption.CREATE);
     FileLock lock = ch.lock(); // 블로킹 잠금
     Writer w = new BufferedWriter(new OutputStreamWriter(Channels.newOutputStream(ch), StandardCharsets.UTF_8))) {
  w.write("exclusive write\n");
}
```

> 잠금은 **프로세스 간 조정**에만 유효. 스레드 간 조정은 자바 동기화 사용.

---

## 코드 포인트(이모지 등) 주의

- `char`는 16비트(UTF-16 코드 유닛). 이모지 등 **보조 평면 문자**는 **2개 `char`**가 필요.
- 코드포인트 안전 순회:
```java
String s = "A💡B";
s.codePoints().forEach(cp -> System.out.println(Integer.toHexString(cp)));
```

---

## 예외 처리·리소스 관리 패턴

- **try-with-resources**로 항상 닫기.
- 예외는 **로깅 + 문맥 정보** 제공(파일 경로, 라인 번호 등).
- Writer는 **반드시 flush/close** 후 읽기 시작(버퍼 잔류 방지).

표준 템플릿:
```java
try (BufferedReader br = Files.newBufferedReader(p, UTF_8)) {
  // ...
} catch (NoSuchFileException e) {
  // 파일 없음 명확 처리
} catch (MalformedInputException e) {
  // 인코딩 오류
} catch (IOException e) {
  // 기타 I/O
}
```

---

## 체크리스트 요약

- [ ] 텍스트 → **Reader/Writer**, 바이너리 → Stream/Channel
- [ ] **UTF-8 명시**(`StandardCharsets.UTF_8`)
- [ ] **Buffered** 사용, 대용량은 스트리밍
- [ ] 라인 끝은 `newLine()` / `System.lineSeparator()`
- [ ] 임시 파일 + `Files.move(..., ATOMIC_MOVE)`로 안전 덮어쓰기
- [ ] 필요 시 파일 잠금(FileLock)
- [ ] BOM/디코딩 오류 대응 정책 정의
- [ ] `try-with-resources`로 누수 방지

---

## 미니 레시피 모음

### 파일 내용 일부 치환(대용량 안전)

```java
static void replaceInFile(Path src, String needle, String repl) throws IOException {
  Path tmp = Files.createTempFile(src.getParent(), "swap-", ".txt");
  try (BufferedReader br = Files.newBufferedReader(src, StandardCharsets.UTF_8);
       BufferedWriter bw = Files.newBufferedWriter(tmp, StandardCharsets.UTF_8)) {
    for (String line; (line = br.readLine()) != null;) {
      bw.write(line.replace(needle, repl));
      bw.newLine();
    }
  }
  Files.move(tmp, src, StandardCopyOption.REPLACE_EXISTING, StandardCopyOption.ATOMIC_MOVE);
}
```

### 라인 수 세기(빠름)

```java
static long countLines(Path p) throws IOException {
  try (var s = Files.lines(p, StandardCharsets.UTF_8)) {
    return s.count();
  }
}
```

### 로그에서 최근 1시간만 추출(간단 필터)

```java
import java.time.*;
import java.time.format.*;

static void filterRecent(Path in, Path out) throws IOException {
  DateTimeFormatter fmt = DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss");
  LocalDateTime threshold = LocalDateTime.now().minusHours(1);
  try (BufferedReader br = Files.newBufferedReader(in, StandardCharsets.UTF_8);
       BufferedWriter bw = Files.newBufferedWriter(out, StandardCharsets.UTF_8)) {
    for (String line; (line = br.readLine()) != null;) {
      // 예: 라인 시작에 타임스탬프 존재 가정
      int end = Math.min(line.length(), 19);
      LocalDateTime ts = LocalDateTime.parse(line.substring(0, end), fmt);
      if (!ts.isBefore(threshold)) { bw.write(line); bw.newLine(); }
    }
  }
}
```

---

## 선택 가이드 표

| 요구 | 권장 API |
|---|---|
| 텍스트 파일 라인 순회 | `Files.newBufferedReader` + `readLine()` |
| 텍스트 파일 전체 문자열(작은 파일) | `Files.readString` / `Files.writeString` |
| 로그 필터링/스트리밍 처리 | `Files.lines(Path, UTF_8)` |
| 빠른 CSV 출력/포매팅 | `PrintWriter` (`autoFlush=true`) |
| 인코딩 변환 | `InputStreamReader`/`OutputStreamWriter` + 명시적 `Charset` |
| 안전한 덮어쓰기 | 임시 파일 → `Files.move(..., ATOMIC_MOVE)` |
| 파일 동시 접근 조정 | `FileChannel.lock()` |
| BOM 있는 파일 | BOM 제거 후 `InputStreamReader(UTF_8)` |

---

## 결론

- **Reader/Writer는 “텍스트 처리의 정석”**입니다. 문제의 대부분은 **인코딩 미명시**에서 시작하므로 **UTF-8을 항상 명시**하세요.
- NIO `Files.*` API를 사용하면 **간결·안전·성능**을 모두 잡을 수 있습니다.
- 대용량/운영 환경에서는 **버퍼링/스트리밍/원자적 교체/파일 잠금** 등 **실전 패턴**을 적용해 **데이터 무결성과 성능**을 동시에 확보하세요.
