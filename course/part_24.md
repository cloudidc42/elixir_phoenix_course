# Part 24: Distributed Elixir

## เป้าหมายการเรียนรู้

หลังจากจบ Part นี้ คุณจะสามารถ:
- เชื่อมต่อ Elixir nodes เข้าด้วยกัน
- ส่ง messages ระหว่าง nodes
- ใช้ `:rpc` module สำหรับ remote calls
- Register processes แบบ global
- สร้าง distributed counter เป็นตัวอย่างจริง

---

## 1. Elixir Distribution พื้นฐาน

### Node คืออะไร?

Node คือ instance ของ Erlang VM ที่รันอยู่ โดยแต่ละ node มีชื่อที่ unique

```
Node 1 (node1@localhost) ---- Network ---- Node 2 (node2@localhost)
     |                                          |
   Processes                               Processes
   (pid1, pid2...)                        (pid3, pid4...)
```

### เริ่ม Node ที่มีชื่อ

```bash
# Terminal 1: เริ่ม node แรก
iex --sname node1

# Terminal 2: เริ่ม node สอง
iex --sname node2

# หรือใช้ longname
iex --name node1@127.0.0.1
iex --name node2@127.0.0.1
```

```elixir
# ดูชื่อ node ปัจจุบัน
iex> Node.self()
:node1@localhost

# ดูว่า node ถูก start เป็น distributed หรือไม่
iex> Node.alive?()
true
```

### เชื่อมต่อ Nodes

```elixir
# จาก node1
iex(node1@localhost)> Node.connect(:"node2@localhost")
true

# ดู nodes ที่เชื่อมต่ออยู่
iex(node1@localhost)> Node.list()
[:"node2@localhost"]

# Disconnect
iex(node1@localhost)> Node.disconnect(:"node2@localhost")
true
```

### Cookie สำหรับ Security

Nodes ต้องมี cookie เดียวกันถึงจะเชื่อมต่อกันได้

```bash
# เริ่ม node พร้อมกำหนด cookie
iex --sname node1 --cookie secret_cookie
iex --sname node2 --cookie secret_cookie
```

```elixir
# ดู cookie ปัจจุบัน
iex> Node.get_cookie()
:secret_cookie

# เปลี่ยน cookie
iex> Node.set_cookie(:new_cookie)
```

---

## 2. ส่ง Messages ระหว่าง Nodes

### send/2 กับ Remote PID

```elixir
# Node 1: รับ message
iex(node1@localhost)> pid = spawn(fn ->
...>   receive do
...>     {:hello, from} ->
...>       IO.puts("Got hello from #{inspect(from)}")
...>       send(from, :world)
...>   end
...> end)
#PID<0.123.0>

iex(node1@localhost)> IO.inspect(pid)
#PID<0.123.0>
```

```elixir
# Node 2: ส่ง message ไปยัง node1
iex(node2@localhost)> remote_pid = :rpc.call(:"node1@localhost", Process, :whereis, [:some_name])

# หรือส่งตรงถ้ารู้ PID
iex(node2@localhost)> send({:node1@localhost, pid}, {:hello, self()})

# รับ reply
receive do
  msg -> IO.puts("Reply: #{inspect(msg)}")
end
```

### ส่งผ่านชื่อ Process

```elixir
# Node 1: register process
iex(node1@localhost)> Process.register(self(), :mailbox)

# Node 2: ส่งไปยัง named process บน node1
iex(node2@localhost)> send({:mailbox, :"node1@localhost"}, "Hello from node2!")
```

---

## 3. :rpc Module

`:rpc` ช่วยให้เรียก function บน remote node ได้

```elixir
# :rpc.call/4 - synchronous call
result = :rpc.call(
  :"node2@localhost",  # remote node
  String,              # module
  :upcase,             # function
  ["hello"]            # arguments
)
IO.puts(result)  # => "HELLO"

# :rpc.cast/4 - asynchronous (fire and forget)
:rpc.cast(:"node2@localhost", IO, :puts, ["Message from node1!"])

# :rpc.multicall/4 - call function on multiple nodes
{results, bad_nodes} = :rpc.multicall(
  [:"node1@localhost", :"node2@localhost"],
  System,
  :os_time,
  [:millisecond]
)
IO.inspect(results)     # => [1234567890, 1234567891]
IO.inspect(bad_nodes)   # => []  (nodes ที่ fail)
```

### Error Handling กับ :rpc

```elixir
defmodule RPC do
  def call(node, module, function, args, timeout \\ 5000) do
    case :rpc.call(node, module, function, args, timeout) do
      {:badrpc, reason} ->
        {:error, {:rpc_failed, reason}}
      result ->
        {:ok, result}
    end
  end

  def multicall(nodes, module, function, args) do
    {results, bad_nodes} = :rpc.multicall(nodes, module, function, args)

    successful = Enum.zip(nodes -- bad_nodes, results)

    %{
      successful: Map.new(successful),
      failed: bad_nodes
    }
  end
end

# ใช้งาน
case RPC.call(:"node2@localhost", MyApp, :get_data, []) do
  {:ok, data} -> IO.inspect(data)
  {:error, {:rpc_failed, :nodedown}} -> IO.puts("Node is down!")
  {:error, reason} -> IO.puts("Error: #{inspect(reason)}")
end
```

---

## 4. Global Process Registration

`:global` module ให้ register process ที่ visible จากทุก node

```elixir
# Node 1: register globally
:global.register_name(:master_process, self())

# Node 2: ค้นหา globally registered process
pid = :global.whereis_name(:master_process)
IO.inspect(pid)  # => #PID<node1@localhost.123.0>

# ส่ง message
send(pid, :hello)

# Unregister
:global.unregister_name(:master_process)
```

### ระวัง Split Brain!

```
Node1 --- เครือข่ายขาด --- Node2
  |                           |
  register :master            register :master
  (conflict!)
```

```elixir
# Global module แก้ conflict ด้วย conflict handler
:global.register_name(
  :my_service,
  self(),
  fn name, pid1, pid2 ->
    # เลือก process ที่จะ keep
    # กฎ: เลือก PID ที่เล็กกว่า (older process)
    min(pid1, pid2)
  end
)
```

---

## 5. :pg (Process Groups)

`:pg` ใช้สำหรับ group processes ข้าม nodes

```elixir
# เริ่ม pg
:pg.start_link()

# หรือใน Supervisor
children = [
  :pg
]

# Join group
:pg.join(:my_group, self())

# ดู members ทุก node
members = :pg.get_members(:my_group)
IO.inspect(members)  # => [pid1, pid2, ...]

# ดู members บน local node เท่านั้น
local_members = :pg.get_local_members(:my_group)

# Leave group
:pg.leave(:my_group, self())
```

---

## 6. Node Monitoring

```elixir
# Monitor node
Node.monitor(:"node2@localhost", true)

# รับ notification เมื่อ node down/up
receive do
  {:nodedown, node} ->
    IO.puts("Node #{node} went down!")
  {:nodeup, node} ->
    IO.puts("Node #{node} came up!")
end

# Monitor ทุก nodes ที่ connect/disconnect
:net_kernel.monitor_nodes(true)

# ดู current nodes
IO.inspect(Node.list())                    # connected nodes
IO.inspect(Node.list(:connected))          # same as above
IO.inspect(Node.list(:visible))            # visible nodes
IO.inspect(Node.list(:hidden))             # hidden nodes
```

---

## 7. ตัวอย่างจริง: Distributed Counter

สร้าง counter ที่ consistent ข้ามหลาย nodes

```elixir
defmodule DistributedCounter do
  @moduledoc """
  Distributed counter ที่ sync ข้ามหลาย nodes
  ใช้ :global สำหรับ master election
  """

  use GenServer

  @counter_name :distributed_counter

  # =========== Client API ===========

  def start_link(_opts \\ []) do
    GenServer.start_link(__MODULE__, :ok)
  end

  def increment(amount \\ 1) do
    call_master({:increment, amount})
  end

  def decrement(amount \\ 1) do
    call_master({:decrement, amount})
  end

  def get_value do
    call_master(:get_value)
  end

  def reset do
    call_master(:reset)
  end

  defp call_master(request) do
    case :global.whereis_name(@counter_name) do
      :undefined ->
        {:error, :no_master}
      pid ->
        GenServer.call(pid, request)
    end
  end

  # =========== Master Election ===========

  def become_master do
    GenServer.call(__MODULE__, :try_become_master)
  end

  def master_node do
    case :global.whereis_name(@counter_name) do
      :undefined -> nil
      pid -> node(pid)
    end
  end

  # =========== Server Callbacks ===========

  @impl true
  def init(:ok) do
    # ลอง become master เมื่อ start
    send(self(), :try_register)
    state = %{value: 0, is_master: false}
    {:ok, state}
  end

  @impl true
  def handle_info(:try_register, state) do
    is_master = try_register_as_master()
    {:noreply, %{state | is_master: is_master}}
  end

  @impl true
  def handle_info({:nodedown, node}, state) do
    IO.puts("Node #{node} went down!")

    # ถ้า master node down, ลองเป็น master
    if master_node() == nil do
      IO.puts("Trying to become new master...")
      is_master = try_register_as_master()
      {:noreply, %{state | is_master: is_master}}
    else
      {:noreply, state}
    end
  end

  @impl true
  def handle_call(:try_become_master, _from, state) do
    is_master = try_register_as_master()
    {:reply, is_master, %{state | is_master: is_master}}
  end

  @impl true
  def handle_call({:increment, amount}, _from, %{is_master: true} = state) do
    new_value = state.value + amount
    broadcast_update(new_value)
    {:reply, {:ok, new_value}, %{state | value: new_value}}
  end

  @impl true
  def handle_call({:decrement, amount}, _from, %{is_master: true} = state) do
    new_value = state.value - amount
    broadcast_update(new_value)
    {:reply, {:ok, new_value}, %{state | value: new_value}}
  end

  @impl true
  def handle_call(:get_value, _from, %{is_master: true} = state) do
    {:reply, {:ok, state.value}, state}
  end

  @impl true
  def handle_call(:reset, _from, %{is_master: true} = state) do
    broadcast_update(0)
    {:reply, :ok, %{state | value: 0}}
  end

  @impl true
  def handle_cast({:sync_value, value}, state) do
    {:noreply, %{state | value: value}}
  end

  # =========== Private Helpers ===========

  defp try_register_as_master do
    # ลอง register ด้วย :global
    case :global.register_name(@counter_name, self(), &resolve_conflict/3) do
      :yes ->
        IO.puts("#{Node.self()} is now the master!")
        # Monitor all nodes
        :net_kernel.monitor_nodes(true)
        true
      :no ->
        IO.puts("#{Node.self()} is a replica")
        false
    end
  end

  defp resolve_conflict(_name, pid1, pid2) do
    # เลือก process เก่ากว่า (PID ที่สร้างก่อน)
    older = if :erlang.pid_to_list(pid1) < :erlang.pid_to_list(pid2) do
      pid1
    else
      pid2
    end
    IO.puts("Master conflict resolved: #{inspect(older)} wins")
    older
  end

  defp broadcast_update(value) do
    # ส่งค่าใหม่ไปยังทุก node
    all_pids = :pg.get_members(:counter_replicas)
    Enum.each(all_pids, fn pid ->
      unless pid == self() do
        GenServer.cast(pid, {:sync_value, value})
      end
    end)
  end
end
```

### Distributed Counter Application

```elixir
defmodule CounterApp do
  @moduledoc """
  Application ที่ใช้ DistributedCounter
  """

  def start do
    # Setup
    :pg.start_link()

    {:ok, pid} = DistributedCounter.start_link()
    :pg.join(:counter_replicas, pid)

    Process.register(pid, DistributedCounter)

    # ลองเป็น master
    DistributedCounter.become_master()

    pid
  end

  def demo do
    # Simulate distributed operations
    IO.puts("Master node: #{DistributedCounter.master_node()}")

    {:ok, v1} = DistributedCounter.increment(10)
    IO.puts("After +10: #{v1}")

    {:ok, v2} = DistributedCounter.increment(5)
    IO.puts("After +5: #{v2}")

    {:ok, v3} = DistributedCounter.decrement(3)
    IO.puts("After -3: #{v3}")

    {:ok, current} = DistributedCounter.get_value()
    IO.puts("Current value: #{current}")

    DistributedCounter.reset()
    {:ok, after_reset} = DistributedCounter.get_value()
    IO.puts("After reset: #{after_reset}")
  end
end
```

---

## 8. Distributed Task Execution

```elixir
defmodule DistributedTask do
  @doc """
  รัน task บน node ที่ load ต่ำที่สุด
  """

  def run_on_best_node(fun) do
    best_node = find_best_node()
    IO.puts("Running on: #{best_node}")

    case :rpc.call(best_node, __MODULE__, :execute, [fun]) do
      {:badrpc, reason} -> {:error, reason}
      result -> {:ok, result}
    end
  end

  def execute(fun), do: fun.()

  def run_on_all_nodes(fun) do
    nodes = [Node.self() | Node.list()]
    tasks = Enum.map(nodes, fn node ->
      Task.async(fn ->
        case :rpc.call(node, __MODULE__, :execute, [fun]) do
          {:badrpc, reason} -> {:error, node, reason}
          result -> {:ok, node, result}
        end
      end)
    end)

    Task.await_many(tasks, 10_000)
  end

  def map_reduce(data, map_fn, reduce_fn) do
    nodes = [Node.self() | Node.list()]
    chunks = chunk_data(data, length(nodes))

    # Map phase: distribute across nodes
    map_results =
      Enum.zip(nodes, chunks)
      |> Enum.map(fn {node, chunk} ->
        Task.async(fn ->
          :rpc.call(node, Enum, :map, [chunk, map_fn])
        end)
      end)
      |> Task.await_many(30_000)
      |> List.flatten()

    # Reduce phase: run locally
    Enum.reduce(map_results, reduce_fn)
  end

  defp find_best_node do
    nodes = [Node.self() | Node.list()]

    node_loads = Enum.map(nodes, fn node ->
      load = get_node_load(node)
      {node, load}
    end)

    {best_node, _load} = Enum.min_by(node_loads, fn {_node, load} -> load end)
    best_node
  end

  defp get_node_load(node) do
    case :rpc.call(node, :cpu_sup, :avg1, []) do
      {:badrpc, _} -> 999  # ถ้าไม่ได้ รือน ให้ load สูง
      load -> load
    end
  end

  defp chunk_data(data, n) when is_list(data) do
    chunk_size = ceil(length(data) / n)
    Enum.chunk_every(data, chunk_size)
  end
end
```

---

## 9. ทดสอบ Distributed Elixir แบบ Local

```elixir
# ทดสอบ distribution ในเครื่องเดียว

defmodule LocalCluster do
  @doc """
  เริ่ม local cluster สำหรับ testing
  """

  def start(node_count \\ 2) do
    # เริ่ม nodes โดยใช้ :slave module (deprecated แต่ยังใช้ได้)
    nodes = for i <- 1..node_count do
      node_name = :"node#{i}@127.0.0.1"
      {:ok, node} = :slave.start('127.0.0.1', :"node#{i}")
      # Load code ไปยัง node
      :rpc.call(node, :code, :add_pathz, [:code.get_path()])
      node
    end

    nodes
  end

  def stop(nodes) do
    Enum.each(nodes, &:slave.stop/1)
  end
end

# การทดสอบ
defmodule DistributedTest do
  def run do
    IO.puts("Starting cluster...")

    # วิธีง่ายกว่า: ใช้ Node.start ใน test
    {:ok, _} = Node.start(:"main@127.0.0.1", :shortnames)
    Node.set_cookie(:test_cookie)

    # ทดสอบ :rpc
    result = :rpc.call(Node.self(), String, :upcase, ["hello"])
    IO.puts("Local RPC result: #{result}")

    # ทดสอบ :global
    Process.register(self(), :test_process)
    :global.register_name(:global_test, self())

    found = :global.whereis_name(:global_test)
    IO.puts("Global lookup: #{inspect(found)}")
    IO.puts("Same PID? #{found == self()}")
  end
end

DistributedTest.run()
```

---

## 10. Exercises

### Exercise 1: Distributed Key-Value Store

สร้าง KV store ที่ replicate ข้ามหลาย nodes

**เฉลย:**

```elixir
defmodule DistributedKV do
  use GenServer

  # Client API

  def start_link(name) do
    GenServer.start_link(__MODULE__, %{}, name: {:global, {__MODULE__, name}})
  end

  def put(store_name, key, value) do
    case :global.whereis_name({__MODULE__, store_name}) do
      :undefined -> {:error, :store_not_found}
      pid -> GenServer.call(pid, {:put, key, value})
    end
  end

  def get(store_name, key) do
    case :global.whereis_name({__MODULE__, store_name}) do
      :undefined -> {:error, :store_not_found}
      pid -> GenServer.call(pid, {:get, key})
    end
  end

  def delete(store_name, key) do
    case :global.whereis_name({__MODULE__, store_name}) do
      :undefined -> {:error, :store_not_found}
      pid -> GenServer.call(pid, {:delete, key})
    end
  end

  def all(store_name) do
    case :global.whereis_name({__MODULE__, store_name}) do
      :undefined -> {:error, :store_not_found}
      pid -> GenServer.call(pid, :all)
    end
  end

  # Server Callbacks

  @impl true
  def init(state), do: {:ok, state}

  @impl true
  def handle_call({:put, key, value}, _from, state) do
    {:reply, :ok, Map.put(state, key, value)}
  end

  @impl true
  def handle_call({:get, key}, _from, state) do
    {:reply, Map.get(state, key), state}
  end

  @impl true
  def handle_call({:delete, key}, _from, state) do
    {:reply, :ok, Map.delete(state, key)}
  end

  @impl true
  def handle_call(:all, _from, state) do
    {:reply, state, state}
  end
end

# ใช้งาน (ต้องรันบน distributed node)
{:ok, _} = DistributedKV.start_link("my_store")

DistributedKV.put("my_store", :user, %{name: "Alice", age: 30})
DistributedKV.put("my_store", :config, %{debug: true})

IO.inspect(DistributedKV.get("my_store", :user))
# => %{name: "Alice", age: 30}

IO.inspect(DistributedKV.all("my_store"))
# => %{user: %{...}, config: %{...}}
```

### Exercise 2: Cluster Health Monitor

สร้าง monitor ที่ติดตาม health ของทุก nodes

**เฉลย:**

```elixir
defmodule ClusterMonitor do
  use GenServer

  def start_link(_) do
    GenServer.start_link(__MODULE__, %{}, name: __MODULE__)
  end

  def status do
    GenServer.call(__MODULE__, :status)
  end

  @impl true
  def init(_) do
    :net_kernel.monitor_nodes(true)
    # ตรวจ health ทุก 5 วินาที
    :timer.send_interval(5000, :check_health)

    state = %{
      nodes: %{},
      events: []
    }

    {:ok, update_node_status(state)}
  end

  @impl true
  def handle_info({:nodeup, node}, state) do
    IO.puts("[Monitor] Node UP: #{node}")
    event = %{type: :nodeup, node: node, at: DateTime.utc_now()}
    new_state = state
    |> Map.update!(:events, &[event | &1])
    |> update_node_status()
    {:noreply, new_state}
  end

  @impl true
  def handle_info({:nodedown, node}, state) do
    IO.puts("[Monitor] Node DOWN: #{node}")
    event = %{type: :nodedown, node: node, at: DateTime.utc_now()}
    new_state = state
    |> Map.update!(:events, &[event | &1])
    |> update_node_status()
    {:noreply, new_state}
  end

  @impl true
  def handle_info(:check_health, state) do
    {:noreply, update_node_status(state)}
  end

  @impl true
  def handle_call(:status, _from, state) do
    {:reply, state, state}
  end

  defp update_node_status(state) do
    all_nodes = [Node.self() | Node.list()]

    node_statuses = Map.new(all_nodes, fn node ->
      health = check_node_health(node)
      {node, health}
    end)

    %{state | nodes: node_statuses}
  end

  defp check_node_health(node) do
    case :rpc.call(node, :erlang, :statistics, [:run_queue], 2000) do
      {:badrpc, reason} ->
        %{status: :unhealthy, reason: reason}
      run_queue ->
        %{
          status: :healthy,
          run_queue: run_queue,
          memory: get_memory(node),
          checked_at: DateTime.utc_now()
        }
    end
  end

  defp get_memory(node) do
    case :rpc.call(node, :erlang, :memory, [:total], 2000) do
      {:badrpc, _} -> :unknown
      bytes -> div(bytes, 1_048_576)  # MB
    end
  end
end

# ใช้งาน
{:ok, _} = ClusterMonitor.start_link(nil)

Process.sleep(1000)

status = ClusterMonitor.status()
IO.inspect(status.nodes)
```

---

## สรุป

```
Distributed Elixir:
├── Nodes
│   ├── Node.connect/1, Node.disconnect/1
│   ├── Node.list/0, Node.self/0
│   └── Cookie สำหรับ authentication
├── Message Passing
│   ├── send({name, node}, message)
│   ├── send(remote_pid, message)
│   └── Process.register ข้าม nodes
├── :rpc module
│   ├── :rpc.call/4 - synchronous
│   ├── :rpc.cast/4 - asynchronous
│   └── :rpc.multicall/4 - all nodes
├── :global module
│   ├── Global process registration
│   ├── resolve_conflict callback
│   └── ระวัง split brain
└── :pg module
    ├── Process groups
    ├── Cross-node broadcasting
    └── get_members, join, leave

ข้อควรระวัง:
├── Network partition
├── Split brain
├── Performance (serialization overhead)
└── ใช้ library เช่น Horde, libcluster สำหรับ production
```

---

*ก่อนหน้า: [Part 23 - Registry และ Process Naming](part_23.md) | ต่อไป: [Part 25 - Phoenix Framework](part_25.md)*
