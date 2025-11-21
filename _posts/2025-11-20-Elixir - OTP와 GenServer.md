---
layout: post
title: Elixir - OTP와 GenServer
date: 2025-11-20 18:25:23 +0900
category: Elixir
---
# OTP와 GenServer

## OTP가 제공하는 것

### OTP란 무엇인가 — “라이브러리” + “설계 규범”

OTP(Open Telecom Platform)는 원래 **통신 시스템**을 위해 만들어진 얼랭/BEAM 생태계의 표준 프레임워크였지만, 오늘날에는 다음 두 가지 의미가 더 크다.

1. **라이브러리 집합**
   - `gen_server`, `supervisor`, `gen_statem`, `application`, `logger`, `sys`, `rpc`, `gen_tcp` 같은 **표준 모듈/행위(behaviour)**.
2. **설계 원칙**
   - “프로세스는 짧고, 잘 격리하고, 실패는 **슈퍼비전 트리**로 복구한다.”
   - “상태는 **프로세스 내부에 캡슐화**하고, 동시성은 메시지로 다룬다.”
   - “코드를 올리는 방식, 릴리즈/업그레이드, 운영 관찰까지 **일관된 규칙**을 제공한다.”

이 장의 초안에서 OTP는 다음 **다섯 축**으로 요약되어 있었다.

1. **행위(Behaviours)** — 콜백 기반 프레임워크
2. **배치/부트스트랩(Application)** — 앱 수명주기 & 구조
3. **슈퍼비전(Supervision)** — 실패 격리 & 자동 복구
4. **관찰(Observability)** — 로깅·텔레메트리·트레이싱 훅
5. **분산/배포(Distribution/Releases)** — 노드·원격 호출·릴리즈

이제 각 축을 조금 더 깊게 파고든다.

---

### 행위(Behaviours) — “콜백만 구현하면 된다”

행위(behaviour)는 **콜백 기반 프레임워크**다.
패턴은 항상 같다:

- “**프레임워크가 제어 흐름을 가진다**”
- “개발자는 **콜백 함수**만 구현한다”

대표적인 것들:

- `GenServer` — 상태를 가진 서버 프로세스
- `Supervisor` — 슈퍼비전 트리 관리
- `GenStateMachine` (`:gen_statem`) — 상태 머신
- `Application` — 애플리케이션 수명주기
- `Task` — 일회성 작업
- 역사적으로 `GenEvent` — 지금은 `GenStage` 등으로 대체

간단한 비교 표:

| Behaviour       | 목적                              | 특징 요약                        |
|----------------|-----------------------------------|----------------------------------|
| `GenServer`    | 상태 가진 서버                     | call/cast/info, 표준 라이프사이클 |
| `Supervisor`   | 자식 프로세스 복구                | 재시작 전략, 강도, 기간          |
| `GenStateMachine` | 상태기계 구현                  | 상태별 콜백, 이벤트 기반 전이    |
| `Application`  | 앱 부트/정지                      | `start/2`, `stop/1`              |
| `Task`         | 일회성 비동기 작업                | async/await, Task.Supervisor     |

핵심 아이디어는 초안에 쓴 것 그대로다.

> “**콜백을 구현하면 프레임워크가 나머지를 해준다**”

GenServer를 예로 들면:

- 메시지 디코딩, 응답을 돌려줄 `reply`, 타임아웃, 링크/모니터, 시스템 메시지 처리 등은
  **OTP 프레임워크가 다 한다.**
- 개발자는 오직 `init/1`, `handle_call/3`, `handle_cast/2`, `handle_info/2` 같은
  **콜백 함수만 구현**한다.

---

### 배치/부트스트랩(Application) — “앱의 수명주기를 정의”

`mix new my_app --sup` 을 하면 기본으로 **OTP Application 템플릿**이 만들어진다.

```bash
mix new my_app --sup
```

생성되는 핵심 파일:

```text
lib/my_app/application.ex
```

내용 예:

```elixir
defmodule MyApp.Application do
  use Application

  @impl true
  def start(_type, _args) do
    children = [
      MyApp.Repo,
      MyAppWeb.Endpoint,
      {MyApp.Worker, []}
    ]

    opts = [strategy: :one_for_one, name: MyApp.Supervisor]
    Supervisor.start_link(children, opts)
  end

  @impl true
  def stop(_state) do
    :ok
  end
end
```

여기서 Application이 제공하는 것:

- **앱 수명주기**:
  - BEAM VM이 이 앱을 시작할 때 `start/2` 호출
  - 종료 시 `stop/1` 호출
- **슈퍼비전 트리 루트**:
  - `start/2`에서 트리의 루트 Supervisor를 띄우고 그 아래에 모든 서버/워커를 매단다.
- “무엇을 먼저 띄우고, 실패 시 어떻게 다시 띄울지”를 **선언적**으로 명시.

이 레벨에서 이미 중요한 규칙이 나온다:

- **모든 중요한 프로세스는 Supervisor 트리 아래에 있어야 한다.**
- 즉, “까먹고 그냥 spawn한 프로세스”는 운영적으로 **리스**처럼 남는다.

---

### 슈퍼비전(Supervision) — 실패 격리 및 자동 복구

OTP의 설계 철학은 자주 인용되는 문장으로 요약된다.

> “Let it crash” + “Supervision trees”

즉:

- 개별 프로세스는 **죽어도 좋다**.
- 중요한 건, **누가, 어떤 전략으로 다시 띄워줄 것인가**이다.

슈퍼바이저(Supervisor)가 하는 일:

- 자식 프로세스를 **링크(link)** 하고,
- 자식이 죽었을 때 **재시작 전략**에 따라 다시 띄우거나,
- 일정 횟수 이상 죽으면 **자기도 죽어 상위 Supervisor에 문제를 위임**.

대표 전략:

- `:one_for_one`
  - 죽은 자식만 다시 시작.
- `:rest_for_one`
  - 죽은 자식과 그 **뒤에 선언된 자식**까지 함께 재시작.
- `:one_for_all`
  - 한 자식이 죽으면 **전부 내렸다가 다시 올림**.
- `DynamicSupervisor`
  - 런타임에 동적으로 자식을 추가/삭제.

슈퍼비전 트리는 계층적으로 쌓인다:

```text
MyApp.Supervisor
├─ MyApp.Repo
├─ MyAppWeb.Endpoint
└─ MyApp.WorkerSupervisor
   ├─ Worker1
   ├─ Worker2
   └─ ...
```

이 트리 구조 덕에 **실패가 위로 전파되며, 특정 부분만 재시작**할 수 있다.

---

### 관찰(Observability) — Logger, Telemetry, 모니터링

OTP/Elixir는 **관찰 가능한 시스템**을 만드는 걸 강하게 밀어붙인다.

- `Logger`
  - 로그 레벨, 메타데이터, 백엔드(콘솔, 파일, 원격) 설정 가능.
- `Telemetry`
  - 라이브러리/애플리케이션이 **이벤트를 발행**하고,
    외부에서 **리스너를 붙여** 메트릭/트레이싱/로깅으로 연결.
- `:observer`
  - GUI 기반 프로세스/메모리/ETS/포트 관찰.
- `recon` (Erlang 라이브러리)
  - 라이브에서 핫 프로파일링, “어떤 프로세스가 GC를 많이 하는가?” 등.
- Phoenix LiveDashboard
  - 웹 기반 대시보드(텔레메트리, ETS, 메모리, Ecto 등 시각화).

GenServer에 Telemetry를 심는 패턴은 뒤에서 다시 볼 것이다.

핵심은:

- **운영 지표와 로그는 처음부터 설계의 일부**로 취급해야 한다.
- 나중에 “문제가 생기면 로그 추가” 식으로는 늦다.

---

### 분산/배포(Distribution/Releases)

16장에서 이미 노드·쿠키·RPC·원격 태스크를 다뤘다.
OTP는 여기에 **릴리즈 시스템**까지 제공한다.

- 노드 이름/쿠키 설정
- `:rpc` 모듈로 원격 함수 호출
- `Task.Supervisor` 로 원격 태스크 관리
- `mix release` / `rebar3` 기반 릴리즈
  - 프로덕션용 self-contained 실행 디렉터리 생성
  - 시스템ctl 등과 통합하기 쉬움

핵심은:

- 개발할 때 사용한 **동일한 추상화**(프로세스/메시지/GenServer/슈퍼비전)를
  여러 노드의 **클러스터 전체**에 그대로 적용할 수 있다는 점이다.

---

### 다섯 축 요약

초안의 요약을 다시 정리하면 다음과 같다.

1. **행위(Behaviours)** — 콜백 기반 프레임워크
   - `GenServer`, `Supervisor`, `GenStateMachine`, `Application`, `Task` …
2. **배치/부트스트랩(Application)** — 앱 수명주기 & 구조
   - `mix new --sup`가 만드는 Application 콜백, 슈퍼비전 트리 루트.
3. **슈퍼비전(Supervision)** — 실패 격리 & 자동 복구
   - 전략(`:one_for_one`, `:rest_for_one`, `:one_for_all`, `DynamicSupervisor`).
4. **관찰(Observability)** — 로깅·텔레메트리·트레이싱 훅
   - `Logger`, `Telemetry`, `:observer`, LiveDashboard 등.
5. **분산/배포(Distribution/Releases)** — 노드·원격 호출·릴리즈
   - 노드 이름/쿠키, `:rpc`, 원격 태스크, 릴리즈 기반 배포.

그리고 이 모든 축의 “사용자 진입점”이 곧 **Behaviour**이며,
그 중 **서버의 기본형**이 **GenServer**다.

---

## OTP 서버(GenServer)

> **GenServer = 상태를 가진 서버 프로세스 + 표준 메시지 프로토콜(call/cast/info)**
> 여러분은 **콜백**만 작성하면 된다. 나머지(링킹·모니터링·타임아웃·reply 등)는 프레임워크가 처리한다.

---

### 최소 예제: Echo 서버

초안에 있던 예제를 그대로 가져와 조금 더 풀어쓴다.

```elixir
defmodule EchoServer do
  use GenServer

  # --- Public API ---
  def start_link(opts \\ []) do
    GenServer.start_link(__MODULE__, :ok, opts)
  end

  def ping(pid \\ __MODULE__) do
    GenServer.call(pid, :ping)
  end

  def notify(pid \\ __MODULE__, msg) do
    GenServer.cast(pid, {:notify, msg})
  end

  # --- Callbacks ---
  @impl true
  def init(:ok) do
    {:ok, %{count: 0}}  # 초기 상태
  end

  @impl true
  def handle_call(:ping, _from, state) do
    {:reply, {:pong, state.count}, %{state | count: state.count + 1}}
  end

  @impl true
  def handle_cast({:notify, msg}, state) do
    IO.puts("notify: #{msg}")
    {:noreply, state}
  end
end

# 시작 (보통은 Supervisor가 대신 호출)

{:ok, _} = EchoServer.start_link(name: EchoServer)
EchoServer.ping()          # {:pong, 0}
EchoServer.notify("hello") # 콘솔에 출력
EchoServer.ping()          # {:pong, 1}
```

중요한 포인트:

- **동기 요청**: `GenServer.call/3`
  - 호출자는 응답을 받을 때까지 **블록**된다.
  - 자연스럽게 **역압**(backpressure)을 만든다.
- **비동기 알림**: `GenServer.cast/2`
  - 응답이 없다. 메일박스에 메시지가 **쌓일 수 있다**.
  - 고부하 환경에서는 **큐 길이 측정 + 정책**이 필수다.
- **상태**는 `state` 인자로 계속 전달되는 **불변 자료구조(Map)**.
  - 동일 프로세스 안에서만 공유되므로,
    락 락킹, 레이스 조건을 직접 걱정할 필요가 거의 없다.

---

### 상태 모델링: 얕은 Map에서 구조체로

상태가 복잡해지면 단순한 `%{}` 대신 **구조체(struct)** 를 쓰는 것이 좋다.

초안에서 제시한 예:

```elixir
defmodule KV.Server do
  use GenServer

  defmodule State do
    @enforce_keys [:data]
    defstruct data: %{}, ttl: :infinity, writes: 0
  end

  alias __MODULE__.State

  def start_link(opts \\ []) do
    GenServer.start_link(__MODULE__, :ok, opts)
  end

  def get(pid, key), do: GenServer.call(pid, {:get, key})
  def put(pid, key, val), do: GenServer.call(pid, {:put, key, val})

  @impl true
  def init(:ok), do: {:ok, %State{}}

  @impl true
  def handle_call({:get, key}, _from, %State{data: m}=st) do
    {:reply, Map.get(m, key), st}
  end

  @impl true
  def handle_call({:put, k, v}, _from, %State{data: m, writes: w}=st) do
    st = %State{st | data: Map.put(m, k, v), writes: w + 1}
    {:reply, :ok, st}
  end
end
```

장점:

- 구조체에 필드를 명시하면,
  - 상태에 어떤 필드가 있는지 한 눈에 보이고,
  - 패턴 매칭 시 **오타/필드 누락**을 더 쉽게 탐지할 수 있다.
- `@enforce_keys` 를 사용하면 구조체 생성 시 **필수 필드**를 강제할 수 있다.

추가로:

- 상태 구조체를 **별도 모듈로 분리**하여,
  - “상태를 순수하게 다루는 함수” (예: `add_item/2`, `expire_old/1`)를 만들면
    GenServer 콜백은 더 얇게 유지할 수 있다.

---

### 타임아웃: 서버 주기 작업 & 클라이언트 응답 제한

타임아웃은 두 방향으로 존재한다.

1. **서버 타임아웃** — 비활성/주기 작업
2. **클라이언트 타임아웃** — 응답 제한

#### 1) 서버 비활성 타임아웃: `{:noreply, state, timeout}`

초안의 패턴:

```elixir
@impl true
def init(:ok) do
  {:ok, %{last: System.monotonic_time()}, 5_000}
end

@impl true
def handle_info(:timeout, st) do
  # 5초간 외부 요청이 없었음 → 주기 작업
  health_check()
  {:noreply, st, 5_000}
end
```

설명:

- `init/1`에서 `{:ok, state, timeout_ms}`를 반환하면,
  - 해당 시간 동안 **아무 메시지도 받지 않을 경우** `handle_info(:timeout, state)`가 호출된다.
- 이를 이용해:
  - 주기적으로 **헬스 체크**,
  - **배치 flush**,
  - idle 상태에서 **자기 종료** 등을 구현할 수 있다.

#### 2) 클라이언트 call 타임아웃: `GenServer.call/3`

```elixir
GenServer.call(pid, :ping, 2_000)
# 2초 안에 응답이 없으면 caller 프로세스에 :timeout exit 발생

```

- 타임아웃은 **호출자 프로세스**의 실패로 나타난다.
- 일반적으로:

```elixir
try do
  GenServer.call(pid, :ping, 2_000)
catch
  :exit, {:timeout, _} ->
    {:error, :timeout}
end
```

또는 상위 Supervisor가 호출자를 관리하도록 설계하기도 한다.

---

### 연속 작업: `:continue`로 초기화 단계를 나누기

초기화가 무거운 경우, `init/1`에서 장시간 블록되면 **애플리케이션 전체 부팅이 뒤로 밀린다.**

이를 피하기 위해 **비동기 초기화**를 사용하는 패턴이 `:continue`다.

```elixir
@impl true
def init(:ok) do
  # 가벼운 초기화만 하고 바로 OK
  {:ok, %{}, {:continue, :warmup}}
end

@impl true
def handle_continue(:warmup, st) do
  :ok = preload_cache()
  {:noreply, st}
end
```

장점:

- Supervisor 입장에서는 `init/1`이 **빠르게 끝나고**,
  나머지 무거운 초기화는 백그라운드에서 처리된다.
- 초기화 중 예외가 발생하면 해당 프로세스만 크래시 → Supervisor가 재시작.

---

### 휴면 모드: `:hibernate`

CPU·메모리를 절약하기 위해, 장시간 일을 하지 않을 서버를 **hibernate 모드**로 넣을 수 있다.

```elixir
@impl true
def handle_cast(:sleep, st) do
  {:noreply, st, :hibernate}
end
```

- `:hibernate`를 반환하면:
  - 프로세스를 디스크립티브하게 **최소 상태**로 축소하고,
  - 다음 메시지가 도착할 때까지 CPU를 거의 쓰지 않는다.
- 단점:
  - 깨어나는 데 추가 오버헤드가 있고,
  - 너무 자주 쓰면 **지연 시간이 들쭉날쭉**해질 수 있다.
- 사용처:
  - 많은 수의 idle 서버를 유지해야 할 때,
  - 메모리 압박 상황에서 일부 서버를 잠재워야 할 때.

---

### 메일박스와 역압: call vs cast, 큐 한도

초안에서 이미 지적한 것처럼:

- `call`은 **요청자 블록**으로 **자연스러운 역압**을 만든다.
- `cast`/`send`는 무한히 쌓일 수 있다.

#### 1) 큐 한도 전략

```elixir
@max_q 2_000

@impl true
def handle_cast(msg, st) do
  {:message_queue_len, q} = Process.info(self(), :message_queue_len)

  if q > @max_q do
    # 정책: 드롭/로그/알람 등
    # 예: 드롭 + 경고 로그
    IO.warn("dropping message due to queue overflow: #{inspect(msg)}")
    {:noreply, st}
  else
    # 정상 처리
    new_state = handle_cast_internal(msg, st)
    {:noreply, new_state}
  end
end

defp handle_cast_internal(msg, st) do
  # 실제 cast 처리 로직
  st
end
```

- 큐 길이가 **임계치**를 넘으면:
  - 메시지를 **무시**, **모니터링 시스템에 경고**,
    **별도 알람 프로세스에 통지** 등 정책을 택할 수 있다.

#### 2) 직관적인 큐 모델

간단한 M/M/1 큐로 직관을 잡으면:

- 평균 도착률: \(\lambda\)
- 평균 처리율: \(\mu\)
- 사용률:
  $$\rho = \frac{\lambda}{\mu}$$

\(\rho\) 가 1에 가까워지면, **평균 대기 시간**이 급증한다.
따라서:

- 서비스 설계에서 기본 규칙은
  $$\lambda \le \mu$$
  즉, **유입률 ≤ 처리율**을 유지하는 것이다.
- 처리율을 높이는 방법:
  - **배치 처리** (여러 메시지를 한번에),
  - **샤딩** (여러 서버로 분산),
  - **워커 풀** (여러 프로세스 병렬 처리).

이 개념은 15장에서 이미 한 번 다뤘고, 여기서는 GenServer 관점에서 다시 강조한 것이다.

---

### 이름 붙이기: 로컬, 글로벌, `:via`(Registry)

##### 1) 로컬 이름

```elixir
{:ok, _} = KV.Server.start_link(name: KV.Server)
GenServer.call(KV.Server, {:put, :a, 1})
```

- 단일 노드 환경 혹은 **“이 서버는 한 노드에 하나”**일 때 편하다.

##### 2) 글로벌 이름

```elixir
{:ok, _} =
  GenServer.start_link(KV.Server, :ok, name: {:global, :kv})

GenServer.call({:global, :kv}, {:put, :a, 1})
```

- 클러스터 전체에서 유일한 이름이 된다.
- 전역 합의/락이 필요하므로, **빈번한 등록/해제**에는 적합하지 않다.

##### 3) Registry + `:via`

```elixir
children = [
  {Registry, keys: :unique, name: MyReg}
]

Supervisor.start_link(children, strategy: :one_for_one)

{:ok, _} =
  GenServer.start_link(
    KV.Server,
    :ok,
    name: {:via, Registry, {MyReg, {:kv, 42}}}
  )

GenServer.call({:via, Registry, {MyReg, {:kv, 42}}}, {:put, :a, 1})
```

- 복합 키, 동적 서버, 샤딩된 서버 등에 적합하다.
- 분산 환경에서는:
  - 각 노드에 자신의 Registry를 두고,
  - **라우터가 대상 노드를 고른 뒤** 그 노드에서 `via` 주소로 호출하는 식으로 쓴다.

---

### 종료와 정리: `terminate/2` + `after` 병행

초안 예제:

```elixir
defmodule FileServer do
  use GenServer

  @impl true
  def init(:ok) do
    {:ok, io} = File.open("write.log", [:append])
    {:ok, %{io: io}}
  end

  @impl true
  def handle_call({:write, line}, _from, st) do
    try do
      IO.write(st.io, line <> "\n")
      {:reply, :ok, st}
    after
      # 필요시 flush 등
      :ok
    end
  end

  @impl true
  def terminate(reason, st) do
    IO.write(st.io, "# terminating: #{inspect(reason)}\n")
    File.close(st.io)
    :ok
  end
end
```

주의사항:

- `terminate/2`는 **모든 종료 케이스에서 보장되는 것은 아니다.**
  - 강제 `:kill` (`Process.exit(pid, :kill)`) 등에서는 호출되지 않을 수 있다.
- 따라서 **리소스 정리**는:
  - 가능한 한 **각 작업 내의 `try … after`** 에서 처리하고,
  - `terminate/2`는 **보너스/최후 방어선** 정도로 생각하는 게 안전하다.

---

### 코드 변경: `code_change/3` (핫 코드 업그레이드 맛보기)

OTP는 **핫 코드 업그레이드** 기능을 갖고 있다.
GenServer는 이를 위해 `code_change/3` 콜백을 제공한다.

```elixir
@impl true
def code_change(_old_vsn, state, _extra) do
  # 예: state 구조 변경
  # 이전 버전에서는 %{data: m} 이었는데, 새 버전에서는 %{data: m, version: 2} 로 변경
  new_state =
    case state do
      %{data: _} = s -> Map.put_new(s, :version, 2)
      s -> s
    end

  {:ok, new_state}
end
```

실전에서는:

- 앱 버전별로 state 스키마를 문서화하고,
- `code_change/3`에서 **마이그레이션 함수**를 호출하여 상태를 변환한다.
- 핫 업그레이드/다운그레이드는 릴리즈 도구와 연동해서 사용한다.

---

### 배치 처리: `handle_info` 타이머 + 버퍼

초안에서 제시한 패턴:

```elixir
defmodule BatchServer do
  use GenServer
  @flush_ms 200

  def start_link(opts \\ []), do: GenServer.start_link(__MODULE__, :ok, opts)
  def enqueue(pid \\ __MODULE__, item), do: GenServer.cast(pid, {:enqueue, item})

  @impl true
  def init(:ok), do: {:ok, %{buf: [], timer?: false}}

  @impl true
  def handle_cast({:enqueue, item}, %{buf: buf}=st) do
    st = if buf == [], do: (Process.send_after(self(), :flush, @flush_ms); st), else: st
    {:noreply, %{st | buf: [item | buf]}}
  end

  @impl true
  def handle_info(:flush, %{buf: buf}=st) do
    if buf != [] do
      write_batch(Enum.reverse(buf))
    end
    {:noreply, %{st | buf: [], timer?: false}}
  end

  defp write_batch(items) do
    # DB/파일 등으로 일괄 쓰기
    :ok = :ok
  end
end
```

설명:

- `enqueue/2`는 `cast`로 item을 보내고, 서버는 버퍼에 쌓는다.
- 버퍼가 비어 있다가 첫 item이 들어오는 순간에만 타이머를 설정하여,
  **일정 시간 후 flush**.
- flush 시에는 버퍼를 뒤집어 정렬된 순서로 일괄 처리.

이 패턴의 장점:

- IO·락·트랜잭션 횟수를 줄인다.
- 배치 크기/주기를 조절하여 **TPS와 지연을 트레이드오프**할 수 있다.

---

### ETS·Agent와의 협업: 읽기-병렬 / 쓰기-직렬

초안에서 보여준 패턴:

```elixir
defmodule KVWithETS do
  use GenServer

  def start_link(opts \\ []), do: GenServer.start_link(__MODULE__, :ok, opts)

  def get(k) do
    case :ets.lookup(:kv, k) do
      [{^k, v}] -> v
      [] -> nil
    end
  end

  def put(k, v), do: GenServer.call(__MODULE__, {:put, k, v})

  @impl true
  def init(:ok) do
    :ets.new(:kv, [:set, :public, :named_table, read_concurrency: true])
    {:ok, %{}}
  end

  @impl true
  def handle_call({:put, k, v}, _from, st) do
    :ets.insert(:kv, {k, v})
    {:reply, :ok, st}
  end
end
```

- 읽기:
  - ETS는 **다중 프로세스가 동시에 읽어도** 락 없이 매우 빠르다.
- 쓰기:
  - GenServer가 직렬화하여 처리 → 데이터 일관성을 보장한다.

Agent를 사용할 때는:

- **작고, 단순한 상태**(카운터, 스냅샷, 메모이제이션)에는 Agent가 충분하다.
- 복합 트랜잭션, 프로토콜, 역압이 필요하면 GenServer가 적합하다.

---

### Telemetry 훅: 관찰 가능성 기본값

초안 예제:

```elixir
:telemetry.span([:kv, :put], %{k: k}, fn ->
  :ets.insert(:kv, {k, v})
  {:ok, %{size: :ets.info(:kv, :size)}}
end)
```

- `span/3`은 “작업의 시작과 끝”을 감싸는 Telemetry 패턴이다.
- 리스너는 다음 정보를 받을 수 있다.
  - 이벤트 이름: `[:kv, :put, :start]`, `[:kv, :put, :stop]`
  - 메타데이터: `%{k: k}`
  - 측정값: 소요 시간, ETS 테이블 크기 등

이를 기반으로:

- Prometheus/Grafana에서 `kv_put_duration_ms`, `kv_table_size` 등의 지표를 만들고,
- LiveDashboard에서 그래프를 그려 **시간에 따른 패턴**을 볼 수 있다.

---

### 테스트: 동기/비동기, assert_receive, Mox

GenServer를 테스트할 때:

1. **동기 API** (`call`)는 일반 함수 테스트와 거의 동일하다.

```elixir
defmodule KV.ServerTest do
  use ExUnit.Case, async: true

  setup do
    {:ok, pid} = KV.Server.start_link()
    %{pid: pid}
  end

  test "put/get", %{pid: pid} do
    assert :ok == KV.Server.put(pid, :a, 1)
    assert 1 == KV.Server.get(pid, :a)
  end
end
```

2. **비동기 효과** (`cast`, `handle_info`)는 `assert_receive`로 검증할 수 있다.

```elixir
test "notify sends message", %{pid: pid} do
  KV.Server.notify(pid, {:echo, self(), "hello"})

  assert_receive {:echo_reply, "hello"}, 1_000
end
```

3. 외부 의존성을 가진 GenServer(HTTP, 메일, 파일 등)는:

- 해당 의존성을 **Behaviour로 추상화**하고,
- 테스트에서는 **Mox** 같은 라이브러리로 목(mock)를 주입해 검증한다.

---

### 성능/복원력 체크리스트

초안의 체크리스트를 다시 정리하면:

- [ ] `call` vs `cast`의 경계가 명확한가?
  - 응답이 꼭 필요한 경로에는 `call`을 써서 **역압**을 만들고 있는가?
- [ ] 메일박스 길이를 주기적으로 측정하고, 알람을 발행하는가?
  - `Process.info(self(), :message_queue_len)` 값을 Telemetry로 내고 있는가?
- [ ] 배치/샤딩/워커 풀 등으로 처리량을 조절하는가?
- [ ] 타임아웃/재시도/백오프 정책이 일관적으로 설계되어 있는가?
- [ ] 모든 외부 리소스(파일/소켓/DB 커넥션)에 대해 `after` 또는 `terminate/2` 에서 정리 보장을 하고 있는가?
- [ ] Telemetry 이벤트를 통해 지연·오류·큐 길이·배치 크기를 관찰할 수 있는가?

이 체크리스트는 **한두 개 서버가 아니라 전체 서비스 아키텍처**에도 그대로 적용된다.

---

## 실전 예제: “주문 집계 서버” (역압 + 배치 + 장애 내성)

초안 마지막에 주어진 “주문 집계 서버”를 좀 더 자세하게 설명한다.

### 요구 사항 복습

- 외부에서 주문 이벤트가 높은 속도로 들어온다.
- 우리는 주문을 **배치로 DB에 적재**하고 싶다.
- 메일박스가 폭주하는 상황을 막아야 한다.
- 성공 응답은 즉시 돌려주되, 내부 적재는 비동기 배치로 처리한다.
- 서버 종료 시 **버퍼에 남은 주문을 가능한 한 플러시**해야 한다.

### 구현 코드

```elixir
defmodule OrderSink do
  use GenServer
  require Logger

  @flush_ms 200
  @max_q 5_000

  # --- Public API ---
  def start_link(opts \\ []) do
    GenServer.start_link(__MODULE__, :ok, opts)
  end

  def submit(pid \\ __MODULE__, order) when is_map(order) do
    GenServer.call(pid, {:submit, order}, 5_000)
  end

  # --- Callbacks ---
  @impl true
  def init(:ok) do
    {:ok, %{buf: [], timer?: false}}
  end

  @impl true
  def handle_call({:submit, order}, _from, st) do
    {:message_queue_len, q} = Process.info(self(), :message_queue_len)

    if q > @max_q do
      {:reply, {:error, :busy}, st}
    else
      st = ensure_timer(st)
      {:reply, :ok, %{st | buf: [order | st.buf]}}
    end
  end

  @impl true
  def handle_info(:flush, st) do
    case Enum.reverse(st.buf) do
      [] ->
        {:noreply, %{st | timer?: false}}

      batch ->
        span_result =
          :telemetry.span([:orders, :flush], %{count: length(batch)}, fn ->
            # 실제로는 Repo.insert_all/2 등으로 DB에 적재
            {:ok, write_batch(batch)}
          end)

        case span_result do
          {:ok, _meta} ->
            {:noreply, %{st | buf: [], timer?: false}}

          _ ->
            Logger.error("flush failed; retrying shortly")
            Process.send_after(self(), :flush, 500)
            {:noreply, %{st | timer?: true}}
        end
    end
  end

  @impl true
  def terminate(_reason, st) do
    if st.buf != [] do
      write_batch(Enum.reverse(st.buf))
    end
    :ok
  end

  # --- Internal helpers ---

  defp ensure_timer(%{timer?: false}=st) do
    Process.send_after(self(), :flush, @flush_ms)
    %{st | timer?: true}
  end

  defp ensure_timer(st), do: st

  defp write_batch(orders) do
    # 실제로는 DB에 insert_all, 외부 API 호출 등
    _ = Enum.count(orders)
    :ok
  end
end
```

### 설계 해설

1. **역압**

   - 외부 API `submit/2`는 `call`로 구현되어 있다.
   - 서버의 메일박스 길이가 `@max_q`를 넘으면 **`{:error, :busy}`** 를 반환.
   - 이 에러를 본 호출자는:
     - 재시도(backoff),
     - 큐에 다시 넣기,
     - downstream에 오류 전달 등 정책을 택할 수 있다.

2. **배치**

   - 첫 주문이 들어와 버퍼가 비어 있다면 `ensure_timer/1`이 `Process.send_after/3`로 `:flush` 메시지를 예약.
   - `@flush_ms` 동안 모인 주문을 한 번에 기술(파일/DB)로 내리면서 **시스템 호출/트랜잭션 수를 줄인다.**

3. **장애 내성**

   - `write_batch/1`에서 예외가 나면 `span_result`가 실패하고,
     - 로그를 남기고,
     - 500ms 뒤 재시도.
   - 이 재시도 전략은 상황에 따라:
     - 최대 시도 횟수,
     - 지수 백오프,
     - dead-letter 큐 등으로 확장할 수 있다.

4. **정리 보장**

   - `terminate/2`에서 버퍼가 비어 있지 않다면 즉시 한 번 더 `write_batch/1`을 호출하여 **최대한 데이터 유실을 줄인다.**
   - 완전한 0% 유실을 보장하려면:
     - 트랜잭션 로그,
     - 이벤트 소싱,
     - 외부 큐(Kafka 등)와의 조합이 필요하다.

5. **관찰**

   - `:telemetry.span/3` 으로 flush 이벤트를 둘러싸,
     - 배치 크기,
     - 소요 시간,
     - 실패율 등을 관찰할 수 있게 만든다.

---

## 마무리

이 장에서 다룬 요점들을 정리하면:

1. **OTP의 다섯 축**
   - 행위(behaviours), Application, Supervision, Observability, Distribution/Releases
   가 합쳐져 **“BEAM 스타일 서버/서비스 설계”**의 규범을 구성한다.

2. **GenServer는 “상태를 가진 서버”의 기본형**
   - 콜백(`init/1`, `handle_call/3`, `handle_cast/2`, `handle_info/2`)만 작성하면,
     메시징, 링크, 모니터, 타임아웃, 정리, 코드 변경 등은 OTP 프레임워크가 담당한다.

3. **역압, 배치, 타임아웃, 정리 보장, 관찰**
   - `call` / `cast` 구분,
   - `message_queue_len` 기반 큐 한도,
   - `:noreply, state, timeout` + `:timeout` 메시지,
   - `terminate/2` + `try … after` 조합,
   - Telemetry 이벤트/Logger
   를 통해 **성능과 복원력**을 동시에 가져간다.

4. **상태는 구조체, ETS, Agent, GenServer를 조합해서 설계**
   - 읽기-집중에는 ETS,
   - 작은 스냅샷/메모이제이션에는 Agent,
   - 정책/프로토콜/역압에는 GenServer를 선택한다.

5. **슈퍼비전 트리와 Application**
   - 모든 서버를 Supervisor 트리 아래에 두고,
   - Application의 `start/2`, `stop/1`를 통해 전체 시스템의 부트/정지를 통제한다.

다음 단계에서는 여기서 구축한 **GenServer/슈퍼비전/노드/Telemtery 기초** 위에
실제 웹 서비스, 백그라운드 워커, 스트리밍 파이프라인, 분산 캐시 등을 쌓아 올릴 수 있다.
핵심은 “**프로세스의 대화**와 **정책(역압/배치/복구)**을 먼저 설계하고, 코드로 구현하는 것”이다.
