---
layout: post
title: 디자인패턴 - Strategy vs State 패턴 차이와 선택 기준
date: 2025-07-05 21:20:23 +0900
category: 디자인패턴
---
# Strategy vs State 패턴 차이와 선택 기준

## 1. 한눈 요약

| 구분 | Strategy | State |
|---|---|---|
| 핵심 목적 | **알고리즘 교체** | **상태 전이에 따른 행위 전환** |
| 전환 주체 | **외부**(클라이언트·설정·컨텍스트 setter) | **내부**(상태 객체 또는 컨텍스트가 전이 결정) |
| 관계 | 전략 간 상호 독립 | 상태 간 **유기적 전이 관계** |
| 흐름 | 선택 → 실행 (전이 개념 없음) | 입력·이벤트에 따른 **전이 그래프** |
| 좋은 예 | 정렬/압축/결제 방식, 포맷터/시리얼라이저 | 세션/문서/네트워크 연결 상태, FSM 기반 UI/워크플로 |
| 흔한 냄새 | 전략임에도 상태 플래그로 분기 | 상태임에도 외부에서 임의로 setState() |

---

## 2. 공통 구조와 중요한 차이

```
Context ─────▶ Strategy/State (인터페이스) ─────▶ ConcreteX (실제 행위)
```

- 공통: Context가 인터페이스로 **행위 위임**.
- 차이:
  - **Strategy**: 어떤 구현을 쓸지 **외부가 선택**. 구현 간 전이는 없음.
  - **State**: **현재 상태 객체**가 다음 상태 전이를 **내부에서** 결정.

---

## 3. 수학적 모델로 보는 차이

### 3.1 Strategy 선택 함수
알고리즘 집합을 \( \mathcal{A} = \{a_1,\dots,a_n\} \)라 하자. 환경/설정 \(e\), 입력 \(x\)에 대해 **선택 함수**:
$$
\sigma: E \times X \rightarrow \mathcal{A}
$$
Context는 \( a = \sigma(e, x) \)를 선택해 실행한다. **전이 개념 없음**.

### 3.2 State 유한상태기계(FSM)
상태 집합 \( S \), 입력 \( I \), 전이 함수 \( \delta \)와 출력 \( \lambda \)가 있을 때:
$$
\delta: S \times I \rightarrow S,\quad \lambda: S \times I \rightarrow O
$$
입력에 따라 상태가 바뀌고, **행동은 상태에 종속**된다.

---

## 4. 코드로 비교: 같은 도메인, 다른 의도

### 4.1 Strategy 예제 — 압축기 교체 (Python)

```python
from abc import ABC, abstractmethod
import gzip, bz2, lzma

# Strategy 인터페이스
class Compressor(ABC):
    @abstractmethod
    def compress(self, data: bytes) -> bytes: ...

# 구체 전략
class GzipCompressor(Compressor):
    def compress(self, data: bytes) -> bytes:
        return gzip.compress(data)

class Bzip2Compressor(Compressor):
    def compress(self, data: bytes) -> bytes:
        return bz2.compress(data)

class LzmaCompressor(Compressor):
    def compress(self, data: bytes) -> bytes:
        return lzma.compress(data)

# Context: 외부가 전략을 선택·교체
class ArchiveService:
    def __init__(self, compressor: Compressor):
        self._c = compressor
    def set_strategy(self, compressor: Compressor):
        self._c = compressor
    def archive(self, payload: bytes) -> bytes:
        return self._c.compress(payload)

# 사용
svc = ArchiveService(GzipCompressor())
blob = svc.archive(b"hello")
svc.set_strategy(LzmaCompressor())  # 외부가 교체
blob2 = svc.archive(b"world")
```

핵심: **전략 간 전이 관계 없음**. 어떤 알고리즘을 쓸지 **외부**가 결정.

---

### 4.2 State 예제 — 업로더 FSM (Python)

요구: Idle → Uploading → Paused/Completed/Error 로 전이. 이벤트는 `start/pause/resume/success/fail`.

```
[Idle] --start--> [Uploading] --success--> [Completed]
                      |  ^                ^
                    pause|              reset
                      v  |resume
                   [Paused] --fail--> [Error] --reset--> [Idle]
```

```python
from abc import ABC, abstractmethod

# 이벤트 정의
class Event:
    START="start"; PAUSE="pause"; RESUME="resume"
    SUCCESS="success"; FAIL="fail"; RESET="reset"

# 전방 선언 위해 컨텍스트 타입 힌트 지연
class Uploader: ...

# 상태 인터페이스
class State(ABC):
    @abstractmethod
    def on(self, ctx: 'Uploader', event: str) -> None: ...

# 구체 상태들
class Idle(State):
    def on(self, ctx, event):
        if event == Event.START:
            ctx.log("업로드 시작")
            ctx.set_state(Uploading())
        else:
            ctx.log("무시")

class Uploading(State):
    def on(self, ctx, event):
        if event == Event.PAUSE:
            ctx.log("일시정지")
            ctx.set_state(Paused())
        elif event == Event.SUCCESS:
            ctx.log("완료")
            ctx.set_state(Completed())
        elif event == Event.FAIL:
            ctx.log("에러")
            ctx.set_state(Error())
        else:
            ctx.log("업로드 중: 무시")

class Paused(State):
    def on(self, ctx, event):
        if event == Event.RESUME:
            ctx.log("재개")
            ctx.set_state(Uploading())
        elif event == Event.FAIL:
            ctx.log("에러")
            ctx.set_state(Error())
        else:
            ctx.log("일시정지: 무시")

class Completed(State):
    def on(self, ctx, event):
        if event == Event.RESET:
            ctx.log("초기화")
            ctx.set_state(Idle())
        else:
            ctx.log("완료 상태: 무시")

class Error(State):
    def on(self, ctx, event):
        if event == Event.RESET:
            ctx.log("다시 시도 준비")
            ctx.set_state(Idle())
        else:
            ctx.log("에러 상태: 무시")

# 컨텍스트
class Uploader:
    def __init__(self):
        self._state: State = Idle()
        self.history = []
    def set_state(self, s: State):
        self._state = s
    def dispatch(self, event: str):
        self._state.on(self, event)
    def log(self, msg: str):
        self.history.append(msg)

# 사용
u = Uploader()
u.dispatch(Event.START)    # Idle -> Uploading
u.dispatch(Event.PAUSE)    # Uploading -> Paused
u.dispatch(Event.RESUME)   # Paused -> Uploading
u.dispatch(Event.SUCCESS)  # Uploading -> Completed
u.dispatch(Event.RESET)    # Completed -> Idle

print("\n".join(u.history))
```

핵심: **전이 로직이 상태/컨텍스트 내부에 존재**. 외부는 보통 이벤트만 보낸다.

---

## 5. 선택 체크리스트 & 의사결정 트리

### 5.1 체크리스트
- **동일 작업의 대체 알고리즘**이 필요하고, 서로 간 전이가 필요 없다 → **Strategy**.
- **시간/이벤트에 따라 합법적 전이**가 존재하고, 상태에 따라 허용 행위가 바뀐다 → **State**.
- 외부 설정이나 기능 토글에 따라 교체 → Strategy.
- 워크플로·UI 단계·프로토콜 단계처럼 **전이 테이블**이 그려진다 → State.

### 5.2 의사결정 트리(ASCII)
```
필요는 "전이"인가 "교체"인가?
           ├─ 전이(FSM) → State
           └─ 교체(알고리즘 선택) → Strategy
```

---

## 6. 실무 매핑 표

| 도메인 | Strategy 적합 | State 적합 |
|---|---|---|
| 정렬/압축/암호화 방식 선택 | ■ | |
| 결제 수단(카드/포인트/간편결제) | ■ | |
| 문서 워크플로(초안→검토→승인) | | ■ |
| 로그인/세션 상태 | | ■ |
| 네트워크 연결(TCP 상태) | | ■ |
| 추천 알고리즘 A/B | ■ | |
| 다운로드 매니저(대기/진행/중지/완료) | | ■ |

---

## 7. 함께 쓰면 좋은 패턴

- Strategy × Factory: 전략 선택 로직을 **팩토리/DI**로 외부화.
- State × Memento: **되돌리기(Undo)/복원**이 필요하면 상태 스냅샷 저장.
- State × Observer: 상태 변경을 **구독**하여 UI/로그/메트릭 반응.
- Strategy × Command: 알고리즘 실행을 **명령 객체**로 큐잉/로깅.

---

## 8. 안티패턴과 냄새

| 냄새 | 설명 | 대책 |
|---|---|---|
| 거대 if/switch 분기 | 알고리즘/상태 분기가 한 클래스에 뒤섞임 | Strategy/State로 분산 |
| 외부 setState 남발 | State인데 외부가 임의 전이 강제 | 전이는 상태/컨텍스트 내부로 캡슐화 |
| 전략이 서로 전이 | Strategy인데 상태처럼 다음 전략을 스스로 지정 | 전이 필요하면 State로 설계 변경 |
| 플래그 남용 | boolean/enum 플래그가 늘어남 | 상태 클래스로 승격 |
| Service Locator | 런타임 의존 숨김 | 생성자 주입/팩토리로 명시 |

---

## 9. 테스트 전략

### 9.1 Strategy 테스트
- 각 전략을 **독립 단위 테스트**.
- 컨텍스트는 **전략 주입/교체**가 가능한지 확인.
- 경계값·성능 비교를 데이터 주도 테스트로.

```python
def test_sort_desc():
    ctx = ArchiveService(LzmaCompressor())  # 단순 예시에서 이름만 차용
    data = b"a"*10
    assert isinstance(ctx.archive(data), bytes)
```

### 9.2 State 테스트
- **전이 테이블 기반** 시나리오 테스트.
- 올바르지 않은 이벤트가 무시/예외 처리되는지.
- 복구 경로(RESET 등) 검증.

```python
def test_uploader_flow():
    u = Uploader()
    u.dispatch("start"); u.dispatch("pause"); u.dispatch("resume"); u.dispatch("success")
    assert "완료" in "\n".join(u.history)
```

---

## 10. 성능·동시성 관점

- Strategy: 호출 오버헤드는 **가상 호출 1회** 수준. 선택 로직(팩토리/DI)이 잦다면 **캐시**.
- State: 전이 빈도가 매우 높다면 **테이블 기반 전이**(2차원 배열/딕셔너리)로 분기 비용을 상수화.
- 멀티스레딩: **Context의 상태 변경(set_state)** 는 원자성 보장 필요. 불변 상태 객체를 전제로 **CAS/락 최소화**.

테이블 전이 예(파이썬 딕셔너리):
```python
TRANSITIONS = {
  ("Idle","start"): "Uploading",
  ("Uploading","pause"): "Paused",
  # ...
}
```

---

## 11. 리팩터링 절차

### 11.1 거대 분기를 Strategy로
1) 분기 기준(알고리즘)을 인터페이스로 추출
2) 구현별 클래스 분리
3) 컨텍스트에 인터페이스 주입
4) 선택 로직은 **팩토리/DI** 로 이동

### 11.2 플래그/열거형 상태를 State로
1) 상태별 행위를 **상태 클래스**로 추출
2) 전이 규칙을 상태/컨텍스트 내부로 이동
3) 외부의 직접 전이 금지(이벤트만 노출)
4) 전이 다이어그램/테이블 문서화

---

## 12. 확장 예시(타 언어)

### 12.1 C# Strategy
```csharp
public interface IDiscount { decimal Apply(decimal price); }
public sealed class NoDiscount : IDiscount { public decimal Apply(decimal p) => p; }
public sealed class RateDiscount : IDiscount { private readonly decimal r; public RateDiscount(decimal r)=>this.r=r; public decimal Apply(decimal p)=>p*(1-r); }

public sealed class Cart {
  public IDiscount Discount { get; set; }
  public Cart(IDiscount discount) => Discount = discount;
  public decimal Checkout(decimal price) => Discount.Apply(price);
}
// 사용: var cart = new Cart(new RateDiscount(0.1m));
```

### 12.2 Java State
```java
interface State { void on(Context c, String evt); }
class Context { State s = new Idle(); void dispatch(String e){ s.on(this,e);} void set(State ns){ s=ns; } }

class Idle implements State { public void on(Context c, String e){ if(e.equals("start")) c.set(new Uploading()); } }
// ...
```

---

## 13. 설계 요령(핵심 문장 모음)

- **전이가 중요**하면 State, **교체가 중요**하면 Strategy.
- State는 **합법적 전이만 허용**하도록 내부에 규칙을 둔다.
- Strategy는 **교체 용이성**과 **테스트의 용이성**이 최우선.
- 전이/교체가 모두 필요하면 **외부 선택(Strategy) + 내부 전이(State)** 를 조합하되, **책임을 분리**한다.

---

## 14. 부록: 전이 검증을 수식으로 명세하기

상태 그래프 \( G = (S, E) \)에서 간선 \( E \subseteq S \times I \times S \). 합법 전이 집합을 \( \mathcal{T} \)라 하면, 구현의 전이 \( \delta \)가 다음을 만족해야 한다.
$$
\forall s \in S, \forall i \in I:\ (\exists s' \text{ s.t. } (s,i,s') \in \mathcal{T}) \Rightarrow \delta(s,i) \in \{ s' \mid (s,i,s') \in \mathcal{T} \}
$$
즉, **허용된 전이 집합 안에서만 이동**해야 한다.

---

## 15. 결론

- **Strategy**: “같은 일을 다른 방법으로” — 외부 선택/교체, 전이 없음.
- **State**: “상태에 맞는 행동을” — 내부 전이, 상태 모델 필수.
- 선택은 간단하다. **전이 다이어그램이 떠오르면 State, 아니면 Strategy**.
- 필요한 경우 두 패턴을 **조합**하되, 선택(교체)과 전이(상태)를 **분리된 책임**으로 관리하라.
