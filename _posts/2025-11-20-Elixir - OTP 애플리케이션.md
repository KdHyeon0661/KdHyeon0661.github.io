---
layout: post
title: Elixir - OTP 애플리케이션
date: 2025-11-20 22:25:23 +0900
category: Elixir
---
# OTP 애플리케이션

## 부모님 세대의 애플리케이션이 아니다

엘릭서에서 “OTP 애플리케이션”을 처음 들으면 많은 사람들이 이렇게 오해한다.

- “앱(app)”이라니까, 윈도우/모바일 GUI 프로그램 같은 거겠지?
- 실행 파일 하나 딱 있고, main 함수에서 뭔가 돌고 있겠지?
- 시작/종료는 우리가 직접 “프로그램 실행/종료”로 관리하겠지?

하지만 OTP 애플리케이션은 **전통적 의미의 애플리케이션과 철학부터 다르다.**
OTP에서 애플리케이션은 **운영과 조합을 위한 표준화된 기능 단위**다.

---

### 용어를 다시 정의하자

- 전통적 의미의 “애플리케이션”
  - GUI 혹은 CLI로 시작되는 **단일 실행 파일**
  - `main()` 같은 진입점
  - 하나의 메인 루프/스레드가 주도
  - “프로그램”의 실행 단위 = “애플리케이션”

- **OTP 애플리케이션**
  - 하나의 **기능적 범위**를 가진 **독립적 단위**
  - 같은 VM 안에서 **여러 개가 동시에 올라간다**
  - “프로그램”과 “라이브러리” 사이의 중간 개념이 아니라,
    **라이브러리 + 운영 단위(프로세스 트리) = 애플리케이션** 이다.

OTP 애플리케이션이 되려면 최소한 아래가 갖춰져야 한다.

1) **자기완결적 슈퍼비전 트리**(필수)
   - 애플리케이션의 기능은 **프로세스들의 트리 구조(슈퍼비전 트리)** 로 구현된다.
   - 이 트리는 “앱 내부의 장애를 앱 내부에서 수습하는 구조”를 보장해야 한다.

2) **라이프사이클 콜백**
   - `Application.start/2`
   - `Application.stop/1`
   - OTP는 앱을 시작/정지할 때 표준화된 규칙으로 이 콜백을 호출한다.

3) **명세(metadata)**
   - `.app` 파일(얼랭 표준)에 “이 앱의 정체성”이 담긴다.
   - 의존성, 포함 모듈, 시작 모듈, 시작 순서 등이 기록된다.

결론적으로:

> OTP 애플리케이션은 **라이브러리이자 프로세스 트리**다.
> 여러 개의 OTP 애플리케이션이 **같은 VM** 위에서,
> 서로 의존 관계를 가지며, **시작 순서대로 올라가고 내려간다.**

---

### 구성 요소 한눈에 보기

OTP 애플리케이션을 구성하는 요소를 “코드”와 “운영 명세”로 나눠 보자.

- **코드**
  - 모듈(Function module)
  - GenServer / GenStateMachine / Agent / Task / GenStage 등 **워커**
  - Supervisor / DynamicSupervisor 등 **슈퍼바이저**
  - 필요에 따라 ETS, Registry, Telemetry, Logger 등

- **명세**
  - `.app` 파일(얼랭 표준, Mix가 생성)
  - “어떤 앱들이 선행되어야 하는가?”
  - “무엇이 시작 모듈인가?”
  - “어떤 모듈들로 구성되는가?”

- **애플리케이션 콜백**
  - `use Application`을 사용하는 “부트스트랩 모듈”
  - 여기서 **루트 슈퍼비전 트리**를 올린다.

- **환경(config)**
  - `config/*.exs`, `config/runtime.exs`
  - `Application.get_env/3`로 조회
  - 런타임 환경으로 바꿀 수 있어 **배포 유연성의 핵심**

- **릴리즈**
  - 런타임에 앱을 **의존 순서대로 올리는 배포 산출물**
  - `mix release`로 생성되는 자급자족 런타임

---

### 왜 이런 구조인가?

왜 OTP는 앱을 “프로그램”이 아니라 “운영 단위 기능 블록”으로 정의했을까?

핵심은 다음 세 가지다.

1) **격리·복구**
   - 앱은 슈퍼비전 트리 단위로 **장애 영역(failure domain)** 을 만든다.
   - 장애가 발생해도 **트리 내부에서 복구**한다.
   - 결과적으로 장애가 **국소화**되고 전체 VM이 무너지지 않는다.

2) **조합성**
   - “기능을 앱 단위로 묶으면”
     다른 앱과 **조합/해체/교체**가 쉬워진다.
   - 예를 들어:
     - 메시지 큐 앱
     - DB 커넥션 풀 앱
     - HTTP 서버 앱
     - 도메인 로직 앱
   - 이들을 같은 VM에서 **조립식으로 붙였다 떼는 구조**가 된다.

3) **배포 용이성**
   - `.app` 명세만 보면:
     - 어떤 앱이 먼저 떠야 하는지
     - 어떤 앱이 필수인지
     - 어디가 부트스트랩인지
     - 무엇이 포함되는지
     를 바로 알 수 있다.
   - 릴리즈는 이 명세를 기반으로 “정확한 시작/종료 순서”를 보장한다.

---

#### 가용도의 직관(단순화 모델)

시스템 가용도(Availability)를 흔히 이렇게 모델링한다.

$$
A=\frac{\text{MTBF}}{\text{MTBF}+\text{MTTR}}
$$

- MTBF: Mean Time Between Failures (평균 고장 간격)
- MTTR: Mean Time To Repair (평균 복구 시간)

OTP 애플리케이션은 **자동 복구(슈퍼비전)** 로 \( \text{MTTR} \)을 낮춰
결과적으로 \( A \)를 끌어올리는 구조다.

즉 OTP의 “앱 = 슈퍼비전 트리” 모델 자체가
**가용도를 위한 구조적 선택**이다.

---

## 애플리케이션 명세 파일

OTP 애플리케이션의 정체성은 **명세**에서 나온다.
그 명세가 바로 `.app` 파일이다.

---

### Mix와 `.app`

- Elixir 프로젝트를 컴파일하면:

```
./_build/<env>/lib/<app>/ebin/<app>.app
```

가 생성된다.

- `.app` 파일은 **Erlang term 파일**이다.
  - `application` 메타데이터가 담긴 표준 포맷
  - Elixir 개발자가 직접 쓰는 파일이 아니라
    **Mix가 `mix.exs`를 보고 생성한다.**

따라서 Elixir에서 `.app`을 이해하는 가장 좋은 방법은:

> **`mix.exs`의 `application/0`이 `.app`으로 어떻게 투사되는지**를 이해하는 것.

---

### `mix.exs`의 핵심: `project/0`와 `application/0`

초안의 코드를 그대로 두고, 포인트를 추가하자.

```elixir
defmodule Seq.MixProject do
  use Mix.Project

  def project do
    [
      app: :seq,
      version: "0.1.0",
      elixir: "~> 1.17",
      start_permanent: Mix.env() == :prod,
      deps: deps()
    ]
  end

  def application do
    [
      # 런타임에 자동으로 시작되는 애플리케이션들
      extra_applications: [:logger],
      # 애플리케이션 콜백 모듈 지정
      mod: {Seq.Application, []}
    ]
  end

  defp deps, do: []
end
```

- `project/0`
  - 빌드/컴파일/배포에 필요한 정체성
  - `app`은 애플리케이션 “이름(원자)”
  - `version`은 `.app`의 vsn 필드가 된다.

- `application/0`
  - OTP 애플리케이션 관점의 “운영 명세”
  - 런타임 부팅/종료와 의존을 정의

`application/0`에서 중요한 것은 두 필드다.

1) `extra_applications`
   - **런타임에 함께 먼저 올라가야 하는 표준 앱**
   - 예: `:logger`, `:crypto`, `:ssl`, `:runtime_tools` 등
   - “내 앱이 실행되기 전에 VM에 반드시 떠 있어야 하는 시스템 기능”을 의미

2) `mod: {Module, args}`
   - **부트스트랩 모듈**
   - OTP가 `Application.start/2`를 호출할 때,
     이 모듈의 `start/2` 콜백을 실행한다.
   - 여기서 **루트 슈퍼바이저**를 띄운다.
   - args는 start 콜백으로 전달되는 초기 인자.

---

### `.app`에 들어가는 필드(개념적)

`.app`은 얼랭 term 문법에서 다음 형태를 가진다.

- `{application, Name, Props}`

`Props`에 대표적으로 들어가는 항목은 다음과 같다.

- **`vsn`**
  - 앱 버전 문자열
- **`modules`**
  - 포함된 모듈 목록(컴파일 결과 기반)
- **`applications`**
  - **런타임 의존 애플리케이션**
  - 이 목록의 앱이 **먼저 떠 있어야** 현재 앱이 시작된다.
- **`included_applications`**
  - 배포물에는 포함되지만, **자동 시작은 안 하는 앱**
  - 보통 상위 애플리케이션이 직접 start/stop을 관리할 때 쓴다.
- **`mod`**
  - `{CallbackModule, StartArgs}`
- **`env`**
  - 초기 환경 키/값 목록
- **`registered`**
  - 전역 등록 프로세스(얼랭 스타일)

Elixir에서:

- `applications` 목록은 Mix가 **deps와 extra_applications를 보고 추론**
- `extra_applications`와 `mod`는 개발자가 직접 명시

즉 “`.app`을 제어한다”는 말은 실질적으로

> **`mix.exs`의 `application/0`을 제어한다**는 것과 같다.

---

### 애플리케이션 라이프사이클

OTP 애플리케이션은 OTP 런타임(Application Controller) 아래에서
**표준 순서로 시작되고, 표준 순서로 내려간다.**

#### 시작 흐름

1. 릴리즈 혹은 `iex -S mix`가 VM을 부팅
2. VM은 `.app`의 `applications` 의존을 토대로
3. **의존 앱부터** 순서대로 `Application.start/2` 실행
4. 마지막에 현재 앱의 `mod` 콜백(`Seq.Application.start/2` 같은 것)을 호출
5. 그 안에서 루트 슈퍼비전 트리를 올림
6. 트리 아래 워커들이 start_link 되어 기능이 활성화됨

#### 정지 흐름

1. VM이 종료되거나 앱이 stop되면
2. `Application.stop/1` 호출
3. 현재 앱의 `stop/1` 콜백 실행
4. 슈퍼비전 트리가 내려가며 워커들이 종료
5. 역순으로 의존 앱도 내려감

#### start phases(심화)

Erlang OTP에는 “start phases” 라는 기능이 있다.

- 앱 시작을 여러 단계로 나누고 싶을 때 사용
- Elixir에서도 사용 가능하지만 흔치 않다.

Elixir에선 보통 더 현대적인 방법을 쓴다.

- **루트 슈퍼바이저를 먼저 올리고**
- 각 워커에서 `handle_continue/2`로 후속 초기화를 수행

장점:

- 초기화 순서가 프로세스 트리 안에서 자연스럽게 정리된다.
- start phases는 앱 레벨에서 순서를 강제하므로, 복잡해지기 쉽다.

---

### 환경 설정과 조회

OTP 애플리케이션의 진짜 운영력은
“**환경을 런타임에서 바꿀 수 있다**”에 있다.

- 정적 설정: `config/config.exs`
- 런타임 설정: `config/runtime.exs`
  - 릴리즈 환경에서 특히 중요
  - ENV/시크릿 매니저 값으로 주입 가능

조회 예시:

```elixir
timeout = Application.get_env(:seq, :timeout, 5_000)
```

설정 예시:

```elixir
# runtime.exs

import Config
config :seq, timeout: 7_500
```

운영 원칙:

> **runtime.exs + ENV**로 설정을 외부화하면
> **이미지 재빌드 없이 설정 변경**이 가능해진다.

---

## 수열 프로그램을 OTP 애플리케이션으로 바꾸기

이제 “지금까지의 정의가 실제 코드로 어떻게 드러나는지”를
단계별로 보자.

---

### 출발점 — “수열 계산기”의 순진한 형태

먼저 초안의 출발점을 그대로 두자.

```elixir
defmodule Seq.Naive do
  # n번째 피보나치 (비효율적)
  def fib(0), do: 0
  def fib(1), do: 1
  def fib(n) when n > 1, do: fib(n-1) + fib(n-2)

  # 등차수열: a_n = a1 + (n-1)d
  def arith(a1, d, n), do: a1 + (n - 1) * d
end
```

이 코드는 “함수 모음”일 뿐이다.

- 상태가 없다.
- 동시성이 없다.
- 장애 복구가 없다.
- 운영/관찰/배포 관점이 없다.

즉 **OTP 애플리케이션의 조건을 하나도 충족하지 않는다.**

---

### 상태·동시성을 입히기 — GenServer(캐시/메모이제이션)

#### 퍼블릭 API와 전략(Behavior) 정의

수열 로직을 “전략 모듈”로 분리한다.
수열마다 상태/다음 항 계산 규칙이 달라지기 때문이다.

```elixir
defmodule Seq.Sequence do
  @callback init(opts :: keyword()) :: term()
  @callback next(state :: term()) :: {value :: term(), new_state :: term()}
  @callback peek(state :: term()) :: value :: term()
end
```

피보나치 수열 구현:

```elixir
defmodule Seq.Fib do
  @behaviour Seq.Sequence
  def init(_opts), do: {0, 1}          # (f(n), f(n+1))
  def next({a, b}), do: {a, {b, a+b}}
  def peek({a, _}), do: a
end
```

등차 수열 구현:

```elixir
defmodule Seq.Arith do
  @behaviour Seq.Sequence
  def init(opts), do: {Keyword.fetch!(opts, :a1), Keyword.fetch!(opts, :d)}
  def next({a, d}), do: {a, {a + d, d}}
  def peek({a, _}), do: a
end
```

핵심:

- 수열 로직은 상태 기반이므로
  **상태를 외부에 노출하지 않고**,
  `next/1`로만 진전한다.

---

#### 서버(GenServer) 구현

```elixir
defmodule Seq.Server do
  use GenServer

  # Public API
  def start_link(opts) do
    GenServer.start_link(__MODULE__, opts, name: Keyword.get(opts, :name))
  end

  def next(server \\ __MODULE__), do: GenServer.call(server, :next)
  def peek(server \\ __MODULE__), do: GenServer.call(server, :peek)

  @impl true
  def init(opts) do
    mod = Keyword.fetch!(opts, :sequence)
    state = mod.init(opts)
    {:ok, %{mod: mod, st: state}}
  end

  @impl true
  def handle_call(:next, _from, %{mod: m, st: st} = s) do
    {val, st2} = m.next(st)
    {:reply, val, %{s | st: st2}}
  end

  @impl true
  def handle_call(:peek, _from, %{mod: m, st: st} = s) do
    {:reply, m.peek(st), s}
  end
end
```

여기서 OTP 관점의 의미:

- 이제 수열은 “함수”가 아니라
  **상태를 갖고 동시 요청을 처리하는 서버 프로세스**가 되었다.
- 수열 로직을 교체하면 같은 서버 구조를 재사용할 수 있다.
- 이는 “기능 단위(애플리케이션)”가 될 수 있는 첫 단계다.

---

### 슈퍼바이저 + 애플리케이션 콜백 구성

#### 슈퍼바이저

수열 서버들을 앱 내부에서 관리할 슈퍼비전 트리를 만든다.

```elixir
defmodule Seq.Supervisor do
  use Supervisor
  def start_link(opts), do: Supervisor.start_link(__MODULE__, opts, name: __MODULE__)

  @impl true
  def init(_opts) do
    children = [
      {Seq.Server, name: Seq.FibServer, sequence: Seq.Fib},
      {Seq.Server, name: Seq.ArithServer, sequence: Seq.Arith, a1: 3, d: 5}
    ]

    Supervisor.init(children, strategy: :one_for_one)
  end
end
```

- `:one_for_one`
  - 특정 수열 서버가 죽으면 **그 서버만 재시작**
  - 장애가 다른 수열로 전파되지 않는다.

---

#### 애플리케이션 콜백 모듈

루트 트리를 실제로 부팅하는 Application 콜백 모듈을 만든다.

```elixir
defmodule Seq.Application do
  use Application

  @impl true
  def start(_type, _args) do
    children = [
      {Seq.Supervisor, []}
    ]

    opts = [strategy: :one_for_one, name: Seq.Root]
    Supervisor.start_link(children, opts)
  end

  @impl true
  def stop(_state) do
    :ok
  end
end
```

- 이 모듈이 `.app`의 `mod` 필드에 들어가면
- OTP가 “앱 시작 시” 자동으로 호출하고
- 그 결과 루트 슈퍼비전 트리가 올라간다.

즉 이 모듈이 **OTP 애플리케이션의 진입점**이다.

---

#### mix.exs 연결

```elixir
def application do
  [
    extra_applications: [:logger],
    mod: {Seq.Application, []}
  ]
end
```

이제 `seq`는 OTP 애플리케이션 요건을 충족한다.

- 자기완결적 슈퍼비전 트리 보유
- 라이프사이클 콜백 보유
- `.app` 명세로 의존/부트스트랩 정의됨

---

### 실제 사용 예(IEx)

```elixir
# iex -S mix

Seq.Server.peek(Seq.FibServer)  # 0
Seq.Server.next(Seq.FibServer)  # 0
Seq.Server.next(Seq.FibServer)  # 1
Seq.Server.next(Seq.FibServer)  # 1
Seq.Server.next(Seq.FibServer)  # 2

Seq.Server.peek(Seq.ArithServer) # 3
Seq.Server.next(Seq.ArithServer) # 3
Seq.Server.next(Seq.ArithServer) # 8
Seq.Server.next(Seq.ArithServer) # 13
```

중요한 점:

- 우리는 `Seq.FibServer`를 직접 start_link 하지 않았다.
- 앱 부팅 시점에 **OTP가 자동으로 올려줬다.**
- 이는 “프로그램 실행”과 완전히 다른 느낌이다.

---

### 환경 설정으로 파라미터 외부화

운영에서는 수열 파라미터를 코드로 박아 두지 않는다.

```elixir
# config/runtime.exs

import Config

config :seq, arith,
  a1: String.to_integer(System.get_env("SEQ_A1", "3")),
  d:  String.to_integer(System.get_env("SEQ_D",  "5"))
```

슈퍼바이저에서 이를 주입한다.

```elixir
defmodule Seq.Supervisor do
  use Supervisor

  @impl true
  def init(_opts) do
    {:ok, arith_cfg} = Application.fetch_env(:seq, :arith)

    children = [
      {Seq.Server, name: Seq.FibServer, sequence: Seq.Fib},
      {Seq.Server, Keyword.merge([name: Seq.ArithServer, sequence: Seq.Arith], arith_cfg)}
    ]

    Supervisor.init(children, strategy: :one_for_one)
  end
end
```

이로써:

- 코드 변경 없이
- 환경 변수만 바꿔
- 수열의 초기값/증분을 바꿀 수 있다.

이게 “OTP 애플리케이션이 운영 단위”라는 의미다.

---

### 관찰성 추가: Telemetry

운영 가능한 앱이라면, 관찰 가능한 앱이어야 한다.

```elixir
defmodule Seq.Server do
  use GenServer

  # ...

  @impl true
  def handle_call(:next, _from, %{mod: m, st: st} = s) do
    :telemetry.span([:seq, :next], %{}, fn ->
      {val, st2} = m.next(st)
      {{:reply, val, %{s | st: st2}}, %{value: val}}
    end)
  end
end
```

핸들러 예시:

```elixir
:telemetry.attach(
  "seq-logger",
  [:seq, :next, :stop],
  fn _event, meas, meta, _ ->
    require Logger
    Logger.debug("next took=#{meas.duration}ns value=#{meta.value}")
  end,
  nil
)
```

- 이제 각 `next/0` 호출이 얼마나 걸렸는지
- 어떤 값이 나왔는지를
- 런타임 로깅/모니터링으로 연결할 수 있다.

---

### 확장: 동적 수열 인스턴스 관리

여러 종류/여러 인스턴스의 수열을 런타임에 추가해야 한다면
`DynamicSupervisor`가 적합하다.

```elixir
defmodule Seq.Dynamic do
  use DynamicSupervisor

  def start_link(opts) do
    DynamicSupervisor.start_link(__MODULE__, :ok, opts)
  end

  @impl true
  def init(:ok) do
    DynamicSupervisor.init(strategy: :one_for_one)
  end

  def start_seq(name, mod, opts) do
    spec = {Seq.Server, Keyword.merge([name: name, sequence: mod], opts)}
    DynamicSupervisor.start_child(__MODULE__, spec)
  end
end
```

이제 앱이 올라간 뒤에도 수열을 계속 추가할 수 있다.

```elixir
Seq.Dynamic.start_seq(:geo1, Seq.Geo, a1: 2, r: 3)
Seq.Server.next(:geo1)
```

OTP 애플리케이션의 “조합성”이 이런 데서 빛난다.

---

### 테스트 전략

OTP 애플리케이션은 “프로세스 기반 기능”이므로
테스트도 프로세스 단위로 해야 한다.

```elixir
defmodule Seq.ServerTest do
  use ExUnit.Case, async: true

  setup do
    {:ok, _} = start_supervised({Seq.Server, sequence: Seq.Fib, name: Seq.FibTest})
    :ok
  end

  test "fib progresses" do
    assert 0 == Seq.Server.peek(Seq.FibTest)
    assert 0 == Seq.Server.next(Seq.FibTest)
    assert 1 == Seq.Server.next(Seq.FibTest)
    assert 1 == Seq.Server.next(Seq.FibTest)
  end
end
```

- `start_supervised/1`를 쓰면 테스트 종료 시 자동 정리.
- 앱 전체를 올리는 통합 테스트라면:

```elixir
Application.ensure_all_started(:seq)
```

같은 방식으로 앱 부팅/의존까지 검증할 수 있다.

---

## OTP 애플리케이션 실전 규율과 심화 포인트

초안에 없던, 하지만 “OTP 애플리케이션을 제대로 쓰려면 꼭 알아야 하는” 항목들을 정리한다.

---

### 런타임 의존 vs 컴파일 의존

- `deps`
  - 컴파일·빌드 시점의 의존

- `.app`의 `applications`
  - 런타임 시작 순서를 위한 의존

Elixir에서 대체로:

- 라이브러리성 앱은 `extra_applications`를 최소화
- “런타임 시스템 기능”만 선별적으로 포함

예:

```elixir
extra_applications: [:logger, :crypto, :ssl]
```

---

### `included_applications`의 의미

`included_applications`는 “내가 다른 앱을 포함하되, 자동 시작은 막는” 장치다.

- 상위 앱이 하위 앱의 start/stop을 직접 통제하려는 경우
- 대형 시스템에서 “앱 묶음”을 만들 때 사용

Elixir에서는 일반적인 서비스에선 드물지만,
Umbrella 프로젝트나 플랫폼형 시스템에선 등장할 수 있다.

---

### Application Controller의 역할

OTP가 앱을 시작/종료할 수 있는 이유는
VM 내부에 **Application Controller**가 있기 때문이다.

- `.app` 명세를 읽고
- 의존 순서를 정하며
- `Application.start/2`를 순차 호출
- 종료는 역순으로 처리

즉, “앱이 스스로 뜨고 내린다”는 느낌은
사실 “VM이 표준 규칙대로 뜨고 내린다”는 뜻이다.

---

### `start_permanent` / 재시작 강도

`project/0`의 `start_permanent: Mix.env() == :prod`는
VM이 앱 종료를 얼마나 심각하게 받아들일지 지정한다.

- permanent로 시작된 앱이 비정상 종료되면
  **VM 자체가 셧다운**할 수 있다.
- prod에서는 “앱 하나가 죽었다” = “서비스가 죽었다”로 취급해야 한다면 유효.

---

### 애플리케이션 업그레이드와 코드 체인지(심화)

Erlang/OTP는 원래 “무중단 업그레이드”를 위한 체계를 갖는다.

- 앱 버전 변경(vsn)
- release handler
- hot code swap
- `code_change/3`

Elixir에서도 가능하지만,
현대 Elixir 생태계에선 보통 릴리즈 롤링/블루그린으로 해결하는 경우가 많다.
그럼에도 OTP 애플리케이션 구조를 갖추면:

- 코드 교체 시점이 명확해지고
- 프로세스 트리가 표준화되어
- 업그레이드 전략을 설계하기 쉬워진다.

---

## 결론

OTP 애플리케이션은 “부모님 세대의 애플리케이션”이 아니다.

- 실행 파일이 아니라
- GUI 프로그램이 아니라
- main 루프가 아니라

**운영 가능한 기능 단위**다.

정의로 다시 말하면:

1) 기능이 “프로세스 트리(슈퍼비전)”로 구성되고
2) 시작/종료가 “OTP 표준 라이프사이클”에 의해 관리되며
3) 의존/부트스트랩/환경이 “명세(.app)”로 선언되는 것

이 세 가지가 갖춰져야
그 코드는 비로소 **OTP 애플리케이션**이 된다.

그리고 그렇게 승격된 기능 단위는:

- 장애를 국소화하고 자동 복구하며
- 다른 앱과 조립식으로 결합되고
- 릴리즈에서 안정적으로 부팅/종료되며
- 런타임 환경 변경과 관찰성을 기본으로 가진다.

단순한 수열 계산기조차 OTP 애플리케이션화하면
“운영 가능한 시스템 부품”이 된다.
이 관점이 OTP/Elixir 생태계의 진짜 핵심이다.
