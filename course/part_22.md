# Part 22: GenStage และ Flow

## เป้าหมายการเรียนรู้

หลังจากจบ Part นี้ คุณจะสามารถ:
- เข้าใจแนวคิด back-pressure ใน data pipeline
- สร้าง Producer, Consumer, และ ProducerConsumer ด้วย GenStage
- ใช้ Flow สำหรับ parallel data processing
- สร้าง data processing pipeline ที่ scale ได้

---

## 1. GenStage คืออะไร?

GenStage เป็น library สำหรับสร้าง **data processing pipeline** โดยมีระบบ **back-pressure** ในตัว

### ปัญหาที่ GenStage แก้

```
ปัญหาทั่วไปในการประมวลผลข้อมูล:

Producer (เร็ว) ---[ข้อมูลล้น]--> Consumer (ช้า)
  |                                    |
  | ผลิต 1000 items/sec                | ประมวลผล 100 items/sec
  |                                    |
  Buffer โตขึ้นเรื่อยๆ จนหน่วยความจำเต็ม!
```

```
GenStage แก้ด้วย back-pressure:

Producer <---[demand: 50 items]--- Consumer
  |                                    |
  | ผลิตเฉพาะตามที่ถูกขอ               | ขอข้อมูลเพิ่มเมื่อพร้อม
  |                                    |
  ไม่มีปัญหา buffer overflow!
```

### ติดตั้ง GenStage

```elixir
# mix.exs
defp deps do
  [
    {:gen_stage, "~> 1.2"},
    {:flow, "~> 1.2"}  # สำหรับ Flow
  ]
end
```

---

## 2. Producer

Producer คือ source ของข้อมูล มีหน้าที่ผลิตข้อมูลเมื่อถูก demand

```elixir
defmodule NumberProducer do
  use GenStage

  def start_link(initial_number \\ 0) do
    GenStage.start_link(__MODULE__, initial_number, name: __MODULE__)
  end

  @impl true
  def init(initial_number) do
    {:producer, initial_number}
  end

  @impl true
  def handle_demand(demand, current_number) when demand > 0 do
    # สร้าง numbers ตาม demand ที่ขอมา
    events = Enum.to_list(current_number..(current_number + demand - 1))
    next_number = current_number + demand

    IO.puts("Producing #{demand} events: #{List.first(events)}..#{List.last(events)}")

    {:noreply, events, next_number}
  end
end
```

### Producer จาก External Source

```elixir
defmodule DatabaseProducer do
  use GenStage

  def start_link(query) do
    GenStage.start_link(__MODULE__, query)
  end

  @impl true
  def init(query) do
    state = %{
      query: query,
      offset: 0,
      batch_size: 100
    }
    {:producer, state}
  end

  @impl true
  def handle_demand(demand, %{query: query, offset: offset, batch_size: batch_size} = state) do
    # จำลองการ query database
    events = fetch_from_db(query, offset, min(demand, batch_size))

    if Enum.empty?(events) do
      # ไม่มีข้อมูลแล้ว
      {:stop, :normal, state}
    else
      new_state = %{state | offset: offset + length(events)}
      {:noreply, events, new_state}
    end
  end

  defp fetch_from_db(_query, offset, limit) do
    # จำลอง DB query
    total_records = 500
    if offset >= total_records do
      []
    else
      end_idx = min(offset + limit - 1, total_records - 1)
      Enum.map(offset..end_idx, fn i ->
        %{id: i, name: "User #{i}", email: "user#{i}@example.com"}
      end)
    end
  end
end
```

---

## 3. Consumer

Consumer รับข้อมูลจาก Producer แล้วประมวลผล

```elixir
defmodule NumberPrinter do
  use GenStage

  def start_link(opts \\ []) do
    GenStage.start_link(__MODULE__, :ok, opts)
  end

  @impl true
  def init(:ok) do
    {:consumer, :ok}
  end

  @impl true
  def handle_events(events, _from, state) do
    # ประมวลผลแต่ละ event
    Enum.each(events, fn event ->
      IO.puts("Processing: #{event}")
      # จำลองการทำงานที่ใช้เวลา
      Process.sleep(10)
    end)

    # Consumer ต้อง return {:noreply, [], state}
    {:noreply, [], state}
  end
end
```

### Consumer พร้อม Error Handling

```elixir
defmodule EmailSender do
  use GenStage

  def start_link(_opts \\ []) do
    GenStage.start_link(__MODULE__, %{sent: 0, failed: 0})
  end

  @impl true
  def init(state) do
    {:consumer, state}
  end

  @impl true
  def handle_events(events, _from, state) do
    {sent, failed} = Enum.reduce(events, {0, 0}, fn user, {s, f} ->
      case send_email(user) do
        :ok -> {s + 1, f}
        {:error, _} -> {s, f + 1}
      end
    end)

    new_state = %{
      sent: state.sent + sent,
      failed: state.failed + failed
    }

    IO.puts("Stats - Sent: #{new_state.sent}, Failed: #{new_state.failed}")

    {:noreply, [], new_state}
  end

  defp send_email(%{email: email}) do
    # จำลองการส่ง email
    if :rand.uniform(10) > 1 do
      IO.puts("Sent email to #{email}")
      :ok
    else
      {:error, "Failed to send to #{email}"}
    end
  end
end
```

---

## 4. ProducerConsumer

ProducerConsumer รับข้อมูลจาก Producer แล้ว transform ก่อนส่งต่อ Consumer

```elixir
defmodule DataTransformer do
  use GenStage

  def start_link(transform_fn) do
    GenStage.start_link(__MODULE__, transform_fn)
  end

  @impl true
  def init(transform_fn) do
    {:producer_consumer, transform_fn}
  end

  @impl true
  def handle_events(events, _from, transform_fn) do
    transformed = Enum.map(events, transform_fn)
    {:noreply, transformed, transform_fn}
  end
end

defmodule DataFilter do
  use GenStage

  def start_link(filter_fn) do
    GenStage.start_link(__MODULE__, filter_fn)
  end

  @impl true
  def init(filter_fn) do
    {:producer_consumer, filter_fn}
  end

  @impl true
  def handle_events(events, _from, filter_fn) do
    filtered = Enum.filter(events, filter_fn)
    {:noreply, filtered, filter_fn}
  end
end
```

---

## 5. เชื่อมต่อ Pipeline

```elixir
defmodule Pipeline do
  def start do
    # Start producer
    {:ok, producer} = NumberProducer.start_link(1)

    # Start transformer (multiply by 2)
    {:ok, transformer} = DataTransformer.start_link(fn n -> n * 2 end)

    # Start filter (only even numbers > 10)
    {:ok, filter} = DataFilter.start_link(fn n -> rem(n, 4) == 0 end)

    # Start consumer
    {:ok, consumer} = NumberPrinter.start_link([])

    # Connect: producer -> transformer -> filter -> consumer
    GenStage.sync_subscribe(transformer, to: producer, max_demand: 10)
    GenStage.sync_subscribe(filter, to: transformer, max_demand: 10)
    GenStage.sync_subscribe(consumer, to: filter, max_demand: 5)

    # ให้ run สักพัก
    Process.sleep(1000)

    # Stop
    GenStage.stop(producer)
  end
end

Pipeline.start()
```

### Pipeline ที่มีหลาย Consumer

```elixir
defmodule MultiConsumerPipeline do
  def start do
    {:ok, producer} = NumberProducer.start_link(1)

    # Consumer หลายตัวแบ่งกันรับ
    consumers = for i <- 1..3 do
      {:ok, pid} = GenStage.start_link(NumberPrinter, :ok)
      GenStage.sync_subscribe(pid,
        to: producer,
        max_demand: 5,
        min_demand: 1
      )
      pid
    end

    Process.sleep(2000)
    Enum.each(consumers, &GenStage.stop/1)
    GenStage.stop(producer)
  end
end
```

---

## 6. Back-pressure ใน Detail

```elixir
defmodule BackPressureDemo do
  use GenStage

  # Producer ที่ track demand
  defmodule TrackingProducer do
    use GenStage

    def start_link(_), do: GenStage.start_link(__MODULE__, 0)

    @impl true
    def init(n), do: {:producer, n}

    @impl true
    def handle_demand(demand, n) do
      IO.puts("[Producer] Received demand for #{demand} events")
      events = Enum.to_list(n..(n + demand - 1))
      IO.puts("[Producer] Sending #{length(events)} events")
      {:noreply, events, n + demand}
    end
  end

  # Consumer ที่ช้า
  defmodule SlowConsumer do
    use GenStage

    def start_link(_), do: GenStage.start_link(__MODULE__, 0)

    @impl true
    def init(n), do: {:consumer, n, subscribe_to: []}

    @impl true
    def handle_events(events, _from, n) do
      IO.puts("[Consumer] Processing #{length(events)} events...")
      # จำลองการทำงานช้า
      Process.sleep(500)
      IO.puts("[Consumer] Done processing #{length(events)} events")
      {:noreply, [], n + length(events)}
    end
  end
end
```

---

## 7. Flow - Parallel Processing

Flow สร้างบน GenStage และทำให้การ parallel processing ง่ายขึ้น

```elixir
# Flow ต้อง add ใน deps ก่อน
# {:flow, "~> 1.2"}

# Word count แบบ parallel
defmodule WordCount do
  def count(text) do
    text
    |> String.split("\n")
    |> Flow.from_enumerable()
    |> Flow.flat_map(&String.split/1)
    |> Flow.partition()
    |> Flow.reduce(fn -> %{} end, fn word, acc ->
      Map.update(acc, word, 1, &(&1 + 1))
    end)
    |> Enum.to_list()
    |> Enum.sort_by(fn {_word, count} -> -count end)
  end
end

text = """
the quick brown fox jumps over the lazy dog
the dog barked at the fox
the fox ran away quickly
"""

result = WordCount.count(text)
IO.inspect(result)
# => [{"the", 5}, {"fox", 3}, ...]
```

### Flow Operations

```elixir
defmodule FlowExample do
  def basic_flow do
    # from_enumerable: สร้าง Flow จาก list
    result =
      1..1000
      |> Flow.from_enumerable()
      |> Flow.map(&(&1 * 2))           # transform แต่ละ element
      |> Flow.filter(&(rem(&1, 3) == 0))  # filter
      |> Flow.partition()              # group by key สำหรับ reduce
      |> Flow.reduce(fn -> [] end, fn item, acc -> [item | acc] end)
      |> Enum.to_list()
      |> Enum.sort()

    result
  end

  def with_windows do
    # Flow.Window สำหรับ streaming data
    result =
      1..100
      |> Flow.from_enumerable()
      |> Flow.partition(window: Flow.Window.count(10))
      |> Flow.reduce(fn -> [] end, fn item, acc -> [item | acc] end)
      |> Flow.on_trigger(fn acc, _index, _trigger ->
        {[Enum.sum(acc)], []}  # คืน sum ของแต่ละ window
      end)
      |> Enum.to_list()

    result
  end
end

IO.inspect(FlowExample.basic_flow() |> Enum.take(10))
```

---

## 8. ตัวอย่างจริง: Data Processing Pipeline

```elixir
# สร้าง data processing pipeline สำหรับ user data

defmodule UserDataPipeline do

  # ========== Producers ==========

  defmodule CSVProducer do
    use GenStage

    def start_link(filename) do
      GenStage.start_link(__MODULE__, filename)
    end

    @impl true
    def init(filename) do
      # จำลองการ read CSV
      lines = generate_csv_data(1000)
      {:producer, %{lines: lines, position: 0}}
    end

    @impl true
    def handle_demand(demand, %{lines: lines, position: pos} = state) do
      available = Enum.drop(lines, pos) |> Enum.take(demand)

      if Enum.empty?(available) do
        {:stop, :normal, state}
      else
        {:noreply, available, %{state | position: pos + length(available)}}
      end
    end

    defp generate_csv_data(n) do
      for i <- 1..n do
        "#{i},user#{i}@example.com,#{Enum.random(["Alice", "Bob", "Charlie", "Dave"])},#{Enum.random(18..80)}"
      end
    end
  end

  # ========== Transformers ==========

  defmodule CSVParser do
    use GenStage

    def start_link(_), do: GenStage.start_link(__MODULE__, :ok)

    @impl true
    def init(:ok), do: {:producer_consumer, :ok}

    @impl true
    def handle_events(lines, _from, state) do
      parsed = Enum.map(lines, &parse_line/1)
      {:noreply, parsed, state}
    end

    defp parse_line(line) do
      [id, email, name, age] = String.split(line, ",")
      %{
        id: String.to_integer(id),
        email: email,
        name: name,
        age: String.to_integer(age)
      }
    end
  end

  defmodule DataValidator do
    use GenStage

    def start_link(_), do: GenStage.start_link(__MODULE__, %{valid: 0, invalid: 0})

    @impl true
    def init(state), do: {:producer_consumer, state}

    @impl true
    def handle_events(users, _from, state) do
      {valid, invalid} = Enum.split_with(users, &valid?/1)

      new_state = %{
        valid: state.valid + length(valid),
        invalid: state.invalid + length(invalid)
      }

      if rem(new_state.valid + new_state.invalid, 100) == 0 do
        IO.puts("Validated: #{new_state.valid} valid, #{new_state.invalid} invalid")
      end

      {:noreply, valid, new_state}
    end

    defp valid?(%{email: email, age: age, name: name}) do
      String.contains?(email, "@") and
      age >= 18 and age <= 120 and
      String.length(name) > 0
    end
  end

  defmodule DataEnricher do
    use GenStage

    def start_link(_), do: GenStage.start_link(__MODULE__, :ok)

    @impl true
    def init(:ok), do: {:producer_consumer, :ok}

    @impl true
    def handle_events(users, _from, state) do
      enriched = Enum.map(users, &enrich/1)
      {:noreply, enriched, state}
    end

    defp enrich(user) do
      user
      |> Map.put(:age_group, age_group(user.age))
      |> Map.put(:domain, extract_domain(user.email))
      |> Map.put(:processed_at, DateTime.utc_now())
    end

    defp age_group(age) when age < 30, do: :young
    defp age_group(age) when age < 60, do: :middle
    defp age_group(_age), do: :senior

    defp extract_domain(email) do
      email |> String.split("@") |> List.last()
    end
  end

  # ========== Consumer ==========

  defmodule DatabaseWriter do
    use GenStage

    def start_link(_), do: GenStage.start_link(__MODULE__, %{written: 0, batch: []})

    @impl true
    def init(state), do: {:consumer, state}

    @impl true
    def handle_events(users, _from, state) do
      batch = state.batch ++ users

      if length(batch) >= 50 do
        # เขียนลง DB
        write_to_db(batch)
        IO.puts("Written #{state.written + length(batch)} records to DB")
        {:noreply, [], %{state | written: state.written + length(batch), batch: []}}
      else
        {:noreply, [], %{state | batch: batch}}
      end
    end

    defp write_to_db(users) do
      # จำลองการเขียน DB
      Process.sleep(div(length(users), 10))
    end
  end

  # ========== Pipeline Supervisor ==========

  def start do
    {:ok, producer} = CSVProducer.start_link("users.csv")
    {:ok, parser} = CSVParser.start_link([])
    {:ok, validator} = DataValidator.start_link([])
    {:ok, enricher} = DataEnricher.start_link([])
    {:ok, writer} = DatabaseWriter.start_link([])

    # Connect pipeline
    GenStage.sync_subscribe(parser, to: producer, max_demand: 100)
    GenStage.sync_subscribe(validator, to: parser, max_demand: 100)
    GenStage.sync_subscribe(enricher, to: validator, max_demand: 100)
    GenStage.sync_subscribe(writer, to: enricher, max_demand: 50)

    # Wait for completion
    ref = Process.monitor(producer)
    receive do
      {:DOWN, ^ref, :process, _, :normal} ->
        IO.puts("Pipeline completed!")
    after
      30_000 ->
        IO.puts("Pipeline timeout!")
    end
  end
end

# รัน pipeline
UserDataPipeline.start()
```

---

## 9. Flow กับ Real Data

```elixir
defmodule LogAnalyzer do
  @doc """
  วิเคราะห์ log files แบบ parallel
  """

  def analyze(log_entries) do
    log_entries
    |> Flow.from_enumerable(max_demand: 50)
    |> Flow.map(&parse_log/1)
    |> Flow.filter(&(&1 != nil))
    |> Flow.partition(key: fn %{level: level} -> level end)
    |> Flow.reduce(fn -> %{} end, fn entry, acc ->
      Map.update(acc, entry.level, [entry], &[entry | &1])
    end)
    |> Flow.on_trigger(fn acc, _index, _trigger ->
      stats = Map.new(acc, fn {level, entries} ->
        {level, length(entries)}
      end)
      {[stats], acc}
    end)
    |> Enum.to_list()
    |> merge_stats()
  end

  defp parse_log(line) do
    case Regex.run(~r/\[(\w+)\] (.+)/, line) do
      [_, level, message] ->
        %{
          level: String.to_atom(String.downcase(level)),
          message: message,
          timestamp: DateTime.utc_now()
        }
      _ ->
        nil
    end
  end

  defp merge_stats(stats_list) do
    Enum.reduce(stats_list, %{}, fn stats, acc ->
      Map.merge(acc, stats, fn _key, v1, v2 -> v1 + v2 end)
    end)
  end
end

# ตัวอย่างข้อมูล
logs = [
  "[ERROR] Database connection failed",
  "[INFO] User logged in: alice",
  "[WARN] Memory usage high: 85%",
  "[INFO] Request processed in 150ms",
  "[ERROR] File not found: config.yml",
  "[DEBUG] Cache hit for key: user:123",
  "[INFO] Server started on port 4000",
  "[ERROR] Timeout connecting to service",
]

result = LogAnalyzer.analyze(logs)
IO.inspect(result)
# => %{error: 3, info: 3, warn: 1, debug: 1}
```

---

## 10. Exercises

### Exercise 1: สร้าง rate-limited producer

สร้าง Producer ที่ produce ไม่เกิน N events ต่อวินาที

**เฉลย:**

```elixir
defmodule RateLimitedProducer do
  use GenStage

  def start_link(rate_per_second, total_items) do
    GenStage.start_link(__MODULE__, {rate_per_second, total_items, 0})
  end

  @impl true
  def init({rate, total, sent}) do
    {:producer, %{rate: rate, total: total, sent: sent, queue: :queue.new()}}
  end

  @impl true
  def handle_demand(demand, state) do
    %{rate: rate, total: total, sent: sent} = state

    available = min(demand, total - sent)

    if available <= 0 do
      {:stop, :normal, state}
    else
      # Rate limit: sleep ระหว่าง batches
      items_per_tick = max(1, div(rate, 10))
      batch = min(available, items_per_tick)

      events = Enum.to_list((sent + 1)..(sent + batch))

      # Schedule timer สำหรับ rate limiting
      if available > batch do
        Process.send_after(self(), {:produce_more, available - batch}, div(1000, 10))
      end

      {:noreply, events, %{state | sent: sent + batch}}
    end
  end

  @impl true
  def handle_info({:produce_more, remaining}, state) do
    %{rate: rate, sent: sent} = state
    items_per_tick = max(1, div(rate, 10))
    batch = min(remaining, items_per_tick)

    events = Enum.to_list((sent + 1)..(sent + batch))

    if remaining > batch do
      Process.send_after(self(), {:produce_more, remaining - batch}, div(1000, 10))
    end

    {:noreply, events, %{state | sent: sent + batch}}
  end
end
```

### Exercise 2: Pipeline ที่ monitor metrics

สร้าง pipeline ที่เก็บ metrics (throughput, error rate) ไว้

**เฉลย:**

```elixir
defmodule MetricsCollector do
  use GenServer

  def start_link(_), do: GenServer.start_link(__MODULE__, %{}, name: __MODULE__)

  def record(metric, value) do
    GenServer.cast(__MODULE__, {:record, metric, value})
  end

  def get_stats do
    GenServer.call(__MODULE__, :get_stats)
  end

  @impl true
  def init(state), do: {:ok, state}

  @impl true
  def handle_cast({:record, metric, value}, state) do
    new_state = Map.update(state, metric, [value], &[value | &1])
    {:noreply, new_state}
  end

  @impl true
  def handle_call(:get_stats, _from, state) do
    stats = Map.new(state, fn {metric, values} ->
      {metric, %{
        count: length(values),
        sum: Enum.sum(values),
        avg: Enum.sum(values) / max(length(values), 1)
      }}
    end)
    {:reply, stats, state}
  end
end

defmodule MonitoredConsumer do
  use GenStage

  def start_link(_), do: GenStage.start_link(__MODULE__, :ok)

  @impl true
  def init(:ok), do: {:consumer, :ok}

  @impl true
  def handle_events(events, _from, state) do
    start = System.monotonic_time(:millisecond)

    Enum.each(events, fn event ->
      # process event
      MetricsCollector.record(:processed, 1)
    end)

    elapsed = System.monotonic_time(:millisecond) - start
    MetricsCollector.record(:latency_ms, elapsed)

    {:noreply, [], state}
  end
end
```

---

## สรุป

```
GenStage:
├── Producer: ผลิตข้อมูล, handle_demand
├── Consumer: รับข้อมูล, handle_events
├── ProducerConsumer: รับและส่งต่อ
└── Back-pressure: Consumer ควบคุม demand

Pipeline Connection:
├── GenStage.sync_subscribe/3
├── max_demand: ขอมากสุด
├── min_demand: ขอเพิ่มเมื่อต่ำกว่า
└── หลาย Consumer รับ load ร่วมกัน

Flow:
├── สร้างบน GenStage
├── Flow.from_enumerable/2
├── map, filter, partition, reduce
├── on_trigger สำหรับ windowing
└── Parallel โดย default

Use Cases:
├── ETL pipeline
├── Log processing
├── Image/video processing
└── Real-time data streams
```

---

*ก่อนหน้า: [Part 21 - Macros และ Metaprogramming](part_21.md) | ต่อไป: [Part 23 - Registry และ Process Naming](part_23.md)*
