---
layout: post
title: Elixir - 타입 명세와 Dialyzer
date: 2025-11-23 23:25:23 +0900
category: Elixir
---
# Elixir 타입 명세(typespec)와 Dialyzer

## 0) 들어가기 — “타입 명세는 정적 문서이자 계약이다”

Elixir의 타입 명세는 C/Java처럼 “컴파일러가 강제하는 정적 타입 시스템”이 아니다.
그 대신 **(1) 문서화, (2) 도구 지원, (3) Dialyzer 검증, (4) 경계 안정화**라는 네 축을 동시에 만족시키는 **“설계의 고정 장치”**다.

### 타입 명세가 있는 코드와 없는 코드가 왜 다르게 굴러가나?

- 타입 명세는 **BEAM 바이트코드(beam)에 메타데이터로 포함**된다.
- 런타임 의미는 없다.
  즉, **typespec을 추가한다고 실행 경로가 바뀌거나 느려지지 않는다.**
- 대신, 메타데이터를 읽는 도구(ExDoc, LSP, Dialyzer)가 **해석을 더 정확히** 한다.

> 결론: typespec은 **“실행을 바꾸지 않고 설계를 바꾸는 도구”**다.

### 최근 Elixir의 “점진적 타입/체커” 흐름

Elixir는 1.18~1.19 계열에서 **타입 기반 정적 분석과 체커를 강화**해,
typespec이 **단순 문서**를 넘어 **실제 오류 검출의 기반**으로 더 중요해졌다.
따라서 “큰 프로젝트일수록 specs-first 문화”가 유리하다.

---

## 1) 타입 명세의 4대 사용처 — 어디에 왜 쓰는가?

### 문서/가독성 (ExDoc)

- `@spec`, `@type`, `@typedoc`은 ExDoc이 **API 문서에 자동 반영**한다.
- 팀원/미래의 나에게 **“이 함수가 무엇을 받으며 무엇을 내놓는지”**를 한 줄로 못 박는다.

### 도구 지원 (IDE/LSP)

- 타입이 있으면
  - 자동완성 후보가 줄고,
  - jump-to-definition 정확도가 올라가며,
  - “이 값이 지금 무엇인지”를 LSP가 더 잘 보여준다.

### Dialyzer 검증 (성공 타입 분석)

Dialyzer는 **“이 코드는 반드시 실패한다”**가 증명되는 버그를 찾는다.

- 런타임 전체를 시뮬레이션하지 않는다.
- 명세 + 실제 구현의 교집합을 **“성공 타입(success typing)”**으로 본다.
- 그래서 **명백한 오류만 확실히 잡고**, 애매한 건 경고를 줄인다.

### 경계 명세 (API/behaviour/protocol 안정화)

- 공개 API, behaviour 콜백, 프로토콜 시그니처를 typespec으로 고정하면
  - 구현체 교체,
  - 모킹,
  - 리팩토링
  에서 **경계가 흔들리지 않는다.**

---

## 2) typespec 문법 A to Z — 실무에서 헷갈리는 것까지

### 기본 타입

Elixir 타입 언어는 Erlang 타입 언어와 호환된다.

대표:

- `integer()`, `float()`, `number()`
- `atom()`, `boolean()`
- `binary()`, `bitstring()`
- `list(t)`, `nonempty_list(t)`
- `map()`, `tuple()`
- `pid()`, `port()`, `reference()`, `fun()`
- `term()` / `any()` (거의 동일 관용)
- `timeout()` = `non_neg_integer() | :infinity`

```elixir
@spec f(integer(), binary()) :: boolean()
def f(i, b), do: i > 0 and is_binary(b)
```

### 리터럴 타입과 유니언 타입

```elixir
@type status :: :pending | :paid | :failed
@type maybe_int :: integer() | nil
@type ok(T) :: {:ok, T}
@type err :: {:error, term()}

@spec pay(amount :: pos_integer(), currency :: atom()) ::
        ok(status()) | err()
```

- 리터럴(`:ok`, `:error`, `"USD"`, `0`, `1`)도 타입이 된다.
- 유니언은 `|` 로 표현.

### 원격 타입(다른 모듈 타입 재사용)

```elixir
@type user_id :: pos_integer()
@type user_name :: String.t()

@spec greet(user_name(), user_id()) :: String.t()
def greet(name, id), do: "#{name}(##{id})"
```

- `Module.type()`로 호출.
- 대표적으로 `String.t()`, `Date.t()` 등이 많이 쓰인다.

### 함수 타입

```elixir
@spec mapper((term() -> term()), list(term())) :: list(term())
def mapper(f, xs), do: Enum.map(xs, f)
```

- `(A -> B)` 형태.
- 다인자 함수는 `(A, B -> C)`.

### 리스트/튜플 정밀 타입

```elixir
@type pair(A, B) :: {A, B}
@type triple :: {atom(), integer(), binary()}

@spec first(pair(term(), term())) :: term()
def first({a, _b}), do: a
```

튜플은 “고정 길이 + 위치별 타입”을 명시할 수 있어 **태그드 유니언**을 표현할 때 중요하다.

```elixir
@type result(T) :: {:ok, T} | {:error, atom(), String.t()}
```

### 맵 타입 — required/optional 키

```elixir
@type user_map ::
        %{
          required(:id) => pos_integer(),
          required(:name) => String.t(),
          optional(:email) => String.t()
        }

@spec greet(user_map()) :: String.t()
def greet(%{name: n}), do: "Hello, " <> n
```

- `%{}`는 **“shape(키-타입 구조)”** 를 표현 가능.
- Dialyzer가 “이 키는 반드시 있음/없음”을 파악할 수 있다.

#### 키 값이 특정 리터럴일 때의 타입 모델링

```elixir
@type event ::
        %{required(:type) => :login, required(:user_id) => pos_integer()}
        | %{required(:type) => :logout, required(:user_id) => pos_integer()}
        | %{required(:type) => :purchase, required(:order_id) => pos_integer(), required(:amount) => pos_integer()}
```

→ **태그 기반 분기**를 Dialyzer가 안정적으로 따라간다.

### 키워드(Keyword) 타입

```elixir
@type opts :: keyword(term())

@spec run(opts()) :: :ok
def run(opts), do: IO.inspect(opts)
```

- `keyword(T)`는 사실상 `[{atom(), T}]`의 별칭.

### 바이너리/비트스트링/iodata/chardata

```elixir
@spec to_bin(iodata()) :: binary()
def to_bin(io), do: IO.iodata_to_binary(io)
```

- `iodata()`는 “중첩된 리스트 + 바이너리”의 I/O 친화 타입.
- 네트워크/파일/로깅에서 필수적.

### 두 가지 최상위 타입: `term()` vs `any()`

- 문서적으론 동일하게 취급해도 된다.
- 관례:
  - “정말 아무거나”면 `term()`
  - “아직 정확히 못 정했지만 나중에 좁힐 거라면” `any()`

---

## 3) 새 타입 정의하기 — @type / @typep / @opaque

### 공개 타입 vs 비공개 타입

```elixir
defmodule Money do
  @enforce_keys [:amount, :currency]
  defstruct [:amount, :currency]

  @type t :: %__MODULE__{
          amount: non_neg_integer(),
          currency: String.t()
        }

  @typep cents :: non_neg_integer()
end
```

- `@type`은 외부 문서/사용자에게 공개.
- `@typep`는 내부에서만 재사용.

### 불투명 타입(@opaque)

```elixir
defmodule Token do
  @opaque t :: binary()

  @spec issue(map()) :: t()
  def issue(payload), do: :erlang.term_to_binary(payload)

  @spec verify(t()) :: {:ok, map()} | {:error, :bad_token}
  def verify(bin) do
    try do
      {:ok, :erlang.binary_to_term(bin)}
    rescue
      _ -> {:error, :bad_token}
    end
  end
end
```

- 외부는 `Token.t()`를 **해부(패턴매칭)할 수 없는 것으로 취급**해야 한다.
- Dialyzer가 외부 해부를 경고하는 식으로 **경계 준수 문화를 강제**한다.

---

## 4) @spec 고급 패턴 — 실무에서 자주 터지는 지점

### 여러 함수 절(헤드)과 명세

```elixir
defmodule Repo do
  @type pk :: pos_integer()
  @type record :: %{required(:id) => pk(), optional(atom()) => term()}

  @spec get(pk()) :: {:ok, record()} | :error
  def get(0), do: :error
  def get(id), do: {:ok, %{id: id, name: "x"}}
end
```

- 명세는 **함수 전체(모든 절)의 합집합**을 표현해야 한다.
- 특정 절에서만 가능한 반환을 정확히 포함하라.

### `!` 접미사와 명세 일관성

```elixir
@spec get!(pk()) :: record()
def get!(id) do
  case get(id) do
    {:ok, r} -> r
    :error -> raise ArgumentError, "not found"
  end
end
```

- 정상 API는 값 기반(`{:ok, v} | :error`),
- `!` API는 예외 기반으로 **반환 타입이 더 좁다.**

### 단계적 파이프(with)와 명세

```elixir
defmodule Flow do
  @type ok(T) :: {:ok, T}
  @type err :: {:error, term()}

  @spec run(term()) :: ok(map()) | err()
  def run(x) do
    with {:ok, a} <- step1(x),
         {:ok, b} <- step2(a) do
      {:ok, %{a: a, b: b}}
    end
  end

  @spec step1(term()) :: ok(integer()) | err()
  defp step1(_), do: {:ok, 1}

  @spec step2(integer()) :: ok(String.t()) | err()
  defp step2(_), do: {:ok, "ok"}
end
```

- `with`는 성공 경로를 **연속적 계약**으로 만든다.
- 명세는 계약의 “타입 그래프”를 문서화한다.

---

## 5) behaviour/protocol과 타입 명세 — 계약을 진짜로 쓰는 법

### behaviour 콜백에 타입 별칭 재사용

```elixir
defmodule Storage do
  @type key :: term()
  @type value :: term()

  @callback put(key(), value()) :: :ok | {:error, term()}
  @callback get(key()) :: {:ok, value()} | :not_found
end
```

### 구현 모듈은 @impl + @spec로 “콜백-구현 동치”를 못 박는다

```elixir
defmodule MemStore do
  @behaviour Storage

  @impl Storage
  @spec put(Storage.key(), Storage.value()) :: :ok
  def put(k, v), do: (Process.put({:k, k}, v); :ok)

  @impl true
  @spec get(Storage.key()) :: {:ok, Storage.value()} | :not_found
  def get(k) do
    case Process.get({:k, k}) do
      nil -> :not_found
      v -> {:ok, v}
    end
  end
end
```

- 콜백과 구현의 반환이 다르면 Dialyzer/컴파일러가 경고한다.
- 팀의 **인터페이스 합의가 코드로 자동 검증**된다.

### 프로토콜 시그니처는 “문서 계약”으로 간단히

```elixir
defprotocol Printable do
  @spec print(term()) :: String.t()
  def print(x)
end
```

- 실제 타입별 정밀성은 `defimpl` 쪽에 자연스럽게 녹여도 된다.

---

## 6) Dialyzer 완전 분해 — 왜 이런 경고가 나오는가?

### 성공 타입(success typing) 직관

Dialyzer가 하는 일은 대략 이렇게 요약된다.

1. typespec이 있으면 **“허용 가능한 입력/출력 영역”**을 만든다.
2. 구현 코드를 분석해 **“실제 가능한 입력/출력 영역”**을 만든다.
3. 두 영역이 **겹치지 않거나, 특정 호출이 겹치지 않으면** 경고를 낸다.

즉, “명세가 틀렸다” 또는 “구현이 틀렸다” 둘 중 하나다.

### 대표 경고와 원인/해결

#### `no_return`

- 구현이 항상 `raise/exit/throw` 또는 무한 루프로 끝날 때.

```elixir
defmodule W do
  @spec die!() :: no_return()
  def die!(), do: raise "boom"
end
```

해결:
- 진짜로 반환하지 않는 함수면 `no_return()`로 명세를 맞춘다.
- 아니라면 구현을 수정.

#### `call with contract`

- 호출자가 피호출자 명세를 위반하는 인자를 넘길 때.

```elixir
defmodule C do
  @spec head(nonempty_list(integer())) :: integer()
  def head([x | _]), do: x
end

C.head([]) # Dialyzer: call with contract
```

해결:
- 호출을 고치거나,
- 명세를 완화(`list(integer())`)하거나,
- 가드를 넣어 호출을 정제.

#### `mismatch`

- 명세의 반환 타입과 구현이 어긋나는 대표 케이스.

```elixir
defmodule M do
  @spec f(pos_integer()) :: pos_integer()
  def f(0), do: 0
  def f(x), do: x
end
```

해결:
- 명세 수정(`non_neg_integer()`),
- 혹은 0 절 제거/가드 강화.

### 가드가 타입 분석에 미치는 효과

```elixir
defmodule Refine do
  @spec only_pos(number()) :: {:ok, pos_integer()} | {:error, :neg}
  def only_pos(n) when is_integer(n) and n > 0, do: {:ok, n}
  def only_pos(_), do: {:error, :neg}
end
```

- `is_integer/1` 같은 가드가 Dialyzer에 **타입 좁히기 힌트**가 된다.
- “경고를 없애기 위한 가드”가 아니라
  **“설계를 더 명확히 하기 위한 가드”**가 결과적으로 경고를 줄인다.

### false positive/negative 감각

- false positive(거짓 경고)는
  “분석이 보수적이라 잠재적으로 가능하다고 보는 케이스”에서 생긴다.
- false negative(놓침)는
  “분석이 너무 비싸거나 불가능해서 넘어가는 케이스”에서 생긴다.

대응 전략:

1. 경고를 **무시하지 말고 원인 분해**
2. 명세/가드/패턴을 **조금 더 구체화**
3. 정말 의도한 구간만 ignore

---

## 7) Dialyxir 설정과 운영 — 프로젝트에 붙이는 법

### 설치

```elixir
# mix.exs

def deps do
  [
    {:dialyxir, "~> 1.4", only: [:dev], runtime: false}
  ]
end
```

- `only: [:dev]`
  → 운영 릴리스에서 불필요.
- `runtime: false`
  → 런타임 의존 제거.

### 실행 흐름

```bash
mix deps.get
mix dialyzer
```

- 첫 실행은 PLT 구축 때문에 느리다.
- 이후는 PLT 재사용으로 빨라진다.

### Umbrella에서의 PLT 공유

- 루트에서 `mix dialyzer` 하면
  전체 앱에 공통 PLT를 재사용해 **빌드 비용이 줄어든다**.

### CI/캐시

PLT 캐시는 성능을 지배한다.

- 캐시 키에
  - Erlang/OTP 버전
  - Elixir 버전
  - deps 해시
  를 포함시켜야 한다.

분석 시간은 대략

$$
T \approx T_{\text{plt}} + \sum f(\text{코드량}, \text{의존도})
$$

→ PLT를 재사용하면 $T_{\text{plt}}$가 거의 0이 된다.

### ignore 전략

- `.dialyzer_ignore.exs` 또는 `ignore_warnings` 옵션을 쓸 수 있다.
- 그러나 **“왜 무시하는지” 설명 주석** 없이는 장기적으로 독이 된다.
- 고의적 캐스팅은 헬퍼로 감싸 명세를 정리하라.

---

## 8) 타입 명세 기반 개발 워크플로우(실전)

### “typespec-first” 흐름

1. **도메인 타입(@type)** 먼저 정의
2. 공개 API **@spec** 먼저 고정
3. behaviour/port 인터페이스에 **@callback + 타입 재사용**
4. 구현
5. Dialyzer 경고 → **명세/가드/구현을 함께 정교화**
6. 테스트로 회귀 방지

### Mox/모킹과 타입 계약

behaviour 기반 모킹은 **타입 계약이 있을 때 더 강해진다.**

- 모킹 모듈이 콜백명/시그니처를 지키지 않으면 컴파일/분석에서 깨진다.
- “테스트 더블이 진짜 인터페이스를 어기는 버그”를 미리 막는다.

---

## 9) 엔드투엔드 미니 프로젝트 — 결제/청구 도메인

### 도메인 타입 정의

```elixir
defmodule Billing.Types do
  @type money :: %{
          required(:amount) => non_neg_integer(),
          required(:currency) => <<_::24>>
        }

  @type customer_id :: pos_integer()
  @type payment_token :: binary()

  @type decline_reason :: :declined | :network

  @type charge_ok ::
          {:ok, %{id: String.t(), captured: boolean()}}

  @type charge_err ::
          {:error, {decline_reason(), String.t()}}

  @type charge_result :: charge_ok() | charge_err()
end
```

- 3바이트 통화코드를 `<<_::24>>`로 제한.
- 결과는 태그드 유니언으로 명확히.

### 포트(behaviour) 정의

```elixir
defmodule Billing.Port.Gateway do
  @callback charge(Billing.Types.money(), Billing.Types.payment_token()) ::
              Billing.Types.charge_result()
end
```

### 서비스 구현

```elixir
defmodule Billing.Service do
  @behaviour Billing.Port.Gateway

  @impl true
  @spec charge(Billing.Types.money(), Billing.Types.payment_token()) ::
          Billing.Types.charge_result()
  def charge(%{amount: a, currency: <<_::24>>} = money, token)
      when is_integer(a) and a >= 0 and is_binary(token) do
    # 실제 구현에선 외부 PG 호출
    {:ok, %{id: "tx_1", captured: true, money: money}}
  end

  @spec bill(Billing.Types.customer_id(), Billing.Types.money(), module()) ::
          {:ok, String.t()} | {:error, term()}
  def bill(cid, money, gateway \\ __MODULE__) do
    with {:ok, token} <- get_default_token(cid),
         {:ok, %{id: id}} <- gateway.charge(money, token) do
      {:ok, id}
    else
      {:error, r} -> {:error, r}
    end
  end

  @spec get_default_token(Billing.Types.customer_id()) ::
          {:ok, Billing.Types.payment_token()} | {:error, :no_card}
  defp get_default_token(_cid), do: {:ok, "tok_abc"}
end
```

핵심:

- `bill/3`는 `gateway`를 주입받는다.
- `gateway`는 **behaviour 계약을 지켜야 함**.
  Dialyzer/컴파일러가 이를 자동 확인한다.

### 테스트 더블 예

```elixir
defmodule Billing.MockGateway do
  @behaviour Billing.Port.Gateway

  @impl true
  def charge(_money, _token), do: {:error, {:declined, "no funds"}}
end

defmodule BillingTest do
  use ExUnit.Case

  test "bill propagates gateway errors" do
    money = %{amount: 100, currency: "USD"}
    assert Billing.Service.bill(1, money, Billing.MockGateway) ==
           {:error, {:declined, "no funds"}}
  end
end
```

- 모킹 모듈이 콜백을 누락하면 **컴파일 경고**부터 난다.

---

## 10) 체크리스트 & 안티패턴

### 체크리스트

- [ ] 공개 API에는 `@spec` 필수
- [ ] 대표 타입은 `@type t :: %__MODULE__{...}`
- [ ] 맵 타입은 `required/optional`로 shape 고정
- [ ] behaviour 콜백은 타입 별칭을 재사용
- [ ] 구현은 `@impl`과 `@spec`을 함께
- [ ] Dialyzer 경고를 무시하지 말고
  - 명세
  - 가드/패턴
  - 구현
  을 같이 정교화
- [ ] CI에 `mix test` + `mix dialyzer` 병렬 루프

### 대표 안티패턴

1. **`term()` 남발**
   - 설계 포기 신호가 된다.
   - 최소한 도메인 별칭이라도 만들어라.

2. **너무 좁은 명세**
   - 실제 구현과 겹치지 않아 Dialyzer가 폭발한다.
   - 명세는 “의도한 입력/출력의 합집합”이어야 한다.

3. **ignore 파일 과용**
   - 지금은 조용해도 미래에 폭탄.
   - 의도적 위반은 헬퍼로 캡슐화.

4. **behaviour 없이 모킹**
   - 테스트 더블이 계약을 어기기 쉬워진다.
   - port/adapter 경계는 behaviour로 고정하라.

---

## 결론

타입 명세는 “정적 타입 시스템 대체재”가 아니다.
대신 **문서·도구·검증·경계 안정화**를 동시에 수행하는 **설계의 핵심 도구**다.

- 정상 흐름은 명세로 **읽히게 만들고**
- 경계는 behaviour/protocol로 **고정하며**
- Dialyzer는 “확실히 틀린 것”을 찾아
  **리팩토링과 확장의 안전망**이 된다.

규율:

> **도메인 타입을 먼저 만들고,
> 계약을 behaviour로 고정하며,
> Dialyzer 경고를 설계 개선의 신호로 받아들여라.**
