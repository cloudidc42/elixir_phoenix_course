# Part 18: ETS: Erlang Term Storage

## เป้าหมายการเรียนรู้

หลังจากจบ Part นี้ คุณจะสามารถ:
- เข้าใจ ETS table types: set, ordered_set, bag, duplicate_bag
- ทำ CRUD operations บน ETS
- เข้าใจว่าเมื่อไหรใช้ ETS vs Agent vs GenServer
- สร้าง In-memory cache ที่ performant ด้วย ETS

---

## 1. ETS คืออะไร?

ETS (Erlang Term Storage) คือ in-memory storage ที่แชร์ระหว่าง processes ได้

```
ETS Features:
┌─────────────────────────────────────────────────────────────┐
│  - Fast O(1) lookups (เหมือน HashMap)                       │
│  - Concurrent reads (concurrent access)                     │
│  - Shared between processes (ต่างจาก Process Dictionary)    │
│  - ไม่มี serialization overhead (เก็บ Elixir terms โดยตรง) │
│  - Transient: ถ้า owning process ตาย -> table หาย          │
└─────────────────────────────────────────────────────────────┘
```

---

## 2. Table Types

```elixir
# :set - key unique, ไม่เรียงลำดับ (default)
:ets.new(:my_set, [:set])
# key: value one-to-one

# :ordered_set - key unique, เรียงลำดับ
:ets.new(:ordered, [:ordered_set])
# เร็วกว่าในการ iterate ตามลำดับ

# :bag - key ซ้ำได้, value ต้อง unique per key
:ets.new(:my_bag, [:bag])
# key: [value1, value2] แต่ value ซ้ำไม่ได้

# :duplicate_bag - key ซ้ำได้, value ซ้ำได้
:ets.new(:dup_bag, [:duplicate_bag])
# key: [value1, value1, value2] ซ้ำได้
```

---

## 3. Access Types

```elixir
# :protected (default) - owner เขียนได้, ทุกคนอ่านได้
:ets.new(:protected_table, [:set, :protected])

# :public - ทุกคนเขียนได้, ทุกคนอ่านได้
:ets.new(:public_table, [:set, :public])

# :private - เฉพาะ owner เท่านั้น
:ets.new(:private_table, [:set, :private])
```

---

## 4. CRUD Operations

### Create (insert)

```elixir
# สร้าง table
table = :ets.new(:users, [:set, :public, :named_table])

# insert tuple - element แรกคือ key
:ets.insert(:users, {1, "Alice", 30, "alice@example.com"})
:ets.insert(:users, {2, "Bob", 25, "bob@example.com"})
:ets.insert(:users, {3, "Charlie", 35, "charlie@example.com"})

# insert หลาย records พร้อมกัน
:ets.insert(:users, [
  {4, "Dave", 28, "dave@example.com"},
  {5, "Eve", 22, "eve@example.com"}
])
```

### Read (lookup)

```elixir
# lookup by key - คืน list ของ tuples
:ets.lookup(:users, 1)
# [{1, "Alice", 30, "alice@example.com"}]

:ets.lookup(:users, 99)
# [] (ไม่เจอ)

# lookup element เฉพาะตัว (เร็วกว่า pattern matching)
:ets.lookup_element(:users, 1, 2)  # element ที่ 2 ของ tuple
# "Alice"

:ets.lookup_element(:users, 1, 3)  # element ที่ 3
# 30

# ตรวจสอบว่า key มีอยู่ไหม
:ets.member(:users, 1)   # true
:ets.member(:users, 99)  # false

# first และ next (สำหรับ iteration)
:ets.first(:users)        # 1 (หรือ key ใดก็ได้ใน set)
:ets.next(:users, 1)      # 2
:ets.next(:users, 5)      # '$end_of_table'
```

### Pattern Matching / Match Spec

```elixir
# match - ใช้ '_' เป็น wildcard
:ets.match(:users, {:'_', '$1', :'_', :'_'})
# ดึงชื่อทุกคน: [["Alice"], ["Bob"], ...]

:ets.match(:users, {'$1', '$2', '$3', :'_'})
# [[1, "Alice", 30], [2, "Bob", 25], ...]

# match_object - คืน full record
:ets.match_object(:users, {:'_', :'_', 30, :'_'})
# [{1, "Alice", 30, "alice@example.com"}]

# select ด้วย match spec
spec = [{{:_, :_, :'$1', :_}, [{:>, :'$1', 25}], [:'$_']}]
:ets.select(:users, spec)
# ดึง users ที่อายุ > 25
```

### Update

```elixir
# insert ซ้ำ = overwrite (สำหรับ :set)
:ets.insert(:users, {1, "Alice Updated", 31, "alice_new@example.com"})

# update_element - เปลี่ยนเฉพาะ field
:ets.update_element(:users, 1, {3, 31})  # เปลี่ยน age (position 3) เป็น 31

:ets.update_element(:users, 1, [{3, 32}, {4, "alice2@example.com"}])

# update_counter - atomic increment
:ets.new(:counters, [:set, :public, :named_table])
:ets.insert(:counters, {:page_views, 0})
:ets.update_counter(:counters, :page_views, 1)   # +1
:ets.update_counter(:counters, :page_views, -1)  # -1
:ets.update_counter(:counters, :page_views, {2, 5})  # element 2 + 5
```

### Delete

```elixir
# ลบ record
:ets.delete(:users, 1)

# ลบด้วย pattern
:ets.match_delete(:users, {:'_', :'_', 30, :'_'})
# ลบ users ที่ age = 30

# ล้าง table
:ets.delete_all_objects(:users)

# ลบ table ทั้งหมด
:ets.delete(:users)
```

---

## 5. All Records Iteration

```elixir
# ดูทุก records
:ets.tab2list(:users)
# [{1, "Alice", 30, "..."}, {2, "Bob", 25, "..."}, ...]

# นับ records
:ets.info(:users, :size)
# 5

# ดูข้อมูล table
:ets.info(:users)
# [id: ..., name: :users, size: 5, type: :set, ...]

# fold over all records
:ets.foldl(fn record, acc ->
  [record | acc]
end, [], :users)
```

---

## 6. Named vs Anonymous Tables

```elixir
# Named table - เข้าถึงด้วยชื่อได้จากทุก process
:ets.new(:my_table, [:set, :public, :named_table])

# ทุก process เข้าถึงได้
:ets.insert(:my_table, {:key, :value})
:ets.lookup(:my_table, :key)

# Anonymous table - เข้าถึงด้วย table reference เท่านั้น
table_ref = :ets.new(:unnamed, [:set, :public])
:ets.insert(table_ref, {:key, :value})
```

---

## 7. ETS vs Agent vs GenServer

```
┌──────────────────────────────────────────────────────────────┐
│                 WHEN TO USE WHAT?                            │
├─────────────────┬───────────────┬──────────────┬────────────┤
│                 │    ETS        │    Agent     │ GenServer  │
├─────────────────┼───────────────┼──────────────┼────────────┤
│ Concurrent read │ ✅ Fast       │ ❌ Serialized│ ❌ Serial │
│ Concurrent write│ ✅ Public     │ ❌ Serialized│ ❌ Serial │
│ Complex state   │ ❌ Raw data   │ ✅ Any data  │ ✅ Any    │
│ Transactions    │ ❌ No         │ ✅ Limited   │ ✅ Full   │
│ Persistence     │ ❌ RAM only   │ ❌ RAM only  │ ❌ RAM    │
│ Pattern match   │ ✅ Powerful   │ ❌ Manual    │ ❌ Manual │
│ Linked to owner │ ✅ Yes        │ ✅ Yes       │ ✅ Yes    │
│ Supervision     │ ❌ Manual     │ ✅ Auto      │ ✅ Auto   │
└─────────────────┴───────────────┴──────────────┴────────────┘

Use ETS when:
- High read throughput (concurrent reads)
- Counter/metrics
- Cache with fast lookup
- Shared data between many processes

Use Agent when:
- Simple shared mutable state
- Development/prototyping
- Low concurrency

Use GenServer when:
- Complex state machine
- Need to serialize operations
- Event handling (handle_info)
- Want OTP supervision
```

---

## 8. ตัวอย่างจริง: In-Memory Cache ด้วย ETS

```elixir
defmodule EtsCache do
  @moduledoc """
  High-performance in-memory cache using ETS
  - Thread-safe reads (concurrent)
  - TTL support
  - Auto cleanup
  """

  use GenServer

  @table_name :ets_cache
  @cleanup_interval 60_000  # 1 minute

  ## Client API (ไม่ต้องผ่าน GenServer สำหรับ reads!)

  def start_link(opts \\ []) do
    GenServer.start_link(__MODULE__, opts, name: __MODULE__)
  end

  # DIRECT ETS READ - ไม่ต้องผ่าน GenServer process!
  def get(key) do
    now = System.system_time(:second)
    case :ets.lookup(@table_name, key) do
      [{^key, value, expires_at}] when expires_at > now ->
        {:ok, value}
      [{^key, _value, _expires_at}] ->
        # Expired - lazy delete
        :ets.delete(@table_name, key)
        {:miss}
      [] ->
        {:miss}
    end
  end

  # WRITE ผ่าน GenServer (serialize writes)
  def put(key, value, ttl \\ 300) do
    GenServer.call(__MODULE__, {:put, key, value, ttl})
  end

  def delete(key) do
    GenServer.cast(__MODULE__, {:delete, key})
  end

  def clear do
    GenServer.cast(__MODULE__, :clear)
  end

  def fetch(key, fallback_fn, ttl \\ 300) do
    case get(key) do
      {:ok, value} -> {:ok, value, :cached}
      {:miss} ->
        case fallback_fn.() do
          {:ok, value} ->
            put(key, value, ttl)
            {:ok, value, :fetched}
          error -> error
        end
    end
  end

  def size do
    :ets.info(@table_name, :size)
  end

  def stats do
    now = System.system_time(:second)
    all = :ets.tab2list(@table_name)
    {active, expired} = Enum.split_with(all, fn {_, _, exp} -> exp > now end)
    %{
      total: length(all),
      active: length(active),
      expired: length(expired),
      memory_bytes: :ets.info(@table_name, :memory) * :erlang.system_info(:wordsize)
    }
  end

  ## Server Callbacks

  @impl GenServer
  def init(_opts) do
    # สร้าง ETS table
    :ets.new(@table_name, [
      :set,
      :public,       # ทุก process อ่านได้โดยตรง
      :named_table,  # เข้าถึงด้วยชื่อ
      {:read_concurrency, true},   # optimize concurrent reads
      {:write_concurrency, true}   # optimize concurrent writes
    ])

    # Schedule periodic cleanup
    Process.send_after(self(), :cleanup, @cleanup_interval)

    {:ok, %{}}
  end

  @impl GenServer
  def handle_call({:put, key, value, ttl}, _from, state) do
    expires_at = System.system_time(:second) + ttl
    :ets.insert(@table_name, {key, value, expires_at})
    {:reply, :ok, state}
  end

  @impl GenServer
  def handle_cast({:delete, key}, state) do
    :ets.delete(@table_name, key)
    {:noreply, state}
  end

  def handle_cast(:clear, state) do
    :ets.delete_all_objects(@table_name)
    {:noreply, state}
  end

  @impl GenServer
  def handle_info(:cleanup, state) do
    now = System.system_time(:second)
    # ลบ entries ที่ expired
    # ใช้ match_delete กับ guard
    spec = [{{:_, :_, :'$1'}, [{:<, :'$1', now}], [true]}]
    deleted = :ets.select_delete(@table_name, spec)

    if deleted > 0 do
      IO.puts("Cache cleanup: removed #{deleted} expired entries")
    end

    Process.send_after(self(), :cleanup, @cleanup_interval)
    {:noreply, state}
  end
end

# ใช้งาน
{:ok, _} = EtsCache.start_link()

EtsCache.put("user:1", %{name: "Alice", age: 30}, 300)
EtsCache.put("user:2", %{name: "Bob", age: 25}, 60)
EtsCache.put(:config, %{debug: false, max_connections: 100}, 3600)

# Read ตรงจาก ETS (ไม่ผ่าน GenServer process)
EtsCache.get("user:1")
# {:ok, %{name: "Alice", age: 30}}

EtsCache.get("user:99")
# {:miss}

# Fetch with auto-populate
EtsCache.fetch("user:3", fn ->
  # Simulate DB query
  :timer.sleep(50)
  {:ok, %{name: "Charlie", age: 35}}
end)
# {:ok, %{name: "Charlie", age: 35}, :fetched}

# ครั้งที่สอง - จาก cache
EtsCache.fetch("user:3", fn -> {:ok, "from_db"} end)
# {:ok, %{name: "Charlie", age: 35}, :cached}

EtsCache.stats()
# %{total: 4, active: 4, expired: 0, memory_bytes: ...}
```

---

## 9. ETS สำหรับ Counters

```elixir
defmodule Metrics do
  @table :metrics

  def start do
    :ets.new(@table, [:set, :public, :named_table, {:write_concurrency, true}])
    :ok
  end

  def increment(metric, amount \\ 1) do
    :ets.update_counter(@table, metric, {2, amount}, {metric, 0})
  end

  def decrement(metric, amount \\ 1) do
    :ets.update_counter(@table, metric, {2, -amount}, {metric, 0})
  end

  def get(metric) do
    case :ets.lookup(@table, metric) do
      [{^metric, count}] -> count
      [] -> 0
    end
  end

  def reset(metric) do
    :ets.insert(@table, {metric, 0})
  end

  def all do
    :ets.tab2list(@table)
    |> Map.new(fn {k, v} -> {k, v} end)
  end
end

# ใช้งาน
Metrics.start()

# Atomic counters - thread-safe!
Metrics.increment(:page_views)
Metrics.increment(:page_views)
Metrics.increment(:page_views, 5)  # +5

Metrics.increment(:api_calls)
Metrics.increment(:errors)

Metrics.get(:page_views)  # 7
Metrics.all()
# %{page_views: 7, api_calls: 1, errors: 1}
```

---

## 10. DETS: Disk-based ETS

```elixir
# DETS เป็น disk-based version ของ ETS
{:ok, table} = :dets.open_file(:my_persistent_table, [type: :set])

:dets.insert(table, {:key1, "persistent_value"})
:dets.lookup(table, :key1)
# [{:key1, "persistent_value"}]

# Close เมื่อเสร็จ
:dets.close(table)

# Data ยังอยู่หลัง restart!
{:ok, same_table} = :dets.open_file(:my_persistent_table, [type: :set])
:dets.lookup(same_table, :key1)
# [{:key1, "persistent_value"}]
```

---

## 11. Exercises

### Exercise 1: Session Store

```elixir
# สร้าง Session Store ด้วย ETS:
# - create_session(user_id) -> session_id
# - get_session(session_id) -> {:ok, session} | {:error, :not_found}
# - update_session(session_id, updates) -> :ok | {:error, :not_found}
# - delete_session(session_id) -> :ok
# - cleanup_expired() -> integer (จำนวน sessions ที่ลบ)

defmodule SessionStore do
  @session_ttl 3600  # 1 hour

  def start, do: # TODO: create ETS table

  def create_session(user_id) do
    session_id = generate_id()
    # TODO: store session
    {:ok, session_id}
  end

  # TODO: implement rest
end
```

### Exercise 2: Phone Book

```elixir
# สร้าง Phone Book ด้วย ETS bag:
# - add(name, phone) -> :ok  (คนๆ เดียวมีได้หลายเบอร์)
# - lookup(name) -> [phone]
# - remove(name, phone) -> :ok
# - remove_all(name) -> :ok
# - search(prefix) -> [{name, [phone]}]  (ค้นหาจาก name prefix)

defmodule PhoneBook do
  def start, do: :ets.new(:phone_book, [:bag, :public, :named_table])
  # TODO: implement
end
```

### Exercise 3: Leaderboard ด้วย ETS

```elixir
# สร้าง Leaderboard ที่ high-performance ด้วย ETS:
# - record_score(player, score) -> :ok
# - get_rank(player) -> {:ok, rank} | {:error, :not_found}
# - top(n) -> [{player, score, rank}]
# - player_stats(player) -> %{score, rank, games_played}

defmodule EtsLeaderboard do
  # ใช้ :ordered_set เรียงตาม key = {-score, player}
  # เพื่อให้ score สูงอยู่หน้า
  def start, do: # TODO
end
```

---

## เฉลย Exercises

### เฉลย Exercise 1

```elixir
defmodule SessionStore do
  use GenServer

  @table :sessions
  @session_ttl 3600

  def start_link(_opts \\ []) do
    GenServer.start_link(__MODULE__, nil, name: __MODULE__)
  end

  def create_session(user_id) do
    session_id = :crypto.strong_rand_bytes(32) |> Base.url_encode64(padding: false)
    expires_at = System.system_time(:second) + @session_ttl
    session = %{
      id: session_id,
      user_id: user_id,
      created_at: System.system_time(:second),
      expires_at: expires_at,
      data: %{}
    }
    :ets.insert(@table, {session_id, session})
    {:ok, session_id}
  end

  def get_session(session_id) do
    now = System.system_time(:second)
    case :ets.lookup(@table, session_id) do
      [{^session_id, session}] ->
        if session.expires_at > now do
          {:ok, session}
        else
          :ets.delete(@table, session_id)
          {:error, :expired}
        end
      [] -> {:error, :not_found}
    end
  end

  def update_session(session_id, updates) do
    case get_session(session_id) do
      {:ok, session} ->
        updated = Map.merge(session, updates)
        :ets.insert(@table, {session_id, updated})
        :ok
      error -> error
    end
  end

  def delete_session(session_id) do
    :ets.delete(@table, session_id)
    :ok
  end

  def cleanup_expired do
    now = System.system_time(:second)
    all = :ets.tab2list(@table)
    expired = Enum.filter(all, fn {_, session} -> session.expires_at <= now end)
    Enum.each(expired, fn {id, _} -> :ets.delete(@table, id) end)
    length(expired)
  end

  @impl GenServer
  def init(_) do
    :ets.new(@table, [:set, :public, :named_table])
    Process.send_after(self(), :cleanup, 60_000)
    {:ok, %{}}
  end

  @impl GenServer
  def handle_info(:cleanup, state) do
    count = cleanup_expired()
    if count > 0, do: IO.puts("Cleaned up #{count} expired sessions")
    Process.send_after(self(), :cleanup, 60_000)
    {:noreply, state}
  end
end
```

---

## สรุป

```
ETS Types:
├── :set - unique key, unordered
├── :ordered_set - unique key, ordered
├── :bag - duplicate keys, unique values per key
└── :duplicate_bag - duplicate keys + values

Access Types:
├── :protected (default) - owner writes, all reads
├── :public - all read/write
└── :private - owner only

Key Operations:
├── :ets.new(name, opts)
├── :ets.insert(table, record)
├── :ets.lookup(table, key)
├── :ets.delete(table, key)
├── :ets.update_counter(table, key, incr)
└── :ets.select(table, match_spec)

Performance Tips:
├── {:read_concurrency, true} - for read-heavy
├── {:write_concurrency, true} - for write-heavy
├── Direct ETS reads (bypass GenServer)
└── ETS > Agent สำหรับ concurrent read workloads
```

---

*ก่อนหน้า: [Part 17](part_17.md) | ต่อไป: [Part 19 - Mix: Build Tool](part_19.md)*
