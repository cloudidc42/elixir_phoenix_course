# Part 23: Registry และ Process Naming

## เป้าหมายการเรียนรู้

หลังจากจบ Part นี้ คุณจะสามารถ:
- ตั้งชื่อ Process ด้วย `Process.register/2`
- ใช้ `Registry` module สำหรับ process naming ขั้นสูง
- เข้าใจ `via_tuple` pattern
- สร้าง dynamic named processes
- จัดการ process lifecycle ด้วย Registry

---

## 1. Process Registration พื้นฐาน

### Process.register/2

วิธีง่ายที่สุดในการตั้งชื่อ Process คือใช้ `Process.register/2`

```elixir
# สร้าง process
pid = spawn(fn ->
  receive do
    {:hello, from} -> send(from, :world)
  end
end)

# ตั้งชื่อ process
Process.register(pid, :my_process)

# ค้นหา PID จากชื่อ
IO.inspect(Process.whereis(:my_process))
# => #PID<0.123.0>

# ส่ง message ผ่านชื่อ
send(:my_process, {:hello, self()})

receive do
  msg -> IO.inspect(msg)  # => :world
end
```

### ข้อจำกัดของ Process.register/2

```elixir
# 1. ชื่อต้องเป็น atom
# Process.register(pid, "string_name")  # ERROR!

# 2. ชื่อต้อง unique ทั่วทั้ง node
# ถ้า register ชื่อเดิมซ้ำ จะ error

# 3. ชื่อไม่ได้ dynamic (ต้อง define ล่วงหน้า)

# ดู registered processes ทั้งหมด
IO.inspect(Process.registered())
# => [:mix, :logger, :user, ...]
```

### ใช้กับ GenServer

```elixir
defmodule Counter do
  use GenServer

  # ลงทะเบียน process ด้วยชื่อ module
  def start_link(initial_value) do
    GenServer.start_link(__MODULE__, initial_value, name: __MODULE__)
  end

  def increment do
    GenServer.cast(__MODULE__, :increment)
  end

  def value do
    GenServer.call(__MODULE__, :value)
  end

  @impl true
  def init(value), do: {:ok, value}

  @impl true
  def handle_cast(:increment, n), do: {:noreply, n + 1}

  @impl true
  def handle_call(:value, _from, n), do: {:reply, n, n}
end

{:ok, _pid} = Counter.start_link(0)

Counter.increment()
Counter.increment()
Counter.increment()

IO.puts(Counter.value())  # => 3

# process ชื่อ Counter สามารถ access ได้จากทุกที่
IO.inspect(Process.whereis(Counter))  # => #PID<0.xxx.0>
```

---

## 2. Registry Module

`Registry` ช่วยให้สามารถ register processes ด้วย **key ที่ dynamic** ได้ มีสองโหมด:
- `:unique` - แต่ละ key มีได้ 1 process
- `:duplicate` - แต่ละ key มีได้หลาย processes

### ติดตั้ง Registry

```elixir
# ไม่ต้องเพิ่ม dependency - Registry อยู่ใน Elixir standard library

# start Registry
{:ok, _} = Registry.start_link(keys: :unique, name: MyRegistry)
```

### Registry แบบ :unique

```elixir
defmodule UserRegistry do
  @registry __MODULE__

  def start_link do
    Registry.start_link(keys: :unique, name: @registry)
  end

  def register(user_id) do
    Registry.register(@registry, user_id, %{})
  end

  def find(user_id) do
    case Registry.lookup(@registry, user_id) do
      [{pid, _value}] -> {:ok, pid}
      [] -> {:error, :not_found}
    end
  end

  def all_users do
    Registry.select(@registry, [{{:_, :"$1", :_}, [], [:"$1"]}])
  end

  def count do
    Registry.count(@registry)
  end
end

# ใช้งาน
{:ok, _} = UserRegistry.start_link()

# สร้าง process และ register
pid1 = spawn(fn -> Process.sleep(:infinity) end)
pid2 = spawn(fn -> Process.sleep(:infinity) end)

# Register ด้วย user_id เป็น key
{:ok, _} = Registry.register(UserRegistry, "user:123", %{name: "Alice"})
{:ok, _} = Registry.register(UserRegistry, "user:456", %{name: "Bob"})

# ค้นหา
IO.inspect(Registry.lookup(UserRegistry, "user:123"))
# => [{#PID<0.xxx.0>, %{name: "Alice"}}]

IO.inspect(UserRegistry.count())
# => 2
```

### Registry แบบ :duplicate (Pub/Sub)

```elixir
defmodule PubSub do
  @registry __MODULE__

  def start_link do
    Registry.start_link(keys: :duplicate, name: @registry)
  end

  def subscribe(topic) do
    Registry.register(@registry, topic, [])
  end

  def publish(topic, message) do
    Registry.dispatch(@registry, topic, fn entries ->
      Enum.each(entries, fn {pid, _value} ->
        send(pid, {:message, topic, message})
      end)
    end)
  end

  def subscribers(topic) do
    Registry.lookup(@registry, topic)
    |> Enum.map(fn {pid, _} -> pid end)
  end
end

{:ok, _} = PubSub.start_link()

# Subscribe
spawn(fn ->
  PubSub.subscribe("news")
  receive do
    {:message, topic, msg} ->
      IO.puts("Subscriber 1 got [#{topic}]: #{msg}")
  end
end)

spawn(fn ->
  PubSub.subscribe("news")
  receive do
    {:message, topic, msg} ->
      IO.puts("Subscriber 2 got [#{topic}]: #{msg}")
  end
end)

Process.sleep(100)

# Publish
PubSub.publish("news", "Breaking news!")
# => "Subscriber 1 got [news]: Breaking news!"
# => "Subscriber 2 got [news]: Breaking news!"
```

---

## 3. via_tuple Pattern

`via_tuple` ใช้สำหรับ register GenServer ด้วย Registry แบบ dynamic

```elixir
defmodule GameSession do
  use GenServer

  @registry __MODULE__.Registry

  # via_tuple เชื่อม Registry กับ GenServer name
  defp via(game_id) do
    {:via, Registry, {@registry, game_id}}
  end

  def start_link(game_id, opts \\ []) do
    GenServer.start_link(__MODULE__, {game_id, opts}, name: via(game_id))
  end

  def join(game_id, player_name) do
    GenServer.call(via(game_id), {:join, player_name})
  end

  def leave(game_id, player_name) do
    GenServer.cast(via(game_id), {:leave, player_name})
  end

  def state(game_id) do
    GenServer.call(via(game_id), :state)
  end

  def exists?(game_id) do
    case Registry.lookup(@registry, game_id) do
      [{_pid, _}] -> true
      [] -> false
    end
  end

  # ============= Server Callbacks =============

  @impl true
  def init({game_id, _opts}) do
    state = %{
      game_id: game_id,
      players: [],
      status: :waiting,
      started_at: DateTime.utc_now()
    }
    {:ok, state}
  end

  @impl true
  def handle_call({:join, player}, _from, state) do
    if player in state.players do
      {:reply, {:error, :already_joined}, state}
    else
      new_state = %{state | players: [player | state.players]}
      {:reply, {:ok, new_state.players}, new_state}
    end
  end

  @impl true
  def handle_call(:state, _from, state) do
    {:reply, state, state}
  end

  @impl true
  def handle_cast({:leave, player}, state) do
    new_players = List.delete(state.players, player)
    {:noreply, %{state | players: new_players}}
  end
end
```

```elixir
# Setup Registry
{:ok, _} = Registry.start_link(keys: :unique, name: GameSession.Registry)

# สร้าง game sessions แบบ dynamic
{:ok, _} = GameSession.start_link("game-001")
{:ok, _} = GameSession.start_link("game-002")

# Join players
{:ok, players} = GameSession.join("game-001", "Alice")
IO.inspect(players)  # => ["Alice"]

{:ok, players} = GameSession.join("game-001", "Bob")
IO.inspect(players)  # => ["Bob", "Alice"]

{:ok, players} = GameSession.join("game-002", "Charlie")

# Check state
IO.inspect(GameSession.state("game-001"))
# => %{game_id: "game-001", players: ["Bob", "Alice"], ...}

# Check existence
IO.puts(GameSession.exists?("game-001"))  # => true
IO.puts(GameSession.exists?("game-999"))  # => false
```

---

## 4. Registry กับ Supervisor

```elixir
defmodule GameApp.Application do
  use Application

  @impl true
  def start(_type, _args) do
    children = [
      # Start Registry ก่อน
      {Registry, keys: :unique, name: GameSession.Registry},
      # Dynamic Supervisor สำหรับ game sessions
      {DynamicSupervisor, name: GameApp.GameSupervisor, strategy: :one_for_one}
    ]

    opts = [strategy: :one_for_one, name: GameApp.Supervisor]
    Supervisor.start_link(children, opts)
  end
end

defmodule GameApp.GameManager do
  @supervisor GameApp.GameSupervisor

  def create_game(game_id, opts \\ []) do
    spec = {GameSession, [game_id, opts]}
    DynamicSupervisor.start_child(@supervisor, spec)
  end

  def stop_game(game_id) do
    case Registry.lookup(GameSession.Registry, game_id) do
      [{pid, _}] ->
        DynamicSupervisor.terminate_child(@supervisor, pid)
      [] ->
        {:error, :not_found}
    end
  end

  def list_games do
    DynamicSupervisor.which_children(@supervisor)
    |> Enum.map(fn {_, pid, _, _} -> pid end)
    |> Enum.flat_map(fn pid ->
      Registry.keys(GameSession.Registry, pid)
    end)
  end
end
```

---

## 5. ตัวอย่างจริง: Named Process Manager

สร้าง process manager ที่จัดการ workers หลายตัวพร้อมกัน

```elixir
defmodule WorkerManager do
  @moduledoc """
  Dynamic worker manager ที่ใช้ Registry สำหรับ naming
  """

  # ========== Registry Setup ==========

  defmodule WorkerRegistry do
    def child_spec(_opts) do
      Registry.child_spec(keys: :unique, name: __MODULE__)
    end

    def via(worker_id) do
      {:via, Registry, {__MODULE__, worker_id}}
    end

    def find(worker_id) do
      case Registry.lookup(__MODULE__, worker_id) do
        [{pid, _}] -> {:ok, pid}
        [] -> {:error, :not_found}
      end
    end

    def all_workers do
      Registry.select(__MODULE__, [{{:"$1", :_, :_}, [], [:"$1"]}])
    end

    def count, do: Registry.count(__MODULE__)
  end

  # ========== Worker Process ==========

  defmodule Worker do
    use GenServer

    def start_link({worker_id, config}) do
      GenServer.start_link(__MODULE__, {worker_id, config},
        name: WorkerRegistry.via(worker_id))
    end

    def assign_task(worker_id, task) do
      GenServer.call(WorkerRegistry.via(worker_id), {:task, task})
    end

    def status(worker_id) do
      GenServer.call(WorkerRegistry.via(worker_id), :status)
    end

    def stop(worker_id) do
      GenServer.stop(WorkerRegistry.via(worker_id))
    end

    # ============= Callbacks =============

    @impl true
    def init({worker_id, config}) do
      state = %{
        id: worker_id,
        config: config,
        tasks_completed: 0,
        current_task: nil,
        status: :idle,
        started_at: DateTime.utc_now()
      }
      {:ok, state}
    end

    @impl true
    def handle_call({:task, task}, _from, %{status: :busy} = state) do
      {:reply, {:error, :worker_busy}, state}
    end

    @impl true
    def handle_call({:task, task}, _from, state) do
      # เริ่มทำ task แบบ async
      send(self(), {:execute, task})
      new_state = %{state | current_task: task, status: :busy}
      {:reply, {:ok, :accepted}, new_state}
    end

    @impl true
    def handle_call(:status, _from, state) do
      {:reply, state, state}
    end

    @impl true
    def handle_info({:execute, task}, state) do
      # จำลองการทำงาน
      result = execute_task(task, state.config)

      new_state = %{state |
        tasks_completed: state.tasks_completed + 1,
        current_task: nil,
        status: :idle
      }

      IO.puts("[Worker #{state.id}] Completed task: #{inspect(task)} => #{inspect(result)}")

      {:noreply, new_state}
    end

    defp execute_task(task, config) do
      # จำลอง task processing
      Process.sleep(Keyword.get(config, :delay, 100))
      {:ok, "Result for #{inspect(task)}"}
    end
  end

  # ========== Manager ==========

  defmodule Manager do
    use GenServer

    def start_link(opts \\ []) do
      GenServer.start_link(__MODULE__, opts, name: __MODULE__)
    end

    def spawn_worker(worker_id, config \\ []) do
      GenServer.call(__MODULE__, {:spawn, worker_id, config})
    end

    def kill_worker(worker_id) do
      GenServer.call(__MODULE__, {:kill, worker_id})
    end

    def assign(worker_id, task) do
      Worker.assign_task(worker_id, task)
    end

    def assign_any(task) do
      GenServer.call(__MODULE__, {:assign_any, task})
    end

    def list_workers do
      WorkerRegistry.all_workers()
    end

    def worker_stats do
      WorkerRegistry.all_workers()
      |> Enum.map(fn worker_id ->
        case Worker.status(worker_id) do
          state -> {worker_id, state}
        end
      end)
      |> Map.new()
    end

    # ============= Callbacks =============

    @impl true
    def init(_opts) do
      {:ok, %{supervisor: nil}}
    end

    @impl true
    def handle_call({:spawn, worker_id, config}, _from, state) do
      case DynamicSupervisor.start_child(
        WorkerManager.DynSupervisor,
        {Worker, {worker_id, config}}
      ) do
        {:ok, pid} ->
          IO.puts("Spawned worker: #{worker_id} (#{inspect(pid)})")
          {:reply, {:ok, pid}, state}
        {:error, reason} ->
          {:reply, {:error, reason}, state}
      end
    end

    @impl true
    def handle_call({:kill, worker_id}, _from, state) do
      case WorkerRegistry.find(worker_id) do
        {:ok, pid} ->
          DynamicSupervisor.terminate_child(WorkerManager.DynSupervisor, pid)
          {:reply, :ok, state}
        {:error, _} ->
          {:reply, {:error, :not_found}, state}
      end
    end

    @impl true
    def handle_call({:assign_any, task}, _from, state) do
      # หา worker ที่ idle
      idle_worker = WorkerRegistry.all_workers()
      |> Enum.find(fn worker_id ->
        case Worker.status(worker_id) do
          %{status: :idle} -> true
          _ -> false
        end
      end)

      result = case idle_worker do
        nil -> {:error, :no_idle_workers}
        worker_id -> Worker.assign_task(worker_id, task)
      end

      {:reply, result, state}
    end
  end

  # ========== Application ==========

  def start do
    children = [
      WorkerRegistry,
      {DynamicSupervisor, name: WorkerManager.DynSupervisor, strategy: :one_for_one},
      Manager
    ]

    Supervisor.start_link(children, strategy: :one_for_one, name: WorkerManager.Supervisor)
  end
end
```

```elixir
# ใช้งาน
{:ok, _} = WorkerManager.start()

# สร้าง workers
WorkerManager.Manager.spawn_worker("worker-1", delay: 200)
WorkerManager.Manager.spawn_worker("worker-2", delay: 150)
WorkerManager.Manager.spawn_worker("worker-3", delay: 100)

Process.sleep(100)

# Assign tasks
WorkerManager.Manager.assign("worker-1", %{type: :email, to: "alice@example.com"})
WorkerManager.Manager.assign("worker-2", %{type: :report, format: :pdf})

# หา idle worker แล้ว assign
WorkerManager.Manager.assign_any(%{type: :cleanup, path: "/tmp"})

# ดู status
IO.inspect(WorkerManager.Manager.list_workers())
# => ["worker-1", "worker-2", "worker-3"]

Process.sleep(500)

IO.inspect(WorkerManager.Manager.worker_stats())
# => %{"worker-1" => %{tasks_completed: 1, ...}, ...}

# Kill worker
WorkerManager.Manager.kill_worker("worker-3")

IO.inspect(WorkerManager.Manager.list_workers())
# => ["worker-1", "worker-2"]
```

---

## 6. Registry Metadata และ Lookup

```elixir
defmodule ServiceRegistry do
  @registry __MODULE__

  def start_link do
    Registry.start_link(keys: :unique, name: @registry)
  end

  def register_service(name, metadata) do
    Registry.register(@registry, name, metadata)
  end

  def find_service(name) do
    case Registry.lookup(@registry, name) do
      [{pid, metadata}] -> {:ok, pid, metadata}
      [] -> {:error, :not_found}
    end
  end

  # ค้นหาตาม metadata
  def find_by_type(type) do
    Registry.select(@registry, [
      {{:"$1", :"$2", %{type: type}}, [], [{{:"$1", :"$2"}}]}
    ])
  end

  # Update metadata
  def update_metadata(name, new_meta) do
    Registry.update_value(@registry, name, fn _old -> new_meta end)
  end
end

{:ok, _} = ServiceRegistry.start_link()

# Register services
spawn(fn ->
  ServiceRegistry.register_service("auth-service", %{type: :http, port: 4000, version: "1.0"})
  Process.sleep(:infinity)
end)

spawn(fn ->
  ServiceRegistry.register_service("db-service", %{type: :database, host: "localhost", version: "2.1"})
  Process.sleep(:infinity)
end)

spawn(fn ->
  ServiceRegistry.register_service("cache-service", %{type: :http, port: 6379, version: "1.5"})
  Process.sleep(:infinity)
end)

Process.sleep(100)

# Lookup
{:ok, pid, meta} = ServiceRegistry.find_service("auth-service")
IO.inspect({pid, meta})

# ค้นหา HTTP services ทั้งหมด
http_services = ServiceRegistry.find_by_type(:http)
IO.inspect(http_services)
```

---

## 7. Exercises

### Exercise 1: Chat Room Registry

สร้าง chat room system ที่ใช้ Registry

**เฉลย:**

```elixir
defmodule ChatRoom do
  use GenServer

  defp via(room_name) do
    {:via, Registry, {ChatRoom.Registry, room_name}}
  end

  def start_link(room_name) do
    GenServer.start_link(__MODULE__, room_name, name: via(room_name))
  end

  def join(room_name, user) do
    GenServer.call(via(room_name), {:join, user})
  end

  def leave(room_name, user) do
    GenServer.cast(via(room_name), {:leave, user})
  end

  def send_message(room_name, user, message) do
    GenServer.cast(via(room_name), {:message, user, message})
  end

  def history(room_name) do
    GenServer.call(via(room_name), :history)
  end

  def members(room_name) do
    GenServer.call(via(room_name), :members)
  end

  # Callbacks

  @impl true
  def init(room_name) do
    state = %{name: room_name, members: [], messages: []}
    {:ok, state}
  end

  @impl true
  def handle_call({:join, user}, _from, state) do
    if user in state.members do
      {:reply, {:error, :already_joined}, state}
    else
      new_state = %{state | members: [user | state.members]}
      {:reply, {:ok, length(new_state.members)}, new_state}
    end
  end

  @impl true
  def handle_call(:history, _from, state) do
    {:reply, Enum.reverse(state.messages), state}
  end

  @impl true
  def handle_call(:members, _from, state) do
    {:reply, state.members, state}
  end

  @impl true
  def handle_cast({:leave, user}, state) do
    {:noreply, %{state | members: List.delete(state.members, user)}}
  end

  @impl true
  def handle_cast({:message, user, content}, state) do
    msg = %{user: user, content: content, at: DateTime.utc_now()}
    {:noreply, %{state | messages: [msg | state.messages]}}
  end
end

# ใช้งาน
{:ok, _} = Registry.start_link(keys: :unique, name: ChatRoom.Registry)
{:ok, _} = DynamicSupervisor.start_link(name: ChatRoom.Supervisor, strategy: :one_for_one)

# สร้าง rooms
DynamicSupervisor.start_child(ChatRoom.Supervisor, {ChatRoom, "general"})
DynamicSupervisor.start_child(ChatRoom.Supervisor, {ChatRoom, "elixir"})

# Join และส่ง messages
ChatRoom.join("general", "Alice")
ChatRoom.join("general", "Bob")
ChatRoom.send_message("general", "Alice", "Hello everyone!")
ChatRoom.send_message("general", "Bob", "Hi Alice!")

IO.inspect(ChatRoom.members("general"))   # => ["Bob", "Alice"]
IO.inspect(ChatRoom.history("general"))   # => [%{user: "Bob", ...}, %{user: "Alice", ...}]
```

### Exercise 2: Load Balancer ด้วย Registry

สร้าง load balancer ที่ distribute work ไปยัง workers โดยใช้ round-robin

**เฉลย:**

```elixir
defmodule LoadBalancer do
  use GenServer

  def start_link(pool_name, worker_count) do
    GenServer.start_link(__MODULE__, {pool_name, worker_count}, name: pool_name)
  end

  def dispatch(pool_name, task) do
    GenServer.call(pool_name, {:dispatch, task})
  end

  @impl true
  def init({pool_name, worker_count}) do
    # Start workers
    workers = for i <- 1..worker_count do
      worker_id = :"#{pool_name}_worker_#{i}"
      {:ok, _pid} = GenServer.start_link(Worker, worker_id, name: worker_id)
      worker_id
    end

    state = %{workers: workers, current: 0}
    {:ok, state}
  end

  @impl true
  def handle_call({:dispatch, task}, _from, %{workers: workers, current: idx} = state) do
    worker = Enum.at(workers, rem(idx, length(workers)))
    result = GenServer.call(worker, {:execute, task})
    {:reply, result, %{state | current: idx + 1}}
  end

  defmodule Worker do
    use GenServer

    @impl true
    def init(id), do: {:ok, %{id: id, tasks: 0}}

    @impl true
    def handle_call({:execute, task}, _from, state) do
      result = "Worker #{state.id} processed: #{inspect(task)}"
      {:reply, result, %{state | tasks: state.tasks + 1}}
    end
  end
end

# ใช้งาน
{:ok, _} = LoadBalancer.start_link(:my_pool, 3)

for i <- 1..9 do
  result = LoadBalancer.dispatch(:my_pool, "Task #{i}")
  IO.puts(result)
end
# Worker my_pool_worker_1 processed: "Task 1"
# Worker my_pool_worker_2 processed: "Task 2"
# Worker my_pool_worker_3 processed: "Task 3"
# Worker my_pool_worker_1 processed: "Task 4"
# ...
```

---

## สรุป

```
Process Naming ใน Elixir:
├── Process.register/2
│   ├── ชื่อต้องเป็น atom
│   ├── Unique per node
│   └── ง่ายแต่ไม่ dynamic
├── Registry module
│   ├── :unique mode - 1 process per key
│   ├── :duplicate mode - หลาย process per key
│   ├── Keys เป็น any term (string, tuple, etc.)
│   └── lookup, dispatch, select, count
└── via_tuple pattern
    ├── {:via, Registry, {RegistryName, key}}
    ├── ใช้กับ GenServer name:
    └── Dynamic process naming

Use Cases:
├── Game sessions: game-001, game-002
├── User connections: user-123
├── Chat rooms: room-general
├── Service discovery
└── Worker pools
```

---

*ก่อนหน้า: [Part 22 - GenStage และ Flow](part_22.md) | ต่อไป: [Part 24 - Distributed Elixir](part_24.md)*
