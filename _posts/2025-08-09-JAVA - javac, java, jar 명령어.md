---
layout: post
title: Java - javac, java, jar 명령어
date: 2025-08-09 22:20:23 +0900
category: Java
---
# `javac`, `java`, `jar` 명령어

## 개요와 큰 그림

### 전체 흐름(개발자 워크플로우)

```
소스(.java)
   │  └── (선택) 애너테이션 프로세싱
   ▼
javac ──▶ .class ──▶ (선택) jar ──▶ (선택) 모듈 JAR
                                  │
                                  ▼
                           java 런처 실행
```

### 클래식 vs 모듈

| 구분 | 클래스패스(ClassPath) | 모듈패스(ModulePath, JPMS) |
|---|---|---|
| 선언 단위 | JAR/폴더 나열 | **모듈**(`module-info.java`) |
| 캡슐화 | `public`이면 전역 접근 | `exports`로 **선택적 공개**, 비공개는 감춤 |
| 의존 검증 | 런타임에야 실패 | **컴파일 시** 모듈 해석·검증 |
| 리플렉션 | 기본 허용 | 기본 차단 → `opens` 필요 |
| 실행 플래그 | `-cp` | `-p`, `-m` |

---

## `javac` — Java 컴파일러

### 기본형식

```bash
javac [옵션] <소스파일...>
```

### 자주 쓰는 옵션(핵심)

| 옵션 | 의미 / 사용 시점 | 메모 |
|---|---|---|
| `-d <dir>` | 클래스 출력 디렉터리 | 패키지 구조로 폴더 자동 생성 |
| `-cp` / `-classpath` | 클래스패스 설정 | 외부 JAR/클래스 포함 |
| `--module-path`, `-p` | 모듈패스 설정 | 모듈 빌드시 필수 |
| `--module-source-path` | 다중 모듈 소스 트리 | 루트 하나에 여러 모듈 소스 |
| `--release <N>` | **API/바이트코드 동시 타겟팅** | JDK 9+에서 권장(예: `--release 8`) |
| `-source` / `-target` | 소스/바이트코드 버전 | 구버전 호환(단, `--release` 권장) |
| `-encoding <enc>` | 소스 인코딩 지정 | UTF-8 권장 |
| `-Xlint[:all]` | 경고 활성화 | 품질/안정성↑ |
| `-Werror` | 경고를 에러로 | CI에서 유용 |
| `-parameters` | 파라미터 이름 보존 | 리플렉션/프레임워크 친화 |
| `-proc:only` / `-processor` | 애너테이션 프로세싱 | Lombok/MapStruct 등 |
| `--enable-preview` | 프리뷰 기능 컴파일 | 실행 시에도 필요 |
| `@argfile` | 옵션/파일 목록 외부파일 | 긴 명령어 정리 |

### 기본 컴파일 예

```bash
# 현재 폴더에 .class 생성

javac Hello.java
```

패키지 구조 반영:
```bash
# src → bin

javac -d bin src/com/example/app/Main.java
```

외부 라이브러리 포함(클래스패스):
```bash
# Unix/macOS

javac -cp "lib/*:bin" -d bin src/com/example/app/Main.java
# Windows

javac -cp "lib/*;bin" -d bin src\com\example\app\Main.java
```

모듈 빌드:
```bash
# src/<module>/module-info.java, src/<module>/com/...

javac -d out --module-source-path src $(find src -name "*.java")
```

구버전 타겟(권장):
```bash
# JDK 17에서 JDK 8용 산출물 생성

javac --release 8 -d bin $(find src -name "*.java")
```

애너테이션 프로세싱만:
```bash
javac -proc:only -processor com.example.ToolsProcessor -cp "lib/*" $(find src -name "*.java")
```

argfile 사용:
```bash
# args.txt에 각 줄마다 인자/경로 기록

javac @args.txt
```

---

## `java` — 애플리케이션 실행기

### 기본형식

```bash
java [옵션] <메인클래스> [args...]
java [옵션] -jar <실행가능-jar> [args...]
java [옵션] -m <모듈>/<메인클래스> [args...]
```

### 자주 쓰는 옵션

| 옵션 | 의미 |
|---|---|
| `-cp` / `--class-path` | 클래스패스 지정 |
| `-p` / `--module-path` | 모듈패스 지정 |
| `-m` | 모듈/메인클래스 실행 (`com.app/com.app.Main`) |
| `-jar` | 매니페스트 `Main-Class`로 실행 |
| `-Dname=value` | 시스템 프로퍼티 지정 |
| `-Xms`, `-Xmx` | 힙 초기/최대 크기 |
| `-ea` / `-enableassertions` | assert 활성화 |
| `--add-opens` | 모듈 패키지를 리플렉션에 개방 |
| `--add-exports` | 모듈 패키지를 다른 모듈에 export |
| `--enable-preview` | 프리뷰 기능 실행 |
| `@argfile` | 인자 목록 외부파일로 전달 |

### 실행 예

클래스패스 기반:
```bash
# bin + lib/* 포함

java -cp "bin:lib/*" com.example.app.Main arg1 arg2
```

실행가능 JAR:
```bash
java -jar build/app.jar --mode=prod
```

모듈 실행:
```bash
# mods 디렉터리에 모듈 JAR들 배치

java -p mods -m com.example.app/com.example.app.Main
```

단일 파일 소스 실행(Java 11+):
```bash
java Hello.java              # 즉시 컴파일+실행
java --enable-preview Foo.java
```

리플렉션 호환(임시):
```bash
java --add-opens com.example.app/com.example.app.model=ALL-UNNAMED -jar app.jar
```

---

## `jar` — JAR 아카이브 도구 (ZIP 기반)

### 기본형식(긴 옵션 권장)

```bash
jar --create   --file app.jar -C bin .
jar --list     --file app.jar
jar --extract  --file app.jar -C out
jar --validate --file app.jar
```
전통 단축 표기: `jar cfe app.jar com.example.app.Main -C bin .`

### 주요 옵션

| 옵션(긴형) | 단축 | 의미 |
|---|---|---|
| `--create` | `c` | 새 JAR 생성 |
| `--update` | `u` | 기존 JAR 업데이트 |
| `--list` | `t` | 목록 보기 |
| `--extract` | `x` | 추출 |
| `--file <jar>` | `f` | 대상 JAR 파일 |
| `--verbose` | `v` | 상세 로그 |
| `--main-class <FQCN>` | (단축 조합: `cfe`) | 매니페스트 `Main-Class` 설정 |
| `--manifest <mfile>` | `m` | 사용자 지정 매니페스트 병합 |
| `--no-compress` |  | 압축 없이 저장 |
| `--module-version <ver>` |  | 모듈 버전 기입 |
| `--describe-module` |  | JAR의 모듈 정보 설명 |
| `--release <N>` |  | **멀티릴리스 JAR(MRJAR)** 섹션 추가 |

### 실행 가능한 JAR 만들기

```bash
# 컴파일

javac -d bin src/com/example/app/*.java

# JAR 생성(메인클래스 지정)

jar --create --file app.jar --main-class com.example.app.Main -C bin .

# 실행

java -jar app.jar
```

사용자 매니페스트 방식:
```text
# MANIFEST.MF

Manifest-Version: 1.0
Main-Class: com.example.app.Main
Class-Path: lib/guava.jar lib/logging.jar
```
```bash
jar --create --file app.jar --manifest MANIFEST.MF -C bin .
```

> 참고: **일반 `jar`는 여러 JAR을 하나로 합쳐주지 않습니다.**
> 의존 JAR까지 한 덩어리로 만들려면 **Maven Shade** / **Gradle Shadow** 같은 **쉐이더**를 사용하세요.

### 예 (JDK9+)

JDK별 구현을 동봉:
```bash
# 컴파일

javac --release 8 -d out/common $(find src/common -name "*.java")

# JDK 11 전용 구현 컴파일

javac --release 11 -d out/11 $(find src/11 -name "*.java")

# MRJAR 생성

jar --create --file app-mr.jar \
    -C out/common . \
    --release 11 -C out/11 .
```
`--release 11` 섹션의 클래스는 JDK 11 이상에서만 **오버라이드**되어 로딩됩니다.

---

## 클래스패스 vs 모듈패스 (JPMS 상세)

### 차이 요약(재정리)

| 항목 | 클래스패스 | 모듈패스 |
|---|---|---|
| 접근 | `public`=전역 공개 | `exports` 공개만 접근 |
| 의존 검증 | 런타임에 깨짐 | 모듈 해석 단계에서 검증 |
| 리플렉션 | 기본 허용 | **차단** → `opens` 필요 |
| 실행 | `-cp` | `-p` + `-m` |
| 구조 | 자유 배치 | 모듈 단위 + `module-info.java` |

### 자동/이름 없는 모듈

- **자동 모듈**: `module-info.class` 없지만 모듈패스에 올리면 JAR 파일명이 모듈명으로 간주(가능하면 `Automatic-Module-Name` 매니페스트로 **안정적 이름** 지정).
- **이름 없는 모듈**: 클래스패스의 모든 코드가 합쳐진 가상 모듈. 모든 모듈을 읽을 수 있지만, **모듈 → 이름 없는 모듈** 방향은 불가.

---

## 실전 시나리오

### 클래식 프로젝트(클래스패스)

```
project/
 ├─ src/
 │   └─ com/example/app/Main.java
 └─ lib/ (외부 JAR들)
```
컴파일:
```bash
# Unix

javac -cp "lib/*" -d bin $(find src -name "*.java")
# Windows

for /R src %f in (*.java) do @echo %f >> sources.txt
javac -cp "lib/*" -d bin @sources.txt
```
실행가능 JAR + 실행:
```bash
jar --create --file app.jar --main-class com.example.app.Main -C bin .
java -cp "app.jar:lib/*" com.example.app.Main
# 또는

java -jar app.jar
```

### 모듈 프로젝트(간단)

```
src/
 └─ com.example.app/
     ├─ module-info.java
     └─ com/example/app/Main.java
```
`module-info.java`:
```java
module com.example.app {
    requires java.sql;                // 예시
    exports com.example.app.api;      // 외부 공개 패키지
}
```
컴파일/실행:
```bash
javac -d mods/com.example.app --module-source-path src $(find src -name "*.java")
java -p mods -m com.example.app/com.example.app.Main
```

### 멀티 모듈 빌드(서비스 예)

```
src/
 ├─ com.example.core/
 │   ├─ module-info.java
 │   └─ com/example/core/...
 ├─ com.example.plugin.add/
 │   ├─ module-info.java
 │   └─ com/example/plugin/add/...
 └─ com.example.app/
     ├─ module-info.java
     └─ com/example/app/Main.java
```
각 module-info 개요:
```java
// core
module com.example.core { exports com.example.core.api; }

// plugin
module com.example.plugin.add {
    requires com.example.core;
    provides com.example.core.api.Service with com.example.plugin.add.AddService;
}

// app
module com.example.app {
    requires com.example.core;
    uses com.example.core.api.Service;
}
```
컴파일 → JAR → 실행:
```bash
# 컴파일

javac -d out --module-source-path src $(find src -name "*.java")

# 모듈 JAR

jar --create --file mods/com.example.core.jar        -C out/com.example.core .
jar --create --file mods/com.example.plugin.add.jar  -C out/com.example.plugin.add .
jar --create --file mods/com.example.app.jar         -C out/com.example.app .

# 실행

java -p mods -m com.example.app/com.example.app.Main
```

### 애너테이션 프로세싱(예: MapStruct)

컴파일 단계에서 프로세서 실행:
```bash
javac -cp "lib/*" -processor org.mapstruct.ap.MappingProcessor \
     -d bin $(find src -name "*.java")
```
생성 소스 출력 디렉터리 제어(프로세서별 옵션 확인 필요).

### 프리뷰 기능(예: switch expressions – JDK 미리보기일 때)

컴파일:
```bash
javac --enable-preview --release 21 -d bin $(find src -name "*.java")
```
실행:
```bash
java --enable-preview -cp bin com.example.app.Main
```
> 컴파일과 실행 **모두** `--enable-preview`가 필요합니다.

---

## — 원인·해결 표

| 에러/증상 | 전형적 원인 | 해결 가이드 |
|---|---|---|
| `Error: Could not find or load main class ...` | 클래스패스/패키지 불일치, FQCN 오타 | `-cp` 재확인, `com.example.Main` **FQCN** 사용, 디렉터리 구조가 `com/example/Main.class`인지 확인 |
| `NoClassDefFoundError: X` | 런타임 의존 JAR 누락 | 실행 시 `-cp` 또는 매니페스트 `Class-Path`에 누락 JAR 추가 |
| `UnsupportedClassVersionError` | JRE가 더 낮음(예: 1.8 런타임에 17 타겟) | `--release`/`-target` 재설정, 런타임 JDK 정렬 |
| `InaccessibleObjectException` | 모듈 캡슐화로 리플렉션 차단 | 모듈 `opens` 선언 또는 `--add-opens module/package=target` 사용 |
| `FindException: Module not found` | 모듈패스에 JAR 누락 | `-p` 경로/파일명 확인, 자동 모듈 이름 충돌 여부 점검 |
| Split-package 경고/실패 | 동일 패키지를 둘 이상의 모듈에 배치 | 패키지 재배치(패키지 경계=모듈 경계), 다형 구조는 서비스로 분리 |
| 프리뷰 기능 실행 실패 | 컴파일/실행 중 하나만 플래그 사용 | **둘 다** `--enable-preview` 지정 |
| 경고가 무시되어 품질 저하 | 경고 기본 비활성 | `-Xlint:all -Werror -parameters`로 강화 |

---

## 베스트 프랙티스 체크리스트

1. **호환 빌드**: JDK 9+에서는 `--release N`로 **API/바이트코드 동시 보장**.
2. **경고를 친구로**: `-Xlint:all -Werror -parameters`로 품질 유지.
3. **경로 휴대성**: OS 구분자 `:`(Unix)/`;`(Windows) 주의, 와일드카드 `lib/*` 적극 활용.
4. **모듈 캡슐화**: 외부 공개는 `exports` 최소화, 리플렉션은 필요한 패키지만 `opens`.
5. **서비스 구조**: 플러그인/확장은 `uses/provides` + `ServiceLoader`.
6. **패키징 전략**: 실행가능 JAR + 쉐이딩(필요 시) 또는 런처 스크립트. 큰 배포는 **jlink**로 경량 런타임 고려.
7. **CI 표준화**: `@argfile`로 긴 인자 관리, 재현성 있는 빌드(고정 JDK/의존 버전).
8. **문제 발생 시 즉시 기록**: `-verbose:class`(로드 추적), `-Xlog:class+load=info`(JDK 9+) 등으로 원인 좁히기.

---

## 부록

### 프로젝트 레이아웃(권장)

```
root/
 ├─ src/                         # (클래스패스) 공통 소스
 │   └─ com/example/app/...
 ├─ src/                         # (모듈패스) 다중 모듈일 때는:
 │   ├─ com.example.core/
 │   │   ├─ module-info.java
 │   │   └─ com/example/core/...
 │   └─ com.example.app/
 │       ├─ module-info.java
 │       └─ com/example/app/...
 ├─ bin/                         # .class 산출물
 ├─ mods/                        # 모듈 JAR 산출물
 └─ lib/                         # 외부 JAR
```

### OS별 클래스패스 예

| OS | 예 |
|---|---|
| Unix/macOS | `-cp "app.jar:lib/*"` |
| Windows(CMD) | `-cp "app.jar;lib/*"` |
| Windows(PowerShell) | `-cp "app.jar;lib/*"` (따옴표 주의) |

### 치트시트

**컴파일**
```bash
# 클래식

javac -cp "lib/*" -d bin $(find src -name "*.java")
# 모듈

javac -d out --module-source-path src $(find src -name "*.java")
# 구버전 타겟

javac --release 8 -d bin $(find src -name "*.java")
# 프리뷰

javac --enable-preview --release 21 -d bin $(find src -name "*.java")
```

**실행**
```bash
# 클래식

java -cp "bin:lib/*" com.example.app.Main
# 모듈

java -p mods -m com.example.app/com.example.app.Main
# 단일 파일

java Hello.java
# 프리뷰

java --enable-preview -cp bin com.example.app.Main
```

**패키징**
```bash
# 실행가능 JAR

jar --create --file app.jar --main-class com.example.app.Main -C bin .
# 목록/추출/검증

jar --list --file app.jar
jar --extract --file app.jar -C out
jar --validate --file app.jar
# MRJAR

jar --create --file app-mr.jar -C out/common . --release 11 -C out/11 .
```

---

## 부가: 실습 예제 (간단 Echo)

### 소스

```
src/
 └─ com/example/echo/Echo.java
```
```java
package com.example.echo;
public class Echo {
    public static void main(String[] args) {
        for (int i = 0; i < args.length; i++)
            System.out.printf("%d: %s%n", i, args[i]);
    }
}
```

### 빌드·패키징·실행

```bash
# 컴파일

javac -d bin src/com/example/echo/Echo.java
# JAR

jar --create --file echo.jar --main-class com.example.echo.Echo -C bin .
# 실행

java -jar echo.jar hello world
```

---

## 마무리

- `javac`는 **타겟 플랫폼**과 **품질 옵션**을 정확히, `java`는 **경로/모듈/리플렉션 플래그**를 명확히, `jar`는 **실행 가능·멀티릴리스·모듈 친화**를 의도대로 다뤄야 합니다.
- 모듈 시스템(JPMS)을 도입하면 **캡슐화 강화·조기 검증** 이점을 얻는 대신, 리플렉션/테스트 경계에 `opens`/`--add-opens` 설계를 병행해야 합니다.
- 본 문서의 표·예제·치트시트를 그대로 베이스라인으로 삼아, **클래식 → 모듈**로 점진 전환해도 무리가 없습니다.
