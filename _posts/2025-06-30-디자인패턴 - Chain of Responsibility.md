---
layout: post
title: 디자인패턴 - Chain of Responsibility
date: 2025-06-30 22:20:23 +0900
category: 디자인패턴
---
# Chain of Responsibility (책임 연쇄 패턴)

## ✅ 정의

**책임 연쇄 패턴(Chain of Responsibility Pattern)**은 **요청을 처리할 수 있는 객체들을 체인 형태로 연결**해 두고,  
각 객체는 요청을 처리하거나 다음 객체로 **책임을 넘기는 방식**의 **행위 패턴**입니다.

> “요청을 보내는 쪽과 처리하는 쪽을 분리하고, 여러 처리자 중 누가 처리할지는 런타임에 결정”

---

## 🎯 의도 (Intent)

- 요청을 처리할 여러 객체가 존재할 때, **하나의 객체에 전적으로 의존하지 않도록**
- 처리자들 사이에 **유연한 연결 구조(연쇄 구조)**를 형성
- **요청 처리자(Handler)의 변경이나 확장**이 쉽도록 유도

---

## 📦 구조 (UML)

```
┌────────────┐
│  Handler   │◄──────────────────┐
├────────────┤                  │
│handle(req) │◄────┐             │
│setNext(h)  │     │             │
└─────┬──────┘     │             │
      ▼            ▼             ▼
┌────────────┐ ┌────────────┐ ┌────────────┐
│Handler A   │ │Handler B   │ │Handler C   │
└────────────┘ └────────────┘ └────────────┘
```

- `Handler`: 요청 처리 인터페이스 및 다음 처리자를 등록하는 메서드 포함
- `ConcreteHandler`: 실제 요청 처리자, 조건에 맞으면 처리하고 아니면 넘김
- `Client`: 요청을 시작하는 주체

---

## 🧑‍💻 구현 예시 (Python)

```python
from abc import ABC, abstractmethod

# 추상 핸들러
class Handler(ABC):
    def __init__(self):
        self._next_handler = None

    def set_next(self, handler):
        self._next_handler = handler
        return handler  # 체이닝 지원

    @abstractmethod
    def handle(self, request):
        pass

# 구체 핸들러 1
class LowLevelSupport(Handler):
    def handle(self, request):
        if request < 10:
            print(f"🔧 LowLevelSupport 처리함: {request}")
        elif self._next_handler:
            self._next_handler.handle(request)

# 구체 핸들러 2
class MidLevelSupport(Handler):
    def handle(self, request):
        if 10 <= request < 50:
            print(f"🛠️ MidLevelSupport 처리함: {request}")
        elif self._next_handler:
            self._next_handler.handle(request)

# 구체 핸들러 3
class HighLevelSupport(Handler):
    def handle(self, request):
        if request >= 50:
            print(f"🚨 HighLevelSupport 처리함: {request}")
        elif self._next_handler:
            self._next_handler.handle(request)

# 체인 구성 및 요청 처리
low = LowLevelSupport()
mid = MidLevelSupport()
high = HighLevelSupport()

low.set_next(mid).set_next(high)

requests = [5, 15, 55, 100]
for r in requests:
    low.handle(r)
```

**출력 예시:**
```
🔧 LowLevelSupport 처리함: 5
🛠️ MidLevelSupport 처리함: 15
🚨 HighLevelSupport 처리함: 55
🚨 HighLevelSupport 처리함: 100
```

---

## ✅ 장점

- **요청 발신자와 수신자 분리** (느슨한 결합)
- 처리자 간의 **동적 연결 변경 가능**
- 새로운 처리자 추가 시, 기존 코드 **수정 없이 확장 가능 (OCP 만족)**
- 일부 처리자는 요청을 무시하거나 선택적으로 처리 가능

---

## ⚠️ 단점

- 디버깅이 어려울 수 있음 (어디서 처리되었는지 추적 어려움)
- 마지막까지 처리가 안 될 경우 → **누락 처리** 위험
- 체인이 길어질수록 **성능 저하 우려**

---

## 📌 사용 사례

| 분야 | 예시 |
|------|------|
| UI 이벤트 처리 | 버튼 클릭 이벤트가 여러 레이어에 전달됨 |
| 기술 지원 시스템 | 고객 요청 → 1차 → 2차 → 관리자 |
| 접근 제어 | 권한 검사 체인 (권한 → 소유자 → 관리자) |
| 로깅 시스템 | 로그 레벨별 처리 (debug → info → warn → error) |
| 게임 | 충돌 처리, 무기 공격 처리 체인 |

---

## 🧠 Chain of Responsibility vs Command vs Decorator

| 패턴 | 목적 | 유사점 | 차이점 |
|------|------|--------|--------|
| Chain of Responsibility | 순차적 책임 위임 | 객체 간 전달 | 한 요청을 하나만 처리할 수도 |
| Command | 요청 캡슐화 | 요청을 객체로 처리 | 요청 큐/실행 취소 중심 |
| Decorator | 기능 확장 | 객체 연결 | 처리 흐름을 바꾸지 않음 |

---

## ✅ 실무 팁

- **권한 체크**, **필터 처리**, **요청 전처리** 등에 매우 효과적
- 체인을 동적으로 구성하거나 설정파일(JSON, XML 등)로 정의 가능
- Python에선 `yield`, `middleware`, `flask`, `Django` 미들웨어 등에서도 유사 구조

---

## 🧠 마무리

**Chain of Responsibility 패턴은 요청 처리 책임을 여러 객체에 위임하면서도**,  
**각 객체가 처리 여부를 판단할 수 있도록 유연한 흐름을 제공합니다.**

수직적 구조의 처리 흐름(고객지원, 필터, 로깅 등)에 이상적이며,  
각 책임을 모듈화해 시스템의 유지보수성과 확장성을 높일 수 있습니다.