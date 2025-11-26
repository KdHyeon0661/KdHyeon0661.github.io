---
layout: post
title: Elixir - Task와 Agent
date: 2025-11-23 14:25:23 +0900
category: Elixir
---
# 동시성 도구 : Task와 Agent

## 태스크(Task)

### Task가 해결하는 문제 (초안 포함)

- **한 번성(oneshot) 작업**을 **별도 프로세스**에서 실행하고, 결과를 **join**(수거)하는 패턴을 단순화.
- 실패는 **연결(link)** 를 통해 상위에 전파되어 **빠른 실패(fail fast)** 를 보장.
- **시간 제한/취소**/동시 실행/결과 수거를 표준 API로 제공.

OTP 관점에서 Task는 “**작업 단위 프로세스**”를 위한 표준 패키지다.
GenServer가 “계속 살아 있는 상태ful 서비스”라면, Task는 “**일을 하고 사라지는 프로세스**”다.

### Task의 내부 모델: link, monitor, exit reason

Task는 크게 두 계열로 나뉜다.

1) **링크된 태스크(Linked Task)**
   - `Task.async/1`, `Task.async/3`, `Task.Supervisor.async/2` 등
   - 부모 프로세스와 **link**가 연결된다.
   - 태스크가 비정상 종료하면 부모도 **EXIT 신호를 받아 함께 실패**한다.
   - “실패를 숨기지 말고 빨리 드러내라”는 OTP 철학을 그대로 따른다.

2) **비링크 태스크(Nolink/Fire-and-forget Task)**
   - `Task.start/1`, `Task.Supervisor.start_child/2`, `Task.Supervisor.async_nolink/2` 등
   - 부모와 link가 없거나, monitor로만 관찰한다.
   - 태스크 실패가 부모의 생존을 위협하지 않는다.
   - 대신 **실패를 직접 수거/로깅/재시도**해야 한다.

#### link vs monitor 한 줄 직관

- **link**: “자식이 죽으면 부모도 같이 위험해져야 한다.”
- **monitor**: “자식이 죽어도 부모는 살아야 하지만, 사망 사실은 알고 싶다.”

Task는 상황에 맞춘 API를 제공할 뿐이다.
**어떤 의미론을 선택할지는 설계자가 결정**해야 한다.

---

### 기본 사용 — `Task.async/await` (초안 포함)

```elixir
task = Task.async(fn -> Enum.sum(1..10_000_000) end)
# 다른 일…

result = Task.await(task, 5_000)  # ms 타임아웃
IO.puts("합계: #{result}")
```

- `Task.async/1`는 **링크된(linked)** 자식 프로세스를 만든다. 작업 실패 시 링크를 통해 상위로 **EXIT**가 전파된다.
- `Task.await/2`는 결과를 기다리며, 타임아웃이면 **exit(:timeout)**.

#### 실전에서 중요한 디테일

1) `await/2`의 타임아웃은 **부모를 죽인다**
   - `exit(:timeout)`이므로 상위로까지 전파될 수 있다.
   - “타임아웃은 실패다”라는 강한 규약을 갖는 API다.

2) 태스크 실패는 부모를 죽인다
   - 아래를 실행하면, `boom`에서 예외가 터지면서 **부모도 함께 크래시**한다.

```elixir
t = Task.async(fn -> raise "boom" end)
Task.await(t)
# => (부모 프로세스도 EXIT)

```

이게 부담스럽다면 `async_nolink` 계열을 쓰거나, `try/catch`로 EXIT를 받아 변환하라.

{% raw %}
```elixir
t = Task.async(fn -> raise "boom" end)

result =
  try do
    {:ok, Task.await(t, 1000)}
  catch
    :exit, reason -> {:error, reason}
  end

result
# => {:error, {%RuntimeError{message: "boom"}, _stack}}
```
{% endraw %}

> **규율**: `Task.async`를 썼으면 **반드시 await/yield/shutdown으로 수거**해야 한다.
> 수거하지 않으면 태스크 메시지가 메일박스에 남고, GC/메모리를 오염시킨다.

---

### 비연결 방식 — `Task.start/1` (초안 포함)

```elixir
{:ok, pid} = Task.start(fn -> do_fire_and_forget() end)
```

- 실패가 상위로 전파되지 않는 **비-링크된** 태스크. 로깅 또는 모니터링을 직접 붙여야 한다.

#### Fire-and-forget의 실전 함정

- “부모와 운명을 같이하지 않는다”는 건
  **실패를 조용히 버릴 수도 있다**는 뜻이다.
- 특히 외부 I/O(HTTP, DB) 작업에서 `Task.start`를 남발하면:
  - 실패가 로그에도 안 남고,
  - 결과가 어디에도 합류되지 않고,
  - 시스템은 “**조용히 틀린 상태**”가 된다.

따라서 `Task.start`는 다음 중 하나일 때만 추천된다.

- **정말 결과가 필요 없는 부수효과 작업**
  예: 비동기 로그 flush, 임시 캐시 warm-up, best-effort 알림
- **외부에서 monitor/재시도/큐를 붙이는 구조가 이미 있음**
  예: GenStage/Broadway에서 작업 단위를 태스크로 분리

---

### 결과를 나중에 받을 때 — `Task.async/1` + `Task.yield/2` + `Task.shutdown/2` (초안 포함)

```elixir
t = Task.async(fn -> long_calc() end)
case Task.yield(t, 1000) || Task.shutdown(t, :brutal_kill) do
  {:ok, value} -> {:ok, value}
  nil -> {:error, :timeout}
end
```

- `yield/2`로 **부분 대기** → 이후에도 안 오면 `shutdown/2`로 **정리**.

#### `yield/2`의 중요한 성질

- `yield/2`는 “**지금까지 결과가 왔는지**”만 본다.
- 결과가 안 오면 `nil`을 리턴하고,
  태스크는 **계속 실행 중**이다.

즉, `yield` 후 반드시

- 기다릴 건지 (`await`)
- 죽일 건지 (`shutdown`)
- 그냥 놔둘 건지(대부분 안 됨)

를 **명시적으로 선택**해야 한다.

#### `shutdown/2` 옵션

- `shutdown(t, timeout_ms)`
  - 먼저 **정상 종료 요청**(`:shutdown`)을 보내고
  - timeout 안에 안 죽으면 **강제 kill**한다.

- `shutdown(t, :brutal_kill)`
  - 곧바로 강제 종료

실무에선 대개:

1) 온화하게 종료 요청
2) 안 되면 brutal kill

순서를 따른다.

---

### 다수의 태스크를 동시에 — `Task.async_stream/3,4` (초안 포함)

```elixir
urls = ["https://a", "https://b", "https://c"]
results =
  urls
  |> Task.async_stream(&fetch/1, max_concurrency: System.schedulers_online(), timeout: 4_000, ordered: false)
  |> Enum.map(fn
    {:ok, body} -> {:ok, body}
    {:exit, _} -> {:error, :crash}
    {:error, :timeout} -> {:error, :timeout}
  end)
```

- **역압**과 **동시성 제어**가 내장. `ordered: false`면 **완료된 순서대로** 빠르게 소비.
- 실패는 `{:exit, reason}`에서 수거 가능.

#### async_stream이 “역압을 내장했다”는 뜻

- 입력 enumerable을 **한꺼번에 태스크로 펼치지 않는다.**
- `max_concurrency` 만큼만 벌리고,
- 하나가 끝나면 그 자리를 **다음 작업으로 채운다.**

즉, **“작업 생산 → 소비” 파이프라인이 자동으로 균형**을 맞춘다.

#### ordered 옵션의 의미

- `ordered: true`(기본)
  - 입력 순서대로 결과를 내보낸다.
  - 느린 작업 하나가 앞에 있으면 **뒤 결과도 막힌다.**

- `ordered: false`
  - **끝나는 순서대로** 결과를 내보낸다.
  - “빠른 결과를 먼저 소비”해야 할 때 유리.

#### timeout 옵션의 의미

- 개별 태스크마다 적용되는 타임아웃.
- 타임아웃이면 결과가 `{:exit, :timeout}` 또는 `{:error, :timeout}` 형태로 온다.
- 타임아웃 정책이 강하면 작업을 **작게 쪼개거나**,
  타임아웃을 충분히 주고 상위에서 다시 제한하는 게 안전하다.

#### 실전형 패턴: 실패를 그냥 흘려보내지 않기

```elixir
def fetch_many(urls) do
  urls
  |> Task.async_stream(&fetch/1,
       max_concurrency: 16,
       timeout: 5000,
       ordered: false)
  |> Enum.reduce({[], []}, fn
       {:ok, body}, {oks, errs} -> {[body | oks], errs}
       {:exit, reason}, {oks, errs} -> {oks, [reason | errs]}
     end)
end
```

- 성공/실패를 **분리 수거**해 관찰/리트라이/알람으로 연결할 수 있다.

---

### 태스크 감독 — `Task.Supervisor` (초안 포함)

- 다수의 태스크를 **동적으로** 띄우고 **감독**하려면 필요.

```elixir
defmodule MyApp.TaskSup do
  use Task.Supervisor
end

# 애플리케이션 트리에 붙이기

children = [
  {Task.Supervisor, name: MyApp.TaskSup, restart: :permanent}
]

# 사용: 링크 여부 선택 가능

{:ok, t} = Task.Supervisor.start_child(MyApp.TaskSup, fn -> work() end)
# 또는 링크된 async

task = Task.Supervisor.async(MyApp.TaskSup, fn -> work() end)
Task.await(task)
```

- **트랜잭션/셋업 비용이 큰** 작업을 **필요할 때만** 띄우는 구조에 적합.

#### Task.Supervisor가 필요한 상황

1) “**태스크가 많고, 수명이 들쭉날쭉**”할 때
   - 루트 트리에 태스크를 직접 넣으면 관리가 어렵다.
2) “**태스크의 실패 정책을 감독 트리로 통일**”하고 싶을 때
3) “**특정 도메인(예: 이미지 처리, 크롤링) 전용 작업 풀**”을 만들고 싶을 때

#### Task.Supervisor의 restart 의미론

- Task.Supervisor 자체는 long-lived supervisor다 → `restart: :permanent`가 보통.
- 그러나 `start_child/2`로 띄운 태스크들은 대개 **transient/temporary 성격**이다.
  (성공하면 사라지고, 실패하면 재시작할 필요가 없는 경우가 많다.)

즉, “감독자는 계속 살되, 작업은 일회성”이 기본 모델이다.

---

### 수학적 직관: 병렬화 이득 (초안 포함)

동일 작업 \(N\)개, 각 작업 시간 \(t\), 동시성 \(k\)라면 이상적 시간은

$$
T \approx \left\lceil \frac{N}{k} \right\rceil t
$$

단, 과도한 \(k\)는 **컨텍스트 스위칭/IO 병목**으로 \(T\)가 악화될 수 있다.
따라서 `max_concurrency`는 **코어 수** 또는 **IO 대기 비율**을 고려해 실측 튜닝한다.

#### 실전 튜닝 팁

- **CPU-bound 작업**:
  `k ≈ System.schedulers_online()` 근처가 보통 최적.
- **IO-bound 작업**:
  대기 시간이 길면 `k`를 더 키워도 이득이 있다.
  단, 외부(서버/DB) 한도를 넘기지 않는 범위에서.

---

## 에이전트(Agent)

### Agent가 해결하는 문제 (초안 포함)

- **작은 공유 상태**(캐시/카운터/맵 등)를 **함수로 변화**시키는 경량 서버.
- `get/update/cast`로 설계가 단순. **강한 동기화 정책**(락, 순서 보장)이 필요하면 GenServer를 고려.

Agent는 내부적으로 **GenServer의 얇은 래퍼**다.

- 상태를 소유하는 프로세스가 1개 있고
- 그 프로세스가 상태 변경 함수를 **직렬로 적용**한다.
- API가 단순화된 대신 **복잡한 프로토콜엔 부적합**하다.

### 기본 사용법 (초안 포함)

```elixir
{:ok, pid} = Agent.start_link(fn -> %{} end, name: :kv)
Agent.update(:kv, &Map.put(&1, :x, 42))
v = Agent.get(:kv, &Map.get(&1, :x))    # 42
```

- 상태는 **함수로만** 변경된다(불변 데이터의 새 사본을 반환).

### 원자적 업데이트와 읽기 (초안 포함)

```elixir
Agent.get_and_update(:kv, fn m ->
  old = Map.get(m, :ctr, 0)
  {old, Map.put(m, :ctr, old + 1)}
end)
```

- **읽기-변경**을 **원자적**으로 수행(중간 경쟁 없음).

Agent로 카운터/캐시를 만들 때
`get_and_update`가 핵심이다.

### 타임아웃/오류 처리 (초안 포함)

```elixir
Agent.update(:kv, fn _ -> heavy_compute() end, 10_000)
```

- 기본 타임아웃 5,000ms. 오래 걸릴 수 있으면 인자 조정.
- 내부에서 예외가 나면 Agent 프로세스가 **크래시**하고 슈퍼바이저가 재시작한다(상태 유실 주의).

#### Agent에서 heavy_compute를 돌리면 안 되는 이유

Agent의 update 함수는 **Agent 프로세스 안에서 실행**된다.
즉,

- 계산이 길면
- Agent가 그동안 **모든 요청을 못 받는다.**

“작은 상태를 빠르게 다루는 서버”라는 Agent의 성격에 어긋난다.

heavy 작업은 Task/별도 워커에서 하고,
Agent에는 **결과만 반영**하는 게 정석이다.

### 초기값/종료 훅 (초안 포함)

```elixir
{:ok, _} = Agent.start_link(fn -> load_snapshot!() end, name: :cache)
# 종료 시 상태 저장

defmodule SnapAgent do
  use Agent
  def start_link(_), do: Agent.start_link(fn -> %{} end, name: __MODULE__)
  def child_spec(_), do: Supervisor.child_spec(%{id: __MODULE__, start: {__MODULE__, :start_link, [[]]}, shutdown: 10_000}, [])
end
```

- 종료猶予 내에 **스냅샷 저장**을 고려(단, 강제 kill 가능성 있으므로 요청 경로에서 `try...after`가 더 중요).

#### 실전 스냅샷 보강 패턴

```elixir
defmodule SnapAgent do
  def start_link(_) do
    Agent.start_link(fn -> load_snapshot() end, name: __MODULE__)
  end

  def put(k, v), do: Agent.update(__MODULE__, &Map.put(&1, k, v))
  def get(k), do: Agent.get(__MODULE__, &Map.get(&1, k))

  defp load_snapshot() do
    case File.read("snap.bin") do
      {:ok, bin} -> :erlang.binary_to_term(bin)
      _ -> %{}
    end
  end

  def save_snapshot!() do
    Agent.get(__MODULE__, fn st ->
      File.write!("snap.bin", :erlang.term_to_binary(st))
    end)
  end

  def child_spec(_) do
    %{
      id: __MODULE__,
      start: {__MODULE__, :start_link, [[]]},
      shutdown: 10_000,
      restart: :permanent
    }
  end
end
```

- “종료 시 저장”만 믿지 말고,
  중요한 상태라면 **주기적 스냅샷/로그 기반 저장**이 필요하다.

### Agent vs ETS (초안 포함)

- **Agent**: 상태를 **함수**로 안전하게 변경, 직관적 API. **쓰기 직렬화**.
- **ETS**: 대량 읽기/병렬 액세스에 유리, **프로세스 외부 표**. 그러나 API가 더 저수준, TTL/정책 직접 구현 필요.
- 작은 캐시/카운터 → Agent, 대량 조회/인덱스 → ETS.

#### 더 깊은 선택 기준

- 읽기가 압도적으로 많고,
  여러 프로세스가 **동시에 접근해야 한다**
  → ETS가 유리

- 쓰기 규칙이 단순하고
  일관성은 “최신이 아니어도 됨” 수준
  → Agent가 유리

- 쓰기 규칙이 복잡하고
  “상태 전이 규약”이 핵심 도메인이라면
  → GenServer(또는 GenStateMachine)

---

## 더 큰 예제 — “URL 인덱서”: Task + Agent + Task.Supervisor (초안 포함)

**요구**:
1) 다수 URL을 병렬로 받아와 본문을 파싱
2) 토큰 빈도수를 **공유 카운터**에 누적 (Agent)
3) 실패/시간초과는 건너뛰고 로그
4) 안전한 종료/재시작 구조

### 슈퍼비전 트리 (초안 포함)

```
App.Supervisor
 ├─ {Task.Supervisor, name: App.FetchSup}
 └─ {Agent, name: App.Index, init: fn -> %{} end}
```

### 구현 (초안 포함)

#### 인덱스(Agent)

```elixir
defmodule App.Index do
  def start_link(_) do
    Agent.start_link(fn -> %{} end, name: __MODULE__)
  end

  def inc_tokens(tokens) do
    Agent.update(__MODULE__, fn m ->
      Enum.reduce(tokens, m, fn t, acc -> Map.update(acc, t, 1, &(&1 + 1)) end)
    end)
  end

  def top(n) do
    Agent.get(__MODULE__, fn m ->
      m |> Enum.sort_by(fn {_t, c} -> -c end) |> Enum.take(n)
    end)
  end
end
```

#### 페치 + 파싱 태스크

{% raw %}
```elixir
defmodule App.Fetcher do
  @timeout 4_000

  def fetch_and_tokenize(url) do
    with {:ok, body} <- http_get(url),
         tokens <- tokenize(body) do
      App.Index.inc_tokens(tokens)
      :ok
    else
      {:error, reason} ->
        require Logger
        Logger.warning("fetch failed #{url}: #{inspect(reason)}")
        {:error, reason}
    end
  end

  defp http_get(url) do
    # 데모를 위해 File.read/1로 대체 가능
    # 실제로는 Finch/Req 등 사용
    case :httpc.request(String.to_charlist(url)) do
      {:ok, {{_, 200, _}, _hdrs, body}} -> {:ok, to_string(body)}
      {:ok, {{_, code, _}, _, _}} -> {:error, {:http, code}}
      {:error, r} -> {:error, r}
    end
  catch
    :exit, _ -> {:error, :timeout}
  end

  defp tokenize(body) do
    body
    |> String.downcase()
    |> String.replace(~r/[^a-z0-9가-힣\s]/u, " ")
    |> String.split(~r/\s+/, trim: true)
    |> Enum.reject(&(&1 == ""))
  end
end
```
{% endraw %}

#### 동시 실행 — Task.Supervisor + async_stream

```elixir
defmodule App.Runner do
  def run(urls, max \\ System.schedulers_online()) do
    urls
    |> Task.async_stream(&App.Fetcher.fetch_and_tokenize/1,
         max_concurrency: max, timeout: 5_000, ordered: false)
    |> Stream.run()

    App.Index.top(20)
  end
end
```

#### 애플리케이션/슈퍼바이저

```elixir
defmodule App.Supervisor do
  use Supervisor
  def start_link(_), do: Supervisor.start_link(__MODULE__, :ok, name: __MODULE__)
  def init(:ok) do
    children = [
      {Task.Supervisor, name: App.FetchSup},
      {App.Index, []}
    ]
    Supervisor.init(children, strategy: :one_for_one)
  end
end

defmodule App.Application do
  use Application
  def start(_t, _a) do
    Supervisor.start_link([{App.Supervisor, []}], strategy: :one_for_one, name: App.Root)
  end
end
```

#### IEx에서 실행

```elixir
# iex -S mix

urls = ~w(https://elixir-lang.org https://hexdocs.pm https://news.ycombinator.com)
App.Runner.run(urls, 8)
# ⇒ 상위 토큰 20개 리스트

```

### 확장 포인트를 실제로 코드로 보강하기

#### 일시적 실패 재시도 + 지수 백오프

```elixir
defmodule App.Retry do
  @max 3
  def with_retry(fun, attempt \\ 0) do
    case fun.() do
      {:ok, v} -> {:ok, v}
      {:error, _} = e when attempt < @max ->
        backoff = trunc(:math.pow(2, attempt)) * 200
        Process.sleep(backoff)
        with_retry(fun, attempt + 1)
      other -> other
    end
  end
end

def fetch_and_tokenize(url) do
  App.Retry.with_retry(fn -> http_get(url) end)
  |> case do
    {:ok, body} ->
      tokens = tokenize(body)
      App.Index.inc_tokens(tokens)
      :ok
    {:error, reason} ->
      require Logger
      Logger.warning("fetch failed #{url}: #{inspect(reason)}")
      {:error, reason}
  end
end
```

#### Telemetry로 지연/성공률 내보내기

```elixir
def fetch_and_tokenize(url) do
  :telemetry.span([:app, :fetch], %{url: url}, fn ->
    res =
      with {:ok, body} <- http_get(url) do
        tokens = tokenize(body)
        App.Index.inc_tokens(tokens)
        :ok
      end

    {res, %{result: res}}
  end)
end
```

핸들러 예:

```elixir
:telemetry.attach(
  "fetch-logger",
  [:app, :fetch, :stop],
  fn _event, meas, meta, _ ->
    require Logger
    Logger.info("fetch #{meta.url} took #{meas.duration}ns result=#{inspect(meta.result)}")
  end,
  nil
)
```

---

## 에이전트와 태스크냐, 젠서버냐 — 선택 기준 (초안 포함 + 보강)

정답은 “**요구사항의 복잡도와 일관성 수준**”이다. 아래 체크리스트로 결정하자.

### Agent를 선택할 때 (초안 포함)

- 상태가 **작고 단순**하다(맵/카운터/간단 캐시).
- 상태 변경 규칙이 **충돌이 적다**. 트랜잭션/순서 보장은 덜 중요.
- **빠른 구현**이 필요하고, 실패 시 **상태 유실**을 감당 가능(또는 스냅샷으로 보완).

추가 경고:

- update 함수 안에서 **I/O나 heavy compute 금지**
- 상태가 커지면 (수만~수십만 항목)
  → GC/복사 비용 때문에 ETS로 넘어가야 한다.

### Task를 선택할 때 (초안 포함)

- **일회성 작업**을 **병렬**로 처리하고 결과를 수거해야 할 때.
- **시간 제한**/취소/감시가 필요할 때(`yield/shutdown`, `Task.Supervisor`).
- 처리 순서가 중요하지 않거나, **완료된 것부터** 빠르게 소비하고 싶다(`async_stream/4`).

추가 경고:

- Task는 **장기 생존 서비스에 쓰면 안 된다.**
  GenServer/Worker로 바꿔라.
- `async`를 쓰면 **반드시 수거 규율**을 지켜라.

### GenServer를 선택할 때 (초안 포함)

- 상태가 **비즈니스 규칙**으로 촘촘히 보호되어야 한다(일관성/순서/검증/승인).
- **동시 접근**을 **직렬화**해야 한다(예: 계좌 이체, 재고 관리, 예약).
- **복잡한 프로토콜**(콜백/타이머/백오프/외부 리소스 수명주기)을 수반한다.
- 장애 시 **정교한 복구**와 **재시작 전략**이 필요.

### 비교 요약 표 (초안 포함)

| 항목 | Agent | Task | GenServer |
|---|---|---|---|
| 주 용도 | 간단 공유 상태 | 일회성 병렬 작업 | 규칙적 상태/프로토콜 서버 |
| 일관성 | 낮음(함수 기반) | N/A | 높음(직렬화) |
| 실패 전파 | 슈퍼바이저 | 링크/감독 | 슈퍼바이저 |
| API 복잡도 | 낮음 | 낮음 | 중간~높음 |
| 확장성 | 상태가 커지면 부적합 | 작업 단위 수평 확장 | 로직/상태 확장에 적합 |
| 언제 X? | 복합 트랜잭션/순서 요구 | 결과 미수거/리소스 누수 | 과도한 보일러플레이트 필요 시 |

---

## 실전 안티패턴 모음

### Task를 “서버처럼” 쓰는 실수

```elixir
# 나쁜 예: 장기 루프를 Task로 돌림

Task.start(fn ->
  loop_forever()
end)
```

- Task는 “작업하고 사라지는 프로세스”라는 전제에서 설계됐다.
- 장기 루프는 **Worker/GenServer**로 올려 슈퍼비전에 붙여야 한다.

### await를 빼먹는 실수

```elixir
Task.async(fn -> heavy() end)
# 끝… (수거 안 함)

```

- 결과 메시지가 호출자 메일박스에 남는다.
- 반복되면 메모리/GC/큐 길이가 터진다.

### Agent에 복잡한 규칙을 넣는 실수

Agent는 **API가 단순한 게 장점**이지
복잡한 도메인 규칙을 담기 위한 도구가 아니다.

- 규칙이 커지면:
  - 코드가 흩어지고
  - 원자적 보장이 흐려지고
  - 감사/테스트가 어려워진다.

그 순간 **GenServer로 승격**하라.

---

## 결론

- **Task**는 OTP가 제공하는 표준 “작업 프로세스” 도구다.
  링크/수거/타임아웃/역압이 내장된 **구조적 병렬화**를 가능하게 한다.
- **Agent**는 작은 공유 상태를 가장 단순한 방식으로 안전하게 다룬다.
  그러나 크기/규칙이 커지는 순간 ETS/GenServer로 넘어가야 한다.
- 무엇을 쓰든 핵심은 같다.

> “**의도를 의미론과 구조로 표현하고,
> 실패는 숨기지 말고 수거하며,
> 부담은 max_concurrency/백오프로 제어하라.**”
