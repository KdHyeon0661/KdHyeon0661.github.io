---
layout: post
title: Java - javac, java, jar 명령어
date: 2025-08-09 22:20:23 +0900
category: Java
---
# `javac`, `java`, `jar` 명령어 — 자세한 정리 (JDK 8+ / 9+ 기준)

Java 개발에서 가장 많이 쓰는 **컴파일(javac)**, **실행(java)**, **패키징(jar)** 명령을 한 번에 정리했습니다.  
클래스패스(ClassPath)와 모듈패스(ModulePath) 모두를 다루며, **자주 쓰는 옵션·실전 예제·트러블슈팅**까지 포함합니다.

---

## 1) `javac` — Java 컴파일러

### 1.1 기본형식
```bash
javac [옵션] <소스파일...>
```

### 1.2 가장 자주 쓰는 옵션
| 옵션 | 의미 / 사용 시점 | 메모 |
|---|---|---|
| `-d <dir>` | 클래스 출력 디렉터리 | 패키지 경로에 맞춰 폴더 생성 |
| `-cp` 또는 `-classpath` | 클래스패스 설정 | 외부 라이브러리(JAR)·클래스 위치 |
| `--module-path` 또는 `-p` | 모듈패스 설정 | JPMS 사용 시 필수 |
| `--module-source-path` | 다중 모듈 소스 트리 | 모듈별 소스 빌드 |
| `--release <n>` | **타겟 플랫폼 API/바이트코드 동시 지정** | JDK 9+ 권장(예: `--release 8`) |
| `-source` / `-target` | 소스/바이트코드 버전 | 구버전 호환, **`--release` 권장** |
| `-encoding <enc>` | 소스 인코딩 | UTF-8 권장 |
| `-Xlint[:all]` | 경고 활성화 | 품질 관리 |
| `-Werror` | 경고를 에러로 처리 | CI에서 유용 |
| `-parameters` | 파라미터 이름 보존 | 리플렉션/프레임워크 친화 |
| `-proc:only` / `-processor` | 애너테이션 프로세싱 | Lombok/MapStruct 등 |
| `--enable-preview` | 프리뷰 기능 컴파일 | 실행 시에도 동일 플래그 필요 |
| `@argfile` | 옵션/파일 목록 외부파일로 | 긴 커맨드를 정리 |

### 1.3 기본 컴파일 예
```bash
# 현재 폴더에 .class 생성
javac Hello.java
```

패키지 구조를 반영하여 산출물 정리:
```bash
# src 밑 소스를 bin으로 출력
javac -d bin src/com/example/app/Main.java
```

클래스패스(외부 라이브러리 포함):
```bash
# Unix/macOS
javac -cp "lib/*:bin" -d bin src/com/example/app/Main.java
# Windows
javac -cp "lib/*;bin" -d bin src\com\example\app\Main.java
```

모듈 빌드:
```bash
javac -d out \
  --module-source-path src \
  $(find src -name "*.java")
```

버전 호환(권장):
```bash
# JDK 17에서 JDK 8용 바이트코드 + 8의 표준 API만 사용하게
javac --release 8 -d bin src/**/*.java
```

---

## 2) `java` — 애플리케이션 실행기

### 2.1 기본형식
```bash
java [옵션] <메인클래스> [args...]
java [옵션] -jar <실행가능-jar> [args...]
java [옵션] -m <모듈>/<메인클래스> [args...]  # 모듈 실행
```

### 2.2 필수/자주 쓰는 옵션
| 옵션 | 의미 |
|---|---|
| `-cp`/`--class-path` | 클래스패스 지정 |
| `-p`/`--module-path` | 모듈패스 지정 |
| `-m` | 모듈/메인클래스 실행 (`com.app/com.app.Main`) |
| `-jar` | 매니페스트의 `Main-Class`로 실행 |
| `-Dname=value` | 시스템 프로퍼티 지정 |
| `-Xms`, `-Xmx` | 힙 초기/최대 크기 |
| `-ea`/`-enableassertions` | assert 활성화 |
| `--add-opens` | 모듈 접근 열기(리플렉션 필요 시) |
| `--enable-preview` | 프리뷰 기능 실행(컴파일에도 동일 플래그 필요) |

### 2.3 실행 예

클래스패스 기반:
```bash
# 컴파일 산출물(bin)과 라이브러리(lib/*) 포함
java -cp "bin:lib/*" com.example.app.Main arg1 arg2
```

실행가능 JAR:
```bash
java -jar build/app.jar --mode=prod
```

모듈 실행:
```bash
java -p mods -m com.example.app/com.example.app.Main
```

단일 파일 소스 실행(Java 11+):
```bash
java Hello.java    # 한 번에 컴파일+실행
```

---

## 3) `jar` — JAR 아카이브 도구 (ZIP 기반)

### 3.1 기본형식 (긴 옵션 권장)
```bash
jar --create --file app.jar -C bin .
jar --list   --file app.jar
jar --extract --file app.jar -C out
```
단축 표기(전통): `jar cfe app.jar com.example.Main -C bin .`

### 3.2 자주 쓰는 옵션
| 옵션(긴형) | 단축 | 의미 |
|---|---|---|
| `--create` | `c` | 새 JAR 생성 |
| `--update` | `u` | 기존 JAR 업데이트 |
| `--list` | `t` | 목록 보기 |
| `--extract` | `x` | 추출 |
| `--file <jar>` | `f` | 대상 JAR 파일 경로 |
| `--verbose` | `v` | 상세 로그 |
| `--main-class <FQCN>` | `e`(과거 `-e`는 `-cfe` 조합) | 매니페스트 `Main-Class` 설정 |
| `--manifest <mfile>` | `m` | 사용자 지정 `MANIFEST.MF` 병합 |
| `--no-compress` |  | 압축하지 않고 저장 |
| `--module-version <ver>` |  | 모듈 버전 기입 |
| `--describe-module` |  | JAR에서 모듈 정보 출력 |
| `--validate` |  | 매니페스트·구조 검증 |
| `--release <N>` |  | **멀티릴리스 JAR**(MRJAR) 엔트리 추가 |

### 3.3 실행 가능한 JAR 만들기

컴파일 산출물 → JAR → 실행:
```bash
# 1) 컴파일 산출물 준비
javac -d bin src/com/example/app/*.java

# 2) Main-Class를 바로 지정하여 JAR 생성
jar --create --file app.jar --main-class com.example.app.Main -C bin .

# 3) 실행
java -jar app.jar
```

사용자 매니페스트 병합:
```text
# MANIFEST.MF (예시)
Manifest-Version: 1.0
Main-Class: com.example.app.Main
Class-Path: lib/guava.jar lib/logging.jar
```
```bash
jar --create --file app.jar --manifest MANIFEST.MF -C bin .
```

라이브러리 동봉(일반 `jar`는 **셔딩(병합)**을 해주지 않습니다):  
여러 JAR을 하나로 합치려면 **쉐이딩 플러그인**(Maven Shade, Gradle Shadow) 사용을 권장.

### 3.4 추출/목록/검증
```bash
jar --list --file app.jar
jar --extract --file app.jar -C out
jar --validate --file app.jar
```

### 3.5 모듈 친화 JAR
`module-info.class`를 포함해 **모듈 JAR**로 배포 가능.  
멀티릴리스 JAR(MRJAR, `--release`)로 JDK 버전별 구현을 함께 담을 수 있습니다.

---

## 4) 클래스패스 vs 모듈패스 — 빠른 비교

| 항목 | 클래스패스(ClassPath) | 모듈패스(ModulePath, JPMS) |
|---|---|---|
| 접근제어 | public 공개 시 전역 접근 | `exports`로 선택적 공개, 강한 캡슐화 |
| 선언 | jar/폴더 나열 | 모듈 단위, `module-info.java` |
| 실행 | `-cp` | `-p`, `-m` |
| 의존성 누락 | 런타임에야 오류 | 컴파일 타임에 조기 검증 가능 |
| 리플렉션 차단 | 없음 | 기본 차단(`opens` 필요) |

---

## 5) 실전 워크플로우 예시

### 5.1 클래식(클래스패스) 프로젝트
```bash
# 컴파일
javac -cp "lib/*" -d bin src/com/example/app/*.java

# JAR 생성
jar --create --file app.jar --main-class com.example.app.Main -C bin .

# 실행
java -cp "app.jar:lib/*" com.example.app.Main
# 또는
java -jar app.jar
```

### 5.2 모듈 프로젝트
```
src/
 └─ com.example.app/
     ├─ module-info.java
     └─ com/example/app/Main.java
```
```bash
# 컴파일
javac -d mods/com.example.app --module-source-path src $(find src -name "*.java")

# 실행
java -p mods -m com.example.app/com.example.app.Main
```

---

## 6) 트러블슈팅(빈출 에러)

| 에러/증상 | 원인 | 해결 |
|---|---|---|
| `Error: Could not find or load main class ...` | 클래스패스/패키지 경로 불일치 | `-cp` 확인, FQCN(패키지 포함)으로 실행 |
| `NoClassDefFoundError` | 런타임에 필요한 JAR 누락 | 실행 시 `-cp`(또는 매니페스트 `Class-Path`)에 추가 |
| `UnsupportedClassVersionError` | 바이트코드 버전이 더 높음 | `--release`/`-target` 재설정, JRE/JDK 버전 맞추기 |
| 모듈 접근 관련 `InaccessibleObjectException` | 모듈 캡슐화로 리플렉션 차단 | `--add-opens module/package=ALL-UNNAMED` 또는 `opens` 사용 |
| 프리뷰 기능 실행 불가 | 컴파일/실행 중 하나만 `--enable-preview` | 둘 다 `--enable-preview` + 동일 JDK |

---

## 7) 베스트 프랙티스

1. **호환 빌드**: JDK 9+에서는 `--release N` 사용(단순 `-source/-target`보다 안전).  
2. **경로 관리**: OS별 클래스패스 구분자(`:` vs `;`) 주의, 와일드카드 `lib/*` 적극 활용.  
3. **품질**: `-Xlint:all -Werror -parameters`로 경고/메타데이터 관리.  
4. **프리뷰**: 학습/실험 외엔 신중하게(`--enable-preview`).  
5. **모듈**: 외부 프레임워크(리플렉션) 사용 시 `opens`/`--add-opens` 설계부터 고려.  
6. **패키징**: 실행가능 JAR + 쉐이딩(필요 시) or 런처 스크립트. 배포 크기 최적화는 `jlink`도 검토.
