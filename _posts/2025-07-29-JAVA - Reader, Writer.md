---
layout: post
title: Java - Reader, Writer
date: 2025-07-29 17:20:23 +0900
category: Java
---
# Reader, Writer 및 파일 읽기/쓰기

문자 기반 입출력(`Reader`/`Writer`)은 **텍스트(문자)**를 처리할 때 사용하는 추상화입니다. 바이트 스트림(`InputStream`/`OutputStream`)이 **이진 데이터**(이미지, 동영상 등)에 적합하다면, `Reader`/`Writer`는 **문자 인코딩을 고려한 텍스트 처리**에 적합합니다. 아래에서 개념, 주요 클래스, 메서드, 실전 예제와 주의사항까지 자세히 다룹니다.

---

## 1. 핵심 개념 요약

- **Reader / Writer**: 문자(char) 단위의 추상 클래스. 문자 인코딩(UTF-8 등)을 고려해 텍스트를 다룸.  
- **InputStreamReader / OutputStreamWriter**: 바이트 스트림 ↔ 문자 스트림을 연결(인코딩 지정 가능).  
- **BufferedReader / BufferedWriter**: 내부 버퍼를 제공해 성능 향상(라인 단위 읽기/쓰기 지원).  
- **FileReader / FileWriter**: 파일에 바로 문자 단위로 읽고 쓰는 간단한 구현(단, 플랫폼 기본 인코딩 사용 — 주의).  
- **java.nio.file.Files / Path**: NIO 유틸리티로 편리한 읽기/쓰기 API 제공 (`Files.readString`, `Files.lines`, `Files.newBufferedReader` 등).  

---

## 2. Reader 주요 메서드

| 메서드 | 설명 |
|--------|------|
| `int read()` | 하나의 문자(유니코드 코드 포인트의 상위/하위)를 읽어 `int`로 반환. 읽을 데이터 없으면 -1 |
| `int read(char[] cbuf)` | `cbuf` 길이만큼 읽어 `char[]`에 채움, 읽은 문자 수 반환 |
| `int read(char[] cbuf, int off, int len)` | 일부 읽기 |
| `boolean ready()` | 다음 읽기 동작이 블록되지 않고 가능한지 |
| `void close()` | 스트림 닫기(자원 해제) |
| `long skip(long n)` | n 글자만큼 건너뜀 |
| `mark(int readAheadLimit)` / `reset()` | `markSupported()`가 true이면 사용 가능 (예: `BufferedReader`) |

---

## 3. Writer 주요 메서드

| 메서드 | 설명 |
|--------|------|
| `void write(int c)` | 하나의 문자 쓰기 |
| `void write(char[] cbuf)` | 문자 배열 쓰기 |
| `void write(String str)` | 문자열 쓰기 |
| `void write(String str, int off, int len)` | 문자열의 일부 쓰기 |
| `void flush()` | 버퍼 강제 비우기 |
| `void close()` | 스트림 닫기(자동 `flush()` 수행) |

---

## 4. 대표 클래스와 용도

- `FileReader`, `FileWriter`  
  - 간단하지만 **시스템 기본 인코딩**을 사용하므로 인코딩 지정이 필요하면 `InputStreamReader/OutputStreamWriter`를 권장.

- `InputStreamReader`, `OutputStreamWriter`  
  - `new InputStreamReader(new FileInputStream(path), StandardCharsets.UTF_8)` 처럼 **명시적 Charset** 지정 가능.

- `BufferedReader`, `BufferedWriter`  
  - `readLine()` (BufferedReader)로 한 줄씩 읽기 가능. 내부 버퍼로 성능 개선.

- `PrintWriter`  
  - `printf`, `println` 등 편리한 포맷 출력. (버퍼링 가능)

- `Files` (NIO, Java 7+)  
  - `Files.newBufferedReader(Path, Charset)`, `Files.newBufferedWriter(...)`, `Files.readString`, `Files.lines` 등 편리한 유틸 제공.

---

## 5. 기본 예제들

### 5.1 BufferedReader로 파일 한 줄씩 읽기 (UTF-8 권장)
```java
import java.io.*;
import java.nio.charset.StandardCharsets;
import java.nio.file.*;

public class ReadLineExample {
    public static void main(String[] args) {
        Path path = Paths.get("input.txt");

        try (BufferedReader br = Files.newBufferedReader(path, StandardCharsets.UTF_8)) {
            String line;
            while ((line = br.readLine()) != null) {
                System.out.println(line);
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

### 5.2 BufferedWriter로 라인 쓰기(덧붙이기/추가)
```java
import java.io.*;
import java.nio.charset.StandardCharsets;
import java.nio.file.*;
import java.nio.file.StandardOpenOption;

public class WriteLineExample {
    public static void main(String[] args) {
        Path path = Paths.get("output.txt");
        try (BufferedWriter bw = Files.newBufferedWriter(path, StandardCharsets.UTF_8,
                                                       StandardOpenOption.CREATE,
                                                       StandardOpenOption.APPEND)) {
            bw.write("첫 번째 라인");
            bw.newLine();
            bw.write("두 번째 라인");
            bw.newLine();
            bw.flush();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

### 5.3 Files.readString / Files.writeString (Java 11+)
```java
import java.nio.file.*;
import java.nio.charset.StandardCharsets;
import java.io.IOException;

public class FilesStringExample {
    public static void main(String[] args) throws IOException {
        Path p = Paths.get("small.txt");
        // 파일 전체를 문자열로 읽기 (작은 파일에 적합)
        String content = Files.readString(p, StandardCharsets.UTF_8);
        System.out.println(content);

        // 문자열을 파일로 쓰기(덮어쓰기)
        Files.writeString(p, content + "\n추가 라인", StandardCharsets.UTF_8);
    }
}
```

### 5.4 Files.lines를 스트림으로 처리 (스트림 닫기 주의)
```java
import java.nio.file.*;
import java.util.stream.*;
import java.io.IOException;

public class StreamLinesExample {
    public static void main(String[] args) throws IOException {
        Path p = Paths.get("input.txt");
        // try-with-resources로 스트림을 닫아야 함
        try (Stream<String> lines = Files.lines(p, StandardCharsets.UTF_8)) {
            lines.filter(l -> l.contains("ERROR"))
                 .forEach(System.out::println);
        }
    }
}
```

### 5.5 FileReader / FileWriter (인코딩 주의)
```java
import java.io.*;

public class FileReaderWriterExample {
    public static void main(String[] args) {
        try (FileReader fr = new FileReader("input.txt");
             FileWriter fw = new FileWriter("output.txt")) {
             
            int c;
            while ((c = fr.read()) != -1) {
                fw.write(c);
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```
> **주의**: `FileReader`/`FileWriter`는 플랫폼 기본 문자셋을 사용하므로, 인코딩 문제가 발생할 수 있음. 가능한 `Files.newBufferedReader(..., Charset)` 또는 `InputStreamReader`/`OutputStreamWriter`로 `Charset`을 명시하라.

---

## 6. 문자 인코딩(Encoding) 주의사항

- 텍스트 파일을 읽거나 쓸 때는 **명시적으로 Charset을 지정**하자 (예: `StandardCharsets.UTF_8`).  
- 인코딩을 지정하지 않으면 **플랫폼 기본 인코딩**(OS마다 다름)이 사용되어 다른 시스템에서 파일을 읽을 때 깨질 수 있음.

예:
```java
try (BufferedReader br = new BufferedReader(
        new InputStreamReader(new FileInputStream("file.txt"), StandardCharsets.UTF_8))) {
    ...
}
```

---

## 7. 성능 팁 및 메모리 주의

- 대용량 파일은 `Files.readAllLines()`나 `Files.readString()`으로 한꺼번에 읽지 말 것(메모리 부족 위험). 대신 `BufferedReader`/`Files.lines()`로 스트리밍 처리.  
- 텍스트를 반복적으로 붙일 때는 `StringBuilder`를 사용하고, `String` 덧셈은 피함(불필요한 객체 생성).  
- 버퍼 크기 조정: 기본 버퍼(예: `BufferedReader` 기본 8KB)가 대부분 충분하지만, 특수 상황에서는 버퍼 크기 조정 고려.  
- 파일 복사(바이너리)는 `Files.copy(source, target, StandardCopyOption.REPLACE_EXISTING)` 사용 권장.

---

## 8. 문자 스트림에서 mark/reset 사용 (BufferedReader 예)
`BufferedReader`는 `mark`와 `reset`을 지원하므로 일부 읽고 복귀하는 동작에 유용함:
```java
try (BufferedReader br = Files.newBufferedReader(Paths.get("input.txt"), StandardCharsets.UTF_8)) {
    if (br.markSupported()) {
        br.mark(1024); // 앞으로 1024 문자까지 reset 가능
        String first = br.readLine();
        System.out.println("첫 줄: " + first);
        br.reset(); // 다시 첫 줄로 복귀 (readAheadLimit 범위 내)
    }
}
```

---

## 9. 텍스트 vs 바이너리 판단 기준

- **텍스트(문자)**: 사람이 읽는 데이터(로그, CSV, JSON, XML) → `Reader`/`Writer` 또는 `Files` 문자 API 사용.  
- **바이너리(바이트)**: 이미지, 영상, 압축 파일 등 → `InputStream`/`OutputStream` 또는 `Files.copy` 사용.  
- **절대적으로 텍스트라면** 문자 스트림을 사용하여 인코딩을 관리하라. 텍스트를 바이트 스트림으로 읽고 문자열로 변환하면 인코딩을 반드시 고려해야 함.

---

## 10. 예외 상황 및 권장 패턴

- **try-with-resources**로 자동 close 권장.  
- 스트림을 닫지 않으면 파일 디스크립터 누수 발생.  
- 가능한 `Files` 유틸(간결) 사용: `Files.newBufferedReader`, `Files.newBufferedWriter`, `Files.lines`, `Files.copy`, `Files.readString`, `Files.writeString` 등.  
- API 문서(메서드)에 **인코딩 요구사항**을 명시하라(특히 라이브러리/공용 코드 작성 시).

---

## 11. 고급: 파일 열기 옵션 (쓰기 시)
`Files.newBufferedWriter`에서 `StandardOpenOption`으로 동작을 제어:
```java
try (BufferedWriter bw = Files.newBufferedWriter(path, StandardCharsets.UTF_8,
        StandardOpenOption.CREATE,         // 없으면 생성
        StandardOpenOption.TRUNCATE_EXISTING // 기존 파일 덮어쓰기
        // StandardOpenOption.APPEND -> 이어쓰기
)) {
    bw.write("내용");
}
```

---

## 12. 요약 체크리스트

- 텍스트 파일을 다룰 땐 **문자 스트림(Reader/Writer)** 사용.  
- **항상 Charset 명시**(UTF-8 권장).  
- 큰 파일은 스트리밍(라인 단위/Stream API)으로 처리.  
- 성능 위해 `BufferedReader`/`BufferedWriter` 사용.  
- 리소스 관리는 **try-with-resources**로.  
- 간단한 작업엔 `Files` 유틸리티를 활용하면 코드 간결성과 안전성 증가.
