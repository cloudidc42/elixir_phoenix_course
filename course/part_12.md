# Part 12: Behaviours และ Callbacks

## เป้าหมายการเรียนรู้

หลังจากจบ Part นี้ คุณจะสามารถ:
- เข้าใจความแตกต่างระหว่าง Behaviour และ Protocol
- กำหนด Behaviour ด้วย `@callback` และ `@optional_callbacks`
- Implement Behaviour ด้วย `@behaviour` และ `@impl`
- สร้างระบบ pluggable backends (Database adapter, Cache, Payment gateway)
- ใช้ Behaviour เพื่อ testability และ modularity

---

## 1. Behaviour คืออะไร?

Behaviour คือ contract ที่บอกว่า module ต้อง implement functions อะไรบ้าง

```
Protocol vs Behaviour:
┌─────────────────────────────────────────────────────────────┐
│                     Protocol                                 │
│  - Polymorphism บน DATA TYPES                               │
│  - defimpl แยกตาม type                                      │
│  - ใช้กับ Enum, String.Chars, Inspect ...                   │
├─────────────────────────────────────────────────────────────┤
│                     Behaviour                                │
│  - Contract สำหรับ MODULES                                  │
│  - @behaviour ใน module ที่ implement                        │
│  - ใช้กับ GenServer, Plug, Ecto.Adapter ...                 │
└─────────────────────────────────────────────────────────────┘
```

---

## 2. กำหนด Behaviour

```elixir
defmodule Worker do
  @callback init(args :: any()) :: {:ok, state :: any()} | {:error, reason :: any()}
  @callback process(job :: map(), state :: any()) :: {:ok, result :: any(), new_state :: any()} | {:error, reason :: any()}
  @callback terminate(reason :: any(), state :: any()) :: :ok
end
```

### Callback Types

```elixir
defmodule Logger.Backend do
  # Required callback
  @callback log(level :: atom(), message :: String.t(), metadata :: keyword()) :: :ok

  # Optional callback
  @optional_callbacks [
    flush: 0,
    configure: 1
  ]

  @callback flush() :: :ok
  @callback configure(opts :: keyword()) :: :ok
end
```

---

## 3. @impl Annotation

`@impl` บอก compiler ว่า function นี้ implement callback ของ Behaviour ไหน

```elixir
defmodule MyLogger do
  @behaviour Logger.Backend

  @impl Logger.Backend
  def log(level, message, metadata) do
    timestamp = DateTime.utc_now()
    IO.puts("[#{timestamp}] #{level}: #{message}")
    :ok
  end

  # ถ้าไม่ implement optional callback ก็ไม่ error
end

# ถ้าไม่ใส่ @impl แต่ implement ผิด signature จะไม่ได้รับ warning
# ใส่ @impl แล้ว implement ผิด -> compiler warning!
```

---

## 4. ตัวอย่างจริง: Database Adapter

```elixir
# กำหนด Behaviour
defmodule Database.Adapter do
  @type connection :: any()
  @type query_result :: {:ok, [map()]} | {:error, String.t()}

  @callback connect(opts :: keyword()) :: {:ok, connection()} | {:error, String.t()}
  @callback disconnect(conn :: connection()) :: :ok
  @callback query(conn :: connection(), sql :: String.t(), params :: [any()]) :: query_result()
  @callback transaction(conn :: connection(), fun :: (connection() -> any())) :: {:ok, any()} | {:error, any()}

  @optional_callbacks [
    begin_transaction: 1,
    commit: 1,
    rollback: 1
  ]

  @callback begin_transaction(conn :: connection()) :: :ok | {:error, String.t()}
  @callback commit(conn :: connection()) :: :ok | {:error, String.t()}
  @callback rollback(conn :: connection()) :: :ok | {:error, String.t()}
end

# PostgreSQL Adapter
defmodule Database.PostgresAdapter do
  @behaviour Database.Adapter

  @impl Database.Adapter
  def connect(opts) do
    host = Keyword.get(opts, :host, "localhost")
    port = Keyword.get(opts, :port, 5432)
    database = Keyword.fetch!(opts, :database)
    username = Keyword.fetch!(opts, :username)
    password = Keyword.get(opts, :password, "")

    # ใน production จะใช้ Postgrex library
    # นี่คือ simulation
    IO.puts("Connecting to PostgreSQL: #{host}:#{port}/#{database}")
    {:ok, %{
      host: host,
      port: port,
      database: database,
      username: username,
      connected: true,
      pid: self()
    }}
  end

  @impl Database.Adapter
  def disconnect(conn) do
    IO.puts("Disconnecting from #{conn.database}")
    :ok
  end

  @impl Database.Adapter
  def query(conn, sql, params) do
    IO.puts("Executing: #{sql} with params: #{inspect(params)}")
    # Simulation - ใน production จะส่งไปยัง PostgreSQL จริงๆ
    {:ok, [%{result: "sample data"}]}
  end

  @impl Database.Adapter
  def transaction(conn, fun) do
    with :ok <- begin_transaction(conn),
         result <- fun.(conn),
         :ok <- commit(conn) do
      {:ok, result}
    else
      {:error, reason} ->
        rollback(conn)
        {:error, reason}
    end
  end

  @impl Database.Adapter
  def begin_transaction(conn) do
    IO.puts("BEGIN TRANSACTION on #{conn.database}")
    :ok
  end

  @impl Database.Adapter
  def commit(conn) do
    IO.puts("COMMIT on #{conn.database}")
    :ok
  end

  @impl Database.Adapter
  def rollback(conn) do
    IO.puts("ROLLBACK on #{conn.database}")
    :ok
  end
end

# SQLite Adapter (สำหรับ testing/development)
defmodule Database.SQLiteAdapter do
  @behaviour Database.Adapter

  @impl Database.Adapter
  def connect(opts) do
    path = Keyword.get(opts, :path, ":memory:")
    IO.puts("Opening SQLite: #{path}")
    {:ok, %{path: path, connected: true}}
  end

  @impl Database.Adapter
  def disconnect(_conn) do
    IO.puts("Closing SQLite connection")
    :ok
  end

  @impl Database.Adapter
  def query(_conn, sql, params) do
    IO.puts("SQLite query: #{sql}")
    {:ok, []}
  end

  @impl Database.Adapter
  def transaction(conn, fun) do
    result = fun.(conn)
    {:ok, result}
  end
end

# Database module ที่ใช้ adapter
defmodule Database do
  def connect(adapter, opts) do
    adapter.connect(opts)
  end

  def query(adapter, conn, sql, params \\ []) do
    adapter.query(conn, sql, params)
  end

  def transaction(adapter, conn, fun) do
    adapter.transaction(conn, fun)
  end
end

# ใช้งาน
adapter = Database.PostgresAdapter
{:ok, conn} = Database.connect(adapter, [
  host: "localhost",
  database: "myapp",
  username: "postgres"
])

Database.query(adapter, conn, "SELECT * FROM users WHERE id = $1", [1])

Database.transaction(adapter, conn, fn conn ->
  Database.query(adapter, conn, "INSERT INTO users (name) VALUES ($1)", ["Alice"])
end)
```

---

## 5. ตัวอย่างจริง: Cache Backend

```elixir
defmodule Cache.Backend do
  @type key :: any()
  @type value :: any()
  @type ttl :: non_neg_integer() | :infinity  # seconds

  @callback get(key()) :: {:ok, value()} | {:miss}
  @callback put(key(), value(), ttl()) :: :ok
  @callback delete(key()) :: :ok
  @callback clear() :: :ok
  @callback size() :: non_neg_integer()

  @optional_callbacks [
    get_many: 1,
    put_many: 2,
    stats: 0
  ]

  @callback get_many([key()]) :: %{key() => value()}
  @callback put_many([{key(), value()}], ttl()) :: :ok
  @callback stats() :: map()
end

# In-Memory Cache using ETS
defmodule Cache.ETSBackend do
  @behaviour Cache.Backend

  @table :ets_cache

  def start do
    :ets.new(@table, [:set, :public, :named_table])
    :ok
  end

  @impl Cache.Backend
  def get(key) do
    case :ets.lookup(@table, key) do
      [{^key, value, expires_at}] ->
        if expires_at == :infinity or System.system_time(:second) < expires_at do
          {:ok, value}
        else
          :ets.delete(@table, key)
          {:miss}
        end
      [] -> {:miss}
    end
  end

  @impl Cache.Backend
  def put(key, value, ttl \\ :infinity) do
    expires_at = case ttl do
      :infinity -> :infinity
      seconds -> System.system_time(:second) + seconds
    end
    :ets.insert(@table, {key, value, expires_at})
    :ok
  end

  @impl Cache.Backend
  def delete(key) do
    :ets.delete(@table, key)
    :ok
  end

  @impl Cache.Backend
  def clear do
    :ets.delete_all_objects(@table)
    :ok
  end

  @impl Cache.Backend
  def size do
    :ets.info(@table, :size)
  end

  @impl Cache.Backend
  def stats do
    %{
      size: size(),
      memory: :ets.info(@table, :memory),
      table: @table
    }
  end
end

# Redis Cache Backend (Simulation)
defmodule Cache.RedisBackend do
  @behaviour Cache.Backend

  # ใน production จะใช้ Redix library
  defstruct [:conn]

  def start(opts \\ []) do
    host = Keyword.get(opts, :host, "localhost")
    port = Keyword.get(opts, :port, 6379)
    IO.puts("Connecting to Redis #{host}:#{port}")
    {:ok, %__MODULE__{conn: %{host: host, port: port}}}
  end

  @impl Cache.Backend
  def get(key) do
    # Simulation
    IO.puts("Redis GET #{inspect(key)}")
    {:miss}
  end

  @impl Cache.Backend
  def put(key, value, ttl \\ :infinity) do
    IO.puts("Redis SET #{inspect(key)} EX #{ttl}")
    :ok
  end

  @impl Cache.Backend
  def delete(key) do
    IO.puts("Redis DEL #{inspect(key)}")
    :ok
  end

  @impl Cache.Backend
  def clear do
    IO.puts("Redis FLUSHDB")
    :ok
  end

  @impl Cache.Backend
  def size do
    IO.puts("Redis DBSIZE")
    0
  end
end

# Cache module ที่ใช้ backend
defmodule Cache do
  def get(backend, key), do: backend.get(key)
  def put(backend, key, value, ttl \\ :infinity), do: backend.put(key, value, ttl)
  def delete(backend, key), do: backend.delete(key)
  def clear(backend), do: backend.clear()

  def get_or_fetch(backend, key, fetch_fn, ttl \\ 300) do
    case backend.get(key) do
      {:ok, value} ->
        {:ok, value, :cached}
      {:miss} ->
        case fetch_fn.() do
          {:ok, value} ->
            backend.put(key, value, ttl)
            {:ok, value, :fetched}
          error ->
            error
        end
    end
  end
end

# ใช้งาน
Cache.ETSBackend.start()

Cache.put(Cache.ETSBackend, "user:1", %{name: "Alice", age: 30})
Cache.put(Cache.ETSBackend, "user:2", %{name: "Bob", age: 25}, 60)

Cache.get(Cache.ETSBackend, "user:1")
# {:ok, %{name: "Alice", age: 30}}

Cache.get_or_fetch(Cache.ETSBackend, "user:3", fn ->
  # simulate DB fetch
  {:ok, %{name: "Charlie", age: 35}}
end)
```

---

## 6. ตัวอย่างจริง: Payment Gateway

```elixir
defmodule Payment.Gateway do
  @type amount :: float()
  @type currency :: String.t()
  @type card_info :: map()

  @type charge_result ::
    {:ok, %{transaction_id: String.t(), status: :success}} |
    {:error, %{code: String.t(), message: String.t()}}

  @callback charge(amount(), currency(), card_info(), opts :: keyword()) :: charge_result()
  @callback refund(transaction_id :: String.t(), amount()) :: {:ok, map()} | {:error, map()}
  @callback get_transaction(transaction_id :: String.t()) :: {:ok, map()} | {:error, :not_found}

  @optional_callbacks [
    capture: 2,
    void: 1,
    webhook_verify: 2
  ]

  @callback capture(transaction_id :: String.t(), amount()) :: {:ok, map()} | {:error, map()}
  @callback void(transaction_id :: String.t()) :: {:ok, map()} | {:error, map()}
  @callback webhook_verify(payload :: String.t(), signature :: String.t()) :: :ok | {:error, :invalid_signature}
end

# Stripe Payment Gateway
defmodule Payment.StripeGateway do
  @behaviour Payment.Gateway

  @api_base "https://api.stripe.com/v1"

  @impl Payment.Gateway
  def charge(amount, currency, card_info, opts \\ []) do
    description = Keyword.get(opts, :description, "")
    # ใน production จะ call Stripe API จริงๆ
    IO.puts("Stripe: Charging #{amount} #{currency}")
    {:ok, %{
      transaction_id: "ch_#{:crypto.strong_rand_bytes(16) |> Base.encode16(case: :lower)}",
      status: :success,
      amount: amount,
      currency: currency
    }}
  end

  @impl Payment.Gateway
  def refund(transaction_id, amount) do
    IO.puts("Stripe: Refunding #{amount} for #{transaction_id}")
    {:ok, %{refund_id: "re_#{transaction_id}", status: :success}}
  end

  @impl Payment.Gateway
  def get_transaction(transaction_id) do
    IO.puts("Stripe: Getting transaction #{transaction_id}")
    {:ok, %{id: transaction_id, status: :success}}
  end

  @impl Payment.Gateway
  def capture(transaction_id, amount) do
    IO.puts("Stripe: Capturing #{amount} for #{transaction_id}")
    {:ok, %{transaction_id: transaction_id, captured: true}}
  end

  @impl Payment.Gateway
  def void(transaction_id) do
    IO.puts("Stripe: Voiding #{transaction_id}")
    {:ok, %{transaction_id: transaction_id, voided: true}}
  end

  @impl Payment.Gateway
  def webhook_verify(payload, signature) do
    # ใน production จะ verify HMAC signature
    if String.length(signature) > 0, do: :ok, else: {:error, :invalid_signature}
  end
end

# Test/Mock Gateway สำหรับ testing
defmodule Payment.MockGateway do
  @behaviour Payment.Gateway

  @impl Payment.Gateway
  def charge(_amount, _currency, _card_info, _opts) do
    {:ok, %{
      transaction_id: "test_txn_001",
      status: :success
    }}
  end

  @impl Payment.Gateway
  def refund(transaction_id, _amount) do
    {:ok, %{refund_id: "test_refund_001", original: transaction_id}}
  end

  @impl Payment.Gateway
  def get_transaction(transaction_id) do
    {:ok, %{id: transaction_id, status: :success, test: true}}
  end
end

# Order processing ที่ใช้ Payment Gateway
defmodule Order do
  defstruct [:id, :items, :total, :currency, :status]

  def process(order, gateway, card_info) do
    case gateway.charge(order.total, order.currency, card_info) do
      {:ok, %{transaction_id: txn_id}} ->
        {:ok, %{order | status: :paid}, txn_id}
      {:error, error} ->
        {:error, error}
    end
  end
end

# ใช้งาน - เปลี่ยน gateway ได้ง่าย
gateway = if Mix.env() == :test do
  Payment.MockGateway
else
  Payment.StripeGateway
end

order = %Order{id: 1, items: ["item1"], total: 99.99, currency: "USD", status: :pending}
card = %{number: "4242424242424242", exp_month: 12, exp_year: 2025, cvc: "123"}

Order.process(order, gateway, card)
```

---

## 7. Behaviour ใน GenServer

GenServer เป็นตัวอย่าง Behaviour ที่มีใน Elixir

```elixir
defmodule Counter do
  use GenServer

  # Client API
  def start_link(initial_value \\ 0) do
    GenServer.start_link(__MODULE__, initial_value, name: __MODULE__)
  end

  def increment(amount \\ 1), do: GenServer.cast(__MODULE__, {:increment, amount})
  def decrement(amount \\ 1), do: GenServer.cast(__MODULE__, {:decrement, amount})
  def value, do: GenServer.call(__MODULE__, :value)
  def reset, do: GenServer.cast(__MODULE__, :reset)

  # GenServer Callbacks (Behaviour implementation)
  @impl GenServer
  def init(initial_value) do
    {:ok, initial_value}
  end

  @impl GenServer
  def handle_call(:value, _from, state) do
    {:reply, state, state}
  end

  @impl GenServer
  def handle_cast({:increment, amount}, state) do
    {:noreply, state + amount}
  end

  @impl GenServer
  def handle_cast({:decrement, amount}, state) do
    {:noreply, state - amount}
  end

  @impl GenServer
  def handle_cast(:reset, _state) do
    {:noreply, 0}
  end
end
```

---

## 8. Custom Behaviour สำหรับ Plugins

```elixir
defmodule Plugin do
  @callback name() :: String.t()
  @callback version() :: String.t()
  @callback description() :: String.t()
  @callback execute(input :: map(), config :: map()) :: {:ok, map()} | {:error, String.t()}

  @optional_callbacks [
    validate_config: 1,
    on_load: 1,
    on_unload: 0
  ]

  @callback validate_config(config :: map()) :: :ok | {:error, [String.t()]}
  @callback on_load(config :: map()) :: :ok
  @callback on_unload() :: :ok

  # Helper function ตรวจสอบว่า module implement behaviour นี้ไหม
  def valid_plugin?(module) do
    required_callbacks = [:name, :version, :description, :execute]
    Enum.all?(required_callbacks, fn callback ->
      function_exported?(module, callback, 0) or
      function_exported?(module, callback, 1) or
      function_exported?(module, callback, 2)
    end)
  end
end

# Markdown Plugin
defmodule Plugin.Markdown do
  @behaviour Plugin

  @impl Plugin
  def name, do: "markdown"

  @impl Plugin
  def version, do: "1.0.0"

  @impl Plugin
  def description, do: "Converts markdown to HTML"

  @impl Plugin
  def execute(%{content: content}, _config) do
    # Simplified markdown conversion
    html = content
      |> String.replace(~r/^# (.+)$/m, "<h1>\\1</h1>")
      |> String.replace(~r/^## (.+)$/m, "<h2>\\1</h2>")
      |> String.replace(~r/\*\*(.+?)\*\*/, "<strong>\\1</strong>")
      |> String.replace(~r/\*(.+?)\*/, "<em>\\1</em>")
    {:ok, %{html: html}}
  end

  @impl Plugin
  def validate_config(_config), do: :ok
end

# Word Count Plugin
defmodule Plugin.WordCount do
  @behaviour Plugin

  @impl Plugin
  def name, do: "word_count"

  @impl Plugin
  def version, do: "1.0.0"

  @impl Plugin
  def description, do: "Counts words and characters in text"

  @impl Plugin
  def execute(%{content: content}, _config) do
    words = content |> String.split() |> length()
    chars = String.length(content)
    {:ok, %{word_count: words, char_count: chars}}
  end
end

# Plugin Manager
defmodule PluginManager do
  def run_plugin(plugin_module, input, config \\ %{}) do
    if Plugin.valid_plugin?(plugin_module) do
      plugin_module.execute(input, config)
    else
      {:error, "Invalid plugin: #{plugin_module}"}
    end
  end

  def run_pipeline(plugins, input) do
    Enum.reduce_while(plugins, {:ok, input}, fn plugin, {:ok, current_input} ->
      case plugin.execute(current_input, %{}) do
        {:ok, result} -> {:cont, {:ok, Map.merge(current_input, result)}}
        {:error, reason} -> {:halt, {:error, {plugin.name(), reason}}}
      end
    end)
  end
end

# ใช้งาน
input = %{content: "# Hello World\n\nThis is **bold** and *italic* text."}

Plugin.Markdown.execute(input, %{})
# {:ok, %{html: "<h1>Hello World</h1>\n\n..."}}

PluginManager.run_pipeline([Plugin.Markdown, Plugin.WordCount], input)
# {:ok, %{content: "...", html: "...", word_count: 7, char_count: 45}}
```

---

## 9. @optional_callbacks ในทางปฏิบัติ

```elixir
defmodule Notifier do
  @callback send(recipient :: String.t(), message :: String.t()) :: :ok | {:error, term()}
  @callback supports_attachments?() :: boolean()

  @optional_callbacks [
    send_with_attachment: 3,
    batch_send: 2
  ]

  @callback send_with_attachment(String.t(), String.t(), attachment :: binary()) :: :ok | {:error, term()}
  @callback batch_send([String.t()], String.t()) :: :ok | {:error, term()}

  # ตรวจสอบว่า module implement optional callback ไหม
  def has_batch_support?(module) do
    function_exported?(module, :batch_send, 2)
  end

  def send_notification(module, recipients, message) when is_list(recipients) do
    if has_batch_support?(module) do
      module.batch_send(recipients, message)
    else
      # Fallback: send one by one
      results = Enum.map(recipients, &module.send(&1, message))
      if Enum.all?(results, &(&1 == :ok)), do: :ok, else: {:error, :partial_failure}
    end
  end
end

defmodule EmailNotifier do
  @behaviour Notifier

  @impl Notifier
  def send(recipient, message) do
    IO.puts("Email to #{recipient}: #{message}")
    :ok
  end

  @impl Notifier
  def supports_attachments?, do: true

  @impl Notifier
  def send_with_attachment(recipient, message, attachment) do
    IO.puts("Email with attachment to #{recipient}")
    :ok
  end

  @impl Notifier
  def batch_send(recipients, message) do
    IO.puts("Batch email to #{length(recipients)} recipients")
    :ok
  end
end

defmodule SMSNotifier do
  @behaviour Notifier

  @impl Notifier
  def send(recipient, message) do
    IO.puts("SMS to #{recipient}: #{message}")
    :ok
  end

  @impl Notifier
  def supports_attachments?, do: false
  # ไม่ implement batch_send - OK เพราะเป็น optional
end
```

---

## 10. Exercises

### Exercise 1: Logger Behaviour

```elixir
# สร้าง Logger behaviour และ 3 implementations:
# 1. ConsoleLogger - print ไปที่ console
# 2. FileLogger - เขียนไฟล์
# 3. MemoryLogger - เก็บใน list (สำหรับ testing)

defmodule AppLogger do
  @callback log(level :: :debug | :info | :warning | :error,
                 message :: String.t(),
                 metadata :: keyword()) :: :ok

  @callback flush() :: :ok  # optional

  @optional_callbacks [flush: 0]
end
```

### Exercise 2: Validator Behaviour

```elixir
# สร้าง Validator behaviour สำหรับ validate ข้อมูล
defmodule Validator do
  @type result :: :ok | {:error, [String.t()]}
  @callback validate(data :: map()) :: result()
  @callback rules() :: [String.t()]  # อธิบาย validation rules
end

# Implement:
# 1. UserValidator - validate user registration data
# 2. ProductValidator - validate product data
# 3. OrderValidator - validate order data
```

### Exercise 3: Serializer Behaviour

```elixir
defmodule Serializer do
  @callback serialize(data :: any()) :: String.t()
  @callback deserialize(string :: String.t()) :: {:ok, any()} | {:error, String.t()}
  @callback content_type() :: String.t()
end

# Implement JSONSerializer และ CSVSerializer
```

---

## เฉลย Exercises

### เฉลย Exercise 1

```elixir
defmodule ConsoleLogger do
  @behaviour AppLogger

  @impl AppLogger
  def log(level, message, metadata) do
    timestamp = DateTime.utc_now() |> DateTime.to_iso8601()
    meta_str = Enum.map(metadata, fn {k, v} -> "#{k}=#{v}" end) |> Enum.join(", ")
    IO.puts("[#{timestamp}] [#{String.upcase(to_string(level))}] #{message} #{meta_str}")
    :ok
  end

  @impl AppLogger
  def flush, do: :ok
end

defmodule FileLogger do
  @behaviour AppLogger

  @impl AppLogger
  def log(level, message, metadata) do
    timestamp = DateTime.utc_now() |> DateTime.to_iso8601()
    meta_str = Enum.map(metadata, fn {k, v} -> "#{k}=#{v}" end) |> Enum.join(", ")
    line = "[#{timestamp}] [#{String.upcase(to_string(level))}] #{message} #{meta_str}\n"
    File.write("app.log", line, [:append])
    :ok
  end

  @impl AppLogger
  def flush, do: :ok
end

defmodule MemoryLogger do
  @behaviour AppLogger

  use Agent

  def start_link(_), do: Agent.start_link(fn -> [] end, name: __MODULE__)

  @impl AppLogger
  def log(level, message, metadata) do
    entry = %{
      timestamp: DateTime.utc_now(),
      level: level,
      message: message,
      metadata: metadata
    }
    Agent.update(__MODULE__, fn logs -> [entry | logs] end)
    :ok
  end

  def get_logs, do: Agent.get(__MODULE__, fn logs -> Enum.reverse(logs) end)
  def clear, do: Agent.update(__MODULE__, fn _ -> [] end)
end
```

### เฉลย Exercise 2

```elixir
defmodule UserValidator do
  @behaviour Validator

  @impl Validator
  def rules do
    [
      "name: required, min 2 chars",
      "email: required, valid email format",
      "age: optional, must be 18+",
      "password: required, min 8 chars"
    ]
  end

  @impl Validator
  def validate(%{name: name, email: email, password: password} = data) do
    errors = []
    errors = if String.length(name) < 2, do: ["name must be at least 2 chars" | errors], else: errors
    errors = if not valid_email?(email), do: ["email is invalid" | errors], else: errors
    errors = if String.length(password) < 8, do: ["password must be at least 8 chars" | errors], else: errors
    errors = case Map.get(data, :age) do
      nil -> errors
      age when age < 18 -> ["must be 18 or older" | errors]
      _ -> errors
    end
    if errors == [], do: :ok, else: {:error, Enum.reverse(errors)}
  end
  def validate(_), do: {:error, ["name, email, and password are required"]}

  defp valid_email?(email) do
    String.match?(email, ~r/^[^\s@]+@[^\s@]+\.[^\s@]+$/)
  end
end
```

### เฉลย Exercise 3

```elixir
defmodule JSONSerializer do
  @behaviour Serializer

  @impl Serializer
  def content_type, do: "application/json"

  @impl Serializer
  def serialize(data) do
    Jason.encode!(data)
  rescue
    _ -> "{}"
  end

  @impl Serializer
  def deserialize(string) do
    case Jason.decode(string) do
      {:ok, data} -> {:ok, data}
      {:error, error} -> {:error, "JSON parse error: #{inspect(error)}"}
    end
  end
end

defmodule CSVSerializer do
  @behaviour Serializer

  @impl Serializer
  def content_type, do: "text/csv"

  @impl Serializer
  def serialize(data) when is_list(data) and length(data) > 0 do
    [first | _] = data
    headers = Map.keys(first) |> Enum.join(",")
    rows = Enum.map(data, fn row ->
      Map.values(row) |> Enum.map(&to_string/1) |> Enum.join(",")
    end)
    Enum.join([headers | rows], "\n")
  end
  def serialize(_), do: ""

  @impl Serializer
  def deserialize(string) do
    [headers_line | data_lines] = String.split(string, "\n")
    headers = String.split(headers_line, ",") |> Enum.map(&String.to_atom/1)
    rows = Enum.map(data_lines, fn line ->
      values = String.split(line, ",")
      Enum.zip(headers, values) |> Map.new()
    end)
    {:ok, rows}
  rescue
    e -> {:error, "CSV parse error: #{Exception.message(e)}"}
  end
end
```

---

## สรุป

```
Behaviour:
├── กำหนดด้วย @callback และ @optional_callbacks
├── Implement ด้วย @behaviour ModuleName
├── @impl AnnotateFunction เพื่อ compiler check
└── ใช้สำหรับ pluggable backends, adapters, plugins

Callbacks:
├── @callback name(arg :: type()) :: return_type()
├── @optional_callbacks [func: arity, ...]
└── function_exported?/3 ตรวจสอบ optional callback

Built-in Behaviours:
├── GenServer - concurrent process
├── Supervisor - process supervision
├── Application - OTP application
└── Plug - web middleware
```

---

*ก่อนหน้า: [Part 11](part_11.md) | ต่อไป: [Part 13 - Error Handling](part_13.md)*
