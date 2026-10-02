# Part 47: GenStage and Flow (Data Processing Pipeline)

## เป้าหมายการเรียนรู้

- เข้าใจ stages ทั้งสามประเภท: Producer, ProducerConsumer, Consumer
- รู้จัก back-pressure mechanism และทำไมมันสำคัญ
- กำหนด subscription options ใน GenStage
- ใช้ Flow สำหรับการประมวลผลแบบ parallel
- ประมวลผลไฟล์ CSV ด้วย Flow
- สร้าง pipeline สำหรับ metrics aggregation
- ควบคุม rate ของ producer

---

## 1. GenStage คืออะไร?

**GenStage** เป็น behaviour สำหรับสร้าง data processing pipeline ที่มี **back-pressure** คือ consumer ควบคุมว่าต้องการข้อมูลมากแค่ไหน ป้องกัน producer ส่งข้อมูลเร็วเกินกว่า consumer จะรับได้

```
Producer → ProducerConsumer → Consumer
   ↑                              |
   |______ demand (back-pressure)_|
```

เพิ่ม dependency ใน `mix.exs`:

```elixir
defp deps do
  [
    {:gen_stage, "~> 1.2"},
    {:flow, "~> 1.2"}
  ]
end
```

---

## 2. Producer — แหล่งข้อมูล

Producer ทำหน้าที่ผลิตข้อมูล ตอบสนองต่อ demand จาก downstream

```elixir
defmodule MyApp.NumberProducer do
  use GenStage

  def start_link(initial) do
    GenStage.start_link(__MODULE__, initial, name: __MODULE__)
  end

  def init(initial) do
    # :producer บอกว่า stage นี้คือ producer
    {:producer, initial}
  end

  # handle_demand เรียกเมื่อ downstream ต้องการข้อมูล
  # demand คือจำนวนที่ consumer ขอมา
  def handle_demand(demand, state) when demand > 0 do
    events = Enum.to_list(state..(state + demand - 1))
    {:noreply, events, state + demand}
  end
end
```

### Producer จาก Database

```elixir
defmodule MyApp.DatabaseProducer do
  use GenStage

  def start_link(opts) do
    GenStage.start_link(__MODULE__, opts, name: __MODULE__)
  end

  def init(opts) do
    state = %{
      offset: 0,
      batch_size: Keyword.get(opts, :batch_size, 100)
    }
    {:producer, state}
  end

  def handle_demand(demand, state) do
    limit = min(demand, state.batch_size)

    records =
      MyApp.Repo.all(
        from r in MyApp.Record,
          where: r.id > ^state.offset,
          limit: ^limit,
          order_by: r.id
      )

    new_offset =
      case List.last(records) do
        nil -> state.offset
        record -> record.id
      end

    {:noreply, records, %{state | offset: new_offset}}
  end
end
```

---

## 3. Consumer — ผู้บริโภคข้อมูล

Consumer รับข้อมูลจาก upstream และประมวลผล ส่ง demand กลับไปขอข้อมูลเพิ่ม

```elixir
defmodule MyApp.PrinterConsumer do
  use GenStage

  def start_link(opts \\ []) do
    GenStage.start_link(__MODULE__, opts, name: __MODULE__)
  end

  def init(_opts) do
    # subscribe_to บอกว่าจะรับข้อมูลจากใคร
    {:consumer, :ok, subscribe_to: [MyApp.NumberProducer]}
  end

  def handle_events(events, _from, state) do
    Enum.each(events, fn event ->
      IO.puts("ประมวลผล: #{event}")
    end)

    # consumer ต้อง return {:noreply, [], state} เสมอ
    {:noreply, [], state}
  end
end
```

### Consumer พร้อม Back-pressure Control

```elixir
defmodule MyApp.ThrottledConsumer do
  use GenStage

  def init(_opts) do
    # max_demand และ min_demand ควบคุม back-pressure
    subscription = [
      to: MyApp.DatabaseProducer,
      max_demand: 10,   # ขอได้สูงสุด 10 items ต่อครั้ง
      min_demand: 5     # ขอใหม่เมื่อเหลือต่ำกว่า 5
    ]
    {:consumer, %{processed: 0}, subscribe_to: [subscription]}
  end

  def handle_events(events, _from, state) do
    events
    |> Enum.each(&process_event/1)

    new_state = %{state | processed: state.processed + length(events)}
    {:noreply, [], new_state}
  end

  defp process_event(event) do
    # จำลองการประมวลผลที่ใช้เวลา
    Process.sleep(10)
    IO.puts("ประมวลผลแล้ว: #{inspect(event)}")
  end
end
```

---

## 4. ProducerConsumer — ตัวกลาง Transform

ProducerConsumer รับข้อมูลจาก upstream แล้ว transform ก่อนส่งต่อ downstream

```elixir
defmodule MyApp.TransformStage do
  use GenStage

  def start_link(_opts) do
    GenStage.start_link(__MODULE__, :ok, name: __MODULE__)
  end

  def init(:ok) do
    {:producer_consumer, :ok,
     subscribe_to: [{MyApp.DatabaseProducer, max_demand: 50}]}
  end

  def handle_events(events, _from, state) do
    transformed =
      events
      |> Enum.filter(&valid?/1)
      |> Enum.map(&transform/1)

    # return ข้อมูลที่ transform แล้วเพื่อส่ง downstream
    {:noreply, transformed, state}
  end

  defp valid?(record), do: not is_nil(record.email)

  defp transform(record) do
    %{
      id: record.id,
      email: String.downcase(record.email),
      name: String.trim(record.name),
      processed_at: DateTime.utc_now()
    }
  end
end
```

### Pipeline หลายขั้นตอน

```elixir
# เริ่ม supervisor ที่จัดการ pipeline ทั้งหมด
defmodule MyApp.PipelineSupervisor do
  use Supervisor

  def start_link(_opts) do
    Supervisor.start_link(__MODULE__, :ok, name: __MODULE__)
  end

  def init(:ok) do
    children = [
      MyApp.DatabaseProducer,
      MyApp.TransformStage,
      # สร้าง consumer หลายตัวสำหรับ parallel processing
      Supervisor.child_spec(
        {MyApp.ThrottledConsumer, []},
        id: :consumer_1
      ),
      Supervisor.child_spec(
        {MyApp.ThrottledConsumer, []},
        id: :consumer_2
      ),
    ]

    Supervisor.init(children, strategy: :one_for_one)
  end
end
```

---

## 5. Back-pressure Mechanism

Back-pressure ทำงานอย่างไร:

```
1. Consumer เริ่มต้นส่ง demand = max_demand ไปยัง Producer
2. Producer ส่ง events ตาม demand
3. Consumer ประมวลผล events
4. เมื่อ pending events < min_demand → Consumer ส่ง demand เพิ่ม
5. Producer ส่ง events อีกครั้ง
```

```elixir
defmodule MyApp.BackpressureDemo do
  use GenStage

  def init(_opts) do
    {:consumer, %{count: 0},
     subscribe_to: [
       {MyApp.FastProducer,
        # กำหนด buffer เมื่อ producer เร็วกว่า consumer
        max_demand: 100,
        min_demand: 10}
     ]}
  end

  def handle_events(events, _from, state) do
    # consumer ช้า — ทำให้เกิด back-pressure
    Enum.each(events, fn event ->
      Process.sleep(100)  # simulate slow processing
      IO.puts("ประมวลผล #{event}")
    end)

    {:noreply, [], %{state | count: state.count + length(events)}}
  end
end
```

---

## 6. Flow — Parallel Processing

**Flow** สร้าง concurrent pipeline บน GenStage โดยอัตโนมัติ เหมาะสำหรับการประมวลผลข้อมูลขนาดใหญ่

```elixir
defmodule MyApp.WordCounter do
  def count_words_in_files(file_paths) do
    file_paths
    |> Flow.from_enumerable(max_demand: 10)
    |> Flow.flat_map(fn path ->
      path
      |> File.stream!()
      |> Stream.flat_map(&String.split/1)
    end)
    |> Flow.partition()   # แจกจ่ายข้อมูลสู่หลาย workers โดย hash
    |> Flow.reduce(fn -> %{} end, fn word, acc ->
      Map.update(acc, word, 1, &(&1 + 1))
    end)
    |> Enum.into(%{})
  end
end
```

### Flow Window สำหรับ Time-based Processing

```elixir
defmodule MyApp.MetricsFlow do
  def process_events(events_stream) do
    events_stream
    |> Flow.from_enumerable()
    |> Flow.partition(key: {:key, :user_id})
    # สะสมข้อมูลทุก 60 วินาที
    |> Flow.window(Flow.Window.periodic(60, :second))
    |> Flow.reduce(fn -> %{count: 0, total: 0} end, fn event, acc ->
      %{acc | count: acc.count + 1, total: acc.total + event.value}
    end)
    |> Flow.emit(:state)
    |> Flow.map(fn {user_id, stats} ->
      %{
        user_id: user_id,
        average: stats.total / stats.count,
        count: stats.count
      }
    end)
    |> Enum.to_list()
  end
end
```

---

## 7. ประมวลผลไฟล์ CSV ด้วย Flow

```elixir
defmodule MyApp.CSVProcessor do
  # เพิ่ม {:nimble_csv, "~> 1.2"} ใน deps
  alias NimbleCSV.RFC4180, as: CSV

  def process_large_csv(file_path) do
    file_path
    |> File.stream!(read_ahead: 100_000)   # read_ahead เพิ่ม performance
    |> CSV.parse_stream()
    |> Flow.from_enumerable(max_demand: 1000)
    |> Flow.map(&parse_row/1)
    |> Flow.filter(&valid_row?/1)
    |> Flow.partition(stages: System.schedulers_online())
    |> Flow.reduce(fn -> [] end, fn row, acc -> [row | acc] end)
    |> Flow.emit(:state)
    |> Stream.flat_map(& &1)
    |> Stream.each(&save_to_database/1)
    |> Stream.run()
  end

  defp parse_row([date, user_id, amount, category]) do
    %{
      date: Date.from_iso8601!(date),
      user_id: String.to_integer(user_id),
      amount: Decimal.new(amount),
      category: category
    }
  rescue
    _ -> nil
  end

  defp valid_row?(nil), do: false
  defp valid_row?(%{amount: amount}), do: Decimal.positive?(amount)

  defp save_to_database(row) do
    MyApp.Repo.insert!(
      MyApp.Transaction.changeset(%MyApp.Transaction{}, row),
      on_conflict: :nothing
    )
  end

  # ดู progress ระหว่างประมวลผล
  def process_with_progress(file_path) do
    total_lines = count_lines(file_path)
    counter = :counters.new(1, [])

    file_path
    |> File.stream!()
    |> CSV.parse_stream()
    |> Flow.from_enumerable()
    |> Flow.map(fn row ->
      :counters.add(counter, 1, 1)
      current = :counters.get(counter, 1)

      if rem(current, 1000) == 0 do
        pct = Float.round(current / total_lines * 100, 1)
        IO.puts("ประมวลผลแล้ว #{current}/#{total_lines} (#{pct}%)")
      end

      parse_row(row)
    end)
    |> Flow.filter(&valid_row?/1)
    |> Enum.to_list()
  end

  defp count_lines(file_path) do
    file_path
    |> File.stream!()
    |> Enum.count()
  end
end
```

---

## 8. Metrics Aggregation Pipeline

```elixir
defmodule MyApp.MetricsPipeline do
  use GenStage

  # Producer ที่รับ metrics แบบ real-time
  defmodule Producer do
    use GenStage

    def start_link(_opts) do
      GenStage.start_link(__MODULE__, :queue.new(), name: __MODULE__)
    end

    # API สำหรับส่ง metric เข้า pipeline
    def push_metric(metric) do
      GenServer.cast(__MODULE__, {:push, metric})
    end

    def init(queue) do
      {:producer, {queue, 0}}
    end

    def handle_cast({:push, metric}, {queue, pending_demand}) do
      queue = :queue.in(metric, queue)
      dispatch_events(queue, pending_demand, [])
    end

    def handle_demand(demand, {queue, pending_demand}) do
      dispatch_events(queue, pending_demand + demand, [])
    end

    defp dispatch_events(queue, demand, events) when demand > 0 do
      case :queue.out(queue) do
        {{:value, event}, queue} ->
          dispatch_events(queue, demand - 1, [event | events])
        {:empty, queue} ->
          {:noreply, Enum.reverse(events), {queue, demand}}
      end
    end

    defp dispatch_events(queue, 0, events) do
      {:noreply, Enum.reverse(events), {queue, 0}}
    end
  end

  # Aggregator stage
  defmodule Aggregator do
    use GenStage

    def start_link(_opts) do
      GenStage.start_link(__MODULE__, %{}, name: __MODULE__)
    end

    def init(state) do
      {:producer_consumer, state,
       subscribe_to: [{Producer, max_demand: 100}]}
    end

    def handle_events(metrics, _from, acc) do
      new_acc =
        Enum.reduce(metrics, acc, fn metric, acc ->
          Map.update(acc, metric.name, [metric.value], &[metric.value | &1])
        end)

      # ส่ง aggregate ทุก 5 วินาทีผ่าน timer
      {:noreply, [], new_acc}
    end

    def handle_info(:flush, acc) do
      aggregated =
        Enum.map(acc, fn {name, values} ->
          %{
            name: name,
            count: length(values),
            sum: Enum.sum(values),
            avg: Enum.sum(values) / length(values),
            min: Enum.min(values),
            max: Enum.max(values)
          }
        end)

      Process.send_after(self(), :flush, 5_000)
      {:noreply, aggregated, %{}}
    end
  end
end
```

---

## 9. Rate-Controlled Producer

ควบคุมความเร็วของ producer เพื่อไม่ให้ส่งข้อมูลเร็วเกินไป

```elixir
defmodule MyApp.RateLimitedProducer do
  use GenStage

  @rate_limit 100  # items per second

  def start_link(source) do
    GenStage.start_link(__MODULE__, source, name: __MODULE__)
  end

  def init(source) do
    state = %{
      source: source,
      queue: :queue.new(),
      demand: 0,
      tokens: @rate_limit,
      last_refill: System.monotonic_time(:millisecond)
    }

    # เติม tokens ทุก 1 วินาที
    Process.send_after(self(), :refill_tokens, 1_000)

    {:producer, state}
  end

  def handle_demand(demand, state) do
    state = %{state | demand: state.demand + demand}
    dispatch(state)
  end

  def handle_info(:refill_tokens, state) do
    Process.send_after(self(), :refill_tokens, 1_000)
    state = %{state | tokens: @rate_limit}
    dispatch(state)
  end

  defp dispatch(%{tokens: 0} = state) do
    {:noreply, [], state}
  end

  defp dispatch(%{demand: 0} = state) do
    {:noreply, [], state}
  end

  defp dispatch(state) do
    to_dispatch = min(state.tokens, state.demand)
    events = fetch_events(state.source, to_dispatch)
    actual = length(events)

    new_state = %{state |
      tokens: state.tokens - actual,
      demand: state.demand - actual
    }

    {:noreply, events, new_state}
  end

  defp fetch_events(source, count) do
    Enum.take(source, count)
  end
end
```

---

## สรุป

```
GenStage & Flow Architecture
├── GenStage Stages
│   ├── Producer          → ผลิตข้อมูล, ตอบสนอง demand
│   ├── ProducerConsumer  → รับและ transform ข้อมูล
│   └── Consumer          → รับและประมวลผล, ส่ง demand กลับ
│
├── Back-pressure
│   ├── max_demand        → ขอสูงสุดต่อครั้ง
│   ├── min_demand        → ขอเพิ่มเมื่อต่ำกว่า threshold
│   └── Consumer controls → ป้องกัน overflow
│
├── Flow
│   ├── from_enumerable   → สร้าง flow จาก list/stream
│   ├── partition         → แจกจ่ายสู่หลาย workers
│   ├── map/filter/reduce → standard transformations
│   └── Window            → time/count-based aggregation
│
└── Use Cases
    ├── CSV processing    → large file parallel processing
    ├── Event streaming   → real-time metrics
    └── Rate limiting     → token bucket algorithm
```

---

*ก่อนหน้า: [Part 46 - LiveView Advanced](part_46.md) | ต่อไป: [Part 48 - Distributed Elixir](part_48.md)*
