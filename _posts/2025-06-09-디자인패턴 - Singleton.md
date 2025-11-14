---
layout: post
title: 디자인패턴 - Singleton
date: 2025-06-09 22:20:23 +0900
category: 디자인패턴
---
# Singleton(싱글턴 패턴)

## 정의(Definition)

**싱글턴 패턴(Singleton Pattern)**은 **클래스 인스턴스를 오직 하나만 생성**하도록 제한하고 그 인스턴스에 접근할 수 있는 **전역 접근점**을 제공하는 생성 패턴이다.
전역으로 공유되어야 하는 **설정, 로깅, 캐시, 리소스 레지스트리, 연결 풀 매니저** 등에서 사용된다.

---

## 의도(Intent)

- 애플리케이션 전역에서 **동일한 인스턴스 하나**만 쓰도록 보장한다.
- 어디서든 접근 가능한 **공용 접근점**을 제공한다(직접 전역변수보다 통제력이 높음).
- **수명 관리**(초기화/파괴 시점)를 객체 내부에서 일관되게 제어한다.

---

## 구조(UML) — 제공 그림 보존

```
┌─────────────────────┐
│     Singleton       │
├─────────────────────┤
│ - instance: static  │
├─────────────────────┤
│ + getInstance()     │
└─────────────────────┘
```

- `instance`: 클래스 내부 유일 인스턴스를 저장(정적/클래스 변수).
- `getInstance()`: 외부에서 인스턴스를 얻는 **정적 메서드/속성**.

---

## 최소 구현(파이썬) — 게으른 초기화

```python
class Singleton:
    _instance = None  # 클래스 변수로 유일 인스턴스 저장

    def __new__(cls):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
            # 선택: 초기화 코드가 있다면 여기서 한 번만
        return cls._instance

# 사용

a = Singleton()
b = Singleton()
print(a is b)  # True
```

- 간단하지만, **멀티스레드 환경**에선 경쟁 조건이 있을 수 있다(아래 §7 참고).

---

## Java 기본 구현 — 게으른 초기화(학습용)

```java
public class Singleton {
    private static Singleton instance;

    private Singleton() {}  // 외부에서 생성 금지

    public static Singleton getInstance() {
        if (instance == null) {
            instance = new Singleton(); // 단일 스레드 전제에서만 안전
        }
        return instance;
    }
}
```

- 단일 스레드 환경 또는 초기화 순서가 보장된 문맥에 한해 안전.
- 멀티스레드에서는 **Double-Checked Locking + volatile**, **정적 홀더**, **enum 싱글턴**을 권장(§7, §8).

---

## C# 기본 구현 — `Lazy<T>`로 안전·간결

```csharp
public sealed class Singleton
{
    private static readonly Lazy<Singleton> _inst = new(() => new Singleton());
    public static Singleton Instance => _inst.Value;

    private Singleton() { /* 1회 초기화 */ }
}
```

- .NET의 `Lazy<T>`는 기본적으로 **스레드 안전**하다.
- 또는 정적 생성자(static ctor)도 **CLR 보장**으로 스레드 안전한 1회 초기화가 가능.

---

## 멀티스레드 환경: 안전한 패턴들

### 파이썬: GIL이 있어도 안전하다고 단정 못 한다

- 객체 생성 경합이 **여러 바퀴** 발생할 가능성이 있다(특히 CPython 외 구현, C 확장, I/O 동반 상황).
- 안전한 구현:

```python
import threading

class ThreadSafeSingleton:
    _instance = None
    _lock = threading.Lock()

    def __new__(cls):
        if cls._instance is None:
            with cls._lock:
                if cls._instance is None:  # Double-checked locking
                    cls._instance = super().__new__(cls)
                    # 1회 초기화 코드
        return cls._instance
```

> 참고: 파이썬에서는 **모듈 자체**가 사실상 싱글턴처럼 동작한다(모듈 전역 상태는 한 번만 로드됨). 간단한 경우 “모듈을 싱글턴”으로 쓰는 것이 가장 명료하다(§10.1).

### 자바: DCL은 `volatile` 필수

```java
public class Singleton {
    private static volatile Singleton instance; // volatile 중요!

    private Singleton() {}

    public static Singleton getInstance() {
        if (instance == null) {
            synchronized (Singleton.class) {
                if (instance == null) {
                    instance = new Singleton(); // 안전
                }
            }
        }
        return instance;
    }
}
```

### 자바: 정적 내부 클래스(Initialization-on-demand holder)

```java
public class Singleton {
    private Singleton() {}

    private static class Holder {
        static final Singleton INSTANCE = new Singleton();
    }
    public static Singleton getInstance() { return Holder.INSTANCE; }
}
```
- 클래스 로딩/초기화 규칙 덕분에 **지연 로딩 + 스레드 안전**.

### 자바: enum 싱글턴(리플렉션/직렬화까지 방어)

```java
public enum SingletonEnum {
    INSTANCE;
    // 필드/메서드 추가 가능
}
```
- 가장 견고한 방식. **리플렉션/직렬화 공격에 강함**.

### C#: 정적 생성자 / `Lazy<T>`

- `static Singleton() { ... }`는 **한 번만** 실행됨(런타임 보장).
- `Lazy<T>`는 구현이 간결하고 예외 캐싱/재시도 정책 등 옵션 제공.

---

## 직렬화/리플렉션 방어

### 자바 직렬화: `readResolve()`로 단일성 보존

```java
private Object readResolve() {
    return getInstance(); // 역직렬화 시 새로 만들지 않고 기존 인스턴스 반환
}
```

### 자바 리플렉션 방어

- 생성자에서 **두 번째 생성 시도**를 감지해 예외:
```java
public class Singleton {
    private static boolean created = false;
    private Singleton() {
        if (created) throw new RuntimeException("Use getInstance()");
        created = true;
    }
}
```
- 단, 완벽하지 않다. 가장 견고한 해법은 **enum 싱글턴**.

### 파이썬 피클링: 동일 인스턴스 유지

```python
import pickle

class PickleSafeSingleton(ThreadSafeSingleton):
    def __reduce__(self):
        # 역직렬화 시 __new__ 경유하여 동일 인스턴스 보장
        return (self.__class__, ())
```

---

## 사용 예시와 주의점(현업 감각)

| 사용 사례 | 설명 | 주의점 |
|---|---|---|
| 로깅 시스템 | 전역 동일 로거/싱크 관리 | 다중 프로세스/컨테이너 환경에서 파일 핸들 충돌 주의 |
| 설정(Config) | 설정값 1회 로드·전역 공유 | 동적 재로딩 설계(감시/원자 교체) |
| 캐시 매니저 | 전역 캐시 정책/풀 관리 | 메모리 상한, 만료, 동시성, 백프레셔 |
| 연결 풀 매니저 | 풀의 생명주기 단일화 | **단일 “연결”**을 싱글턴으로 만들지 말 것(고장/재연결 취약) |
| 피처 플래그 | 전역 플래그 조회 | 강한 일관성/최신성 요구 시 구독/폴링/웹훅 설계 |

> 특히 “DB 연결” 자체를 싱글턴으로 만드는 것은 위험하다. **연결 풀**(매니저)을 싱글턴으로 두고, **개별 연결은 짧게 빌리고 반납**하라.

---

## 언어별 구현 대안/변형

### 파이썬

- **모듈 = 싱글턴**: 설정/레지스트리처럼 간단한 전역 상태는 모듈 전역으로 두는 것이 가장 명확하다.
- **모노스테이트(Borg) 패턴**: 인스턴스는 여러 개여도 **상태를 공유**.
```python
class Borg:
    _shared = {}
    def __init__(self):
        self.__dict__ = self._shared

class AppConfig(Borg):
    def __init__(self):
        super().__init__()
        self.__dict__.setdefault("region", "ap-northeast-2")

a = AppConfig(); b = AppConfig()
a.region = "us-east-1"
print(b.region)  # us-east-1 (상태 공유)
```
- **메타클래스 기반**: `__call__`을 가로채어 단일 인스턴스 보장(필요할 때만, 과잉 설계 금지).

### 자바

- **enum 싱글턴** 추천(리플렉션·직렬화에 강함).
- **홀더 패턴**(§7.3)도 안전·간결.

### C#/.NET

- DI 컨테이너의 **ServiceLifetime.Singleton**과 함께 쓰는 것이 현실적.
- `IHostedService`/백그라운드 서비스의 **수명 관리**와 충돌하지 않게 설계.

---

## DI/IoC와의 관계 — “전역 vs 싱글턴 수명”

- 현대 애플리케이션에서는 **전역 접근**보다 **DI 컨테이너가 주입하는 싱글턴 수명**을 더 선호한다.
- DI의 세 가지 수명(ASP.NET Core):
  - **Singleton**: 앱 전체에서 1개 인스턴스
  - **Scoped**: 요청(스코프)당 1개
  - **Transient**: 요청마다 새 인스턴스
- 권장: “진짜 전역”이 필요하지 않다면 **DI 싱글턴**으로 충분. 테스트·교체 가능성이 훨씬 좋다.
- **Service Locator**는 안티패턴으로 간주(숨은 의존성 증가). 가능하면 **명시적 주입**을 유지.

---

## 테스트 전략

- **문제**: 전역 싱글턴은 **테스트 간 상태 누수**를 일으킨다.
- 해결책:
  1) **인터페이스/팩토리**로 감싸고, 테스트에서 **목 구현**을 주입.
  2) 테스트 프로세스 격리(별도 프로세스/도메인) 또는 테스트 후 **명시적 리셋 API**(테스트 전용) 제공.
  3) 파이썬: `monkeypatch`로 싱글턴 접근점을 테스트 더블로 교체.

예시(파이썬, 감싸기):
```python
class Logger:
    def log(self, msg): print(msg)

class LoggerProvider:
    # “싱글턴 접근점”은 이 프로바이더로 한 단계 감싼다
    _instance = Logger()
    @classmethod
    def get(cls) -> Logger: return cls._instance
    @classmethod
    def set(cls, inst: Logger): cls._instance = inst  # 테스트에서 교체

# 테스트

class FakeLogger(Logger):
    def __init__(self): self.buf=[]
    def log(self, msg): self.buf.append(msg)

fake = FakeLogger()
LoggerProvider.set(fake)
LoggerProvider.get().log("hello")
assert fake.buf == ["hello"]
```

---

## 수명/성능/동시성/프로세스 주의

- **수명 종료(Dispose/Close)**: C#에서는 `IDisposable` 자원(파일/소켓/핸들)을 싱글턴에 붙일 때 종료 시점이 애매해진다. 애플리케이션 종료 훅에서 명시 처리.
- **동시성**: 읽기 성능을 위해 불변(immutable) 상태 + 원자적 스왑(예: 참조 교체) 구조를 고려.
- **프로세스 경계**: 멀티프로세스(프리포크/컨테이너)에서는 “프로세스별 싱글턴”이다. **진짜 전역**이 필요하면 IPC/분산 캐시/코디네이션(ZooKeeper/Etcd) 등 별도 메커니즘을 쓴다.

---

## 싱글턴 vs 정적 클래스

| 항목 | 싱글턴 | 정적 클래스 |
|---|---|---|
| 인스턴스 | 있음(1개 보장) | 없음 |
| 다형성/인터페이스 | 가능 | 불가 |
| DI 주입/테스트 더블 | 용이 | 곤란 |
| 상태 수명 관리 | 가능(생성/파괴) | 런타임 전역 |
| 추천 상황 | 다형성/교체 필요 | 유틸리티 함수 묶음 |

- 다형적 교체/테스트가 중요하면 **싱글턴**(혹은 DI 싱글턴 수명).
- 순수 함수/유틸 모음이면 **정적 클래스/모듈**이 간단하고 충분.

---

## 리팩토링 경로 — “나쁜 전역”에서 “관리되는 싱글턴/DI”로

냄새
- 군데군데 **전역 변수** 참조, 초기화 순서 의존.
- 테스트에서 전역 상태 초기화가 어려움.

절차
1) 전역 접근을 **Provider/Factory**로 캡슐화(접근점 단일화).
2) Provider 뒤에 싱글턴을 배치(또는 DI 싱글턴으로 대체).
3) 테스트에서 Provider 교체 API/주입 경로 제공.
4) 초기화/파괴 수명 훅 정리.

---

## 실전 예제

### 설정(핫 리로드 지원) — 파이썬

```python
import json, threading, os, time

class ConfigSingleton:
    _instance = None
    _lock = threading.Lock()

    def __new__(cls, path="config.json", watch=False):
        if cls._instance is None:
            with cls._lock:
                if cls._instance is None:
                    inst = super().__new__(cls)
                    inst._path = path
                    inst._lock2 = threading.RLock()
                    inst._data = {}
                    inst._mtime = 0
                    inst._load()
                    if watch:
                        inst._start_watcher()
                    cls._instance = inst
        return cls._instance

    def _load(self):
        with self._lock2:
            st = os.stat(self._path)
            if st.st_mtime != self._mtime:
                with open(self._path, "r", encoding="utf-8") as f:
                    self._data = json.load(f)
                self._mtime = st.st_mtime

    def get(self, key, default=None):
        with self._lock2:
            return self._data.get(key, default)

    def _start_watcher(self):
        def loop():
            while True:
                try:
                    self._load()
                except Exception:
                    pass
                time.sleep(1.0)
        t = threading.Thread(target=loop, daemon=True)
        t.start()

# 사용

cfg = ConfigSingleton("config.json", watch=True)
print(cfg.get("region", "ap-northeast-2"))
```

포인트
- **원자적 교체**와 **락 범위 최소화**로 읽기 경합을 줄인다.
- 핫 리로드가 필요 없으면 더 단순화 가능.

### 로거 — C# (DI와 조화)

```csharp
public interface IMyLogger { void Log(string msg); }

public sealed class MyLogger : IMyLogger, IDisposable
{
    private readonly StreamWriter _writer;
    public MyLogger(string path) => _writer = new(path, append: true);
    public void Log(string msg) => _writer.WriteLine($"{DateTime.UtcNow:o} {msg}");
    public void Dispose() => _writer.Dispose();
}

// DI 등록: services.AddSingleton<IMyLogger>(sp => new MyLogger("app.log"));
```

- “전역”이 필요하더라도 **DI 싱글턴 수명**으로 관리하면 테스트/교체가 수월하다.

---

## 장점/단점(요약)

**장점**
- 전역 공용 리소스/정책을 **일관되게 관리**.
- 초기화/수명 관리가 한 곳에 모임.
- 전역 변수 난립보다 통제력이 높다.

**단점**
- **숨은 전역 상태** → 결합도 증가, 테스트 격리 어려움.
- 멀티스레드/프로세스에서 **경합/수명** 이슈.
- 남용 시 **God Object**로 비대해진다.

---

## 체크리스트

- 진짜로 **하나만** 필요한가(풀 매니저 vs 개별 연결)?
- 멀티스레드/프로세스, 직렬화/리플렉션, 종료 훅까지 고려했는가?
- 테스트에서 **교체 가능한 접근점**을 마련했는가(Provider/인터페이스)?
- DI 컨테이너의 **Singleton 수명**으로 대체 가능한가?
- 전역 상태에 **동기화/일관성** 요구가 과하지 않은가?

---

## 자주 하는 질문(FAQ)

- Q. 로거/설정을 전역 변수로 두면 안 되나?
  A. 가능하나 **초기화 순서/테스트 격리**에 취약하다. 접근점을 캡슐화하고, 가능하면 **DI 싱글턴**으로 관리하라.

- Q. 멀티프로세스에서 싱글턴은?
  A. “프로세스당 싱글턴”이다. 프로세스 간 공유는 IPC/공유 저장소/분산 코디네이터를 사용하라.

- Q. 연결을 싱글턴으로 두면 성능이 좋은가?
  A. 오히려 **장애 전파/재연결 문제**를 키운다. **연결 풀 관리자를 싱글턴**으로 두고, 연결은 짧게 빌려 쓰는 것이 표준.

---

## 결론

싱글턴은 “전역”을 **통제된 방식**으로 캡슐화하는 도구다.
그러나 **전역 상태 자체의 위험**은 남는다. 가능한 경우 **DI/IoC의 싱글턴 수명**으로 대체하고, 정말로 “유일성”이 필요한 리소스에만 적용하라. 스레드 안전, 직렬화/리플렉션, 종료 훅, 테스트 교체 가능성까지 확보해야 한다. 남용을 경계하되, **정확한 문제에 정확히 적용**하면 강력하다.
