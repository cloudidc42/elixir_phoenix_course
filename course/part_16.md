# Part 16: GenServer พื้นฐาน

## เป้าหมายการเรียนรู้

หลังจากจบ Part นี้ คุณจะสามารถ:
- เข้าใจ Client/Server pattern ใน GenServer
- Implement callbacks ทุกตัว: `init`, `handle_call`, `handle_cast`, `handle_info`
- จัดการ State ใน GenServer
- สร้าง Todo List Server และ Rate Limiter จากโค้ดจริง

---

## 1. GenServer คืออะไร?

GenServer = Generic Server - process ที่มี state และรับ messages

```
Client                  GenServer Process
  │                           │
  │──call({:add, item})──────>│ handle_call
  │<─────{:reply, :ok}────────│
  │                           │
  │──cast({:delete, id})─────>│ handle_cast (no reply)
  │                           │
  │  (external event)────────>│ handle_info
  │                           │
```

---

## 2. GenServer Callbacks

```elixir
defmodule MyServer do
  use GenServer

  # 1. init/1 - เรียกครั้งแรกเมื่อ start
  @impl GenServer
  def init(args) do
    {:ok, initial_state}
    # หรือ {:ok, state, timeout}
    # หรือ {:ok, state, {:continue, :setup}}
    # หรือ {:stop, reason}
  end

  # 2. handle_call/3 - synchronous request (รอ reply)
  @impl GenServer
  def handle_call(request, from, state) do
    {:reply, response, new_state}
    # หรือ {:reply, response, new_state, timeout}
    # หรือ {:noreply, new_state}  # reply later
    # หรือ {:stop, reason, response, new_state}
  end

  # 3. handle_cast/2 - asynchronous request (ไม่รอ reply)
  @impl GenServer
  def handle_cast(request, state) do
    {:noreply, new_state}
    # หรือ {:noreply, new_state, timeout}
    # หรือ {:stop, reason, new_state}
  end

  # 4. handle_info/2 - messages อื่นๆ (timer, process messages)
  @impl GenServer
  def handle_info(message, state) do
    {:noreply, new_state}
    # หรือ {:stop, reason, new_state}
  end

  # 5. terminate/2 - เรียกก่อน process จบ
  @impl GenServer
  def terminate(reason, state) do
    # cleanup
    :ok
  end

  # 6. handle_continue/2 - ทำงานต่อหลัง init/handle_call
  @impl GenServer
  def handle_continue(:setup, state) do
    # do expensive setup
    {:noreply, new_state}
  end
end
```

---

## 3. ตัวอย่างพื้นฐาน: Stack Server

```elixir
defmodule StackServer do
  use GenServer

  ## Client API

  def start_link(opts \\ []) do
    initial = Keyword.get(opts, :initial, [])
    GenServer.start_link(__MODULE__, initial, name: __MODULE__)
  end

  def push(item) do
    GenServer.cast(__MODULE__, {:push, item})
  end

  def pop do
    GenServer.call(__MODULE__, :pop)
  end

  def peek do
    GenServer.call(__MODULE__, :peek)
  end

  def size do
    GenServer.call(__MODULE__, :size)
  end

  def empty? do
    GenServer.call(__MODULE__, :empty?)
  end

  def to_list do
    GenServer.call(__MODULE__, :to_list)
  end

  def clear do
    GenServer.cast(__MODULE__, :clear)
  end

  ## Server Callbacks

  @impl GenServer
  def init(initial_items) do
    {:ok, initial_items}
  end

  @impl GenServer
  def handle_call(:pop, _from, []) do
    {:reply, {:error, :empty}, []}
  end

  def handle_call(:pop, _from, [head | tail]) do
    {:reply, {:ok, head}, tail}
  end

  def handle_call(:peek, _from, []) do
    {:reply, {:error, :empty}, []}
  end

  def handle_call(:peek, _from, [head | _] = state) do
    {:reply, {:ok, head}, state}
  end

  def handle_call(:size, _from, state) do
    {:reply, length(state), state}
  end

  def handle_call(:empty?, _from, state) do
    {:reply, state == [], state}
  end

  def handle_call(:to_list, _from, state) do
    {:reply, state, state}
  end

  @impl GenServer
  def handle_cast({:push, item}, state) do
    {:noreply, [item | state]}
  end

  def handle_cast(:clear, _state) do
    {:noreply, []}
  end
end

# ใช้งาน
{:ok, _pid} = StackServer.start_link()

StackServer.push(1)
StackServer.push(2)
StackServer.push(3)

StackServer.size()     # 3
StackServer.peek()     # {:ok, 3}
StackServer.pop()      # {:ok, 3}
StackServer.to_list()  # [2, 1]
```

---

## 4. ตัวอย่างจริง: Todo List Server

```elixir
defmodule TodoServer do
  use GenServer

  defmodule Todo do
    defstruct [:id, :title, :completed, :created_at, :updated_at]

    def new(id, title) do
      now = DateTime.utc_now()
      %__MODULE__{
        id: id,
        title: title,
        completed: false,
        created_at: now,
        updated_at: now
      }
    end
  end

  ## Client API

  def start_link(name \\ __MODULE__) do
    GenServer.start_link(__MODULE__, nil, name: name)
  end

  def add(server \\ __MODULE__, title) do
    GenServer.call(server, {:add, title})
  end

  def complete(server \\ __MODULE__, id) do
    GenServer.call(server, {:complete, id})
  end

  def delete(server \\ __MODULE__, id) do
    GenServer.call(server, {:delete, id})
  end

  def update_title(server \\ __MODULE__, id, new_title) do
    GenServer.call(server, {:update_title, id, new_title})
  end

  def list(server \\ __MODULE__, filter \\ :all) do
    GenServer.call(server, {:list, filter})
  end

  def get(server \\ __MODULE__, id) do
    GenServer.call(server, {:get, id})
  end

  def stats(server \\ __MODULE__) do
    GenServer.call(server, :stats)
  end

  def clear_completed(server \\ __MODULE__) do
    GenServer.cast(server, :clear_completed)
  end

  ## Server Callbacks

  defstruct todos: %{}, next_id: 1

  @impl GenServer
  def init(_args) do
    {:ok, %__MODULE__{}}
  end

  @impl GenServer
  def handle_call({:add, title}, _from, state) when byte_size(title) > 0 do
    todo = Todo.new(state.next_id, title)
    new_state = %{state |
      todos: Map.put(state.todos, state.next_id, todo),
      next_id: state.next_id + 1
    }
    {:reply, {:ok, todo}, new_state}
  end

  def handle_call({:add, _title}, _from, state) do
    {:reply, {:error, :empty_title}, state}
  end

  def handle_call({:complete, id}, _from, state) do
    case Map.get(state.todos, id) do
      nil ->
        {:reply, {:error, :not_found}, state}
      todo ->
        updated = %{todo | completed: true, updated_at: DateTime.utc_now()}
        new_state = %{state | todos: Map.put(state.todos, id, updated)}
        {:reply, {:ok, updated}, new_state}
    end
  end

  def handle_call({:delete, id}, _from, state) do
    case Map.get(state.todos, id) do
      nil ->
        {:reply, {:error, :not_found}, state}
      todo ->
        new_state = %{state | todos: Map.delete(state.todos, id)}
        {:reply, {:ok, todo}, new_state}
    end
  end

  def handle_call({:update_title, id, new_title}, _from, state) do
    case Map.get(state.todos, id) do
      nil ->
        {:reply, {:error, :not_found}, state}
      todo ->
        updated = %{todo | title: new_title, updated_at: DateTime.utc_now()}
        new_state = %{state | todos: Map.put(state.todos, id, updated)}
        {:reply, {:ok, updated}, new_state}
    end
  end

  def handle_call({:list, :all}, _from, state) do
    todos = Map.values(state.todos) |> Enum.sort_by(& &1.id)
    {:reply, todos, state}
  end

  def handle_call({:list, :pending}, _from, state) do
    todos = state.todos
      |> Map.values()
      |> Enum.filter(fn t -> not t.completed end)
      |> Enum.sort_by(& &1.id)
    {:reply, todos, state}
  end

  def handle_call({:list, :completed}, _from, state) do
    todos = state.todos
      |> Map.values()
      |> Enum.filter(fn t -> t.completed end)
      |> Enum.sort_by(& &1.id)
    {:reply, todos, state}
  end

  def handle_call({:get, id}, _from, state) do
    result = case Map.get(state.todos, id) do
      nil -> {:error, :not_found}
      todo -> {:ok, todo}
    end
    {:reply, result, state}
  end

  def handle_call(:stats, _from, state) do
    all = Map.values(state.todos)
    stats = %{
      total: length(all),
      completed: Enum.count(all, & &1.completed),
      pending: Enum.count(all, fn t -> not t.completed end)
    }
    {:reply, stats, state}
  end

  @impl GenServer
  def handle_cast(:clear_completed, state) do
    new_todos = Map.filter(state.todos, fn {_, todo} -> not todo.completed end)
    {:noreply, %{state | todos: new_todos}}
  end
end

# ใช้งาน
{:ok, _} = TodoServer.start_link()

{:ok, t1} = TodoServer.add("Learn Elixir")
{:ok, t2} = TodoServer.add("Build a Phoenix app")
{:ok, t3} = TodoServer.add("Deploy to production")

TodoServer.list()
# [%Todo{id: 1, title: "Learn Elixir"}, ...]

TodoServer.complete(t1.id)
TodoServer.complete(t2.id)

TodoServer.stats()
# %{total: 3, completed: 2, pending: 1}

TodoServer.list(:pending)
# [%Todo{id: 3, title: "Deploy to production"}]

TodoServer.clear_completed()
TodoServer.stats()
# %{total: 1, completed: 0, pending: 1}
```

---

## 5. ตัวอย่างจริง: Rate Limiter GenServer

```elixir
defmodule RateLimiter do
  @moduledoc """
  Rate limiter ที่ใช้ sliding window algorithm
  """
  use GenServer

  ## Client API

  def start_link(opts \\ []) do
    name = Keyword.get(opts, :name, __MODULE__)
    config = %{
      max_requests: Keyword.get(opts, :max_requests, 10),
      window_ms: Keyword.get(opts, :window_ms, 60_000)
    }
    GenServer.start_link(__MODULE__, config, name: name)
  end

  @doc "ตรวจสอบว่าอนุญาตให้ทำ request ไหม"
  def check(server \\ __MODULE__, key) do
    GenServer.call(server, {:check, key})
  end

  @doc "รีเซ็ต rate limit สำหรับ key"
  def reset(server \\ __MODULE__, key) do
    GenServer.cast(server, {:reset, key})
  end

  @doc "ดูสถิติ"
  def status(server \\ __MODULE__, key) do
    GenServer.call(server, {:status, key})
  end

  ## Server Callbacks

  @impl GenServer
  def init(config) do
    # Schedule cleanup every 5 minutes
    Process.send_after(self(), :cleanup, 5 * 60 * 1000)
    {:ok, %{config: config, requests: %{}}}
  end

  @impl GenServer
  def handle_call({:check, key}, _from, state) do
    now = System.system_time(:millisecond)
    window_start = now - state.config.window_ms

    # Get valid timestamps (within window)
    timestamps = Map.get(state.requests, key, [])
    valid = Enum.filter(timestamps, fn ts -> ts > window_start end)

    if length(valid) >= state.config.max_requests do
      # Rate limited
      oldest = Enum.min(valid)
      retry_after = div(oldest + state.config.window_ms - now, 1000)
      new_state = put_in(state, [:requests, key], valid)
      {:reply, {:error, {:rate_limited, retry_after}}, new_state}
    else
      # Allow - add current timestamp
      new_timestamps = [now | valid]
      new_state = put_in(state, [:requests, key], new_timestamps)
      remaining = state.config.max_requests - length(new_timestamps)
      {:reply, {:ok, remaining}, new_state}
    end
  end

  def handle_call({:status, key}, _from, state) do
    now = System.system_time(:millisecond)
    window_start = now - state.config.window_ms
    timestamps = Map.get(state.requests, key, [])
    valid = Enum.filter(timestamps, fn ts -> ts > window_start end)
    status = %{
      key: key,
      requests_in_window: length(valid),
      max_requests: state.config.max_requests,
      remaining: state.config.max_requests - length(valid),
      window_ms: state.config.window_ms
    }
    {:reply, status, state}
  end

  @impl GenServer
  def handle_cast({:reset, key}, state) do
    {:noreply, %{state | requests: Map.delete(state.requests, key)}}
  end

  @impl GenServer
  def handle_info(:cleanup, state) do
    now = System.system_time(:millisecond)
    window_start = now - state.config.window_ms

    cleaned = state.requests
      |> Map.filter(fn {_, timestamps} ->
        Enum.any?(timestamps, fn ts -> ts > window_start end)
      end)
      |> Map.map(fn {_, timestamps} ->
        Enum.filter(timestamps, fn ts -> ts > window_start end)
      end)

    removed = map_size(state.requests) - map_size(cleaned)
    if removed > 0, do: IO.puts("Cleanup: removed #{removed} expired entries")

    # Schedule next cleanup
    Process.send_after(self(), :cleanup, 5 * 60 * 1000)
    {:noreply, %{state | requests: cleaned}}
  end
end

# ใช้งาน
{:ok, _} = RateLimiter.start_link(max_requests: 3, window_ms: 10_000)

# Test: ทำ requests จนเกิน limit
Enum.each(1..5, fn i ->
  result = RateLimiter.check("user:alice")
  IO.puts("Request #{i}: #{inspect(result)}")
end)

# Output:
# Request 1: {:ok, 2}
# Request 2: {:ok, 1}
# Request 3: {:ok, 0}
# Request 4: {:error, {:rate_limited, 9}}
# Request 5: {:error, {:rate_limited, 9}}

RateLimiter.status("user:alice")
# %{key: "user:alice", requests_in_window: 3, max_requests: 3, remaining: 0, ...}

RateLimiter.reset("user:alice")
RateLimiter.check("user:alice")
# {:ok, 2}
```

---

## 6. handle_continue

ใช้เพื่อทำงานหลัง init โดยไม่ block

```elixir
defmodule DatabaseServer do
  use GenServer

  def start_link(opts) do
    GenServer.start_link(__MODULE__, opts, name: __MODULE__)
  end

  @impl GenServer
  def init(opts) do
    # Return quickly, do expensive setup in handle_continue
    {:ok, %{config: opts, conn: nil, ready: false}, {:continue, :connect}}
  end

  @impl GenServer
  def handle_continue(:connect, state) do
    IO.puts("Connecting to database...")
    # Simulate connection
    :timer.sleep(100)
    conn = %{connected: true, host: state.config[:host]}
    IO.puts("Connected!")
    {:noreply, %{state | conn: conn, ready: true}}
  end

  def handle_call(:ping, _from, %{ready: true} = state) do
    {:reply, :pong, state}
  end

  def handle_call(:ping, _from, state) do
    {:reply, {:error, :not_ready}, state}
  end
end
```

---

## 7. Timeout ใน GenServer

```elixir
defmodule SessionServer do
  use GenServer

  @session_timeout 30_000  # 30 seconds

  def start_link(session_id) do
    GenServer.start_link(__MODULE__, session_id)
  end

  @impl GenServer
  def init(session_id) do
    # start with timeout
    {:ok, %{id: session_id, data: %{}}, @session_timeout}
  end

  @impl GenServer
  def handle_call({:set, key, value}, _from, state) do
    new_state = put_in(state, [:data, key], value)
    # reset timeout on activity
    {:reply, :ok, new_state, @session_timeout}
  end

  def handle_call({:get, key}, _from, state) do
    {:reply, Map.get(state.data, key), state, @session_timeout}
  end

  @impl GenServer
  def handle_info(:timeout, state) do
    IO.puts("Session #{state.id} expired")
    {:stop, :normal, state}
  end
end
```

---

## 8. GenServer ที่ใช้ struct เป็น state

```elixir
defmodule AccountServer do
  use GenServer

  defstruct [:id, :owner, balance: 0.0, transactions: []]

  ## Client API
  def start_link(id, owner) do
    GenServer.start_link(__MODULE__, {id, owner}, name: via(id))
  end

  def deposit(id, amount), do: GenServer.call(via(id), {:deposit, amount})
  def withdraw(id, amount), do: GenServer.call(via(id), {:withdraw, amount})
  def balance(id), do: GenServer.call(via(id), :balance)
  def transactions(id), do: GenServer.call(via(id), :transactions)

  defp via(id), do: {:via, Registry, {AccountRegistry, id}}

  ## Server Callbacks
  @impl GenServer
  def init({id, owner}) do
    {:ok, %__MODULE__{id: id, owner: owner}}
  end

  @impl GenServer
  def handle_call({:deposit, amount}, _from, state) when amount > 0 do
    txn = %{type: :deposit, amount: amount, at: DateTime.utc_now()}
    new_state = %{state |
      balance: state.balance + amount,
      transactions: [txn | state.transactions]
    }
    {:reply, {:ok, new_state.balance}, new_state}
  end

  def handle_call({:deposit, _amount}, _from, state) do
    {:reply, {:error, :invalid_amount}, state}
  end

  def handle_call({:withdraw, amount}, _from, state) when amount > 0 do
    if state.balance >= amount do
      txn = %{type: :withdrawal, amount: amount, at: DateTime.utc_now()}
      new_state = %{state |
        balance: state.balance - amount,
        transactions: [txn | state.transactions]
      }
      {:reply, {:ok, new_state.balance}, new_state}
    else
      {:reply, {:error, :insufficient_funds}, state}
    end
  end

  def handle_call({:withdraw, _amount}, _from, state) do
    {:reply, {:error, :invalid_amount}, state}
  end

  def handle_call(:balance, _from, state) do
    {:reply, state.balance, state}
  end

  def handle_call(:transactions, _from, state) do
    {:reply, Enum.reverse(state.transactions), state}
  end
end
```

---

## 9. Exercises

### Exercise 1: Key-Value Store

```elixir
# สร้าง KVStore GenServer ที่:
# - set(key, value) -> :ok
# - get(key) -> {:ok, value} | {:error, :not_found}
# - delete(key) -> :ok
# - keys() -> [key]
# - values() -> [value]
# - size() -> integer
# - exists?(key) -> boolean

defmodule KVStore do
  use GenServer
  # TODO: implement
end
```

### Exercise 2: Job Queue

```elixir
# สร้าง simple job queue:
# - enqueue(job) -> {:ok, job_id}
# - dequeue() -> {:ok, job} | {:error, :empty}
# - size() -> integer
# - peek() -> {:ok, job} | {:error, :empty}
# - jobs() -> [job]

defmodule JobQueue do
  use GenServer
  # TODO: implement
end
```

### Exercise 3: Leaderboard

```elixir
# สร้าง leaderboard server:
# - add_score(player, score) -> :ok
# - get_rank(player) -> {:ok, rank} | {:error, :not_found}
# - top(n) -> [{player, score, rank}]
# - player_score(player) -> {:ok, score} | {:error, :not_found}
# - reset() -> :ok

defmodule Leaderboard do
  use GenServer
  # TODO: implement
end
```

---

## เฉลย Exercises

### เฉลย Exercise 1

```elixir
defmodule KVStore do
  use GenServer

  def start_link(opts \\ []) do
    GenServer.start_link(__MODULE__, nil, Keyword.put_new(opts, :name, __MODULE__))
  end

  def set(server \\ __MODULE__, key, value), do: GenServer.cast(server, {:set, key, value})
  def get(server \\ __MODULE__, key), do: GenServer.call(server, {:get, key})
  def delete(server \\ __MODULE__, key), do: GenServer.cast(server, {:delete, key})
  def keys(server \\ __MODULE__), do: GenServer.call(server, :keys)
  def values(server \\ __MODULE__), do: GenServer.call(server, :values)
  def size(server \\ __MODULE__), do: GenServer.call(server, :size)
  def exists?(server \\ __MODULE__, key), do: GenServer.call(server, {:exists?, key})

  @impl GenServer
  def init(_), do: {:ok, %{}}

  @impl GenServer
  def handle_cast({:set, key, value}, state), do: {:noreply, Map.put(state, key, value)}
  def handle_cast({:delete, key}, state), do: {:noreply, Map.delete(state, key)}

  @impl GenServer
  def handle_call({:get, key}, _from, state) do
    result = case Map.get(state, key) do
      nil -> {:error, :not_found}
      value -> {:ok, value}
    end
    {:reply, result, state}
  end
  def handle_call(:keys, _from, state), do: {:reply, Map.keys(state), state}
  def handle_call(:values, _from, state), do: {:reply, Map.values(state), state}
  def handle_call(:size, _from, state), do: {:reply, map_size(state), state}
  def handle_call({:exists?, key}, _from, state), do: {:reply, Map.has_key?(state, key), state}
end
```

### เฉลย Exercise 3

```elixir
defmodule Leaderboard do
  use GenServer

  def start_link(opts \\ []) do
    GenServer.start_link(__MODULE__, nil, Keyword.put_new(opts, :name, __MODULE__))
  end

  def add_score(server \\ __MODULE__, player, score),
    do: GenServer.cast(server, {:add_score, player, score})
  def get_rank(server \\ __MODULE__, player),
    do: GenServer.call(server, {:get_rank, player})
  def top(server \\ __MODULE__, n),
    do: GenServer.call(server, {:top, n})
  def player_score(server \\ __MODULE__, player),
    do: GenServer.call(server, {:player_score, player})
  def reset(server \\ __MODULE__),
    do: GenServer.cast(server, :reset)

  @impl GenServer
  def init(_), do: {:ok, %{}}

  @impl GenServer
  def handle_cast({:add_score, player, score}, state) do
    current = Map.get(state, player, 0)
    {:noreply, Map.put(state, player, current + score)}
  end

  def handle_cast(:reset, _state), do: {:noreply, %{}}

  @impl GenServer
  def handle_call({:get_rank, player}, _from, state) do
    case Map.get(state, player) do
      nil ->
        {:reply, {:error, :not_found}, state}
      score ->
        rank = state
          |> Map.values()
          |> Enum.count(fn s -> s > score end)
        {:reply, {:ok, rank + 1}, state}
    end
  end

  def handle_call({:top, n}, _from, state) do
    ranked = state
      |> Enum.sort_by(fn {_, score} -> score end, :desc)
      |> Enum.take(n)
      |> Enum.with_index(1)
      |> Enum.map(fn {{player, score}, rank} ->
        %{player: player, score: score, rank: rank}
      end)
    {:reply, ranked, state}
  end

  def handle_call({:player_score, player}, _from, state) do
    result = case Map.get(state, player) do
      nil -> {:error, :not_found}
      score -> {:ok, score}
    end
    {:reply, result, state}
  end
end

# Test
{:ok, _} = Leaderboard.start_link()

Leaderboard.add_score("Alice", 100)
Leaderboard.add_score("Bob", 150)
Leaderboard.add_score("Charlie", 80)
Leaderboard.add_score("Alice", 50)

Leaderboard.top(3)
# [%{player: "Alice", score: 150, rank: 1},
#  %{player: "Bob", score: 150, rank: 1},
#  %{player: "Charlie", score: 80, rank: 3}]

Leaderboard.get_rank("Charlie")  # {:ok, 3}
```

---

## สรุป

```
GenServer:
├── use GenServer ใน module
├── Client API: start_link, public functions
└── Server Callbacks:
    ├── init(args) -> {:ok, state}
    ├── handle_call(req, from, state) -> {:reply, resp, state}
    ├── handle_cast(req, state) -> {:noreply, state}
    ├── handle_info(msg, state) -> {:noreply, state}
    ├── handle_continue(msg, state) -> {:noreply, state}
    └── terminate(reason, state) -> :ok

เมื่อไหรใช้ call vs cast:
├── call: เมื่อต้องการ response
└── cast: fire-and-forget (update state โดยไม่รอผล)
```

---

*ก่อนหน้า: [Part 15](part_15.md) | ต่อไป: [Part 17 - Supervisor และ OTP Trees](part_17.md)*
