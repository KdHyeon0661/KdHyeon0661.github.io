---
layout: post
title: Elixir - Elixir Behaviour와 use/__using__
date: 2025-11-23 18:25:23 +0900
category: Elixir
---
# Elixir Behaviour와 `use/__using__`

## 비헤이비어(behaviour)

비헤이비어는 **콜백 계약**이다. “이 인터페이스를 구현하려면 이런 함수/타입을 제공해야 한다”를 **컴파일 타임**에 강제한다.
Elixir에서 비헤이비어는 **@callback/@macrocallback/@optional_callbacks**로 정의하고, 구현 모듈은 **@behaviour Mod**로 선언한다.

### 기본 예제: `Storage` 계약 (초안 포함)

```elixir
defmodule Storage do
  @moduledoc """
  키-값 저장소 계약. 구현자는 put/get/delete를 제공해야 한다.
  """

  @type key :: term()
  @type value :: term()
  @callback put(key, value) :: :ok | {:error, term()}
  @callback get(key) :: {:ok, value} | :not_found | {:error, term()}
  @callback delete(key) :: :ok | {:error, term()}
  @optional_callbacks delete: 1
end
```

구현:

```elixir
defmodule MemStore do
  @behaviour Storage
  @table __MODULE__

  def start_link do
    {:ok, :ets.new(@table, [:set, :public, :named_table])}
  end

  @impl Storage
  def put(k, v) do
    true = :ets.insert(@table, {k, v})
    :ok
  end

  @impl Storage
  def get(k) do
    case :ets.lookup(@table, k) do
      [{^k, v}] -> {:ok, v}
      [] -> :not_found
    end
  end

  @impl Storage
  def delete(k) do
    true = :ets.delete(@table, k)
    :ok
  end
end
```

- **@impl Storage**: 구현 함수가 **어느 비헤이비어 콜백**의 구현인지 명시. 시그니처 불일치 시 **컴파일 경고**.
- **@optional_callbacks**: 선택 구현 허용.

사용:

```elixir
{:ok, _} = MemStore.start_link()
:ok = MemStore.put(:a, 1)
MemStore.get(:a)     # {:ok, 1}
MemStore.delete(:a)  # :ok
```

---

### 비헤이비어가 “해결하는 문제”를 정확히 보자

비헤이비어는 다음 영역에서 특히 강력하다.

1) **구현 교체 가능성(Polymorphism)**
   호출자는 “계약”만 보고, 구현은 마음대로 바꿀 수 있다.

2) **테스트 용이성**
   실제 외부 의존(네트워크/DB/결제/알림)을 **가짜 모듈로 치환**하기 쉽다.

3) **컴파일 타임 검증**
   구현 누락/시그니처 오류/타입 불일치가 **런타임이 아니라 컴파일 때** 드러난다.

4) **문서화/자기 설명성**
   “어떤 기능을 제공해야 하는지”가 코드에 명확히 남는다.

---

### 실전 패턴: **어댑터/포트 & 드라이버** (초안 포함)

“도메인 포트(비헤이비어)”에 대해 “어댑터(구현)”를 주입한다:

```elixir
defmodule Notifier do
  @callback send(String.t(), String.t()) :: :ok | {:error, term()}
end

defmodule EmailNotifier do
  @behaviour Notifier
  @impl true
  def send(to, msg) do
    # SMTP/SES 등
    :ok
  end
end

defmodule SmsNotifier do
  @behaviour Notifier
  @impl true
  def send(to, msg), do: :ok
end
```

서비스는 **구체 타입을 모른다**:

```elixir
defmodule Alert do
  @type notifier_mod :: module()
  def send_alert(mod, to, msg) when is_atom(mod) do
    mod.send(to, "[ALERT] " <> msg)
  end
end

Alert.send_alert(EmailNotifier, "a@b.com", "disk full")
```

→ 테스트에서 **가짜 구현**을 넣기 쉽고, 런타임에도 **구현 교체**가 간단.

---

### 콜백 정의의 세 가지 도구

#### `@callback`

가장 일반적인 함수 계약.

```elixir
@callback charge(map()) :: {:ok, map()} | {:error, term()}
```

#### `@macrocallback`

“구현자가 **매크로를 제공해야 하는 계약**”을 만들고 싶을 때.

```elixir
defmodule DSLContract do
  @macrocallback route(String.t(), term()) :: Macro.t()
end
```

- DSL 프레임워크(라우팅/스키마 정의 등)에서 사용 가능.
- 구현자는 `defmacro`로 해당 이름을 제공해야 한다.

#### `@optional_callbacks`

“있으면 쓰고, 없어도 되는” 선택 계약.

```elixir
@optional_callbacks delete: 1
```

- “캐시 구현은 delete를 제공하지만, 읽기 전용 구현은 제공 안 해도 됨” 같은 상황에서 유용.

---

### `@impl`의 역할을 더 깊게

`@impl`은 두 가지 가치를 갖는다.

1) **사람에게 명확성**
   “이 함수는 계약 구현이다”를 코드에서 즉시 알 수 있다.

2) **컴파일러 검증**
   - 콜백 이름/arity가 없는데 `@impl`을 붙이면 경고
   - 콜백 시그니처와 구현이 어긋나면 경고
   - 여러 behaviour를 구현할 때 어떤 콜백인지 구분

선호 패턴:

```elixir
@impl Storage
def get(k), do: ...
```

또는 behaviour가 하나뿐이면:

```elixir
@impl true
def get(k), do: ...
```

---

### Dialyzer와 계약 (초안 포함 + 확장)

타입 명세가 있으면 Dialyzer가 **콜백-구현** 불일치, **반환형 미스매치**를 잡아준다.
규모가 크면 **계약+타입**으로 콤비를 맞추자.

추가로 중요한 점:

- behaviour에서 타입을 정의하면, 구현이 그 타입 계약을 **자동으로 상속**받는다.
- 구현 타입이 더 구체적이어도 괜찮지만, **계약보다 느슨하면 위험**하다.

예:

```elixir
defmodule Storage do
  @type key :: term()
  @type value :: term()
  @callback get(key) :: {:ok, value} | :not_found
end

defmodule MyStore do
  @behaviour Storage
  @impl true
  def get(k) when is_atom(k), do: {:ok, k}
end
```

- 구현이 `is_atom(k)`로 더 좁혀도 계약 위반은 아니다.
- 반대로 계약이 `term()`인데 구현이 `integer()`만 받는다고 문서화 없이 좁히면 호출자가 혼란.

---

### Behaviour vs Protocol — 헷갈리는 지점 정리

| 구분 | Behaviour | Protocol |
|---|---|---|
| 목적 | **모듈 수준 계약** | **타입(데이터) 기반 다형성** |
| “누가 구현?” | 모듈이 구현 | 각 타입(struct)별 구현 |
| 디스패치 | 호출자가 모듈을 선택 | 런타임에 타입으로 자동 선택 |
| 주 용도 | 포트/어댑터, OTP 콜백 계약 | Enumerable, String.Chars 같은 타입별 동작 |
| 컴파일 타임 강제 | 강함 | 구현 누락 시 런타임 에러 |

**짧은 결론**
- “교체 가능한 모듈”을 원하면 behaviour
- “데이터 타입에 따른 자동 디스패치”가 필요하면 protocol

---

### OTP Behaviour와의 관계

Elixir/OTP에서 흔히 말하는 “OTP behaviour”는 BEAM이 제공하는 **특수 behaviour**들이다.

- `GenServer`
- `Supervisor`
- `GenStateMachine(:gen_statem)`
- `GenEvent`
- 등

이들은 모두 “콜백 계약”을 갖고 있고, Elixir에서는 다음처럼 구현한다.

```elixir
defmodule Counter do
  use GenServer

  @impl true
  def init(_), do: {:ok, 0}

  @impl true
  def handle_call(:get, _from, st), do: {:reply, st, st}

  @impl true
  def handle_cast(:inc, st), do: {:noreply, st + 1}
end
```

즉, OTP behaviour도 “behaviour의 거대 버전”이며,
Elixir의 `use GenServer`가 **behaviour + 보일러 주입**을 한 번에 해주는 셈이다.

---

### Behaviour 기반 테스트 패턴 3가지

#### “가짜 구현 모듈” 직접 만들기

가장 단순하고 외부 라이브러리 없이 가능.

```elixir
defmodule FakeNotifier do
  @behaviour Notifier
  @impl true
  def send(_to, _msg), do: :ok
end
```

#### 실행 시점 DI(모듈 주입)

서비스 함수에 “모듈”을 인자로 전달.

```elixir
def send_alert(notifier_mod, to, msg) do
  notifier_mod.send(to, msg)
end
```

#### 애플리케이션 환경으로 모듈 바꾸기

구성에서 구현 교체.

```elixir
# config

config :myapp, notifier: EmailNotifier

# code

def notifier(), do: Application.fetch_env!(:myapp, :notifier)
```

테스트에선:

```elixir
Application.put_env(:myapp, :notifier, FakeNotifier)
```

---

## _23.2 `use`와 `__using__/1`_

`use SomeModule` 은 `SomeModule.__using__(opts)` 매크로를 호출해, **공통 보일러**(import/alias/함수/콜백)를 **주입**한다.
Phoenix/Ecto가 이 패턴으로 대형 DSL을 구성한다.

### 최소 예제: 로거 DSL (초안 포함)

```elixir
defmodule MiniLogger do
  defmacro __using__(opts) do
    level = Keyword.get(opts, :level, :info)
    quote do
      import unquote(__MODULE__)      # log/2 주입
      @mini_logger_level unquote(level)
    end
  end

  defmacro log(msg, level \\ :info) do
    quote bind_quoted: [msg: msg, level: level] do
      if allowed?(@mini_logger_level, level) do
        IO.puts("[#{level}] #{msg}")
      end
    end
  end

  def allowed?(min, lv) do
    order = %{debug: 0, info: 1, warn: 2, error: 3}
    order[lv] >= order[min]
  end
end

defmodule MyApp.Component do
  use MiniLogger, level: :debug

  def run do
    log("hello", :debug)
    log("world") # :info
  end
end

MyApp.Component.run()
```

- `use` 시점에 **설정이 바운드**되고, 모듈 속성으로 **간단한 상태**를 둔다.
- `quote` 내부에서 `bind_quoted:`를 적극 사용해 **겹평가/위생** 문제를 피하자.

---

### `use`가 정확히 하는 일 (매크로 관점)

`use Mod, opts`는 컴파일 타임에 아래를 실행한다.

1) `require Mod`
2) `Mod.__using__(opts)` 호출
3) 반환된 AST를 호출 모듈에 주입

즉,

```elixir
use MiniLogger, level: :debug
```

는 사실상:

```elixir
require MiniLogger
MiniLogger.__using__(level: :debug)
```

를 컴파일 타임에 수행한 것과 같다.

---

### `__using__/1` 설계의 핵심 규칙

1) **바인딩은 bind_quoted로**
   옵션/인자를 여러 번 `unquote`하지 말고 한 번만 평가하게 한다.

2) **주입 범위는 최소화**
   import/alias를 남발하면 호출자 스코프가 오염된다.

3) **재정의 포인트는 명시**
   `defoverridable`로 “앱이 덮어써도 되는 부분”을 분리한다.

4) **문서/타입을 같이 주입**
   DSL이 커질수록 `@doc`, `@type`, `@spec`의 가치가 커진다.

---

### `@before_compile`로 **최종 생성물** 주입 (초안 포함)

여러 매크로 호출로 누적한 설정을 **마지막에** 한 번에 코드로 만든다.

```elixir
defmodule RouterDSL do
  defmacro __using__(_opts) do
    quote do
      import RouterDSL
      Module.register_attribute(__MODULE__, :routes, accumulate: true)
      @before_compile RouterDSL
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

defmodule Web.Router do
  use RouterDSL
  get "/health", {Web.Health, :ok}
end

Web.Router.__routes__()
# [{:GET,"/health",{Web.Health,:ok}}]

```

이 패턴에서 중요한 포인트:

- `Module.register_attribute/3`로 **호출자 모듈 내부에만** 상태를 누적
- `@before_compile`이 “마지막 한 번” 최종 코드 생성
- `Macro.escape/1`로 데이터 구조를 **안전하게 리터럴화**

---

### `defoverridable` — 기본 구현 + 재정의 허용 (초안 포함)

```elixir
defmodule BaseController do
  defmacro __using__(_opts) do
    quote do
      def handle(conn), do: {:ok, conn}  # 기본
      defoverridable handle: 1           # 재정의 허용
    end
  end
end

defmodule UsersController do
  use BaseController
  def handle(conn), do: {:ok, Map.put(conn, :user, :loaded)}
end
```

→ 프레임워크가 **합리적 디폴트**를 제공하고, 앱은 필요한 곳만 **덮어쓴다**.

---

### `use` + 비헤이비어를 함께 (초안 포함)

- 비헤이비어로 **계약**을, use로 **보일러**를 준다.

```elixir
defmodule PaymentGateway do
  @callback charge(map()) :: {:ok, map()} | {:error, term()}
end

defmodule Payment.Base do
  defmacro __using__(_opts) do
    quote do
      @behaviour PaymentGateway
      @impl true
      def charge(%{"amount" => a} = req) when is_number(a) do
        process(req)
      end
      defoverridable charge: 1

      # 구현자는 이 함수만 제공하면 된다.
      @callback process(map()) :: {:ok, map()} | {:error, term()}
    end
  end
end

defmodule Payment.Stripe do
  use Payment.Base
  @impl true
  def process(req), do: {:ok, Map.put(req, "id", "tx_123")}
end
```

→ 호출자는 `PaymentGateway` 계약만 본다. 구현은 `use Payment.Base` 로 쉽게 맞춘다.

이 패턴의 실무적 장점:

- **검증/로깅/측정 같은 공통 전처리**를 Base에 몰아 넣을 수 있다.
- 구현자는 “핵심 로직”(`process/1`)만 집중한다.
- 교체/테스트 시 behaviour로 강하게 고정된다.

---

## _23.3 종합 예제 — Behaviour + use 조합으로 “플러그인형 파이프라인” 만들기_

이번 섹션은 앞의 개념을 “한 번에 감각적으로” 익히는 종합 실전이다.

### 요구사항

1) “파이프라인 스텝”의 계약을 behaviour로 정의
2) 각 스텝은 `use Step`으로 기본 보일러를 받는다
3) 앱은 원하는 스텝 모듈들을 조합해 파이프라인을 만든다
4) 실패/성공 스텝을 표준 형태로 수거한다

### 계약 정의

```elixir
defmodule PipeStep do
  @type input :: term()
  @type output :: term()
  @callback run(input) :: {:ok, output} | {:error, term()}
end
```

### 제공

```elixir
defmodule StepBase do
  defmacro __using__(_opts) do
    quote do
      @behaviour PipeStep

      # 공통 래퍼: 스텝 실행 전후를 표준화
      @impl true
      def run(input) do
        case process(input) do
          {:ok, out} -> {:ok, out}
          {:error, r} -> {:error, r}
          other -> {:error, {:bad_return, other}}
        end
      end

      defoverridable run: 1

      @callback process(term()) :: {:ok, term()} | {:error, term()}
    end
  end
end
```

### 스텝 구현

```elixir
defmodule Step.Trim do
  use StepBase
  @impl true
  def process(s) when is_binary(s), do: {:ok, String.trim(s)}
  def process(_), do: {:error, :not_string}
end

defmodule Step.ToInt do
  use StepBase
  @impl true
  def process(s) do
    case Integer.parse(s) do
      {i, ""} -> {:ok, i}
      _ -> {:error, :not_int}
    end
  end
end

defmodule Step.AssertPos do
  use StepBase
  @impl true
  def process(i) when is_integer(i) and i > 0, do: {:ok, i}
  def process(_), do: {:error, :not_positive}
end
```

### 파이프라인 러너

```elixir
defmodule Pipeline do
  @type step_mod :: module()

  def run(steps, input) do
    Enum.reduce_while(steps, {:ok, input}, fn step, {:ok, acc} ->
      case step.run(acc) do
        {:ok, v} -> {:cont, {:ok, v}}
        {:error, r} -> {:halt, {:error, {step, r}}}
      end
    end)
  end
end

steps = [Step.Trim, Step.ToInt, Step.AssertPos]
Pipeline.run(steps, "  42  ")
# {:ok, 42}

Pipeline.run(steps, "  -7 ")
# {:error, {Step.AssertPos, :not_positive}}

```

**이 예제의 의미**

- behaviour로 **스텝 계약을 모듈 교체 가능하게** 만들었고
- use(Base)로 **표준 run/1 보일러**를 주입했고
- 스텝 구현자는 `process/1`만 제공하면 된다
- 파이프라인 러너는 **구현 세부를 알 필요가 없다**

---

## _23.4 Agent/Task/GenServer와 Behaviour/use의 연결_

Elixir에서 behaviour/use는 OTP 동시성 도구와 자주 결합된다.

### GenServer를 behaviour로 추상화하기

예: “캐시 서버 계약”과 “각 캐시 백엔드(ETS/Redis/메모리)” 교체

```elixir
defmodule CacheBackend do
  @callback init() :: term()
  @callback get(term(), term()) :: {:ok, term()} | :miss
  @callback put(term(), term(), term()) :: :ok
end

defmodule CacheServer do
  use GenServer

  def start_link(backend) do
    GenServer.start_link(__MODULE__, backend, name: __MODULE__)
  end

  @impl true
  def init(backend) do
    st = backend.init()
    {:ok, {backend, st}}
  end

  def get(k), do: GenServer.call(__MODULE__, {:get, k})
  def put(k, v), do: GenServer.cast(__MODULE__, {:put, k, v})

  @impl true
  def handle_call({:get, k}, _from, {b, st}) do
    case b.get(st, k) do
      {:ok, v} -> {:reply, {:ok, v}, {b, st}}
      :miss -> {:reply, :miss, {b, st}}
    end
  end

  @impl true
  def handle_cast({:put, k, v}, {b, st}) do
    :ok = b.put(st, k, v)
    {:noreply, {b, st}}
  end
end
```

백엔드 구현:

```elixir
defmodule ETSBackend do
  @behaviour CacheBackend
  @impl true
  def init do
    :ets.new(__MODULE__, [:set, :public, :named_table])
  end

  @impl true
  def get(tab, k) do
    case :ets.lookup(tab, k) do
      [{^k, v}] -> {:ok, v}
      [] -> :miss
    end
  end

  @impl true
  def put(tab, k, v) do
    true = :ets.insert(tab, {k, v})
    :ok
  end
end
```

→ 서버는 backend 계약만 보고 동작, 백엔드는 교체 가능.

---

## 마무리

- **behaviour**는 “모듈이 지켜야 하는 콜백 계약”이며,
  교체성/테스트성/컴파일 타임 검증/문서화를 동시에 해결한다.
- **use/__using__**은 “공통 보일러·DSL·디폴트 구현을 호출자 모듈에 주입”하는 표준 패턴이다.
  `bind_quoted`, 누적 속성, `@before_compile`, `defoverridable`를 조합하면 Phoenix/Ecto급 DSL도 만들 수 있다.
- 두 도구를 합치면 “계약은 behaviour, 보일러는 use”라는 깔끔한 경계가 생기고,
  Elixir의 대규모 시스템에서 가장 강력한 설계축이 된다.

