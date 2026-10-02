# Part 15: Tasks และ Agents

## เป้าหมายการเรียนรู้

หลังจากจบ Part นี้ คุณจะสามารถ:
- ใช้ `Task.async/await` สำหรับ concurrent operations
- ใช้ `Task.async_stream` สำหรับ parallel processing
- จัดการ timeouts และ errors ใน Tasks
- สร้างและใช้งาน Agent สำหรับ state management
- สร้าง Parallel File Processor และ Simple Cache

---

## 1. Task คืออะไร?

Task คือ abstraction ระดับสูงสำหรับ spawn process และดึง result

```
Process  <---  low level, ควบคุมเอง
Task     <---  high level, สำหรับ one-off concurrent operations
Agent    <---  high level, สำหรับ mutable state
GenServer <--- high level, สำหรับ complex stateful servers
```

---

## 2. Task.async และ Task.await

```elixir
# Basic usage
task = Task.async(fn ->
  :timer.sleep(1000)
  "result after 1 second"
end)

# ทำงานอื่นในระหว่างนั้น...
IO.puts("Doing other work...")

# รอผล
result = Task.await(task)
IO.puts("Got: #{result}")
# Got: result after 1 second
```

### Parallel Execution

```elixir
# ทำ 3 HTTP calls พร้อมกัน (simulation)
defmodule HttpSimulator do
  def fetch(url) do
    :timer.sleep(:rand.uniform(2000))
    {:ok, "Response from #{url}"}
  end
end

urls = [
  "https://api.example.com/users",
  "https://api.example.com/products",
  "https://api.example.com/orders"
]

# Sequential - ช้า (รอทีละ request)
{sequential_time, results} = :timer.tc(fn ->
  Enum.map(urls, &HttpSimulator.fetch/1)
end)

# Parallel - เร็วกว่า (รันพร้อมกัน)
{parallel_time, results} = :timer.tc(fn ->
  urls
  |> Enum.map(&Task.async(fn -> HttpSimulator.fetch(&1) end))
  |> Enum.map(&Task.await/1)
end)

IO.puts("Sequential: #{sequential_time / 1000}ms")
IO.puts("Parallel:   #{parallel_time / 1000}ms")
```

### Task.await timeout

```elixir
task = Task.async(fn ->
  :timer.sleep(10_000)  # ช้ามาก
  "slow result"
end)

# Default timeout = 5000ms
try do
  Task.await(task, 2000)  # 2 second timeout
rescue
  _ -> IO.puts("Task timed out!")
end

# หรือใช้ Task.yield ที่ไม่ crash
case Task.yield(task, 2000) || Task.shutdown(task) do
  {:ok, result} -> IO.puts("Got: #{result}")
  nil -> IO.puts("Task timed out, shutting down")
end
```

---

## 3. Task.async_stream

สำหรับ process list ของ items แบบ parallel

```elixir
items = 1..10

# Basic async_stream
results = items
  |> Task.async_stream(fn n ->
    :timer.sleep(100)
    n * n
  end)
  |> Enum.map(fn {:ok, result} -> result end)

# [1, 4, 9, 16, 25, 36, 49, 64, 81, 100]
```

### async_stream options

```elixir
Task.async_stream(
  items,
  fn item -> process(item) end,
  max_concurrency: 4,     # จำนวน workers (default: System.schedulers_online())
  timeout: 30_000,        # timeout per item (default: 5000ms)
  on_timeout: :kill_task, # หรือ :exit
  ordered: true           # preserve input order (default: true)
)
```

### Handle errors in async_stream

```elixir
results = 1..10
  |> Task.async_stream(fn n ->
    if rem(n, 3) == 0 do
      raise "Error on #{n}"
    end
    n * 2
  end, on_timeout: :kill_task)
  |> Enum.reduce({[], []}, fn
    {:ok, result}, {oks, errors} -> {[result | oks], errors}
    {:exit, reason}, {oks, errors} -> {oks, [reason | errors]}
  end)

{successes, failures} = results
```

---

## 4. Task.Supervisor

```elixir
# สร้าง Task.Supervisor ใน application supervisor
children = [
  {Task.Supervisor, name: MyApp.TaskSupervisor}
]

# รัน task ภายใต้ supervisor
task = Task.Supervisor.async(MyApp.TaskSupervisor, fn ->
  heavy_computation()
end)

result = Task.await(task)

# Fire-and-forget (ไม่รอผล)
Task.Supervisor.start_child(MyApp.TaskSupervisor, fn ->
  send_notification(user)
end)
```

---

## 5. Agent พื้นฐาน

Agent ใช้สำหรับ maintain state ใน separate process

```elixir
# Start agent
{:ok, agent} = Agent.start_link(fn -> [] end)

# Update state
Agent.update(agent, fn list -> [1 | list] end)
Agent.update(agent, fn list -> [2 | list] end)
Agent.update(agent, fn list -> [3 | list] end)

# Get state
Agent.get(agent, fn list -> list end)
# [3, 2, 1]

# Get and update atomically
Agent.get_and_update(agent, fn list ->
  {hd(list), tl(list)}
end)
# Returns: 3, state becomes [2, 1]

# Stop agent
Agent.stop(agent)
```

### Named Agent

```elixir
# Named agent สำหรับ global access
Agent.start_link(fn -> %{} end, name: :config_store)

Agent.update(:config_store, fn cfg ->
  Map.put(cfg, :debug, true)
end)

Agent.get(:config_store, fn cfg -> cfg end)
# %{debug: true}
```

---

## 6. ตัวอย่างจริง: Simple Cache ด้วย Agent

```elixir
defmodule SimpleCache do
  use Agent

  @default_ttl 300  # 5 minutes

  def start_link(opts \\ []) do
    Agent.start_link(fn -> %{} end, name: __MODULE__)
  end

  def get(key) do
    Agent.get(__MODULE__, fn cache ->
      case Map.get(cache, key) do
        nil -> {:miss}
        {value, expires_at} ->
          if System.system_time(:second) < expires_at do
            {:hit, value}
          else
            {:expired}
          end
      end
    end)
  end

  def put(key, value, ttl \\ @default_ttl) do
    expires_at = System.system_time(:second) + ttl
    Agent.update(__MODULE__, fn cache ->
      Map.put(cache, key, {value, expires_at})
    end)
    :ok
  end

  def delete(key) do
    Agent.update(__MODULE__, fn cache -> Map.delete(cache, key) end)
    :ok
  end

  def clear do
    Agent.update(__MODULE__, fn _ -> %{} end)
    :ok
  end

  def fetch(key, default_fn, ttl \\ @default_ttl) do
    case get(key) do
      {:hit, value} ->
        {:ok, value, :cached}
      _ ->
        case default_fn.() do
          {:ok, value} ->
            put(key, value, ttl)
            {:ok, value, :fetched}
          error -> error
        end
    end
  end

  def size do
    Agent.get(__MODULE__, fn cache ->
      now = System.system_time(:second)
      Enum.count(cache, fn {_, {_, expires_at}} ->
        expires_at > now
      end)
    end)
  end

  def cleanup do
    Agent.update(__MODULE__, fn cache ->
      now = System.system_time(:second)
      Enum.reject(cache, fn {_, {_, expires_at}} ->
        expires_at <= now
      end)
      |> Map.new()
    end)
  end

  def stats do
    Agent.get(__MODULE__, fn cache ->
      now = System.system_time(:second)
      {active, expired} = Enum.split_with(cache, fn {_, {_, exp}} -> exp > now end)
      %{
        total: map_size(cache),
        active: length(active),
        expired: length(expired)
      }
    end)
  end
end

# ใช้งาน
{:ok, _} = SimpleCache.start_link()

SimpleCache.put("user:1", %{name: "Alice", age: 30})
SimpleCache.put("user:2", %{name: "Bob", age: 25}, 60)
SimpleCache.put("config:theme", "dark", 3600)

SimpleCache.get("user:1")
# {:hit, %{name: "Alice", age: 30}}

SimpleCache.get("user:99")
# {:miss}

# fetch with automatic caching
SimpleCache.fetch("user:3", fn ->
  # Simulate DB query
  :timer.sleep(100)
  {:ok, %{name: "Charlie", age: 35}}
end)
# {:ok, %{name: "Charlie"...}, :fetched}

SimpleCache.fetch("user:3", fn -> {:ok, "from_db"} end)
# {:ok, %{name: "Charlie"...}, :cached}  <- ครั้งที่ 2 จาก cache

SimpleCache.stats()
# %{total: 4, active: 4, expired: 0}
```

---

## 7. ตัวอย่างจริง: Parallel File Processing

```elixir
defmodule ParallelFileProcessor do
  @doc """
  Process ไฟล์หลายไฟล์พร้อมกัน
  """
  def process_directory(dir_path, opts \\ []) do
    max_concurrency = Keyword.get(opts, :max_concurrency, System.schedulers_online())
    timeout = Keyword.get(opts, :timeout, 30_000)
    extensions = Keyword.get(opts, :extensions, [".txt", ".csv", ".json"])

    with {:ok, files} <- list_files(dir_path, extensions),
         {:ok, results} <- process_files(files, max_concurrency, timeout) do
      summary = build_summary(results)
      {:ok, summary}
    end
  end

  defp list_files(dir_path, extensions) do
    case File.ls(dir_path) do
      {:ok, files} ->
        full_paths = files
          |> Enum.filter(fn f -> Path.extname(f) in extensions end)
          |> Enum.map(fn f -> Path.join(dir_path, f) end)
        {:ok, full_paths}
      {:error, reason} ->
        {:error, "Cannot list directory: #{inspect(reason)}"}
    end
  end

  defp process_files(files, max_concurrency, timeout) do
    results = files
      |> Task.async_stream(
        fn file -> process_single_file(file) end,
        max_concurrency: max_concurrency,
        timeout: timeout,
        on_timeout: :kill_task
      )
      |> Enum.zip(files)
      |> Enum.map(fn
        {{:ok, result}, file} -> {:ok, file, result}
        {{:exit, reason}, file} -> {:error, file, reason}
      end)

    {:ok, results}
  end

  defp process_single_file(file_path) do
    case File.stat(file_path) do
      {:ok, stat} ->
        case File.read(file_path) do
          {:ok, content} ->
            %{
              path: file_path,
              name: Path.basename(file_path),
              size: stat.size,
              lines: length(String.split(content, "\n")),
              words: length(String.split(content)),
              chars: String.length(content),
              extension: Path.extname(file_path),
              processed_at: DateTime.utc_now()
            }
          {:error, reason} ->
            raise "Cannot read #{file_path}: #{inspect(reason)}"
        end
      {:error, reason} ->
        raise "Cannot stat #{file_path}: #{inspect(reason)}"
    end
  end

  defp build_summary(results) do
    successes = Enum.filter(results, &match?({:ok, _, _}, &1))
    failures = Enum.filter(results, &match?({:error, _, _}, &1))

    file_data = Enum.map(successes, fn {:ok, _file, data} -> data end)

    %{
      total_files: length(results),
      processed: length(successes),
      failed: length(failures),
      total_size: Enum.sum(Enum.map(file_data, & &1.size)),
      total_lines: Enum.sum(Enum.map(file_data, & &1.lines)),
      total_words: Enum.sum(Enum.map(file_data, & &1.words)),
      files: file_data,
      errors: Enum.map(failures, fn {:error, file, reason} ->
        %{file: file, reason: inspect(reason)}
      end)
    }
  end
end

# ใช้งาน
case ParallelFileProcessor.process_directory("/tmp/data", [
  max_concurrency: 8,
  extensions: [".txt", ".csv"]
]) do
  {:ok, summary} ->
    IO.puts("Processed #{summary.processed}/#{summary.total_files} files")
    IO.puts("Total size: #{summary.total_size} bytes")
    IO.puts("Total lines: #{summary.total_lines}")
    IO.puts("Total words: #{summary.total_words}")

    if summary.failed > 0 do
      IO.puts("\nFailed files:")
      Enum.each(summary.errors, fn err ->
        IO.puts("  #{err.file}: #{err.reason}")
      end)
    end

  {:error, reason} ->
    IO.puts("Error: #{reason}")
end
```

---

## 8. Task.yield_many

รอหลาย tasks พร้อมกัน

```elixir
tasks = Enum.map(1..5, fn n ->
  Task.async(fn ->
    :timer.sleep(n * 200)
    "result_#{n}"
  end)
end)

# รอ 1 วินาที แล้วเก็บผลที่ได้
results = Task.yield_many(tasks, 1000)

Enum.each(results, fn {task, result} ->
  case result do
    {:ok, value} ->
      IO.puts("Task completed: #{value}")
    nil ->
      Task.shutdown(task, :brutal_kill)
      IO.puts("Task timed out, killed")
  end
end)
```

---

## 9. Agent ที่ซับซ้อนขึ้น: Shopping Cart

```elixir
defmodule ShoppingCart do
  use Agent

  defstruct items: [], discount: nil

  def start_link(user_id) do
    Agent.start_link(fn -> %__MODULE__{} end, name: via_tuple(user_id))
  end

  def add_item(user_id, item) do
    Agent.update(via_tuple(user_id), fn cart ->
      existing = Enum.find(cart.items, fn i -> i.id == item.id end)
      if existing do
        new_items = Enum.map(cart.items, fn i ->
          if i.id == item.id, do: %{i | quantity: i.quantity + item.quantity}, else: i
        end)
        %{cart | items: new_items}
      else
        %{cart | items: [item | cart.items]}
      end
    end)
  end

  def remove_item(user_id, item_id) do
    Agent.update(via_tuple(user_id), fn cart ->
      %{cart | items: Enum.reject(cart.items, fn i -> i.id == item_id end)}
    end)
  end

  def apply_discount(user_id, discount_code) do
    discounts = %{"SAVE10" => 0.10, "SAVE20" => 0.20, "HALF" => 0.50}
    case Map.get(discounts, discount_code) do
      nil ->
        {:error, :invalid_code}
      percent ->
        Agent.update(via_tuple(user_id), fn cart ->
          %{cart | discount: %{code: discount_code, percent: percent}}
        end)
        :ok
    end
  end

  def total(user_id) do
    Agent.get(via_tuple(user_id), fn cart ->
      subtotal = Enum.reduce(cart.items, 0.0, fn item, acc ->
        acc + item.price * item.quantity
      end)

      case cart.discount do
        nil -> subtotal
        %{percent: p} -> subtotal * (1 - p)
      end
    end)
  end

  def summary(user_id) do
    Agent.get(via_tuple(user_id), fn cart ->
      subtotal = Enum.reduce(cart.items, 0.0, fn item, acc ->
        acc + item.price * item.quantity
      end)
      discount_amount = case cart.discount do
        nil -> 0.0
        %{percent: p} -> subtotal * p
      end
      %{
        items: cart.items,
        item_count: length(cart.items),
        subtotal: Float.round(subtotal, 2),
        discount: cart.discount,
        discount_amount: Float.round(discount_amount, 2),
        total: Float.round(subtotal - discount_amount, 2)
      }
    end)
  end

  def clear(user_id) do
    Agent.update(via_tuple(user_id), fn _ -> %__MODULE__{} end)
  end

  def stop(user_id) do
    Agent.stop(via_tuple(user_id))
  end

  defp via_tuple(user_id) do
    {:via, Registry, {ShoppingCart.Registry, user_id}}
  end
end

# ต้อง start Registry ก่อน
# children = [{Registry, keys: :unique, name: ShoppingCart.Registry}]

# ใช้งาน (แบบไม่ใช้ Registry สำหรับ simplicity)
{:ok, cart} = Agent.start_link(fn -> %{items: [], discount: nil} end)

# เพิ่มสินค้า
Agent.update(cart, fn c ->
  %{c | items: [%{id: 1, name: "Laptop", price: 999.99, quantity: 1} | c.items]}
end)

Agent.get(cart, fn c -> c end)
```

---

## 10. Exercises

### Exercise 1: Parallel Download Simulator

```elixir
# สร้าง parallel downloader ที่:
# 1. รับ list ของ URLs
# 2. Download พร้อมกัน (max 5 concurrent)
# 3. Progress tracking (กี่ไฟล์เสร็จแล้ว)
# 4. รองรับ retry (2 ครั้ง) ถ้า download ล้มเหลว

defmodule ParallelDownloader do
  def download_all(urls, opts \\ []) do
    max_concurrent = Keyword.get(opts, :max_concurrent, 5)
    max_retries = Keyword.get(opts, :max_retries, 2)
    # TODO: implement
  end

  defp download_with_retry(url, max_retries) do
    # TODO: implement with retry logic
  end
end
```

### Exercise 2: Rate Limiter Agent

```elixir
# สร้าง Rate Limiter ด้วย Agent
# - จำกัด N requests ต่อ time window
# - check_rate(key) -> :ok | {:error, :rate_limited}
# - Window แบบ sliding (ไม่ใช่ fixed)

defmodule RateLimiter do
  use Agent

  def start_link(opts \\ []) do
    # max_requests: 10, window_ms: 60_000
    Agent.start_link(fn -> %{} end, name: __MODULE__)
  end

  def check_rate(key, max_requests \\ 10, window_ms \\ 60_000) do
    # TODO: implement
  end
end
```

### Exercise 3: Word Frequency Counter

```elixir
# นับความถี่ของคำในไฟล์หลายไฟล์แบบ parallel
# แล้ว merge ผลลัพธ์

defmodule WordFrequency do
  def count_in_directory(dir_path) do
    # 1. อ่านไฟล์ .txt ทั้งหมดใน directory
    # 2. Count words ใน parallel
    # 3. Merge results
    # 4. คืน top 20 words
    # TODO: implement
  end
end
```

---

## เฉลย Exercises

### เฉลย Exercise 2: Rate Limiter

```elixir
defmodule RateLimiter do
  use Agent

  def start_link(_opts \\ []) do
    Agent.start_link(fn -> %{} end, name: __MODULE__)
  end

  def check_rate(key, max_requests \\ 10, window_ms \\ 60_000) do
    now = System.system_time(:millisecond)
    window_start = now - window_ms

    Agent.get_and_update(__MODULE__, fn state ->
      # Get current timestamps for this key
      timestamps = Map.get(state, key, [])

      # Remove expired timestamps (outside window)
      valid_timestamps = Enum.filter(timestamps, fn ts -> ts > window_start end)

      if length(valid_timestamps) >= max_requests do
        oldest = List.first(Enum.sort(valid_timestamps))
        wait_ms = oldest + window_ms - now
        result = {:error, {:rate_limited, wait_ms}}
        {result, Map.put(state, key, valid_timestamps)}
      else
        new_timestamps = [now | valid_timestamps]
        {:ok, Map.put(state, key, new_timestamps)}
      end
    end)
  end

  def reset(key) do
    Agent.update(__MODULE__, fn state -> Map.delete(state, key) end)
  end

  def stats(key) do
    Agent.get(__MODULE__, fn state ->
      timestamps = Map.get(state, key, [])
      %{key: key, request_count: length(timestamps)}
    end)
  end
end

# Test
{:ok, _} = RateLimiter.start_link()

# ทำ requests จนเกิน limit
results = Enum.map(1..15, fn i ->
  result = RateLimiter.check_rate("user:1")
  IO.puts("Request #{i}: #{inspect(result)}")
  result
end)

ok_count = Enum.count(results, &(&1 == :ok))
rate_limited_count = Enum.count(results, &match?({:error, _}, &1))
IO.puts("OK: #{ok_count}, Rate limited: #{rate_limited_count}")
```

### เฉลย Exercise 3: Word Frequency Counter

```elixir
defmodule WordFrequency do
  def count_in_directory(dir_path, top_n \\ 20) do
    with {:ok, files} <- list_text_files(dir_path),
         {:ok, counts} <- parallel_count(files),
         merged <- merge_counts(counts) do
      top = merged
        |> Enum.sort_by(fn {_, count} -> count end, :desc)
        |> Enum.take(top_n)
      {:ok, top}
    end
  end

  defp list_text_files(dir) do
    case File.ls(dir) do
      {:ok, files} ->
        txt_files = files
          |> Enum.filter(&String.ends_with?(&1, ".txt"))
          |> Enum.map(&Path.join(dir, &1))
        {:ok, txt_files}
      error -> error
    end
  end

  defp parallel_count(files) do
    results = files
      |> Task.async_stream(
        fn file -> count_file(file) end,
        max_concurrency: System.schedulers_online(),
        timeout: 10_000
      )
      |> Enum.flat_map(fn
        {:ok, counts} -> [counts]
        {:exit, _} -> []
      end)
    {:ok, results}
  end

  defp count_file(path) do
    case File.read(path) do
      {:ok, content} ->
        content
        |> String.downcase()
        |> String.replace(~r/[^a-z\s]/, "")
        |> String.split()
        |> Enum.frequencies()
      {:error, _} -> %{}
    end
  end

  defp merge_counts(counts) do
    Enum.reduce(counts, %{}, fn file_counts, acc ->
      Map.merge(acc, file_counts, fn _k, v1, v2 -> v1 + v2 end)
    end)
  end
end

# Test
case WordFrequency.count_in_directory("/tmp/texts") do
  {:ok, top_words} ->
    IO.puts("Top 20 words:")
    Enum.each(top_words, fn {word, count} ->
      IO.puts("  #{word}: #{count}")
    end)
  {:error, reason} ->
    IO.puts("Error: #{inspect(reason)}")
end
```

---

## สรุป

```
Task:
├── Task.async(fn) + Task.await(task)
├── Task.async_stream(list, fn, opts)
│   ├── max_concurrency: N
│   ├── timeout: ms
│   └── on_timeout: :kill_task | :exit
├── Task.yield(task, timeout) - non-raising await
├── Task.yield_many(tasks, timeout)
└── Task.shutdown(task) - cancel task

Agent:
├── Agent.start_link(fn -> initial_state end)
├── Agent.get(agent, fn state -> ... end)
├── Agent.update(agent, fn state -> new_state end)
├── Agent.get_and_update(agent, fn state -> {return, new_state} end)
└── Agent.stop(agent)

เมื่อไหรใช้อะไร:
├── Task   - one-off async computation, parallel processing
├── Agent  - simple shared mutable state
└── GenServer - complex state, need handle_info, supervision
```

---

*ก่อนหน้า: [Part 14](part_14.md) | ต่อไป: [Part 16 - GenServer พื้นฐาน](part_16.md)*
