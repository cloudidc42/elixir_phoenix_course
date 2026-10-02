# Part 49: Metaprogramming Advanced

## เป้าหมายการเรียนรู้

- จัดการ AST (Abstract Syntax Tree) ด้วยมือ
- สร้าง custom sigils
- ใช้ `compile_env` และ `Application.compile_env`
- สร้าง code ณ compile time
- เข้าใจ `defdelegate` และ behaviour-based patterns
- ขยาย Protocol
- สร้าง pluggable behaviours
- ตรวจสอบ configuration ณ compile time

---

## 1. AST — Abstract Syntax Tree

ทุก Elixir code ถูก represent เป็น AST ก่อน compile ซึ่งเป็น nested tuple

```elixir
# ดู AST ของ expression ด้วย quote
quote do
  1 + 2
end
# => {:+, [context: Elixir, imports: [{2, Kernel}]], [1, 2]}

quote do
  if x > 0 do
    :positive
  else
    :negative
  end
end
# => {:if, [context: Elixir, ...],
#     [{:>, [...], [{:x, [], Elixir}, 0]},
#      [do: :positive, else: :negative]]}

# unquote แทรกค่า runtime เข้า AST
name = "world"
quote do
  IO.puts("Hello, " <> unquote(name))
end
# => {{:., [], [IO, :puts]}, [], ["Hello, " <> "world"]}
```

### Macro พื้นฐาน

```elixir
defmodule MyMacros do
  # defmacro สร้าง macro ที่ทำงานณ compile time
  defmacro unless(condition, do: block) do
    quote do
      if !unquote(condition) do
        unquote(block)
      end
    end
  end

  # macro สำหรับ timing
  defmacro timed(label, do: block) do
    quote do
      start = System.monotonic_time(:microsecond)
      result = unquote(block)
      elapsed = System.monotonic_time(:microsecond) - start
      IO.puts("#{unquote(label)}: #{elapsed}μs")
      result
    end
  end
end

# ใช้งาน
require MyMacros

MyMacros.unless false do
  IO.puts("นี่จะถูก execute")
end

MyMacros.timed "การคำนวณ" do
  Enum.sum(1..1_000_000)
end
```

---

## 2. AST Manipulation

การแก้ไข AST โดยตรงด้วย `Macro.traverse/4` และ `Macro.prewalk/2`

```elixir
defmodule ASTInspector do
  # นับจำนวน function calls ใน AST
  def count_calls(ast) do
    {_, count} =
      Macro.prewalk(ast, 0, fn
        # match function call pattern: {func_name, meta, args}
        {name, _meta, args} = node, acc
        when is_atom(name) and is_list(args) ->
          {node, acc + 1}

        other, acc ->
          {other, acc}
      end)

    count
  end

  # แทนที่ function calls ทั้งหมด
  def replace_function(ast, old_name, new_name) do
    Macro.postwalk(ast, fn
      {^old_name, meta, args} -> {new_name, meta, args}
      other -> other
    end)
  end

  # ดึง variable names ทั้งหมดจาก AST
  def extract_variables(ast) do
    {_, vars} =
      Macro.prewalk(ast, [], fn
        {name, _meta, context} = node, acc
        when is_atom(name) and is_atom(context) ->
          {node, [name | acc]}

        other, acc ->
          {other, acc}
      end)

    Enum.uniq(vars)
  end
end

# ทดสอบ
ast = quote do
  x = foo(1, 2)
  y = bar(x)
  x + y
end

ASTInspector.count_calls(ast)         # => 3 (foo, bar, +)
ASTInspector.extract_variables(ast)   # => [:x, :y]
```

### สร้าง Macro ที่ Generate Functions

```elixir
defmodule MyApp.FieldAccessor do
  # สร้าง getter functions ณ compile time จาก field list
  defmacro defgetters(fields) do
    Enum.map(fields, fn field ->
      quote do
        def unquote(field)(struct) do
          Map.get(struct, unquote(field))
        end
      end
    end)
  end
end

defmodule MyApp.User do
  require MyApp.FieldAccessor

  defstruct [:name, :email, :age]

  # สร้าง name/1, email/1, age/1 อัตโนมัติ
  MyApp.FieldAccessor.defgetters([:name, :email, :age])
end

user = %MyApp.User{name: "สมชาย", email: "somchai@example.com", age: 30}
MyApp.User.name(user)   # => "สมชาย"
MyApp.User.email(user)  # => "somchai@example.com"
```

---

## 3. Custom Sigils

Sigils เป็น syntactic sugar สำหรับสร้างค่าพิเศษ สร้างได้โดย define function `sigil_X`

```elixir
defmodule MyApp.Sigils do
  # ~v"1.2.3" แปลงเป็น version tuple
  def sigil_v(string, _opts) do
    string
    |> String.split(".")
    |> Enum.map(&String.to_integer/1)
    |> List.to_tuple()
  end

  # ~c สร้าง color struct
  def sigil_c(string, _opts) do
    [r, g, b] =
      string
      |> String.trim_leading("#")
      |> String.graphemes()
      |> Enum.chunk_every(2)
      |> Enum.map(fn hex -> String.to_integer(Enum.join(hex), 16) end)

    %{r: r, g: g, b: b}
  end

  # ~sql สำหรับ SQL queries (raw string ที่มีการตรวจสอบ)
  defmacro sigil_sql({:<<>>, _meta, [query]}, _opts) do
    validated = validate_sql!(query)
    quote do: unquote(validated)
  end

  defp validate_sql!(query) do
    # ตรวจสอบ SQL injection patterns ณ compile time
    dangerous = ~w(DROP DELETE TRUNCATE)

    if Enum.any?(dangerous, &String.contains?(String.upcase(query), &1)) do
      raise CompileError,
        description: "Dangerous SQL detected: #{query}"
    end

    query
  end
end

# ใช้งาน
import MyApp.Sigils

version = ~v"2.1.0"   # => {2, 1, 0}
color = ~c"#FF5733"    # => %{r: 255, g: 87, b: 51}
```

---

## 4. compile_env และ Application.compile_env

`compile_env` ดึงค่า configuration ณ compile time ทำให้ code ถูก optimize ตาม env

```elixir
# config/config.exs
config :my_app, :debug_mode, false
config :my_app, :max_connections, 100

# config/dev.exs
config :my_app, :debug_mode, true
```

```elixir
defmodule MyApp.Config do
  # ดึงค่าณ compile time — ถ้าค่าเปลี่ยน ต้อง recompile
  @debug_mode Application.compile_env(:my_app, :debug_mode, false)
  @max_connections Application.compile_env(:my_app, :max_connections, 50)

  # compile_env! raise ถ้าไม่มีค่า
  @api_key Application.compile_env!(:my_app, :api_key)

  def debug_mode?, do: @debug_mode
  def max_connections, do: @max_connections
end

# ใช้ compile_env ใน macro สำหรับ conditional compilation
defmodule MyApp.Logger do
  require Logger

  defmacro debug(message) do
    if Application.compile_env(:my_app, :debug_mode, false) do
      quote do
        Logger.debug(unquote(message))
      end
    else
      # production: ไม่ compile log code เลย
      :ok
    end
  end
end
```

---

## 5. Code Generation ณ Compile Time

สร้าง functions จำนวนมากจาก data ณ compile time

```elixir
defmodule MyApp.CountryCodes do
  # อ่านไฟล์ JSON ณ compile time
  @external_resource "priv/data/countries.json"

  countries =
    "priv/data/countries.json"
    |> File.read!()
    |> Jason.decode!()

  # สร้าง function สำหรับแต่ละ country ณ compile time
  for %{"code" => code, "name" => name, "dial" => dial} <- countries do
    def country_name(unquote(code)), do: unquote(name)
    def dial_code(unquote(code)), do: unquote(dial)
    def valid_country_code?(unquote(code)), do: true
  end

  # fallback clause
  def country_name(_code), do: {:error, :unknown_country}
  def dial_code(_code), do: {:error, :unknown_country}
  def valid_country_code?(_code), do: false

  # ฟังก์ชัน list ทั้งหมด (เก็บ codes ณ compile time)
  @all_codes Enum.map(countries, & &1["code"])
  def all_codes, do: @all_codes
end
```

### HTTP Route Generation

```elixir
defmodule MyApp.Router do
  defmacro __using__(_opts) do
    quote do
      import MyApp.Router, only: [resource: 2]
      Module.register_attribute(__MODULE__, :routes, accumulate: true)
      @before_compile MyApp.Router
    end
  end

  defmacro resource(path, module) do
    quote do
      @routes {unquote(path), unquote(module)}
    end
  end

  defmacro __before_compile__(env) do
    routes = Module.get_attribute(env.module, :routes)

    route_functions =
      for {path, module} <- routes do
        quote do
          def match("GET", unquote(path)), do: {unquote(module), :index}
          def match("POST", unquote(path)), do: {unquote(module), :create}
          def match("GET", unquote(path) <> "/:id"), do: {unquote(module), :show}
        end
      end

    quote do
      unquote(route_functions)
      def match(_, _), do: {:error, :not_found}
    end
  end
end
```

---

## 6. defdelegate และ Delegation Patterns

`defdelegate` ส่งต่อ function calls ไปยัง module อื่น ใช้สำหรับ facade patterns

```elixir
defmodule MyApp.Accounts do
  # delegate ไปยัง submodules — Accounts เป็น public API
  defdelegate get_user(id), to: MyApp.Accounts.Users
  defdelegate create_user(attrs), to: MyApp.Accounts.Users
  defdelegate update_user(user, attrs), to: MyApp.Accounts.Users
  defdelegate delete_user(user), to: MyApp.Accounts.Users

  defdelegate authenticate(email, password), to: MyApp.Accounts.Auth
  defdelegate generate_token(user), to: MyApp.Accounts.Auth
  defdelegate verify_token(token), to: MyApp.Accounts.Auth

  # ใช้ as: เพื่อเปลี่ยนชื่อ function ปลายทาง
  defdelegate list(opts \\ []), to: MyApp.Accounts.Users, as: :list_users
end
```

### Pluggable Behaviour Pattern

```elixir
defmodule MyApp.PaymentProvider do
  @callback charge(amount :: integer(), token :: String.t()) ::
    {:ok, transaction_id :: String.t()} | {:error, reason :: String.t()}

  @callback refund(transaction_id :: String.t()) ::
    {:ok, refund_id :: String.t()} | {:error, reason :: String.t()}

  @callback verify_webhook(payload :: map(), signature :: String.t()) ::
    :ok | {:error, :invalid_signature}

  # Helper สำหรับดึง provider จาก config
  def provider do
    Application.get_env(:my_app, :payment_provider, MyApp.Payments.Stripe)
  end

  def charge(amount, token) do
    provider().charge(amount, token)
  end

  def refund(transaction_id) do
    provider().refund(transaction_id)
  end
end

# Stripe implementation
defmodule MyApp.Payments.Stripe do
  @behaviour MyApp.PaymentProvider

  @impl true
  def charge(amount, token) do
    # Stripe API call
    {:ok, "txn_stripe_#{:rand.uniform(99999)}"}
  end

  @impl true
  def refund(transaction_id) do
    {:ok, "re_#{transaction_id}"}
  end

  @impl true
  def verify_webhook(payload, signature) do
    # verify Stripe webhook signature
    :ok
  end
end

# Mock สำหรับ testing
defmodule MyApp.Payments.Mock do
  @behaviour MyApp.PaymentProvider

  @impl true
  def charge(_amount, _token), do: {:ok, "mock_txn_123"}

  @impl true
  def refund(txn_id), do: {:ok, "mock_refund_#{txn_id}"}

  @impl true
  def verify_webhook(_payload, _sig), do: :ok
end
```

---

## 7. Protocol Extensions

Protocol ช่วยให้เพิ่ม polymorphism โดยไม่ต้องแก้ไข original module

```elixir
# สร้าง Protocol ใหม่
defprotocol MyApp.Serializable do
  @doc "แปลง struct เป็น map สำหรับส่ง API"
  def to_api(data)

  @doc "แปลง struct เป็น string สำหรับ logging"
  def to_log_string(data)
end

# Implement สำหรับ structs ของเรา
defimpl MyApp.Serializable, for: MyApp.User do
  def to_api(user) do
    %{
      id: user.id,
      name: user.name,
      email: user.email,
      created_at: DateTime.to_iso8601(user.inserted_at)
    }
  end

  def to_log_string(user) do
    "User(id=#{user.id}, email=#{user.email})"
  end
end

defimpl MyApp.Serializable, for: MyApp.Order do
  def to_api(order) do
    %{
      id: order.id,
      total: Decimal.to_string(order.total),
      status: order.status,
      items: Enum.map(order.items, &MyApp.Serializable.to_api/1)
    }
  end

  def to_log_string(order) do
    "Order(id=#{order.id}, total=#{order.total}, status=#{order.status})"
  end
end

# Fallback implementation สำหรับ types ที่ไม่รู้จัก
defimpl MyApp.Serializable, for: Any do
  def to_api(data), do: %{value: inspect(data)}
  def to_log_string(data), do: inspect(data)
end
```

---

## 8. Compile-time Configuration Validation

ตรวจสอบ configuration ให้ครบถ้วนก่อน application start

```elixir
defmodule MyApp.ConfigValidator do
  defmacro __using__(_opts) do
    quote do
      import MyApp.ConfigValidator
      @before_compile MyApp.ConfigValidator
      Module.register_attribute(__MODULE__, :required_configs, accumulate: true)
    end
  end

  defmacro require_config(app, key, opts \\ []) do
    quote do
      @required_configs {unquote(app), unquote(key), unquote(opts)}
    end
  end

  defmacro __before_compile__(_env) do
    quote do
      def validate_config! do
        errors =
          @required_configs
          |> Enum.flat_map(fn {app, key, opts} ->
            validate_single!(app, key, opts)
          end)

        if errors != [] do
          raise """
          Configuration errors found:
          #{Enum.map_join(errors, "\n", &"  - #{&1}")}
          """
        end

        :ok
      end

      defp validate_single!(app, key, opts) do
        value = Application.get_env(app, key)
        errors = []

        errors =
          if is_nil(value) and Keyword.get(opts, :required, false) do
            ["#{app}[:#{key}] is required but not set" | errors]
          else
            errors
          end

        errors =
          if not is_nil(value) do
            case Keyword.get(opts, :type) do
              :string when not is_binary(value) ->
                ["#{app}[:#{key}] must be a string, got: #{inspect(value)}" | errors]
              :integer when not is_integer(value) ->
                ["#{app}[:#{key}] must be an integer, got: #{inspect(value)}" | errors]
              _ ->
                errors
            end
          else
            errors
          end

        errors
      end
    end
  end
end

# ใช้งาน
defmodule MyApp.AppConfig do
  use MyApp.ConfigValidator

  require_config :my_app, :database_url, required: true, type: :string
  require_config :my_app, :secret_key_base, required: true, type: :string
  require_config :my_app, :max_connections, required: false, type: :integer
end

# ตรวจสอบใน Application.start/2
defmodule MyApp.Application do
  def start(_type, _args) do
    MyApp.AppConfig.validate_config!()

    children = [...]
    Supervisor.start_link(children, strategy: :one_for_one)
  end
end
```

---

## สรุป

```
Metaprogramming Advanced
├── AST Manipulation
│   ├── quote/unquote     → สร้าง/แทรก AST
│   ├── Macro.prewalk     → traverse AST
│   └── defmacro          → compile-time code generation
│
├── Custom Sigils
│   └── sigil_X           → custom literal syntax
│
├── Compile-time Config
│   ├── compile_env       → ดึงค่าณ compile time
│   └── @external_resource → track file dependencies
│
├── Code Generation
│   ├── for loops in module → generate many functions
│   ├── @before_compile   → generate code after attrs set
│   └── Module.register_attribute → accumulate metadata
│
├── Delegation
│   ├── defdelegate       → facade pattern
│   └── @behaviour        → contract enforcement
│
├── Protocols
│   ├── defprotocol       → polymorphic interface
│   ├── defimpl           → implementation per type
│   └── for: Any          → fallback implementation
│
└── Validation
    └── __before_compile__ → validate at compile time
```

---

*ก่อนหน้า: [Part 48 - Distributed Elixir](part_48.md) | ต่อไป: [Part 50 - Real-world Project: Blog Platform](part_50.md)*
