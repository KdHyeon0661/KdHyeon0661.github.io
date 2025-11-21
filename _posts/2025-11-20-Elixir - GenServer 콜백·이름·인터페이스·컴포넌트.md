---
layout: post
title: Elixir - GenServer 콜백·이름·인터페이스·컴포넌트
date: 2025-11-20 19:25:23 +0900
category: Elixir
---
# GenServer 콜백·이름·인터페이스·컴포넌트

## GenServer 콜백 — RefServer 레퍼런스

GenServer는 **세 가지 메시지 경로**를 표준화한다.

| 경로  | 함수              | 특성                           | 용도                           |
|-------|-------------------|--------------------------------|--------------------------------|
| call  | `GenServer.call`  | 동기 요청/응답, 호출자 블록   | 읽기/쓰기, 결과가 중요한 작업 |
| cast  | `GenServer.cast`  | 비동기, 응답 없음              | 알림, 로깅, 모니터링          |
| info  | `handle_info/2`   | 일반 메시지·타이머·시그널 처리 | 타이머, 외부 프로세스 메시지   |

### RefServer 전체 코드

우선 초안에 있던 **모든 콜백을 사용하는 레퍼런스 서버**를 그대로 정리한다.
이 모듈 하나에 GenServer 콜백의 “전체 메뉴판”이 다 들어 있다.

```elixir
defmodule RefServer do
  use GenServer

  # ──────────────── Public API ────────────────

  # opts 예:
  #   name: RefServer
  #   table: :ets.new(:my_table, [:set, :private])
  def start_link(opts \\ []) do
    GenServer.start_link(__MODULE__, opts, name: Keyword.get(opts, :name))
  end

  def ping(server) do
    GenServer.call(server, :ping, 2_000)
  end

  def put(server, k, v) do
    GenServer.call(server, {:put, k, v}, 5_000)
  end

  def get(server, k) do
    GenServer.call(server, {:get, k})
  end

  def notify(server, msg) do
    GenServer.cast(server, {:notify, msg})
  end

  def sleep(server) do
    GenServer.cast(server, :sleep)
  end

  def schedule(server, ms) do
    Process.send_after(server, {:tick, System.monotonic_time()}, ms)
  end

  # ──────────────── Callbacks ────────────────

  @impl true
  def init(opts) do
    table =
      Keyword.get_lazy(opts, :table, fn ->
        :ets.new(:ref_kv, [:set, :private])
      end)

    # 서버 부팅을 막지 않도록 무거운 초기화는 :continue로 미룬다.
    {:ok, %{table: table, last_ping: nil}, {:continue, :warmup}}
  end

  @impl true
  def handle_continue(:warmup, st) do
    # 예: 캐시 프리로드, 외부 시스템 핸드셰이크 등
    # 실패 시 크래시 → Supervisor가 재시작
    {:noreply, st}
  end

  @impl true
  def handle_call(:ping, _from, st) do
    now = System.monotonic_time()
    {:reply, {:pong, st.last_ping}, %{st | last_ping: now}}
  end

  @impl true
  def handle_call({:put, k, v}, _from, st) do
    :ets.insert(st.table, {k, v})
    # 다음 외부 입력이 없으면 10초 후 :timeout 신호 수신
    {:reply, :ok, st, 10_000}
  end

  @impl true
  def handle_call({:get, k}, _from, st) do
    reply =
      case :ets.lookup(st.table, k) do
        [{^k, v}] -> {:ok, v}
        [] -> {:error, :not_found}
      end

    {:reply, reply, st}
  end

  @impl true
  def handle_cast({:notify, msg}, st) do
    # 알림 하나마다 Telemetry 이벤트 발행
    :telemetry.execute(
      [:ref_server, :notify],
      %{count: 1},
      %{msg: msg}
    )

    {:noreply, st}
  end

  @impl true
  def handle_cast(:sleep, st) do
    # 다음 메시지까지 휴면. 장수 프로세스에 유리.
    {:noreply, st, :hibernate}
  end

  @impl true
  def handle_info({:tick, ts}, st) do
    :telemetry.execute(
      [:ref_server, :tick],
      %{time: ts},
      %{}
    )

    {:noreply, st}
  end

  @impl true
  def handle_info(:timeout, st) do
    # 비활성 타임아웃 시 주기 작업
    # 예: 오래 동안 put이 없으면 캐시를 비우거나 스냅샷 저장
    {:noreply, st}
  end

  @impl true
  def terminate(reason, st) do
    # 파일/소켓 등 정리 위치 (강제 :kill 시 호출 안 될 수 있음)
    :telemetry.execute(
      [:ref_server, :terminate],
      %{},
      %{reason: reason}
    )

    # 필요시 현재 스택트레이스를 참고해 디버깅 (예: 로그에 함께 남길 때)
    _ = Process.info(self(), :current_stacktrace)
    :ok
  end

  @impl true
  def code_change(_old_vsn, state, _extra) do
    # 핫업그레이드 시 상태 스키마 변환
    {:ok, state}
  end

  # Observer/Logger에서 상태를 보기 좋게 포맷
  @impl true
  def format_status(_opt, [pdict, state]) do
    [
      pdict: pdict,
      summary: %{
        size: :ets.info(state.table, :size),
        last_ping: state.last_ping
      }
    ]
  end
end
```

이 RefServer 하나로, GenServer 콜백의 대부분을 다 보는 셈이다.
이제 콜백별로 **어떤 상황에서 쓰는지**를 예제와 함께 깔끔하게 정리해 보자.

---

### `init/1` + `handle_continue/2` — 부트스트랩과 무거운 초기화 분리

#### `init/1`의 책임

- 최초 상태(state) 생성
- ETS/Agent/하위 프로세스 등 **필수 의존성** 초기화
- 필요 시 **초기 타이머/타임아웃 설정**

RefServer에서:

```elixir
@impl true
def init(opts) do
  table =
    Keyword.get_lazy(opts, :table, fn ->
      :ets.new(:ref_kv, [:set, :private])
    end)

  {:ok, %{table: table, last_ping: nil}, {:continue, :warmup}}
end
```

여기서:

- `table`은 ETS 테이블 핸들.
- 초기 상태는 `%{table: table, last_ping: nil}`.
- 그리고 `{:continue, :warmup}`를 통해 **추가 초기화 단계**를 예약한다.

#### `handle_continue/2` — 초기화의 2단계

```elixir
@impl true
def handle_continue(:warmup, st) do
  # 예: 캐시 프리로드, 외부 서비스와의 초기 동기화
  {:noreply, st}
end
```

이 패턴의 장점:

- Supervisor 입장에서 **init/1이 빠르게 끝나므로 부팅이 지연되지 않는다.**
- warmup 중 예외가 나면 **해당 서버만 크래시** → Supervisor가 재시작.

실전 예:

- Config에서 `:warm_cache` 옵션이 켜져 있으면, `handle_continue/2` 안에서:
  - DB에서 인기 키 100개를 읽어와 ETS에 채워두는 식의 프리로드.

---

### `handle_call/3` — 동기 요청/응답과 역압

`call`은 서버에 **질문을 하고, 대답을 기다리는** 패턴이다.

RefServer의 예:

```elixir
@impl true
def handle_call(:ping, _from, st) do
  now = System.monotonic_time()
  {:reply, {:pong, st.last_ping}, %{st | last_ping: now}}
end

@impl true
def handle_call({:put, k, v}, _from, st) do
  :ets.insert(st.table, {k, v})
  {:reply, :ok, st, 10_000}
end

@impl true
def handle_call({:get, k}, _from, st) do
  reply =
    case :ets.lookup(st.table, k) do
      [{^k, v}] -> {:ok, v}
      [] -> {:error, :not_found}
    end

  {:reply, reply, st}
end
```

핵심:

- `handle_call/3`는 항상 **반드시 `{:reply, reply, new_state}`** 를 반환해야 한다 (혹은 `{:noreply, ...}` + 나중에 `GenServer.reply/2`).
- `call`은 호출한 프로세스를 **블록**하기 때문에 자연스럽게 **역압**을 만든다.
  - 서버가 처리 속도를 따라가지 못하면, 클라이언트 측에 대기열이 생기고 타임아웃이 발생한다.

`{:reply, :ok, st, 10_000}` 처럼 네 번째 인자를 넘기면:

- 이후 10초 동안 외부 메시지가 없다면 `handle_info(:timeout, st)`가 호출된다.
- 이걸 이용해 **비활성 타임아웃**이나 **주기 작업**을 구현할 수 있다.

---

### `handle_cast/2` — 비동기 알림과 큐 관리

`cast`는 “던지고 잊는다” 스타일의 메시지다.

RefServer의 예:

```elixir
@impl true
def handle_cast({:notify, msg}, st) do
  :telemetry.execute(
    [:ref_server, :notify],
    %{count: 1},
    %{msg: msg}
  )

  {:noreply, st}
end

@impl true
def handle_cast(:sleep, st) do
  {:noreply, st, :hibernate}
end
```

특징:

- 클라이언트는 **응답을 기다리지 않는다.**
- 서버의 메일박스에 메시지가 계속 쌓일 수 있으므로,
  고부하 상황에서는 **큐 길이 한도**와 **드롭/거부 정책**을 반드시 생각해야 한다.

큐 한도 예시:

```elixir
@max_q 2_000

@impl true
def handle_cast(msg, st) do
  {:message_queue_len, q} = Process.info(self(), :message_queue_len)

  if q > @max_q do
    # 정책: 드롭 + Telemetry + Logger
    :telemetry.execute([:ref_server, :drop], %{count: 1}, %{msg: msg})
    {:noreply, st}
  else
    handle_cast_internal(msg, st)
  end
end

defp handle_cast_internal({:notify, msg}, st) do
  :telemetry.execute([:ref_server, :notify], %{count: 1}, %{msg: msg})
  {:noreply, st}
end

defp handle_cast_internal(:sleep, st) do
  {:noreply, st, :hibernate}
end
```

이렇게 하면:

- 정상 구간에서는 `handle_cast_internal/2` 로 처리.
- 큐가 너무 길어지면 메시지를 드롭하고, Telemetry로 **알람**을 보낼 수 있다.

---

### `handle_info/2` — 타이머, 일반 메시지, 시스템 시그널

`handle_info/2`는:

- `Process.send/2` 또는 `send/2`로 들어온 일반 메시지,
- `Process.send_after/3`로 예약된 타이머 메시지,
- 비 OTP 메시지 등을 처리하는 데 사용한다.

RefServer에서는:

```elixir
@impl true
def handle_info({:tick, ts}, st) do
  :telemetry.execute(
    [:ref_server, :tick],
    %{time: ts},
    %{}
  )
  {:noreply, st}
end

@impl true
def handle_info(:timeout, st) do
  {:noreply, st}
end
```

여기서 `:timeout`은:

- `handle_call/3`, `handle_cast/2`, `handle_info/2`에서
  `{:noreply, state, timeout_ms}`를 반환했을 때, 그 이후 일정 시간 동안 아무 메시지가 없으면 자동으로 오는 메시지다.

타이머 사용 예:

```elixir
def schedule(server, ms) do
  Process.send_after(server, {:tick, System.monotonic_time()}, ms)
end
```

- 이 메시지는 `handle_info({:tick, ts}, st)`에서 처리된다.
- 주기 작업이나 **배치 flush** 구현에 자주 사용된다.

---

### 종료/정리: `terminate/2` + `try ... after`

RefServer의 `terminate/2`:

```elixir
@impl true
def terminate(reason, st) do
  :telemetry.execute(
    [:ref_server, :terminate],
    %{},
    %{reason: reason}
  )

  _ = Process.info(self(), :current_stacktrace)
  :ok
end
```

주의할 점:

- `terminate/2`는 모든 경우에 호출되는 것은 아니다.
  - 예: `Process.exit(pid, :kill)` 은 비가역 강제 종료라서 `terminate/2`가 호출되지 않는다.
- 따라서 **파일/소켓과 같은 중요한 리소스 정리**는
  가능하면 각 처리 코드 안에서 **`try ... after`** 로 보장하는 것이 먼저다.

예:

```elixir
@impl true
def handle_call({:write, path, data}, _from, st) do
  reply =
    try do
      {:ok, io} = File.open(path, [:append])
      IO.write(io, data)
      :ok
    after
      # io가 있다면 닫기
      File.close(io)
    end

  {:reply, reply, st}
end
```

- 여기서 파일 닫기는 **메시지별로 보장**된다.
- `terminate/2`는 “추가적인 정리/로깅”에 더 가깝다.

---

### 코드 변경: `code_change/3` — 상태 스키마 마이그레이션

핫 업그레이드 환경에서는:

- 예전 버전의 GenServer가 들고 있던 `state`를
- 새 버전의 코드가 이해할 수 있도록 **변환**할 필요가 있다.

RefServer의 기본형:

```elixir
@impl true
def code_change(_old_vsn, state, _extra) do
  {:ok, state}
end
```

실전 예시:

```elixir
@impl true
def code_change({:down, "1.0.0"}, state, _extra) do
  # 1.0.0 → 2.0.0 업그레이드에서 필드를 하나 추가
  new_state =
    case state do
      %{table: t} = s -> Map.put(s, :last_ping, nil)
      s -> s
    end

  {:ok, new_state}
end
```

- 릴리즈 도구(예: `mix release`)와 함께 사용하면 **다운타임 없이** 새 버전으로 교체가 가능하다.
- 상태 스키마를 변경할 때 **명시적 마이그레이션 함수**를 두는 것이 좋다.

---

### 상태 포맷: `format_status/2` — 관찰친화 포맷

`format_status/2`는:

- `:sys.get_status/1`,
- `:observer` GUI 등에서 상태를 보기 좋게 보여주기 위한 콜백이다.

RefServer의 예:

```elixir
@impl true
def format_status(_opt, [pdict, state]) do
  [
    pdict: pdict,
    summary: %{
      size: :ets.info(state.table, :size),
      last_ping: state.last_ping
    }
  ]
end
```

- `pdict`는 프로세스 딕셔너리.
- `summary`에 우리가 보고 싶은 핵심 정보만 정리한다.
- 관찰/디버깅 시에 상태를 직관적으로 이해하는 데 도움이 된다.

---

### 큐 혼잡도 직관: λ, μ, ρ

초안의 수식:

> 큐의 혼잡도 직관: 요청 유입률을 \(\lambda\), 처리율을 \(\mu\)라 하면
> $$\rho = \frac{\lambda}{\mu}$$
> \(\rho \to 1\) 일 때 지연이 급증한다. **`call` 중심 설계(역압)**·**배치**·**샤딩**으로 \(\lambda \le \mu\)를 유지하라.

조금 더 풀어서 설명하면:

- \(\lambda\): 단위 시간당 들어오는 요청 수
- \(\mu\): 단위 시간당 서버가 처리할 수 있는 최대 요청 수
- 사용률 \(\rho = \lambda / \mu\) 가 1에 가까워질수록,
  - 평균 대기시간 \(W\)는 큐잉 이론에서
    대략
    $$W \propto \frac{1}{1 - \rho}$$
    형태로 증가한다(직관용).

따라서:

- **`cast`만 잔뜩 쓰면** \(\lambda\)가 설계 없이 늘어날 수 있다.
- **`call`로 중요한 경로를 감싸고**,
  - 부하가 높을 때는 호출자가 타임아웃/에러를 보고 **속도를 줄인다**.
- 또는:
  - 여러 서버로 **샤딩**(키 해시 → 서버),
  - **배치 처리**(여러 요청을 한 번에)로 \(\mu\)를 키우는 전략이 필요하다.

---

## 프로세스에 이름 짓기 — 로컬, 글로벌, via(Registry)

이름은 **발견(discovery)** 과 **라우팅(routing)** 의 핵심이다.
하나의 GenServer를 다른 프로세스에서 쓰려면, 그 프로세스를 **어떻게 찾을지**를 정해야 한다.

### 이름 전략 개관

| 방식          | 예시                                         | 범위          | 장점                           | 단점                           |
|---------------|----------------------------------------------|---------------|--------------------------------|--------------------------------|
| 로컬 atom     | `name: RefServer`                            | 노드 로컬     | 단순                           | 분산 시 노드 구분 필요         |
| 글로벌        | `name: {:global, :ref}`                      | 클러스터 전체 | 사용하기 쉽다                  | 스케일링·합의 비용             |
| via + Registry| `{:via, Registry, {MyReg, {:kv, 42}}}`       | 노드/클러스터 | 유연한 키·메타데이터·동적 등록 | 구현 복잡도 조금 증가          |

필요에 따라 이 셋을 조합해서 쓴다.

---

### 로컬 이름 — 단일 노드/간단 서비스

```elixir
{:ok, _} = RefServer.start_link(name: RefServer)

RefServer.ping(RefServer)
RefServer.put(RefServer, :a, 1)
```

- 이 방식은 **단일 노드**에선 매우 단순하고 좋다.
- 하지만 다른 노드에서 접근하려면:

```elixir
GenServer.call({RefServer, :"b@host"}, :ping)
```

처럼 `{name, node}` 튜플을 사용해야 한다.

---

### 글로벌 이름 — `:global`

```elixir
{:ok, _} = GenServer.start_link(RefServer, [], name: {:global, :ref})

GenServer.call({:global, :ref}, :ping)
```

특징:

- 같은 쿠키를 가진 노드들로 이루어진 클러스터 전체에서 `:ref` 이름이 **유일**하다.
- 어느 노드에서든 `GenServer.call({:global, :ref}, ...)` 로 접근할 수 있다.
- 내부적으로 전역 등록/합의를 위해 **비용**이 들고,
  전역 이름 갱신이 잦으면 병목이 될 수 있다.

실전 가이드:

- **정말로 “딱 하나만 있어야 하는” 서비스**에 사용.
  - 예: 클러스터 전체의 **리더 프로세스**,
    전체 설정을 관리하는 **중앙 Config 서버** 등.
- 많은 인스턴스를 가지는 **샤드/세션 서버**에는 `:global` 보다는 `Registry` 기반 분산을 쓰는 것이 좋다.

---

### `Registry` + `:via` — 동적·샤딩된 서버에 적합

먼저 Registry와 Supervisor를 애플리케이션에서 띄운다.

```elixir
children = [
  {Registry, keys: :unique, name: MyReg}
]

Supervisor.start_link(children, strategy: :one_for_one)
```

이제 서버에 이름을 붙일 때:

```elixir
{:ok, _} =
  GenServer.start_link(
    RefServer,
    [],
    name: {:via, Registry, {MyReg, {:kv, 42}}}
  )

GenServer.call({:via, Registry, {MyReg, {:kv, 42}}}, {:put, :a, 1})
```

장점:

- 키를 `{MyReg, {:kv, 42}}`처럼 **구조화된 데이터**로 쓸 수 있다.
- Registry에 **메타데이터**를 함께 저장할 수 있어,
  “이 프로세스가 어떤 상태인지”도 함께 관리 가능.
- 고유 키뿐 아니라 **중복 키/다중 구독** 같은 모드도 지원(`keys: :duplicate`).

분산 환경에서는:

- 각 노드에 자기 Registry를 띄우고,
- **라우터 모듈**이 키를 보고 노드를 선택한 후,
  `{via, node}` 튜플로 GenServer를 호출한다.

---

### 분산 주소 표기 — 주소와 노드의 분리

샤딩된 채팅 서버를 예로 들면:

```elixir
defmodule RoomRouter do
  def target_node(room_id, nodes \\ [node() | Node.list()]) do
    idx = :erlang.phash2(room_id, length(nodes))
    Enum.at(nodes, idx)
  end

  def whereis(room_id) do
    addr = {:via, Registry, {MyReg, {:room, room_id}}}
    {addr, target_node(room_id)}
  end
end

# 사용

{addr, nod} = RoomRouter.whereis(42)
GenServer.call({addr, nod}, {:post, "hello"})
```

주소 구조:

```elixir
{ {:via, Registry, {MyReg, {:room, 42}}}, :"b@host" }
```

- **앞부분**은 노드 안에서 프로세스를 찾는 방법(via).
- **뒤부분**은 어떤 노드에 있는지.

이렇게 **주소와 노드를 분리**하면:

- 샤딩 방식(해시, 일관 해시, 룰 기반)을 바꿀 때
  라우터만 고치면 된다.
- 프로세스 코드(GenServer 구현)는 라우팅 정책을 몰라도 된다.

---

## 인터페이스 다듬기 — 메시지 튜플을 숨기고, API를 드러내기

GenServer 자체는 “메시지 처리기”일 뿐이고,
외부에 보여줄 것은 **함수 기반의 퍼블릭 API**가 되어야 한다.

### 메시지 감추기 + 타입 명세

초안의 예:

```elixir
defmodule KV do
  @moduledoc "KV Server public API"

  @type key :: term()
  @type value :: term()
  @type server :: GenServer.server()
  @default_timeout 5_000

  @spec put(server(), key(), value(), timeout()) :: :ok
  def put(server, k, v, t \\ @default_timeout) do
    validate_key!(k)
    GenServer.call(server, {:put, k, v}, t)
  end

  @spec get(server(), key(), timeout()) :: {:ok, value()} | {:error, :not_found}
  def get(server, k, t \\ @default_timeout) do
    validate_key!(k)
    GenServer.call(server, {:get, k}, t)
  end

  defp validate_key!(k) do
    if is_nil(k) do
      raise ArgumentError, "key cannot be nil"
    end
  end
end
```

설명:

- 사용자는 `{:put, k, v}` 같은 내부 메시지를 몰라도 된다.
- 함수는:
  - 인자 검증 (`validate_key!/1`),
  - 타임아웃 지정 (`t \\ @default_timeout`),
  - 결과 형식(성공/실패 튜플)을 **계약**으로 제공한다.
- 나중에 GenServer 내부 메시지 포맷을 바꾸더라도,
  이 API 계층만 유지하면 **호출자는 그대로** 유지된다.

---

### 동기/비동기 경계 설계

설계 원칙 하나:

- **상태를 바꾸는 작업(쓰기/수정)** 은 대부분 **동기 `call`** 로 두어:
  - 실패 전파,
  - 역압,
  - 순서 보장 등을 확보한다.
- **로그/메트릭/보조 알림**은 **비동기 `cast`** 로 해도 좋지만,
  큐 한도와 실패 전략을 명확히 하고 간다.

초안에 있던 “큐 길이 확인 후 cast” 예시를 약간 확장해 보자.

```elixir
defmodule QueueAware do
  @max_q 1_000

  def notify(server, payload) do
    case mailbox(server) do
      :unknown ->
        # 원격 노드 정보 부족 등
        {:error, :unknown_mailbox}

      q when q > @max_q ->
        {:error, :busy}

      _ ->
        GenServer.cast(server, {:notify, payload})
        :ok
    end
  end

  defp mailbox(pid) do
    # 원격 노드에서도 Process.info를 호출하기 위해 :rpc 사용
    case :rpc.call(node(pid), Process, :info, [pid, :message_queue_len]) do
      {:message_queue_len, q} -> q
      _ -> :unknown
    end
  end
end
```

이제 `notify/2`를 쓰는 쪽에서는:

- `{:error, :busy}` 를 보고 **재시도/드롭** 정책을 결정할 수 있다.

---

### 에러 정책 — 예외 vs 태그드 튜플

일반적인 규칙:

- **사용자 오용**(인자 타입 오류, 명세 위반) → 예외(`raise`)
- **운영 환경의 실패**(네트워크, 디스크, 외부 시스템 오류) → 태그드 튜플 `{:error, reason}`

예:

```elixir
defmodule KVClient do
  @spec get_positive(KV.server(), KV.key()) :: {:ok, integer()} | {:error, term()}
  def get_positive(server, k) do
    with {:ok, v} <- KV.get(server, k),
         true <- is_integer(v) or {:error, :not_integer},
         true <- v > 0 or {:error, :non_positive}
    do
      {:ok, v}
    end
  end
end
```

- 내부에서 `KV.get/3`이 실패하면 그대로 `{:error, reason}`으로 위로 올라간다.
- `with`를 쓰면 실패 흐름을 한 눈에 볼 수 있다.

---

### 타임아웃을 API 표면으로 노출

타임아웃은 **호출자 문맥**에 따라 달라야 한다.

- 웹 요청 핸들러 → 수백 ms 내에 응답해야 할 수도 있고,
- 배치 작업 → 수 초~수 분까지 기다릴 수도 있다.

따라서:

- API에 `timeout` 인자를 기본값과 함께 제공해 두는 것이 좋다.
- 시스템 전역 설정(ex. Application config)에서 기본값을 주고,
  특정 호출에서만 오버라이드하도록 설계.

```elixir
@default_timeout Application.compile_env(:my_app, :kv_timeout, 5_000)

def put(server, k, v, timeout \\ @default_timeout) do
  GenServer.call(server, {:put, k, v}, timeout)
end
```

---

### 배치 API — round-trip·락 비용 줄이기

여러 개의 put/get을 한 번에 처리할 수 있는 API를 제공하는 것도 중요하다.

```elixir
@spec put_many(server(), [{key(), value()}], timeout()) :: :ok
def put_many(server, pairs, timeout \\ @default_timeout) do
  GenServer.call(server, {:put_many, pairs}, timeout)
end
```

GenServer 쪽은:

```elixir
@impl true
def handle_call({:put_many, pairs}, _from, st) do
  Enum.each(pairs, fn {k, v} ->
    :ets.insert(st.table, {k, v})
  end)

  {:reply, :ok, st}
end
```

효과:

- call 한 번에 여러 개의 내부 작업을 묶어 처리할 수 있어
  **왕복 지연**과 **락 횟수**를 줄인다.

---

### 멱등성과 재시도 — idempotency key

네트워크/분산 환경에서는 재시도를 고려해야 한다.

- 같은 요청이 **여러 번 들어와도** 상태가 한 번만 적용되도록 하는 것이 멱등성이다.

간단 예:

```elixir
defmodule IdempotentKV do
  use GenServer

  def start_link(opts), do: GenServer.start_link(__MODULE__, opts, name: Keyword.get(opts, :name))

  def put_once(server, idempotency_key, k, v) do
    GenServer.call(server, {:put_once, idempotency_key, k, v})
  end

  @impl true
  def init(_opts) do
    {:ok, %{table: :ets.new(:idem_kv, [:set, :private]), done: MapSet.new()}}
  end

  @impl true
  def handle_call({:put_once, key, k, v}, _from, st) do
    if MapSet.member?(st.done, key) do
      {:reply, :already_done, st}
    else
      :ets.insert(st.table, {k, v})
      {:reply, :ok, %{st | done: MapSet.put(st.done, key)}}
    end
  end
end
```

- 외부에서 `idempotency_key`를 생성해서 전달하면,
- 네트워크 재시도·중복 호출에도 **한 번만** 실행된다.

---

### Telemetry로 API 계층 계측

API 함수에 Telemetry span을 씌우면:

```elixir
def get(server, k, timeout \\ @default_timeout) do
  :telemetry.span([:kv, :get], %{key: k}, fn ->
    reply = GenServer.call(server, {:get, k}, timeout)
    {reply, %{ok?: match?({:ok, _}, reply)}}
  end)
end
```

리스너는:

- `[:kv, :get, :stop]` 이벤트에서 duration, ok? 여부, key 등을 보고,
- 지연/에러율/핫 키 등을 분석할 수 있다.

---

## 서버를 컴포넌트로 만들기 — 수명주기, 관찰, 테스트까지

이제 GenServer 하나를 **컴포넌트 단위**로 본다.

> 컴포넌트 = (1) 독립적인 수명주기, (2) 명확한 인터페이스,
> (3) 관찰 가능성, (4) 테스트 용이성

### `child_spec/1`과 슈퍼비전 트리 통합

GenServer를 컴포넌트처럼 쓰려면,
슈퍼비전 트리에 올릴 때 **child_spec**을 명시하는 것이 좋다.

```elixir
defmodule KV.Server do
  use GenServer

  def start_link(opts) do
    GenServer.start_link(__MODULE__, opts, Keyword.take(opts, [:name]))
  end

  @impl true
  def init(opts) do
    {:ok, %{table: Keyword.get(opts, :table, :ets.new(:kv, [:set, :private]))}}
  end

  @impl true
  def child_spec(opts) do
    %{
      id: Keyword.get(opts, :id, __MODULE__),
      start: {__MODULE__, :start_link, [opts]},
      restart: :permanent,
      shutdown: 10_000,
      type: :worker
    }
  end
end
```

애플리케이션에서:

```elixir
children = [
  {Registry, keys: :unique, name: MyReg},
  {KV.Server, name: {:via, Registry, {MyReg, {:kv, 1}}}},
  {DynamicSupervisor, name: KV.Dynamic, strategy: :one_for_one}
]

Supervisor.start_link(children, strategy: :one_for_one)
```

- 이 구조를 쓰면, **한 앱 안에 같은 GenServer 구현을 여러 인스턴스로 띄우는 것**이 자연스러워진다.

---

### 구성과 의존성 주입 — Behaviour를 통한 추상화

외부 시스템 의존성(파일, HTTP, 메일 등)은
**Behaviour 인터페이스**로 추상화하고, 모듈을 주입하는 방식이 좋다.

초안의 예를 확장해 보자.

```elixir
defmodule Storage do
  @callback write(term()) :: :ok | {:error, term()}
end

defmodule FileStorage do
  @behaviour Storage

  @impl true
  def write(x) do
    File.write!("out.log", inspect(x) <> "\n")
    :ok
  rescue
    e -> {:error, e}
  end
end

defmodule CompServer do
  use GenServer

  def start_link(opts) do
    GenServer.start_link(__MODULE__, opts, name: Keyword.get(opts, :name))
  end

  @impl true
  def init(opts) do
    {:ok, %{storage: Keyword.fetch!(opts, :storage)}}
  end

  @impl true
  def handle_call({:persist, data}, _from, st) do
    reply = st.storage.write(data)
    {:reply, reply, st}
  end
end
```

실행:

```elixir
{:ok, _} = CompServer.start_link(name: CompServer, storage: FileStorage)
CompServer |> GenServer.call({:persist, 123})
```

테스트 시:

- `StorageMock`을 정의하고,
- `CompServer`를 그 mock과 함께 띄우면 된다.

---

### 문서화 — @moduledoc, @doc, Doctest

컴포넌트의 **표면(퍼블릭 API)** 는 문서로 명확히 해야 한다.

```elixir
defmodule KV do
  @moduledoc """
  키-값 저장소 서버의 퍼블릭 API.

  예:

      iex> {:ok, pid} = KV.Server.start_link([])
      iex> KV.put(pid, :a, 1)
      :ok
      iex> KV.get(pid, :a)
      {:ok, 1}

  """

  @doc """
  키에 값을 저장한다.

  키가 nil이면 ArgumentError를 발생시킨다.
  """
  @spec put(KV.Server.server(), term(), term()) :: :ok
  def put(server, k, v) do
    if is_nil(k), do: raise ArgumentError, "key cannot be nil"
    GenServer.call(server, {:put, k, v})
  end
end
```

- 이 예시는 Doctest로도 바로 사용 가능하다.
- 내부 모듈은 `@moduledoc false` 로 숨기면 API 표면이 깔끔해진다.

---

### 관찰 가능성 — Telemetry 이벤트 계약

컴포넌트는 **자기가 내보내는 Telemetry 이벤트**를 문서화해야 한다.

예:

- 이벤트 이름: `[:kv, :put, :stop]`
- 측정값(measurements): `%{duration: native_time}`
- 메타데이터(metadata): `%{key: key, ok?: boolean}`

코드:

```elixir
:telemetry.span([:kv, :put], %{key: k}, fn ->
  t0 = System.monotonic_time()
  reply = GenServer.call(server, {:put, k, v})
  duration = System.monotonic_time() - t0
  {reply, %{duration: duration, ok?: match?(:ok, reply)}}
end)
```

리스너:

```elixir
:telemetry.attach(
  "kv-put-logger",
  [:kv, :put, :stop],
  fn _event, meas, meta, _config ->
    require Logger
    Logger.info("kv.put key=#{inspect(meta.key)} duration=#{meas.duration} ok=#{meta.ok?}")
  end,
  nil
)
```

이렇게 하면:

- 나중에 Prometheus/LiveDashboard 등 어떤 관찰 도구를 써도
  이 이벤트를 **중심으로 지표를 구성** 할 수 있다.

---

### 페일-세이프 — 종료/복구 전략

컴포넌트 설계 시 체크할 것:

1. **리소스 정리**
   - 파일/소켓/외부 커넥션은 `try ... after` + `terminate/2`로 두 번 보호.
2. **슈퍼비전 전략**
   - 이 서버가 죽었을 때 **누가 영향을 받을지** 기준으로
     `:one_for_one`, `:rest_for_one`, `:one_for_all`을 고른다.
3. **부하·에러 폭풍 대응**
   - 큐 한도, 배치, 샤딩, 백오프 재시도, 서킷 브레이커.

예: 서킷 브레이커 느낌의 상태 머신은 `GenStateMachine` 으로 구현할 수도 있고,
간단하게는 GenServer 상태에 `:open | :half_open | :closed` 필드를 두고 관리할 수도 있다.

---

### 패키징 — Umbrella, 독립 앱, Hex

프로젝트 규모가 커질수록:

- 공통 컴포넌트(예: KV, Transform, Cache)는 **Umbrella 앱의 별도 프로젝트**로 분리하는 것이 좋다.
- 한 회사 내 여러 서비스에서 재사용해야 한다면 Hex 프라이빗 레지스트리에 올려,
  - 버전 관리(semver),
  - Telemetry 이벤트 스펙,
  - 구성 옵션 등을 문서화하고 공유한다.

---

## 전체 예제 — “지연 변환 + 캐시 + 관찰” 컴포넌트

마지막으로 초안의 **Transform 컴포넌트 예제**를 다시 정리하고 약간 확장한다.

### 요구

- 입력 enum(예: `1..5`)을 받아 **함수로 변환**한 뒤,
- 한 줄씩 파일에 쓰는 서버.
- I/O 구현(Storage)은 Behaviour로 추상화.
- Telemetry를 통해 `:transform, :run` 이벤트를 발행.
- 테스트에서는 가짜 Storage(mock)를 주입.

### Storage Behaviour와 구현

```elixir
defmodule Transform.Storage do
  @callback open(path :: binary()) :: {:ok, term()} | {:error, term()}
  @callback write(io :: term(), data :: iodata()) :: :ok | {:error, term()}
  @callback close(io :: term()) :: :ok
end

defmodule Transform.FileStorage do
  @behaviour Transform.Storage

  @impl true
  def open(path) do
    {:ok, File.open!(path, [:write])}
  rescue
    e -> {:error, e}
  end

  @impl true
  def write(io, data) do
    IO.binwrite(io, data)
    :ok
  rescue
    e -> {:error, e}
  end

  @impl true
  def close(io) do
    File.close(io)
  end
end
```

### Transform.Server 구현

{% raw %}
```elixir
defmodule Transform.Server do
  use GenServer
  @moduledoc "Stream 변환을 배치로 파일에 쓰는 컴포넌트"

  # Public API
  def start_link(opts) do
    GenServer.start_link(__MODULE__, opts, name: Keyword.get(opts, :name))
  end

  @doc """
  enum의 각 원소에 fun을 적용해 한 줄씩 path에 쓴다.
  """
  def run(server, enum, path, fun, timeout \\ 60_000) do
    GenServer.call(server, {:run, enum, path, fun}, timeout)
  end

  # Callbacks
  @impl true
  def init(opts) do
    storage = Keyword.fetch!(opts, :storage)
    {:ok, %{storage: storage}}
  end

  @impl true
  def handle_call({:run, enum, path, fun}, _from, st) do
    storage = st.storage

    reply =
      :telemetry.span([:transform, :run], %{path: path}, fn ->
        case storage.open(path) do
          {:ok, io} ->
            try do
              # 옵션: 카운트를 프로세스 딕셔너리에 기록
              :erlang.put(:count, 0)

              enum
              |> Stream.map(fun)
              |> Stream.each(fn line ->
                :ok = storage.write(io, [line, ?\n])
                :erlang.put(:count, (:erlang.get(:count) || 0) + 1)
              end)
              |> Stream.run()

              count = :erlang.get(:count) || 0
              {{:ok, count}, %{count: count}}
            after
              storage.close(io)
            end

          {:error, reason} ->
            {{:error, reason}, %{count: 0, error: reason}}
        end
      end)

    {:reply, elem(reply, 0), st}
  end
end
```
{% endraw %}

### 사용 예

```elixir
children = [
  {Transform.Server, name: Transform, storage: Transform.FileStorage}
]

Supervisor.start_link(children, strategy: :one_for_one)

{:ok, count} =
  case Transform.Server.run(
         Transform,
         1..5,
         "out.txt",
         fn x -> Integer.to_string(x * x) end
       ) do
    {:ok, c} -> {:ok, c}
    {:error, reason} -> raise "failed: #{inspect(reason)}"
  end
```

이 코드는 `out.txt`에:

```text
1
4
9
16
25
```

를 쓰고, Telemetry 이벤트를 통해:

- 변환에 걸린 전체 시간,
- 몇 줄 썼는지,
- 에러 여부 등을 관찰할 수 있다.

---

### 테스트 — Storage 목으로 검증

```elixir
# test_helper.exs 등에서

Mox.defmock(TStorage, for: Transform.Storage)

defmodule TransformServerTest do
  use ExUnit.Case, async: true

  setup do
    TStorage
    |> Mox.expect(:open, fn "ignored" -> {:ok, self()} end)
    |> Mox.expect(:write, 5, fn _io, data ->
      send(self(), {:written, data})
      :ok
    end)
    |> Mox.expect(:close, fn _io -> :ok end)

    {:ok, _} = Transform.Server.start_link(name: TransformTest, storage: TStorage)
    :ok
  end

  test "run writes transformed lines" do
    assert {:ok, 5} ==
             Transform.Server.run(
               TransformTest,
               1..5,
               "ignored",
               &Integer.to_string/1
             )

    # 메일박스로 결과 확인
    for _ <- 1..5 do
      assert_receive {:written, data}
      assert String.trim(to_string(data)) =~ ~r/^\d+$/
    end
  end
end
```

이렇게:

- 외부 I/O 없이도 Transform.Server의 로직을 **완전히 검증**할 수 있다.

---

## 흔한 함정 → 교정

초안에 있던 항목들을 확장해서 정리하면:

1. **`cast` 남용으로 큐 폭주**

   - 증상: CPU는 놀고 있는데, 특정 서버의 `message_queue_len` 이 수만 단위로 증가.
   - 교정:
     - 쓰기/변경 연산은 원칙적으로 `call` 사용.
     - `cast`는 큐 한도 + 드롭/경고 정책과 함께.
     - 부수적인 로그/메트릭/알림만 `cast`로.

2. **메시지 튜플을 외부에 노출**

   - 증상: 외부 코드가 `GenServer.call(pid, {:put, :a, 1})` 같은 튜플에 의존.
   - 교정:
     - 항상 퍼블릭 API 모듈(예: `KV`)을 두고, 내부 메시지는 감춘다.
     - 메시지 포맷이 바뀌어도 API가 바뀌지 않도록 설계.

3. **무거운 init으로 부팅 지연**

   - 증상: Application 시작이 수 초 이상 걸림. Supervisors가 Timeouts.
   - 교정:
     - `init/1`에서는 최소한의 작업만 하고,
     - 나머지는 `{:continue, step}` + `handle_continue/2`로 넘김.

4. **리소스 정리 누락**

   - 증상: 파일 핸들/소켓 누수, 재시작 후에도 핸들이 남아 있음.
   - 교정:
     - 모든 I/O 경로에 `try ... after`를 사용.
     - `terminate/2`에서는 “최후 방어선 + 로그” 역할.

5. **관찰 부재**

   - 증상: 장애 시 “무슨 일이 있었는지” 알 수 없음.
   - 교정:
     - Telemetry 이벤트 네이밍 규칙 정의 (예: `[:comp, :op, :stop]`).
     - 주요 경로(call, 배치 flush, 에러 등)에 span/execute 삽입.
     - LiveDashboard/로그/메트릭 시스템과 연결.

---

## 연습 문제

1. **API 경계 재설계**
   - 기존 서버 구현에서 모든 `cast` 호출을 조사하여,
     - 진짜 알림용 메시지만 남기고
     - 나머지는 `call`로 옮겨라.
   - 큐 길이 한도와 타임아웃을 점검하는 테스트를 작성하라.

2. **TTL 캐시**
   - `KV.Server`에 키별 TTL을 도입하라.
   - 각 키마다 타이머를 하나씩 두지 말고, 1초/10초 단위 **버킷**으로 묶어
     타이머 수를 줄이는 설계를 시도해 보라.

3. **샤딩 라우터**
   - Registry + 해시 기반 라우터를 구현하여
     `room_id` 나 `user_id`를 기준으로 노드/프로세스를 선택하고,
     호스트 추가/제거 시 재배치 전략(일관 해시 등)을 설계하라.

4. **Telemetry 대시보드**
   - `[:kv, :get]`, `[:kv, :put]`에 대해 duration histogram과 ok/error 카운터를 노출하고,
     임계값(예: 95퍼센타일 지연이 200ms 초과) 이상일 때 로그 경고를 출력하라.

5. **핫 코드 변경**
   - `KV.Server`의 상태 구조체에 새로운 필드를 추가하고,
     `code_change/3`에서 예전 버전의 state를 새 버전으로 마이그레이션하는 코드를 작성하라.
   - 구 버전 state를 만든 뒤 새 버전 `code_change/3`을 호출하는 테스트를 작성해 보라.

---

## 마무리

- **GenServer 콜백**은 단순히 “콜백 몇 개”가 아니라,
  **요청/응답·비동기 알림·타이머·초기화·종료·코드 변경**까지 다루는 **서버 수명주기 전체 모델**이다.
- **이름 전략**(로컬/글로벌/via)과 **라우팅(키→노드)** 을 잘 설계하면,
  단일 서버에서 시작한 코드가 자연스럽게 **분산 클러스터**로 확장된다.
- **퍼블릭 API**를 함수/타입/타임아웃/에러 규약으로 명확히 정의하고,
  내부 메시지/상태는 컴포넌트 안에 캡슐화하라.
- **컴포넌트화**는 수명주기, 관찰 가능성, 테스트 용이성을 기본값으로 만든다.
  이런 컴포넌트 몇 개를 안정적으로 쌓아 올리면,
  나머지는 **슈퍼비전 트리와 라우터**가 자연스럽게 전체 시스템을 지탱해 준다.
