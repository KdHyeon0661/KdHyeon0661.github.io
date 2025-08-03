---
layout: post
title: 디자인패턴 - Singleton
date: 2025-06-09 22:20:23 +0900
category: 디자인패턴
---
# Singleton (싱글턴 패턴)

## ✅ 정의

**싱글턴 패턴(Singleton Pattern)**은 **클래스의 인스턴스를 오직 하나만 생성하도록 제한**하고, **그 인스턴스에 접근할 수 있는 전역적인 접근점을 제공**하는 디자인 패턴입니다.

주로 **애플리케이션 전체에서 공유되어야 하는 자원**을 다룰 때 사용됩니다. 대표적인 예로는 **설정 클래스, 로깅 시스템, 데이터베이스 연결, 캐시 관리** 등이 있습니다.

---

## 🎯 의도 (Intent)

- 하나의 클래스에 대해 단 하나의 인스턴스만 존재하도록 보장한다.
- 어디서든 그 인스턴스에 접근할 수 있게 한다.

---

## 📦 구조 (UML)

```
┌─────────────────────┐
│     Singleton       │
├─────────────────────┤
│ - instance: static  │
├─────────────────────┤
│ + getInstance()     │
└─────────────────────┘
```

- `instance`: 클래스 내부에서 유일한 객체를 저장
- `getInstance()`: 외부에서 객체를 가져오기 위한 정적 메서드

---

## 🧑‍💻 구현 예시 (Python)

```python
class Singleton:
    _instance = None  # 클래스 변수로 인스턴스 저장

    def __new__(cls):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
        return cls._instance

# 사용
a = Singleton()
b = Singleton()
print(a is b)  # True
```

---

## 🔐 구현 예시 (Java)

```java
public class Singleton {
    private static Singleton instance;

    private Singleton() {}  // 외부에서 생성 불가

    public static Singleton getInstance() {
        if (instance == null) {
            instance = new Singleton();
        }
        return instance;
    }
}
```

---

## 🚀 멀티스레드 환경 대응 (Python)

```python
import threading

class ThreadSafeSingleton:
    _instance = None
    _lock = threading.Lock()

    def __new__(cls):
        if cls._instance is None:
            with cls._lock:
                if cls._instance is None:
                    cls._instance = super().__new__(cls)
        return cls._instance
```

> 💡 더블 체크 락킹(Double-checked locking) 기법을 사용하여 스레드 안전성 확보

---

## ✅ 장점

- **전역 상태 공유**가 가능
- 메모리 절약 (객체 하나만 생성)
- 접근이 간편함 (`getInstance()` 방식)

---

## ⚠️ 단점

- **전역 상태 공유**가 오히려 **결합도 증가**와 **테스트 어려움**을 유발할 수 있음
- 다중 스레드 환경에서는 별도의 **동기화 처리 필요**
- 의존성 주입(DI)과 함께 쓰기 어려움

---

## 📌 사용 예시

| 사용 사례             | 설명 |
|----------------------|------|
| 로깅 시스템          | 애플리케이션 전체에서 동일한 로거 인스턴스 사용 |
| 설정 정보 (Config)   | 설정값을 어디서나 공유해야 할 때 |
| 캐시 또는 리소스 풀  | 하나의 리소스 풀을 전역에서 공유 |
| 데이터베이스 연결    | 하나의 연결 객체를 여러 컴포넌트에서 공유 |

---

## 🧪 테스트 및 단위 테스트 고려

싱글턴은 테스트하기 어렵기 때문에, **의존성 주입**과 함께 사용하는 것이 좋습니다. 예를 들어, 테스트 시에는 Mock 싱글턴 객체를 주입할 수 있어야 합니다.

---

## 🔄 싱글턴 vs 정적 클래스

| 항목 | 싱글턴 | 정적 클래스 |
|------|--------|--------------|
| 인스턴스 | 있음 | 없음 |
| 상속 가능 | 가능 | 불가능 |
| 상태 유지 | 가능 | 가능 |
| 유연성 | 높음 | 낮음 |

---

## 🧠 마무리

싱글턴 패턴은 매우 강력한 도구지만 **남용하면 코드가 하드 코딩되고 테스트가 어려워질 수 있습니다.**  
따라서, 꼭 인스턴스를 하나만 써야 하는 상황에서만 사용하는 것이 좋습니다.  
DI 프레임워크(예: Spring, ASP.NET Core 등)에서는 자체적으로 싱글턴 주입 기능을 제공하기 때문에, **프레임워크의 도움을 받는 것이 일반적**입니다.
