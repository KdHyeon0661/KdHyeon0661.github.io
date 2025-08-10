---
layout: post
title: 디자인패턴 - Singleton vs DI 컨테이너 비교
date: 2025-07-05 20:20:23 +0900
category: 디자인패턴
---
# Singleton vs DI 컨테이너 비교

---

## 🔍 개요

어플리케이션을 설계할 때 자주 나오는 개념 중 하나는 **"객체의 생명주기를 어떻게 관리할 것인가"**입니다.  
이때 흔히 두 가지 방법이 등장합니다:

- **Singleton 패턴 (디자인 패턴)**
- **DI 컨테이너 (의존성 주입 컨테이너)**

이 둘은 모두 **객체의 공유**를 전제로 하지만,  
**목적과 철학, 구현 방법, 유연성**에서 뚜렷한 차이를 보입니다.

---

## 📌 용어 정리

| 용어 | 설명 |
|------|------|
| Singleton | 클래스의 인스턴스를 오직 하나만 생성하도록 보장하는 디자인 패턴 |
| DI (Dependency Injection) | 객체가 자신의 의존 객체를 직접 생성하지 않고 외부로부터 주입받는 방식 |
| DI 컨테이너 | 의존성 주입을 자동화해주는 프레임워크 or 라이브러리 (ex: Spring, .NET Core) |

---

## 🧱 Singleton 패턴이란?

```python
class Singleton:
    _instance = None

    def __new__(cls):
        if not cls._instance:
            cls._instance = super().__new__(cls)
        return cls._instance
```

- 클래스 내부에서 인스턴스를 static 변수로 보관
- 외부에서 호출할 때 항상 동일한 인스턴스를 반환
- **어디에서든 접근 가능하지만, 강하게 결합됨**

---

## 🧰 DI 컨테이너란?

```csharp
// C# 예시 (.NET Core)
services.AddSingleton<ILogger, ConsoleLogger>();
services.AddTransient<IUserService, UserService>();
```

- 의존성 그래프를 DI 컨테이너가 관리
- 원하는 수명 주기 (`Singleton`, `Scoped`, `Transient`)에 따라 인스턴스를 생성 및 주입
- **설정 중심**, **동작은 외부 위임**, **테스트 및 교체 용이**

---

## ⚖️ Singleton vs DI 컨테이너: 비교

| 항목 | Singleton 패턴 | DI 컨테이너 |
|------|----------------|-------------|
| **객체 수명** | 애플리케이션 전체에서 단 하나 | DI 설정에 따라 유연하게 설정 가능 (`Singleton`, `Scoped`, `Transient`) |
| **접근 방식** | 전역 접근 (static 참조 또는 getInstance()) | 생성자 주입, 생성 시점 주입 |
| **테스트 용이성** | 테스트 어려움 (강한 결합, 모킹 어려움) | 인터페이스 기반 주입 → 모킹, 테스트 유리 |
| **확장성** | 단일 객체 → 유연성 부족 | 다양한 생명 주기 및 주입 방식 제공 |
| **의존성 관리** | 직접 생성 또는 내부 보관 | 외부 주입, IoC(Inversion of Control) 적용 |
| **스레드 안전성** | 직접 구현 필요 | 컨테이너가 보장 또는 설정 가능 |
| **OOP 원칙 준수** | SRP, DIP 위배 소지 있음 | SRP, DIP 등 OOP 원칙을 장려 |

---

## 🎯 언제 Singleton을 쓰고, 언제 DI를 써야 할까?

| 상황 | 추천 방식 |
|------|-----------|
| 매우 단순한 프로젝트 / 설정이 필요 없는 도구성 객체 | Singleton 패턴 |
| 복잡한 객체 간 관계가 있고, 테스트 가능해야 함 | DI 컨테이너 |
| 상태를 가지지 않는 유틸 클래스 (ex. 암호화, 설정 로더) | Singleton |
| 상태가 있는 비즈니스 객체, 데이터베이스 접근 객체 등 | DI 사용 권장 |

---

## 🧪 테스트 관점에서의 차이

```python
# Singleton
class Logger:
    _instance = None
    def __new__(cls):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
        return cls._instance

# Test 시 모킹 불가능 (결합도 높음)
log = Logger()
log2 = Logger()
assert log is log2  # True
```

```python
# DI 방식
class Logger: pass
class Service:
    def __init__(self, logger: Logger):
        self.logger = logger

# 테스트 시 모킹 가능
mock_logger = Mock(spec=Logger)
service = Service(logger=mock_logger)
```

---

## 🧠 실제 예시 비교

### Singleton 방식

```python
# 설정 객체를 전역으로 유지
class Config:
    _instance = None
    def __new__(cls):
        if not cls._instance:
            cls._instance = super().__new__(cls)
            cls._instance.value = "Production"
        return cls._instance
```

### DI 컨테이너 방식 (.NET Core)

```csharp
services.AddSingleton<IConfig, AppConfig>();

public class Service {
    private readonly IConfig config;
    public Service(IConfig config) {
        this.config = config;
    }
}
```

---

## ✅ 결론

| 항목 | Singleton 패턴 | DI 컨테이너 |
|------|----------------|-------------|
| ✅ 장점 | 간단, 빠름, 코드 줄어듦 | 테스트 용이, 확장성, 유지보수 좋음 |
| ❌ 단점 | 테스트 어려움, 전역 상태 문제 | 설정 필요, 학습 곡선 존재 |

> **결론**: "단순한 도구성 객체는 Singleton도 괜찮지만,  
> **애플리케이션 규모가 커질수록 DI 컨테이너를 통한 관리가 훨씬 유리합니다.**"