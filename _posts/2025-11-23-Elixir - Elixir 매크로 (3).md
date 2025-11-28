---
layout: post
title: Elixir - Elixir 매크로 (3)
date: 2025-11-23 17:25:23 +0900
category: Elixir
---
# Elixir 매크로와 연산자

## 매크로와 연산자

### 사용자 정의 연산자: “연산자 = 얇은 매크로” 패턴 (초안 포함)

Elixir는 **비알파벳 기호들의 조합**으로 된 **이항/단항 연산자**를 **매크로**로 정의할 수 있다. 핵심은 그저 **이름이 연산자인 매크로**일 뿐이라는 점이다.

#### `~>`: 함수형 “스레딩/파이프 with 변형” 연산자

```text
defmodule Op do
  @moduledoc "연산자 예제: left ~> fun or {mod, fun}"
  # 사용 전: require Op

  # (a) 오른쪽이 캡처/람다일 때: x ~> (&fun/1) == fun.(x)
  defmacro left ~> right do
    quote do
      case unquote(right) do
        fun when is_function(fun, 1) ->
          fun.(unquote(left))
        {m, f} ->
          apply(m, f, [unquote(left)])
        other ->
          raise ArgumentError,
            "right side of ~> must be 1-arity function or {Module, :fun}, got: #{inspect(other)}"
      end
    end
  end
end

require Op

1
|> Op.~>(&(&1 * 10))       # => 10
|> Op.~>({Enum, :to_list}) # => [10]
```

- **핵심**: 연산자 **본문 전체가 `quote`** 이며, **`unquote`로 호출부 AST**를 삽입한다.
- 오른쪽에 **람다** 또는 `{Module, :fun}`을 허용해 **표현력**을 높인다.

#### `<~>`: “Map-like transform” (키-값 변환 DSL)

```text
defmodule Mx do
  @moduledoc "map <~> {src_key, dst_key, transform}"
  defmacro map <~> {src, dst, fun} do
    quote do
      v = Map.fetch!(unquote(map), unquote(src))
      Map.put(unquote(map), unquote(dst), unquote(fun).(v))
    end
  end
end

require Mx
m = %{a: 1}
m2 = (m <~> {:a, :b, &(&1 + 41)})  # => %{a: 1, b: 42}
```

> **팁**: 실무에선 연산자가 남발되면 **가독성/검색성**이 떨어진다. **“정말 자주 쓰이는 보일러 제거”**에만 도입하고, 나머지는 명시적 함수로 남겨라.

---

### 연산자 정의 문법의 본질

Elixir에서 “연산자”는 파서가 특별 취급하는 **문자열 이름**일 뿐이고, 실제 정의는 **매크로/함수 정의와 동일한 규칙**을 따른다.

- 이항 연산자 매크로:
  ```text
  defmacro left <op> right do
    ...
  end
  ```

- 단항 연산자 매크로:
  ```text
  defmacro <op>(x) do
    ...
  end
  ```

여기서 `<op>`는 “연산자 토큰으로 허용되는 기호 조합”이어야 한다.

**연산자 토큰 규칙(실전 감각)**
- 주로 `+ - * / < > = ~ | & ^ ? ! @ # $` 등의 조합
- **공백 없이** 붙여서 정의
- 너무 긴 조합은 가독성 및 도구 지원이 떨어진다.

---

### “연산자 매크로는 자동 import되지 않는다”

대부분의 커스텀 연산자는 **특정 모듈에 정의**할 텐데, 이때:

- **사용하는 모듈에서 `require`가 필요**
- 연산자 토큰이 Kernel에 이미 정의되어 있으면 **충돌 가능**

실전 패턴:

```text
defmodule MyOps do
  defmacro left ~~>> right do
    quote do
      unquote(right).(unquote(left))
    end
  end
end

defmodule UseOps do
  require MyOps
  import MyOps  # 연산자만 자주 쓰면 import로 짧게

  def demo(x), do: x ~~>> (&(&1 + 1))
end
```

- `import MyOps`를 해도 **매크로 사용을 위해 require는 여전히 필요**하다.
- (좋은 습관) 연산자 모듈은 **작게 끊고** 이름이 충돌할 여지를 줄여라.

---

### 연산자 매크로도 `bind_quoted`가 기본

연산자는 “예쁘게 읽히는 얇은 매크로”지만, 매크로인 이상 **겹평가/부작용 위험은 동일**하다.
따라서 내부에서 인자를 여러 번 쓰면 반드시 `bind_quoted`로 보호한다.

```text
defmodule SafeOp do
  # x <+> y : 두 값을 모두 한 번씩만 평가한 후 더한다
  defmacro left <+> right do
    quote bind_quoted: [l: left, r: right] do
      l + r
    end
  end
end
```

---

### 실전 팁 (초안 포함)

사용자 연산자는 **어떤 우선순위 그룹으로 파싱되는지**가 중요하다.
정확한 규칙을 모두 암기할 필요는 없고, **두 가지 원칙**을 지키자.

1) **항상 괄호로 명시**: 애매하면 `(a ~> &f/1) + 1`처럼 괄호로 의도를 고정.
2) **AST로 확인**: `quote` 후 `Macro.to_string/1`으로 파싱 결과를 눈으로 본다.

```text
ast = quote(do: 1 + 2 ~> &(&1 * 3))
IO.puts Macro.to_string(ast)
# ~> (& &1 * 3) " 형태인지 확인하고, 필요하면 괄호 보강

```

> **실수 방지**: 새 연산자를 **파이프 `|>` 옆**에 둘 때는 반드시 괄호를 써서 평가 순서를 고정하라.

---

### Elixir 연산자 우선순위 “감각 지도”

Elixir는 연산자를 **우선순위 그룹**으로 분류한다. 커스텀 연산자도 **토큰 첫 글자/형태에 따라 어느 그룹에 들어갈지 결정**된다.
모든 그룹을 외우기보다, 실전에서 자주 헷갈리는 지점만 감각적으로 잡자.

대략적 우선순위 흐름(높음 → 낮음):

1) **단항**: `!`, `not`, `+x`, `-x`, `^`
2) **곱셈류**: `* / div rem`
3) **덧셈류**: `+ - ++ --`
4) **비교류**: `< > <= >= == != === !== =~`
5) **논리연산**: `and/or/not` (엄격), `&&/||/!` (느슨)
6) **파이프/매치/비트**: `|>`, `=`, `<<< >>> &&& |||`
7) **저우선 그룹 커스텀**: 보통 `~`, `|`, `<`, `>` 등 시작하는 연산자가 여기에 자주 묶인다.

**핵심 규율**
- 새로운 연산자가 **기존 연산자와 섞여 길게 이어질 때**는
  **“반드시 괄호 + quote 확인”**이 유일한 안전장치다.

---

### 결합성(Associativity)도 확인하라

연산자가 좌결합/우결합인지에 따라 AST가 달라진다.

```text
ast1 = quote(do: a <~> b <~> c)
IO.puts Macro.to_string(ast1)
```

- 의도한 결합이 아니면 **괄호로 그룹을 고정**해야 한다.

실전에서는 “연산자 체인”이 길어지는 순간 **가독성도 급격히 하락**하므로,
연산자 체인은 **2~3개 이상이면 함수로 빼는 편**이 안전하다.

---

### 단항 연산자도 매크로다 (초안 포함)

예: 커스텀 단항 “부정” 만들기(학습용)

```text
defmodule Uop do
  # !!x 같은 중복 부정 연습
  defmacro !~(x) do
    quote do
      case unquote(x) do
        false -> true
        nil   -> true
        _     -> false
      end
    end
  end
end

require Uop
Uop.!~(nil)   # true
Uop.!~(1)     # false
```

---

### 단항 연산자에서 자주 터지는 함정

1) **이항/단항 충돌**
   같은 토큰이 단항과 이항으로 파싱될 여지가 있을 때 AST가 혼란스러워질 수 있다.
   → 단항은 **되도록 독특한 토큰**을 쓰고, 사용부에 괄호를 강제하라.

2) **가독성 역전**
   단항 연산자는 짧아서 강력하지만, 의미가 불명확하면 코드가 암호가 된다.
   → 단항은 “연산 의미가 모든 팀원에게 즉시 읽히는 것”만 허용.

---

### — `defguard/defguardp` (초안 포함)

**가드 절에서 사용할 수 있는** 매크로를 만들려면 `defguard`를 쓴다.
가드가 허용하는 **제한된 표현만** 사용 가능해야 한다.

```text
defmodule G do
  import Kernel, except: [abs: 1]  # 예시로 가드 안전 abs 만들기
  defguard is_pos_int(x) when is_integer(x) and x > 0
  defguardp abs_lt(x, y) when (x >= 0 and x < y) or (x < 0 and -x < y)
end

defmodule UseG do
  import G
  def good(x) when is_pos_int(x), do: :ok
  def near0(x) when abs_lt(x, 10), do: :small
end
```

- **가드 전용 매크로**는 **연산자처럼** 읽히는 DSL을 만들 때 유용하다.
- 가드 내부는 **함수 호출/동적 로직**이 제한되므로, 불가하면 **평범한 `if/2`** 로 우회한다.

---

### defguard가 허용하는 것/허용하지 않는 것

가드에서 허용되는 것은 **컴파일러가 안전하다고 보장하는 제한된 함수/연산**뿐이다.

- 허용:
  - 타입 검사: `is_integer/1`, `is_binary/1` …
  - 비교/산술/불 연산
  - 일부 BIF(안전 내장)

- 불허:
  - 임의 모듈 함수 호출
  - `Process.*`, `IO.*`, 파일/네트워크
  - 상태 변화가 가능한 로직

그래서 defguard는:

- “**순수하고 결정적이며 빠른 조건식**”
- “패턴 매칭/함수 헤드에서 자주 반복되는 검증”

에만 쓰는 것이 맞다.

---

## 한걸음 더 깊이 (초안 포함)

### 매크로 “위생”과 `bind_quoted`/`var!`의 궁합 복습 (초안 포함)

- **기본**: `quote` 내부에서 만든 변수들은 **위생적으로 새 이름**으로 바뀐다(호출부와 충돌 X).
- **호출부 식을 한 번만 평가**하려면 `bind_quoted`.
- **호출부 변수 그 자체**를 건드릴 땐 `var!(name)`.

```text
defmodule DeepHyg do
  defmacro once(expr) do
    quote bind_quoted: [v: expr] do
      {v, v}
    end
  end

  defmacro overwrite!(name_ast, value) do
    quote bind_quoted: [n: name_ast, v: value] do
      var!(n) = v
    end
  end
end
```

> **규율**: `var!` 사용은 코드리뷰에서 **항상 경고등**. 증빙/주석을 남겨라.

---

### 매크로 확장 단계: `expand_once/2`, `expand/2` 테스트 기법 (초안 포함 + 확장)

매크로가 **어떻게 치환되는지**는 테스트에서 **확장 결과를 문자열로 비교**하면 좋다.

```text
defmodule ExpandTest do
  use ExUnit.Case

  test "my_if expands to case" do
    require MyIf
    ast = quote(do: MyIf.my_if(1 < 2, do: :t, else: :f))
    expanded = Macro.expand(ast, __ENV__)
    assert Macro.to_string(expanded) =~ "case"
  end
end
```

- 매크로의 **출력 안정성**을 스냅샷처럼 검증 가능.
- 다만 **줄/공백**은 달라질 수 있으므로 **정규식/포함 비교**가 실용적.

추가 팁:
- **여러 단계 매크로가 중첩**돼 있을 때
  `Macro.expand_once/2`로 “한 번만” 펼쳐가며 원인을 좁히면 디버깅이 쉬워진다.

---

### `__CALLER__`(Macro.Env)로 호출자 문맥 반영 (초안 포함 + 확장)

- 파일/라인, alias/import, 현재 모듈 등을 **컴파일 타임**에 읽을 수 있다.
- **호출자 상대 경로** 처리, 자동 네이밍 등에 활용.

```text
defmodule Banner do
  defmacro note(msg) do
    env = __CALLER__
    file = env.file |> Path.relative_to_cwd()
    line = env.line

    quote bind_quoted: [file: file, line: line, msg: msg] do
      IO.warn("#{file}:#{line} — #{msg}")
    end
  end
end

defmodule Demo do
  require Banner
  Banner.note("compile-time annotation")
end
```

- “생성 코드가 어디서 왔는지”를 호출부 위치로 고정할 때 매우 강력하다.
- 이 패턴은 커스텀 연산자/DSL에서도 **오류 메시지 품질을 압도적으로 올린다**.

---

### 코드 생성 파이프라인: `__using__/1` + 누적 속성 + `__before_compile__` (초안 포함)

DSL/프레임워크에서 **보일러플레이트를 모아서** 한 번에 생성한다.

```text
defmodule Routes do
  defmacro __using__(_opts) do
    quote do
      import Routes
      Module.register_attribute(__MODULE__, :routes, accumulate: true)
      @before_compile Routes
    end
  end

  defmacro get(path, modfun) do
    quote bind_quoted: [path: path, modfun: modfun] do
      @routes {:GET, path, modfun}
    end
  end

  defmacro __before_compile__(env) do
    routes = Module.get_attribute(env.module, :routes)
    quote bind_quoted: [routes: Macro.escape(routes)] do
      def __routes__, do: routes
    end
  end
end
```

- 이 패턴은 Phoenix/Ecto류 DSL에서 **정석**이다.
- 커스텀 연산자 DSL을 설계할 때도
  “연산자 호출로 구성만 누적 → before_compile에서 최종 코드 생성”
  구조가 유지보수성이 가장 높다.

---

### “매크로 vs 인라인 함수 vs 일반 함수” (초안 포함 + 보강)

- **매크로**: 런타임 비용 제거(주입). 단, 코드 팽창/컴파일시간 증가 가능.
- **인라인 함수**: 호출 오버헤드 감소, 매크로보다 안전(위생 문제 X).
- **일반 함수**: 가장 단순. 유지보수성 우선.

원칙:
1) 함수로 가능한가? → 함수
2) 성능/표현력 때문에 함수가 부족한가? → 매크로
3) 매크로가 커지기 시작하는가? → 핵심 로직은 함수로 빼고 매크로는 thin wrapper

---

### 수학적 직관: 매크로 주입의 이득/대가 (초안 유지 + 보강)

주어진 연산 \(f(x)\)를 \(N\)회 호출한다.
- 함수 호출 비용 \(c\), 실제 연산 비용 \(t\)일 때
$$
T_{\text{func}} \approx N \cdot (c + t)
$$
- 매크로 주입은 호출 비용이 0에 가까우면
$$
T_{\text{macro}} \approx N \cdot t
$$

하지만 매크로는 호출부마다 코드가 복제되므로
- 코드 크기 증가 → I-캐시 압박
- 컴파일 시간 증가
- 빔 파일 크기 증가

이 대가를 함께 지불한다. 따라서 **측정 기반으로만 선택**한다.

---

## 실전에서 연산자/매크로를 쓰는 패턴 6가지

> 아래는 “실무에서 실제로 가치가 있는 경우”만 추려서 정리.

### 연산자

“성공이면 다음 단계 실행, 실패면 그대로 통과”는 파이프라인에서 가장 흔하다.

```text
defmodule ResultPipe do
  @moduledoc "Result-aware pipe: {:ok, v} ~|> fun  ;  {:error, r}는 그대로"
  defmacro left ~|> right do
    quote bind_quoted: [l: left, r: right] do
      case l do
        {:ok, v} ->
          case r do
            f when is_function(f, 1) -> f.(v)
            {m, f} -> apply(m, f, [v])
          end
        {:error, _} = err -> err
        other ->
          raise ArgumentError, "~|> expects {:ok, v} | {:error, r}, got: #{inspect(other)}"
      end
    end
  end
end
```

- Ecto/Req/Finch 같은 “태그드 리턴” 파이프에서 아주 유용하다.
- 단, 팀에 Result 표준이 없으면 **오히려 혼란**이 될 수 있다.

---

### “옵션을 먹는” DSL 연산자

`<~>` 예시를 확장하면, “작은 보일러 제거”에 탁월하다.

```text
defmodule MapDSL do
  # map <~~> {:src, :dst}  => dst에 src값 복사
  defmacro map <~~> {src, dst} do
    quote bind_quoted: [m: map, s: src, d: dst] do
      v = Map.fetch!(m, s)
      Map.put(m, d, v)
    end
  end
end
```

- “키 이동/복사/변환”처럼 반복이 매우 잦은 도메인에서만 쓴다.

---

### Guard-friendly “문장처럼 읽히는” 연산자/가드

조건이 함수 헤드에 반복될 때 defguard가 빛난다.

```text
defmodule GuardDSL do
  defguard is_user_id(x) when is_integer(x) and x > 0 and x < 10_000_000
end

defmodule UserSvc do
  import GuardDSL
  def get_user(id) when is_user_id(id), do: {:ok, id}
  def get_user(_), do: {:error, :bad_id}
end
```

- “가드로 표현 가능한 규칙”을 선별하는 능력이 중요하다.

---

### “데이터 파이프를 더 읽기 좋게” 만드는 연산자

가끔은 도메인에서 반복되는 변환을 “문장 형태”로 읽히게 만들고 싶다.

```text
defmodule TransformOp do
  # x ~~> f : x를 f에 넣은 결과를 반환
  defmacro left ~~> right do
    quote bind_quoted: [l: left, r: right] do
      r.(l)
    end
  end
end
```

- 이건 사실상 `|>`의 변형이지만,
  특정 도메인(예: 데이터 정제)에서만 쓰는 어휘를 제공한다.

---

### 패턴 매칭을 “얇게 감싸는” 연산자

복잡한 case/with 패턴을 한 줄로 접어 표현할 수 있다.

```text
defmodule MatchOp do
  # value =~? pattern : 패턴 매치 성공 여부만 반환
  defmacro value =~? pattern do
    quote bind_quoted: [v: value, p: pattern] do
      case v do
        ^p -> true
        _ -> false
      end
    end
  end
end
```

- 하지만 이런 식의 연산자는 **이미 `match?/2`가 있다면 중복**이다.
- 표준 함수가 존재하면 **표준을 우선**한다.

---

### 성능/지연 측정을 끼워 넣는 연산자(관찰 전용)

성능 계측을 “표현식처럼” 읽히게 만들어, 코드의 소음을 줄인다.

```text
defmodule Timed do
  defmacro expr <@> label do
    quote bind_quoted: [e: expr, l: label] do
      t0 = System.monotonic_time()
      v  = e
      dt = System.convert_time_unit(System.monotonic_time() - t0, :native, :microsecond)
      IO.puts("[TIMED #{l}] #{dt}us")
      v
    end
  end
end

require Timed
(1..1_000_000 |> Enum.sum()) <@> "sum"
```

- 실전에서는 IO 대신 Telemetry 이벤트를 보내는 쪽이 바람직하다.
- 관찰용 연산자는 **프로덕션에서 on/off 가능한 형태**로 설계하라.

---

## 연산자/매크로 설계 체크리스트 (실전용)

- [ ] **표준 함수로 충분한가?** (중복 도구 만들지 말 것)
- [ ] 연산자 의미가 **문맥 없이 읽혀서 이해되는가?**
- [ ] `bind_quoted`로 **중복 평가가 원천 차단**돼 있는가?
- [ ] 우선순위/결합이 애매한가? → **괄호 + quote 검사**
- [ ] 팀원이 IDE 검색으로 **쉽게 추적할 수 있는 이름/모듈 구조**인가?
- [ ] 테스트에서 `Macro.expand` 결과를 검증하고 있는가?
- [ ] 연산자 체인이 길어지지 않도록 **사용 규칙**이 있는가?
- [ ] `var!` 같은 탈위생 도구가 쓰였다면 **왜 필요한지 문서화**돼 있는가?

---

## 결론

- **매크로로 연산자를 정의할 수 있다는 사실은 Elixir DSL의 핵심 무기**다.
- 하지만 연산자는 “보기 좋은 매크로”일 뿐이므로,
  **겹평가/위생/스코프/우선순위/코드 팽창** 문제를 똑같이 안고 간다.
- 따라서 **작고 자주 쓰이는 보일러 제거**에만 도입하고,
  `bind_quoted` + 괄호 + 확장 테스트로 안전망을 깔아야 한다.

이 기준을 지키면, 연산자는
**읽기 좋은 도메인 문장**이 되면서도 **운영 가능한 코드**가 된다.
