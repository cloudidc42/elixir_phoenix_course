# Part 63: Advanced OTP Patterns

## เป้าหมายการเรียนรู้
- ใช้ Process Registry patterns อย่างมีประสิทธิภาพ
- Swarm library สำหรับ distributed processes
- Circuit Breaker pattern ด้วย Fuse
- Poolboy process pool แบบ advanced
- Backoff strategies สำหรับ retry
- ออกแบบ supervision tree อย่างถูกต้อง
- Graceful shutdown

---

## 1. Process Registry Patterns

Registry ใน Elixir ช่วยให้ค้นหา process ด้วยชื่อหรือ key ได้อย่างมีประสิทธิภาพ

```elixir
# lib/my_app/application.ex - ตั้งค่า Registry
children = [
  # Registry ที่รองรับ multiple processes ต่อ key
  {Registry, keys: :unique, name: MyApp.Registry},

  # Duplicate registry สำหรับ pub/sub
  {Registry, keys: :duplicate, name: MyApp.PubSubRegistry},

  # ...
]
```

```elixir
# lib/my_app/user_session.ex
defmodule MyApp.UserSession do
  @moduledoc """
  Process ที่ represent user session
  ใช้ Registry เพื่อ look up ด้วย user_id
  """

  use GenServer

  def start_link(user_id) do
    # ลงทะเบียน process ด้วย user_id เป็น key
    GenServer.start_link(
      __MODULE__,
      user_id,
      name: via_tuple(user_id)
    )
  end

  # Helper สำหรับสร้าง via tuple
  def via_tuple(user_id) do
    {:via, Registry, {MyApp.Registry, {:user_session, user_id}}}
  end

  # หา session โดยใช้ user_id
  def find(user_id) do
    case Registry.lookup(MyApp.Registry, {:user_session, user_id}) do
      [{pid, _value}] -> {:ok, pid}
      [] -> {:error, :not_found}
    end
  end

  # หรือ start ถ้ายังไม่มี
  def get_or_start(user_id) do
    case find(user_id) do
      {:ok, pid} ->
        {:ok, pid}

      {:error, :not_found} ->
        DynamicSupervisor.start_child(
          MyApp.SessionSupervisor,
          {__MODULE__, user_id}
        )
    end
  end

  def update_activity(user_id) do
    case find(user_id) do
      {:ok, pid} -> GenServer.cast(pid, :update_activity)
      error -> error
    end
  end

  # --- Server callbacks ---

  @impl true
  def init(user_id) do
    # Set idle timeout - terminate เมื่อ idle นานเกินไป
    {:ok, %{
      user_id: user_id,
      last_activity: DateTime.utc_now(),
      data: %{}
    }, {:continue, :setup}}
  end

  @impl true
  def handle_continue(:setup, state) do
    # โหลดข้อมูล session จาก database
    schedule_idle_check()
    {:noreply, state}
  end

  @impl true
  def handle_cast(:update_activity, state) do
    {:noreply, %{state | last_activity: DateTime.utc_now()}}
  end

  @impl true
  def handle_info(:check_idle, state) do
    idle_seconds = DateTime.diff(DateTime.utc_now(), state.last_activity)

    if idle_seconds > 1800 do  # 30 นาที
      {:stop, :normal, state}
    else
      schedule_idle_check()
      {:noreply, state}
    end
  end

  defp schedule_idle_check do
    Process.send_after(self(), :check_idle, 60_000)
  end
end
```

```elixir
# ใช้ Registry สำหรับ pub/sub pattern
defmodule MyApp.EventBus do
  @registry MyApp.PubSubRegistry

  def subscribe(topic) do
    Registry.register(@registry, topic, [])
  end

  def unsubscribe(topic) do
    Registry.unregister(@registry, topic)
  end

  def publish(topic, message) do
    Registry.dispatch(@registry, topic, fn entries ->
      Enum.each(entries, fn {pid, _} ->
        send(pid, {:event, topic, message})
      end)
    end)
  end

  # Dispatch ด้วย callback
  def broadcast_to_all(message) do
    Registry.dispatch(@registry, :all, fn entries ->
      Enum.each(entries, fn {pid, _} ->
        GenServer.cast(pid, {:broadcast, message})
      end)
    end)
  end
end
```

---

## 2. Swarm สำหรับ Distributed Processes

Swarm ช่วยจัดการ processes ข้ามหลาย nodes ใน cluster

```elixir
# mix.exs
{:swarm, "~> 3.4"}
```

```elixir
# lib/my_app/distributed_worker.ex
defmodule MyApp.DistributedWorker do
  @moduledoc """
  Worker ที่ทำงานแบบ distributed ด้วย Swarm
  Process จะถูก distribute ข้าม cluster nodes
  """

  use GenServer

  def start_link({name, args}) do
    GenServer.start_link(__MODULE__, {name, args}, name: {:via, :swarm, name})
  end

  # สร้าง worker บน node ที่เหมาะสม
  def create_worker(name, args) do
    case Swarm.whereis_name(name) do
      :undefined ->
        # Worker ยังไม่มี - สร้างใหม่
        {:ok, pid} = Swarm.register_name(
          name,
          MyApp.WorkerSupervisor,
          :start_child,
          [{name, args}]
        )
        {:ok, pid}

      pid ->
        # Worker มีอยู่แล้ว
        {:ok, pid}
    end
  end

  def call(name, message) do
    case Swarm.whereis_name(name) do
      :undefined -> {:error, :not_found}
      pid -> GenServer.call(pid, message)
    end
  end

  # Swarm callbacks สำหรับ handoff ระหว่าง nodes

  # เรียกเมื่อ node join cluster และ process ควรย้ายไปอยู่ที่ node ใหม่
  def handle_call({:swarm, :begin_handoff}, _from, state) do
    {:reply, {:resume, state}, state}
  end

  # เรียกเมื่อรับ state จาก node เดิม
  def handle_cast({:swarm, :end_handoff, state}, _state) do
    {:noreply, state}
  end

  # เรียกเมื่อ node leave cluster
  def handle_cast({:swarm, :resolve_conflict, _other_state}, state) do
    {:noreply, state}
  end

  # Notify ว่า process กำลังจะถูก shutdown เพราะ node left
  def handle_info({:swarm, :die}, state) do
    {:stop, :shutdown, state}
  end

  @impl true
  def init({name, args}) do
    {:ok, %{name: name, args: args, started_at: DateTime.utc_now()}}
  end

  @impl true
  def handle_call(:get_state, _from, state) do
    {:reply, state, state}
  end
end
```

---

## 3. Circuit Breaker Pattern ด้วย Fuse

Circuit Breaker ป้องกันการเรียก service ที่ล้มเหลวซ้ำๆ โดยเปิด "circuit" เมื่อเกิด error เยอะเกิน

```
Circuit Breaker States:
                           
  ┌─────────┐  failures++   ┌──────────┐
  │ CLOSED  │──────────────►│   OPEN   │
  │(Normal) │               │(Blocked) │
  └─────────┘               └────┬─────┘
       ▲                         │ timeout
       │                         ▼
       │                   ┌──────────┐
       └───────────────────│HALF-OPEN │
         success            │(Testing) │
                            └──────────┘
```

```elixir
# mix.exs
{:fuse, "~> 2.4"}
```

```elixir
# lib/my_app/circuit_breaker.ex
defmodule MyApp.CircuitBreaker do
  @moduledoc """
  Wrapper สำหรับ Circuit Breaker ด้วย Fuse library
  """

  require Logger

  # ตั้งค่า fuses
  @fuses %{
    payment_service: {
      {:standard, 5, 10_000},  # 5 failures ใน 10 วินาที → open
      {:reset, 30_000}          # reset หลัง 30 วินาที
    },
    email_service: {
      {:standard, 3, 5_000},
      {:reset, 60_000}
    },
    external_api: {
      {:standard, 10, 30_000},
      {:reset, 120_000}
    }
  }

  def setup do
    Enum.each(@fuses, fn {name, {strategy, reset}} ->
      :fuse.install(name, {strategy, reset})
    end)
  end

  # เรียกใช้ function ผ่าน circuit breaker
  def call(fuse_name, fun) when is_function(fun, 0) do
    case :fuse.ask(fuse_name, :sync) do
      :ok ->
        # Circuit ปิด - เรียกได้ปกติ
        try do
          result = fun.()
          {:ok, result}
        rescue
          error ->
            # นับ failure
            :fuse.melt(fuse_name)
            Logger.warning("Circuit #{fuse_name} - failure: #{inspect(error)}")
            {:error, {:service_error, error}}
        catch
          :exit, reason ->
            :fuse.melt(fuse_name)
            {:error, {:exit, reason}}
        end

      :blown ->
        # Circuit เปิด - reject request ทันที
        Logger.warning("Circuit #{fuse_name} is OPEN - request rejected")
        {:error, :circuit_open}
    end
  end

  # Check state
  def state(fuse_name) do
    case :fuse.ask(fuse_name, :sync) do
      :ok -> :closed
      :blown -> :open
    end
  end

  # Reset circuit manually
  def reset(fuse_name) do
    :fuse.reset(fuse_name)
    Logger.info("Circuit #{fuse_name} manually reset")
  end
end
```

```elixir
# ตัวอย่างการใช้งาน
defmodule MyApp.PaymentService do
  alias MyApp.CircuitBreaker

  def charge_customer(customer_id, amount) do
    CircuitBreaker.call(:payment_service, fn ->
      ExternalPaymentGateway.charge(%{
        customer_id: customer_id,
        amount: amount
      })
    end)
  end

  def refund(transaction_id, amount) do
    case CircuitBreaker.call(:payment_service, fn ->
      ExternalPaymentGateway.refund(transaction_id, amount)
    end) do
      {:ok, result} ->
        {:ok, result}

      {:error, :circuit_open} ->
        # Circuit เปิด - queue refund ไว้ทำในภายหลัง
        MyApp.RefundQueue.enqueue({transaction_id, amount})
        {:ok, :queued}

      {:error, reason} ->
        {:error, reason}
    end
  end
end
```

---

## 4. Poolboy Advanced - Process Pool Management

```elixir
# mix.exs
{:poolboy, "~> 1.5"}
```

```elixir
# lib/my_app/db_pool.ex
defmodule MyApp.DbPool do
  @moduledoc """
  Advanced Poolboy usage สำหรับ database connections
  """

  # Pool configuration
  @pool_config [
    name: {:local, :db_pool},
    worker_module: MyApp.DbWorker,
    size: 10,          # จำนวน workers ปกติ
    max_overflow: 5    # workers เพิ่มเติมเมื่อ pool เต็ม
  ]

  def child_spec(_opts) do
    :poolboy.child_spec(:db_pool, @pool_config, [])
  end

  # ใช้ transaction สำหรับ checkout/checkin อัตโนมัติ
  def query(sql, params \\ [], timeout \\ 5_000) do
    :poolboy.transaction(:db_pool, fn worker ->
      MyApp.DbWorker.query(worker, sql, params)
    end, timeout)
  end

  # ใช้ checkout/checkin แบบ manual สำหรับ long operations
  def checkout(timeout \\ 5_000) do
    worker = :poolboy.checkout(:db_pool, true, timeout)
    {:ok, worker}
  end

  def checkin(worker) do
    :poolboy.checkin(:db_pool, worker)
  end

  # Pool statistics
  def stats do
    status = :poolboy.status(:db_pool)
    %{
      state: elem(status, 0),
      workers: elem(status, 1),
      overflow: elem(status, 2),
      waiting: elem(status, 3)
    }
  end
end
```

```elixir
# lib/my_app/db_worker.ex
defmodule MyApp.DbWorker do
  use GenServer

  def start_link(_args) do
    GenServer.start_link(__MODULE__, nil)
  end

  def query(worker, sql, params) do
    GenServer.call(worker, {:query, sql, params}, 10_000)
  end

  @impl true
  def init(_args) do
    # สร้าง database connection
    {:ok, conn} = Postgrex.start_link(
      hostname: "localhost",
      username: "postgres",
      password: "postgres",
      database: "my_app"
    )
    {:ok, %{conn: conn}}
  end

  @impl true
  def handle_call({:query, sql, params}, _from, state) do
    result = Postgrex.query(state.conn, sql, params)
    {:reply, result, state}
  end

  @impl true
  def terminate(_reason, state) do
    GenServer.stop(state.conn)
    :ok
  end
end
```

```elixir
# Multi-pool strategy: pools แยกกันสำหรับ read/write
defmodule MyApp.ReadWritePool do
  def query_read(sql, params) do
    :poolboy.transaction(:read_pool, fn worker ->
      MyApp.DbWorker.query(worker, sql, params)
    end)
  end

  def query_write(sql, params) do
    :poolboy.transaction(:write_pool, fn worker ->
      MyApp.DbWorker.query(worker, sql, params)
    end)
  end

  # Child specs สำหรับ Supervisor
  def child_specs do
    [
      :poolboy.child_spec(:read_pool,
        [name: {:local, :read_pool}, worker_module: MyApp.DbWorker, size: 20, max_overflow: 10],
        [read_only: true]
      ),
      :poolboy.child_spec(:write_pool,
        [name: {:local, :write_pool}, worker_module: MyApp.DbWorker, size: 5, max_overflow: 2],
        [read_only: false]
      )
    ]
  end
end
```

---

## 5. Backoff Strategies

```elixir
# lib/my_app/backoff.ex
defmodule MyApp.Backoff do
  @moduledoc """
  Backoff strategies สำหรับ retry operations
  """

  # Exponential backoff: 1s, 2s, 4s, 8s, 16s, ...
  def exponential(attempt, opts \\ []) do
    base = Keyword.get(opts, :base, 1_000)
    max = Keyword.get(opts, :max, 60_000)
    jitter = Keyword.get(opts, :jitter, true)

    delay = min(base * :math.pow(2, attempt - 1), max) |> round()

    if jitter do
      # เพิ่ม randomness เพื่อป้องกัน thundering herd
      jitter_range = div(delay, 4)
      delay + :rand.uniform(jitter_range)
    else
      delay
    end
  end

  # Linear backoff: 1s, 2s, 3s, 4s, ...
  def linear(attempt, opts \\ []) do
    step = Keyword.get(opts, :step, 1_000)
    max = Keyword.get(opts, :max, 30_000)
    min(attempt * step, max)
  end

  # Fibonacci backoff: 1, 1, 2, 3, 5, 8, 13, ...
  def fibonacci(attempt, opts \\ []) do
    base = Keyword.get(opts, :base, 1_000)
    max = Keyword.get(opts, :max, 60_000)
    min(fib(attempt) * base, max)
  end

  defp fib(0), do: 1
  defp fib(1), do: 1
  defp fib(n), do: fib(n - 1) + fib(n - 2)
end
```

```elixir
# lib/my_app/retry.ex
defmodule MyApp.Retry do
  @moduledoc """
  Generic retry mechanism ที่ใช้ร่วมกับ backoff strategies
  """

  alias MyApp.Backoff

  @default_opts [
    max_attempts: 5,
    backoff: :exponential,
    on_retry: nil
  ]

  def with_retry(fun, opts \\ []) do
    opts = Keyword.merge(@default_opts, opts)
    do_retry(fun, 1, opts)
  end

  defp do_retry(fun, attempt, opts) do
    case fun.() do
      {:ok, result} ->
        {:ok, result}

      {:error, reason} when attempt < opts[:max_attempts] ->
        delay = calculate_delay(attempt, opts[:backoff])

        if notify = opts[:on_retry] do
          notify.({attempt, reason, delay})
        end

        require Logger
        Logger.warning("Attempt #{attempt} failed: #{inspect(reason)}. Retrying in #{delay}ms")
        Process.sleep(delay)
        do_retry(fun, attempt + 1, opts)

      {:error, _reason} = error ->
        error
    end
  rescue
    exception when attempt < opts[:max_attempts] ->
      delay = calculate_delay(attempt, opts[:backoff])
      Process.sleep(delay)
      do_retry(fun, attempt + 1, opts)

    exception ->
      {:error, {:exception, exception}}
  end

  defp calculate_delay(attempt, :exponential) do
    Backoff.exponential(attempt)
  end
  defp calculate_delay(attempt, :linear) do
    Backoff.linear(attempt)
  end
  defp calculate_delay(attempt, :fibonacci) do
    Backoff.fibonacci(attempt)
  end
  defp calculate_delay(attempt, delay_fn) when is_function(delay_fn, 1) do
    delay_fn.(attempt)
  end
end

# ใช้งาน
result = MyApp.Retry.with_retry(
  fn -> ExternalAPI.fetch_data() end,
  max_attempts: 3,
  backoff: :exponential,
  on_retry: fn {attempt, reason, delay} ->
    Logger.warning("Retry #{attempt} after #{delay}ms due to: #{inspect(reason)}")
  end
)
```

---

## 6. Supervision Tree Design

```elixir
# lib/my_app/application.ex
defmodule MyApp.Application do
  use Application

  @impl true
  def start(_type, _args) do
    children = [
      # ─── Infrastructure ───────────────────────────────
      # PubSub ต้องมาก่อน
      {Phoenix.PubSub, name: MyApp.PubSub},

      # Registries ต้องมาก่อน processes ที่ใช้มัน
      {Registry, keys: :unique, name: MyApp.Registry},
      {Registry, keys: :duplicate, name: MyApp.PubSubRegistry},

      # ─── Database Layer ───────────────────────────────
      MyApp.Repo,
      MyApp.ReadWritePool,  # เพิ่ม pool specs

      # ─── Core Services ────────────────────────────────
      MyApp.CircuitBreaker.Supervisor,

      # ─── Dynamic Supervisors ──────────────────────────
      # ใช้สำหรับ create processes on demand
      {DynamicSupervisor, name: MyApp.SessionSupervisor, strategy: :one_for_one},
      {DynamicSupervisor, name: MyApp.WorkerSupervisor, strategy: :one_for_one},

      # ─── Application Services ─────────────────────────
      MyApp.JobQueue,
      MyApp.NotificationService,

      # ─── Web Layer ────────────────────────────────────
      MyAppWeb.Endpoint,
      MyAppWeb.Telemetry,
    ]

    opts = [strategy: :one_for_one, name: MyApp.Supervisor]
    Supervisor.start_link(children, opts)
  end
end
```

```elixir
# lib/my_app/circuit_breaker_supervisor.ex
defmodule MyApp.CircuitBreaker.Supervisor do
  @moduledoc """
  Supervisor ที่จัดการ Fuse circuit breakers
  """

  use Supervisor

  def start_link(_opts) do
    Supervisor.start_link(__MODULE__, [], name: __MODULE__)
  end

  @impl true
  def init(_opts) do
    # Setup circuit breakers
    MyApp.CircuitBreaker.setup()

    # Monitor circuit breaker status
    children = [
      MyApp.CircuitBreaker.Monitor
    ]

    Supervisor.init(children, strategy: :one_for_one)
  end
end

defmodule MyApp.CircuitBreaker.Monitor do
  use GenServer
  require Logger

  def start_link(_opts) do
    GenServer.start_link(__MODULE__, [], name: __MODULE__)
  end

  @impl true
  def init(_opts) do
    Process.send_after(self(), :check_circuits, 5_000)
    {:ok, %{}}
  end

  @impl true
  def handle_info(:check_circuits, state) do
    # Log circuit states ทุก 5 วินาที
    Enum.each([:payment_service, :email_service, :external_api], fn fuse ->
      status = MyApp.CircuitBreaker.state(fuse)
      if status == :open do
        Logger.warning("Circuit #{fuse} is OPEN")
        # ส่ง alert ไปยัง monitoring system
        MyApp.Monitoring.alert("circuit_open", %{fuse: fuse})
      end
    end)

    Process.send_after(self(), :check_circuits, 5_000)
    {:noreply, state}
  end
end
```

---

## 7. Graceful Shutdown

```elixir
# lib/my_app/graceful_shutdown.ex
defmodule MyApp.GracefulShutdown do
  @moduledoc """
  จัดการ graceful shutdown เพื่อให้ in-flight requests เสร็จก่อน terminate
  """

  use GenServer
  require Logger

  @shutdown_timeout 30_000  # 30 วินาที

  def start_link(_opts) do
    GenServer.start_link(__MODULE__, [], name: __MODULE__)
  end

  def initiate do
    GenServer.cast(__MODULE__, :initiate)
  end

  @impl true
  def init(_opts) do
    # Trap exit signals
    Process.flag(:trap_exit, true)

    {:ok, %{
      shutting_down: false,
      in_flight_requests: 0,
      shutdown_ref: nil
    }}
  end

  @impl true
  def handle_cast(:initiate, state) do
    Logger.info("Graceful shutdown initiated...")

    # บอก load balancer ว่าไม่รับ requests ใหม่
    stop_accepting_new_requests()

    if state.in_flight_requests == 0 do
      Logger.info("No in-flight requests, shutting down immediately")
      System.stop(0)
      {:noreply, %{state | shutting_down: true}}
    else
      Logger.info("Waiting for #{state.in_flight_requests} in-flight requests...")
      # Set timeout สำหรับ force shutdown
      ref = Process.send_after(self(), :force_shutdown, @shutdown_timeout)
      {:noreply, %{state | shutting_down: true, shutdown_ref: ref}}
    end
  end

  # Track in-flight requests
  @impl true
  def handle_cast(:request_started, state) do
    {:noreply, %{state | in_flight_requests: state.in_flight_requests + 1}}
  end

  @impl true
  def handle_cast(:request_completed, state) do
    new_count = state.in_flight_requests - 1
    new_state = %{state | in_flight_requests: new_count}

    if state.shutting_down && new_count == 0 do
      Logger.info("All requests completed, shutting down")
      if state.shutdown_ref, do: Process.cancel_timer(state.shutdown_ref)
      System.stop(0)
    end

    {:noreply, new_state}
  end

  @impl true
  def handle_info(:force_shutdown, state) do
    Logger.warning("Force shutdown after timeout with #{state.in_flight_requests} in-flight requests")
    System.stop(1)
    {:noreply, state}
  end

  # SIGTERM handler
  @impl true
  def handle_info({:signal, :sigterm}, state) do
    Logger.info("Received SIGTERM")
    handle_cast(:initiate, state)
  end

  defp stop_accepting_new_requests do
    # บอก Phoenix endpoint ให้หยุดรับ requests ใหม่
    Phoenix.PubSub.broadcast(MyApp.PubSub, "system", {:shutdown_initiated})
  end
end
```

```elixir
# lib/my_app_web/endpoint.ex - Handle shutdown ใน Plug
defmodule MyAppWeb.ShutdownPlug do
  import Plug.Conn

  def init(opts), do: opts

  def call(conn, _opts) do
    if MyApp.GracefulShutdown.shutting_down?() do
      conn
      |> put_resp_header("connection", "close")
      |> put_status(503)
      |> Phoenix.Controller.json(%{error: "Service shutting down"})
      |> halt()
    else
      # Track request
      MyApp.GracefulShutdown.track_request_start()

      conn
      |> register_before_send(fn conn ->
        MyApp.GracefulShutdown.track_request_end()
        conn
      end)
    end
  end
end
```

---

## 8. Process Hibernation สำหรับ Memory Efficiency

```elixir
defmodule MyApp.HibernatingWorker do
  @moduledoc """
  Worker ที่ hibernate เมื่อ idle เพื่อลด memory usage
  """

  use GenServer

  @idle_timeout 5_000  # hibernate หลัง 5 วินาที idle

  def start_link(args) do
    GenServer.start_link(__MODULE__, args)
  end

  @impl true
  def init(args) do
    {:ok, %{args: args, last_activity: :erlang.monotonic_time()},
     @idle_timeout}
  end

  @impl true
  def handle_call(msg, _from, state) do
    result = process_message(msg, state)
    # Reply และ reset timeout
    {:reply, result, %{state | last_activity: :erlang.monotonic_time()},
     @idle_timeout}
  end

  # Timeout เกิดขึ้น → hibernate
  @impl true
  def handle_info(:timeout, state) do
    {:noreply, state, :hibernate}
  end

  # หลัง hibernate, process จะ wake up เมื่อมี message ใหม่
  @impl true
  def handle_call(_msg, _from, state) do
    # ตอนแรกที่ wake up จาก hibernate
    {:reply, :woken_up, state, @idle_timeout}
  end

  defp process_message(msg, _state) do
    msg
  end
end
```

---

## สรุป

```
Advanced OTP Patterns Summary:

┌─────────────────────────────────────────────┐
│              Supervision Tree                │
│  ┌──────────────────────────────────────┐   │
│  │           Application               │   │
│  │  ┌───────────┐  ┌───────────────┐   │   │
│  │  │ Registry  │  │ DynamicSupv   │   │   │
│  │  └───────────┘  └───────────────┘   │   │
│  │  ┌──────────────────────────────┐   │   │
│  │  │     Service Supervisors      │   │   │
│  │  │  ┌──────┐  ┌──────────────┐ │   │   │
│  │  │  │Fuse  │  │   Poolboy    │ │   │   │
│  │  │  │(CB)  │  │  (Workers)   │ │   │   │
│  │  │  └──────┘  └──────────────┘ │   │   │
│  │  └──────────────────────────────┘   │   │
│  └──────────────────────────────────────┘   │
└─────────────────────────────────────────────┘

Pattern                Use When
────────────────────────────────────────────
Registry             ค้นหา process ด้วย key
Swarm                processes ข้าม cluster
Circuit Breaker      เรียก external services
Poolboy              จัดการ resource pools
Backoff              retry with smart delays
Graceful Shutdown    zero-downtime deployment
Hibernation          idle processes, save RAM

Key Rules:
1. Registry ตั้งค่าก่อน processes ที่ใช้
2. DynamicSupervisor สำหรับ create on demand
3. Circuit Breaker ทุก external service call
4. Exponential backoff + jitter ป้องกัน thundering herd
5. Trap :EXIT ใน processes ที่ต้องการ cleanup
6. Hibernate processes ที่ idle นาน
```

---

*ก่อนหน้า: [Part 62 - Phoenix Livebook](part_62.md) | ต่อไป: [Part 64 - Production Deployment](part_64.md)*
