---
layout: post
title: Elixir - Elixir 매크로 (2)
date: 2025-11-23 16:25:23 +0900
category: Elixir
---
# Elixir 매크로

## 값 주입에 바인딩 사용하기 (bind_quoted)

### 왜 `bind_quoted:` 가 필요한가 (초안 포함)

`unquote/1`는 **그 자리에** 식을 집어넣는다. 문제는:

1) **중복 평가**: 매크로 본문이 구조상 `unquote(expr)`를 **여러 번** 사용하면 `expr`가 여러 번 평가될 수 있다.
2) **부작용·순서**: `expr`에 부작용이 있으면 의도치 않은 행동 발생.
3) **위생(hygiene) 의도 표현**: 호출부 식을 **한 번만 평가**하고 내부 변수로 **바인딩**하는 것이 더 안전하다.

`bind_quoted:`는 호출부 인자를 **한 번만 평가하여** 로컬 변수에 **바인딩**한 뒤, 그 변수를 `quote` 내부에서 사용하게 한다.

#### (A) 문제를 재현하는 나쁜 예 (초안 포함)

```elixir
defmodule Bad do
  defmacro twice(expr) do
    quote do
      # expr가 두 번 평가될 수 있다
      {unquote(expr), unquote(expr)}
    end
  end
end

require Bad

count = :counters.new(1, [:atomics])
:ok   = :counters.put(count, 1, 0)

inc = fn ->
  v = :counters.get(count, 1)
  :counters.put(count, 1, v + 1)
  v + 1
end

Bad.twice(inc.())
# 기대: {1, 1} 이지만 실제: {1, 2} (두 번 실행됨)

```

#### (B) `bind_quoted`로 해결 (초안 포함)

```elixir
defmodule Good do
  defmacro twice(expr) do
    quote bind_quoted: [v: expr] do
      {v, v}
    end
  end
end

require Good
Good.twice(inc.())
# => {3, 3}  (한 번만 평가)

```

- `bind_quoted: [v: expr]`는 **컴파일 타임**에 `expr`을 **한 번** 평가하고 결과를 `v`로 **고정**한다.
- `expr`에 부작용이 있어도 **한 번만** 발생한다.

---

### 22.4.1-추가) “컴파일 타임에 한 번 평가”의 정확한 의미

여기서 오해가 자주 생긴다.
`bind_quoted`는 “컴파일 타임에 expr을 실제로 실행해 값을 고정한다”라기보다:

1) 호출부에서 넘어온 AST를
2) **주입 코드 내에서 ‘한 번만 평가되도록’**
3) **임시 변수에 바인딩**하는 코드를 만든다.

즉 다음과 같은 AST를 만들어 주입하는 것에 가깝다.

```elixir
# 개념적 확장 결과

(fn ->
  v = (inc.())     # 호출부 expr 평가 "한 번"
  {v, v}
end).()
```

- “한 번만 평가”는 런타임 시점에서 보장되는 것이고,
- 컴파일 타임에는 “한 번만 평가되도록 **코드를 생성하는 것**”이다.

이 차이를 명확히 이해하면, bind_quoted가 왜 **겹평가와 순서 문제를 동시에 해결**하는지 직관이 더 또렷해진다.

---

### 22.4.1-추가) 중복 평가가 더 위험해지는 케이스 3가지

중복 평가는 단순히 “값이 두 번 계산된다”를 넘어서, 다음 상황에서 심각해진다.

1) **부작용이 있는 함수**
   - 파일 쓰기, 네트워크 전송, DB 업데이트, 카운터 증가 등
   - 두 번 실행되면 **데이터가 꼬이거나 비용이 2배**

2) **비결정적 함수**
   - `System.monotonic_time/0`, 난수 생성, 외부 API 응답
   - 두 번 실행되면 `{x, x}`를 기대했는데 `{x, y}`가 된다.

3) **비싼 계산**
   - 고비용 파싱/정규화, 큰 리스트/맵 생성
   - 중복 실행은 “성능 버그”로 직결.

실전에서는 “중복 평가가 가능한 구조” 자체를 매크로에서 제거해야 한다.
그 가장 간단하고 명확한 도구가 `bind_quoted`다.

---

### 여러 값·맵·리스트를 안전하게 주입하기 (초안 포함 + 보강)

복합 자료구조는 `Macro.escape/1`로 **리터럴화**한 뒤 바인딩하면 좋다.

```elixir
defmodule Embed do
  defmacro embed(map_ast) do
    # 호출부에서 넘어온 map_ast를 안전하게 코드로 박제
    quote bind_quoted: [m: Macro.escape(map_ast)] do
      m
    end
  end
end

require Embed
cfg = %{limit: 10, tags: ~w(a b)}
Embed.embed(cfg)  # 컴파일된 코드에 그대로 박힌다
```

#### 왜 “escape + bind_quoted” 조합이 좋은가

- `Macro.escape`는 큰 구조체/맵/리스트를 **AST 리터럴로 안전 변환**한다.
- `bind_quoted`는 그 리터럴 AST를 **한 번만 계산되도록 바인딩**한다.

예를 들어, 매크로가 내부에서 같은 상수를 여러 번 쓰면:

```elixir
defmacro use_cfg(cfg) do
  quote do
    do_a(unquote(cfg))
    do_b(unquote(cfg))
  end
end
```

cfg가 큰 맵이면, **매 호출부마다 큰 AST가 2번 주입**된다.
즉 코드 팽창이 두 배다.

반면

```elixir
defmacro use_cfg(cfg) do
  quote bind_quoted: [c: Macro.escape(cfg)] do
    do_a(c)
    do_b(c)
  end
end
```

- cfg AST는 **한 번만 주입/바인딩**
- 내부 참조는 `c`로 반복 사용
- 컴파일 시간, 빔 파일 크기, 로딩 비용이 눈에 띄게 줄어든다.

“큰 상수/테이블/패턴을 매크로로 박제할 때는
escape → bind_quoted → 내부 변수 재사용”을 기본형으로 삼아라.

---

### 22.4.2-추가) bind_quoted가 “위생 의도”를 드러내는 이유

매크로는 “값을 주입하려는 건지, 변수를 주입하려는 건지”가 혼동되기 쉽다.

```elixir
defmacro foo(x) do
  quote do
    unquote(x) + unquote(x)
  end
end
```

이 코드는
- `x`가 값인지 식인지 상관없이
- **식 자체를 2번 주입**한다.

그 결과:

- 값 주입이라면 문제 없을 수 있지만
- 식 주입이라면 중복 평가
- 변수 주입이라면 스코프/위생 혼동 가능

이런 “의도 불명확”이 매크로의 대표적 유지보수 리스크다.

반면:

```elixir
defmacro foo(x) do
  quote bind_quoted: [v: x] do
    v + v
  end
end
```

이렇게 쓰면 문장 자체가 말해 준다.

- “나는 호출부 식 `x`를 한 번 평가해서 값 `v`로 고정하고 쓰겠다.”
- 중복 평가 없음
- 스코프/위생 안정

즉 bind_quoted는 **기술적 해결 + 의도 표현의 문서화**다.

---

### `bind_quoted`와 위생의 조합 (초안 포함 + 보강)

호출부의 변수 **그 자체를** 쓰고 싶다면(위생 해제) `var!/1`이 필요하지만, 대부분의 경우는 호출부 식을 **값**으로 바인딩하는 편이 안전하다.

```elixir
defmodule Assign do
  # 호출부 변수의 "값"을 복사해 쓴다 (위생 유지)
  defmacro copy_into(name, value) do
    quote bind_quoted: [val: value] do
      # 단지 val을 사용
      {unquote(name), val}
    end
  end

  # 호출부 "변수" 자체를 갱신 (위험·명시적으로만)
  defmacro mutate!(name_ast, value) do
    quote bind_quoted: [n: name_ast, v: value] do
      var!(n) = v
      :ok
    end
  end
end
```

#### copy_into/2가 “위생 유지”인 이유

- `val`은 매크로 내부의 새 변수로 위생적으로 생성된다.
- 호출부 스코프와 충돌하지 않는다.
- 호출부 식은 값만 복사되어 사용된다.

#### mutate!/2는 왜 위험한가

- `var!(n)`으로 호출부 변수를 **직접 갱신**한다.
- 이게 필요한 DSL도 존재하지만,
  실제 코드에서는 다음 문제를 낳는다.

1) 호출부에서 변수 `n`의 정체를 추적하기 어려움
2) 매크로 확장 전엔 부작용이 안 보임
3) 테스트/디버깅 시 의도 파악 난이도 급상승

따라서 mutate류 매크로는:

- 반드시 명확한 이름(`mutate!`, `assign!`)
- 문서와 주석
- 사용 범위 제한

이 3종 세트를 함께 가져가야 한다.

---

### 성능 노트 — 코드 팽창 방지 (초안 포함 + 확장)

`bind_quoted:`로 **값을 고정**하면, 동일 값을 여러 곳에 주입하는 것보다 **AST 크기가 작아진다**(특히 큰 자료구조일 때).
컴파일 시간·바이트코드 크기 관점에서도 유리하다.

#### 간단한 모델

- 큰 상수 AST 크기 \(S\)
- 매크로 본문에서 그 상수를 `k`번 unquote
- 사용처 수 \(N\)

그냥 unquote 반복:
$$
\text{총 AST 크기} \approx N \cdot k \cdot S
$$

bind_quoted로 1번만 주입:
$$
\text{총 AST 크기} \approx N \cdot (1 \cdot S + \epsilon)
$$

즉 `k`가 커질수록 격차가 커지고, 특히 DSL/스키마 매크로 같은 곳에서 **컴파일 타임 체감 차이**가 크게 난다.

---

## 매크로의 스코프 분리

매크로는 호출부에 **코드를 주입**하므로, 호출자 스코프를 오염시키지 않도록 **설계상의 룰**을 가져야 한다.

### 호출자 스코프 오염 방지 규칙 (초안 포함)

1) **내부 변수는 새 이름**(위생 기본값) — 호출부의 변수와 **무관**
2) **호출자 변수 조작 금지** — 반드시 필요할 때만 `var!/1` 사용
3) **모듈·함수 참조는 절대경로/명시 모듈** — 주입 위치에 따라 다른 `import/alias` 영향을 받지 않도록

---

### 22.5.1-A) 모듈 참조를 명시적으로 고정하기 (초안 포함 + 보강)

매크로 본문에서 동일 모듈 함수를 참조해야 할 때, **호출자 alias/import 영향**을 피하려면 **절대 모듈**을 박는다.

```elixir
defmodule StableRef do
  defmacro call_local_fun(x) do
    m = __MODULE__
    quote do
      unquote(m).local_fun(unquote(x))
    end
  end

  def local_fun(x), do: {:ok, x}
end
```

- `unquote(__MODULE__)`로 **현재 모듈**을 고정해 주입.

#### 왜 이게 필요한가

호출부에서:

```elixir
defmodule Caller do
  alias StableRef, as: S
  import OtherModule  # local_fun/1을 가진 모듈이 import돼 있다고 가정

  require StableRef
  StableRef.call_local_fun(10)
end
```

만약 매크로가 `local_fun(x)`처럼 **상대 참조**로 주입했다면,
호출자의 import/alias에 의해 **전혀 다른 함수가 호출**될 수 있다.

따라서 “매크로 내부에서 특정 모듈을 반드시 호출해야 하는 경우”는
저렇게 절대 참조를 고정해야 한다.

---

### 22.5.1-B) `quote location: :keep` (초안 포함 + 실전 팁)

디버깅 시, 생성된 코드 에러/경고 위치가 **호출부 코드**로 찍히도록 유지할 수 있다.

```elixir
defmacro safe_if(cond, do: t, else: f) do
  quote location: :keep do
    case unquote(cond) do
      x when x in [false, nil] -> unquote(f)
      _ -> unquote(t)
    end
  end
end
```

- 매크로가 커질수록 location 유지가 **개발 경험을 좌우**한다.
- 특히 DSL에서 “부정확한 파일/라인”은 디버깅 지옥을 만든다.

---

### `__CALLER__`와 컴파일 환경 사용 (초안 포함 + 보강)

매크로는 호출 시점의 컴파일 환경 `__CALLER__ :: Macro.Env`를 가진다.
이를 활용해 **호출자 모듈에 종속된 이름 생성**, **환경 검사** 등을 할 수 있다.

```elixir
defmodule LocalNames do
  defmacro defcounter(name) do
    caller = __CALLER__.module
    inner = Module.concat(caller, :"#{name}_Counter")

    quote bind_quoted: [inner: inner, name: name] do
      defmodule inner do
        def start_link, do: Agent.start_link(fn -> 0 end, name: __MODULE__)
        def next, do: Agent.get_and_update(__MODULE__, &{&1, &1 + 1})
      end

      def unquote(:"start_#{name}")(), do: unquote(inner).start_link()
      def unquote(:"next_#{name}")(),  do: unquote(inner).next()
    end
  end
end

defmodule M do
  require LocalNames
  LocalNames.defcounter(:job)
end

M.start_job()
M.next_job()  # 0
M.next_job()  # 1
```

#### __CALLER__로 할 수 있는 것들(실전 목록)

1) **호출자 모듈명 기반 namespacing**
2) 호출자 파일/라인 기록(로그/경고)
3) 호출자 환경 검사
   - prod에서만 허용, test에서만 허용 등
4) 호출자 모듈 속성 읽기/쓰기(DSL 누적)
5) 호출자의 alias/import 상황에 맞춘 AST 생성(주의 깊게)

#### 위험 신호

- 호출자 모듈에 **새 모듈을 무분별하게 생성**하면 코드가 폭증한다.
- defcounter처럼 **구조가 반복되는 DSL**에서만 쓰는 편이 좋다.
- 단일 기능을 위해 모듈을 자동 생성하는 순간, 유지보수가 어렵고 빔이 비대해진다.

---

### `__using__/1` + `@before_compile`로 DSL 스코프 분리 (초안 포함)

DSL은 보통 `use`로 **구성 정의 단계**를 모으고, `@before_compile`에서 **최종 코드 생성**을 한다.
이때, DSL 내부 상태는 **모듈 속성**으로 누적한다(호출자 모듈 스코프에만 영향).

```elixir
defmodule TinyDSL do
  defmacro __using__(_opts) do
    quote do
      import TinyDSL
      Module.register_attribute(__MODULE__, :routes, accumulate: true)
      @before_compile TinyDSL
    end
  end

  defmacro get(path, {mod, fun}) do
    quote bind_quoted: [path: path, mod: mod, fun: fun] do
      @routes {:GET, path, {mod, fun}}
    end
  end

  defmacro __before_compile__(env) do
    routes = Module.get_attribute(env.module, :routes)

    quote bind_quoted: [routes: Macro.escape(routes)] do
      def __routes__, do: routes
    end
  end
end

defmodule App.Router do
  use TinyDSL
  get "/ping", {App.Health, :ping}
end

App.Router.__routes__() # => [{:GET, "/ping", {App.Health, :ping}}]
```

#### 왜 이 구조가 “스코프 분리의 정석”인가

- DSL을 쓰는 모듈(App.Router)은
  - `@routes`라는 속성만 **자기 모듈 내부**에 쌓는다.
- DSL 제공자(TinyDSL)는
  - 호출자 모듈의 내부 상태만 조작하고
  - 외부/전역 상태를 건드리지 않는다.
- 최종 코드 생성은 `before_compile`에서 **한 번만** 일어난다.

즉 “구성 누적 단계 vs 코드 생성 단계”를 분리함으로써
**호출자 스코프를 깔끔하게 유지**한다.

---

### 호출자 import에 영향받지 않기 (초안 포함 + 보강)

주입 코드가 `if`, `with`, `case` 등 **Kernel 특수양식**을 사용한다면 보통 안전하다.
하지만 일반 함수 호출은 **호출자의 `import`에 가려질 수** 있으므로 **절대 모듈 명시**를 습관화하자.

```elixir
quote do
  Kernel.if(unquote(cond), do: unquote(t), else: unquote(f))
  Kernel.|||(true, false)
end
```

#### 실전 규칙

- 매크로 내부 호출은 가능하면:
  - `Kernel.*`
  - `Elixir.*`
  - `__MODULE__`/특정 모듈 절대경로

로 고정한다.

이 규칙 하나만 지켜도
“호출자 import 때문에 매크로가 다른 의미로 변질되는 사고”를 거의 막는다.

---

## 코드 조각을 실행하는 다른 방법

매크로 외에도, **AST·문자열·템플릿·런타임 디스패치**를 실행하는 루트가 여럿 있다.
목적과 안전성을 기준으로 선택하자.

### `Code.eval_quoted/3` — AST 즉석 평가 (초안 포함 + 확장)

- 이미 **quoted AST**를 갖고 있고, 현재 VM에서 **즉석 실행**하고 싶을 때.

```elixir
ast = quote do: Enum.map([1,2,3], &(&1 * 2))
{result, _binding} = Code.eval_quoted(ast)
result  # => [2,4,6]
```

- 바인딩 주입:

```elixir
ast = quote do: x + y
{sum, _} = Code.eval_quoted(ast, [x: 10, y: 32])
sum # => 42
```

#### 안전성 심화

`eval_quoted`는 “완전한 코드 실행 권한”이다.
따라서 외부 입력(사용자 제공 AST 등)을 그대로 eval하면:

- 파일 삭제/쓰기
- 네트워크 접속
- 시스템 명령 실행(Port/OS 모듈)
- 무한 루프/메모리 폭주

모든 위험이 열린다.

**원칙**: 외부 입력을 eval해야 한다면
“허용할 AST만 화이트리스트로 통제한 후 eval”한다.

---

### `Code.compile_quoted/1` — 모듈 동적 컴파일/로드 (초안 포함 + 확장)

```elixir
mod_ast =
  quote do
    defmodule DynCalc do
      @moduledoc "동적 생성 모듈"
      def add(a, b), do: a + b
    end
  end

Code.compile_quoted(mod_ast)
DynCalc.add(1, 2)  # => 3
```

#### 언제 쓰나

- 코드 제너레이터(스키마, 라우터, 자동 API 등)가
  “AST를 만들고 모듈로 컴파일”해야 할 때.
- 컴파일 시점이 아니라 **런타임에 플러그인 로딩**이 필요한 경우.

#### 운영 주의

- 릴리즈 환경에서 동적 컴파일은
  - 보안 정책상 금지되거나
  - 핫 로딩으로 인해 메모리/코드 서버가 복잡해질 수 있다.
- 빈번한 동적 모듈 생성은
  - code server 테이블 증가
  - 빔 파일 누수
  - 디버깅 난이도 상승

을 유발한다.
필요한 경우라도 **주기를 제한**하고 **정리 전략**을 세워야 한다.

---

### `Code.string_to_quoted/2` + `Code.eval_string/3` (초안 포함 + 확장)

```elixir
{:ok, ast} = Code.string_to_quoted("for x <- [1,2,3], do: x*x")
{res, _} = Code.eval_quoted(ast)
res # => [1,4,9]
```

```elixir
{v, _} = Code.eval_string("Enum.reduce(1..5, 0, &+/2)")
v # => 15
```

#### 실전 안전장치 패턴

- 문자열 입력을 받는다
- AST로 파싱한다
- AST를 walk하며 연산자/모듈/함수 제한
- 통과한 AST만 eval한다

초안에 근거한 간단 화이트리스트 예:

```elixir
defmodule SafeEval do
  @allowed_ops [:+, :-, :*, :/]
  def allow?(ast) do
    Macro.prewalk(ast, true, fn
      {op, _m, [_, _]} = node, acc when op in @allowed_ops -> {node, acc}
      i, acc when is_number(i) -> {i, acc}
      node, _acc -> {node, false}
    end)
    |> elem(1)
  end

  def eval_math(str) do
    with {:ok, ast} <- Code.string_to_quoted(str),
         true <- allow?(ast) do
      {:ok, elem(Code.eval_quoted(ast), 0)}
    else
      _ -> {:error, :not_allowed}
    end
  end
end
```

현업에선 더 엄격해야 한다.

- 허용 모듈 목록
- 허용 함수 목록
- 리터럴/튜플/리스트 타입 제한
- 재귀·루프 제한(가드)
- 실행 타임아웃(별도 프로세스 + 모니터)

---

### EEx(템플릿) — 코드가 섞인 문자열을 제너레이터로 (초안 포함 + 확장)

```elixir
tpl = """
defmodule <%= mod %> do
  def hello, do: "hi <%= who %>"
end
"""

code = EEx.eval_string(tpl, assigns: [mod: "Gened", who: "Elixir"])
{:ok, ast} = Code.string_to_quoted(code)
Code.compile_quoted(ast)
Gened.hello() # => "hi Elixir"
```

#### EEx를 코드 생성에 쓰는 이유

- AST를 직접 만들기 어렵거나
- “텍스트 기반 스캐폴딩”이 편한 상황에서
- 빠르게 모듈/함수 뼈대를 만들 수 있다.

하지만 EEx도 결국 “코드 실행”이다.
템플릿에 외부 입력을 섞는다면 동일한 위험이 있다.

---

### 런타임 디스패치: `apply/3`, 캡처, :rpc (초안 취지 확장)

```elixir
mod = String.to_atom("List")
apply(mod, :flatten, [[1, [2, 3]]]) # => [1,2,3]
```

#### 장점

- 문자열/AST eval 없이도 **동적 호출**이 가능
- 성능/안전성이 eval보다 낫다
- 호출 가능한 범위를 **모듈/함수 이름 레벨에서 통제**하기 쉽다.

#### :rpc는 “분산 환경의 eval”이다

- `:rpc.call(node, M, f, args)`는
  본질적으로 “원격 실행 권한”이다.

실전 안전 규칙:

1) 쿠키/노드 ACL 철저
2) 관리 네트워크 분리
3) 허용 RPC 목록만 노출
4) 감사 로그/레이트 리밋

---

### 언제 “매크로 대신” 다른 루트를 쓰나 (초안 포함 + 정리)

- **정적 주입**(컴파일 타임)
  - 표현식/DSL/보일러 제거
  - 런타임 비용 0, 문서·경고·타입 유리
  → 매크로가 1순위

- **런타임 구성/플러그인/사용자 규칙 입력**
  - 런타임에 기능을 확장/교체해야 함
  - 매크로로는 불가능(컴파일 타임이라)
  → AST/문자열 eval/compile 루트 고려
  단, **화이트리스트 + 격리 + 타임아웃**이 필수 전제

---

## 최종 정리

- **bind_quoted는 매크로의 안전벨트다.**
  - 중복 평가/부작용/의도 불명확을 한번에 제거한다.
  - 큰 구조는 `Macro.escape`와 조합해 코드 팽창까지 줄인다.

- **스코프 분리는 매크로 설계의 생명선이다.**
  - 호출자 변수 조작은 금지, 정말 필요할 때만 `var!`.
  - 모듈 참조는 절대 경로로 고정.
  - DSL은 `__using__` + 모듈 속성 누적 + `@before_compile` 패턴을 따른다.

- **코드 실행 루트는 목적에 맞게, 안전을 전제로 선택한다.**
  - eval/compile/string/EEx/:rpc는 강력하지만 위험하다.
  - 외부 입력을 섞는 순간, 화이트리스트·격리·타임아웃이 없으면 설계 실패다.

다음 단계에서는 이 기반으로
“사용자 정의 DSL을 OTP 애플리케이션/슈퍼비전/릴리즈와 연결하는 실전 패턴”을 더 큰 예제로 확장할 수 있다.
