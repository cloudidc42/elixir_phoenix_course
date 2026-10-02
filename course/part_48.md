# Part 48: Distributed Elixir (Elixir แบบกระจาย)

## เป้าหมายการเรียนรู้

- เชื่อมต่อ Elixir nodes ด้วย `Node.connect/1`
- ใช้ `:rpc` module สำหรับ remote procedure calls
- ลงทะเบียน process แบบ global ด้วย `:global`
- ใช้ Horde สำหรับ distributed supervisors และ registries
- ทำความเข้าใจ consistent hashing
- ตั้งค่า clustering บน Fly.io ด้วย DNS
- ใช้ Nebulex สำหรับ distributed caching
- จัดการ network partitions

---

## 1. Elixir Node คืออะไร?

**Node** คือ Erlang VM instance หนึ่งตัว ซึ่งสามารถเชื่อมต่อกับ VM instances อื่นผ่านเครือข่ายได้ เมื่อเชื่อมต่อแล้ว nodes เหล่านั้นสามารถส่ง messages ระหว่างกัน เรียก functions และแชร์ข้อมูลได้

```bash
# เปิด terminal 1: สร้าง node ชื่อ alice
iex --sname alice --cookie secret_cookie

# เปิด terminal 2: สร้าง node ชื่อ bob
iex --sname bob --cookie secret_cookie
```

> **สำคัญ:** `--cookie` ต้องเหมือนกันทุก node เพื่อความปลอดภัย

---

## 2. Node.connect และการจัดการ Nodes

```elixir
# ดูชื่อ node ปัจจุบัน
Node.self()
# => :alice@hostname

# เชื่อมต่อกับ node อื่น
Node.connect(:"bob@hostname")
# => true

# ดู nodes ทั้งหมดที่เชื่อมต่ออยู่
Node.list()
# => [:bob@hostname]

# ส่ง message ไปยัง process บน node อื่น
# {process_name, node} เป็น tuple ที่ใช้ระบุ remote process
send({:some_process, :"bob@hostname"}, {:hello, "จาก alice"})

# spawn process บน node อื่น
Node.spawn(:"bob@hostname", fn ->
  IO.puts("ฉันทำงานบน #{Node.self()}")
end)

# ตรวจสอบว่า node อยู่ใน cluster
Node.ping(:"bob@hostname")
# => :pong (หรือ :pang ถ้าไม่ตอบ)
```

### Monitor Node ที่ down

```elixir
defmodule MyApp.NodeMonitor do
  use GenServer

  def start_link(_opts) do
    GenServer.start_link(__MODULE__, [], name: __MODULE__)
  end

  def init(_args) do
    # subscribe รับแจ้งเตือนเมื่อ node เข้า/ออก cluster
    :net_kernel.monitor_nodes(true)
    {:ok, %{nodes: Node.list()}}
  end

  def handle_info({:nodeup, node}, state) do
    IO.puts("Node เข้า cluster: #{node}")
    {:noreply, %{state | nodes: [node | state.nodes]}}
  end

  def handle_info({:nodedown, node}, state) do
    IO.puts("Node ออกจาก cluster: #{node}")
    MyApp.failover(node)  # ทำ failover logic
    {:noreply, %{state | nodes: List.delete(state.nodes, node)}}
  end
end
```

---

## 3. :rpc Module — Remote Procedure Calls

`:rpc` อนุญาตให้เรียก function บน remote node

```elixir
# เรียก function บน remote node (blocking)
:rpc.call(:"bob@hostname", String, :upcase, ["hello"])
# => "HELLO"

# เรียกแบบ async (non-blocking)
key = :rpc.async_call(:"bob@hostname", MyApp.Calculator, :heavy_compute, [1_000_000])

# รอรับผลลัพธ์ในภายหลัง
result = :rpc.yield(key)

# broadcast ไปทุก node ใน cluster
:rpc.multicall(Node.list(), MyApp.Cache, :clear, [])

# เรียกบน node กลุ่มหนึ่ง
nodes = [:"node1@host", :"node2@host"]
results = :rpc.multicall(nodes, MyApp.Stats, :get, [])
```

### ตัวอย่างจริง: Distributed Job Queue

```elixir
defmodule MyApp.DistributedQueue do
  def submit_job(job) do
    # เลือก node ที่มี load น้อยที่สุด
    target_node = find_least_loaded_node()

    case :rpc.call(target_node, MyApp.Worker, :process, [job], 5_000) do
      {:badrpc, reason} ->
        {:error, {:rpc_failed, reason}}
      result ->
        {:ok, result}
    end
  end

  defp find_least_loaded_node do
    [Node.self() | Node.list()]
    |> Enum.min_by(fn node ->
      :rpc.call(node, :erlang, :statistics, [:run_queue])
    end)
  end
end
```

---

## 4. Global Process Registration

ใช้ `:global` module ลงทะเบียน process ให้เข้าถึงได้จากทุก node

```elixir
defmodule MyApp.GlobalCache do
  use GenServer

  def start_link(_opts) do
    # ลงทะเบียนชื่อ global — มีได้เพียงหนึ่งตัวใน cluster
    GenServer.start_link(__MODULE__, %{}, name: {:global, __MODULE__})
  end

  def get(key) do
    GenServer.call({:global, __MODULE__}, {:get, key})
  end

  def put(key, value) do
    GenServer.cast({:global, __MODULE__}, {:put, key, value})
  end

  def init(state), do: {:ok, state}

  def handle_call({:get, key}, _from, state) do
    {:reply, Map.get(state, key), state}
  end

  def handle_cast({:put, key, value}, state) do
    {:noreply, Map.put(state, key, value)}
  end
end

# เรียกใช้จากทุก node
MyApp.GlobalCache.put("session:123", %{user_id: 42})
MyApp.GlobalCache.get("session:123")
```

> **ข้อเสีย:** `:global` มีปัญหาเมื่อเกิด network partition อาจมีสองตัวทำงานพร้อมกัน (split-brain)

---

## 5. Horde — Distributed Supervisors และ Registries

**Horde** แก้ปัญหา split-brain โดยใช้ CRDT (Conflict-free Replicated Data Type) ทำให้ state sync กันอัตโนมัติ

```elixir
# mix.exs
{:horde, "~> 0.9"}
```

### Horde Registry

```elixir
defmodule MyApp.HordeRegistry do
  use Horde.Registry

  def start_link(_opts) do
    Horde.Registry.start_link(__MODULE__, [keys: :unique], name: __MODULE__)
  end

  def init(init_arg) do
    [members: members()]
    |> Keyword.merge(init_arg)
    |> Horde.Registry.init()
  end

  defp members do
    [Node.self() | Node.list()]
    |> Enum.map(&{__MODULE__, &1})
  end
end
```

### Horde DynamicSupervisor

```elixir
defmodule MyApp.HordeSupervisor do
  use Horde.DynamicSupervisor

  def start_link(_opts) do
    Horde.DynamicSupervisor.start_link(__MODULE__, [strategy: :one_for_one],
      name: __MODULE__)
  end

  def init(init_arg) do
    [members: members(), distribution_strategy: Horde.UniformRandomDistribution]
    |> Keyword.merge(init_arg)
    |> Horde.DynamicSupervisor.init()
  end

  defp members do
    [Node.self() | Node.list()]
    |> Enum.map(&{__MODULE__, &1})
  end
end

# Worker ที่ใช้ Horde
defmodule MyApp.GameSession do
  use GenServer

  def start_link(%{session_id: session_id} = opts) do
    GenServer.start_link(__MODULE__, opts,
      name: {:via, Horde.Registry, {MyApp.HordeRegistry, session_id}}
    )
  end

  # เริ่ม game session บน node ที่ว่าง
  def create_session(session_id) do
    child_spec = %{
      id: session_id,
      start: {__MODULE__, :start_link, [%{session_id: session_id}]},
      restart: :transient
    }

    Horde.DynamicSupervisor.start_child(MyApp.HordeSupervisor, child_spec)
  end

  # ค้นหา session จากทุก node
  def find_session(session_id) do
    case Horde.Registry.lookup(MyApp.HordeRegistry, session_id) do
      [{pid, _}] -> {:ok, pid}
      [] -> {:error, :not_found}
    end
  end
end
```

---

## 6. Consistent Hashing

Consistent Hashing ช่วยกระจาย load อย่างสม่ำเสมอและลด re-routing เมื่อ node เข้า/ออก cluster

```elixir
defmodule MyApp.ConsistentHash do
  @virtual_nodes 150  # จำนวน virtual nodes ต่อ physical node

  def new(nodes) do
    nodes
    |> Enum.flat_map(fn node ->
      Enum.map(1..@virtual_nodes, fn i ->
        {hash("#{node}:#{i}"), node}
      end)
    end)
    |> Enum.sort_by(&elem(&1, 0))
  end

  def get_node(ring, key) do
    h = hash(key)

    # หา node แรกที่มี hash >= key hash (clockwise)
    case Enum.find(ring, fn {node_hash, _} -> node_hash >= h end) do
      nil ->
        # wrap around — ใช้ node แรกใน ring
        ring |> List.first() |> elem(1)

      {_, node} ->
        node
    end
  end

  defp hash(key) do
    :erlang.phash2(key, 4_294_967_296)
  end
end

# ใช้งาน
nodes = [:node1, :node2, :node3]
ring = MyApp.ConsistentHash.new(nodes)

MyApp.ConsistentHash.get_node(ring, "user:123")  # => :node2
MyApp.ConsistentHash.get_node(ring, "user:456")  # => :node1
MyApp.ConsistentHash.get_node(ring, "user:789")  # => :node3
```

---

## 7. Fly.io Clustering ด้วย DNS

Fly.io ใช้ DNS-based clustering ผ่าน library `dns_cluster`

```elixir
# mix.exs
{:dns_cluster, "~> 0.1"}
```

### ตั้งค่าใน application.ex

```elixir
defmodule MyApp.Application do
  use Application

  def start(_type, _args) do
    children = [
      # DNS cluster discovery
      {DNSCluster, query: Application.get_env(:my_app, :dns_cluster_query) || :ignore},
      MyApp.Repo,
      MyAppWeb.Endpoint,
    ]

    Supervisor.start_link(children, strategy: :rest_for_one, name: MyApp.Supervisor)
  end
end
```

### config/runtime.exs

```elixir
# กำหนด fly.io app name สำหรับ DNS discovery
if app_name = System.get_env("FLY_APP_NAME") do
  config :my_app, :dns_cluster_query, "#{app_name}.internal"
end

# กำหนด node name ใช้ Fly.io private IP
if System.get_env("FLY_APP_NAME") do
  app_name = System.get_env("FLY_APP_NAME")
  fly_ip = System.get_env("FLY_PRIVATE_IP") || raise "FLY_PRIVATE_IP not set"

  config :my_app, MyAppWeb.Endpoint,
    url: [host: "#{app_name}.fly.dev"]

  # กำหนด node name แบบ long names สำหรับ Fly.io
  release_node = "#{app_name}@#{fly_ip}"
  System.put_env("RELEASE_NODE", release_node)
end
```

### fly.toml

```toml
[env]
  PHX_HOST = "myapp.fly.dev"
  PORT = "8080"

[deploy]
  release_command = "/app/bin/my_app eval MyApp.Release.migrate"

[[services]]
  internal_port = 8080
  protocol = "tcp"

  [[services.ports]]
    handlers = ["http"]
    port = 80

  [[services.ports]]
    handlers = ["tls", "http"]
    port = 443
```

---

## 8. Distributed Caching ด้วย Nebulex

**Nebulex** เป็น caching library ที่รองรับหลาย adapters รวมถึง distributed cache

```elixir
# mix.exs
{:nebulex, "~> 2.6"},
{:nebulex_adapters_partitioned, "~> 2.1"}
```

### ตั้งค่า Cache

```elixir
defmodule MyApp.DistributedCache do
  use Nebulex.Cache,
    otp_app: :my_app,
    adapter: NebulexAdaptersPartitioned
end
```

### config/config.exs

```elixir
config :my_app, MyApp.DistributedCache,
  # Primary cache (local)
  primary_storage_adapter: Nebulex.Adapters.Local,
  primary: [
    gc_interval: :timer.hours(12),
    max_size: 1_000_000,
    allocated_memory: 2_000_000_000  # 2GB
  ]
```

### ใช้งาน Cache

```elixir
defmodule MyApp.Users do
  alias MyApp.DistributedCache, as: Cache

  def get_user(user_id) do
    Cache.fetch("user:#{user_id}", fn ->
      # cache miss: ดึงจาก DB
      case Repo.get(User, user_id) do
        nil -> {:ignore, nil}
        user -> {:commit, user}
      end
    end,
    ttl: :timer.hours(1)
    )
  end

  def update_user(user, attrs) do
    case Repo.update(User.changeset(user, attrs)) do
      {:ok, updated} ->
        # invalidate cache เมื่อ update
        Cache.delete("user:#{user.id}")
        {:ok, updated}

      error ->
        error
    end
  end

  # ดึงข้อมูลหลาย users ด้วย cache
  def get_users(user_ids) do
    {cached, missing_ids} =
      user_ids
      |> Enum.map(fn id -> {id, Cache.get("user:#{id}")} end)
      |> Enum.split_with(fn {_, v} -> not is_nil(v) end)

    # ดึง missing จาก DB
    missing_users =
      if missing_ids != [] do
        ids = Enum.map(missing_ids, &elem(&1, 0))
        users = Repo.all(from u in User, where: u.id in ^ids)

        # เก็บเข้า cache
        Enum.each(users, fn u ->
          Cache.put("user:#{u.id}", u, ttl: :timer.hours(1))
        end)

        users
      else
        []
      end

    (Enum.map(cached, &elem(&1, 1)) ++ missing_users)
    |> Enum.sort_by(& &1.id)
  end
end
```

---

## 9. Network Partition Handling

**Network partition** เกิดเมื่อ nodes ในกลุ่มสื่อสารกันไม่ได้ ต้องออกแบบให้ระบบรับมือได้

### ใช้ libcluster สำหรับ Auto-reconnect

```elixir
# mix.exs
{:libcluster, "~> 3.3"}
```

```elixir
# config/config.exs
config :libcluster,
  topologies: [
    gossip: [
      strategy: Cluster.Strategy.Gossip,
      config: [
        port: 45892,
        if_addr: "0.0.0.0",
        multicast_addr: "230.1.1.251",
        multicast_ttl: 1
      ]
    ]
  ]
```

### Partition Tolerance Pattern

```elixir
defmodule MyApp.PartitionAware do
  use GenServer

  def init(state) do
    :net_kernel.monitor_nodes(true, node_type: :all)
    {:ok, state}
  end

  def handle_info({:nodedown, node, _info}, state) do
    IO.warn("Partition detected: #{node} is unreachable")

    # 1. ทำ local decisions โดยไม่ต้อง coordinate
    # 2. บันทึก operations ที่ต้อง sync ทีหลัง
    state = queue_for_sync(state, node)

    {:noreply, state}
  end

  def handle_info({:nodeup, node, _info}, state) do
    IO.puts("Partition healed: #{node} rejoined")

    # sync ข้อมูลที่ค้างอยู่
    reconcile_with_node(node, state.pending_sync)

    {:noreply, %{state | pending_sync: []}}
  end

  defp queue_for_sync(state, node) do
    Map.update(state, :pending_sync, [node], &[node | &1])
  end

  defp reconcile_with_node(node, pending) do
    # Vector clock หรือ CRDTs สำหรับ conflict resolution
    :rpc.call(node, __MODULE__, :merge_state, [pending])
  end
end
```

### Quorum-based Decisions

```elixir
defmodule MyApp.QuorumManager do
  @quorum_size 3  # ต้องการ majority

  def is_quorum_available? do
    cluster_size = length([Node.self() | Node.list()])
    cluster_size >= div(@quorum_size, 2) + 1
  end

  def execute_with_quorum(func) do
    if is_quorum_available?() do
      func.()
    else
      {:error, :no_quorum}
    end
  end
end

# ใช้งาน
MyApp.QuorumManager.execute_with_quorum(fn ->
  MyApp.CriticalOperation.run()
end)
```

---

## สรุป

```
Distributed Elixir Stack
├── Basics
│   ├── Node.connect/1      → เชื่อมต่อ nodes
│   ├── Node.list/0         → ดู cluster members
│   └── :net_kernel         → monitor node up/down
│
├── Communication
│   ├── send/2              → async message to remote pid
│   ├── :rpc.call/4         → synchronous remote call
│   └── :rpc.multicall/4    → broadcast to multiple nodes
│
├── Process Registration
│   ├── :global             → simple, แต่มี split-brain risk
│   └── Horde               → CRDT-based, partition tolerant
│
├── Load Distribution
│   ├── Consistent Hashing  → สม่ำเสมอ, minimal re-routing
│   └── Horde Distribution  → random/ring strategies
│
├── Clustering
│   ├── dns_cluster         → Fly.io DNS-based discovery
│   └── libcluster          → Gossip/Kubernetes/Epmd
│
├── Distributed Cache
│   └── Nebulex             → Partitioned cache across nodes
│
└── Fault Tolerance
    ├── Quorum              → majority decision
    ├── Partition detection → monitor_nodes
    └── Reconciliation      → sync on rejoin
```

---

*ก่อนหน้า: [Part 47 - GenStage and Flow](part_47.md) | ต่อไป: [Part 49 - Metaprogramming Advanced](part_49.md)*
