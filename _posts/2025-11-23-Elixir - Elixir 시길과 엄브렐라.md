---
layout: post
title: Elixir - Elixir 시길과 엄브렐라
date: 2025-11-23 21:25:23 +0900
category: Elixir
---
# 사용자 정의와 엄브렐라(Umbrella)

## _25.1 사용자 정의 시길 만들기_

Elixir의 **시길(sigil)** 은 `~X"..."` 또는 `~X/.../` 형태로 **문자열/리터럴을 확장**하는 문법이다.
핵심은 “**시길도 결국 함수(또는 매크로)**”라는 점이며, 모듈 내부에서 `sigil_x/2` 를 정의하면 된다.

시길은 실전에서 다음 두 가지 목적에 특히 강력하다.

1) **도메인 리터럴(DSL) 제공**
   - 사람이 읽기 쉬운 리터럴을 안전한 내부 타입으로 바꿈
   - 예: 바이트/시간/화폐/좌표/정규식/SQL/그래프QL/CRON 등

2) **컴파일 타임 검증 + 최종 런타임 비용 제거(선택)**
   - 리터럴이 고정된 경우, 컴파일 중 파싱·검증하고 결과를 코드에 주입할 수 있다

---

### 시길의 기본 규칙 (초안 유지 + 내부 동작 보강)

- **소문자 시길**: **보간(interpolation)**·**이스케이프**가 _활성_
  ```elixir
  ~x"#{1 + 1}\n"  # content가 "2\n"이 된 뒤 sigil_x/2 호출
  ```
- **대문자 시길**: 보간/이스케이프가 _비활성_
  ```elixir
  ~X"#{1 + 1}\n"  # content가 "#{1 + 1}\n" 그대로 sigil_X/2 호출
  ```
- 함수 시그니처(함수로 정의 시):
  `def sigil_x(content :: binary, modifiers :: charlist) :: any`
- 표준 구분자:
  `"..."`, `'...'`, `/.../`, `|...|`, `[...]`, `{...}`, `(...)`, `<...>`

> 내장 시길 예:
> `~r`(Regex), `~S/~s`(문자열), `~w`(단어 리스트),
> `~D`(Date), `~T`(Time), `~U`(DateTime), `~N`(NaiveDateTime),
> `~H`(HEEx) 등.

#### 시길이 호출되는 “실제 함수 이름”

- `~x"..."` → `sigil_x("...", 'mods')`
- `~X"..."` → `sigil_X("...", 'mods')`

즉 **대문자/소문자는 다른 함수**로 디스패치된다.
따라서 서로 다른 의미를 줄 수도 있다.

#### “함수 시길 vs 매크로 시길”의 차이

Elixir는 아래 둘 다 허용한다.

- 함수 시길: `def sigil_x/2`
  - content는 이미 바이너리
  - 보간/이스케이프는 언어가 먼저 처리(소문자일 때)
  - **리터럴이든 변수든 동일하게 런타임 호출**

- 매크로 시길: `defmacro sigil_x/2`
  - content가 AST 형태로 들어온다
  - 리터럴일 때 **컴파일 타임 파싱/검증 후 주입 가능**
  - 변수/동적 값일 때는 런타임 처리로 폴백하도록 설계

실전에서 “컴파일 타임 검증”을 하고 싶다면 매크로 시길이 유리하다.
단, 매크로는 **위생/코드팽창/컴파일시간**을 관리해야 한다.

---

### 예제 ①: 바이트/시간 단위를 파싱하는 시길 (기존 예제 개선)

#### 요구

- `~b"10MB"` → `10 * 1024 * 1024`(바이트)
- `~t"1h30m15s"` → 총 초(second)
- 수정자(modifier)로 1000기반(=SI) vs 1024기반(=IEC) 선택
- 입력이 잘못되면 **명확한 에러 메시지**로 실패
- 실수/소수 허용(예: `1.5GiB`, `0.25h`)

아래는 기존 초안을 **표현은 유지하되**, 파서·단위·경계·정밀도·경고 메시지를 실전 수준으로 보강한 코드다.

```elixir
defmodule MySigils do
  @moduledoc """
  사용자 정의 시길 모음.

  - ~b"10MB"     : 바이트 단위 리터럴
  - ~b"10MB"s    : SI(1000) 기반
  - ~b"1.5GiB"   : IEC(1024) 기반
  - ~t"1h30m15s" : 시간 리터럴 → 초
  - ~t"250ms"m   : 시간 리터럴 → 밀리초(수정자 'm')
  """

  # ----------------------
  # ~b"...": byte literal
  # ----------------------
  # modifiers:
  #   's' -> SI base 1000
  #   'i' -> IEC base 1024 (기본)
  #   'u' -> "unit strict": 단위 누락 금지
  #
  def sigil_b(content, mods) when is_binary(content) do
    base = if ?s in mods, do: 1000, else: 1024
    strict_unit? = ?u in mods
    parse_bytes(content, base, strict_unit?)
  end

  @byte_re ~r/^\s*([+-]?\d+(?:\.\d+)?)\s*([kmgtpe]?i?b)?\s*$/i

  defp parse_bytes(str, base, strict_unit?) do
    case Regex.run(@byte_re, str) do
      [_, num_s, unit_s] ->
        num = String.to_float(num_s)
        unit = String.downcase(unit_s || "")

        if strict_unit? and unit == "" do
          raise ArgumentError, "byte literal requires a unit when 'u' modifier is used: #{inspect(str)}"
        end

        mult =
          case unit do
            ""   -> 1
            "b"  -> 1

            # SI or IEC depending on base
            "kb" -> pow(base, 1)
            "mb" -> pow(base, 2)
            "gb" -> pow(base, 3)
            "tb" -> pow(base, 4)
            "pb" -> pow(base, 5)
            "eb" -> pow(base, 6)

            # Explicit IEC units (always 1024)
            "kib" -> pow(1024, 1)
            "mib" -> pow(1024, 2)
            "gib" -> pow(1024, 3)
            "tib" -> pow(1024, 4)
            "pib" -> pow(1024, 5)
            "eib" -> pow(1024, 6)

            _ ->
              raise ArgumentError, "unknown byte unit: #{inspect(unit_s)} in #{inspect(str)}"
          end

        bytes = num * mult
        if bytes < 0 do
          raise ArgumentError, "byte literal must be non-negative: #{inspect(str)}"
        end

        # 실무적 판단: 정수 바이트로 반환
        trunc(bytes)

      _ ->
        raise ArgumentError, "invalid byte literal: #{inspect(str)} (examples: 10MB, 1.5GiB, 512kB)"
    end
  end

  defp pow(b, e), do: :math.pow(b, e)

  # ----------------------
  # ~t"...": time literal
  # ----------------------
  # modifiers:
  #   'm' -> millisecond 반환
  #   's' -> second 반환(기본)
  #   'u' -> unit strict: 단위 누락 금지
  #
  # 허용 입력 예:
  #   1h30m15s
  #   250ms
  #   2.5h
  #
  def sigil_t(content, mods) when is_binary(content) do
    strict_unit? = ?u in mods
    ms = parse_time_to_ms(content, strict_unit?)

    if ?m in mods, do: ms, else: div(ms, 1000)
  end

  @time_re ~r/([+-]?\d+(?:\.\d+)?)\s*(ms|s|m|h|d)/i

  defp parse_time_to_ms(str, strict_unit?) do
    parts = Regex.scan(@time_re, str)

    if parts == [] do
      raise ArgumentError, "invalid time literal: #{inspect(str)} (examples: 1h30m15s, 250ms, 2.5h)"
    end

    total_ms =
      Enum.reduce(parts, 0.0, fn [_, n_s, u_s], acc ->
        n = String.to_float(n_s)
        u = String.downcase(u_s)

        if strict_unit? and u == "" do
          raise ArgumentError, "time literal requires units when 'u' modifier is used: #{inspect(str)}"
        end

        acc +
          case u do
            "ms" -> n
            "s"  -> n * 1000
            "m"  -> n * 60_000
            "h"  -> n * 3_600_000
            "d"  -> n * 86_400_000
          end
      end)

    if total_ms < 0 do
      raise ArgumentError, "time literal must be non-negative: #{inspect(str)}"
    end

    round(total_ms)
  end
end
```

사용:

```elixir
import MySigils

~b"10MB"           # 10485760  (IEC 기본)
~b"10MB"s          # 10000000  (SI)
~b"1.5GiB"         # 1610612736
~b"512kb"s         # 512000
~b"512kb"          # 524288

~t"1h30m15s"       # 5415
~t"1h30m15s"m      # 5_415_000
~t"250ms"m         # 250
~t"2.5h"           # 9000
```

#### 왜 “수정자(modifier)”가 실무에서 중요한가

수정자는 시길을 **작은 DSL**로 만든다.
동일한 리터럴 문법이라도 “운영 요구”에 따라 쉽게 의미를 바꿀 수 있다.

- `~b"10MB"` 기본은 IEC
- `~b"10MB"s`는 SI로 명시
- 즉 “문법은 같되 의미를 옵션으로 제어”

실무에서 “수정자 없이 암묵적으로 다르게 동작”하게 만들면 혼란이 커지므로
**수정자는 가능하면 1~2개 선에서 의미가 명확할 때만 도입**하자.

#### 입력 검증을 “시길에서 하는 이유”

시길은 리터럴의 성격이 강하다.
리터럴은 **입력에서 오류가 나면 빨리 실패(fail fast)** 해야 한다.

- 그래서 `raise ArgumentError`로 즉시 실패
- 에러 메시지는 “허용 스펙 + 예시”까지 포함

---

### 예제 ②: JSON 리터럴 시길 (컴파일 타임 검증 강화)

초안의 목적은 좋다.

- `~j"{\"a\": 1}"` → `%{"a" => 1}`
- 대문자 `~J`는 보간 비활성

여기서 중요한 확장은 두 가지다.

1) **리터럴 JSON은 컴파일 타임에 검증**하고 싶다.
2) 동적 JSON(변수/보간)은 런타임 디코딩으로 폴백해야 한다.

이를 위해 **매크로 시길 + 런타임 폴백 함수** 패턴을 쓴다.

```elixir
defmodule SigilJSON do
  @moduledoc """
  JSON 시길.

  ~j"..."  : JSON을 map/list로
  ~J"..."  : 보간/이스케이프 비활성

  modifiers:
    'a' -> atom keys (unsafe, 테스트/내부용)
    's' -> strict (기본), 키는 문자열
    'p' -> pretty/format 후 저장(예시)
  """

  # 런타임 폴백 함수
  def decode_json!(content, mods) when is_binary(content) do
    opts =
      cond do
        ?a in mods -> [keys: :atoms!]
        true -> []
      end

    case Jason.decode(content, opts) do
      {:ok, data} -> data
      {:error, reason} ->
        raise ArgumentError, "invalid JSON: #{inspect(reason)} in #{content}"
    end
  end

  # 매크로 시길: 리터럴이면 컴파일 타임 파싱
  defmacro sigil_j({:<<>>, _meta, parts} = ast, mods) do
    # parts가 전부 binary 조각이면 컴파일 타임 상수
    if Enum.all?(parts, &is_binary/1) do
      content = IO.iodata_to_binary(parts)
      data = decode_json!(content, mods)
      quote do
        unquote(Macro.escape(data))
      end
    else
      # 동적 조각(보간)이 섞이면 런타임 폴백
      quote do
        SigilJSON.decode_json!(unquote(ast), unquote(mods))
      end
    end
  end

  # 대문자 버전도 동일 로직으로 제공
  defmacro sigil_J({:<<>>, _meta, parts} = ast, mods) do
    sigil_j(ast, mods)
  end
end
```

사용:

```elixir
import SigilJSON

# 리터럴 JSON → 컴파일 타임 검증/주입

data = ~j"{\"a\": 1, \"b\": [1,2,3]}"
# => %{"a" => 1, "b" => [1,2,3]}

# 보간 포함 → 런타임 디코딩

v = 10
data2 = ~j"{\"a\": #{v}}"
# => %{"a" => 10}

# 보간 비활성(대문자)

template = ~J"{\"a\": #{v}}"
# => %{"a" => 10}  (대문자라도 매크로가 ast를 받으므로, 최종 content는 런타임)

```

#### 왜 `keys: :atoms!`가 위험한가

JSON 키를 `atoms!`로 바꾸면 **새로운 atom을 무한히 생성**할 수 있다.
BEAM의 atom 테이블은 **GC 대상이 아니므로**, 입력이 악의적이면 OOM/노드 장애로 이어질 수 있다.

따라서 실무 규칙:

- 운영에서는 `atoms!` 금지
- 정말 필요하면 **화이트리스트 기반 변환**만 허용

화이트리스트 스케치:

```elixir
def safe_keys_to_atoms(map, allowed) do
  for {k, v} <- map, into: %{} do
    if k in allowed do
      {String.to_existing_atom(k), v}
    else
      {k, v}
    end
  end
end
```

---

### 시길 테스트 전략 (확장)

초안에 더해 실전적으로는 다음 4층으로 테스트한다.

1) **문법 테스트**
   - 소문자/대문자 보간 차이
   - 구분자 `"..."`, `/.../` 등 케이스

2) **수정자 테이블 테스트**
   - 단일/복합 수정자 조합
   - 의미 충돌이 없는지 확인

3) **경계/실패 테스트**
   - 단위 누락, 음수, 이상 단위 등

4) **프로퍼티 테스트(StreamData)**
   - `~t`는 `ms -> literal -> parse` 역함수 성질이 부분적으로 유지되는지
   - `~b`는 base/SI/IEC 변환에서 단조성 유지

테스트 예:

```elixir
defmodule MySigilsTest do
  use ExUnit.Case
  import MySigils

  describe "~b byte sigil" do
    test "IEC default" do
      assert ~b"1KB" == 1024
      assert ~b"1MB" == 1_048_576
      assert ~b"1.5GiB" == 1_610_612_736
    end

    test "SI modifier" do
      assert ~b"1KB"s == 1000
      assert ~b"1MB"s == 1_000_000
    end

    test "invalid literals fail fast" do
      assert_raise ArgumentError, fn -> ~b"abc" end
      assert_raise ArgumentError, fn -> ~b"-1MB" end
      assert_raise ArgumentError, fn -> ~b"10XB" end
    end
  end

  describe "~t time sigil" do
    test "parses mixed units" do
      assert ~t"1h" == 3600
      assert ~t"30m" == 1800
      assert ~t"15s" == 15
      assert ~t"1h30m15s" == 5415
    end

    test "milliseconds modifier" do
      assert ~t"250ms"m == 250
      assert ~t"1s"m == 1000
    end

    test "invalid time fails" do
      assert_raise ArgumentError, fn -> ~t"1x" end
      assert_raise ArgumentError, fn -> ~t"-1h" end
    end
  end
end
```

---

## _25.2 여러 앱을 관리하는 엄브렐라 프로젝트_

**엄브렐라(umbrella)** 는 **여러 OTP 앱**을 하나의 저장소/빌드/CI 파이프라인에서 통합 관리하는 Mix 구조다.
대규모 시스템을 **도메인별 앱**(코어/인프라/어댑터/웹 등)로 분해해

- 경계를 고정하고
- 의존 방향을 강제하며
- 재사용과 병렬 빌드/테스트를 가능하게 한다

즉, “코드 조직과 배포 단위의 일치”를 반강제하는 **아키텍처 도구**다.

---

### 생성과 디렉터리 구조 (초안 유지)

```bash
mix new my_umbrella --umbrella
cd my_umbrella
tree -L 2
```

```
my_umbrella/
  apps/
  config/
    config.exs
    dev.exs
    test.exs
    prod.exs
    runtime.exs
  mix.exs
```

앱 추가:

```bash
cd apps
mix new core
mix new repo
mix new web --sup
```

---

### 앱 간 의존성 규율 (강조)

**루트 mix.exs가 아니라, 각 앱이 자기 deps를 선언한다.**

```elixir
# apps/web/mix.exs

def deps do
  [
    {:core, in_umbrella: true},
    {:repo, in_umbrella: true},
    {:plug, "~> 1.16"}
  ]
end
```

#### 의존 “한 방향” 그래프

가장 안전한 방향은 다음이다.

```
core  <- repo  <- web
```

- `core`: 도메인 규칙/비헤이비어/프로토콜/순수 로직
- `repo`: 외부 리소스 접근(Ecto, ETS, HTTP 클라이언트 등)
- `web`: 입력/출력(HTTP, CLI, gRPC, WebSocket)

**순환 의존은 금지**한다.

```
core -> repo -> web -> core   (금지)
```

순환이 생긴다면 “포트/비헤이비어를 core로 올리고, 구현을 아래로 내리는 방식”으로 끊는다.

---

### 공용/앱별 설정 전략

- 루트: 전역 설정과 환경별 공통값
- 앱별: 오직 **그 앱에만 특별한 값**이 필요할 때

예:

```elixir
# config/config.exs

import Config

config :repo, Repo,
  database: "app_dev",
  username: "postgres",
  password: "postgres",
  hostname: "localhost"

config :web,
  user_repo_mod: Repo.Adapters.UserRepoImpl
```

`runtime.exs`에서 ENV 기반으로 덮어쓰는 게 운영 표준이다.

---

### 슈퍼비전 트리와 앱 경계

각 앱은 독립적인 `Application` 콜백을 가진다.

```elixir
defmodule Web.Application do
  use Application
  def start(_type, _args) do
    children = [
      {Plug.Cowboy, scheme: :http, plug: Web.Router, options: [port: 4000]}
    ]
    Supervisor.start_link(children, strategy: :one_for_one, name: Web.Supervisor)
  end
end
```

루트 실행:

```bash
mix run --no-halt
```

#### 경계를 유지하는 방법: “core는 절대 외부를 모른다”

- core는 “플러그/데이터베이스/HTTP/파일 IO” 같은 인프라 개념을 알면 안 된다.
- core에는 오직 “계약(비헤이비어/프로토콜)”만 둔다.

---

### 테스트, 빌드, CI 파이프라인

- 전체 테스트: `mix test --umbrella`
- 앱 단위: `cd apps/core && mix test`

**대형 시스템에서의 실전 패턴**

1) **변경된 앱만 선택 테스트**
2) **apps별 캐시**로 CI 시간 절감
3) core 계약 테스트를 “컨슈머(웹/레포)”에서 재검증

---

### (초안 유지 + 주의)

루트 `mix.exs`에서 통합 릴리스 정의:

```elixir
def project do
  [
    apps_path: "apps",
    start_permanent: Mix.env() == :prod,
    releases: [
      my_umbrella: [
        applications: [
          core: :permanent,
          repo: :permanent,
          web: :permanent
        ],
        include_executables_for: [:unix],
        steps: [:assemble, :tar]
      ]
    ]
  ]
end
```

빌드:

```bash
MIX_ENV=prod mix release
```

실행:

```bash
_build/prod/rel/my_umbrella/bin/my_umbrella start
```

#### 릴리스에서 자주 터지는 문제

- 빌드 환경과 런타임 환경의 설정 불일치
  → `runtime.exs`에서 ENV를 읽어 해결

- 프로토콜 컨솔리데이션 타이밍
  → 남의 타입 구현을 추가하는 앱이 있다면 “빌드 순서/옵셔널 의존”을 문서화

---

### 이름/모듈 충돌 방지 관례

- 앱별 최상위 네임스페이스 고정
  - `Core.*`
  - `Repo.*`
  - `Web.*`
- 공용 타입/프로토콜은 `Core` 아래로
- 어댑터 구현은 `Repo.Adapters.*` 또는 `Web.Adapters.*`

---

### 실제 “앱 간 경계” 예 (초안 유지)

`core` — 포트(계약)

```elixir
defmodule Core.Ports.UserRepo do
  @callback get_user(id :: term()) :: {:ok, map()} | {:error, term()}
end
```

`repo` — 구현

```elixir
defmodule Repo.Adapters.UserRepoImpl do
  @behaviour Core.Ports.UserRepo
  @impl true
  def get_user(id), do: {:ok, %{id: id, name: "Dana"}}
end
```

`web` — 의존 주입

```elixir
defmodule Web.Router do
  use Plug.Router
  plug :match
  plug :dispatch

  @user_repo Application.compile_env(:web, :user_repo_mod, Repo.Adapters.UserRepoImpl)

  get "/users/:id" do
    case @user_repo.get_user(id) do
      {:ok, u} -> send_resp(conn, 200, Jason.encode!(u))
      {:error, _} -> send_resp(conn, 404, "not found")
    end
  end
end
```

---

## _25.3 아직 끝이 아니다!_ — 실전 운영 체크리스트(확장)

엄브렐라와 시길을 손에 넣었다면, 다음은 “프로덕션 품질”이다.
아래 항목들은 실제 운영에서 빠지면 반드시 비용으로 돌아온다.

---

### 관찰성(Observability)

- **Telemetry 이벤트**는 “시스템의 혈압”이다.
- 핵심 연산(쿼리, 외부 호출, 도메인 처리, 큐 작업)에 이벤트를 박아라.

```elixir
:telemetry.execute([:repo, :query], %{time_ms: 12}, %{sql: "SELECT 1"})
```

- 이벤트는 `[:app, :layer, :action]` 네임스페이스로 일관되게

---

### 성능 & 동시성

Elixir의 동시성 최적화는 “스레드 튜닝”이 아니라

- 작업 단위 설계
- 메시지 흐름/백프레셔
- 자료구조와 I/O 경계 정리

에서 나온다.

병렬화의 한계를 단순 모델로 보면:

$$
\text{Speedup} = \frac{1}{(1-p) + \frac{p}{n}}
$$

- \(p\): 병렬 가능한 비율
- \(n\): 워커/스케줄러 수

즉, I/O·알고리즘 병목을 해결하지 않으면 병렬화는 금방 포화된다.

---

### 배포/운영

- 릴리스 실행 스크립트(`bin/app start|stop|restart`)
- 컨테이너/시스템 서비스에서 Healthcheck
- 다중 노드는 `--name`과 쿠키, 클러스터 자동 조인

```bash
_build/prod/rel/my_umbrella/bin/my_umbrella start --name web@127.0.0.1
```

---

### 안정성

- 슈퍼비전 전략을 “장애 격리 단위”로 생각하라.
- 외부 리소스 의존 프로세스는 재시도/타임아웃을 반드시 둔다.
- 백프레셔가 필요한 곳은 GenStage/Broadway 계열이 실전 표준이다.

---

### 테스팅

- 단위: ExUnit
- 모킹: Mox (비헤이비어 기반)
- 속성: StreamData
- 장애/부하: 작은 시뮬레이터를 만들어 “실패 모드”를 돌려봐라

---

### 언어 경계

- NIF/Rustler: 고성능이 필요한 곳
- Port: 격리/재시작이 필요한 외부 프로세스 연동

규칙:

- **핵심 도메인은 Elixir에 남겨라.**
- 네이티브는 정말 필요한 연산만 넣어라.

---

### 웹/실시간

- Phoenix Umbrella는 대형 시스템의 표준 선택지다.
- `Web` 앱은 IO/세션/렌더만 담당하고,
  도메인/인프라는 core/repo로 내려라.

---

### 코드 건강

- `mix format`과 CI gate
- Credo로 냄새 점검
- Dialyzer로 계약·타입 회귀 방지

---

### 문서

- ExDoc + doctest
- `@doc`에 “옵션/수정자/오버라이드 포인트”를 표로 써라.
- 특히 시길은 “문법 예시”가 문서의 절반이다.

---

## 결론

- **25.1 사용자 정의 시길**은 “작은 리터럴 DSL”을 안전하게 만들고,
  필요하면 **컴파일 타임 검증**으로 런타임 비용까지 지울 수 있다.
  소문자/대문자, 수정자, 함수/매크로 기반 차이를 이해하면 실전에서 강력한 도구다.
- **25.2 엄브렐라 프로젝트**는 시스템을 “앱 경계”로 분해해
  의존 방향을 고정하고, 테스트/배포/릴리스를 통합한다.
- **25.3 다음 단계**는 운영 품질이다.
  관찰성·성능·배포·안정성·테스팅·문서화를 함께 가져가야
  “Elixir답게 견고한 시스템”이 된다.

이 장을 그대로 바탕으로,
당신의 도메인에 맞는 시길 DSL 하나와 엄브렐라 구조 하나를 실제로 만들어 붙여보면
Elixir의 “언어 확장력”과 “시스템 확장력”을 동시에 체감하게 될 것이다.
