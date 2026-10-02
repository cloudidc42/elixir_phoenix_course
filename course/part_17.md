# Part 17: Supervisor และ OTP Trees

## เป้าหมายการเรียนรู้

หลังจากจบ Part นี้ คุณจะสามารถ:
- เข้าใจ Supervisor และ supervision strategies
- กำหนด child specs สำหรับ supervised processes
- ใช้ DynamicSupervisor สำหรับ dynamic process management
- สร้าง supervision tree ที่เหมาะสม
- สร้าง Worker Pool ด้วย Supervisor

---

## 1. Supervisor คืออะไร?

Supervisor คือ process พิเศษที่คอย watch และ restart child processes

```
Supervision Tree:
        Application
            │
        Supervisor
       /     │     \
   Worker  Worker  Sub-Supervisor
                   /    \
               Worker  Worker

ถ้า Worker crash -> Supervisor restart ให้อัตโนมัติ
```

---

## 2. Supervisor Strategies

```
Strategies:
┌───────────────────────────────────────────────────────────┐
│  :one_for_one  - restart เฉพาะ process ที่ crash         │
│  :one_for_all  - restart ทุก children ถ้ามีคนนึง crash   │
│  :rest_for_one - restart คนที่ crash + ทุกคนหลังจากนั้น  │
└───────────────────────────────────────────────────────────┘
```

### one_for_one (ใช้บ่อยสุด)

```
ก่อน crash: [A] [B] [C]
C crash ->  [A] [B] [C'] ← restart C เท่านั้น
```

### one_for_all

```
ก่อน crash: [A] [B] [C]
B crash ->  [A'] [B'] [C'] ← restart ทุกคน
```

### rest_for_one

```
ก่อน crash: [A] [B] [C] [D]
B crash ->  [A] [B'] [C'] [D'] ← restart B, C, D (B onwards)
```

---

## 3. Basic Supervisor

```elixir
defmodule MyApp.Supervisor do
  use Supervisor

  def start_link(opts) do
    Supervisor.start_link(__MODULE__, opts, name: __MODULE__)
  end

  @impl Supervisor
  def init(_opts) do
    children = [
      # {Module, args}
      {MyApp.Cache, []},
      {MyApp.Worker, name: :worker_1},
      {MyApp.Worker, name: :worker_2},
    ]

    Supervisor.init(children, strategy: :one_for_one)
  end
end
```

---

## 4. Child Spec

```elixir
# Child spec กำหนดว่า Supervisor จะ start/stop/restart child อย่างไร

# แบบ explicit
{
  id: MyWorker,                # unique id ใน supervisor
  start: {MyWorker, :start_link, [args]},  # MFA
  restart: :permanent,         # :permanent | :temporary | :transient
  shutdown: 5000,              # ms หรือ :brutal_kill หรือ :infinity
  type: :worker                # :worker | :supervisor
}

# แบบ shorthand (ถ้า module มี child_spec/1)
MyWorker  # ใช้ default child_spec
{MyWorker, args}  # ส่ง args
```

### Restart types

```
:permanent  - restart เสมอ (GenServer, ใช้บ่อยสุด)
:temporary  - ไม่ restart เลย (one-off tasks)
:transient  - restart เฉพาะถ้า crash (ไม่ restart ถ้าจบปกติ)
```

---

## 5. GenServer ที่ Supervisor-friendly

```elixir
defmodule Cache.Server do
  use GenServer

  # child_spec สำหรับ Supervisor
  def child_spec(opts) do
    %{
      id: __MODULE__,
      start: {__MODULE__, :start_link, [opts]},
      restart: :permanent,
      shutdown: 5000,
      type: :worker
    }
  end

  def start_link(opts) do
    name = Keyword.get(opts, :name, __MODULE__)
    GenServer.start_link(__MODULE__, opts, name: name)
  end

  @impl GenServer
  def init(opts) do
    ttl = Keyword.get(opts, :ttl, 300)
    {:ok, %{entries: %{}, default_ttl: ttl}}
  end

  # ... rest of implementation
end

# ใน Supervisor
children = [
  {Cache.Server, name: MyApp.Cache, ttl: 600}
]
```

---

## 6. Application Supervisor

```elixir
# lib/my_app/application.ex
defmodule MyApp.Application do
  use Application

  @impl true
  def start(_type, _args) do
    children = [
      # Registry
      {Registry, keys: :unique, name: MyApp.Registry},
      # PubSub
      {Phoenix.PubSub, name: MyApp.PubSub},
      # Database pool
      MyApp.Repo,
      # Cache
      {MyApp.Cache, ttl: 300},
      # Web endpoint
      MyAppWeb.Endpoint
    ]

    opts = [strategy: :one_for_one, name: MyApp.Supervisor]
    Supervisor.start_link(children, opts)
  end
end

# mix.exs
def application do
  [
    mod: {MyApp.Application, []},
    extra_applications: [:logger, :runtime_tools]
  ]
end
```

---

## 7. DynamicSupervisor

สำหรับ spawn processes แบบ dynamic (ไม่รู้จำนวนล่วงหน้า)

```elixir
defmodule MyApp.DynamicWorkerSupervisor do
  use DynamicSupervisor

  def start_link(opts) do
    DynamicSupervisor.start_link(__MODULE__, opts, name: __MODULE__)
  end

  @impl DynamicSupervisor
  def init(_opts) do
    DynamicSupervisor.init(strategy: :one_for_one)
  end

  def start_worker(args) do
    DynamicSupervisor.start_child(__MODULE__, {MyWorker, args})
  end

  def stop_worker(pid) do
    DynamicSupervisor.terminate_child(__MODULE__, pid)
  end

  def list_workers do
    DynamicSupervisor.which_children(__MODULE__)
  end

  def count_workers do
    DynamicSupervisor.count_children(__MODULE__)
  end
end
```

---

## 8. ตัวอย่างจริง: Worker Pool

```elixir
# Worker ที่ทำงานจริงๆ
defmodule WorkerPool.Worker do
  use GenServer

  def start_link(opts) do
    id = Keyword.fetch!(opts, :id)
    GenServer.start_link(__MODULE__, opts, name: via(id))
  end

  def process(id, job) do
    GenServer.call(via(id), {:process, job}, 30_000)
  end

  def status(id) do
    GenServer.call(via(id), :status)
  end

  defp via(id), do: {:via, Registry, {WorkerPool.Registry, {:worker, id}}}

  @impl GenServer
  def init(opts) do
    id = Keyword.fetch!(opts, :id)
    IO.puts("Worker #{id} started")
    {:ok, %{id: id, jobs_processed: 0, current_job: nil}}
  end

  @impl GenServer
  def handle_call({:process, job}, _from, state) do
    IO.puts("Worker #{state.id} processing job: #{inspect(job)}")
    # Simulate work
    result = do_work(job)
    new_state = %{state | jobs_processed: state.jobs_processed + 1}
    {:reply, {:ok, result}, new_state}
  end

  def handle_call(:status, _from, state) do
    {:reply, %{id: state.id, jobs: state.jobs_processed}, state}
  end

  defp do_work(job) do
    :timer.sleep(:rand.uniform(200))
    Map.put(job, :result, "processed by worker")
  end
end

# Pool Manager
defmodule WorkerPool.Manager do
  use GenServer

  def start_link(opts) do
    size = Keyword.get(opts, :size, 3)
    GenServer.start_link(__MODULE__, size, name: __MODULE__)
  end

  def submit(job) do
    GenServer.call(__MODULE__, {:submit, job}, 60_000)
  end

  def stats do
    GenServer.call(__MODULE__, :stats)
  end

  @impl GenServer
  def init(pool_size) do
    # Start workers ใต้ supervision
    worker_ids = Enum.to_list(1..pool_size)
    {:ok, %{workers: worker_ids, round_robin: 0, total_submitted: 0}}
  end

  @impl GenServer
  def handle_call({:submit, job}, _from, state) do
    worker_id = Enum.at(state.workers, rem(state.round_robin, length(state.workers)))
    result = WorkerPool.Worker.process(worker_id, job)
    new_state = %{state |
      round_robin: state.round_robin + 1,
      total_submitted: state.total_submitted + 1
    }
    {:reply, result, new_state}
  end

  def handle_call(:stats, _from, state) do
    worker_stats = Enum.map(state.workers, fn id ->
      WorkerPool.Worker.status(id)
    end)
    stats = %{
      pool_size: length(state.workers),
      total_submitted: state.total_submitted,
      workers: worker_stats
    }
    {:reply, stats, state}
  end
end

# Supervisor Tree สำหรับ Worker Pool
defmodule WorkerPool.Supervisor do
  use Supervisor

  def start_link(opts) do
    Supervisor.start_link(__MODULE__, opts, name: __MODULE__)
  end

  @impl Supervisor
  def init(opts) do
    pool_size = Keyword.get(opts, :size, 3)

    worker_children = Enum.map(1..pool_size, fn i ->
      Supervisor.child_spec(
        {WorkerPool.Worker, id: i},
        id: {:worker, i}
      )
    end)

    children = [
      {Registry, keys: :unique, name: WorkerPool.Registry},
      {WorkerPool.Manager, size: pool_size}
    ] ++ worker_children

    Supervisor.init(children, strategy: :one_for_one)
  end
end

# ใช้งาน
{:ok, _} = WorkerPool.Supervisor.start_link(size: 3)

# Submit jobs
results = Enum.map(1..9, fn i ->
  WorkerPool.Manager.submit(%{id: i, data: "job_#{i}"})
end)

WorkerPool.Manager.stats()
# %{pool_size: 3, total_submitted: 9, workers: [...]}
```

---

## 9. Nested Supervisors

```elixir
defmodule MyApp.Supervisor do
  use Supervisor

  def start_link(opts) do
    Supervisor.start_link(__MODULE__, opts, name: __MODULE__)
  end

  @impl Supervisor
  def init(_opts) do
    children = [
      # Core services (one_for_all - ถ้า core crash ให้ restart ทุกอย่าง)
      MyApp.Database,
      MyApp.Cache,

      # Worker subsystem (แยก supervisor)
      {WorkerSubsystem.Supervisor, size: 5},

      # Web subsystem
      {WebSubsystem.Supervisor, port: 4000},
    ]

    Supervisor.init(children, strategy: :one_for_one)
  end
end

defmodule WorkerSubsystem.Supervisor do
  use Supervisor

  def start_link(opts) do
    Supervisor.start_link(__MODULE__, opts, name: __MODULE__)
  end

  @impl Supervisor
  def init(opts) do
    size = Keyword.get(opts, :size, 3)
    workers = Enum.map(1..size, fn i ->
      Supervisor.child_spec({Worker, id: i}, id: {:worker, i})
    end)
    Supervisor.init(workers, strategy: :one_for_one)
  end
end
```

---

## 10. restart_limit ป้องกัน restart loop

```elixir
Supervisor.init(children, [
  strategy: :one_for_one,
  max_restarts: 3,    # restart ได้สูงสุด 3 ครั้ง
  max_seconds: 5      # ภายใน 5 วินาที
  # ถ้าเกิน limit -> supervisor เองก็ crash
])
```

---

## 11. Process.supervisor และ inspect tree

```elixir
# ดู supervision tree
Supervisor.which_children(MyApp.Supervisor)
# [
#   {:worker_1, #PID<0.123.0>, :worker, [MyWorker]},
#   {:cache, #PID<0.124.0>, :worker, [MyApp.Cache]},
# ]

# นับ children
Supervisor.count_children(MyApp.Supervisor)
# %{active: 3, specs: 3, supervisors: 0, workers: 3}
```

---

## 12. Exercises

### Exercise 1: Chat Room Supervisor

```elixir
# สร้าง supervised chat system:
# - ChatSupervisor supervise ChatRoom processes
# - แต่ละ room เป็น GenServer แยกกัน
# - Room crash -> Supervisor restart ให้
# - DynamicSupervisor สำหรับสร้าง rooms ตอน runtime

defmodule ChatSystem.Supervisor do
  use DynamicSupervisor

  def start_link(opts), do: DynamicSupervisor.start_link(__MODULE__, opts, name: __MODULE__)

  def create_room(name) do
    # TODO: start ChatRoom.Server ภายใต้ supervisor
  end

  def destroy_room(pid) do
    # TODO: terminate child
  end
end

defmodule ChatRoom.Server do
  use GenServer
  # TODO: implement join, leave, send_message
end
```

### Exercise 2: Retry Supervisor

```elixir
# สร้าง supervisor ที่:
# - ถ้า worker fail ให้รอ exponential backoff ก่อน restart
# - 1st failure: รอ 1 วิ
# - 2nd failure: รอ 2 วิ
# - 3rd failure: รอ 4 วิ
# - หลังจาก 5 ครั้ง: stop

defmodule RetryWorker do
  use GenServer
  # TODO: implement
end
```

---

## เฉลย Exercises

### เฉลย Exercise 1

```elixir
defmodule ChatSystem.Supervisor do
  use DynamicSupervisor

  def start_link(opts) do
    DynamicSupervisor.start_link(__MODULE__, opts, name: __MODULE__)
  end

  @impl DynamicSupervisor
  def init(_opts) do
    DynamicSupervisor.init(strategy: :one_for_one)
  end

  def create_room(name) do
    spec = {ChatRoom.Server, name: name}
    case DynamicSupervisor.start_child(__MODULE__, spec) do
      {:ok, pid} -> {:ok, pid}
      {:error, {:already_started, pid}} -> {:ok, pid}
      error -> error
    end
  end

  def destroy_room(pid) do
    DynamicSupervisor.terminate_child(__MODULE__, pid)
  end

  def list_rooms do
    DynamicSupervisor.which_children(__MODULE__)
    |> Enum.map(fn {_, pid, _, _} -> pid end)
    |> Enum.filter(&is_pid/1)
  end
end

defmodule ChatRoom.Server do
  use GenServer

  def start_link(opts) do
    name = Keyword.fetch!(opts, :name)
    GenServer.start_link(__MODULE__, name, name: via(name))
  end

  def join(room_name, user) do
    GenServer.call(via(room_name), {:join, user})
  end

  def leave(room_name, user) do
    GenServer.cast(via(room_name), {:leave, user})
  end

  def send_message(room_name, from, text) do
    GenServer.cast(via(room_name), {:message, from, text})
  end

  def get_messages(room_name, last_n \\ 50) do
    GenServer.call(via(room_name), {:messages, last_n})
  end

  def get_users(room_name) do
    GenServer.call(via(room_name), :users)
  end

  defp via(name), do: {:via, Registry, {ChatRoom.Registry, name}}

  @impl GenServer
  def init(name) do
    IO.puts("Room '#{name}' started")
    {:ok, %{name: name, users: MapSet.new(), messages: []}}
  end

  @impl GenServer
  def handle_call({:join, user}, _from, state) do
    new_state = %{state | users: MapSet.put(state.users, user)}
    {:reply, {:ok, MapSet.to_list(new_state.users)}, new_state}
  end

  def handle_call({:messages, n}, _from, state) do
    {:reply, Enum.take(state.messages, n), state}
  end

  def handle_call(:users, _from, state) do
    {:reply, MapSet.to_list(state.users), state}
  end

  @impl GenServer
  def handle_cast({:leave, user}, state) do
    new_state = %{state | users: MapSet.delete(state.users, user)}
    {:noreply, new_state}
  end

  def handle_cast({:message, from, text}, state) do
    msg = %{from: from, text: text, at: DateTime.utc_now()}
    new_messages = [msg | state.messages] |> Enum.take(100)
    {:noreply, %{state | messages: new_messages}}
  end
end

# Application setup
defmodule ChatSystem.Application do
  use Application

  def start(_type, _args) do
    children = [
      {Registry, keys: :unique, name: ChatRoom.Registry},
      ChatSystem.Supervisor
    ]
    Supervisor.start_link(children, strategy: :one_for_one, name: ChatSystem.AppSupervisor)
  end
end

# Test
ChatSystem.Application.start(:normal, [])

{:ok, _room1} = ChatSystem.Supervisor.create_room("general")
{:ok, _room2} = ChatSystem.Supervisor.create_room("elixir")

ChatRoom.Server.join("general", "alice")
ChatRoom.Server.join("general", "bob")

ChatRoom.Server.send_message("general", "alice", "Hello!")
ChatRoom.Server.send_message("general", "bob", "Hi Alice!")

ChatRoom.Server.get_messages("general")
```

---

## สรุป

```
Supervisor:
├── use Supervisor + def init
├── Strategies: :one_for_one, :one_for_all, :rest_for_one
├── max_restarts + max_seconds
└── Supervisor.init(children, opts)

DynamicSupervisor:
├── use DynamicSupervisor
├── start_child(supervisor, spec)
├── terminate_child(supervisor, pid)
└── which_children(supervisor)

Child Spec:
├── {id, start, restart, shutdown, type}
├── restart: :permanent | :temporary | :transient
└── Supervisor.child_spec({Mod, args}, overrides)

OTP Tree:
Application
└── Top Supervisor
    ├── Core GenServers
    ├── Sub-Supervisor
    │   └── Worker pool
    └── DynamicSupervisor
        └── Dynamic workers
```

---

*ก่อนหน้า: [Part 16](part_16.md) | ต่อไป: [Part 18 - ETS: Erlang Term Storage](part_18.md)*
