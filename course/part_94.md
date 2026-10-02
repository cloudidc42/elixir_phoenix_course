# Part 94: Performance Optimization (การปรับประสิทธิภาพ)

## เป้าหมายการเรียนรู้

หลังจากจบ Part นี้ คุณจะสามารถ:
- Profiling ด้วย :fprof และ Benchee
- Database query optimization
- Process pool patterns
- Memory optimization

---

## 1. Benchmarking ด้วย Benchee

```elixir
# mix.exs: {:benchee, "~> 1.3", only: :dev}

# bench/list_operations.exs
Benchee.run(
  %{
    "Enum.map + filter" => fn ->
      1..10_000
      |> Enum.map(&(&1 * 2))
      |> Enum.filter(&(rem(&1, 3) == 0))
    end,
    "Enum.flat_map" => fn ->
      Enum.flat_map(1..10_000, fn n ->
        doubled = n * 2
        if rem(doubled, 3) == 0, do: [doubled], else: []
      end)
    end,
    "for comprehension" => fn ->
      for n <- 1..10_000,
          doubled = n * 2,
          rem(doubled, 3) == 0,
          do: doubled
    end,
    "Stream (lazy)" => fn ->
      1..10_000
      |> Stream.map(&(&1 * 2))
      |> Stream.filter(&(rem(&1, 3) == 0))
      |> Enum.to_list()
    end
  },
  warmup: 2,
  time: 5,
  memory_time: 2,
  formatters: [
    {Benchee.Formatters.HTML, file: "bench/results.html"},
    Benchee.Formatters.Console
  ]
)

# Run: mix run bench/list_operations.exs
```

---

## 2. Profiling ด้วย :fprof

```elixir
# In IEx:
:fprof.start()

# Profile a function call
:fprof.apply(&MyApp.Reports.generate/1, [report_params])

:fprof.profile()
:fprof.analyse(dest: 'profile.txt', sort: :own)

# Read profile output
# {totals, []} format shows which functions are slowest

# Alternative: :eprof for simple call counting
:eprof.start_profiling([self()])
MyApp.SomeModule.slow_function()
:eprof.stop_profiling()
:eprof.analyze()
```

---

## 3. Database N+1 Query Detection

```elixir
# Install: {:ecto_psql_extras, "~> 0.8"} for analysis

# Detect N+1 with Ecto logger
# config/dev.exs
config :my_app, MyApp.Repo,
  log: :debug

# The problem:
posts = Repo.all(Post)
posts_with_authors = Enum.map(posts, fn post ->
  %{post | author: Repo.get(User, post.author_id)}  # N queries!
end)

# The fix: preload
posts = Repo.all(Post) |> Repo.preload(:author)

# Or with join:
import Ecto.Query
posts = from(p in Post, preload: [:author]) |> Repo.all()

# Multiple level preload:
from(p in Post,
  preload: [author: :organization, comments: :author]
) |> Repo.all()

# Selective preload with query:
author_query = from(u in User, select: [:id, :name, :avatar_url])
from(p in Post,
  preload: [author: ^author_query]
) |> Repo.all()
```

---

## 4. ETS Process Pool

```elixir
defmodule MyApp.WorkerPool do
  use GenServer

  @pool_size 10
  @table :worker_pool

  def start_link(_) do
    GenServer.start_link(__MODULE__, [], name: __MODULE__)
  end

  def init([]) do
    :ets.new(@table, [:named_table, :set, :public])

    workers = for i <- 1..@pool_size do
      {:ok, pid} = MyApp.Worker.start_link(id: i)
      {i, pid, :available}
    end

    Enum.each(workers, &:ets.insert(@table, &1))
    {:ok, %{next_id: 1}}
  end

  def checkout() do
    case :ets.select(@table, [{{:"$1", :"$2", :available}, [], [{{:"$1", :"$2"}}]}]) do
      [{id, pid} | _] ->
        :ets.update_element(@table, id, {3, :busy})
        {:ok, pid}

      [] ->
        {:error, :pool_exhausted}
    end
  end

  def checkin(pid) do
    case :ets.match(@table, {:_, pid, :_}) do
      [[id]] ->
        :ets.update_element(@table, id, {3, :available})
        :ok
      _ ->
        :error
    end
  end

  def with_worker(fun) do
    case checkout() do
      {:ok, worker} ->
        try do
          fun.(worker)
        after
          checkin(worker)
        end

      {:error, :pool_exhausted} ->
        {:error, :pool_exhausted}
    end
  end
end

# Usage:
MyApp.WorkerPool.with_worker(fn worker ->
  MyApp.Worker.process(worker, job_params)
end)
```

---

## 5. Binary and String Optimization

```elixir
defmodule MyApp.StringUtils do
  # BAD: String.concat/2 creates many intermediate binaries
  def bad_join(list) do
    Enum.reduce(list, "", fn item, acc ->
      acc <> "," <> item  # creates new binary each time
    end)
  end

  # GOOD: IO lists avoid intermediate allocations
  def good_join(list) do
    list
    |> Enum.intersperse(",")
    |> IO.iodata_to_binary()
  end

  # BEST for output: use IO lists directly (Phoenix does this)
  def build_html(items) do
    # Returns IO list, not binary - Phoenix can send directly
    ["<ul>",
      Enum.map(items, fn item ->
        ["<li>", item.name, "</li>"]
      end),
     "</ul>"]
  end

  # Large binary processing - use :binary module
  def process_large_binary(binary) do
    # Don't pattern match on large binaries unnecessarily
    # Use :binary.part/3 for slicing
    header = :binary.part(binary, 0, 100)
    body = :binary.part(binary, 100, byte_size(binary) - 100)
    {header, body}
  end
end
```

---

## 6. Process Hibernation

```elixir
defmodule MyApp.InactiveGenServer do
  use GenServer

  @hibernate_after 5_000  # 5 seconds

  def init(state) do
    # Automatically hibernate after inactivity
    {:ok, state, {:continue, :init}}
  end

  def handle_continue(:init, state) do
    {:noreply, state, @hibernate_after}
  end

  def handle_info(:timeout, state) do
    # Hibernate to free memory (stops GC on this process)
    {:noreply, state, :hibernate}
  end

  def handle_call(:request, _from, state) do
    result = process(state)
    # Return to hibernation timeout after activity
    {:reply, result, state, @hibernate_after}
  end

  # GenServer option: built-in hibernate_after
  def start_link(state) do
    GenServer.start_link(__MODULE__, state,
      hibernate_after: @hibernate_after
    )
  end
end
```

---

## สรุป

```
Performance Tools:
├── Benchee: micro-benchmarks with statistics
├── :fprof: function-level profiling
├── :eprof: call counting profiling
└── ExUnit: timing with --slowest

Database:
├── Preload: eliminate N+1 queries
├── Indexes: cover WHERE/ORDER BY columns
├── EXPLAIN ANALYZE: query plan inspection
└── Connection pool: tune pool_size

Memory:
├── IO lists > string concatenation
├── Process.hibernate/3 for idle processes
├── ETS: shared state without GC pressure
└── Binary references: avoid unnecessary copies

Process Pools:
├── Poolboy/NimblePool: classic pools
├── Task.async_stream: bounded concurrency
└── Custom ETS pool: for control
```

---

*ก่อนหน้า: [Part 93](part_93.md) | ต่อไป: [Part 95 - Security Best Practices](part_95.md)*
