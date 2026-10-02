# Part 40: Performance Optimization

## เป้าหมายการเรียนรู้

หลังจากจบ Part นี้ คุณจะสามารถ:
- Profile Elixir applications
- Optimize database queries
- ใช้ caching strategies
- Monitor production performance

---

## 1. Profiling ด้วย :timer และ :observer

```elixir
# วัดเวลา
{time_us, result} = :timer.tc(fn -> expensive_function() end)
IO.puts("Took #{time_us / 1000}ms")

# :observer - GUI profiler
:observer.start()

# Benchee - micro benchmarking
# mix.exs: {:benchee, "~> 1.0", only: :dev}

Benchee.run(%{
  "Enum.map + filter" => fn ->
    [1..10_000]
    |> Enum.flat_map(&Function.identity/1)
    |> Enum.map(&(&1 * 2))
    |> Enum.filter(&(&1 > 100))
  end,
  "Stream.map + filter" => fn ->
    1..10_000
    |> Stream.map(&(&1 * 2))
    |> Stream.filter(&(&1 > 100))
    |> Enum.to_list()
  end
}, memory_time: 2)
```

---

## 2. Database Query Optimization

```elixir
# ปัญหา N+1
users = Repo.all(User)
Enum.each(users, fn user ->
  # N queries!
  posts = Repo.all(from p in Post, where: p.user_id == ^user.id)
end)

# แก้ด้วย preload
users = Repo.all(from u in User, preload: [:posts])

# ดู queries ที่ส่ง ใน dev
# config/dev.exs
config :my_app, MyApp.Repo, log: :debug

# Explain query
import Ecto.Query

query = from u in User, where: u.active == true
Ecto.Adapters.SQL.explain(MyApp.Repo, :all, query)

# Index ที่ช่วย
# Migration
create index(:users, [:email])           # single column
create index(:users, [:role, :active])   # composite index
create index(:posts, [:user_id, :published, :inserted_at])  # covering

# Partial index
create index(:posts, [:inserted_at], where: "published = true",
  name: :posts_published_inserted_at_index)
```

---

## 3. Connection Pool Tuning

```elixir
# config/prod.exs
config :my_app, MyApp.Repo,
  pool_size: 20,           # จำนวน connections
  queue_target: 50,        # ms: warn ถ้า checkout queue > 50ms
  queue_interval: 1_000,   # ms: ระยะเวลา check interval
  timeout: 15_000,         # ms: query timeout
  connect_timeout: 5_000   # ms: connection timeout

# Rule of thumb:
# pool_size = (max_concurrency * 1.5) + overhead
# ถ้า 10 Dynos: pool_size = 15-20
```

---

## 4. Caching Strategies

```elixir
# 1. ETS Cache (in-process)
defmodule MyApp.Cache do
  use GenServer

  @table :app_cache
  @default_ttl 60_000

  def get_or_fetch(key, fetch_fn, ttl \\ @default_ttl) do
    case lookup(key) do
      {:ok, value} -> value
      :miss ->
        value = fetch_fn.()
        store(key, value, ttl)
        value
    end
  end

  defp lookup(key) do
    case :ets.lookup(@table, key) do
      [{^key, value, expires_at}] when expires_at > System.system_time(:millisecond) ->
        {:ok, value}
      _ ->
        :miss
    end
  end

  defp store(key, value, ttl) do
    expires_at = System.system_time(:millisecond) + ttl
    :ets.insert(@table, {key, value, expires_at})
  end
end

# 2. Cachex (feature-rich)
# {:cachex, "~> 3.6"}
Cachex.start_link(:my_cache, [])

Cachex.get_and_update!(:my_cache, "key", fn
  nil ->
    {:commit, fetch_from_db()}
  value ->
    {:ok, value}
end)

Cachex.put(:my_cache, "key", value, ttl: :timer.minutes(5))

# 3. Redis (Redix)
# {:redix, "~> 1.2"}
{:ok, conn} = Redix.start_link("redis://localhost:6379")
Redix.command!(conn, ["SET", "key", "value", "EX", "300"])
Redix.command!(conn, ["GET", "key"])
```

---

## 5. Process Pool ด้วย Poolboy

```elixir
# mix.exs
{:poolboy, "~> 1.5"}

# lib/my_app/worker_pool.ex
defmodule MyApp.WorkerPool do
  @pool_name :worker_pool

  def child_spec(_opts) do
    poolboy_config = [
      name: {:local, @pool_name},
      worker_module: MyApp.Worker,
      size: 10,        # initial pool size
      max_overflow: 5  # temporary workers
    ]

    :poolboy.child_spec(@pool_name, poolboy_config)
  end

  def transaction(fun) do
    :poolboy.transaction(@pool_name, fun)
  end

  def checkout do
    :poolboy.checkout(@pool_name)
  end

  def checkin(worker) do
    :poolboy.checkin(@pool_name, worker)
  end
end

# ใช้งาน
result = MyApp.WorkerPool.transaction(fn worker ->
  GenServer.call(worker, {:process, data})
end)
```

---

## 6. Telemetry และ Monitoring

```elixir
# mix.exs
{:telemetry, "~> 1.2"},
{:telemetry_metrics, "~> 0.6"},
{:telemetry_poller, "~> 1.0"},
{:phoenix_live_dashboard, "~> 0.8"}

# lib/my_app_web/telemetry.ex
defmodule MyAppWeb.Telemetry do
  use Supervisor
  import Telemetry.Metrics

  def start_link(arg) do
    Supervisor.start_link(__MODULE__, arg, name: __MODULE__)
  end

  @impl true
  def init(_arg) do
    children = [
      {:telemetry_poller,
       measurements: periodic_measurements(),
       period: 10_000},
    ]

    Supervisor.init(children, strategy: :one_for_one)
  end

  def metrics do
    [
      # Phoenix Metrics
      summary("phoenix.endpoint.stop.duration",
        unit: {:native, :millisecond}
      ),
      summary("phoenix.router_dispatch.stop.duration",
        tags: [:route],
        unit: {:native, :millisecond}
      ),

      # Ecto Metrics
      summary("my_app.repo.query.total_time",
        unit: {:native, :millisecond},
        description: "Total time for Ecto queries"
      ),
      summary("my_app.repo.query.queue_time",
        unit: {:native, :millisecond},
        description: "Time spent waiting for a connection"
      ),

      # VM Metrics
      last_value("vm.memory.total", unit: {:byte, :megabyte}),
      last_value("vm.total_run_queue_lengths.total"),
      last_value("vm.system_counts.process_count"),
    ]
  end

  defp periodic_measurements do
    [
      {MyAppWeb.Telemetry, :dispatch_worker_count, []}
    ]
  end

  def dispatch_worker_count do
    count = MyApp.WorkerPool |> DynamicSupervisor.which_children() |> length()
    :telemetry.execute([:my_app, :worker_pool, :count], %{count: count})
  end
end
```

---

## 7. Memory Optimization

```elixir
# Binary references - อย่าเก็บ large binaries ใน process state
# Bad
def handle_call(:get_data, _from, %{large_binary: binary} = state) do
  {:reply, binary, state}  # binary ถูก copy ทุกครั้ง
end

# Better - เก็บใน ETS
def handle_call(:get_data, _from, state) do
  data = :ets.lookup(:data_table, :current)
  {:reply, data, state}  # ไม่ copy
end

# Process hibernation เมื่อ idle นาน
def handle_info(:timeout, state) do
  {:noreply, state, :hibernate}  # ลด memory
end

# Large binary processing ด้วย Stream
def process_large_file(path) do
  path
  |> File.stream!([], 1024 * 1024)  # 1MB chunks
  |> Stream.map(&process_chunk/1)
  |> Stream.run()
end
```

---

## 8. Flame Graphs และ Profiling

```elixir
# :fprof - detailed function profiling
:fprof.apply(&slow_function/1, [args])
:fprof.profile()
:fprof.analyse(dest: 'fprof_output.txt')

# :eprof - process profiling
:eprof.start_profiling([self()])
# รัน code ที่ต้องการ profile
slow_operation()
:eprof.stop_profiling()
:eprof.analyse()

# recon_trace - tracing ใน production
# {:recon, "~> 2.5"}
:recon_trace.calls({MyApp.Module, :function, '_'}, 100)
:recon_trace.stop()
```

---

## แบบฝึกหัด

### Exercise 1: Optimize N+1 Query
หา N+1 queries ใน code ด้วยการ enable query logging แล้วแก้ด้วย preload

### Exercise 2: Cache Layer
เพิ่ม caching สำหรับ expensive queries:
- Cache user profile 5 นาที
- Invalidate เมื่อ profile update

---

## สรุป

```
Performance:
├── Profiling: :timer.tc, Benchee, :observer
├── Database: preload, indexes, EXPLAIN
├── Caching: ETS, Cachex, Redis
└── Monitoring: Telemetry, LiveDashboard

Database Optimization:
├── Avoid N+1: preload associations
├── Indexes on frequently queried columns
├── Partial indexes สำหรับ filtered queries
└── Connection pool tuning

Memory:
├── Binary references ไม่ copy ถ้าใช้ > 64 bytes
├── Process hibernation สำหรับ idle processes
└── Stream สำหรับ large data
```

---

*ก่อนหน้า: [Part 39](part_39.md) | ต่อไป: [Part 41 - Security](part_41.md)*
