---
layout: post
title: Elixir - IEx 디버깅, 테스트, 모니터링, 포맷팅
date: 2025-11-20 14:25:23 +0900
category: Elixir
---
# IEx 디버깅, 테스트, 모니터링, 포맷팅

## IEx를 사용한 디버깅

### IEx 기본: 헬퍼, 상태, 다시 컴파일

```elixir
# IEx 헬퍼

h Enum.map/2          # 문서(타입/예제)
i "안녕"               # 타입/메타 정보
t DateTime            # 타입/구조체 정보
s                      # 현재 세션 바인딩
v                      # 마지막 결과
```

- `h/1`
  - 모듈/함수의 **@doc / @moduledoc** 을 보여준다.
  - 직접 작성한 문서가 **실제로 어떻게 노출되는지** 확인할 때 필수.
- `i/1`
  - 값의 **타입, 구조, 구현 모듈** 등을 보여준다.
  - “이 값이 정확히 뭐지?” 싶은 모든 순간에 사용.
- `t/1`
  - 구조체/모듈의 **타입 정보(@type/@typedoc)** 확인.
- `s/0`
  - 현재 IEx 세션에서 **바인딩된 변수들**을 키워드 리스트로 보여준다.
- `v/0`
  - **직전 결과값**을 다시 가져온다. `v(-2)`처럼 숫자 인수로 **이전 결과들**도 접근 가능.

#### 세션 유지한 채 다시 컴파일

장기 IEx 세션에서 코드를 고친 뒤, 매번 `iex -S mix` 를 다시 띄우면 너무 느리다.
IEx는 세션을 유지하고 **모듈만 다시 컴파일**할 수 있다.

```elixir
recompile()            # mix run -e "IEx.configure..." 환경에서 전체 프로젝트 재컴파일
r(MyApp.Mod)           # 해당 모듈만 재로드
```

- `recompile/0`
  - `mix`로 시작한 IEx에서 **변경된 파일만** 다시 컴파일한다.
  - Phoenix나 Mix 프로젝트에서 **앱 전체 변경**을 반영할 때 사용.
- `r/1`
  - 지정한 모듈만 **강제로 재로드**.
  - “이 모듈만 계속 바꾸면서 실험할 때” 유용.

스크립트(파일) 단독 실행도 자주 쓰게 된다.

```elixir
c "scratch.exs"        # 컴파일 & 로드 (스크립트용)
```

- `c/1` 로 컴파일된 모듈은 **현재 IEx 세션**에 바로 올라오므로,
  스크립트로 실험 코드 작성 → IEx에서 결과를 바로 확인하는 식으로 사용 가능하다.

#### 실전 팁: “실험용” 세션과 “서비스 디버깅용” 세션 분리

- **실험 세션**
  - 로컬에서 모듈/함수 설계, 간단 실험. `c/1`, `recompile/0` 활용.
- **서비스 디버깅 세션**
  - 실제 앱/서버에 붙어서 상태 확인. 이때는 모듈 재컴파일이 **실제 서비스 동작을 바꾼다**는 점을 항상 의식해야 한다.
  - 운영 환경에서는 보통 **read-only 관찰** 위주, 수정은 최소화.

---

### `dbg/2` — 값과 위치를 같이 본다

```elixir
def work(x) do
  x
  |> Enum.map(&(&1 * &1))
  |> dbg()
  |> Enum.sum()
end
```

- `dbg/2` 는 **파일 이름, 줄 번호, 파이프라인 중간 표현식**과 그 **평가 결과**를 함께 찍어준다.
- 단순 `IO.inspect/2` 와 달리, **어디서 찍힌 값인지**까지 바로 보인다.

예:

```elixir
iex> work([1,2,3])
[debug] lib/my_app.ex:5:
      Enum.map(x, &(&1 * &1)) #=> [1, 4, 9]
14
```

파이프라인 중간에 `dbg/2` 를 여러 개 넣어 **데이터가 어떻게 변하는지** 관찰하는 식으로 활용한다.

#### `IO.inspect/2` 와의 비교

```elixir
def work2(x) do
  x
  |> Enum.map(&(&1 * &1))
  |> IO.inspect(label: "after square")
  |> Enum.sum()
end
```

- `IO.inspect/2`
  - **레이블**을 붙일 수 있어 좋고, 포맷 옵션도 다양하지만
  - “파일/줄/표현식” 정보는 직접 적어야 한다.
- `dbg/2`
  - **표현식 자체**와 결과를 보여 주기 때문에, 파이프라인 디버깅에 더 최적화.

실천적인 규칙:

1. 초기에 **데이터 흐름**이 헷갈릴 때는 `dbg/2` 를 파이프 중간에 여러 개 박아서 전체 흐름을 눈으로 본다.
2. 나중에 “영구적인 로그”가 필요할 때 일부를 `Logger.debug/2` 나 `IO.inspect/2` 로 교체한다.

---

### `IEx.pry/0` — 런타임 브레이크포인트

```elixir
defmodule Demo do
  require IEx

  def run(x) do
    y = x * 2
    IEx.pry()         # 여기서 인터랙티브 쉘에 진입
    y + 1
  end
end
```

실행:

```elixir
iex -S mix
iex> Demo.run(10)

Break reached: Demo.run/1 (lib/demo.ex:6)

    4:   def run(x) do
    5:     y = x * 2
    6:     IEx.pry()         # 여기서 인터랙티브 쉘에 진입
    7:     y + 1
    8:   end

pry(1)> y
20
pry(1)> continue()
21
```

- `IEx.pry/0` 가 실행되는 시점에 **현재 프로세스를 잠시 멈추고**,
  그 시점의 **바인딩/환경을 가진 IEx 프롬프트**로 들어간다.
- `continue/0`, `respawn/0` 등 IEx 명령으로 흐름을 재개.

#### 웹 요청 한가운데 멈추기

Phoenix 컨트롤러에서:

```elixir
def show(conn, %{"id" => id}) do
  require IEx
  IEx.pry()
  user = MyApp.get_user!(id)
  render(conn, :show, user: user)
end
```

- 브라우저에서 `/users/123` 을 요청 → 서버가 해당 줄에서 멈추고 IEx로 진입.
- IEx에서 `conn`, `id`, `user` 등을 직접 평가해 볼 수 있다.

운영 환경에서는 보통 `pry` 를 **코드에 남겨두지 않는다**.
디버깅이 끝나면 반드시 제거하거나 조건부로 감싼다.

---

### `break!/4` — 소스 줄에 브레이크 설치

```elixir
# 함수 헤드 단위 브레이크 포인트

break! MyApp.Calc, :sum, 2   # sum/2 진입 시 멈춤

# 파일의 특정 줄에도 가능

break! MyApp.Calc, 42        # 42행에 브레이크
```

브레이크 이후 제어:

```elixir
continue()   # 다음 브레이크 지점까지 계속
whereami()   # 현재 위치 주변 코드 보여줌
flush()      # 메시지 버퍼 비우기
```

- `break!/3,4` 는 **코드를 수정하지 않고** 브레이크포인트를 설치할 수 있다는 점에서 `IEx.pry/0` 와 다르다.
- 특히, 이미 배포된 코드를 손대지 않고 **특정 조건에서만 잠시 멈추고 싶을 때** 유용하다.

#### 가드/매칭이 많은 함수에 브레이크 걸기

패턴이 여러 개인 함수에 브레이크를 걸면, 어느 헤드에 걸리는지 헷갈릴 수 있다.
이때는:

1. `:sum, 2` 전체에 브레이크 →
2. **어떤 패턴에 매칭되었는지** `whereami/0` 로 확인 →
3. 필요한 패턴만 선택해 다시 브레이크를 세분화.

---

### 리모트 셸/다중 노드 디버깅

```bash
# 노드 A: 앱 실행

elixir --name a@127.0.0.1 -S mix phx.server

# 노드 B: IEx 붙이기

iex --name b@127.0.0.1 --remsh a@127.0.0.1
```

- `--remsh` 로 **다른 노드에서 실행 중인 VM**에 붙는다.
- 붙은 뒤에는 로컬 IEx와 거의 동일하게 `h/1`, `i/1`, `dbg/2`, `break!/4` 등을 사용할 수 있다.

몇 가지 추가 도구:

```elixir
Node.list()                # 연결된 노드 목록
:rpc.call(:"a@127.0.0.1", MyApp, :info, [])  # 원격 호출
Process.list()             # 현재 노드의 프로세스
```

다중 노드 시스템에서는 **어느 노드에 문제가 있는지**를 먼저 판단한 뒤,
그 노드에 붙어서 프로세스 상태, 메시지 큐, ETS, 로깅 등을 점검하는 흐름이 일반적이다.

---

### 경량 트레이싱: `:erlang.trace/3`, `:dbg`

```elixir
:dbg.tracer()
:dbg.p(:all, :c)                                # 모든 프로세스 호출 추적(고위험)
:dbg.tpl(MyApp.Worker, :handle_call, :x)       # 특정 함수만
```

- `:dbg` 는 **Erlang/OTP 레벨**의 트레이싱 도구로,
  함수 호출/반환/메시지 송수신/프로세스 이벤트 등을 **실행 중에** 추적할 수 있다.
- 꼭 필요한 범위에만 사용해야 한다. 잘못 쓰면 **전체 시스템에 큰 오버헤드**를 줄 수 있다.

좀 더 정교한 패턴:

```elixir
:dbg.tracer()
:dbg.p(self(), :c)  # 현재 프로세스만
:dbg.tpl(MyApp.Worker, :handle_call, :x)

# 이후 MyApp.Worker.handle_call/3 이 호출될 때마다 트레이스 메시지가 출력

```

생산 환경에서는 `:dbg` 보다 **recon 계열 라이브러리**(프로세스/메모리/포트/메시지 큐 분석)에 의존하는 경우가 많다.

---

## 테스트

### ExUnit 기본

```elixir
# test/my_app/calc_test.exs

defmodule MyApp.CalcTest do
  use ExUnit.Case, async: true

  test "sum/1 sums positives" do
    assert MyApp.Calc.sum([1,2,3]) == 6
  end

  test "sum/1 handles empty" do
    assert MyApp.Calc.sum([]) == 0
  end
end
```

실행:

```bash
mix test
mix test --only focus
mix test test/my_app/calc_test.exs:12
```

- `async: true`
  - 각 테스트가 **독립 프로세스**에서 실행됨을 의미.
  - **공유 상태/DB 접근**이 없다면 적극 활용해 CPU를 모두 사용.
- `mix test --only focus`
  - `@moduletag :focus` 나 `@tag :focus` 가 붙은 테스트만 실행.
- `mix test path:line`
  - 특정 테스트만 빠르게 실행할 때 유용.

#### 단언(Assertion) 패턴 확장

```elixir
assert result == 10
refute user == nil
assert_raise ArgumentError, fn -> MyApp.faulty(nil) end
assert_in_delta 3.14, MyApp.pi(), 0.001
assert_receive {:done, id}, 1_000
```

- **값 비교** 뿐 아니라, **예외/메시지/시간**을 포괄하는 다양한 단언을 제공한다.
- 특히 `assert_receive/2` 는 **동시성 코드 테스트**에서 핵심.

---

### Doctest — 문서가 곧 테스트

```elixir
defmodule MyApp.Calc do
  @moduledoc """
  더하기 유틸.

      iex> MyApp.Calc.sum([1,2,3])
      6

      iex> MyApp.Calc.sum([])
      0
  """
  def sum(xs), do: Enum.sum(xs)
end

# test

defmodule MyApp.Calc.DocTest do
  use ExUnit.Case, async: true
  doctest MyApp.Calc
end
```

- **문서 안의 iex 예제**가 실제 테스트로 실행된다.
- API 문서에 예제를 적을 때
  “**실제로 이렇게 동작하는지**” 를 자동으로 검증할 수 있어, **문서/코드 불일치**를 막아 준다.

팁:

- 라이브러리 프로젝트에서는 **공개 API** 의 대부분에 Doctest를 붙여 두면,
  문서가 자연스럽게 “사용 예제 + 테스트”를 겸하게 된다.

---

### 프로퍼티 테스트: StreamData

```elixir
# mix.exs deps: {:stream_data, "~> 0.6", only: :test}

use ExUnitProperties

property "sum is commutative" do
  check all a <- list_of(integer()),
            b <- list_of(integer()) do
    assert Enum.sum(a ++ b) == Enum.sum(b ++ a)
  end
end
```

- **임의 데이터 생성기(generator)** 를 바탕으로,
  수십/수백 개의 입력에 대해 **일반적인 성질(property)** 이 유지되는지 검사한다.
- 실패 시, 라이브러리가 자동으로 **가장 단순한 반례(minimal counterexample)** 를 찾아 준다.

유용한 생성기 예:

```elixir
check all m <- map_of(string(:alphanumeric), integer(), max_length: 10),
          k <- string(:alphanumeric) do
  assert is_map(m)
end
```

- CRUD/파서/인코더/디코더 같이 **입력 도메인이 커서 예시 몇 개로 검증하기 힘든 로직**에 특히 강력하다.

---

### 목킹: Mox (동적 비헤이비어 주입)

```elixir
# behavior

defmodule MyApp.Mailer do
  @callback deliver(map()) :: :ok | {:error, term()}
end

# 실제 구현

defmodule MyApp.SMTPMailer do
  @behaviour MyApp.Mailer
  def deliver(email) do
    # 실제 SMTP 전송
  end
end
```

테스트에서:

```elixir
# test_helper.exs

Mox.defmock(MyApp.MailerMock, for: MyApp.Mailer)
Application.put_env(:my_app, :mailer, MyApp.MailerMock)

# 테스트

defmodule MyApp.NotifierTest do
  use ExUnit.Case, async: true
  import Mox
  setup :verify_on_exit!

  test "sends email" do
    MyApp.MailerMock
    |> expect(:deliver, fn _ -> :ok end)

    assert :ok == MyApp.Notifier.notify(%{to: "a@b.com"})
  end
end
```

- 프로덕션 코드에서는 `Application.fetch_env!(:my_app, :mailer)` 로 구현 모듈을 가져다가 사용.
- 테스트에서는 해당 자리에 **Mock 모듈을 주입**해,
  실제 네트워크/외부 서비스를 호출하지 않고도 로직을 검증할 수 있다.

---

### DB 테스트: Ecto SQL Sandbox

```elixir
# test/test_helper.exs

Ecto.Adapters.SQL.Sandbox.mode(MyApp.Repo, :manual)
ExUnit.start()
```

케이스 템플릿:

```elixir
defmodule MyApp.DataCase do
  use ExUnit.CaseTemplate

  using do
    quote do
      alias MyApp.Repo
      import Ecto.Query
    end
  end

  setup tags do
    :ok = Ecto.Adapters.SQL.Sandbox.checkout(MyApp.Repo)

    unless tags[:async] do
      Ecto.Adapters.SQL.Sandbox.mode(MyApp.Repo, {:shared, self()})
    end

    :ok
  end
end
```

이후 테스트에서:

```elixir
defmodule MyApp.UserTest do
  use MyApp.DataCase, async: true

  test "create user" do
    {:ok, user} = MyApp.create_user(%{email: "a@b.com"})
    assert user.id
  end
end
```

- 각 테스트가 **독립된 트랜잭션** 안에서 실행되고, 테스트가 끝나면 롤백되어 **DB 상태가 깨끗하게 유지**된다.

---

### 성능 테스트 힌트

- 마이크로벤치마크는 `Benchee` 를 많이 사용한다.

```elixir
Benchee.run(%{
  "iodata" => fn -> build_big_iodata() end,
  "<>"     => fn -> build_big_concat() end
})
```

- `<>` 연쇄는 최악의 경우 $$O(n^2)$$, iodata는 $$O(n)$$ 근사다.
  문자열/바이너리 장에서 본 것처럼, **성능 민감한 코드**에서는 iodata 패턴을 습관화해야 한다.

---

## 코드 의존성

### Mix deps — 라이브러리 의존성 관리

```elixir
# mix.exs

defp deps do
  [
    {:jason, "~> 1.4"},
    {:ecto_sql, "~> 3.12"},
    {:mox, "~> 1.0", only: :test},
    {:stream_data, "~> 0.6", only: :test}
  ]
end
```

- `mix deps.get / update / clean` 으로 의존성 관리.
- `only: :test`, `only: [:dev, :test]` 등으로 **환경별 의존성**을 분리해
  프로덕션 릴리즈에는 꼭 필요한 것만 포함시키는 것이 좋다.

---

### xref — 순환/금지 의존성 검사

```bash
mix xref graph
mix xref callers MyApp.Context.User
mix xref warnings
```

- `mix xref graph`
  - 모듈 간 **호출 그래프**를 그려준다.
- `mix xref warnings`
  - **순환 의존성**이나 사용되지 않는 코드 등을 경고.

#### 레이어드 아키텍처에서의 활용

예:

- `MyApp.Core` (도메인/순수 로직)
- `MyApp.Repo` (DB)
- `MyApp.Web` (Phoenix/HTTP)

규칙:

- `Web -> Core` 는 허용하되 `Core -> Web` 은 금지.
- `Core` 가 `Web` 을 참조하면 안 된다.

이때, CI에서 `mix xref warnings` 를 실행하고,
`Core` 가 `Web` 모듈을 호출하는 순간 **빌드 실패**로 처리하면 **레이어 침식**을 막을 수 있다.

---

### 애플리케이션 바운더리 설계

Umbrella 프로젝트나 멀티 앱 구조에서는 디렉터리 구조로 바운더리를 명시한다.

예:

```text
apps/
  core/          # 순수 도메인 로직
  repo/          # DB, Ecto
  web/           # Phoenix, HTTP
```

- `core`는 `repo`와 `web`에 대해 **알지 못한다**. (인터페이스는 behavior/포트로만)
- `web`은 `core` 를 호출해 도메인 로직을 사용한다.

이렇게 해 두면:

1. **도메인 테스트를 매우 빠르게** 돌릴 수 있고,
2. 웹 프레임워크를 바꾸거나 확장해도 도메인 로직은 최대한 재사용 가능하다.

---

### Code ownership/visibility 가이드

- `MyApp.Internal.*` 네임스페이스를 활용해 “내부용” 모듈을 구분한다.
- 내부 모듈에는 `@moduledoc false` 를 붙여 공개 문서에서 숨긴다.
- 외부에 노출되는 API에는:
  - **명확한 @doc/@typedoc**
  - **반환 타입(@spec)**
  - **예외/오류 계약(raise/튜플)** 을 모두 써 두는 것이 좋다.

---

### Dialyzer 힌트(정적 분석)

```elixir
# mix.exs

{:dialyxir, "~> 1.4", only: [:dev], runtime: false}
```

- `Dialyzer` 는 **타입 기반 정적 분석기**로,
  실제 런타임 타입과 사용 패턴에 기반해 **잠재 버그나 API 오사용**을 찾아 준다.

예:

```elixir
@spec sum([integer()]) :: integer()
def sum(xs), do: Enum.sum(xs)
```

- 스펙과 실제 구현이 어긋나면 Dialyzer가 경고를 낸다.
- 특히 **라이브러리/도메인 레이어**에 스펙을 잘 적어두면,
  상위 레이어에서 잘못 사용했을 때 즉시 경고를 받을 수 있다.

---

## 서버 모니터링

### :observer / :etop — VM 내부를 시각화

```elixir
:observer.start()          # 프로세스/메모리/포트/ETS 시각화
:etop.start()              # top-유사 프로세스 뷰
```

- `:observer`
  - **프로세스 리스트**, **메시지 큐 길이**, **메모리(프로세스/ETS/바이너리)**,
    **애플리케이션/릴리즈** 등을 GUI로 보여준다.
- `:etop`
  - 터미널 기반 **프로세스 top**.
  - 레덕션/메시지 큐 기준으로 상위 N개 프로세스를 보여 준다.

병목 분석에서 중요한 관찰 포인트:

1. **메시지 큐 길이**
   - 특정 프로세스(예: GenServer)의 큐가 계속 증가하면,
     해당 프로세스가 일을 처리하는 속도보다 요청이 더 빠르게 들어온다는 의미.
2. **레덕션 수**
   - 특정 프로세스가 전체 CPU를 많이 소모하는지 여부를 판단.
3. **바이너리 메모리**
   - 서브바이너리 홀드나 대용량 응답이 메모리를 잡고 있는지 확인.

---

### Telemetry — 표준 이벤트 버스

```elixir
# 계측 emit

:telemetry.execute(
  [:my_app, :db, :query],
  %{duration: native_time},
  %{query: sql, source: :users}
)

# 핸들러

:telemetry.attach(
  "log-db",
  [:my_app, :db, :query],
  fn _event, measures, meta, _ ->
    require Logger
    micros =
      System.convert_time_unit(
        measures.duration,
        :native,
        :microsecond
      )

    Logger.info("[db] #{meta.query} took #{micros}µs (#{meta.source})")
  end,
  nil
)
```

- Telemetry는 **라이브러리들이 서로 약속하고 사용하는 이벤트 포맷/버스**다.
- Phoenix, Ecto, Oban 같은 주요 라이브러리들은 이미 Telemetry 이벤트를 방출한다.
- 애플리케이션 자체에서도 Telemetry를 사용해 **일관된 메트릭/이벤트 스트림**을 만들 수 있다.

장점:

1. 라이브러리 구현과 **관찰 코드(로그/메트릭)** 를 분리할 수 있다.
2. 나중에 Prometheus/StatsD/Log 등 어떤 백엔드를 쓰더라도,
   **핸들러만 교체**하면 된다.

---

### Phoenix LiveDashboard — 런타임 관측

라우터 설정:

```elixir
if Application.compile_env(:my_app, :dev_routes) do
  import Phoenix.LiveDashboard.Router

  scope "/" do
    pipe_through :browser

    live_dashboard "/dashboard",
      metrics: MyAppWeb.Telemetry,
      ecto_repo: MyApp.Repo
  end
end
```

- LiveDashboard는 Phoenix + Telemetry 기반으로
  **요청 지연, DB 쿼리 시간, 메모리, ETS, 프로세스 샘플링** 등을 웹 UI로 보여준다.
- dev 환경에서는 기본 활성화되는 경우가 많고,
  프로덕션에서는 **VPN/인증** 뒤에 숨기는 것이 일반적이다.

`MyAppWeb.Telemetry` 모듈에서 `telemetry_metrics` 를 정의:

```elixir
summary("phoenix.endpoint.stop.duration",
  unit: {:native, :millisecond},
  tags: [:route]
)
```

- 이 정의를 바탕으로 LiveDashboard가 **그래프/요약 통계**를 그린다.

---

### PromEx — Prometheus 통합

```elixir
# mix.exs

{:prom_ex, "~> 1.11"}
```

구성 예:

```elixir
# config/config.exs

config :my_app, MyApp.PromEx,
  metrics_server: true,
  grafana: [host: "http://grafana.internal"]

# application.ex

children = [
  MyApp.PromEx,
  MyApp.Repo,
  MyAppWeb.Endpoint
]
```

- PromEx는 Elixir/BEAM, Phoenix, Ecto 등 주요 라이브러리의
  **Prometheus 메트릭과 Grafana 대시보드**를 한 번에 구성해 준다.
- `/metrics` 엔드포인트에서 Prometheus 텍스트 포맷으로 메트릭을 노출하고,
  Grafana에서는 준비된 대시보드를 바로 사용할 수 있다.

---

### 로깅 전략

구조화 로깅 예:

```elixir
config :logger, backends: [LoggerJSON]
config :logger_json,
  metadata: :all,
  json_encoder: Jason
```

- JSON 형태의 로그를 사용하면,
  ELK/Opensearch/Grafana Loki 같은 시스템에서 **필드 기반 검색/집계**가 가능해진다.

실제 로그 예:

```elixir
Logger.info("user login",
  user_id: user.id,
  email: user.email,
  source: :web
)
```

- 나중에 로그 수집 시스템에서 `source:web AND message:"user login"` 같은 식으로 쿼리.

에러 로깅 패턴:

```elixir
try do
  work!()
rescue
  e ->
    require Logger
    Logger.error(Exception.format(:error, e, __STACKTRACE__),
      error: e.__struct__,
      message: e.message
    )

    reraise e, __STACKTRACE__
end
```

- 예외를 **로깅 + 재전파**하여
  Supervisor가 재시작을 처리하게 하면서도, 운영 로그에는 상세 정보가 남도록 한다.

---

### 운영 팁

- **메시지 큐 임계치**
  - 큐 길이가 N 이상인 프로세스가 일정 시간 지속되면 알람.
- **서브바이너리 홀드**
  - 메모리가 이상하게 늘어날 때는 `recon:bin_leak/1` 류의 도구로
    어떤 프로세스가 대형 바이너리를 쥐고 있는지 확인.
- **스케줄러 활용도**
  - 네이티브 NIF/포트가 스케줄러를 블로킹하는지,
    혹은 특정 스케줄러만 과도하게 사용되는지 관찰.

---

## 소스 코드 포맷팅

### `mix format` — 팀 공통 스타일

```bash
mix format
mix format --check-formatted
```

- `mix format`
  - Elixir의 **표준 코드 포맷터**.
  - “스타일 논쟁”을 거의 제거해 준다.
- `--check-formatted`
  - CI에서 코드가 포맷 규칙을 따르는지 확인용.

일반적인 CI 단계:

1. `mix format --check-formatted`
2. `mix credo --strict`
3. `mix test`

---

### `.formatter.exs` 설정

```elixir
# .formatter.exs

[
  import_deps: [:ecto, :phoenix],
  inputs: [
    "{mix,.formatter}.exs",
    "{config,lib,test}/**/*.{ex,exs}"
  ],
  line_length: 100,
  locals_without_parens: [
    plug: 1, plug: 2,
    field: 2, field: 3
  ]
]
```

- `import_deps`
  - 의존 라이브러리(예: Phoenix)의 DSL 스타일을 가져온다.
- `line_length`
  - 팀에서 합의한 줄 길이.
- `locals_without_parens`
  - DSL 스타일 함수를 지정해 괄호를 생략할 수 있다.

예:

```elixir
scope "/", MyAppWeb do
  pipe_through :browser
  get "/", PageController, :index
end
```

- 이런 DSL는 괄호를 생략하는 편이 더 읽기 좋다.
  `.formatter.exs` 에서 이를 지정해두면, `mix format` 이 자동으로 맞춰 준다.

---

### Credo(린팅)와 역할 분리

- 포맷터는 **“어떻게 보이는가”** 를 담당,
- Credo는 **“무슨 냄새가 나는가”** 를 담당.

```elixir
# .credo.exs

checks: [
  {Credo.Check.Readability.MaxLineLength, max_length: 100},
  {Credo.Check.Refactor.CyclomaticComplexity, []},
  {Credo.Check.Warning.UnusedEnumOperation, []}
]
```

- 복잡도, 중복, 사용되지 않는 변수/연산 등 코드 품질을 점검한다.

---

### 문서/타입/테스트의 삼각 일관성

코드 품질을 높이는 기본 루틴:

1. `mix format` 으로 **형식**을 맞추고,
2. `mix credo` 로 **구조/냄새**를 점검하고,
3. `mix test` 로 **행동**을 검증한다.
4. 공개 API에는 `@moduledoc/@doc/@typedoc/@spec` 을 꼼꼼히 작성하고 Doctest를 붙인다.

이렇게 하면:

- IDE/편집기에서 문서/타입을 바로 참고할 수 있고,
- 호환성/계약이 테스트로 항상 검증되고,
- 변경 시 실패하는 테스트/타입 경고를 통해 **파급 범위**를 쉽게 파악할 수 있다.

---

## 전체 실전 예제: “지연 변환 + 파일 스트리밍 + 모니터링 + 테스트”

아래 예제는 이 장에서 나온 도구들을 **한 번에** 묶어 보여준다.

- `IEx` 로 디버깅 가능
- `Telemetry` 로 실행 시간/라인 카운트 측정
- `ExUnit` 으로 성공/실패 케이스 테스트
- `mix format` 으로 스타일 유지

### 1) 구현

```elixir
defmodule MyApp.Report do
  @moduledoc """
  대용량 CSV를 스트리밍으로 읽고 변환하여 출력한다.

      iex> MyApp.Report.transform!("in.csv", "out.csv")
      :ok
  """

  require Logger

  def transform!(src, dst) do
    :telemetry.span([:my_app, :report, :transform], %{src: src, dst: dst}, fn ->
      {:ok, input} = File.open(src, [:read])
      {:ok, output} = File.open(dst, [:write])

      try do
        IO.stream(input, :line)
        |> Stream.drop_while(&String.starts_with?(&1, "#"))  # 주석 스킵
        |> Stream.map(&String.trim_trailing/1)
        |> Stream.reject(&(&1 == ""))
        |> Stream.map(&rewrite_line/1)
        |> Enum.into(IO.stream(output, :line))

        :ok
      after
        File.close(input)
        File.close(output)
      end
      |> then(fn result ->
        {result, %{lines: :erlang.get(:lines) || 0}}
      end)
    end)
  end

  defp rewrite_line(line) do
    :erlang.put(:lines, (:erlang.get(:lines) || 0) + 1)

    case String.split(line, ",") do
      [id, name, score] ->
        "#{id},#{String.upcase(name)},#{score}\n"

      _ ->
        raise ArgumentError, message: "bad csv line: #{inspect(line)}"
    end
  end
end
```

- 스트림으로 읽어 한 줄씩 처리 → **메모리 사용 최소화**.
- Telemetry `span/3` 로 전체 실행 시간, 처리 라인 수를 기록.
- 잘못된 라인에 대해서는 `ArgumentError` 로 **빠르게 실패**.

---

### 2) Telemetry 핸들러

```elixir
:telemetry.attach(
  "report-logger",
  [:my_app, :report, :transform, :stop],
  fn _event, meas, meta, _ ->
    usec =
      System.convert_time_unit(
        meas.duration,
        :native,
        :microsecond
      )

    Logger.info("[report] #{meta.src} -> #{meta.dst} " <>
                  "in #{usec}µs (#{meta.lines} lines)")
  end,
  nil
)
```

- 작업 하나가 끝날 때마다
  “어떤 파일을 몇 줄 처리했고, 얼마나 걸렸는지” 를 로그에 남긴다.

---

### 3) 테스트(예외/성공/로그 캡처)

```elixir
defmodule MyApp.ReportTest do
  use ExUnit.Case, async: true
  import ExUnit.CaptureLog

  test "transforms csv" do
    in_  = Path.join(System.tmp_dir!(), "in.csv")
    out_ = Path.join(System.tmp_dir!(), "out.csv")

    File.write!(in_, "#comment\n1,kim,10\n2,lee,20\n")

    log =
      capture_log(fn ->
        assert :ok == MyApp.Report.transform!(in_, out_)
      end)

    assert File.read!(out_) == "1,KIM,10\n2,LEE,20\n"
    assert log =~ "[report]"
    assert log =~ "2 lines"
  end

  test "bad line raises" do
    in_  = Path.join(System.tmp_dir!(), "bad.csv")
    out_ = Path.join(System.tmp_dir!(), "out.csv")

    File.write!(in_, "oops\n")

    assert_raise ArgumentError, fn ->
      MyApp.Report.transform!(in_, out_)
    end
  end
end
```

- `capture_log/1` 으로 Telemetry 핸들러가 남긴 로그를 검증.
- 정상/예외 케이스 모두 명시적으로 테스트한다.

---

## 흔한 함정 → 교정

1) **IEx에서 모듈을 reload 안 함**
   - 증상: 코드를 고쳤는데 IEx에서 결과가 바뀌지 않는다.
   - 교정: `recompile()/0`, `r(Mod)` 를 습관적으로 사용.

2) **테스트에서 전역 상태/시간 의존**
   - 증상: 로컬에서는 되는데 CI에서 가끔 실패하는 플래키 테스트.
   - 교정: 전역 ETS/Agent/시계 대신 **의존성 주입**, **샌드박스**, **가짜 시계**를 사용.

3) **의존성 레이어 역참조**
   - 증상: 도메인 모듈이 웹/인프라 모듈을 호출, 테스트/재사용이 급격히 어려워짐.
   - 교정: `mix xref` 로 의존 그래프를 확인하고 레이어 규칙을 문서화/CI로 강제.

4) **모니터링 없이 추측 디버깅**
   - 증상: 성능 문제를 재현하지 못해 “감”으로 튜닝.
   - 교정: Telemetry/LiveDashboard/Prometheus 같은 **관찰 도구**를 먼저 깔고,
     실제 메트릭을 보면서 최적화.

5) **포맷/린트 미적용 코드 누적**
   - 증상: PR마다 스타일 논쟁, 점점 난독화.
   - 교정: `.formatter.exs` + `mix format --check-formatted` + Credo를
     **필수 파이프라인**으로 고정.

---

## 연습 문제

1) **IEx 세션 디버깅**
   - Phoenix 컨트롤러에 `IEx.pry/0` 를 넣고 실제 요청을 보내
     `conn`/`params` 바인딩을 확인해 보라.
   - 같은 액션에 `break!/2` 를 걸어, 요청이 도달할 때마다 멈추는 것도 실습해 보라.

2) **프로퍼티 테스트**
   - 임의 문자열/숫자를 포함하는 CSV 라인에 대해,
     “encode → decode → encode” round-trip 이 항상 동일한지 StreamData로 검증해 보라.

3) **xref 규칙**
   - `core`/`web`/`repo` 세 레이어를 가진 작은 Umbrella 프로젝트를 만들고,
     `core` 가 `web` 을 참조하면 실패하도록 `mix xref warnings` 를 CI에 추가해 보라.

4) **Telemetry→Prometheus**
   - 간단한 카운터/요약 메트릭을 PromEx로 노출하고,
     `/metrics` 엔드포인트를 Prometheus로 스크랩해 Grafana에서 조회해 보라.

5) **포맷팅 DSL 최적화**
   - `.formatter.exs` 의 `locals_without_parens` 를 이용해
     Phoenix 라우터/플러그 DSL 가독성을 개선하고, 프로젝트 전체에 포맷을 적용해 보라.

---

## 마무리

- **IEx**: `dbg/2`, `IEx.pry/0`, `break!/4`, `:dbg` 를 활용해 **실행 중인 시스템**을 안전하게 관찰하라.
- **테스트**: ExUnit/Doctest/StreamData/Mox/Ecto Sandbox 로 **명세 수준의 테스트**를 작성하라.
- **코드 의존성**: `mix xref`, 레이어 규칙, Dialyzer 스펙으로 **경계를 지키고 타입 계약을 명확히** 하라.
- **모니터링**: Telemetry/LiveDashboard/Prometheus/구조화 로깅으로 **운영 가시성을 기본값**으로 가져가라.
- **포맷팅**: `mix format` + Credo로 **스타일과 품질을 자동화**하라.

이 다섯 축이 맞물릴 때, 엘릭서 프로젝트는 **디버깅이 쉬운 코드베이스**에서 출발해
**운영/테스트/관찰이 자연스럽게 통합된 시스템**으로 성장하게 된다.
