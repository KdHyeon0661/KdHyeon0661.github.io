---
layout: post
title: Elixir - Duper
date: 2025-11-20 21:25:23 +0900
category: Elixir
---
# Duper

## Duper 소개

### 문제 정의 — “내용이 완전히 같은 파일” 찾기

**Duper**는 **디렉터리 트리 전체**를 순회하면서 **중복 파일(내용이 완전히 같은 파일)** 을 찾아주는 도구다.

- 입력:
  - 하나 이상의 **루트 디렉터리**
    - 예: `~/Pictures`, `/data`, `/mnt/backup`
  - 옵션:
    - 최소 파일 크기, 해시 알고리즘, 헤더 크기, 워커 수 등
- 출력:
  - **내용 해시가 동일한 파일 그룹 목록**
  - 각 그룹은 **2개 이상의 경로(path)** 로 구성
  - 예:
    ```text
    Size: 3_145_728, Algo: SHA-256
      Hash: 9A7F...E21B
        /data/photos/2020/IMG_0001.JPG
        /data/backup/photos/IMG_0001_copy.JPG
    ```

“내용이 완전히 같다”는 의미는 보통 **바이트 스트림이 완전히 동일**하다는 뜻이다.
Duper는 이를 위해 **내용 해시(Content Hash)** 를 사용한다.

- 주로 사용하는 해시 알고리즘:
  - 보편적 예: SHA-256
  - 성능이 더 중요하면 **비암호학적 해시(xxhash64, blake3 등)** 도 옵션이 될 수 있다.
- 중복 판단 방식:
  1. 먼저 **파일 크기(size)** 로 거칠게 그룹핑
  2. 같은 크기 내에서 **헤더 일부(head hash)** 로 후보 축소
  3. 마지막으로 전체 파일에 대해 **스트리밍 해시(full hash)** 를 계산해 최종 확정

이렇게 하면, **작업량 대부분을 “중복 가능성이 높은 후보”에 집중**시키면서
정확한 중복 탐지를 할 수 있다.

---

### 핵심 난제 — 단순 “find + sha256sum” 으로는 부족한 이유

현실 시스템에서는 다음과 같은 난제가 있다.

1. **파일 수가 너무 많다**
   - 수십만 ~ 수백만 개 파일이 흔하다.
   - 단일 프로세스에서 순차 I/O로 처리하면 시간이 너무 오래 걸린다.
   - **I/O 병렬성**(여러 워커가 동시에 해시)과 **역압(backpressure)** 설계가 필요.

2. **대용량 파일**
   - 개별 파일 크기가 수~수십 GB인 경우,
     - 파일 하나를 해싱하는 데도 시간이 오래 걸리고,
     - 잘못된 설계로 한 번에 메모리로 읽으면 OOM을 유발할 수 있다.
   - 반드시 **스트리밍/청크 기반으로 해시**해야 한다.

3. **같은 크기지만 다른 내용**
   - 파일 크기만 가지고는 중복을 확정할 수 없다.
   - 1단계: 크기 필터
   - 2단계: 헤더 N KiB만 읽어 **헤더 해시** (이때도 충돌 가능성 존재)
   - 3단계: 후보 그룹에 대해서만 전체 해시
   - 이렇게 **단계별 정밀도(upgrade)** 를 적용해야 규모가 커져도 감당 가능하다.

4. **인덱스 관리**
   - `(size, head_hash?, full_hash) → [paths...]` 형태의 인덱스를 유지해야 한다.
   - 중복이 많을수록 리스트 길이가 길어질 수 있다.
   - 단순한 Map만으로는 **메모리/동시성 문제가 발생**할 수 있으므로,
     - **ETS**(Erlang Term Storage)를 적극 활용하고,
     - **읽기 병렬, 쓰기 직렬** 구조를 설계해야 한다.

5. **중단 및 재개**
   - 장시간(수 시간 이상) 작업이 일반적이다.
   - 중간에 프로세스가 죽거나 시스템이 재부팅되면 **재시작 전략**을 생각해야 한다.
   - 완전한 중단/재개(Checkpoint/Resume)를 지원하려면:
     - 워크 큐 상태를 주기적으로 디스크에 저장하거나,
     - “어느 시점까지 스캔했는지” 로그를 남겨야 한다.

6. **파일 변경 레이스**
   - 해시 계산 중에 파일이 변경될 수 있다.
   - 이를 무시하면, “중복이라고 출력했지만 실제로는 달라진 파일”이 생길 수 있다.
   - mtime/ino를 함께 저장하고, **결과 출력 시 다시 확인**하는 전략이 필요하다.

이 난제들을 해결하기 위해 Duper는 **OTP/BEAM의 동시성 모델**을 적극 활용한다.
즉, 전체 시스템을 여러 **프로세스(GenServer)** 로 나누고,
**슈퍼비전 트리**로 안정성을 확보한다.

---

### 성능 직관 — 큐잉 이론으로 보는 “병목”과 “역압”

Duper의 파이프라인은 본질적으로 “**작업 큐 + 워커**” 구조를 가진다.

- 파일이 발견될 때마다 작업(“해시를 계산하라”)이 큐에 쌓이고,
- 해시 워커 풀에서 이 큐를 소비한다.

여기서 처리율을 \(\mu\), 단위 시간당 들어오는 작업 수를 \(\lambda\)라고 하자.
단순 M/M/1 큐(작업 도착/서비스가 포아송/지수 분포라고 가정)에서:

- 시스템 부하계수:
  $$
  \rho = \frac{\lambda}{\mu}
  $$
- 평균 큐 길이:
  $$
  L_q \approx \frac{\rho^2}{1 - \rho}
  $$
- 평균 대기 시간:
  $$
  W_q \approx \frac{\rho}{\mu(1 - \rho)}
  $$

여기서 핵심 직관:

- \(\rho \to 1\)에 가까워질수록, \(L_q\)와 \(W_q\)가 **폭발적으로 커진다**.
- 즉, **처리율 \(\mu\)와 도착율 \(\lambda\)** 사이에 **여유가 조금만 있어도** (예: \(\rho=0.7\)),
  대기열이 안정적이다.
- 반대로 \(\rho=0.95\) 수준이면, 평균큐 길이는 작은 변화에도 크게 출렁인다.

Duper 설계에서 할 일은 명확하다.

1. 각 스테이지(해시, 인덱스, I/O)의 **실제 처리율 \(\mu\)** 를 측정한다.
2. 입력(파일 수, 파일 생성 속도 등)에 따른 \(\lambda\)를 추정한다.
3. **워커 수 조절, 배치 처리, 캐시** 등을 통해 **\(\lambda \le \mu\)** 를 유지한다.
4. Telemetry로 큐 길이와 처리율을 측정해, **실제 \(\rho\)가 어디쯤인지** 관찰한다.

---

## Duper 애플리케이션 설계

우리는 Duper를 **파이프라인**으로 설계한다.

```text
[Scanner] --(파일 경로)--> [Stat Filter] --(후보)--> [Head Hasher] --(헤더 해시)--> [Full Hasher] --(콘텐츠 해시)--> [Index] --(중복 그룹)
```

각 스테이지는 **서로 다른 OTP 프로세스**(GenServer, DynamicSupervisor 등)로 구현한다.

- **Scanner**: 디렉터리 트리를 순회하면서 파일 경로를 스트림으로 생산
- **Stat Filter**: `File.stat/1`을 통해 정규 파일/크기 조건 등으로 필터링
- **Head Hasher**: 파일의 **처음 N KiB**만 읽어 “헤더 해시”를 계산
- **Full Hasher**: 동일 크기·헤더 해시 그룹에 대해서만 전체 해시
- **Index**: ETS + GenServer로 `(size, head, full) → [paths...]` 인덱스 관리

### 전체 구조 스케치

텍스트로 간단히 트리를 그려보면:

```text
Duper.Supervisor (root, :rest_for_one)
  ├─ Registry (Duper.Reg)
  ├─ Duper.Index (GenServer + ETS)
  ├─ Duper.ScanSup (DynamicSupervisor; Scanner 워커)
  ├─ Duper.HashSup (DynamicSupervisor; HeadHasher/FullHasher 워커)
  └─ Duper.Pipeline.Supervisor (Scanner/Filter/Hasher/Sink 조립)
```

- `Duper.Application`은 이 루트 슈퍼바이저를 애플리케이션 시작점으로 등록한다.
- 각 스테이지는 `Duper.Pipeline.Supervisor` 아래에서 `:rest_for_one` 전략으로 묶인다.
  - HeadHasher에서 문제 발생 → FullHasher 이후만 재시작
  - Index가 죽으면, Index만 재시작(상위 전략에 따라 달라질 수 있음)

---

### Mix 스캐폴딩 — 프로젝트 뼈대 만들기

```bash
mix new duper --sup
cd duper
```

`--sup` 옵션은 `Application` 모듈과 기본 슈퍼비전 트리를 포함한 템플릿을 생성한다.

`mix.exs` 예시:

```elixir
defmodule Duper.MixProject do
  use Mix.Project

  def project do
    [
      app: :duper,
      version: "0.1.0",
      elixir: "~> 1.17",
      start_permanent: Mix.env() == :prod,
      deps: deps()
    ]
  end

  def application do
    [
      extra_applications: [:logger, :crypto],
      mod: {Duper.Application, []}
    ]
  end

  defp deps do
    [
      {:telemetry, "~> 1.2"},
      # 선택: 개발 중 파일 변경 감지하거나, 향후 실시간 모니터링에 활용 가능
      {:file_system, "~> 1.0", only: :dev}
    ]
  end
end
```

- `:crypto` 는 SHA-256 해시 계산용.
- `:telemetry` 는 성능/에러/큐 길이 관측용.

---

### 애플리케이션 루트와 슈퍼비전 트리

`lib/duper/application.ex`:

```elixir
defmodule Duper.Application do
  @moduledoc false
  use Application

  def start(_type, _args) do
    children = [
      {Registry, keys: :unique, name: Duper.Reg},
      Duper.Index,
      {DynamicSupervisor, name: Duper.ScanSup, strategy: :one_for_one},
      {DynamicSupervisor, name: Duper.HashSup, strategy: :one_for_one},
      Duper.Pipeline.Supervisor
    ]

    opts = [strategy: :rest_for_one, name: Duper.Supervisor]
    Supervisor.start_link(children, opts)
  end
end
```

- `Registry`:
  - 각 스테이지/워커가 자신의 이름을 등록해, **동적 주소 찾기**에 활용할 수 있다.
- `Duper.Index`:
  - 파일 해시들을 모으는 중앙 인덱스 서버.
- `Duper.ScanSup`, `Duper.HashSup`:
  - Scanner/Hasher 워커를 동적으로 올렸다 내리기 위한 DynamicSupervisor.
- `Duper.Pipeline.Supervisor`:
  - Scanner → StatFilter → HeadHasher → FullHasher → Sink 를 조립하는 Supervisor.

`strategy: :rest_for_one` 을 사용한 이유:

- 파이프라인 상에서 뒤쪽 스테이지가 죽으면,
  - 그 스테이지와 **그 이후에 시작된 스테이지만** 재시작하는 것이 자연스럽다.
- 예를 들어 FullHasher가 죽으면,
  - FullHasher와 Sink만 재시작하고 Scanner/StatFilter는 그대로 유지할 수 있다.

---

### Index — ETS + GenServer로 중복 그룹 관리

`lib/duper/index.ex`:

```elixir
defmodule Duper.Index do
  use GenServer

  @table :duper_index

  # Public API
  def start_link(_opts),
    do: GenServer.start_link(__MODULE__, :ok, name: __MODULE__)

  def put(size, head, full, path),
    do: GenServer.call(__MODULE__, {:put, size, head, full, path})

  def groups do
    :ets.tab2list(@table)
    |> Enum.map(fn {{size, head, full}, paths} -> {size, head, full, paths} end)
  end

  def group_stream do
    Stream.resource(
      fn -> :ets.first(@table) end,
      fn
        :"$end_of_table" -> {:halt, nil}
        key ->
          next = :ets.next(@table, key)
          {[{key, elem(List.first(:ets.lookup(@table, key)), 1)}], next}
      end,
      fn _ -> :ok end
    )
  end

  # Callbacks
  @impl true
  def init(:ok) do
    :ets.new(@table, [
      :set,
      :public,
      :named_table,
      read_concurrency: true,
      write_concurrency: true
    ])

    {:ok, %{put_count: 0}}
  end

  @impl true
  def handle_call({:put, size, head, full, path}, _from, st) do
    key = {size, head, full}

    paths =
      case :ets.lookup(@table, key) do
        [] ->
          [path]

        [{^key, ps}] ->
          [path | ps]
      end

    :ets.insert(@table, {key, paths})

    {:reply, length(paths), %{st | put_count: st.put_count + 1}}
  end
end
```

설명:

- 인덱스의 키: `{size, head_hash, full_hash}`
  - `head_hash`가 `nil`인 구조도 설계 가능하지만, 여기서는 “헤더 해시 후 전체 해시까지 완료된” 상태를 표현한다.
- 값: 해당 해시를 가진 `paths` 리스트.
- `put/4`:
  - 새로운 파일이 들어오면 리스트에 추가하고, **현재 리스트 길이**를 반환한다.
  - 길이가 2가 되는 순간이 “중복이 처음 발견되는 시점”이다.
- `groups/0`:
  - 모든 그룹을 리스트로 반환.
- `group_stream/0`:
  - 결과가 매우 많을 수 있으므로, **스트리밍** 인터페이스도 제공.

---

### Scanner — 디렉터리 트리 순회

간단 버전(와일드카드 기반):

```elixir
defmodule Duper.Scanner do
  use GenServer

  def start_link(opts),
    do: GenServer.start_link(__MODULE__, opts, name: Keyword.get(opts, :name))

  @impl true
  def init(opts) do
    roots = Keyword.fetch!(opts, :roots)
    downstream = Keyword.fetch!(opts, :downstream)

    send(self(), {:walk, roots, downstream})

    {:ok, %{downstream: downstream}}
  end

  @impl true
  def handle_info({:walk, roots, downstream}, st) do
    for root <- roots,
        path <- Path.wildcard(Path.join([root, "**", "*"]), match_dot: true) do
      if File.regular?(path) do
        send(downstream, {:file, path})
      end
    end

    send(downstream, :eof)
    {:noreply, st}
  end
end
```

이 버전은 이해하기 쉽지만, 파일 수가 **매우 많을 때(Path.wildcard 전체 리스트가 메모리에 올라감)** 는 비효율적일 수 있다.

보다 안전한 DFS/큐 기반 버전:

```elixir
defmodule Duper.Scanner.DFS do
  use GenServer

  def start_link(opts),
    do: GenServer.start_link(__MODULE__, opts, name: Keyword.get(opts, :name))

  @impl true
  def init(opts) do
    roots = Keyword.fetch!(opts, :roots)
    downstream = Keyword.fetch!(opts, :downstream)

    send(self(), {:walk, roots})
    {:ok, %{downstream: downstream}}
  end

  @impl true
  def handle_info({:walk, roots}, %{downstream: downstream}=st) do
    Enum.each(roots, &walk(&1, downstream))
    send(downstream, :eof)
    {:noreply, st}
  end

  defp walk(path, downstream) do
    case File.lstat(path) do
      {:ok, %File.Stat{type: :directory}} ->
        case File.ls(path) do
          {:ok, entries} ->
            Enum.each(entries, fn entry ->
              walk(Path.join(path, entry), downstream)
            end)

          {:error, _} ->
            :ok
        end

      {:ok, %File.Stat{type: :regular}} ->
        send(downstream, {:file, path})

      _ ->
        :ok
    end
  end
end
```

- `File.lstat/1`을 사용하면 심볼릭 링크를 별도로 처리할 수 있다.
- 필요하다면, 심볼릭 링크를 따라갈지 여부를 옵션으로 두고, 무한 루프(링크 사이클)를 감지하기 위한 방문 집합도 유지할 수 있다.

---

### Stat Filter — 파일 메타데이터 필터링 및 후보 선정

StatFilter는 “파일 경로 → (크기, 경로)”로 변환하고,
0바이트/비정규 파일/권한 없는 파일 등을 제외한다.

```elixir
defmodule Duper.StatFilter do
  use GenServer

  def start_link(opts),
    do: GenServer.start_link(__MODULE__, opts, name: Keyword.get(opts, :name))

  @impl true
  def init(opts) do
    downstream = Keyword.fetch!(opts, :downstream)
    {:ok, %{downstream: downstream, seen: 0, skipped: 0}}
  end

  @impl true
  def handle_info({:file, path}, %{downstream: downstream}=st) do
    case File.stat(path) do
      {:ok, %File.Stat{size: size, type: :regular}} when size > 0 ->
        send(downstream, {:candidate, size, path})
        {:noreply, %{st | seen: st.seen + 1}}

      _ ->
        {:noreply, %{st | skipped: st.skipped + 1}}
    end
  end

  @impl true
  def handle_info(:eof, %{downstream: downstream}=st) do
    send(downstream, {:eof, :stat})
    {:noreply, st}
  end
end
```

필요하다면 다음과 같은 정책도 추가할 수 있다.

- 특정 확장자만 허용(예: `.jpg`, `.png`, `.heic` 등)
- 특정 디렉터리를 제외(예: `.git`, `node_modules`)
- 최소/최대 파일 크기 필터(예: `size >= 1_000_000`)

---

### Head Hasher — 헤더 일부만 빠르게 해시

HeadHasher는 “대부분의 파일이 서로 다르다”는 가정하에,
**파일의 처음 N KiB만 읽어 해시**를 계산한다.

```elixir
defmodule Duper.HeadHasher do
  use GenServer

  def start_link(opts),
    do: GenServer.start_link(__MODULE__, opts)

  @impl true
  def init(opts) do
    downstream = Keyword.fetch!(opts, :downstream)
    header_bytes = Keyword.get(opts, :header_bytes, Duper.Config.header_bytes())
    algo = Keyword.get(opts, :algo, Duper.Config.algo())
    {:ok, %{downstream: downstream, header_bytes: header_bytes, algo: algo}}
  end

  @impl true
  def handle_info({:candidate, size, path}, st) do
    %{downstream: downstream, header_bytes: n, algo: algo} = st

    head_hash =
      case File.open(path, [:read]) do
        {:ok, io} ->
          try do
            bin = IO.binread(io, n)
            :crypto.hash(algo, bin)
          after
            File.close(io)
          end

        {:error, _reason} ->
          nil
      end

    if is_binary(head_hash) do
      send(downstream, {:head, size, head_hash, path})
    end

    {:noreply, st}
  end

  @impl true
  def handle_info({:eof, :stat}, %{downstream: downstream}=st) do
    send(downstream, {:eof, :head})
    {:noreply, st}
  end
end
```

- `header_bytes`는 설정값으로 주입 가능.
- `algo`도 설정으로 주입(SHA-256 기본).
- 헤더만 읽기 때문에, 대용량 파일이라도 비교적 빠르게 1차 필터를 수행할 수 있다.

---

### Full Hasher — 스트리밍 전체 해시

**동일 크기·동일 헤더 해시** 그룹에 대해서만 전체 해시를 수행한다.

```elixir
defmodule Duper.FullHasher do
  use GenServer

  @chunk 1 * 1024 * 1024

  def start_link(opts),
    do: GenServer.start_link(__MODULE__, opts, name: Keyword.get(opts, :name))

  @impl true
  def init(opts) do
    downstream = Keyword.fetch!(opts, :downstream)
    algo = Keyword.get(opts, :algo, Duper.Config.algo())
    {:ok, %{index: %{}, downstream: downstream, algo: algo}}
  end

  @impl true
  def handle_info({:head, size, head, path}, %{index: idx}=st) do
    key = {size, head}
    group = Map.get(idx, key, [])
    idx = Map.put(idx, key, [path | group])
    st = %{st | index: idx}

    # 그룹 길이가 2 이상이 되면, 해당 그룹 전체에 대해 full hash 수행
    if length(idx[key]) >= 2 do
      Enum.each(idx[key], fn p ->
        case full_hash(p, st.algo) do
          {:ok, full} ->
            send(st.downstream, {:full, size, head, full, p})

          {:error, _} ->
            :ok
        end
      end)

      st = %{st | index: Map.delete(st.index, key)}
      {:noreply, st}
    else
      {:noreply, st}
    end
  end

  @impl true
  def handle_info({:eof, :head}, %{downstream: downstream}=st) do
    send(downstream, {:eof, :full})
    {:noreply, st}
  end

  defp full_hash(path, algo) do
    case File.open(path, [:read]) do
      {:ok, io} ->
        try do
          ctx = :crypto.hash_init(algo)
          stream_hash(io, ctx, algo)
        after
          File.close(io)
        end

      {:error, reason} ->
        {:error, reason}
    end
  end

  defp stream_hash(io, ctx, algo) do
    case IO.binread(io, @chunk) do
      :eof ->
        {:ok, :crypto.hash_final(ctx)}

      data when is_binary(data) ->
        ctx2 = :crypto.hash_update(ctx, data)
        stream_hash(io, ctx2, algo)

      {:error, reason} ->
        {:error, reason}
    end
  end
end
```

포인트:

- `@chunk` 크기는 디스크 특성에 따라 튜닝 가능.
- `stream_hash`는 **tail recursion**으로 전체 파일을 읽어 해시를 계산한다.
- 헤더 해시 그룹이 작을수록, 전체 해시를 계산하는 파일 수도 줄어든다.

---

### Sink — Index에 결과 적재 및 로그 출력

Sink는 FullHasher에서 전달받은 결과를 `Duper.Index`로 넘기고,
필요하다면 로그를 남기거나 최종 리포트를 만든다.

```elixir
defmodule Duper.Sink do
  use GenServer

  def start_link(opts),
    do: GenServer.start_link(__MODULE__, opts, name: Keyword.get(opts, :name))

  @impl true
  def init(_), do: {:ok, %{dupes: 0, groups_first_seen: 0}}

  @impl true
  def handle_info({:full, size, head, full, path}, st) do
    group_size = Duper.Index.put(size, head, full, path)

    if group_size == 2 do
      require Logger
      Logger.info("duplicate group first seen for hash=#{Base.encode16(full)}")
      {:noreply, %{st | dupes: st.dupes + 1, groups_first_seen: st.groups_first_seen + 1}}
    else
      {:noreply, %{st | dupes: st.dupes + 1}}
    end
  end

  @impl true
  def handle_info({:eof, :full}, st) do
    require Logger
    Logger.info("pipeline completed; dupes=#{st.dupes}, groups=#{st.groups_first_seen}")
    {:noreply, st}
  end
end
```

- `group_size == 2`일 때만 “새로운 중복 그룹 발견”으로 로그를 남김.
- 전체 파일 개수·중복 그룹 수 등은 Telemetry 이벤트로도 발행할 수 있다.

---

### 파이프라인 조립 — Pipeline Supervisor

실전에서 사용하기 조금 더 명확한 파이프라인 예시는 다음과 같이 구성할 수 있다.

```elixir
defmodule Duper.Pipeline.Supervisor do
  use Supervisor

  def start_link(opts),
    do: Supervisor.start_link(__MODULE__, opts, name: __MODULE__)

  @impl true
  def init(_opts) do
    children = [
      {Duper.Sink, name: Duper.Sink},
      {Duper.FullHasher, name: Duper.FullHasher, downstream: Duper.Sink},
      {DynamicSupervisor, name: Duper.HashSup, strategy: :one_for_one},
      {Duper.HeadPool, name: Duper.HeadPool},
      {Duper.StatFilter, name: Duper.StatFilter, downstream: Duper.HeadPool},
      {Duper.Scanner.DFS, name: Duper.Scanner, roots: Duper.Config.roots(), downstream: Duper.StatFilter}
    ]

    Supervisor.init(children, strategy: :rest_for_one)
  end
end
```

여기서 `Duper.HeadPool`은 HeadHasher 워커들에 대한 **라우터**다.

```elixir
defmodule Duper.HeadPool do
  use GenServer

  @pool Duper.HashSup

  def start_link(opts),
    do: GenServer.start_link(__MODULE__, opts, name: __MODULE__)

  @impl true
  def init(_opts) do
    workers =
      for _ <- 1..System.schedulers_online() do
        {:ok, pid} =
          DynamicSupervisor.start_child(@pool, {Duper.HeadHasher, [downstream: Duper.FullHasher]})

        pid
      end

    {:ok, %{workers: workers, index: 0}}
  end

  def dispatch(msg),
    do: GenServer.cast(__MODULE__, {:dispatch, msg})

  @impl true
  def handle_cast({:dispatch, msg}, %{workers: workers, index: i} = st) do
    pid = Enum.at(workers, rem(i, length(workers)))
    send(pid, msg)
    {:noreply, %{st | index: i + 1}}
  end

  @impl true
  def handle_info({:candidate, size, path}, st) do
    handle_cast({:dispatch, {:candidate, size, path}}, st)
  end

  @impl true
  def handle_info({:eof, :stat}, _st) do
    send(Duper.FullHasher, {:eof, :head})
    {:noreply, %{}}
  end
end
```

- `StatFilter`는 `HeadPool`로 메시지를 보내고,
  `HeadPool`은 내부 워커 목록을 기준으로 라운드 로빈으로 배분한다.

---

### Config — 루트/헤더 크기/알고리즘/워커 수

```elixir
defmodule Duper.Config do
  @moduledoc false

  @default_roots [System.user_home!()]
  @default_header_bytes 256 * 1024

  def roots,
    do: Application.get_env(:duper, :roots, @default_roots)

  def header_bytes,
    do: Application.get_env(:duper, :header_bytes, @default_header_bytes)

  def algo,
    do: Application.get_env(:duper, :algo, :sha256)

  def head_workers,
    do: Application.get_env(:duper, :head_workers, System.schedulers_online())
end
```

`config/config.exs`:

```elixir
import Config

config :duper,
  roots: ["/data", "/var/shared"],
  header_bytes: 512 * 1024,
  algo: :sha256,
  head_workers: 8
```

환경별로 다른 설정을 적용하고 싶다면 `config/dev.exs`, `config/prod.exs` 에서 덮어쓰면 된다.

---

### CLI — Mix Task로 한 번에 돌려보기

```elixir
defmodule Mix.Tasks.Duper.Run do
  use Mix.Task

  @shortdoc "Run Duper duplicate finder"

  @moduledoc """
  중복 파일 탐색기를 실행한다.

      mix duper.run [PATH...]

  인자를 주지 않으면 `Duper.Config.roots/0` 에서 정의된 기본 루트를 사용한다.
  """

  def run(args) do
    Mix.Task.run("app.start")

    roots =
      case args do
        [] -> Duper.Config.roots()
        _ -> args
      end

    IO.puts("Scanning roots: #{Enum.join(roots, ", ")}")

    # Scanner가 roots를 사용하도록 동적으로 설정하고 싶다면,
    # Application 환경 또는 Registry 기반 설정 주입을 사용할 수 있다.
    # 이 예제에서는 간단히 무한 대기로 두고, 로그/Telemetry를 통해 진행 상황을 본다.
    Process.sleep(:infinity)
  end
end
```

옵션 파서(예: `--json`, `--min-size`)를 추가하면 다음과 같이 확장할 수 있다.

```elixir
{opts, paths, _} =
  OptionParser.parse(args,
    switches: [json: :boolean, min_size: :integer],
    aliases: [j: :json]
  )
```

---

## 잘 실행될까? — 정확성, 성능, 장애 복원, 관찰

### 정확성 체크리스트 — “정말 같은 파일만 묶였는가?”

1. **서로 다른 파일은 서로 다른 해시**
   - SHA-256 기준으로 충돌 가능성이 극히 낮으므로, 실무에서는 “충분히 동일”로 본다.
   - 내용이 다른데도 중복으로 나오는 경우가 없도록, 전체 해시를 반드시 계산해야 한다.

2. **같은 파일 다른 경로는 같은 그룹**
   - 백업 디렉터리, 사진 편집본, 복사된 자료 등을 정확히 묶어야 한다.

3. **하드링크/심볼릭 링크**
   - 하드링크는 OS 차원에서 **같은 inode** 를 가리키므로,
     - 둘 다 같은 해시를 가져야 한다.
   - 심볼릭 링크는 정책에 따라:
     - 링크 자체를 파일로 취급(작은 텍스트 파일로)
     - 링크 대상 파일의 내용을 따라가서 해시
     - 링크 무시
   - 루프(링크 사이클)를 방지하려면, 방문 집합을 유지하거나 적절한 제한이 필요하다.

4. **0바이트 파일**
   - 0바이트 파일은 모두 같은 해시가 되지만,
     - 정책상 제외할지, 포함할지 옵션으로 둔다.

단위 테스트 예시:

```elixir
defmodule Duper.IndexTest do
  use ExUnit.Case, async: true

  test "grouping by (size, head, full)" do
    {:ok, _} = start_supervised(Duper.Index)

    size = 3
    head = :crypto.hash(:sha256, "abc")
    full = head

    assert 1 == Duper.Index.put(size, head, full, "/tmp/a")
    assert 2 == Duper.Index.put(size, head, full, "/tmp/b")

    assert [{^size, ^head, ^full, paths}] = Duper.Index.groups()
    assert Enum.sort(paths) == ["/tmp/a", "/tmp/b"]
  end
end
```

파이프라인 통합 테스트:

- 작은 테스트 디렉터리를 만들고:
  - `a.txt`, `b.txt`, `c.txt` 세 개 파일을 준비
  - `b.txt` 를 `cp a.txt d.txt` 로 복사
  - 예상: `a.txt`, `d.txt` 는 하나의 중복 그룹, `b.txt`, `c.txt` 는 단독 그룹

---

### 성능과 역압 — 숫자로 보는 튜닝 포인트

간단한 수치를 가정해보자.

- 디스크 실효 읽기 속도: 200 MB/s
- 평균 파일 크기: 10 MB
- 전체 파일 수: 1,000,000개
- 헤더 해시 크기: 512 KiB

순진한 전체 해시 방식이라면:

- 총 데이터량: \(10 \text{ MB} \times 1,000,000 = 10 \text{ TB}\)
- 200 MB/s의 단일 디스크에서 이론적 최소 시간:
  $$
  T_{\text{min}} = \frac{10 \text{ TB}}{200 \text{ MB/s}}
  = \frac{10 \times 1024 \text{ GB}}{0.195 \text{ GB/s}} \approx 52,500 \text{s} \approx 14.5 \text{시간}
  $$
- 실제로는 seek, 캐시, 기타 오버헤드 때문에 더 오래 걸린다.

헤더 해시(512 KiB) + 후보 그룹에 대해서만 전체 해시를 수행하면:

- 모든 파일이 서로 다르다면:
  - 총 읽기량: \(0.5 \text{ MiB} \times 1,000,000 ≈ 500 \text{ GiB}\)
- 200 MB/s 환경에서 이론적 최소:
  $$
  T_{\text{min}} \approx \frac{500 \text{ GiB}}{0.195 \text{ GB/s}} \approx 2,560 \text{s} \approx 42.7 \text{분}
  $$
- 여기에 후보 그룹(동일 헤더 해시 그룹)에 대해서만 전체 해시를 수행하게 되므로,
  중복 정도에 따라 추가 비용이 들어간다.

즉:

- 헤더 해시는 **대부분의 파일이 서로 다르다는 가정에서 큰 이득**을 준다.
- 헤더 크기 \(H\) 를 키워갈수록 후보 그룹이 줄어들지만,
  전체 읽기량은 \(N \cdot H\) 비율로 늘어난다.
- 파일들의 **실제 크기 분포/중복 패턴**에 맞춰 \(H\) 를 튜닝해야 한다.

역압 관점에서:

- HeadHasher 워커 수를 \(k\) 라 하고, 워커당 처리율(헤더 해시 계산)을 \(\mu_c\) 라 하면,
  CPU 측 처리율은 \(k \cdot \mu_c\) 정도가 된다.
- 하지만 디스크 처리율 \(\mu_d\) 가 병목이면,
  $$
  \mu_{\text{effective}} \approx \min(k \cdot \mu_c, \mu_d)
  $$
- 워커를 너무 많이 늘려도 **디스크 큐만 길어지고 실질 속도는 증가하지 않는다**.

따라서:

1. 초기에는 `head_workers = System.schedulers_online()` 정도로 설정.
2. Telemetry로 “해싱에 걸린 시간”, “I/O 대기 시간” 등을 측정.
3. 병목이 CPU인지, 디스크인지 보고 워커 수를 조절한다.

---

### 장애 복원 — “스스로 회복하는 중복 탐색기”

슈퍼비전 전략이 제대로 설계되었다면:

- 해시 계산 중 특정 파일에서 예외가 나더라도,
  - 해당 워커 프로세스만 크래시 → 슈퍼바이저가 다시 띄운다.
- FullHasher가 반복해서 죽는다면,
  - `max_restarts`, `max_seconds`에 따라 슈퍼바이저가 자신의 한계를 인지하고 종료 → 상위에서 다시 재시작.

예제 테스트:

```elixir
defmodule Duper.FullHasherRestartTest do
  use ExUnit.Case

  test "full hasher crash restarts only itself" do
    {:ok, _sup} = start_supervised(Duper.Pipeline.Supervisor)

    pid = Process.whereis(Duper.FullHasher)
    assert is_pid(pid)

    Process.exit(pid, :kill)
    Process.sleep(50)

    new_pid = Process.whereis(Duper.FullHasher)
    assert is_pid(new_pid)
    assert pid != new_pid
  end
end
```

Index가 죽었을 때:

- `Duper.Index`가 `:one_for_one` 전략의 자식이라면,
  - Index만 다시 시작되고, 다른 스테이지는 유지된다.
- 다만 ETS 테이블은 다시 생성되므로,
  - “중간까지 구축한 중복 인덱스”는 잃게 된다.
  - 중간 결과를 유지하고 싶다면, 주기적인 스냅샷(디스크 기록) 전략을 추가해야 한다.

---

### 관찰 가능성 — Telemetry로 해시와 재시작을 보는 법

관찰 포인트:

- 각 스테이지별 처리 시간
- 워커의 메일박스 길이
- 재시작 횟수/사유
- 전체 파일 수, 후보 수, 최종 중복 그룹 수

예: FullHasher에 Telemetry 스팬 넣기

```elixir
defp full_hash(path, algo) do
  :telemetry.span([:duper, :hash, :full], %{path: path}, fn ->
    result =
      case File.open(path, [:read]) do
        {:ok, io} ->
          try do
            ctx = :crypto.hash_init(algo)
            stream_hash(io, ctx, algo)
          after
            File.close(io)
          end

        {:error, reason} ->
          {:error, reason}
      end

    {result, %{ok?: match?({:ok, _}, result)}}
  end)
end
```

핸들러:

```elixir
:telemetry.attach(
  "duper-full-hash-logger",
  [:duper, :hash, :full, :stop],
  fn _event, measurements, metadata, _config ->
    require Logger
    Logger.info(
      "full hash duration=#{measurements.duration} ns ok=#{metadata.ok?}"
    )
  end,
  nil
)
```

재시작 Telemetry(슈퍼바이저가 지원하도록 이벤트를 정의했다고 가정):

```elixir
:telemetry.attach(
  "duper-sup-restarts",
  [:duper, :supervisor, :restart],
  fn _e, meas, meta, _ ->
    require Logger
    Logger.warning(
      "restart child=#{inspect(meta.child)} reason=#{inspect(meta.reason)} count=#{meas.count}"
    )
  end,
  nil
)
```

---

### 실전 튜닝 예제 — “처음 돌렸더니 너무 느리다”면?

가상의 시나리오:

- 디렉터리: 200만 개 파일, 총 크기 20 TB
- 초기 설정:
  - `header_bytes: 256 * 1024`
  - `head_workers: schedulers_online()`
  - `algo: :sha256`

1차 실행 결과:

- 전체 소요 시간: 18시간
- Telemetry 상:
  - 헤더 해시 평균 시간: 매우 짧음
  - FullHasher 처리 시간: 디스크 대기 시간이 많음
  - HeadHasher 메일박스 길이: 거의 0
  - FullHasher 메일박스 길이: 길게 유지

해석:

- FullHasher가 병목이다.
- 헤더 해시 단계에서 충분히 후보를 줄이지 못하거나,
- FullHasher 워커 수가 부족하다(현재 단일 서버).

튜닝 방향:

1. FullHasher를 HeadHasher처럼 **워커 풀**로 확장:
   - DynamicSupervisor + 라운드로빈 라우터 적용.
2. 헤더 크기를 더 키워 후보를 줄인다:
   - `header_bytes: 512 * 1024` ~ `1 * 1024 * 1024` 시도.
3. FullHasher 워커 수를 코어 수에 맞춰 늘리고,
   디스크가 복수라면 I/O 병렬성이 충분히 활용되는지 확인.

반대로 Telemetry 결과가 “HeadHasher는 항상 바쁘고, FullHasher는 여유”라면,
HeadHasher 쪽 워커를 늘리거나, 헤더 크기를 줄여서 밸런스를 맞춰야 한다.

---

## 엘릭서 애플리케이션 설계하기 — Duper에서 일반화하기

Duper 설계 과정은 **다른 OTP 애플리케이션에도 거의 그대로 적용**할 수 있다.

### 단계적 설계 체크리스트 (복습 + 확장)

1. **문제 분해**
   - 입력(Scanner) → 전처리(Stat) → 핵심 처리(Hasher) → 결과 집계(Index/Sink)
   - 각 단계를 **독립된 컴포넌트**로 나눈다.

2. **프로세스 경계**
   - 어떤 부분을 별도 프로세스로 분리할 것인가?
     - I/O 무거운 부분
     - CPU 집약적 처리
     - 상태를 캡슐화해야 하는 부분

3. **슈퍼비전 전략**
   - 독립 모듈: `:one_for_one`
   - 순차 의존(파이프라인): `:rest_for_one`
   - 강한 공유 설정/자원: `:one_for_all`

4. **이름과 라우팅**
   - `Registry`를 사용해 “키 → 프로세스” 매핑.
   - 라우터(예: HeadPool)를 도입해 워커 풀에 작업을 배분.

5. **역압과 배치**
   - `call`/`cast` 경계를 잘 나누고,
   - 메일박스 길이를 측정해 **한도/드롭/지연** 정책을 적용.
   - 배치 쓰기를 통해 I/O 왕복 횟수를 줄인다.

6. **관찰 가능성**
   - Telemetry로 스팬/이벤트를 발행하고,
   - 로그/메트릭/대시보드에서 지연, 에러율, 큐 길이, 처리량을 본다.

7. **구성 가능성**
   - 중요 파라미터(워커 수, 헤더 크기, 알고리즘 등)를 config로 빼서,
   - 환경별로 쉽게 튜닝 가능하게 만든다.

8. **테스트 용이성**
   - 각 컴포넌트를 독립적으로 테스트하고,
   - 필요한 곳에 Behaviour + Mox를 사용해 외부 의존성을 목(mock) 처리.
   - 파이프라인 전체에 대한 통합 테스트도 작성.

---

### GenStage/Broadway 대안 — 역압이 내장된 파이프라인

파일 수가 매우 크고, **역압(backpressure)** 을 보다 강하게 보장하고 싶다면
`GenStage` 또는 `Broadway`를 사용할 수 있다.

아이디어:

- `Scanner` → `StatFilter` → `HeadHasher` → `FullHasher` → `Sink` 를 각각 Stage로 만들고,
- 각 Stage 간에 demand 기반으로 메시지를 주고받는다.

예: 간단한 GenStage 프로듀서

```elixir
defmodule Duper.ScannerStage do
  use GenStage

  def start_link(roots),
    do: GenStage.start_link(__MODULE__, roots, name: __MODULE__)

  @impl true
  def init(roots) do
    paths = Duper.Scanner.DFS.collect_paths(roots)
    {:producer, paths}
  end

  @impl true
  def handle_demand(demand, state) when demand > 0 do
    {events, rest} = Enum.split(state, demand)
    {:noreply, events, rest}
  end
end
```

- 실전에서는 DFS를 “한 번에 다 모으지 않고” 스트리밍 방식으로 구현해야 한다.
- GenStage는 demand를 기반으로 다음 스테이지에서 요청할 때만 이벤트를 내보낸다.

Broadway를 사용하면:

- 파일 경로를 큐(예: SQS, RabbitMQ)에 올려놓고,
- Broadway 파이프라인이 워커 풀을 관리하며,
- 실패 재시도/배치/역압 등을 자동으로 지원하게 만들 수 있다.

---

### 분산 스케일 — 여러 노드로 Duper 확장하기

여러 노드에서 동시에 스캔하고 싶다면:

1. **루트 분할**
   - 루트 디렉터리 목록을 노드별로 나눠서 할당:
     - 노드 A: `/data/a`, `/data/b`
     - 노드 B: `/data/c`, `/data/d`
2. **로컬 인덱스**
   - 각 노드에서 **자기 루트에 대한 Index**를 구축
3. **최종 머지 단계**
   - 작업이 끝난 뒤 각 노드의 인덱스를 가져와:
     - `(size, full_hash)` 기준으로 다시 머지
     - 또는 중앙 저장소(예: PostgreSQL)에 통합 기록

또 다른 방법:

- 파일 경로를 해싱해서 `(hash(path) mod num_nodes)` 방식으로 **키 기반 라우팅**을 적용.
- 이렇게 하면, 어떤 디렉터리에서 나왔든, 특정 키 범위에 속하는 파일들은 항상 같은 노드에서 처리된다.

---

### 안전성과 정합성 — 파일 변경, 권한, 특수 파일

1. **파일 변경 레이스**
   - Scanner에서 파일을 발견한 뒤, 해시를 계산하는 사이에 파일이 변경될 수 있다.
   - 해결책:
     - `File.stat/1`에서 `mtime`, `size`, `inode` 등을 저장해두고,
     - 최종 결과 출력 시 다시 확인해 변경된 파일을 제외하거나 “변경됨”으로 표시.

2. **권한 문제**
   - 특정 디렉터리/파일에 접근 권한이 없을 수 있다.
   - `File.stat/1`, `File.open/2` 에서 `{:error, :eacces}` 가 뜨면:
     - 해당 파일은 스킵하고,
     - 로그로만 남긴다.

3. **특수 파일**
   - FIFO, 소켓, 블록 디바이스 등은 스킵해야 한다.
   - `File.stat/1`의 `type` 필드로 식별 후 제외.

---

### 복잡도와 비용 추정 — 어느 정도 리소스를 먹는지

파일 수를 \(N\), 평균 파일 크기를 \(\bar{S}\), 헤더 크기를 \(H\) 라고 하자.

1차 헤더 해시 비용:

- 총 읽기량: \(N \cdot H\)
- 시간 근사:
  $$
  T_{\text{head}} \approx \frac{N \cdot H}{B}
  $$
  여기서 \(B\)는 디스크 실효 대역폭.

2차 전체 해시 비용:

- 후보 비율을 \(p\) (헤더 해시까지 동일한 그룹에 속한 비율)라고 하면,
  - 후보 파일 수: \(pN\)
  - 총 읽기량: \(pN \cdot \bar{S}\)
- 시간 근사:
  $$
  T_{\text{full}} \approx \frac{pN \cdot \bar{S}}{B}
  $$

총 시간:

$$
T_{\text{total}} \approx T_{\text{head}} + T_{\text{full}}
= \frac{N \cdot H}{B} + \frac{pN \cdot \bar{S}}{B}
= \frac{N}{B}(H + p\bar{S})
$$

튜닝 목표:

- \(H\)와 \(p\) 사이의 균형.
  - \(H\)를 늘리면 \(p\)가 줄어들지만, \(H\) 자체가 커진다.
- 실제 데이터셋에서 몇 번 측정하면서 최적점을 찾는 것이 중요하다.

---

### 유지보수와 문서화 — 팀이 이해할 수 있는 형태로

1. **@moduledoc / @doc**
   - 각 모듈의 역할, 주요 개념(예: “헤더 해시 vs 전체 해시”)을 명시.
   - Doctest를 통해 예시 코드가 실제로 실행되는지 검증.

2. **슈퍼비전 트리 문서화**
   - README나 내부 문서에 Supervisor 트리를 그림/텍스트로 남기고,
   - 각 슈퍼바이저의 전략과 자식 목록을 설명.

3. **Telemetry 이벤트 명세**
   - 이벤트 이름(예: `[:duper, :hash, :full, :stop]`)
   - 측정값(예: `duration`, `size`)
   - 메타데이터(예: `path`, `ok?`)
   - 이를 문서화해, 모니터링/대시보드 담당자가 참고할 수 있게 한다.

---

## 개선된 파이프라인 요약

앞에서 만든 개선된 파이프라인을 텍스트로 다시 정리해보면:

```text
Duper.Supervisor
  ├─ Registry (Duper.Reg)
  ├─ Duper.Index
  ├─ DynamicSupervisor (Duper.ScanSup)
  ├─ DynamicSupervisor (Duper.HashSup)
  └─ Duper.Pipeline.Supervisor
       ├─ Duper.Sink
       ├─ Duper.FullHasher
       ├─ DynamicSupervisor (Duper.HashSup, HeadHasher용)
       ├─ Duper.HeadPool (HeadHasher 라우터)
       ├─ Duper.StatFilter
       └─ Duper.Scanner.DFS
```

데이터 흐름:

```text
Scanner.DFS → StatFilter → HeadPool → HeadHasher[*] → FullHasher → Sink → Index
```

---

## 샘플 사용법

### B.1 기본 실행

```bash
mix deps.get
mix compile
mix duper.run
```

- `config/config.exs` 에 설정된 `roots` 기준으로 스캔.

### B.2 특정 디렉터리 지정

```bash
mix duper.run ~/Downloads ~/Pictures
```

### B.3 IEx에서 직접 조회

```bash
iex -S mix

# 파이프라인 시작

{:ok, _} = Duper.Application.start(:normal, [])

# 작업이 어느 정도 진행된 후, 현재 중복 그룹 보기

Duper.Index.groups()
|> Enum.filter(fn {_size, _head, _full, paths} -> length(paths) >= 2 end)
|> Enum.take(10)
|> IO.inspect(label: "sample duplicate groups")
```

출력 예시:

```elixir
sample duplicate groups: [
  {3145728, <<...head...>>, <<...full...>>,
    [
      "/data/photos/2020/IMG_0001.JPG",
      "/data/backup/photos/IMG_0001_copy.JPG"
    ]},
  ...
]
```

---

## 연습 문제

1. **확장자 필터 추가**
   - StatFilter에 “허용 확장자 목록” 옵션을 추가하고,
   - 테스트 디렉터리에서 특정 확장자만 중복 검사하도록 구현하라.

2. **TTL 로그 스냅샷**
   - Index를 일정 주기마다 디스크로 덤프하는 기능을 추가하라.
   - 재시작 후 이 스냅샷을 읽어 **이전 인덱스를 복구**하는 기능을 만들어보라.

3. **GenStage 버전 Duper**
   - Scanner/StatFilter/Hasher/Sink를 GenStage 파이프라인으로 재구성하고,
   - demand 기반으로 역압을 테스트해보라.

4. **분산 Duper**
   - 로컬 노드에서만 동작하던 Duper를,
   - 두 노드에서 각각 다른 루트를 스캔하도록 확장하고,
   - 작업 후 두 인덱스를 합쳐 최종 중복 그룹을 만들라.

5. **성능 리포트**
   - Telemetry 데이터를 기반으로 “헤더 해시/전체 해시에 각각 얼마의 시간이 걸렸는지”,
   - “중복 파일 비율이 어느 정도인지”를 정리하는 리포트를 생성하라.

---

## 마무리

- Duper는 단순히 “중복 파일 찾기” 도구를 넘어,
  **OTP/Elixir 애플리케이션 설계 패턴의 집약 예제**로 사용할 수 있다.
- 문제를 **파이프라인으로 분해**하고,
  각 단계를 **프로세스와 슈퍼비전 트리로 구조화**하며,
  **역압·관찰·테스트·구성 가능성**을 기본값으로 넣는다면,
  규모가 커져도 **스스로 회복하고, 튜닝 가능한 시스템**을 만들 수 있다.
- 이 패턴은 파일 시스템 뿐 아니라,
  **로그 수집, ETL, 웹 크롤링, 멀티미디어 인덱싱, 머신러닝 전처리 파이프라인** 등에도 그대로 적용할 수 있다.
