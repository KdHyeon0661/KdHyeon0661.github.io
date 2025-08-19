---
layout: post
title: Java - Stream의 개념, InputStream, OutputStream
date: 2025-07-29 16:20:23 +0900
category: Java
---
# 스트림(Stream)의 개념과 InputStream, OutputStream

자바(Java)에서 **스트림(Stream)**은 데이터를 입출력(I/O)하기 위한 추상화된 통로(파이프라인)입니다. 파일, 네트워크, 메모리, 키보드 입력 등 다양한 데이터 소스로부터 데이터를 읽거나, 데이터를 출력 장치로 보낼 때 스트림을 사용합니다.

---

## 1. 스트림(Stream)의 개념

- **단방향 통신**: 스트림은 한 방향으로만 흐릅니다.  
  - 입력 스트림(InputStream): 데이터 소스 → 프로그램  
  - 출력 스트림(OutputStream): 프로그램 → 데이터 목적지  

- **순차 처리**: 스트림은 보통 데이터를 **순차적으로** 읽거나 씁니다. 즉, 앞에서 읽은 데이터를 건너뛰고 뒤에서 바로 읽을 수 없습니다.

- **추상화 계층**: 스트림은 하드웨어(파일, 네트워크 등)의 복잡한 입출력 과정을 추상화하여 프로그래머가 쉽게 사용하도록 돕습니다.

---

## 2. InputStream (입력 스트림)

### 특징
- 바이트 기반 입력 스트림의 최상위 추상 클래스 (`java.io.InputStream`)
- 1바이트 단위로 데이터를 읽습니다.
- 대표적인 메서드:
  - `int read()` : 1바이트 읽기 (더 이상 읽을 데이터가 없으면 `-1` 반환)
  - `int read(byte[] b)` : 배열 단위로 읽기
  - `void close()` : 스트림 닫기

### 주요 구현 클래스
- `FileInputStream`: 파일에서 바이트 단위로 읽기
- `BufferedInputStream`: 버퍼를 사용하여 성능 향상
- `ByteArrayInputStream`: 메모리(byte 배열)에서 읽기
- `ObjectInputStream`: 객체 단위로 읽기(직렬화된 객체 복원)

### 예제
```java
import java.io.FileInputStream;
import java.io.IOException;

public class InputStreamExample {
    public static void main(String[] args) {
        try (FileInputStream fis = new FileInputStream("example.txt")) {
            int data;
            while ((data = fis.read()) != -1) {
                System.out.print((char) data); // 바이트 → 문자 변환
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

---

## 3. OutputStream (출력 스트림)

### 특징
- 바이트 기반 출력 스트림의 최상위 추상 클래스 (`java.io.OutputStream`)
- 1바이트 단위로 데이터를 출력합니다.
- 대표적인 메서드:
  - `void write(int b)` : 1바이트 쓰기
  - `void write(byte[] b)` : 배열 단위로 쓰기
  - `void flush()` : 버퍼에 남아 있는 데이터를 강제로 출력
  - `void close()` : 스트림 닫기

### 주요 구현 클래스
- `FileOutputStream`: 파일에 바이트 단위로 쓰기
- `BufferedOutputStream`: 버퍼를 사용하여 성능 향상
- `ByteArrayOutputStream`: 메모리에 쓰기
- `ObjectOutputStream`: 객체 단위로 쓰기(직렬화 지원)

### 예제
```java
import java.io.FileOutputStream;
import java.io.IOException;

public class OutputStreamExample {
    public static void main(String[] args) {
        try (FileOutputStream fos = new FileOutputStream("output.txt")) {
            String text = "Hello, Java Stream!";
            fos.write(text.getBytes()); // 문자열을 바이트 배열로 변환 후 쓰기
            fos.flush(); // 데이터 강제 출력
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

---

## 4. InputStream vs OutputStream

| 구분            | InputStream                   | OutputStream                    |
|----------------|--------------------------------|---------------------------------|
| 방향            | 데이터 읽기 (입력)             | 데이터 쓰기 (출력)              |
| 단위            | 바이트 (1byte)                | 바이트 (1byte)                  |
| 주요 클래스      | FileInputStream, BufferedInputStream | FileOutputStream, BufferedOutputStream |
| 활용 예시        | 파일 읽기, 네트워크 데이터 수신 | 파일 쓰기, 네트워크 데이터 송신 |

---

## 5. 정리

- 스트림은 데이터의 **흐름을 추상화한 구조**로, 입력과 출력을 담당합니다.
- **InputStream**은 읽기 전용, **OutputStream**은 쓰기 전용입니다.
- 효율성을 위해 **버퍼 스트림(BufferedInputStream, BufferedOutputStream)**을 주로 함께 사용합니다.
- 객체 입출력, 메모리 기반 입출력, 네트워크 입출력 등 다양한 스트림 구현체들이 존재합니다.


## 6. 스트림(Stream) 사용 시 주의사항

스트림은 자바 입출력의 핵심이지만, 사용 방법을 잘못 이해하거나 관리하지 않으면 **리소스 누수, 데이터 손상, 성능 저하** 같은 문제가 발생할 수 있습니다. 따라서 아래와 같은 주의사항을 숙지하는 것이 중요합니다.

---

### 스트림은 반드시 닫아야 한다
- 모든 스트림은 **리소스를 점유**합니다(파일 핸들, 소켓 연결, 버퍼 등).
- 닫지 않으면 **메모리 누수**나 **파일 잠금(File Lock)** 문제가 발생할 수 있습니다.
- **권장 방법**: `try-with-resources` 구문 사용
  ```java
  try (FileInputStream fis = new FileInputStream("data.txt")) {
      // 파일 읽기
  } catch (IOException e) {
      e.printStackTrace();
  } // 자동으로 close() 호출
  ```

---

### `flush()` 호출 필요
- 출력 스트림(`OutputStream`)은 데이터를 **버퍼(buffer)**에 저장해두었다가 일정 크기에 도달하거나 `flush()`/`close()` 호출 시 실제 출력 장치에 기록합니다.
- `flush()`를 호출하지 않으면 **데이터가 파일이나 네트워크로 완전히 전송되지 않는 문제**가 발생할 수 있습니다.
  ```java
  fos.write("Hello".getBytes());
  fos.flush(); // 반드시 호출
  ```

---

### 바이트 스트림과 문자 스트림 구분
- `InputStream`/`OutputStream`은 **바이트 단위**로 처리 → 바이너리 데이터(이미지, 동영상, 압축 파일)에 적합.
- `Reader`/`Writer`는 **문자 단위**로 처리 → 텍스트 파일, 국제화(한글/유니코드) 대응에 적합.
- 잘못 사용할 경우 한글 깨짐 같은 **인코딩 문제** 발생.

---

### 예외 처리 필수
- 입출력 과정은 **IOException**이 자주 발생할 수 있음 (파일 없음, 네트워크 끊김, 권한 부족 등).
- 예외를 적절히 처리하지 않으면 프로그램이 **비정상 종료**될 수 있음.
- 항상 `try-catch` 블록 또는 `throws` 선언 필요.

---

### 버퍼링 사용 권장
- `FileInputStream`, `FileOutputStream`은 기본적으로 **한 바이트 단위로 처리** → 속도가 느림.
- `BufferedInputStream`, `BufferedOutputStream`을 사용하면 성능 향상.
  ```java
  try (BufferedInputStream bis = new BufferedInputStream(new FileInputStream("data.txt"))) {
      int data;
      while ((data = bis.read()) != -1) {
          // 빠른 읽기 가능
      }
  }
  ```

---

### 멀티스레드 환경 주의
- 스트림 객체는 기본적으로 **스레드 안전(Thread-safe)** 하지 않습니다.
- 여러 스레드가 동시에 같은 스트림에 접근하면 **데이터 손상** 발생 가능.
- 필요하다면 `synchronized` 블록 또는 별도의 동기화 메커니즘 사용.

---

### 플랫폼 종속성 주의
- 파일 경로 구분자는 운영체제마다 다릅니다.  
  - Windows: `C:\\data\\file.txt`  
  - Linux/Mac: `/home/user/data/file.txt`
- **해결 방법**: `File.separator` 또는 `Paths.get()` 사용.

---

## 7. 정리
- 스트림은 **닫기 필수**, `flush()` 잊지 말기.
- **바이트 스트림 vs 문자 스트림**을 구분해 사용해야 함.
- **예외 처리, 버퍼링, 멀티스레드 안전성** 등을 고려해야 함.
- 운영체제 종속성을 줄이기 위해 경로 처리 시 `File`/`Path` API 사용 권장.