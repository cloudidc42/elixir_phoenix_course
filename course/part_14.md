# Part 14: Processes และ Concurrency

## เป้าหมายการเรียนรู้

หลังจากจบ Part นี้ คุณจะสามารถ:
- สร้างและจัดการ Elixir Processes
- ส่ง/รับ Messages ระหว่าง Processes
- เข้าใจ Process isolation
- Link และ Monitor processes

---

## 1. Processes ใน Elixir

Elixir Process ไม่ใช่ OS thread - มันเบากว่ามาก:
- ขนาดเริ่มต้น ~1-2KB
- สร้างได้เร็วมาก (~microseconds)
- แต่ละ Process isolated (ไม่แชร์ memory)
- สื่อสารผ่าน message passing

```elixir
# สร้าง Process ด้วย spawn
pid = spawn(fn ->
  IO.puts("Hello from process #{inspect(self())}")
end)

IO.puts("Main process: #{inspect(self())}")
IO.puts("Spawned: #{inspect(pid)}")

# spawn/3 - module, function, args
pid = spawn(IO, :puts, ["Hello from spawn/3"])

# ตรวจสอบว่า process ยังอยู่ไหม
Process.alive?(pid)
```

---

## 2. Message Passing

```elixir
# ส่ง message ด้วย send/2
send(pid, {:hello, "world"})

# รับ message ด้วย receive/1
receive do
  {:hello, name} ->
    IO.puts("Hello, #{name}!")

  {:goodbye, name} ->
    IO.puts("Goodbye, #{name}!")

  other ->
    IO.puts("Unknown: #{inspect(other)}")
after
  5000 ->  # timeout: 5 วินาที
    IO.puts("No message received")
end

# ตัวอย่าง: Process สื่อสารกัน
defmodule Pinger do
  def start do
    spawn(__MODULE__, :loop, [])
  end

  def loop do
    receive do
      {:ping, caller} ->
        send(caller, :pong)
        loop()

      :stop ->
        IO.puts("Stopping...")
    end
  end
end

pid = Pinger.start()
send(pid, {:ping, self()})

receive do
  :pong -> IO.puts("Got pong!")
end

send(pid, :stop)
```

---

## 3. self() และ Process Info

```elixir
# self() คือ PID ของ process ปัจจุบัน
iex> self()
#PID<0.107.0>

# ข้อมูล process
iex> Process.info(self())
[
  current_function: {:prim_eval, :receive, 2},
  initial_call: {:erlang, :apply, 2},
  status: :waiting,
  message_queue_len: 0,
  # ...
]

# Registered names
Process.register(self(), :my_process)
send(:my_process, :hello)
```

---

## 4. Process Linking

```elixir
# Link: ถ้า process หนึ่ง crash อีก process ก็ crash ด้วย
pid = spawn_link(fn ->
  Process.sleep(100)
  raise "Crash!"
end)

# Link แบบ manual
Process.link(pid)

# Trap exits แทนที่จะ crash ด้วย
Process.flag(:trap_exit, true)

receive do
  {:EXIT, pid, reason} ->
    IO.puts("Process #{inspect(pid)} exited: #{inspect(reason)}")
end

# ตัวอย่าง: Worker ที่ link กับ parent
defmodule Worker do
  def start_linked do
    spawn_link(__MODULE__, :run, [self()])
  end

  def run(parent) do
    Process.flag(:trap_exit, true)

    receive do
      {:work, task} ->
        result = do_work(task)
        send(parent, {:result, self(), result})
        run(parent)

      {:EXIT, ^parent, _reason} ->
        IO.puts("Parent died, shutting down worker")
    end
  end

  defp do_work(task), do: task * 2
end
```

---

## 5. Process Monitoring

```elixir
# Monitor: รับ :DOWN message เมื่อ process ตาย แต่ไม่ crash ด้วย
ref = Process.monitor(pid)

receive do
  {:DOWN, ^ref, :process, ^pid, reason} ->
    IO.puts("Process died: #{inspect(reason)}")
end

# Monitor + spawn
{pid, ref} = spawn_monitor(fn ->
  Process.sleep(100)
  :done
end)

receive do
  {:DOWN, ^ref, :process, ^pid, :normal} ->
    IO.puts("Worker finished normally")

  {:DOWN, ^ref, :process, ^pid, reason} ->
    IO.puts("Worker failed: #{inspect(reason)}")
end
```

### Link vs Monitor

```
Link:
├── Bidirectional
├── Either crash -> both crash
├── ใช้สำหรับ processes ที่ต้องอยู่ด้วยกัน
└── spawn_link/1

Monitor:
├── Unidirectional (monitor -> monitored)
├── เมื่อ monitored ตาย -> monitor รับ :DOWN message
├── ใช้สำหรับ temporary workers
└── spawn_monitor/1, Process.monitor/1
```

---

## 6. Process Dictionary

```elixir
# Process dictionary = per-process mutable state (ใช้น้อยๆ)
Process.put(:key, "value")
Process.get(:key)   # "value"
Process.delete(:key)
Process.get_keys()  # ทุก keys

# ตัวอย่างการใช้: request ID tracking
defmodule RequestContext do
  def set_request_id(id) do
    Process.put(:request_id, id)
  end

  def get_request_id do
    Process.get(:request_id)
  end

  def clear do
    Process.delete(:request_id)
  end
end
```

---

## 7. Task Module

```elixir
# Task ใช้สำหรับ async computations ที่ง่ายกว่า spawn

# Task.async + Task.await
task = Task.async(fn ->
  Process.sleep(1000)
  :result
end)

result = Task.await(task, 5000)  # timeout: 5 วินาที

# หลาย tasks พร้อมกัน
tasks = Enum.map(1..5, fn i ->
  Task.async(fn ->
    Process.sleep(:rand.uniform(1000))
    {:task, i, :done}
  end)
end)

results = Task.await_many(tasks, 10_000)

# Task.start - fire and forget (no await)
Task.start(fn ->
  send_email("alice@example.com", "Hello!")
end)

# Task.start_link - linked to parent
{:ok, pid} = Task.start_link(fn ->
  process_data(data)
end)
```

---

## 8. ตัวอย่างจริง: Parallel Data Processor

```elixir
defmodule ParallelProcessor do
  def process_all(items, processor_fn, opts \\ []) do
    max_concurrency = Keyword.get(opts, :max_concurrency, System.schedulers_online())
    timeout = Keyword.get(opts, :timeout, 30_000)

    items
    |> Enum.chunk_every(max_concurrency)
    |> Enum.flat_map(fn batch ->
      batch
      |> Enum.map(fn item ->
        Task.async(fn ->
          try do
            {:ok, processor_fn.(item)}
          rescue
            e -> {:error, item, Exception.message(e)}
          end
        end)
      end)
      |> Task.await_many(timeout)
    end)
  end

  def process_with_results(items, processor_fn, opts \\ []) do
    results = process_all(items, processor_fn, opts)

    successes = Enum.filter(results, &match?({:ok, _}, &1))
    failures = Enum.filter(results, &match?({:error, _, _}, &1))

    %{
      success_count: length(successes),
      failure_count: length(failures),
      successes: Enum.map(successes, fn {:ok, r} -> r end),
      failures: Enum.map(failures, fn {:error, item, reason} ->
        %{item: item, reason: reason}
      end)
    }
  end
end

# ตัวอย่างการใช้งาน
urls = ["https://api1.example.com", "https://api2.example.com", "https://api3.example.com"]

results = ParallelProcessor.process_with_results(urls, fn url ->
  # Fetch each URL concurrently
  fetch_url(url)
end, max_concurrency: 5, timeout: 10_000)

IO.inspect(results)
```

---

## 9. ตัวอย่าง: Simple Actor Pattern

```elixir
defmodule Counter do
  def start(initial \\ 0) do
    spawn(__MODULE__, :loop, [initial])
  end

  def increment(pid, amount \\ 1) do
    send(pid, {:increment, amount})
  end

  def get(pid) do
    send(pid, {:get, self()})
    receive do
      {:value, v} -> v
    after
      5000 -> {:error, :timeout}
    end
  end

  def reset(pid) do
    send(pid, :reset)
  end

  def stop(pid) do
    send(pid, :stop)
  end

  def loop(count) do
    receive do
      {:increment, n} ->
        loop(count + n)

      {:get, caller} ->
        send(caller, {:value, count})
        loop(count)

      :reset ->
        loop(0)

      :stop ->
        :ok
    end
  end
end

# ใช้งาน
counter = Counter.start(0)
Counter.increment(counter)
Counter.increment(counter, 5)
IO.inspect(Counter.get(counter))  # 6
Counter.reset(counter)
IO.inspect(Counter.get(counter))  # 0
Counter.stop(counter)
```

---

## แบบฝึกหัด

### Exercise 1: Chat Room Process
สร้าง process ที่ทำหน้าที่เป็น chat room:
- Join: ลงทะเบียน user
- Leave: ออก
- Message: ส่ง message ไปทุก user ที่อยู่ใน room
- List: ดูรายชื่อ users ปัจจุบัน

### Exercise 2: Worker Pool
สร้าง pool ของ worker processes:
- Pool มี N workers
- Job queue
- Workers ดึง job มาทำ
- ส่ง result กลับ caller

---

## สรุป

```
Elixir Processes:
├── Lightweight (KB, ไม่ใช่ MB)
├── Isolated (ไม่แชร์ memory)
├── Communicate ด้วย messages
└── Millions สามารถ run พร้อมกัน

spawn/spawn_link/spawn_monitor:
├── spawn: independent
├── spawn_link: crash together
└── spawn_monitor: observe from outside

Message Passing:
├── send(pid, message)
├── receive do ... end
└── mailbox-based (async by default)

Task:
├── Task.async/await สำหรับ parallel work
├── Task.start สำหรับ fire-and-forget
└── Built on top of processes
```

---

## 13. Process Isolation

```elixir
# Process ใน Elixir แยก memory กัน
# การ "share" ข้อมูลทำผ่าน message passing เท่านั้น

pid1 = spawn(fn ->
  data = %{list: [1, 2, 3]}
  # data นี้อยู่ใน heap ของ pid1 เท่านั้น
  receive do
    {:get, sender} -> send(sender, data)
  end
end)

# เมื่อส่ง message -> BEAM copy data ไปยัง mailbox ของ receiver
send(pid1, {:get, self()})
receive do
  result ->
    # result เป็น copy ของ data จาก pid1
    IO.inspect(result)
end

# Large Binary (>= 64 bytes) ใช้ reference counting ไม่ copy
# เพื่อประสิทธิภาพ
```

---

## 14. Process ที่ Robust

```elixir
defmodule RobustProcess do
  def start do
    spawn(fn -> loop(%{errors: 0}) end)
  end

  defp loop(state) do
    receive do
      {:work, task, sender} ->
        result = try do
          {:ok, do_work(task)}
        rescue
          e -> {:error, Exception.message(e)}
        end
        send(sender, result)
        loop(state)

      {:error_count, sender} ->
        send(sender, state.errors)
        loop(state)

      :stop ->
        IO.puts("Process stopping gracefully")
    end
  end

  defp do_work(task) when is_integer(task), do: task * 2
  defp do_work(_), do: raise ArgumentError, "invalid task"
end

# ใช้งาน
pid = RobustProcess.start()

send(pid, {:work, 5, self()})
receive do
  {:ok, result} -> IO.puts("Result: #{result}")
end

send(pid, {:work, "invalid", self()})
receive do
  {:error, reason} -> IO.puts("Error: #{reason}")
end
```

---

## 15. Processes และ Memory

```elixir
# ดู memory usage ของ process
pid = spawn(fn -> :timer.sleep(10_000) end)

info = Process.info(pid, [:memory, :heap_size, :stack_size, :message_queue_len])
IO.inspect(info)
# [memory: 2688, heap_size: 233, stack_size: 10, message_queue_len: 0]

# ดู process ทั้งหมด
length(Process.list())  # จำนวน processes ที่รันอยู่

# Total memory usage
:erlang.memory(:total)  # bytes
:erlang.memory(:processes)  # memory used by all processes
```

---

## 16. Selective Receive ที่ถูกต้อง

```elixir
# ปัญหา: Selective receive อาจทำให้ mailbox ใหญ่มาก
# ถ้ามี messages ที่ไม่ match เยอะ -> mailbox scan นาน

# ไม่ดี: รอเฉพาะ response ของตัวเอง แต่ messages อื่นค้างอยู่
def send_and_wait(pid, request) do
  ref = make_ref()
  send(pid, {ref, request})
  receive do
    {^ref, response} -> response
  after
    5000 -> {:error, :timeout}
  end
end

# ดี: pattern ที่ใช้ ref เพื่อ match response ของตัวเอง
# ทำให้ messages อื่นไม่รบกวน แต่ยังค้างอยู่ใน mailbox
# GenServer ทำสิ่งนี้อยู่แล้วด้วย ref

# Best practice: ใช้ GenServer.call/cast แทน
# GenServer จัดการ mailbox ให้อย่างถูกต้อง
```

---

## สรุป

```
Process Basics:
├── spawn(fn) / spawn(mod, fun, args) - สร้าง process
├── self() - PID ของ process ปัจจุบัน
├── send(pid, message) - ส่ง message
└── receive do ... end - รับ message

Process Linking & Monitoring:
├── spawn_link - link กัน (crash propagate)
├── spawn_monitor - monitor (get :DOWN message)
├── Process.flag(:trap_exit, true)
└── Process.monitor(pid)

Process Tools:
├── Process.alive?(pid)
├── Process.register(pid, name)
├── Process.send_after(pid, msg, ms)
├── Process.put/get/delete - process dictionary
└── Process.registered()

spawn/spawn_link/spawn_monitor:
├── spawn: independent
├── spawn_link: crash together
└── spawn_monitor: observe from outside

Message Passing:
├── send(pid, message)
├── receive do ... end
└── mailbox-based (async by default)

Task:
├── Task.async/await สำหรับ parallel work
├── Task.start สำหรับ fire-and-forget
└── Built on top of processes
```

---

*ก่อนหน้า: [Part 13](part_13.md) | ต่อไป: [Part 15 - Tasks และ Agents](part_15.md)*
