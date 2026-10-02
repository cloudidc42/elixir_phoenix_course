# Part 59: Stream Processing (การประมวลผลข้อมูลแบบ Stream)

## เป้าหมายการเรียนรู้
- เข้าใจความแตกต่างระหว่าง Elixir Stream และ Enum
- ใช้ Lazy Evaluation เพื่อประหยัด memory
- ประมวลผลไฟล์ขนาดใหญ่แบบ line by line
- สร้าง custom stream ด้วย Stream.resource
- ใช้ Broadway สำหรับ data ingestion pipelines
- จัดการ backpressure ใน stream processing

---

## 1. Enum vs Stream: ความแตกต่างพื้นฐาน

**Enum** ประมวลผลทุกอย่างทันที (eager evaluation) และคืนค่า list กลับมา
**Stream** เป็น lazy — ไม่ทำงานจริงจนกว่าจะ "materialize"

```elixir
# Enum: สร้าง intermediate lists ทุกขั้นตอน (ใช้ memory มาก)
result = 1..1_000_000
|> Enum.map(fn x -> x * 2 end)      # สร้าง list 1 ล้านตัว
|> Enum.filter(fn x -> rem(x, 3) == 0 end)  # สร้าง list ใหม่อีกครั้ง
|> Enum.take(10)                     # เอาแค่ 10 ตัว

# Stream: ไม่สร้าง intermediate lists
result = 1..1_000_000
|> Stream.map(fn x -> x * 2 end)    # แค่ define transformation
|> Stream.filter(fn x -> rem(x, 3) == 0 end)  # เพิ่ม filter เข้า pipeline
|> Enum.take(10)                     # ประมวลผลจริง หยุดเมื่อได้ 10 ตัว
```

เปรียบเทียบ memory usage:
```elixir
# วัด memory ด้วย :erlang.memory(:processes)
before_memory = :erlang.memory(:total)

# Enum: ใช้ ~8MB สำหรับ 1 ล้านตัวเลข
_eager = 1..1_000_000 |> Enum.map(& &1 * 2) |> Enum.filter(&(rem(&1, 6) == 0))

after_enum = :erlang.memory(:total)
IO.puts("Enum used: #{(after_enum - before_memory) / 1024 / 1024} MB")

# Stream: ใช้ memory น้อยมาก (แค่ element ที่กำลัง process)
_lazy = 1..1_000_000 |> Stream.map(& &1 * 2) |> Stream.filter(&(rem(&1, 6) == 0))
# ยังไม่ได้ทำงาน!
```

---

## 2. Stream Functions ที่ใช้บ่อย

```elixir
# Stream.map - transform แต่ละ element
Stream.map(1..10, fn x -> x * x end) |> Enum.to_list()
# [1, 4, 9, 16, 25, 36, 49, 64, 81, 100]

# Stream.filter - กรอง elements
Stream.filter(1..20, fn x -> rem(x, 2) == 0 end) |> Enum.to_list()
# [2, 4, 6, 8, 10, 12, 14, 16, 18, 20]

# Stream.take - เอาแค่ N ตัวแรก
Stream.iterate(0, &(&1 + 1)) |> Stream.take(5) |> Enum.to_list()
# [0, 1, 2, 3, 4]

# Stream.drop - ข้ามไป N ตัว
1..10 |> Stream.drop(3) |> Enum.to_list()
# [4, 5, 6, 7, 8, 9, 10]

# Stream.chunk_every - แบ่งเป็น chunks
1..10 |> Stream.chunk_every(3) |> Enum.to_list()
# [[1, 2, 3], [4, 5, 6], [7, 8, 9], [10]]

# Stream.flat_map - map แล้ว flatten
1..5 |> Stream.flat_map(fn x -> [x, x * 10] end) |> Enum.to_list()
# [1, 10, 2, 20, 3, 30, 4, 40, 5, 50]

# Stream.with_index - เพิ่ม index
~w(a b c d) |> Stream.with_index() |> Enum.to_list()
# [{"a", 0}, {"b", 1}, {"c", 2}, {"d", 3}]

# Stream.scan - accumulate ค่า (เหมือน reduce แต่ emit ทุก step)
1..5 |> Stream.scan(0, &(&1 + &2)) |> Enum.to_list()
# [1, 3, 6, 10, 15]

# Stream.zip - รวม 2 streams เข้าด้วยกัน
Stream.zip(1..5, ~w(a b c d e)) |> Enum.to_list()
# [{1, "a"}, {2, "b"}, {3, "c"}, {4, "d"}, {5, "e"}]
```

---

## 3. ประมวลผลไฟล์ขนาดใหญ่แบบ Line by Line

```elixir
# lib/file_processor.ex
defmodule FileProcessor do
  @moduledoc """
  ตัวอย่างการประมวลผลไฟล์ขนาดใหญ่ด้วย Stream
  ไม่โหลดไฟล์ทั้งหมดเข้า memory
  """

  # อ่านและประมวลผลไฟล์ CSV ขนาดใหญ่
  def process_large_csv(file_path) do
    file_path
    |> File.stream!()              # เปิด stream ทีละบรรทัด
    |> Stream.map(&String.trim/1)  # ลบ whitespace
    |> Stream.reject(&(&1 == ""))  # ข้ามบรรทัดว่าง
    |> Stream.drop(1)              # ข้าม header row
    |> Stream.map(&parse_csv_line/1)
    |> Stream.filter(&valid_record?/1)
    |> Enum.to_list()
  end

  # นับจำนวนบรรทัดโดยไม่โหลดทั้งหมด
  def count_lines(file_path) do
    file_path
    |> File.stream!()
    |> Enum.reduce(0, fn _line, acc -> acc + 1 end)
  end

  # หา unique values จากคอลัมน์หนึ่ง
  def unique_values_in_column(file_path, column_index) do
    file_path
    |> File.stream!()
    |> Stream.drop(1)  # ข้าม header
    |> Stream.map(fn line ->
      line
      |> String.trim()
      |> String.split(",")
      |> Enum.at(column_index)
    end)
    |> Stream.uniq()
    |> Enum.to_list()
  end

  # Process ทีละ batch และบันทึกลง DB
  def import_csv_to_db(file_path) do
    file_path
    |> File.stream!()
    |> Stream.drop(1)  # ข้าม header
    |> Stream.map(&parse_csv_line/1)
    |> Stream.filter(&valid_record?/1)
    |> Stream.chunk_every(1000)  # batch ทีละ 1000 rows
    |> Stream.each(fn batch ->
      MyApp.Repo.insert_all(MyApp.Record, batch)
    end)
    |> Stream.run()  # เรียก run() เพื่อ materialize โดยไม่ return list
  end

  # ประมวลผลหลายไฟล์พร้อมกัน
  def process_multiple_files(file_paths) do
    file_paths
    |> Stream.flat_map(fn path ->
      path |> File.stream!() |> Stream.map(&{path, &1})
    end)
    |> Stream.map(fn {path, line} ->
      {path, process_line(line)}
    end)
    |> Enum.group_by(fn {path, _} -> path end)
  end

  defp parse_csv_line(line) do
    line
    |> String.trim()
    |> String.split(",")
    |> case do
      [id, name, amount] ->
        %{
          id: String.to_integer(id),
          name: name,
          amount: String.to_float(amount)
        }
      _ -> nil
    end
  end

  defp valid_record?(nil), do: false
  defp valid_record?(record), do: record.amount > 0

  defp process_line(line), do: String.trim(line)
end
```

```elixir
# อ่านไฟล์ JSON หลายๆ บรรทัด (JSONL format)
defmodule JsonlProcessor do
  def stream_jsonl(file_path) do
    file_path
    |> File.stream!()
    |> Stream.map(&String.trim/1)
    |> Stream.reject(&(&1 == ""))
    |> Stream.map(fn line ->
      case Jason.decode(line) do
        {:ok, data} -> {:ok, data}
        {:error, error} -> {:error, {line, error}}
      end
    end)
  end

  def process_valid_only(file_path) do
    file_path
    |> stream_jsonl()
    |> Stream.filter(fn
      {:ok, _} -> true
      {:error, _} -> false
    end)
    |> Stream.map(fn {:ok, data} -> data end)
    |> Enum.to_list()
  end
end
```

---

## 4. Stream.resource สำหรับ Custom Sources

`Stream.resource/3` ช่วยสร้าง Stream จาก resource ใดๆ เช่น database cursor, API pagination, queue

```elixir
# สร้าง stream จาก database cursor
defmodule DatabaseStream do
  @moduledoc """
  Stream records จาก database โดยไม่โหลดทั้งหมดเข้า memory
  """

  def stream_records(query, batch_size \\ 100) do
    Stream.resource(
      # ฟังก์ชัน start: สร้าง initial state
      fn -> {query, 0} end,

      # ฟังก์ชัน next: ดึง next batch
      fn
        {_query, :done} ->
          {:halt, :done}

        {query, offset} ->
          records = MyApp.Repo.all(
            from r in query,
            limit: ^batch_size,
            offset: ^offset
          )

          case records do
            [] -> {:halt, :done}
            records ->
              {records, {query, offset + batch_size}}
          end
      end,

      # ฟังก์ชัน cleanup: ทำความสะอาด resources
      fn _state -> :ok end
    )
  end
end

# ใช้งาน: Stream through 1 ล้าน records โดยใช้ memory แค่ 100 records ต่อครั้ง
DatabaseStream.stream_records(MyApp.Order)
|> Stream.filter(fn order -> order.status == :pending end)
|> Stream.map(fn order -> process_order(order) end)
|> Stream.run()
```

```elixir
# สร้าง stream จาก paginated API
defmodule ApiPaginationStream do
  def stream_all_pages(url, params \\ %{}) do
    Stream.resource(
      fn -> {url, Map.put(params, :page, 1)} end,

      fn
        :done -> {:halt, :done}

        {url, params} ->
          case fetch_page(url, params) do
            {:ok, %{data: [], next_page: nil}} ->
              {:halt, :done}

            {:ok, %{data: items, next_page: nil}} ->
              {items, :done}

            {:ok, %{data: items, next_page: next}} ->
              {items, {url, Map.put(params, :page, next)}}

            {:error, _reason} ->
              {:halt, :done}
          end
      end,

      fn _ -> :ok end
    )
  end

  defp fetch_page(url, params) do
    # HTTP call to API
    case HTTPoison.get(url, [], params: params) do
      {:ok, %{body: body}} ->
        {:ok, Jason.decode!(body, keys: :atoms)}
      error -> error
    end
  end
end

# ใช้งาน: stream ข้อมูลทุก page โดยอัตโนมัติ
ApiPaginationStream.stream_all_pages("https://api.example.com/orders")
|> Stream.each(fn order -> save_order(order) end)
|> Stream.run()
```

```elixir
# Stream จาก GenServer/Queue
defmodule QueueStream do
  def stream_from_queue(queue_pid) do
    Stream.resource(
      fn -> queue_pid end,

      fn pid ->
        case GenServer.call(pid, :pop, 5_000) do
          {:ok, item} -> {[item], pid}
          :empty -> {:halt, pid}
          {:error, :timeout} -> {:halt, pid}
        end
      end,

      fn _pid -> :ok end
    )
  end
end
```

---

## 5. Concurrent Stream Processing

```elixir
# lib/concurrent_processor.ex
defmodule ConcurrentProcessor do
  @moduledoc """
  ประมวลผล stream แบบ concurrent ด้วย Task
  """

  # Process items concurrently ด้วย Task.async_stream
  def process_concurrently(items, worker_fn, opts \\ []) do
    max_concurrency = Keyword.get(opts, :max_concurrency, System.schedulers_online())
    timeout = Keyword.get(opts, :timeout, 5_000)

    items
    |> Task.async_stream(
      worker_fn,
      max_concurrency: max_concurrency,
      timeout: timeout,
      on_timeout: :kill_task
    )
    |> Stream.filter(fn
      {:ok, _} -> true
      {:exit, _} -> false
    end)
    |> Stream.map(fn {:ok, result} -> result end)
    |> Enum.to_list()
  end

  # ตัวอย่าง: download หลาย URLs พร้อมกัน
  def download_all(urls) do
    urls
    |> Task.async_stream(
      fn url ->
        {:ok, response} = HTTPoison.get(url)
        {url, response.body}
      end,
      max_concurrency: 10,
      timeout: 30_000
    )
    |> Enum.into(%{}, fn {:ok, {url, body}} -> {url, body} end)
  end

  # Process ไฟล์ขนาดใหญ่แบบ parallel
  def parallel_file_processing(file_path) do
    file_path
    |> File.stream!()
    |> Stream.chunk_every(100)  # แบ่งเป็น chunks ขนาด 100 บรรทัด
    |> Task.async_stream(
      fn chunk ->
        chunk
        |> Enum.map(&process_line/1)
        |> Enum.filter(&valid?/1)
      end,
      max_concurrency: 4
    )
    |> Stream.flat_map(fn {:ok, results} -> results end)
    |> Enum.to_list()
  end

  defp process_line(line), do: String.trim(line)
  defp valid?(line), do: String.length(line) > 0
end
```

---

## 6. Broadway สำหรับ Data Ingestion Pipelines

Broadway เป็น library สำหรับสร้าง production-grade data pipelines ที่ handle backpressure และ fault tolerance โดยอัตโนมัติ

```elixir
# mix.exs - เพิ่ม Broadway
{:broadway, "~> 1.0"},
{:broadway_rabbitmq, "~> 0.7"},  # หรือ broadway_sqs, broadway_kafka
```

```elixir
# lib/my_app/order_processor.ex
defmodule MyApp.OrderProcessor do
  @moduledoc """
  Broadway pipeline สำหรับประมวลผล orders จาก RabbitMQ
  """

  use Broadway

  alias Broadway.Message

  def start_link(_opts) do
    Broadway.start_link(__MODULE__,
      name: __MODULE__,
      producer: [
        module: {
          BroadwayRabbitMQ.Producer,
          queue: "orders",
          connection: [
            host: "localhost",
            port: 5672,
            username: "guest",
            password: "guest"
          ],
          qos: [prefetch_count: 50]  # จำนวน message ที่ fetch ต่อครั้ง
        },
        concurrency: 1  # producer processes
      ],
      processors: [
        default: [
          concurrency: 10,  # 10 concurrent workers
          min_demand: 5,
          max_demand: 20
        ]
      ],
      batchers: [
        database: [
          concurrency: 3,
          batch_size: 100,       # batch 100 records
          batch_timeout: 2_000   # หรือ flush ทุก 2 วินาที
        ],
        analytics: [
          concurrency: 1,
          batch_size: 500,
          batch_timeout: 5_000
        ]
      ]
    )
  end

  # ประมวลผล message แต่ละตัว
  @impl true
  def handle_message(_processor, message, _context) do
    order = message.data |> Jason.decode!() |> atomize_keys()

    case validate_order(order) do
      {:ok, validated_order} ->
        message
        |> Message.put_data(validated_order)
        |> Message.put_batcher(:database)   # ส่งไป database batcher
        |> tap(fn _ ->
          if order[:type] == "premium" do
            # Premium orders ส่งไป analytics ด้วย
            Message.put_batcher(message, :analytics)
          end
        end)

      {:error, reason} ->
        Message.failed(message, reason)
    end
  end

  # Handle batch ที่จะบันทึกลง database
  @impl true
  def handle_batch(:database, messages, _batch_info, _context) do
    orders = Enum.map(messages, & &1.data)

    case MyApp.Repo.insert_all(MyApp.Order, orders, on_conflict: :nothing) do
      {count, _} ->
        require Logger
        Logger.info("Inserted #{count} orders to database")
        messages

      _ ->
        Enum.map(messages, &Message.failed(&1, "Database insert failed"))
    end
  end

  # Handle batch สำหรับ analytics
  @impl true
  def handle_batch(:analytics, messages, _batch_info, _context) do
    events = Enum.map(messages, fn msg ->
      %{
        order_id: msg.data.id,
        event_type: "order_created",
        timestamp: DateTime.utc_now()
      }
    end)

    MyApp.Analytics.track_batch(events)
    messages
  end

  defp validate_order(order) do
    if order[:customer_id] && order[:items] do
      {:ok, order}
    else
      {:error, "Missing required fields"}
    end
  end

  defp atomize_keys(map) when is_map(map) do
    Map.new(map, fn {k, v} -> {String.to_atom(k), v} end)
  end
end
```

---

## 7. Handling Backpressure

Backpressure คือกลไกที่ทำให้ producer หยุดส่งข้อมูลเมื่อ consumer ยังประมวลผลไม่ทัน

```elixir
# lib/my_app/backpressure_demo.ex
defmodule MyApp.BackpressureDemo do
  @moduledoc """
  ตัวอย่างการจัดการ backpressure ด้วย GenStage
  """

  # Producer: ส่งข้อมูลตาม demand
  defmodule Producer do
    use GenStage

    def start_link(items) do
      GenStage.start_link(__MODULE__, items)
    end

    def init(items) do
      {:producer, items}
    end

    def handle_demand(demand, state) when demand > 0 do
      # เอาแค่ demand ที่ consumer ขอ
      {to_emit, remaining} = Enum.split(state, demand)
      {:noreply, to_emit, remaining}
    end
  end

  # Consumer: ประมวลผลช้า (simulate heavy work)
  defmodule SlowConsumer do
    use GenStage

    def start_link(opts \\ []) do
      GenStage.start_link(__MODULE__, opts)
    end

    def init(_opts) do
      {:consumer, %{processed: 0}}
    end

    def handle_events(events, _from, state) do
      Enum.each(events, fn event ->
        # Simulate slow processing
        Process.sleep(100)
        IO.puts("Processing: #{event}")
      end)

      {:noreply, [], %{state | processed: state.processed + length(events)}}
    end
  end
end

# ใช้งาน: GenStage จะ control flow โดยอัตโนมัติ
{:ok, producer} = MyApp.BackpressureDemo.Producer.start_link(1..1000)
{:ok, consumer} = MyApp.BackpressureDemo.SlowConsumer.start_link()
GenStage.sync_subscribe(consumer, to: producer, max_demand: 5, min_demand: 2)
```

---

## 8. Stream Transformation Patterns

```elixir
# lib/stream_patterns.ex
defmodule StreamPatterns do

  # Pattern 1: Sliding window
  def sliding_window(stream, size) do
    stream
    |> Stream.chunk_every(size, 1, :discard)
  end

  # ตัวอย่าง: คำนวณ moving average
  def moving_average(numbers, window_size) do
    numbers
    |> sliding_window(window_size)
    |> Stream.map(fn window ->
      Enum.sum(window) / length(window)
    end)
  end

  # Pattern 2: Throttle/Rate limiting
  def throttle(stream, delay_ms) do
    stream
    |> Stream.each(fn _ -> Process.sleep(delay_ms) end)
  end

  # Pattern 3: Retry on error
  def with_retry(stream, max_retries \\ 3) do
    stream
    |> Stream.flat_map(fn item ->
      Stream.resource(
        fn -> {item, 0} end,
        fn
          :done -> {:halt, :done}
          {item, attempts} when attempts < max_retries ->
            case process_with_possible_error(item) do
              {:ok, result} -> {[result], :done}
              {:error, _} ->
                Process.sleep(100 * (attempts + 1))  # exponential backoff
                {[], {item, attempts + 1}}
            end
          {_item, _attempts} ->
            # Max retries reached
            {[], :done}
        end,
        fn _ -> :ok end
      )
    end)
  end

  # Pattern 4: Partition stream
  def partition(stream, predicate) do
    {lefts, rights} = stream
    |> Enum.split_with(predicate)
    {lefts, rights}
  end

  # Pattern 5: Merge multiple streams
  def merge_streams(streams) do
    streams
    |> Enum.map(&Task.async(fn -> Enum.to_list(&1) end))
    |> Enum.flat_map(&Task.await/1)
  end

  defp process_with_possible_error(item) do
    # Simulate potential failure
    if :rand.uniform() > 0.3 do
      {:ok, item * 2}
    else
      {:error, :random_failure}
    end
  end
end
```

---

## 9. ตัวอย่าง Real-World: Log Analytics Pipeline

```elixir
# lib/log_analytics.ex
defmodule LogAnalytics do
  @moduledoc """
  วิเคราะห์ log files ขนาดใหญ่ด้วย Stream Processing
  """

  # Pattern สำหรับ parse Apache/Nginx access log
  @log_pattern ~r/(\S+) - \S+ \[([^\]]+)\] "(\S+) (\S+) \S+" (\d+) (\d+)/

  def analyze_access_log(log_file_path) do
    log_file_path
    |> File.stream!()
    |> Stream.map(&parse_log_line/1)
    |> Stream.reject(&is_nil/1)
    |> Stream.transform(
      # initial state: สะสม statistics
      fn -> %{total: 0, errors: 0, status_counts: %{}} end,
      # accumulator function
      fn entry, acc ->
        new_acc = %{
          total: acc.total + 1,
          errors: acc.errors + if(entry.status >= 400, do: 1, else: 0),
          status_counts: Map.update(
            acc.status_counts,
            entry.status,
            1,
            & &1 + 1
          )
        }
        {[entry], new_acc}
      end,
      # after_fun: emit final stats
      fn acc -> {[{:stats, acc}], acc} end
    )
    |> Enum.reduce(%{entries: [], stats: nil}, fn
      {:stats, stats}, acc -> %{acc | stats: stats}
      entry, acc -> %{acc | entries: [entry | acc.entries]}
    end)
  end

  # ค้นหา slow requests (> 1 วินาที)
  def find_slow_requests(log_file_path, threshold_ms \\ 1000) do
    log_file_path
    |> File.stream!()
    |> Stream.map(&parse_log_line/1)
    |> Stream.reject(&is_nil/1)
    |> Stream.filter(fn entry -> entry.response_time > threshold_ms end)
    |> Stream.map(fn entry ->
      %{
        timestamp: entry.timestamp,
        path: entry.path,
        status: entry.status,
        response_time: entry.response_time
      }
    end)
    |> Enum.sort_by(& &1.response_time, :desc)
    |> Enum.take(100)  # Top 100 slowest
  end

  # นับ requests per endpoint
  def requests_per_endpoint(log_file_path) do
    log_file_path
    |> File.stream!()
    |> Stream.map(&parse_log_line/1)
    |> Stream.reject(&is_nil/1)
    |> Enum.reduce(%{}, fn entry, acc ->
      Map.update(acc, entry.path, 1, & &1 + 1)
    end)
    |> Enum.sort_by(fn {_path, count} -> count end, :desc)
    |> Enum.take(20)
  end

  defp parse_log_line(line) do
    case Regex.run(@log_pattern, String.trim(line)) do
      [_, ip, timestamp, method, path, status, bytes] ->
        %{
          ip: ip,
          timestamp: timestamp,
          method: method,
          path: path,
          status: String.to_integer(status),
          bytes: String.to_integer(bytes),
          response_time: :rand.uniform(2000)  # ในตัวอย่างนี้ simulate
        }
      _ -> nil
    end
  end
end
```

---

## 10. Testing Stream Processing

```elixir
# test/stream_patterns_test.exs
defmodule StreamPatternsTest do
  use ExUnit.Case

  describe "moving_average/2" do
    test "คำนวณ moving average ถูกต้อง" do
      result = StreamPatterns.moving_average([1, 2, 3, 4, 5], 3)
      |> Enum.to_list()

      assert result == [2.0, 3.0, 4.0]
    end
  end

  describe "sliding_window/2" do
    test "สร้าง windows ถูกต้อง" do
      result = StreamPatterns.sliding_window(1..5, 3) |> Enum.to_list()
      assert result == [[1, 2, 3], [2, 3, 4], [3, 4, 5]]
    end
  end

  test "Stream.resource สร้าง stream จาก list ถูกต้อง" do
    source = [1, 2, 3, 4, 5]

    result = Stream.resource(
      fn -> source end,
      fn
        [] -> {:halt, []}
        [h | t] -> {[h * 2], t}
      end,
      fn _ -> :ok end
    ) |> Enum.to_list()

    assert result == [2, 4, 6, 8, 10]
  end

  test "Task.async_stream ประมวลผล concurrent ถูกต้อง" do
    items = 1..10

    results = items
    |> Task.async_stream(fn x -> x * x end, max_concurrency: 4)
    |> Enum.map(fn {:ok, result} -> result end)
    |> Enum.sort()

    assert results == [1, 4, 9, 16, 25, 36, 49, 64, 81, 100]
  end
end
```

---

## สรุป

```
Stream Processing Decision Tree:

ข้อมูลทั้งหมดอยู่ใน memory แล้ว?
├── ใช่ → Enum (simple, eager)
└── ไม่ → Stream (lazy, memory efficient)
         │
         ├── ต้องการ concurrent processing?
         │   ├── ใช่ → Task.async_stream
         │   └── ไม่ → Stream pipeline
         │
         ├── Custom data source (DB, API, Queue)?
         │   └── Stream.resource
         │
         └── Production pipeline + backpressure?
             └── Broadway

Key Rules:
1. Stream ไม่ทำงานจนกว่าจะ "terminal" (Enum.to_list, Enum.each, etc.)
2. ใช้ Stream.run() เมื่อไม่ต้องการ collect results
3. File.stream! > File.read! สำหรับไฟล์ขนาดใหญ่
4. chunk_every ก่อน insert หลายๆ rows ลง DB
5. Broadway สำหรับ message queues + fault tolerance
```

---

*ก่อนหน้า: [Part 58 - CQRS Pattern](part_58.md) | ต่อไป: [Part 60 - Machine Learning Integration](part_60.md)*
