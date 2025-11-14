---
layout: post
title: 디자인패턴 - Chain of Responsibility
date: 2025-06-30 22:20:23 +0900
category: 디자인패턴
---
# Chain of Responsibility (책임 연쇄 패턴)

## 정의 요약

**책임 연쇄 패턴(Chain of Responsibility, CoR)**은 요청을 처리할 수 있는 여러 **핸들러(Handler)**를 **연쇄(Chain)**로 연결하고, 각 핸들러가 요청을 **처리하거나 다음으로 넘기는** 행위 패턴이다.
요청자(클라이언트)와 처리자의 결합을 끊고, 처리 주체를 **런타임에 유연하게 교체/확장**할 수 있게 해 준다.

---

## 구조 (UML/흐름)

```
      ┌────────────┐     setNext()     ┌────────────┐     setNext()     ┌────────────┐
req ▶ │  Handler A │ ─────────────────▶ │  Handler B │ ─────────────────▶ │  Handler C │
      └─────┬──────┘                    └─────┬──────┘                    └─────┬──────┘
            │   handle(req)                    │   handle(req)                    │   handle(req)
            └─────────────▶ 처리 or 다음       └─────────────▶ 처리 or 다음       └─────────────▶ 처리 or 끝
```

핵심 역할
- **Handler**: `handle(req)`와 `setNext(h)` 인터페이스를 가진다.
- **ConcreteHandler**: 조건을 만족하면 처리, 아니면 다음으로 전달한다.
- **Client**: 첫 핸들러에 요청을 보낸다(연쇄의 시작점).

---

## 기본 구현 — 불리언 처리 모델 (Python)

처리 시 **True(처리됨)** / **False(넘김)** 을 반환해 흐름을 제어하는 가장 단순한 형태다.

```python
from abc import ABC, abstractmethod
from typing import Optional

class Handler(ABC):
    def __init__(self) -> None:
        self._next: Optional["Handler"] = None

    def set_next(self, nxt: "Handler") -> "Handler":
        self._next = nxt
        return nxt  # 체이닝 편의

    def handle(self, req) -> bool:
        if self._process(req):
            return True
        if self._next:
            return self._next.handle(req)
        return False  # 연쇄 끝까지 미처리

    @abstractmethod
    def _process(self, req) -> bool:
        ...

class LowLevel(Handler):
    def _process(self, req) -> bool:
        if req["level"] < 10:
            print(f"LowLevel 처리: {req}")
            return True
        return False

class MidLevel(Handler):
    def _process(self, req) -> bool:
        if 10 <= req["level"] < 50:
            print(f"MidLevel 처리: {req}")
            return True
        return False

class HighLevel(Handler):
    def _process(self, req) -> bool:
        if req["level"] >= 50:
            print(f"HighLevel 처리: {req}")
            return True
        return False

# 구성

h1 = LowLevel(); h2 = MidLevel(); h3 = HighLevel()
h1.set_next(h2).set_next(h3)

for r in [{"level": 5}, {"level": 15}, {"level": 55}, {"level": -1}]:
    handled = h1.handle(r)
    if not handled:
        print(f"미처리: {r}")
```

특징
- 간결하지만, **응답 값**(데이터)을 전달하긴 어렵다.
- 미처리 시 `False`로 누락을 감지하고 별도 기본 처리자를 둘 수 있다.

---

## 응답 객체 모델 — 처리 결과/데이터 반환

핸들러가 **응답 객체**를 만들어 반환하고, 미처리 시 `None`을 반환해 다음으로 넘긴다.

```python
from typing import Optional, Any, Dict

class Result:
    def __init__(self, handled: bool, data: Optional[Dict[str, Any]] = None):
        self.handled = handled
        self.data = data or {}

class Handler2(ABC):
    def __init__(self) -> None:
        self._next: Optional["Handler2"] = None
    def set_next(self, nxt: "Handler2") -> "Handler2":
        self._next = nxt; return nxt
    def handle(self, req) -> Optional[Result]:
        res = self._process(req)
        if res and res.handled:
            return res
        return self._next.handle(req) if self._next else None
    @abstractmethod
    def _process(self, req) -> Optional[Result]: ...

class CacheLookup(Handler2):
    def _process(self, req) -> Optional[Result]:
        key = req.get("key")
        cache = {"k1": "v1"}
        if key in cache:
            return Result(True, {"source": "cache", "value": cache[key]})
        return None  # 다음으로

class DatabaseLookup(Handler2):
    def _process(self, req) -> Optional[Result]:
        db = {"k2": "v2"}
        key = req.get("key")
        if key in db:
            return Result(True, {"source": "db", "value": db[key]})
        return None

class FallbackConst(Handler2):
    def _process(self, req) -> Optional[Result]:
        return Result(True, {"source": "default", "value": "N/A"})

c = CacheLookup(); d = DatabaseLookup(); f = FallbackConst()
c.set_next(d).set_next(f)

print(c.handle({"key": "k1"}).data)  # {'source': 'cache', 'value': 'v1'}
print(c.handle({"key": "k2"}).data)  # {'source': 'db', 'value': 'v2'}
print(c.handle({"key": "k3"}).data)  # {'source': 'default', 'value': 'N/A'}
```

장점
- 데이터 반환과 출처 추적이 쉬움.
- 마지막 기본 처리자(Fallback)로 **누락 방지**.

---

## 미들웨어 스타일 — `next()` 콜백, 여러 처리자 모두 실행(필터형)

웹 미들웨어처럼 **전처리 → 다음 호출 → 후처리** 형태를 지원한다. 모든 핸들러가 실행될 수 있다.

```python
from typing import Callable, List, Any

Middleware = Callable[[dict, Callable[[], Any]], Any]

def compose(middlewares: List[Middleware]) -> Callable[[dict], Any]:
    # 오른쪽에서 왼쪽으로 next를 감싸며 체인을 빌드
    def dispatch(ctx: dict) -> Any:
        def _run(i: int):
            if i == len(middlewares):
                return ctx  # 최종 응답(또는 실제 핸들러) 대체
            return middlewares[i](ctx, lambda: _run(i + 1))
        return _run(0)
    return dispatch

def auth(ctx, nxt):
    if not ctx.get("user"):
        ctx["status"] = 401; ctx["reason"] = "unauthorized"; return ctx
    return nxt()

def rate_limit(ctx, nxt):
    ctx["rate_checked"] = True
    return nxt()

def logger(ctx, nxt):
    ctx["log"] = ctx.get("log", []) + ["before"]
    res = nxt()
    ctx["log"].append("after")
    return res

pipeline = compose([logger, auth, rate_limit])
print(pipeline({"user": "kim", "path": "/api"}))   # 모든 단계 통과
print(pipeline({"path": "/api"}))                  # auth에서 차단
```

특징
- CoR(첫 처리자만 처리)와 달리 **여러 처리자가 순차 실행**될 수 있다.
- 전/후처리로 **트랜잭션, 타이밍, 로깅, 메트릭**을 쉽게 삽입.

---

## 동적/설정 기반 체인 구성 — 레지스트리 + 구성

환경/설정 파일로 체인을 바꾸면 배포 없이 정책을 전환할 수 있다.

```python
from typing import Dict, Type, List

class ReqHandler(ABC):
    def __init__(self, **kwargs) -> None: ...
    @abstractmethod
    def __call__(self, ctx: dict, nxt: Callable[[], Any]) -> Any: ...

_REG: Dict[str, Type[ReqHandler]] = {}

def register(name: str):
    def deco(cls):
        _REG[name] = cls; return cls
    return deco

@register("auth")
class AuthMW(ReqHandler):
    def __init__(self, required_role: str = "") -> None:
        self.required_role = required_role
    def __call__(self, ctx, nxt):
        roles = ctx.get("roles", [])
        if self.required_role and self.required_role not in roles:
            ctx["status"] = 403; return ctx
        return nxt()

@register("logger")
class LoggerMW(ReqHandler):
    def __call__(self, ctx, nxt):
        ctx.setdefault("logs", []).append(f"path={ctx.get('path')}")
        return nxt()

@register("cap")
class CapMW(ReqHandler):
    def __init__(self, cap: int = 1000) -> None: self.cap = cap
    def __call__(self, ctx, nxt):
        if ctx.get("amount", 0) > self.cap:
            ctx["status"] = 422; return ctx
        return nxt()

def build_pipeline(spec: List[dict]) -> Callable[[dict], Any]:
    mws: List[Middleware] = []
    for item in spec:
        t = item["type"]; params = item.get("params", {})
        cls = _REG[t]
        inst = cls(**params)
        mws.append(lambda ctx, nxt, inst=inst: inst(ctx, nxt))
    return compose(mws)

config = [
    {"type": "logger"},
    {"type": "auth", "params": {"required_role": "admin"}},
    {"type": "cap", "params": {"cap": 5000}},
]

pipeline = build_pipeline(config)
print(pipeline({"user": "kim", "roles": ["user", "admin"], "amount": 1200, "path": "/p"}))
print(pipeline({"user": "lee", "roles": ["user"], "amount": 100, "path": "/p"}))  # 403
```

---

## 분기형 체인 — 라우터/서브체인

요청 속성에 따라 **다른 체인**으로 라우팅한다.

```python
def router(routes: Dict[str, Callable[[dict], Any]]) -> Middleware:
    def mw(ctx, nxt):
        kind = ctx.get("kind", "default")
        pipe = routes.get(kind)
        if pipe:
            return pipe(ctx)
        return nxt()
    return mw

admin_chain = compose([lambda c, n: ({**c, "admin": True})])
user_chain  = compose([lambda c, n: ({**c, "user": True})])
root = compose([
    router({"admin": admin_chain, "user": user_chain}),
    lambda c, n: ({**c, "fallback": True})
])

print(root({"kind": "admin"}))  # admin 서브체인
print(root({"kind": "guest"}))  # fallback
```

---

## 에러 처리·복구·재시도·회로차단

에러를 **상부에서 가로채기** 쉬운 것도 미들웨어형 연쇄의 장점이다.

```python
import time

def retry(times: int = 3, backoff: float = 0.01) -> Middleware:
    def mw(ctx, nxt):
        last = None
        for i in range(times):
            try:
                return nxt()
            except Exception as e:
                last = e; time.sleep(backoff * (2 ** i))
        ctx["status"] = 500; ctx["error"] = str(last); return ctx
    return mw

def error_boundary(ctx, nxt):
    try:
        return nxt()
    except Exception as e:
        return {"status": 500, "error": f"caught: {e}"}

def may_fail(ctx, nxt):
    if ctx.get("fail"):
        raise RuntimeError("boom")
    return nxt()

p = compose([error_boundary, retry(2, 0.001), may_fail])
print(p({"fail": True}))
print(p({"fail": False, "ok": 1}))
```

회로 차단기(간단 스케치)
- 실패 카운트를 메모리에 유지, 임계치 초과 시 일정 시간 **바로 실패**(차단) 후 반열림(half-open)에서 탐침.

---

## 타임아웃/취소/데드라인 전파

요청마다 **데드라인(마감 시간)** 을 컨텍스트에 넣고, 각 핸들러가 이를 확인해 **시간 초과 시 조기 종료**한다.

```python
import time

def with_deadline(ctx, nxt):
    deadline = ctx.get("deadline")  # epoch seconds
    if deadline and time.time() > deadline:
        ctx["status"] = 504; ctx["error"] = "deadline exceeded"; return ctx
    return nxt()

def slow(ctx, nxt):
    time.sleep(ctx.get("sleep", 0))
    return nxt()

pipe = compose([with_deadline, slow, lambda c, n: ({**c, "done": True})])
print(pipe({"sleep": 0.05, "deadline": time.time() + 0.02}))  # 504
```

비동기(`asyncio`) 환경에서는 `async with timeout(...)` 혹은 `asyncio.wait_for`를 사용한다(별도 async 파이프라인 구현 필요).

---

## 멀티캐스트/집계 — 로깅·메트릭 등 부수효과 결합

CoR의 “첫 처리자만 처리”와 달리, **필터형 연쇄**에서는 여러 핸들러가 **부수효과(로그/메트릭/태깅)** 를 남길 수 있다.
필요하면 **집계 핸들러**에서 이전 단계가 쌓아둔 데이터(`ctx["events"]`)를 모아 최종 결론을 도출한다.

```python
def tap(name: str) -> Middleware:
    def mw(ctx, nxt):
        ctx.setdefault("events", []).append(f"enter:{name}")
        res = nxt()
        ctx["events"].append(f"exit:{name}")
        return res
    return mw

def aggregator(ctx, nxt):
    res = nxt()
    ctx["summary"] = {"count": len(ctx.get("events", []))}
    return res

pipe = compose([tap("a"), tap("b"), aggregator])
print(pipe({}))
```

---

## 동시성·스레드 안전·컨텍스트 전파

- **핸들러는 가급적 무상태(stateless)** 로 설계한다. 공유 상태가 필요하다면 락/원자성 보장.
- 컨텍스트(추적 ID 등)는 **불변 딕셔너리/DTO**로 넘기거나, 수정 시 복사해 사이드이펙트 최소화.
- 로깅 MDC/트레이싱 스팬은 **입구에서 생성→연쇄에 전파→출구에서 종료**.

스냅샷 호출 예(전략 교체와 유사)

```python
import threading
class ChainHolder:
    def __init__(self, pipeline): self._p = pipeline; self._lk = threading.RLock()
    def swap(self, pipeline):
        with self._lk: self._p = pipeline
    def handle(self, ctx):
        with self._lk: p = self._p
        return p(ctx)
```

---

## 성능·확률 모델(간단 수식)

핸들러가 순서대로 `H_1, H_2, ..., H_n` 이고,
- 각 핸들러의 평균 처리 시간을 \( t_i \),
- 요청이 \( i \)번째에서 처리될 확률을 \( p_i \) (단, \( \sum_i p_i \le 1 \), 나머지는 미처리)라고 하자.

**기대 지연시간 \( \mathbb{E}[T] \)** 의 근사:

$$
\mathbb{E}[T] \approx \sum_{i=1}^n \left( \prod_{k=1}^{i-1} (1 - p_k) \right) \cdot t_i
$$

설명: \( i \)번째까지 도달할 확률이 앞선 미처리 확률의 곱이고, 그 때 \( t_i \)가 더해진다.
필터형 연쇄(모두 실행)라면 단순 합 \( \sum_i t_i \) 가 된다(동기 직렬 기준).

튜닝 팁
- **가성비 높은 필터를 앞쪽에** 배치한다(빠르고 거를 수 있는 것부터).
- I/O는 비동기화/병렬화하고, **캐시/스킵** 규칙을 적극 사용한다.

---

## 언어별 스케치

### Java — 전통적인 CoR

```java
interface Handler {
    void setNext(Handler next);
    boolean handle(Request req); // 처리 시 true
}

abstract class BaseHandler implements Handler {
    private Handler next;
    public void setNext(Handler n) { this.next = n; }
    public boolean handle(Request req) {
        if (process(req)) return true;
        return next != null && next.handle(req);
    }
    protected abstract boolean process(Request req);
}

class AuthHandler extends BaseHandler {
    protected boolean process(Request req) {
        if (req.user != null) return false; // 다음
        System.out.println("401"); return true;
    }
}
```

### C# — DelegatingHandler 스타일(HTTP 파이프라인 유사)

```csharp
public abstract class Middleware {
    private Middleware _next;
    public Middleware SetNext(Middleware next) { _next = next; return next; }
    public virtual Task InvokeAsync(HttpContext ctx) =>
        _next != null ? _next.InvokeAsync(ctx) : Task.CompletedTask;
}

public sealed class Logging : Middleware {
    public override async Task InvokeAsync(HttpContext ctx) {
        Console.WriteLine("enter");
        await base.InvokeAsync(ctx);
        Console.WriteLine("exit");
    }
}
```

---

## 실제 시나리오 1 — API 요청 파이프라인(권한→레이트리밋→캐시→핸들러)

```python
def auth(ctx, nxt):
    if not ctx.get("user"):
        return {"status": 401}
    return nxt()

def rate(ctx, nxt):
    ctx["rate_ok"] = True
    return nxt()

def cache(ctx, nxt):
    key = ("GET", ctx.get("path"))
    store = ctx.setdefault("_cache", {})
    if key in store:
        return store[key]
    res = nxt()
    store[key] = res
    return res

def handler(ctx, nxt):
    return {"status": 200, "data": f"hello {ctx.get('user')}"}

pipeline = compose([auth, rate, cache, lambda c, n: handler(c, n)])
print(pipeline({"user":"kim","path":"/hello"}))
print(pipeline({"user":"kim","path":"/hello"}))  # 캐시 히트
print(pipeline({"path":"/hello"}))               # 401
```

---

## 실제 시나리오 2 — 이벤트 처리(첫 처리자만 처리)

```python
class Event:
    def __init__(self, kind, payload): self.kind = kind; self.payload = payload

class KindA(Handler):
    def _process(self, req: Event) -> bool:
        if req.kind == "a":
            print(f"A 처리: {req.payload}"); return True
        return False

class KindB(Handler):
    def _process(self, req: Event) -> bool:
        if req.kind == "b":
            print(f"B 처리: {req.payload}"); return True
        return False

a = KindA(); b = KindB(); a.set_next(b)
for ev in [Event("a", 1), Event("b", 2), Event("c", 3)]:
    handled = a.handle(ev)
    if not handled: print("미처리 이벤트:", ev.kind)
```

---

## 테스트 전략

### 순서/전파 검증(미들웨어형)

```python
def stamp(name):
    def mw(ctx, nxt):
        ctx.setdefault("trace", []).append(f"enter:{name}")
        res = nxt()
        ctx["trace"].append(f"exit:{name}")
        return res
    return mw

p = compose([stamp("a"), stamp("b")])
ctx = {}
p(ctx)
assert ctx["trace"] == ["enter:a", "enter:b", "exit:b", "exit:a"]
```

### 미처리 감지(CoR형)

```python
h = LowLevel(); m = MidLevel(); h.set_next(m)
assert h.handle({"level": 3}) is True
assert h.handle({"level": 30}) is True
assert h.handle({"level": 300}) is False  # 누락 검출
```

### 장애·복구

```python
p = compose([retry(2, 0), may_fail])
assert p({"fail": True})["status"] == 500
assert "error" in p({"fail": True})
```

---

## 다른 패턴과의 비교

| 패턴 | 목적 | 유사점 | 차이점/주의 |
|------|------|--------|-------------|
| Chain of Responsibility | 요청을 순차적으로 위임 | 객체 연결 | 첫 처리자에서 끝날 수 있음(전형 CoR). 미들웨어형은 모두 실행 가능 |
| Command | 요청 캡슐화 | 요청/실행 추상화 | 큐잉, 취소, 로그에 초점 |
| Decorator | 기능을 동적으로 추가 | 여러 객체 감싸기 | 요청 처리 주체는 동일, 흐름은 단일 |
| Interceptor/Filter | 전후처리 삽입 | 미들웨어형과 유사 | 보안/서블릿/HTTP 필터에서 일반적 |
| Pipeline | 단계별 변환 | 구성/연쇄 | 각 단계가 항상 실행되는 변환 파이프 |

---

## 안티패턴·체크리스트

체크리스트
- [ ] 마지막 **기본 처리자**가 있는가(미처리 누락 방지)?
- [ ] **빠르고 거르는** 핸들러를 앞에 두었는가?
- [ ] 핸들러는 **무상태** 또는 동시성 안전한가?
- [ ] **오류 경계/로깅/트레이싱**이 입구/출구에 있는가?
- [ ] 구성으로 **동적 재구성**이 가능한가?
- [ ] 타임아웃/취소가 **연쇄 전체**에 전파되는가?

안티패턴
- 연쇄 길이 과도 → 지연 증가, 추적 난이도↑
- 핸들러에서 `next()` 호출 누락 → 요청이 **소리소문 없이 소멸**
- 예외를 **삼켜서** 상위로 신호가 안 올라감
- 상태ful 핸들러를 **여러 스레드**에서 공유

---

## 요약

- 책임 연쇄는 **요청자와 처리자 분리**로 확장성과 테스트 용이성을 높인다.
- **불리언/응답객체/미들웨어** 3가지 운용 모델을 이해하고 요구에 맞게 선택하라.
- 구성 파일/레지스트리로 **동적 체인**을 만들면 운영 유연성이 커진다.
- 에러/재시도/회로차단/타임아웃/추적/메트릭을 **연쇄 상단에서 일괄 관리**하라.
- 성능은 **앞단 필터의 효율**과 **I/O 비동기화**, **캐시**가 좌우한다.
