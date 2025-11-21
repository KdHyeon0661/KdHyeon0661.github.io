---
layout: post
title: Elixir - Elixir 프로토콜
date: 2025-11-23 19:25:23 +0900
category: Elixir
---
# Elixir 프로토콜(Protocol)

## _24.1 프로토콜 정의하기

Elixir **프로토콜(Protocol)** 은 “**첫 번째 인자의 타입**”에 따라 **다형적으로 디스패치**되는 **함수 묶음의 계약**이다.
즉, 같은 함수 이름이라도 **리스트, 맵, 구조체** 등 타입별로 **구현이 달라질 수 있다**.

### 가장 작은 예 — `Size` 프로토콜 (초안 포함)

```elixir
defprotocol Size do
  @moduledoc """
  컬렉션/값의 크기를 정수로 반환하는 계약.
  반드시 `size/1`를 구현해야 한다.
  """
  @type t :: term()
  @fallback_to_any true
  @doc "값의 크기를 정수로 반환한다."
  @spec size(t) :: non_neg_integer()
  def size(value)
end
```

핵심 요소

- `defprotocol Name do ... end` 블록 안에 **콜백 시그니처**를 나열한다.
- `@fallback_to_any true` 를 쓰면, **정확한 타입 구현이 없을 때** `Any` 구현으로 **폴백**한다(선택).
- `@doc`, `@spec` 으로 문서와 타입을 붙이면 **Dialyzer/문서** 품질이 좋아진다.

### 디스패치 규칙 한 장 요약 (초안 포함)

- **항상 첫 번째 인자의 타입**으로 디스패치한다.
- “타입”이란 **내장 타입**(List, Map, BitString, Tuple 등)과 **구조체(struct)** 를 의미한다.
  (plain map `%{}` 은 **Map 타입**으로 취급. **구조체**는 `%Mod{}` 형태)
- 해당 타입에 대한 `defimpl` 이 있으면 그 구현으로, 없으면 `@fallback_to_any` 가 설정된 경우 **Any** 로 간다.
- **프로토콜은 멀티 메서드**처럼 보이지만, Elixir 런타임에서 **컴파일-컨솔리데이션**을 통해 효율적 테이블 디스패치를 사용한다(뒤에 설명).

### 실무에서 자주 쓰는 기본 프로토콜들 (초안 포함)

- `Enumerable` / `Collectable` / `Inspect` / `String.Chars` / `List.Chars` 등
  → 이 글의 예제에서 몇 가지를 참고 구현한다.

---

### 프로토콜의 “정체”: behaviour와 무엇이 다른가?

behaviour와 프로토콜은 둘 다 “계약”이지만, **다형성의 축이 다르다.**

| 구분 | Behaviour | Protocol |
|---|---|---|
| 다형성 기준 | “**어떤 모듈**이냐” | “**첫 번째 인자의 타입**이 무엇이냐” |
| 디스패치 시점 | 호출부가 모듈을 선택 | 런타임이 타입으로 자동 선택 |
| 구현 단위 | 모듈 | 타입(내장 타입 또는 구조체) |
| 대표 예 | GenServer 콜백 합의 | Enumerable/String.Chars/Inspect |
| 설계 감각 | 포트/어댑터(교체 가능한 모듈) | 컬렉션/도메인 타입의 “동작 확장” |

**한 문장 요약**
- “여러 구현 모듈을 바꿔 끼우고 싶다” → behaviour
- “내 타입을 Elixir 생태계의 표준 연산(열거/수집/표시/문자화)에 자연스럽게 녹이고 싶다” → protocol

---

### 프로토콜이 좋은 이유: “내 타입을 언어의 원주민처럼”

프로토콜을 구현하면 **우리 구조체**가 **내장 타입과 동등한 시민권**을 얻는다.

- `Enum.map/2`가 “그냥 된다”
- `Enum.into/2`로 “그냥 들어간다”
- `inspect/1`이 “보기 좋게 나온다”
- `to_string/1`이 “의미 있게 변환된다”

즉, “**표준 도구와 합쳐지는 확장성**”이 프로토콜의 핵심 가치다.

---

## _24.2 프로토콜 구현하기_

`defimpl` 로 **특정 타입**에 대한 구현을 제공한다.

### 내장 타입에 대한 구현 (초안 포함)

#### 리스트

```elixir
defimpl Size, for: List do
  def size(list), do: length(list)
end
```

#### 맵(plain map)

```elixir
defimpl Size, for: Map do
  def size(map), do: map_size(map)
end
```

#### 비트스트링(예: 바이너리)

```elixir
defimpl Size, for: BitString do
  def size(<<>>), do: 0          # 빈 바이너리
  def size(bin) when is_binary(bin), do: byte_size(bin)
  def size(bitstring), do: bit_size(bitstring)  # 일반 비트스트링일 수도 있음
end
```

#### 튜플

```elixir
defimpl Size, for: Tuple do
  def size(t), do: tuple_size(t)
end
```

#### Any 폴백

```elixir
defimpl Size, for: Any do
  def size(_), do: 1
end
```

- `@fallback_to_any true` 를 켠 덕분에 **정의되지 않은 타입**은 1을 반환하도록 폴백.
- 폴백은 편리하지만, **조용히 잘못된 값**을 낼 위험도 있다.
  **도메인에 따라** 폴백 대신 **명시적 구현 강제**를 선호한다.

사용 예:

```elixir
Size.size([1,2,3])        # 3
Size.size(%{a: 1, b: 2})  # 2
Size.size("abc")          # 3
Size.size({:ok, 1, 2})    # 3
Size.size(:foo)           # 1 (Any 폴백)
```

---

### 구조체에 대한 구현 — “도메인 모델” 예제 (초안 포함 + 확장)

이번엔 간단한 **도형** 구조체를 만들어 `Area` 프로토콜로 **넓이**를 구해보자.

#### 프로토콜 정의

```elixir
defprotocol Area do
  @moduledoc "도형의 넓이를 계산"
  @spec area(term()) :: number()
  def area(shape)
end
```

#### 구조체 정의

```elixir
defmodule Rect do
  @enforce_keys [:w, :h]
  defstruct [:w, :h]
end

defmodule Circle do
  @enforce_keys [:r]
  defstruct [:r]
end
```

#### 구현

```elixir
defimpl Area, for: Rect do
  def area(%Rect{w: w, h: h}), do: w * h
end

defimpl Area, for: Circle do
  def area(%Circle{r: r}), do: :math.pi() * r * r
end
```

수학 확인:

- 직사각형: $$A = w \cdot h$$
- 원: $$A = \pi r^2$$

사용:

```elixir
Area.area(%Rect{w: 3, h: 4})   # 12
Area.area(%Circle{r: 10})      # 314.159...
```

> 팁: **구조체에 대한 프로토콜 구현은 그 구조체를 소유한 앱에서** 제공하는 것이 충돌을 줄인다.
> 제3의 라이브러리에서 **남의 구조체** + **남의 프로토콜**에 동시에 구현을 추가하면 **중복/충돌 위험**이 있다(“고아 구현” 문제).
> 필요 시 **새 래퍼 타입**을 만들거나 **로컬 프로토콜**을 쓰는 전략도 있다.

---

### “고아 구현(orphan impl)” 문제를 제대로 이해하자

**고아 구현**이란:

- A 라이브러리가 정의한 **프로토콜 P**
- B 라이브러리가 정의한 **구조체 T**
- C(우리 코드)가 **P for T** 를 추가하는 상황

이때 생기는 문제:

1) **구현 충돌**
   다른 패키지(D)가 같은 조합을 구현하면 **중복 구현**이 된다.

2) **컨솔리데이션 결과 불안정**
   어떤 구현이 최종 디스패치 테이블에 들어가는지 **빌드/릴리즈 순서**에 달라질 수 있다.

3) **내 코드가 남의 타입 의미를 멋대로 고정**
   타입 소유자(B)가 의도한 의미와 다를 수 있다.

**실무 규율(강력 추천)**

- “내 타입 + 남의 프로토콜”은 OK
- “남의 타입 + 내 프로토콜”은 OK
- “남의 타입 + 남의 프로토콜”은 **최대한 피하라**

정말 필요하면:

- **래퍼 타입 만들기**
- 또는 **로컬 프로토콜로 감싸기**

---

### `@derive` — 선언적으로 기본 구현 얻기 (초안 포함 + 확장)

여러 프로토콜은 **자동 파생(derive)** 을 지원한다. 가장 흔한 예는 `Inspect`.

```elixir
defmodule User do
  @derive {Inspect, only: [:id, :name]}  # 특정 필드만 보여주기
  defstruct [:id, :name, :password_hash, :role]
end

inspect(%User{id: 1, name: "A", password_hash: "xxx", role: :admin})
# => %User{id: 1, name: "A"}

```

여러 개 파생도 가능:

```elixir
defmodule Point do
  @derive [Inspect, Jason.Encoder]  # (예) Jason.Encoder 파생
  defstruct [:x, :y]
end
```

> 프로토콜마다 파생 시 전달 가능한 옵션이 다르다.
> 파생이 제공되지 않는 프로토콜이라면, **직접 `defimpl`** 을 작성하거나
> **커스텀 파생 매크로(__deriving__)** 를 만들어야 한다(뒤의 고급 섹션 참조).

**derive가 주는 이득**

- 반복되는 defimpl 보일러 제거
- “필드 기반 자동 구현”이 쉬움
- 컨벤션 유지

---

### 프로토콜 파생의 내부 원리 (고급)

`@derive` 는 구조체 컴파일 과정에서:

1) “파생할 프로토콜 목록”을 읽고
2) 해당 프로토콜 모듈에 `__deriving__/3` 콜백이 있으면 실행
3) 없으면 기본 파생 규칙(프로토콜이 제공)으로 구현을 생성

즉, 파생은 **컴파일 타임 코드 생성**이다.
앞에서 배운 매크로/AST 지식이 그대로 적용된다.

---

### `Any` 폴백의 공학적 의미

폴백은 “안정성”과 “엄격성” 사이의 트레이드오프다.

- **폴백 ON**: 새로운 타입이 들어와도 시스템이 동작한다
- **폴백 OFF**: 구현 누락을 조기에 드러낸다

예를 들어 `Size` 같은 “대부분의 타입에도 의미가 있는 연산”은 폴백 ON이 자연스러울 수 있다.
반면 `Area` 처럼 “도형에만 의미가 있는 연산”은 폴백 OFF가 안전하다.

폴백을 끄는 방법:

```elixir
defprotocol Area do
  @fallback_to_any false
  def area(shape)
end
```

이러면 미구현 타입 호출 시 `Protocol.UndefinedError`가 난다.

---

## _24.3 내장 프로토콜에 맞춰 우리 타입을 “생태계 표준”으로 만들기_

프로토콜의 진짜 맛은 **내장 프로토콜 구현**에서 나온다.
직접 손으로 구현해보자.

---

### `Enumerable` — “Enum이 먹는 타입” 만들기

`Enumerable`은 “열거 가능한 컬렉션” 표준 계약이다.
구현해야 할 최소 콜백:

- `reduce/3`
- `count/1` (가능하면)
- `member?/2` (가능하면)
- `slice/1` (가능하면)

#### 예제: 멀티셋 Bag 구조체

```elixir
defmodule Bag do
  @moduledoc "단순 멀티셋: items 리스트를 보유"
  defstruct items: []
end
```

Enumerable 구현:

```elixir
defimpl Enumerable, for: Bag do
  # 원소 개수
  def count(%Bag{items: items}), do: {:ok, length(items)}

  # membership 검사
  def member?(%Bag{items: items}, x), do: {:ok, Enum.member?(items, x)}

  # slice 최적화가 없으면 error
  def slice(_bag), do: {:error, __MODULE__}

  # 핵심: reduce. 대부분 List.reduce에 위임 가능
  def reduce(%Bag{items: items}, acc, fun) do
    Enumerable.List.reduce(items, acc, fun)
  end
end
```

사용:

```elixir
b = %Bag{items: [1,2,2,3]}
Enum.map(b, &(&1 * 10))         # [10,20,20,30]
Enum.reduce(b, 0, &+/2)         # 8
Enum.filter(b, &(&1 == 2))      # [2,2]
Enum.member?(b, 3)              # true
```

**reduce의 의미**

- Enum 전부는 “reduce를 어떻게 구현했는지”에 의해 정의된다.
- reduce가 정확하면 나머지는 자연스럽게 따라온다.
- 성능 최적화(예: slice)도 여기서 추가할 수 있다.

---

### `Collectable` — “Enum.into로 수집 가능한 타입” 만들기

Collectable은 “데이터를 받아서 내 타입으로 수집하는 방법”을 정의한다.

`into/1`이 반환해야 하는 형태:

```elixir
{initial, collector_fun}
```

`collector_fun` 는 세 단계 이벤트를 받는다.

- `{:cont, elem}`  → 원소 추가
- `:done`          → 최종 결과 반환
- `:halt`          → 중단 처리

Bag 구현:

```elixir
defimpl Collectable, for: Bag do
  def into(%Bag{items: items}) do
    {items, fn
      acc, {:cont, elem} ->
        [elem | acc]   # 앞에 붙이고
      acc, :done ->
        %Bag{items: Enum.reverse(acc)}  # 마지막에 뒤집어 순서 맞춤
      _acc, :halt ->
        :ok
    end}
  end
end
```

사용:

```elixir
Enum.into(1..5, %Bag{})     # %Bag{items: [1,2,3,4,5]}
Enum.into([:a, :b], %Bag{}) # %Bag{items: [:a, :b]}
```

---

### `String.Chars` / `List.Chars` — 문자열 변환 표준화

- `String.Chars` : `to_string/1`을 지원하는 타입
- `List.Chars`   : 문자 리스트(charlist)로 변환할 수 있는 타입

Bag을 문자열로 보기 좋게:

```elixir
defimpl String.Chars, for: Bag do
  def to_string(%Bag{items: items}) do
    inner =
      items
      |> Enum.map(&Kernel.to_string/1)
      |> Enum.join(", ")

    "Bag(" <> inner <> ")"
  end
end
```

사용:

```elixir
to_string(%Bag{items: [1,2,3]}) # "Bag(1, 2, 3)"
```

---

### `Inspect` — 디버깅/로그 가독성 결정

Inspect는 REPL/로그/에러 메시지에서 “어떻게 보일 것인가”를 정의한다.

간단 구현:

```elixir
defimpl Inspect, for: Bag do
  import Inspect.Algebra

  def inspect(%Bag{items: items}, opts) do
    concat([
      "Bag<",
      to_doc(items, opts),
      ">"
    ])
  end
end
```

사용:

```elixir
inspect(%Bag{items: [1,2,3]})
# "Bag<[1, 2, 3]>"

```

Inspect는 단순 문자열이 아니라 **Algebra 문서**를 이용해
**예쁜 줄바꿈/들여쓰기 제어**가 가능하다.
규모가 있는 구조체는 Inspect 구현이 “디버깅 생산성”을 크게 좌우한다.

---

## — 성능과 운영의 핵_

프로토콜은 런타임에도 동적으로 보일 수 있지만, 실제로는 **컴파일 타임 최적화**가 매우 크다.

### 컨솔리데이션이 하는 일

프로토콜 `P`가 있고 구현 타입이 `T1, T2, ..., Tk`라 하자.

- 컴파일러는 “각 타입 → 구현 모듈” 매핑 테이블을 만든다.
- 디스패치는 그 테이블을 통해 **O(1)에 가까운** 점프가 된다.

대략적인 비용 모델로 보면:

- 함수 if/cond 디스패치:
  $$T_{\text{if}} \approx N \cdot (k \cdot c_{\text{branch}} + t)$$
- 컨솔리데이션 후 프로토콜 디스패치:
  $$T_{\text{proto}} \approx N \cdot (c_{\text{table}} + t)$$

여기서
- \(N\): 호출 횟수
- \(k\): 구현 타입 수
- \(c_{\text{branch}}\): 분기 비용
- \(c_{\text{table}}\): 테이블 조회 비용
- \(t\): 실제 연산 비용

따라서 타입이 많아질수록 프로토콜이 구조적으로 유리해진다.

---

### 언제 컨솔리데이션이 깨질 수 있나?

1) **릴리즈에서 동적 모듈 로딩/플러그인**
   릴리즈 후에 새로운 `defimpl`을 로드하면
   이미 굳어진 테이블에 반영되지 않는다.

2) **고아 구현 + 의존 순서 문제**
   어떤 구현이 테이블에 들어갈지 **빌드 순서**에 의존할 수 있다.

3) **Mix 설정에서 consolidate_protocols 꺼짐**
   학습용/핫로딩 환경에서는 꺼질 수 있어 성능 특성이 달라진다.

---

### 실무 규칙

- 릴리즈/프로덕션은 **컨솔리데이션 ON**이 기본.
- 플러그인 구조가 필요하면:
  - “프로토콜이 아니라 behaviour 기반 DI”가 더 안전한 경우가 많다.
  - 또는 **컨솔리데이션을 의도적으로 끄고** 런타임 확장을 허용해야 하지만
    그 비용/안정성 책임을 팀이 함께 져야 한다.

---

## _24.5 고급: 커스텀 파생(__deriving__)으로 defimpl 자동 생성_

프로토콜이 반복적인 패턴 구현을 강제한다면, 파생 훅으로 줄일 수 있다.

### “간단 직렬화(MiniEnc)” 프로토콜

```elixir
defprotocol MiniEnc do
  @doc "값을 단순 문자열로 직렬화"
  def dump(term)
end
```

### 파생 훅 제공

```elixir
defmodule MiniEnc do
  defmacro __deriving__(module, struct, opts) do
    fields =
      struct
      |> Map.keys()
      |> Enum.reject(&(&1 in [:__struct__]))
      |> then(fn ks ->
        ks = if only = opts[:only], do: Enum.filter(ks, &(&1 in only)), else: ks
        if except = opts[:except], do: Enum.reject(ks, &(&1 in except)), else: ks
      end)

    quote do
      defimpl MiniEnc, for: unquote(module) do
        def dump(%unquote(module){} = s) do
          pairs =
            unquote(fields)
            |> Enum.map(fn k -> {k, Map.get(s, k)} end)
            |> Enum.map(fn {k, v} -> "#{k}=#{inspect(v)}" end)

          "#{"#{unquote(module)}"}<" <> Enum.join(pairs, ", ") <> ">"
        end
      end
    end
  end
end
```

### 구조체에서 사용

```elixir
defmodule Article do
  @derive {MiniEnc, only: [:id, :title]}
  defstruct [:id, :title, :body, :tags]
end

MiniEnc.dump(%Article{id: 1, title: "Hello", body: "..."} )
# "Elixir.Article<id=1, title=\"Hello\">"

```

이 패턴은

- Ecto schema, Phoenix params, Telemetry event 구조체 등
  “필드 기반 일관 변환”이 필요한 곳에서 아주 자주 쓰인다.

---

## _24.6 실전 종합 예제 — Reportable 프로토콜로 다형 리포팅_

**문제**
주문, 청구서, 사용자 등 서로 다른 도메인 객체를
“보고서 한 줄(row)”로 통일해 출력하고 싶다.

### 프로토콜 정의

```elixir
defprotocol Reportable do
  @doc "보고용 키-값 목록으로 변환"
  @spec to_row(term()) :: [{String.t(), String.t()}]
  def to_row(item)
end
```

### 도메인 타입

```elixir
defmodule Order do
  defstruct [:id, :user_id, :amount, :status]
end

defmodule Invoice do
  defstruct [:no, :customer, :total, :due]
end
```

### 구현

```elixir
defimpl Reportable, for: Order do
  def to_row(%Order{id: id, user_id: u, amount: a, status: s}) do
    [{"type","order"}, {"id","#{id}"}, {"user","#{u}"}, {"amount","#{a}"}, {"status","#{s}"}]
  end
end

defimpl Reportable, for: Invoice do
  def to_row(%Invoice{no: no, customer: c, total: t, due: d}) do
    [{"type","invoice"}, {"no","#{no}"}, {"customer","#{c}"}, {"total","#{t}"}, {"due","#{d}"}]
  end
end

defimpl Reportable, for: Map do
  def to_row(map) do
    map
    |> Enum.map(fn {k, v} -> {to_string(k), to_string(v)} end)
  end
end

defimpl Reportable, for: Any do
  def to_row(x), do: [{"type", "unknown"}, {"value", inspect(x)}]
end
```

### 사용

```elixir
defmodule Reporter do
  def print(items) do
    Enum.each(items, fn it ->
      row = Reportable.to_row(it)
      IO.puts(Enum.map_join(row, " | ", fn {k, v} -> "#{k}=#{v}" end))
    end)
  end
end

Reporter.print([
  %Order{id: 10, user_id: 1, amount: 1200, status: :paid},
  %Invoice{no: "INV-9", customer: "Acme", total: 9000, due: ~D[2025-11-30]},
  %{misc: "ok", n: 3}
])
```

**결과적으로**

- Reporter는 “어떤 타입이 들어오든” 로직이 동일하다.
- 타입별 의미(필드/형식)는 구현에서만 관리된다.
- 보고 라인이 추가되면 **프로토콜 구현만 늘리면 된다.**

---

## _24.7 프로토콜 설계 체크리스트_

- [ ] 이 다형성은 **타입 기준**이 맞나? (모듈 기준이면 behaviour가 더 적합)
- [ ] 첫 번째 인자가 **디스패치 축**이라는 게 자연스러운가?
- [ ] `Any` 폴백이 **도메인 오류를 숨기지 않는가?**
- [ ] 고아 구현을 만들고 있지는 않은가?
- [ ] 릴리즈/플러그인 구조에서 컨솔리데이션 문제가 없는가?
- [ ] 내장 프로토콜(Enumerable/Collectable/Inspect/String.Chars)을 구현해
      “생태계 표준”과 자연스럽게 합쳐지는가?

---

## 결론

- 프로토콜은 **첫 번째 인자 타입 기반 다형성 계약**이다.
- `defprotocol`로 “해야 할 함수”를 정의하고, `defimpl`로 타입별 구현을 붙인다.
- `@derive`/`__deriving__`는 반복 보일러를 제거하는 **컴파일 타임 자동화 장치**다.
- 컨솔리데이션은 성능의 핵심이며, 고아 구현/릴리즈/플러그인 구조에서 특히 주의해야 한다.
- 내장 프로토콜을 제대로 구현하면, **우리 구조체가 Elixir 언어의 원주민처럼** 동작한다.
