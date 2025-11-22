---
layout: post
title: Elixir - Elixir 매크로 (1)
date: 2025-11-23 15:25:23 +0900
category: Elixir
---
# Elixir 매크로

## 왜 “if를 매크로로 구현”해야 하는가

Elixir에서 `if`는 “언어에 박힌 키워드”가 아니다. `Kernel.if/2`라는 **매크로**다.
즉, 호출부에서:

```elixir
if cond do
  a
else
  b
end
```

를 쓰면 컴파일러는 **AST(코드의 내부 표현)** 를 Kernel의 `if/2` 매크로에 넘기고, 그 매크로가 **다른 AST로 바꿔치기**하여 최종 바이트코드를 만든다.

이걸 손으로 구현해 보면 다음을 한 번에 배운다.

- AST는 어떤 모양이며(내부 표현),
- `quote`로 AST를 만들고 `unquote`로 끼워 넣는 방식,
- 매크로 위생이 왜 필요한지,
- 매크로 확장/평가를 어떻게 검증하는지,
- “함수로도 가능한 일을 매크로로 하면 왜 위험/유용한지” 판단 기준.

---

## if 문 구현하기

### 기본 뼈대: `defmacro` + `do:`/`else:` 키워드 (초안 포함)

```elixir
defmodule MyIf do
  @moduledoc "우리만의 if(else 지원) — 학습용 구현"
  # 매크로를 쓰려면 호출하는 쪽에서 require 필요: require MyIf

  defmacro my_if(condition, do: then_block, else: else_block) do
    quote do
      case unquote(condition) do
        x when x in [false, nil] -> unquote(else_block)
        _ -> unquote(then_block)
      end
    end
  end

  # else 생략 오버로드
  defmacro my_if(condition, do: then_block) do
    quote do
      case unquote(condition) do
        x when x in [false, nil] -> nil
        _ -> unquote(then_block)
      end
    end
  end
end
```

#### 설명 포인트 (확장)

1) **매크로 인자는 값이 아니라 AST다**

- `condition`, `then_block`, `else_block`은 실제 실행 결과가 아니라 **코드 조각 그 자체**(quoted AST)로 들어온다.
- 즉 다음은 **컴파일 타임의 코드 변환**이다.

2) **`quote`는 “AST를 만드는 공장”이다**

- `quote do ... end`에서 적은 코드는 **즉시 실행되지 않는다.**
- “이 코드를 앞으로 실행할 AST로 만들어서 반환하겠다”는 선언이다.

3) **`unquote`는 “호출부 AST를 꽂는 핀”이다**

- `unquote(condition)` 자리에 호출부의 조건 AST가 삽입된다.
- `unquote(then_block)`/`unquote(else_block)`도 동일.

4) Elixir의 진리값 규칙을 `case`로 구현

- Elixir에서 거짓은 **오직 `false`와 `nil`뿐**이다.
- 그래서 `case`의 첫 번째 절에서 `[false, nil]`을 명시한다.

#### 사용 예 (초안 포함)

```elixir
require MyIf

MyIf.my_if 1 < 2, do: "T", else: "F"
# => "T"

MyIf.my_if :ok, do: :then
# => :then

MyIf.my_if nil, do: 10
# => nil

```

---

### “왜 하필 case인가?” — `if`의 실제 표준 확장

Kernel의 진짜 `if`도 본질적으로 `case`로 풀린다.
왜냐면 `case`는

- 패턴매칭 기반 분기
- 가드 지원
- 단일 값을 기준으로 깔끔한 분기

라는 OTP 스타일 분기 도구이기 때문이다.

즉 `if`는 “문법 설탕(syntax sugar)”이며, 매크로로 구현됐기 때문에 **언제나 실제 AST로는 case가 남는다.**

---

### else 생략 오버로드: 왜 두 개의 defmacro가 필요한가

초안처럼 else 생략을 지원하려면 오버로드가 필요하다.

- `my_if(condition, do: then, else: else)` 형태
- `my_if(condition, do: then)` 형태

이 두 가지는 AST 인자 구조가 다르다. 매크로는 **패턴 매칭으로 선택된다.**

실제로는:

```elixir
MyIf.my_if x > 0, do: :pos
```

이 호출은 AST 레벨에서 `opts` 키워드 리스트에 `:else`가 없으므로
**두 번째 defmacro로 디스패치**된다.

---

### 매크로의 “중복 평가” 위험

매크로는 코드가 “복사·주입”된다.
따라서 매크로 설계가 나쁘면 **호출부 코드가 여러 번 평가**될 수 있다.

예를 들어 다음 나쁜 if를 생각해보자.

```elixir
defmacro bad_if(cond, do: t, else: e) do
  quote do
    if unquote(cond) do
      unquote(t)
    else
      unquote(e)
    end
  end
end
```

겉보기엔 괜찮아 보이지만, `if` 자체가 또 매크로이므로
확장 과정에서 `cond`가 구조에 따라 **중복 주입/중복 평가**될 여지가 생긴다.

그래서 **조건/인자를 한 번만 평가**하려면 `bind_quoted`가 필요하다.

```elixir
defmacro safe_if(cond, do: t, else: e) do
  quote bind_quoted: [c: cond, tb: t, eb: e] do
    case c do
      x when x in [false, nil] -> eb
      _ -> tb
    end
  end
end
```

- `cond`가 어떤 함수 호출/부작용이 있어도 **c에 한 번만 바인딩**된다.
- 이것이 실무에서 `bind_quoted`를 쓰는 1순위 이유다.

---

### 위생(Hygiene)과 변수 캡처 문제 (초안 포함)

매크로는 호출부에 **코드를 주입**하므로, 내부에서 만든 변수명이 **호출부 변수**와 충돌할 수 있다.
Elixir 매크로는 기본적으로 **위생적**이어서, `quote` 내부에서 만든 변수는 **새 이름**으로 리네임된다(충돌 방지). 하지만 **호출부 변수**를 의도적으로 **재사용**하려면 `var!/1`을 써야 한다.

```elixir
defmodule HygieneDemo do
  defmacro shadow_example(do: body) do
    quote do
      x = 0          # 위생적으로 다른 이름으로 변환됨(호출부 x와 충돌 안 함)
      unquote(body)  # 호출부 코드가 들어옴
    end
  end

  defmacro mutate!(name, value) do
    # 호출부의 변수 name을 "그대로" 쓰고 싶을 때 var!/1
    quote do
      var!(unquote(name)) = unquote(value)
    end
  end
end

require HygieneDemo

x = 41
HygieneDemo.shadow_example do
  x = x + 1   # ← 호출부 x (42가 됨)
end
x
# => 42

HygieneDemo.mutate!(:y, 100)
y
# => 100

```

#### 위생이 실제로 무슨 일을 하나 — 스코프 관점 상세

- `quote` 내부에서 `x = 0`을 쓰면, 컴파일러는 이를 **고유 이름으로 치환**한다.
  호출부의 `x`와 동일해 보여도, 실제 AST에는 `x@1`, `x@2` 같은 **다른 변수로 존재**한다.

이를 그림으로 보면:

```
호출부:
x = 41
shadow_example do
  x = x + 1
end

확장 후(개념):
x = 41
(fn ->
  x@shadow = 0
  x = x + 1
end).()
```

- `x@shadow`는 호출부 `x`와 독립적이다.
- 그래서 안전하게 주입할 수 있다.

#### `var!`는 언제 쓰나

- 호출부 변수에 **강제로 쓰기**해야 할 때(매우 드뭄)
- DSL에서 “외부 스코프 변수에 값 주입”이 요구될 때

하지만 위험하다.

- 코드 리뷰에서 반드시 주석/근거가 있어야 한다.
- 가능하면 `var!` 대신 **함수 반환 + 패턴매칭으로 상태 전달**을 선호한다.

---

### `unquote_splicing`으로 가변 인자 주입 (초안 포함)

인자가 리스트 형태의 AST일 때 **시퀀스 자체**를 펼쳐 넣고 싶으면 `unquote_splicing`을 쓴다.

```elixir
defmodule MyIf.Debug do
  defmacro my_if_debug(cond, opts) do
    {do_block, else_block} = Keyword.pop(opts, :do)
    else_block = Keyword.get(opts, :else, nil)

    debug_lines = [
      quote(do: IO.puts(">>> entering my_if_debug")),
      quote(do: IO.inspect(unquote(cond), label: "cond"))
    ]

    quote do
      (fn ->
        unquote_splicing(debug_lines)
        case unquote(cond) do
          x when x in [false, nil] -> unquote(else_block)
          _ -> unquote(do_block)
        end
      end).()
    end
  end
end
```

#### 왜 splicing이 필요한가

`unquote(debug_lines)`만 쓰면 리스트 전체가 **하나의 AST 자리**에 들어간다.
그 결과는 의미적으로:

```elixir
(fn ->
  [IO.puts(...), IO.inspect(...)]   # 리스트 평가만 하고 끝
  case ...
end).()
```

가 되어버린다.

반면 `unquote_splicing(debug_lines)`는 리스트를 **문장 시퀀스로 펼쳐**서:

```elixir
(fn ->
  IO.puts(...)
  IO.inspect(...)
  case ...
end).()
```

로 들어간다.
즉, **“코드 조각 n개를 그대로 줄줄이 주입”**하는 데 쓰는 도구다.

---

## 매크로는 코드를 주입한다

### “주입”의 의미: 컴파일 타임 치환 (초안 포함)

매크로는 **함수가 아니다**. 호출부의 코드를 **다른 코드로 바꿔치기** 한다. 그래서:

- **비용**: 런타임 오버헤드가 없다(주입된 코드가 직접 실행).
- **한계**: **실행 결과**가 필요한 경우(런타임 값)에 매크로를 쓰면 오히려 복잡해진다.
- **위험**: **주입된 코드**가 호출부의 스코프에 **낄** 수 있다(위생/스코프 주의).

---

### 매크로로 로깅을 최적화하는 이유 (초안 포함)

```elixir
defmodule Logx do
  defmacro debug(msg_ast) do
    quote do
      if Application.get_env(:myapp, :debug?, false) do
        IO.puts("[DEBUG] " <> to_string(unquote(msg_ast)))
      else
        :ok
      end
    end
  end
end

# 사용

require Logx
Logx.debug("expensive: " <> Enum.join(for i <- 1..3, do: "#{i*i}"))
# debug?가 false면 Enum.join 자체가 실행되지 않는다.

```

#### 왜 함수로 하면 손해인가

만약 `debug/1`이 함수라면:

```elixir
def debug(msg) do
  if debug?(), do: IO.puts(msg)
end
```

호출부에서 `debug(expensive_expr())`를 하면
`expensive_expr()`는 **먼저 실행되어 값이 만들어지고**
그 다음 `debug/1`이 호출된다.

즉, debug가 꺼져 있어도 **비싼 계산을 이미 해버린다.**

매크로는

- 계산 AST를 **그대로 들고 있다가**
- 조건이 참일 때만 실행

하도록 구조를 바꿔버린다.

---

### `require` 규칙과 호출 위치 (초안 포함)

- 매크로를 **사용하는 모듈**에서 **`require ModuleName`** 필요.
- 컴파일러는 호출 시점에 **매크로 본문(AST)** 을 가져와야 한다.

```elixir
defmodule A do
  defmacro m, do: quote(do: :ok)
end

defmodule B do
  # require 없으면 컴파일 에러
  require A
  A.m()
end
```

#### 왜 import만으로는 부족한가

- `import A`는 **함수 호출을 간단히 하도록 이름을 스코프에 가져오는 것**이다.
- `require A`는 **매크로 본문을 컴파일 타임에 불러오라는 지시**다.
- 매크로는 **컴파일러가 실제로 몸체를 실행해 AST를 돌려받아야** 한다.
  그래서 require가 필수다.

---

### `quote/unquote` 안전 패턴 (초안 포함)

- **항상** `quote`로 AST를 만들고, **필요한 위치**에만 `unquote`.
- 외부 값을 AST로 “에스케이프”하고 싶다면 `Macro.escape/1`.

```elixir
defmodule Embed do
  defmacro embed_value(map) do
    quote do
      unquote(Macro.escape(map))
    end
  end
end

require Embed
Embed.embed_value(%{a: 1, b: [1, 2, 3]})
```

#### `Macro.escape/1`이 필요한 이유

`quote` 안에 그냥 `map`을 unquote 하면
그 값이 AST로 적절히 표현되지 못하는 경우가 있다.

- **리터럴로 박아도 되는 값인지**
- **변수/구조체/함수 캡처인지**

를 Elixir 컴파일러가 안전하게 판단하기 어렵기 때문이다.

`Macro.escape`는

- 런타임 값을
- “해당 값을 그대로 표현하는 AST”로 변환

해 준다.
그래서 “컴파일 타임 상수 테이블” 같은 데서 필수다.

---

### 코드 생성의 책임 경계: 절대 규칙 4가지 (초안 포함 + 심화)

1) **부작용 최소화**
   - `quote` 내부에서 IO/파일/네트워크 금지
   - 정말 필요하면 “왜 컴파일 타임에 해야 하는지” 문서화

2) **스코프 명확화**
   - 의도적 변수 재사용: `var!/1`
   - 겹평가 방지: `bind_quoted:`
   - 모호한 스코프는 사고의 1번 원인

3) **성능감각 유지**
   - 매크로는 호출부마다 **코드가 복제**된다.
   - 큰 코드를 수십 군데에 주입하면
     - 빔 크기 증가
     - 컴파일 시간 증가
     - I-cache 압박
     - 디버깅 난이도 상승
   - 반복적 큰 로직은 **함수로 뽑고**, 매크로는 얇게.

4) **테스트**
   - 가능하면 똑같은 기능을 **함수 버전으로 먼저 검증**
   - 매크로는 **문법 설탕/보일러 제거 레이어**로 유지
   - 매크로 확장 결과를 스냅샷처럼 테스트

`bind_quoted:` 예 (초안 포함):

```elixir
defmodule BindQuotedDemo do
  defmacro put3(map_ast, key_ast, val_ast) do
    quote bind_quoted: [m: map_ast, k: key_ast, v: val_ast] do
      Map.put(m, k, v)
    end
  end
end
```

---

## 내부 표현을 코드로서 다루기

Elixir의 내부 표현은 **quoted AST**다.

- 리터럴은 자기 자신이 AST다.
- 일반 호출은 `{:name, meta, args}` 꼴의 3-튜플이다.
- 특별한 구문은 special form으로 별도 구조를 갖는다.

---

### AST를 보는 법 — `quote`, `Macro.to_string/1` (초안 포함)

```elixir
ast = quote do
  fun(1, 2, a + b)
end

ast
# => {:fun, [context: Elixir, import: Kernel], [1, 2, {:+, [context: Elixir, import: Kernel], [:a, :b]}]}

Macro.to_string(ast)
# => "fun(1, 2, a + b)"

```

#### meta의 의미

`meta`는

- 파일/라인
- import 정보
- context(어느 모듈에서 왔는지)

같은 컴파일러 힌트다.
매크로를 만들 때는 이 값을 **그대로 전달하는 쪽이 디버깅에 유리**하다.

---

### AST 변환 — prewalk/postwalk (초안 포함 + 보강)

```elixir
defmodule Rewriter do
  def strict_equals(ast) do
    Macro.prewalk(ast, fn
      {op, m, [l, r]} when op == :== -> {:"===", m, [l, r]}
      node -> node
    end)
  end
end
```

#### prewalk vs postwalk

- `prewalk`
  - 노드에 먼저 적용 → 자식으로 내려감
  - “상위 구조를 보고 자식 변환을 통제”할 때 유리

- `postwalk`
  - 자식을 먼저 변환 → 상위로 돌아오며 적용
  - “자식 변환 결과를 한 번 더 정리”할 때 유리

---

### AST 확장 — `Macro.expand/2` (초안 포함)

```elixir
require MyIf
env = __ENV__

ast = quote(do: MyIf.my_if(1 < 2, do: :t, else: :f))
Macro.expand(ast, env) |> Macro.to_string()
# => "case 1 < 2 do x when x in [false, nil] -> :f; _ -> :t end"

```

`expand`는 다음을 풀어쓴다.

- alias
- import된 매크로/함수 이름
- require된 매크로 본문
- 일부 special form 정규화

즉, “**컴파일러가 보는 실제 코드**”를 확인할 수 있다.

---

### AST 평가/컴파일 — `Code.eval_quoted`, `Code.compile_quoted` (초안 포함)

```elixir
{result, _binding} =
  Code.eval_quoted(quote(do: 1 + 2))
result
# => 3

mod_ast =
  quote do
    defmodule Dyn do
      def hello(name), do: "hi " <> name
    end
  end

Code.compile_quoted(mod_ast)
Dyn.hello("elixir")
# => "hi elixir"

```

주의:

- 신뢰되지 않은 입력을 eval하면 **원격 코드 실행 취약점**이다.
- 실무에서 동적 컴파일이 필요하면:
  - 허용 모듈/연산자 화이트리스트
  - sandbox 노드
  - 타임아웃
  - 리소스 한도

를 반드시 동반해야 한다.

---

### 생성 코드에 메타데이터 심기 (초안 포함)

```elixir
defmodule Autodoc do
  defmacro defconst(name, value) do
    quote do
      @doc "컴파일 타임 상수 #{unquote(name)}"
      def unquote(name)(), do: unquote(value)
    end
  end
end

defmodule K do
  require Autodoc
  Autodoc.defconst(:answer, 42)
end
```

이런 방식으로

- docs
- types
- compile 옵션
- external_resource

를 주입하면, “생성된 코드도 일반 코드처럼” 툴링에서 취급된다.

---

### 패턴 주의 3종: 스코프·겹평가·코드 팽창 (초안 확장)

1) **스코프**
   - 호출부 스코프에 변수 주입 → 기본 위생 때문에 안전
   - 의도적 캡처가 필요하면 `var!`/`bind_quoted`로 의도를 드러내라

2) **겹평가**
   - `unquote(expr)`가 여러 위치에 들어가면 expr이 여러 번 실행될 수 있다
   - `bind_quoted:`로 한 번만 계산하도록 고정

3) **코드 팽창**
   - 매크로가 큰 로직을 주입하면, 사용처마다 코드가 복제된다
   - “N회 호출 비용 0”이라는 장점은
     “N회 코드 복제 비용”을 동반한다
   - 함수 vs 매크로는 **측정 기반**으로만 선택

---

### 응용: 컴파일 타임 assert (초안 포함)

```elixir
defmodule CAssert do
  defmacro cassert(boolean_ast, message \\ "compile-time assert failed") do
    expanded = Macro.expand(boolean_ast, __CALLER__)
    case safe_eval(expanded) do
      {:ok, true} ->
        quote(do: :ok)
      {:ok, false} ->
        IO.warn("CAssert: #{unquote(message)} at #{__CALLER__.file}:#{unquote(__CALLER__.line)}")
        quote(do: :ok)
      :unknown ->
        quote do
          if !unquote(boolean_ast), do: raise(unquote(message))
        end
    end
  end

  defp safe_eval(ast) do
    try do
      {val, _} = Code.eval_quoted(ast, [], file: __ENV__.file, line: __ENV__.line)
      cond do
        is_boolean(val) -> {:ok, val}
        true -> :unknown
      end
    rescue
      _ -> :unknown
    end
  end
end
```

이 패턴은

- 리터럴/단순 식은 컴파일 타임에 검증하고
- 런타임 의존 식은 런타임 검사로 내리는

**하이브리드 안전장치**에 쓰인다.

---

## 실전 레퍼토리: 자주 쓰는 매크로 패턴 7가지 (초안 포함)

1) 조건부 import/require/use
2) 로깅 가드 최적화
3) DSL(누적 속성 + before_compile)
4) 반복 함수 생성
5) with 확장
6) 성능 카운터/텔레메트리 주입
7) 큰 상수 테이블 박제

DSL 예시 (초안 포함):

```elixir
defmodule TinyDSL do
  defmacro __using__(_opts) do
    quote do
      import TinyDSL
      Module.register_attribute(__MODULE__, :routes, accumulate: true)
      @before_compile TinyDSL
    end
  end

  defmacro get(path, mod_fun) do
    quote do
      @routes {:GET, unquote(path), unquote(mod_fun)}
    end
  end

  defmacro __before_compile__(env) do
    routes = Module.get_attribute(env.module, :routes)
    quote do
      def __routes__, do: unquote(Macro.escape(routes))
    end
  end
end
```

---

## (수학/성능 직관) 함수 vs 매크로 오버헤드의 간단 모델 + Octave 실험

매크로는 호출 오버헤드를 제거하지만, 코드 복제 비용을 만든다.
이를 아주 단순화해서 보자.

- 함수 호출 비용을 \(c\)
- 실제 일 비용을 \(t\)
- 호출 횟수를 \(N\)

이라고 하면:

함수 버전 총비용:
$$
T_{\text{func}} \approx N(c + t)
$$

매크로로 인라인 주입했다고 가정하면 호출 비용이 사라져:
$$
T_{\text{macro}} \approx Nt
$$

순수 계산만 보면 “언제나 매크로가 이득처럼” 보이지만, 실제로는

- 코드 크기 증가
- I-cache miss 증가
- 컴파일 시간 증가

항이 추가로 붙는다. 이를 \(b(N)\)로 쓰면:

$$
T_{\text{macro-real}} \approx Nt + b(N)
$$

즉 \(b(N)\)이 커지는 순간 매크로가 손해가 된다.

아래는 Octave로 “호출 오버헤드 비율이 커질 때/작을 때”를 시각화하는 작은 실험이다.

```octave
% gnu octave: macro_vs_func.m
% 매우 단순한 모델 시각화
N = 1:1e6;

c = 50;   % 가짜 호출 비용(나노초라고 가정)
t = 200;  % 가짜 실제 연산 비용

T_func = N .* (c + t);
T_macro = N .* t;

% 코드 팽창 비용을 N에 비례하는 선형 항으로 가정(아주 거친 모델)
k = 0.02;           % 코드 팽창 계수
b = k .* N .* (c+t);
T_macro_real = T_macro + b;

figure;
plot(N, T_func, N, T_macro, N, T_macro_real);
xlabel("N (calls)");
ylabel("Relative time units");
legend("Function", "Macro ideal", "Macro real (with b(N))");
title("Function vs Macro cost model");
grid on;
```

- `Macro ideal`은 항상 빠르지만,
- `Macro real`은 \(b(N)\)이 커지면 함수보다 느려질 수 있음을 보여준다.

Octave 실험은 “진짜 런타임을 잰 것”이 아니라 **직관 모델**이다.
실제 선택은 항상 **벤치마크로 확인**하라.

---

## 마무리

- **22.1**: `if`를 직접 만들며 **매크로는 AST를 받아 case AST로 바꿔치기**한다는 사실을 코드로 확인했다.
- **22.2**: 매크로는 **컴파일 타임 주입**이다. 로깅 같은 “조건부 비싼 계산 제거”에서 특히 유용하지만, **부작용·스코프·코드 팽창**을 항상 경계해야 한다.
- **22.3**: AST를 **보고(quote/to_string) → 바꾸고(pre/postwalk) → 확장하고(expand) → 평가/컴파일(eval_quoted/compile_quoted)**하는 전체 파이프라인을 실습했다.
