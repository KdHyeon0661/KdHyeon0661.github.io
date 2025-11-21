---
layout: post
title: Elixir - 분산 노드, 이름짓기, PID, IO
date: 2025-11-20 17:25:23 +0900
category: Elixir
---
# 분산 노드, 이름짓기, PID, IO

## 노드에 이름 짓기

### 노드란?

- **노드(node)**란?
  - 하나의 **BEAM VM 인스턴스**를 가리키는 논리적인 이름.
  - 엘릭서/얼랭 런타임이 실행되는 단위.
- 이름이 없는 IEx(`iex -S mix`)는:
  - 단순히 **로컬 싱글 노드**로만 동작.
  - 같은 머신에 다른 VM이 떠 있어도, **분산 통신 기능이 비활성화**되어 있다.
- 이름을 부여하면:
  - 노드 간 **메시지 송수신**,
  - **링크/모니터**,
  - **원격 프로세스 스폰**,
  - **RPC** (`:rpc.call/5`) 등이 가능해진다.

정리하면:

- “노드 이름이 없다” → 그냥 **로컬 VM**.
- “노드 이름이 있다” → 다른 이름 있는 VM과 **클러스터**를 구성할 수 있다.

간단 비교 표:

| 상태 | 예 | 분산 기능 |
|------|----|-----------|
| 이름 없음 | `iex -S mix` | X (로컬만) |
| 단축 이름 | `iex --sname a -S mix` | O (로컬 네트워크) |
| 정규 이름 | `iex --name a@host.example.com -S mix` | O (멀티 호스트, FQDN 의존) |

---

### 단축/정규 이름: `--sname` vs `--name`

로컬·실습·운영 환경별로 보통 다음 패턴을 쓴다.

```bash
# 같은 호스트 혹은 단일 LAN 실습에 편한 단축 이름

iex --sname a -S mix
iex --sname b -S mix

# 완전한 FQDN을 사용하는 정규 이름 (멀티 호스트용)

iex --name a@host1.example.com -S mix
iex --name b@host2.example.com -S mix
```

- `--sname a`
  - 실제 노드 이름은 `:"a@<local-hostname>"`.
  - 예: 호스트가 `devbox` 라면 `:"a@devbox"`.
  - 로컬 개발/테스트에 많이 사용.
- `--name a@host1.example.com`
  - 실제 노드 이름은 `:"a@host1.example.com"`.
  - DNS가 해결되어야 하고, 방화벽/포트(기본 4369, 분산 포트 범위)가 열려 있어야 다른 호스트와 통신 가능.
  - 여러 서버를 묶는 **운영/스테이징 클러스터**에서 일반적.

주의할 점:

- 노드 이름은 **원자(atom)** 형태로 쓰인다: `:"a@devbox"`.
- 호스트 이름/도메인을 잘못 넣으면, 연결이 안 되거나 이상한 이름으로 뜰 수 있다.
- OS/네트워크 레벨에서 **hostname/FQDN 설정이 깔끔**해야 분산 이름도 안정적이다.

---

### 쿠키(cookie) — 인증 키

분산 노드끼리 마음대로 연결되면 위험하므로,
얼랭/엘릭서는 **쿠키(cookie)** 라는 공유 비밀 키를 사용한다.

- 규칙: **같은 쿠키를 가진 노드끼리만 연결 가능**.

```bash
iex --sname a --cookie SECRET -S mix
iex --sname b --cookie SECRET -S mix
```

- 위 두 노드는 `SECRET` 이라는 쿠키를 공유하므로 서로 연결 가능.

런타임에서 확인/설정:

```elixir
Node.get_cookie()
Node.set_cookie(node(), :SECRET)  # 현재 노드의 쿠키 변경
```

실수 예:

```bash
iex --sname a --cookie APP1 -S mix
iex --sname b --cookie APP2 -S mix
```

이때 `Node.connect(:"b@...")` 는 항상 `false` 가 된다(쿠키 불일치).

운영에서의 패턴:

- 쿠키를 **환경 변수**나 **시크릿 매니저**에 넣고, 릴리즈에서 거기서 읽어온다.
- 개발/테스트 환경에서는 간단히 `--cookie dev` 정도로 써도 괜찮지만,
- 프로덕션에서는 쿠키가 **노출되면 임의 코드 실행** 위험이 생길 수 있으므로, 관리에 주의해야 한다.

---

### 노드 연결과 RPC

이제 실제로 노드를 연결해보고, **RPC(Remote Procedure Call)** 을 해보자.

#### 1) 노드 연결

```elixir
# a 노드 셸에서

Node.connect(:"b@<host>")   # true/false
Node.list()                 # [:"b@<host>"] 혹은 []
```

- 성공 시 `true`, 실패 시 `false`.
- `Node.list/0` 는 현재 노드가 **연결된 노드 목록**을 반환한다.

#### 2) 단일 RPC 호출

```elixir
# a 노드에서 b 노드의 Enum.sum 실행

:rpc.call(:"b@<host>", Enum, :sum, [[1, 2, 3]])
# => 6

```

- 인자:
  - 첫 번째: 대상 노드
  - 두 번째: 모듈
  - 세 번째: 함수 이름(atom)
  - 네 번째: 인자 리스트(리스트의 각 원소가 인자가 된다)

실패 시:

```elixir
:rpc.call(:"b@unknown", Enum, :sum, [[1, 2, 3]])
# => {:badrpc, :nodedown} 등

```

#### 3) 멀티 노드 브로드캐스트

```elixir
:rpc.multi_call(Node.list(), Enum, :count, [[1, 2, 3]])
# => { [3, 3, ...], [] }  형태 (성공 리스트, 실패 노드 리스트)

```

- 여러 노드에 **동시에 같은 함수를 호출**하고, 결과를 모아서 돌려준다.
- 예: **캐시 warm-up**, **노드 상태 점검**, **통계 수집** 등에 사용 가능.

---

### 원격 셸 붙기 (`--remsh`)

운영 중인 노드를 디버깅할 때 매우 자주 쓰는 기능:

```bash
# A 노드가 이미 실행 중일 때, B에서 원격 셸로 붙기

iex --sname b --remsh a@<host> --cookie SECRET
```

- `--remsh` 옵션은 “이 IEx 세션을 **기존 노드의 셸에 붙인다**”는 의미.
- 이 명령을 실행한 터미널에서는,
  - 로컬 VM이 아니라
  - **원격 노드의 프로세스/상태**를 보는 셸이 뜬다.

운영에서의 패턴:

- 장애가 발생했을 때:
  - 특정 노드에 `--remsh` 로 붙어서,
  - `Process.info/2`, `:sys.get_state/1`, `:observer.start/0` 등을 사용해 상태를 조사한다.
- 보안 측면:
  - `--remsh` 는 **굉장히 강력한 권한**이므로,
  - 노드 쿠키 노출, 운영 서버 접근 권한을 심각하게 관리해야 한다.

---

### 실습: 두 노드 간 간단 채팅

로컬에서 두 노드를 만들고, 서로 메시지를 주고받는 간단 예제.

1) 터미널 1:

```bash
iex --sname a --cookie CHOCO
```

2) 터미널 2:

```bash
iex --sname b --cookie CHOCO
```

3) 각 노드에서 연결 확인:

```elixir
# a 노드에서

Node.connect(:"b@#{:net_adm.localhost()}")
Node.list()

# b 노드에서

Node.list()
```

(실제 이름은 `:"b@<hostname>"` 형태이므로 `Node.list/0` 결과를 보고 그대로 가져오면 좋다.)

4) 간단한 “ping 서버”를 b에서 띄우기:

```elixir
defmodule PingServer do
  def loop do
    receive do
      {:ping, from} ->
        send(from, {:pong, self()})
        loop()
    end
  end
end

pid = spawn(fn -> PingServer.loop() end)
Process.register(pid, :ping_server)
```

5) a 노드에서 b 노드의 ping 서버 호출:

```elixir
send({:ping_server, :"b@<host>"}, {:ping, self()})

receive do
  {:pong, pid} ->
    IO.puts("pong from #{inspect(pid)}")
after
  1_000 ->
    IO.puts("timeout")
end
```

이 간단한 예제를 통해:

- 노드 이름 + 프로세스 이름으로 **원격 주소 지정**이 가능함을 보고,
- 메시지가 **노드 경계를 넘어서도 같은 추상화**로 동작함을 확인할 수 있다.

---

## 프로세스에 이름 짓기

> 노드를 넘어서는 **식별·발견**이 있어야 분산 설계가 단순해진다.

### 로컬 등록: `Process.register/2`, `GenServer.start_link(name: …)`

가장 기본적인 이름짓기.

```elixir
pid = spawn(fn ->
  receive do
    msg -> IO.inspect({:got, msg})
  end
end)

Process.register(pid, :echo)      # 현재 노드 안에서만 유효

send(:echo, "hello")
```

- `:echo` 라는 이름으로 pid가 **현재 노드의 로컬 레지스트리에 등록**된다.
- 다른 노드에서는 이 이름을 모른다.

GenServer에서 자주 쓰는 패턴:

```elixir
defmodule MyServer do
  use GenServer

  def start_link(opts \\ []) do
    GenServer.start_link(__MODULE__, :ok, Keyword.put_new(opts, :name, __MODULE__))
  end

  @impl true
  def init(:ok), do: {:ok, %{}}

  @impl true
  def handle_call(:ping, _from, state) do
    {:reply, :pong, state}
  end
end

{:ok, _pid} = MyServer.start_link()
GenServer.call(MyServer, :ping)
```

- 이름을 모듈명으로 주는 패턴은 매우 흔하다.
- 어디에서든 `MyServer` 이름만 알면 호출할 수 있다는 점이 장점.

---

### 원격 호출(로컬 이름 + 노드 튜플)

다른 노드에서 로컬 등록 이름을 이용하려면, `{:name, node}` 튜플을 사용한다.

```elixir
# b 노드에서 MyServer가 :my_server 로 등록되어 있다고 가정

GenServer.call({:my_server, :"b@<host>"}, {:echo, "hi"})
GenServer.cast({:my_server, :"b@<host>"}, {:notify, "hello"})
```

- `{:my_server, :"b@<host>"}` 는 “**b 노드에서 :my_server 이름을 가진 프로세스**”라는 의미의 **표준 주소 형태**.
- 이 방식은:
  - 노드가 명시적으로 드러나므로,
    라우팅/장애 처리/로그 분석 측면에서 **명확**하다.
  - 반면, 모든 호출자가 어떤 노드에 있는지 알아야 한다는 점이 단점 → 보통 **라우터**를 둔다.

---

### 전역 등록: `:global`

클러스터 전체에서 **유일한 이름**을 쓸 수 있도록 돕는 모듈.

```elixir
# 전역 등록 (클러스터 전체에서 유일)

{:ok, _} =
  GenServer.start_link(MyServer, :ok, name: {:global, :world_server})

# 어느 노드에서든

GenServer.call({:global, :world_server}, :ping)
```

- 내부적으로는 **합의/락**을 사용해 전체 클러스터에서 이름의 유일성을 유지한다.
- 장점:
  - 사용법이 간단하고, “이름 하나만 있으면 된다”는 점에서 편하다.
- 단점:
  - 노드 수가 커질수록 **전역 락/합의 비용**이 커질 수 있다.
  - 특정 노드 장애/넷스플릿 상황에서 복잡성이 증가한다.

실전에서는:

- 소규모 클러스터, 리더 선출 등 **한두 개의 전역 객체**에만 쓰는 경우가 많고,
- 나머지 대부분은 `Registry` 기반 로컬 등록 + 라우팅 계층을 선호한다.

---

### 로컬 레지스트리: `Registry` + `:via`

`Registry`는 OTP가 제공하는 **유연한 로컬 이름 레지스트리**다.

```elixir
# 애플리케이션 시작 시 (Supervisor 트리)

children = [
  {Registry, keys: :unique, name: MyReg}
]

Supervisor.start_link(children, strategy: :one_for_one)
```

서버 등록:

```elixir
defmodule RoomServer do
  use GenServer

  def start_link(room_id) do
    name = {:via, Registry, {MyReg, {:room, room_id}}}
    GenServer.start_link(__MODULE__, room_id, name: name)
  end

  @impl true
  def init(room_id), do: {:ok, %{id: room_id, messages: []}}
end

{:ok, _} = RoomServer.start_link(42)
```

호출:

```elixir
GenServer.call({:via, Registry, {MyReg, {:room, 42}}}, :ping)
```

장점:

- 이름을 **복합 키**로 만들 수 있다: `{:room, room_id}`, `{:user, uid, tenant}` 등.
- 동적으로 추가/제거되는 프로세스에 잘 어울린다.
- `Registry` 자체를 통해 현재 살아있는 프로세스를 질의할 수도 있다.

---

### 분산 라우팅 패턴(간단 샤딩)

여러 노드에 **키 기반 샤딩(sharding)** 을 하려면:

1. 키를 해시해서 노드를 결정하고,
2. 해당 노드의 Registry에 등록된 서버를 찾아 호출한다.

간단한 예:

```elixir
defmodule DistRegistry do
  def nodes do
    [node() | Node.list()] |> Enum.sort()
  end

  def target_node(key, nodes \\ nodes()) do
    idx = :erlang.phash2(key, length(nodes))
    Enum.at(nodes, idx)
  end

  def whereis_room(room_id) do
    node = target_node(room_id)
    addr = {:via, Registry, {MyReg, {:room, room_id}}}
    {addr, node}
  end
end

# 사용 예

{addr, nod} = DistRegistry.whereis_room(42)
GenServer.call({addr, nod}, {:post, "hello"})
```

- `:erlang.phash2/2` 는 균등한 해시 분포를 제공한다.
- 노드 목록이 변경되면 할당이 바뀌므로, 실제 서비스에서는:
  - **일관 해시(consistent hashing)**,
  - **메타데이터 기반 라우팅**,
  - **상태 이동(rebalancing)** 메커니즘이 필요해진다.

---

### 실습: 채팅방 서버 샘플

Registry 기반으로 간단한 채팅방 서버를 구현해 보자.

```elixir
defmodule Chat.Room do
  use GenServer

  def start_link(room_id) do
    name = {:via, Registry, {Chat.Reg, {:room, room_id}}}
    GenServer.start_link(__MODULE__, room_id, name: name)
  end

  def post(room_id, msg) do
    GenServer.cast(via(room_id), {:post, self(), msg})
  end

  def join(room_id) do
    GenServer.call(via(room_id), {:join, self()})
  end

  defp via(room_id), do: {:via, Registry, {Chat.Reg, {:room, room_id}}}

  @impl true
  def init(room_id), do: {:ok, %{id: room_id, members: MapSet.new()}}

  @impl true
  def handle_call({:join, pid}, _from, state) do
    {:reply, :ok, %{state | members: MapSet.put(state.members, pid)}}
  end

  @impl true
  def handle_cast({:post, from, msg}, state) do
    Enum.each(state.members, fn pid ->
      send(pid, {:room_msg, state.id, from, msg})
    end)

    {:noreply, state}
  end
end
```

Supervisor 설정:

```elixir
children = [
  {Registry, keys: :unique, name: Chat.Reg},
  {DynamicSupervisor, name: Chat.RoomSup, strategy: :one_for_one}
]

Supervisor.start_link(children, strategy: :one_for_one)
```

채팅방 생성/사용:

```elixir
{:ok, _} = DynamicSupervisor.start_child(Chat.RoomSup, {Chat.Room, 1})

# 클라이언트 A

Chat.Room.join(1)
Chat.Room.post(1, "안녕하세요")

receive do
  {:room_msg, room_id, from, msg} ->
    IO.puts("[#{room_id}] #{inspect(from)}: #{msg}")
end
```

- 이 구조를 **노드 샤딩**과 결합하면,
  - room_id 기반으로 **특정 노드에 채팅방을 배치**할 수 있다.
- 실전에서는 이 위에:
  - 외부 소켓/웹소켓,
  - 인증,
  - persist log 등을 얹어 전체 채팅 시스템을 만든다.

---

## 입력, 출력, PID, 노드

> 분산에서 **입력/출력(IO)** 과 **PID** 는 **노드 경계**를 의식해야 한다.

### PID와 노드

PID는 얼랭/엘릭서에서 **프로세스의 주소**를 의미한다.

```elixir
self()          # 현재 프로세스 PID, 예: #PID<0.123.0>
inspect(self()) # 문자열 표현 "#PID<0.123.0>"

node()          # 현재 노드명, 예: :"a@devbox"
node(self())    # 현재 프로세스가 속한 노드 (항상 node()와 동일)
```

원격 PID도 가능하다:

```elixir
# b 노드에서 pid를 생성

pid_on_b = :rpc.call(:"b@<host>", :erlang, :self, [])

node(pid_on_b)  # :"b@<host>"
is_pid(pid_on_b) # true

# 메시지 보내기

send(pid_on_b, {:hello, self()})
```

중요한 점:

- PID에는 **노드 정보가 포함**되어 있다.
- 노드 연결이 되어 있는 한, 원격 PID에도 **로컬 PID와 똑같이 메시지를 보낼 수 있다**.
- 노드가 끊어지면:
  - 메시지는 전달되지 않고 버려질 수 있으며,
  - 링크/모니터를 통해 장애를 감지해야 한다.

---

### 메시지 순서 보장과 장애

BEAM의 메시지 순서 보장 규칙:

- **같은 송신자 → 같은 수신자** 경로에서는 **메시지 순서가 보존**된다.
  - A가 B에게 `m1`, `m2` 를 이 순서로 보냈다면, B는 항상 `m1`, `m2` 순서로 받는다.
- 다른 송신자 여러 개가 한 수신자에게 메시지를 보낼 경우:
  - 송신자 간의 **상대적인 순서**는 보장되지 않는다.

분산에서의 추가 고려사항:

- 노드 연결이 끊어지면:
  - 그 이후에 보낸 메시지는 도달하지 않는다.
  - 그 이전에 보낸 메시지는 도착했는지 여부를 **애플리케이션 레벨**에서 판단해야 한다.
- 재연결 후에 메시지 순서에 대한 기대를 유지하려면:
  - **시퀀스 번호/타임스탬프**를 메시지에 포함해 도메인 레벨에서 정합성을 맞추는 설계가 필요하다.

예: 간단한 시퀀스 번호 적용

```elixir
send(pid, {:seq_msg, seq, payload})
```

수신 측에서:

- seq가 1,2,3,... 순으로 도착하는지 검사하고,
- 누락/중복을 발견하면 재요청이나 로그를 남길 수 있다.

---

### 입력/출력과 `group_leader`

모든 프로세스는 **group leader** 라는 프로세스를 가진다.

- `IO.puts/1`, `IO.inspect/2` 등은 실제로는:
  - 자신의 group leader 프로세스에 메시지를 보내서,
  - 그 group leader가 최종적으로 **셸/파일** 등에 출력한다.

기본적으로:

- 셸에서 스폰한 프로세스들의 group leader는 **해당 셸 프로세스**이다.

```elixir
# group_leader 확인

Process.group_leader()

# group_leader 변경

parent_gl = Process.group_leader()
spawn(fn ->
  Process.group_leader(self(), parent_gl)
  IO.puts("이 출력은 부모 셸에 나타난다")
end)
```

분산에서:

- 원격에서 실행한 코드의 출력이,
  - 원격 노드의 셸에 찍히거나,
  - 우리가 원하는 로컬 셸에 찍히거나 하는 차이가 생긴다.

예: a 노드에서 b 노드의 group leader를 a 셸로 설정

```elixir
# a 노드

gl = Process.group_leader()

:rpc.call(:"b@<host>", fn ->
  # 원격에서 현재 프로세스를 하나 만들어 group_leader를 전달
  spawn(fn ->
    Process.group_leader(self(), gl)
    IO.puts("이 출력은 a 노드 셸에 나타난다")
  end)
end)
```

실전에서는:

- 표준 출력 대신 **Logger/Telemetry** 기반으로 로그를 수집하고,
- 필요하다면 group leader를 이용해 특정 IO를 특정 노드로 리다이렉트하는 식으로 설계한다.

---

### 원격 스폰과 작업 실행

원격 노드에 프로세스를 띄우는 방법은 여러 가지가 있다.

#### 1) `Node.spawn/2`

```elixir
pid =
  Node.spawn(:"b@<host>", fn ->
    send({:echo, node()}, {:hello, self()})
  end)
```

- `Node.spawn(node, fun)`:
  - 지정된 노드에서 `fun`을 실행하는 프로세스를 생성한다.
- 이때 group leader는 **그 노드의 셸**이 된다.

#### 2) `Task.Supervisor` 를 통한 원격 태스크

원격 작업을 관리하려면 `Task.Supervisor` 가 편리하다.

```elixir
# a 노드에서

{:ok, _} = Task.Supervisor.start_link(name: MyTaskSup)

t =
  Task.Supervisor.async_nolink({MyTaskSup, :"b@<host>"}, fn ->
    Enum.sum(1..10_000)
  end)

Task.await(t, 5_000)
```

- `async_nolink/3` 는 태스크가 죽어도 호출자에게 **EXIT 신호를 전파하지 않는다**.
- 연결 단절/노드 다운 상황 등에서:
  - `Task.await/2` 는 `exit` 를 발생시킬 수 있고,
  - 호출자는 `try/rescue` 나 Supervisor 정책으로 이를 처리할 수 있다.

---

### 간단한 분산 RPC 래퍼

`:rpc.call/5` 는 강력하지만, 에러 처리/타임아웃 등을 동일 패턴으로 쓰기 위해 래퍼를 만드는 편이 좋다.

```elixir
defmodule DRPC do
  @default_timeout 5_000

  def call(node, mod, fun, args, timeout \\ @default_timeout) do
    case :rpc.call(node, mod, fun, args, timeout) do
      {:badrpc, reason} -> {:error, reason}
      ok -> {:ok, ok}
    end
  end
end

# 사용

DRPC.call(:"b@<host>", Enum, :max, [[1, 9, 3]])
# => {:ok, 9} 혹은 {:error, reason}

```

이렇게 해두면:

- 상위 레이어에서는 `with` 구문으로 쉽게 합성할 수 있다.

```elixir
with {:ok, max} <- DRPC.call(:"b@<host>", Enum, :max, [[1, 9, 3]]),
     {:ok, min} <- DRPC.call(:"b@<host>", Enum, :min, [[1, 9, 3]]) do
  {:ok, max - min}
else
  {:error, reason} -> {:error, {:remote_failure, reason}}
end
```

---

## 노드는 분산 처리의 기본이다

> BEAM의 분산은 **위치 투명성(location transparency)** 과 **프로세스 격리**를
> “많은 작은 것들이 서로 메시지로 대화한다”는 모델을 **노드 간**으로 확장한다.

### 설계 원칙 요약

노드를 어떻게 나눌지, 그 위에 프로세스를 어떻게 배치할지에 대한 몇 가지 원칙:

1. **경계 == 노드**

   - 장애 격리를 위해 기능을 적어도 **노드 단위**로 나눈다.
   - 예: API 노드 / 배치 노드 / 백그라운드 워커 노드 / 캐시 노드 등.

2. **발견/라우팅**

   - 프로세스 이름(Registry/글로벌)과 키→노드 해시/메타데이터를 이용해,
   - 요청이 **어느 노드의 어떤 프로세스**로 가야 하는지 명확히 설계한다.

3. **일관된 실패 신호**

   - 링크/모니터/타임아웃을 이용해,
   - 노드/프로세스 장애를 **일관된 형태(reason)** 로 상위에 전달한다.

4. **역압**

   - 외부 입력(HTTP 큐, 메시지 큐 등)부터 내부 프로세스까지,
   - **`call`/큐 한도/배치/GenStage/Broadway** 등을 통해 **과부하를 제어**한다.

5. **관찰 가능성**

   - Telemetry/Logger로 노드·프로세스 지표를 수집하고,
   - 대시보드에서 **요청량, 지연, 에러율, 메일박스 길이, 레덕션** 등을 모니터링한다.

---

### 미니 프로젝트: 분산 키-값 저장소(샤딩)

초안의 구조를 그대로 확장해보자. 목표:

- 여러 노드에 키-값 저장소를 **샤딩(sharding)**.
- 클라이언트는 **키만 알고** `put/2`, `get/1` 을 호출.
- 라우터가 키→노드 결정, 각 노드에서 해당 키 전담 **프로세스**가 값을 관리.

#### Router: 키 → 노드 매핑

```elixir
defmodule Dist.Router do
  # 현재 클러스터 노드 목록
  def nodes do
    [node() | Node.list()] |> Enum.sort()
  end

  # 키를 해시해서 특정 노드 선택
  def shard(key, nodes \\ nodes()) do
    idx = :erlang.phash2(key, length(nodes))
    Enum.at(nodes, idx)
  end
end
```

- 샤딩은 단순히 “키를 해시해서 특정 인덱스를 고르는” 방식.
- 실제 서비스에서는:
  - **일관 해시**,
  - **메타데이터 테이블** (예: “파티션 0~99는 노드 A,B”) 를 사용하는 경우가 많다.

#### KVServer: 키별 프로세스

```elixir
defmodule Dist.KVServer do
  use GenServer

  def start_link(key) do
    name = {:via, Registry, {KVReg, {:kv, key}}}
    GenServer.start_link(__MODULE__, {key, nil}, name: name)
  end

  def put(addr, v), do: GenServer.call(addr, {:put, v})
  def get(addr), do: GenServer.call(addr, :get)

  @impl true
  def init({key, _}) do
    {:ok, %{key: key, value: nil}}
  end

  @impl true
  def handle_call({:put, v}, _from, st) do
    {:reply, :ok, %{st | value: v}}
  end

  @impl true
  def handle_call(:get, _from, st) do
    {:reply, st.value, st}
  end
end
```

- 각 키마다 `Dist.KVServer` 프로세스 하나가 담당.
- 이 프로세스는:
  - 해당 키의 값을 메모리에 들고 있고,
  - 필요하다면 지속성(파일/DB/로그 등)을 추가할 수 있다.

#### Client: 키만 알고 사용

```elixir
defmodule Dist.Client do
  def put(key, v) do
    node = Dist.Router.shard(key)
    addr = via(key)
    ensure_started(node, key)
    GenServer.call({addr, node}, {:put, v})
  end

  def get(key) do
    node = Dist.Router.shard(key)
    addr = via(key)
    ensure_started(node, key)
    GenServer.call({addr, node}, :get)
  end

  defp via(key), do: {:via, Registry, {KVReg, {:kv, key}}}

  defp ensure_started(node, key) do
    :rpc.call(node, Dist.Boot, :ensure, [key])
  end
end
```

- 클라이언트는:
  - 키만 알고 있으며, 나머지는 Router/Boot가 해준다.

#### Boot 모듈: 서버 존재 보장

```elixir
defmodule Dist.Boot do
  def ensure(key) do
    case Registry.lookup(KVReg, {:kv, key}) do
      [] ->
        DynamicSupervisor.start_child(KVSup, {Dist.KVServer, key})

      _ ->
        :ok
    end
  end
end
```

Supervisor 트리:

```elixir
children = [
  {Registry, keys: :unique, name: KVReg},
  {DynamicSupervisor, name: KVSup, strategy: :one_for_one}
]

Supervisor.start_link(children, strategy: :one_for_one)
```

#### 사용 예

```elixir
# 노드 a, b가 연결되어 있다고 가정

Dist.Client.put("user:42", "kim")
Dist.Client.get("user:42")  # "kim"
```

- `"user:42"` 키는 Router가 정한 특정 노드에 항상 배치된다.
- 같은 키에 대한 모든 요청은 **같은 프로세스**로 가므로,
  - 해당 키에 대한 변경/일관성을 **프로세스 단위로 보장**할 수 있다.

---

### 장애/분할 네트워크(넷스플릿) 대비

분산 시스템에서 피할 수 없는 문제:

- 노드 다운
- 네트워크 단절(net split)
- RPC 실패/타임아웃

이를 처리하기 위한 일반적인 패턴:

1. **모니터링**

   - `Node.list/0` 의 변화를 주기적으로 체크하거나,
   - `:net_kernel` 이벤트를 구독해 노드 연결/해제를 감지한다.
   - RPC 실패율을 Telemetry 지표로 관찰한다.

2. **정책**

   - 쓰기 작업은:
     - “리더 노드” 개념을 두고, 일관성 강화를 위해 리더만 쓰기 허용,
     - 혹은 이벤트 소싱/로그 기반으로 기록 후 나중에 재플레이.
   - 읽기 작업은:
     - 캐시/복제를 통해 어느 노드에서든 처리 가능하도록 할 수도 있다.

3. **타임아웃 + 재시도**

   ```elixir
   defmodule RetryRPC do
     def with_retry(fun, tries \\ 3)
     def with_retry(fun, 0), do: {:error, :exhausted}

     def with_retry(fun, tries) do
       case fun.() do
         {:ok, v} -> {:ok, v}
         _ ->
           Process.sleep(1_000)
           with_retry(fun, tries - 1)
       end
     end
   end

   # 예:
   RetryRPC.with_retry(fn ->
     DRPC.call(:"b@<host>", Enum, :max, [[1, 9, 3]])
   end)
   ```

4. **멱등(idempotent) 작업 설계**

   - 재시도가 가능한 작업은,
   - 같은 요청이 여러 번 들어와도 **한 번 처리한 것과 동일한 결과**가 되도록 설계하는 것이 좋다.

---

### 관찰 지표(권장)

분산 시스템에서 **무엇을 모니터링해야 하는지**는 구조만큼 중요하다.

- 노드 단위:
  - 스케줄러 수 / 활용률
  - 초당 레덕션(reductions/sec)
  - 메모리 사용량, ETS 테이블 수
  - 최대/평균 메일박스 길이

- 분산 레벨:
  - 연결된 노드 수, 연결/해제 이벤트 수
  - RPC 호출 횟수, 실패율, 평균/최대 지연 시간
  - 노드 간 네트워크 지연

- 서비스 레벨:
  - 키→노드 라우팅 skew(특정 노드에 과도하게 키가 몰리는지)
  - 재배치/리밸런싱 횟수
  - 재시도/타임아웃 카운트

이 지표들을 Telemetry 이벤트로 발행하고:

- Prometheus/Grafana,
- Phoenix LiveDashboard 등으로 시각화하면,
- 운영 중 이상 상황을 **빠르게 파악**할 수 있다.

---

### 부록: 실습 레시피 모음

#### A) 빠른 로컬 클러스터 실습

```bash
# 터미널 1

iex --sname a --cookie CHOCO -S mix

# 터미널 2

iex --sname b --cookie CHOCO -S mix
```

a 노드에서:

```elixir
Node.connect(:"b@#{:net_adm.localhost()}")
Node.list()

:rpc.call(:"b@#{:net_adm.localhost()}", IO, :puts, ["hello from a"])
```

#### B) 원격 태스크 + 출력 리다이렉션

```elixir
# a 노드

{:ok, _} = Task.Supervisor.start_link(name: MyTaskSup)
gl = Process.group_leader()

t =
  Task.Supervisor.async_nolink({MyTaskSup, :"b@<host>"}, fn ->
    Process.group_leader(self(), gl)
    IO.puts("remote says hi on local shell")
  end)

Task.await(t)
```

#### C) 원격 GenServer 생성/호출

```elixir
# b 노드에 MyServer를 띄우기

:rpc.call(:"b@<host>", GenServer, :start_link, [MyServer, :ok, [name: :calc]])

# a 노드에서 호출

GenServer.call({:calc, :"b@<host>"}, {:sum, [1, 2, 3]})
```

---

### 흔한 함정 → 교정

1) **쿠키 불일치**

   - 증상: `Node.connect/1` 실패, `:nodedown` 에러.
   - 교정: 실행 옵션/릴리즈 설정에서 쿠키를 환경별로 **일관되게 관리**.

2) **로컬 이름을 클러스터 전체에서 유일하다고 착각**

   - 증상: 다른 노드에서 `GenServer.call(:name, ...)` 가 `:noproc` 에러.
   - 교정: `{:name, node}` 튜플, `:global`, `:via Registry` 를 올바르게 사용.

3) **표준 출력에만 의존**

   - 증상: 원격 워커의 로그가 어떤 셸/파일에 찍히는지 혼란, 분산 환경에서 로그 추적 난이도 증가.
   - 교정: Logger/Telemetry 기반 중앙 로깅, 필요 시 `group_leader` 로 IO 재지정.

4) **메시지 손실·순서 혼동**

   - 증상: 넷스플릿/재연결 후 상태 불일치, 이벤트 순서 꼬임.
   - 교정: 링크/모니터/타임아웃/재시도, 메시지에 시퀀스 번호/버전 추가, 멱등 설계.

5) **중앙 전역 등록에 모든 걸 의존**

   - 증상: 노드 수 증가 시 전역 네임 서비스가 병목/지연 요인이 됨.
   - 교정: 로컬 Registry와 라우팅/샤딩 구조를 도입해 **이름 해석을 분산**.

---

### 연습 문제

1) **분산 카운터**

   - 키별 카운터를 샤딩하여 각 노드에서 `inc/1`, `get/1` 을 제공하라.
   - 넷스플릿 후 두 클러스터가 다른 값을 갖게 됐을 때,
     - 병합 정책(최대/최소/합산 등)을 설계하고 코드로 구현해 보라.

2) **원격 스트리밍 파이프라인**

   - `Task.async_stream/3` 을 사용하되, 입력을 노드별 파티션으로 나눠
     - 각 노드에서 병렬 작업을 수행하고,
     - 결과를 로컬 노드에서 다시 합치는 파이프라인을 구현하라.

3) **장애 주입 테스트**

   - 특정 노드에 강제 종료(`:init.stop/0`)를 호출하고,
   - 클라이언트가 재시도/페일오버를 통해 어느 정도까지 견딜 수 있는지 테스트하라.

4) **서비스 디스커버리 어댑터**

   - 정적 리스트 대신, 외부 디스커버리 시스템에서 노드 목록을 받아와
     `Dist.Router.nodes/0` 를 갱신하는 모듈을 설계해 보라(구현은 Mock로 대체해도 좋다).

5) **관찰 보강**

   - RPC 지연/실패율을 `[:dist, :rpc, :stop]` Telemetry 이벤트로 발행하고,
   - 대시보드에서 **노드별 RPC 지표**를 비교할 수 있도록 구성하라.

---

### 마무리

- 노드에 이름을 붙이고 쿠키를 맞추는 순간,
  프로세스/메시지/슈퍼비전의 모델이 **단일 VM**에서 **클러스터 전체**로 확장된다.
- PID, IO, group leader, Registry, :global, RPC 를 이해하면,
  **분산 환경에서도 로컬 프로세스와 거의 같은 추상화로 사고**할 수 있다.
- 최종적으로 분산 설계는:
  - **발견/라우팅/역압/장애 감지/관찰**을 함께 설계하는 일이다.
- 이 장에서 다룬 예제들을 실습 프로젝트에 옮겨보면,
  “노드는 분산 처리의 기본 단위”라는 말이 **구체적인 코드와 구조로 체감**될 것이다.
