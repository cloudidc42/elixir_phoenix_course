# Part 05: Functions และ Modules

## เป้าหมายการเรียนรู้

หลังจากจบ Part นี้ คุณจะสามารถ:
- สร้างและใช้งาน named functions และ anonymous functions
- จัดการ module hierarchy ได้
- ใช้ function capturing, closures, หรือ higher-order functions
- เข้าใจ function arity และ default arguments

---

## 1. Named Functions

### การสร้าง Function

```elixir
defmodule Math do
  # Function พื้นฐาน
  def add(a, b) do
    a + b
  end

  # Short form (single expression)
  def subtract(a, b), do: a - b

  # Multi-line function
  def multiply(a, b) do
    result = a * b
    IO.puts("#{a} * #{b} = #{result}")
    result
  end
end

iex> Math.add(5, 3)
8

iex> Math.subtract(10, 4)
6

iex> Math.multiply(6, 7)
6 * 7 = 42
42
```

### Return Values

```elixir
# Elixir ไม่มี return statement
# return value คือ expression สุดท้ายเสมอ

def calculate(x) do
  doubled = x * 2      # ไม่ใช่ return value
  tripled = x * 3      # ไม่ใช่ return value
  doubled + tripled    # นี่คือ return value
end

iex> calculate(5)
25  # 10 + 15
```

### Private Functions

```elixir
defmodule BankAccount do
  # Public function
  def deposit(account, amount) when amount > 0 do
    new_balance = account.balance + amount
    transaction = create_transaction(:deposit, amount)
    %{account | balance: new_balance, transactions: [transaction | account.transactions]}
  end

  def withdraw(account, amount) when amount > 0 do
    if account.balance >= amount do
      new_balance = account.balance - amount
      transaction = create_transaction(:withdrawal, amount)
      {:ok, %{account | balance: new_balance, transactions: [transaction | account.transactions]}}
    else
      {:error, :insufficient_funds}
    end
  end

  # Private function - ใช้ defp แทน def
  defp create_transaction(type, amount) do
    %{
      type: type,
      amount: amount,
      timestamp: DateTime.utc_now()
    }
  end
end
```

---

## 2. Function Arity

Arity คือจำนวน arguments ของ function

```elixir
defmodule Example do
  def hello, do: "Hello!"           # arity 0 (hello/0)
  def hello(name), do: "Hello #{name}!"  # arity 1 (hello/1)
  def hello(a, b), do: "Hello #{a} and #{b}!"  # arity 2 (hello/2)
end

# เรียกใช้ตาม arity
iex> Example.hello()
"Hello!"

iex> Example.hello("World")
"Hello World!"

iex> Example.hello("Alice", "Bob")
"Hello Alice and Bob!"

# Functions ต่าง arity เป็นคนละ function กัน
iex> &Example.hello/0   # อ้างถึง hello/0
iex> &Example.hello/1   # อ้างถึง hello/1
```

---

## 3. Default Arguments

```elixir
defmodule Formatter do
  # Default argument ด้วย \\
  def format(value, unit \\ "px") do
    "#{value}#{unit}"
  end

  def greet(name, greeting \\ "Hello", punctuation \\ "!") do
    "#{greeting}, #{name}#{punctuation}"
  end

  # ระวัง! ถ้า function มีหลาย clauses ต้องแยก header
  def greet_with_title(title \\ "Mr.", name, greeting \\ "Hello")

  def greet_with_title(title, name, greeting) when is_binary(title) do
    "#{greeting}, #{title} #{name}!"
  end
end

iex> Formatter.format(100)
"100px"

iex> Formatter.format(100, "em")
"100em"

iex> Formatter.greet("Alice")
"Hello, Alice!"

iex> Formatter.greet("Alice", "Hi")
"Hi, Alice!"

iex> Formatter.greet("Alice", "Hi", ".")
"Hi, Alice."
```

### Default Arguments และ Function Arity

```elixir
defmodule Config do
  # def connect(host, port \\ 5432, timeout \\ 5000)
  # สร้าง 3 functions จริงๆ:
  # connect/1 (port=5432, timeout=5000)
  # connect/2 (timeout=5000)
  # connect/3

  def connect(host, port \\ 5432, timeout \\ 5000) do
    IO.puts("Connecting to #{host}:#{port} with #{timeout}ms timeout")
  end
end

iex> Config.connect("localhost")
Connecting to localhost:5432 with 5000ms timeout

iex> Config.connect("localhost", 3306)
Connecting to localhost:3306 with 5000ms timeout

iex> Config.connect("localhost", 3306, 10000)
Connecting to localhost:3306 with 10000ms timeout
```

---

## 4. Anonymous Functions

Anonymous functions (lambdas) ไม่มีชื่อ

```elixir
# สร้าง anonymous function
add = fn a, b -> a + b end

# เรียกใช้ (ต้องมี . ก่อน ())
iex> add.(1, 2)
3

# Short form ด้วย &
double = &(&1 * 2)
iex> double.(5)
10

add10 = &(&1 + 10)
iex> add10.(5)
15

# Multi-line
process = fn x ->
  x
  |> abs()
  |> Integer.to_string()
  |> String.pad_leading(5, "0")
end

iex> process.(-42)
"00042"
```

### Capture Operator (&)

```elixir
# Capture named function เป็น anonymous function
iex> upcase = &String.upcase/1
#Function<...>

iex> upcase.("hello")
"HELLO"

# ใช้ใน Enum
iex> Enum.map(["hello", "world"], &String.upcase/1)
["HELLO", "WORLD"]

# Capture function ใน module ปัจจุบัน
defmodule Utils do
  def double(x), do: x * 2
  def triple(x), do: x * 3

  def process_list(list) do
    list
    |> Enum.map(&double/1)
    |> Enum.map(&triple/1)
  end
end

# Capture ด้วย &n
add = &(&1 + &2)        # fn a, b -> a + b end
multiply = &(&1 * &2)   # fn a, b -> a * b end
triple = &(&1 * 3)      # fn x -> x * 3 end
inspect_all = &IO.inspect(&1, label: &2)

# Named function capture กับ arity
iex> &String.split/2
iex> &String.split/3
```

---

## 5. Higher-Order Functions

Functions ที่รับหรือส่ง function เป็น value

```elixir
# Enum.map - รับ function
iex> Enum.map([1, 2, 3, 4, 5], fn x -> x * 2 end)
[2, 4, 6, 8, 10]

# ใช้ & shorthand
iex> Enum.map([1, 2, 3], &(&1 * 2))
[2, 4, 6]

# Enum.filter
iex> Enum.filter([1, 2, 3, 4, 5], fn x -> rem(x, 2) == 0 end)
[2, 4]

# Enum.reduce
iex> Enum.reduce([1, 2, 3, 4, 5], 0, fn x, acc -> acc + x end)
15

# Function ที่ return function (closure)
def multiplier(n) do
  fn x -> x * n end
end

iex> double = multiplier(2)
iex> double.(5)
10

iex> triple = multiplier(3)
iex> triple.(5)
15
```

### Closures

```elixir
# Closure จำ environment ที่มันถูกสร้าง
def make_counter(initial \\ 0) do
  count = initial

  increment = fn -> count + 1 end
  get = fn -> count end

  {increment, get}
end

# หมายเหตุ: ใน Elixir closures เป็น immutable
# ถ้าต้องการ mutable state ต้องใช้ Agent หรือ Process

defmodule Counter do
  def start(initial \\ 0) do
    Agent.start_link(fn -> initial end)
  end

  def increment(pid, by \\ 1) do
    Agent.update(pid, fn count -> count + by end)
  end

  def get(pid) do
    Agent.get(pid, fn count -> count end)
  end
end

{:ok, counter} = Counter.start(0)
Counter.increment(counter)
Counter.increment(counter)
Counter.increment(counter, 5)
IO.puts(Counter.get(counter))  # 7
```

---

## 6. Modules

### สร้าง Module

```elixir
defmodule MyApp.Utils do
  @moduledoc """
  Utility functions สำหรับ MyApp
  """

  @version "1.0.0"

  def version, do: @version

  def hello(name) do
    "Hello, #{name}!"
  end
end

# เรียกใช้
iex> MyApp.Utils.hello("World")
"Hello, World!"

iex> MyApp.Utils.version()
"1.0.0"
```

### Module Hierarchy

```elixir
# ชื่อ module ใช้ . เป็น separator
defmodule MyApp do
end

defmodule MyApp.Users do
end

defmodule MyApp.Users.Auth do
end

# แต่นี่ไม่ใช่ nesting จริงๆ มันแค่ชื่อ
# MyApp.Users ไม่ได้ "อยู่ใน" MyApp

# สามารถเขียน module ใน module ได้ (แต่ไม่แนะนำ)
defmodule Outer do
  defmodule Inner do
    def hello, do: "Hello from Inner!"
  end

  def hello do
    Inner.hello()
  end
end
```

### Module Attributes

```elixir
defmodule Config do
  # Compile-time constants
  @app_name "MyApp"
  @version "1.0.0"
  @max_connections 100
  @timeout_ms 5_000
  @valid_roles [:admin, :user, :moderator]

  def app_name, do: @app_name
  def version, do: @version

  def max_connections, do: @max_connections

  def valid_role?(role) when is_atom(role) do
    role in @valid_roles
  end

  # Accumulating attributes
  @features []
  @features [:auth | @features]
  @features [:notifications | @features]

  def features, do: @features
end
```

---

## 7. import, alias, use, require

### alias

```elixir
defmodule MyApp.Orders do
  # ใช้แทนชื่อเต็ม
  alias MyApp.Users
  alias MyApp.Products

  # ตั้งชื่อใหม่
  alias MyApp.Payments.Gateway, as: PayGateway

  # Alias หลาย modules จาก parent เดียวกัน
  alias MyApp.{Users, Products, Orders}

  def create_order(user_id, product_ids) do
    user = Users.get(user_id)          # แทน MyApp.Users.get
    products = Products.get_many(product_ids)  # แทน MyApp.Products.get_many
    # ...
  end
end
```

### import

```elixir
defmodule MyModule do
  # Import ทุก functions จาก module
  import Enum

  # Import เฉพาะบาง functions
  import String, only: [upcase: 1, downcase: 1]

  # Import ยกเว้นบาง functions
  import List, except: [first: 1]

  def process(list) do
    list
    |> map(fn x -> upcase(x) end)  # ไม่ต้องเขียน Enum.map, String.upcase
    |> sort()
    |> uniq()
  end
end

# ระวัง! import ทำให้ namespace รก
# แนะนำให้ใช้ alias แทน
```

### require

```elixir
# require ใช้เมื่อต้องการใช้ macros
defmodule MyModule do
  require Logger
  require MyMacros

  def process(data) do
    Logger.info("Processing: #{inspect(data)}")
    MyMacros.some_macro(data)
  end
end
```

### use

```elixir
# use เรียก __using__/1 macro ของ module
# ให้ features/behavior เพิ่มเติม

defmodule MyServer do
  use GenServer  # inject GenServer behavior

  # ต้อง implement callbacks
  def init(state), do: {:ok, state}
  def handle_call(:get, _from, state), do: {:reply, state, state}
end

defmodule MyTest do
  use ExUnit.Case  # inject test helpers

  test "1 + 1 = 2" do
    assert 1 + 1 == 2
  end
end
```

---

## 8. Behaviours

Behaviours กำหนด interface ที่ module ต้อง implement

```elixir
# กำหนด behaviour
defmodule Notifier do
  @callback send(recipient :: String.t(), message :: String.t()) :: {:ok, any} | {:error, any}
  @callback format(message :: String.t()) :: String.t()

  # Optional callback
  @optional_callbacks [format: 1]
end

# Implement behaviour
defmodule EmailNotifier do
  @behaviour Notifier

  @impl Notifier
  def send(email, message) do
    # ส่ง email จริงๆ
    IO.puts("Sending email to #{email}: #{message}")
    {:ok, :sent}
  end

  @impl Notifier
  def format(message) do
    "<html><body>#{message}</body></html>"
  end
end

defmodule SMSNotifier do
  @behaviour Notifier

  @impl Notifier
  def send(phone, message) do
    IO.puts("Sending SMS to #{phone}: #{message}")
    {:ok, :sent}
  end
  # format ไม่ต้อง implement เพราะ optional
end

# ใช้งาน
defmodule NotificationService do
  def notify(notifier, recipient, message) do
    formatted = if function_exported?(notifier, :format, 1) do
      notifier.format(message)
    else
      message
    end

    notifier.send(recipient, formatted)
  end
end

iex> NotificationService.notify(EmailNotifier, "alice@example.com", "Hello!")
Sending email to alice@example.com: <html><body>Hello!</body></html>
{:ok, :sent}
```

---

## 9. Protocols

Protocols เป็น polymorphism แบบ open - add behavior ให้ type ใดๆ ได้

```elixir
# กำหนด Protocol
defprotocol Stringify do
  def to_string(value)
  def to_short_string(value)
end

# Implement สำหรับ Integer
defimpl Stringify, for: Integer do
  def to_string(n), do: "Integer(#{n})"
  def to_short_string(n), do: "#{n}"
end

# Implement สำหรับ Float
defimpl Stringify, for: Float do
  def to_string(f), do: "Float(#{Float.round(f, 2)})"
  def to_short_string(f), do: "#{Float.round(f, 2)}"
end

# Implement สำหรับ Map
defimpl Stringify, for: Map do
  def to_string(m) do
    pairs = Enum.map(m, fn {k, v} -> "#{k}: #{v}" end)
    "Map{#{Enum.join(pairs, ", ")}}"
  end
  def to_short_string(m), do: "Map(#{map_size(m)} keys)"
end

# Implement สำหรับ List
defimpl Stringify, for: List do
  def to_string(l), do: "List[#{Enum.join(l, ", ")}]"
  def to_short_string(l), do: "List(#{length(l)} items)"
end

iex> Stringify.to_string(42)
"Integer(42)"

iex> Stringify.to_string(3.14159)
"Float(3.14)"

iex> Stringify.to_string(%{name: "Alice", age: 30})
"Map{name: Alice, age: 30}"

iex> Stringify.to_string([1, 2, 3])
"List[1, 2, 3]"

iex> Stringify.to_short_string([1, 2, 3, 4, 5])
"List(5 items)"
```

---

## 10. Struct

Struct คือ Map ที่มี defined fields และ compile-time guarantees

```elixir
defmodule User do
  @enforce_keys [:name, :email]  # fields ที่ต้องมีค่า

  defstruct [
    :name,
    :email,
    age: nil,
    role: :user,
    active: true,
    created_at: nil
  ]

  # Custom function สำหรับ struct
  def new(name, email, opts \\ []) do
    %__MODULE__{
      name: name,
      email: email,
      age: Keyword.get(opts, :age),
      role: Keyword.get(opts, :role, :user),
      created_at: DateTime.utc_now()
    }
  end

  def admin?(user), do: user.role == :admin
  def active?(user), do: user.active

  def deactivate(user) do
    %{user | active: false}
  end
end

# สร้าง struct
iex> user = User.new("Alice", "alice@example.com", age: 30)
%User{
  active: true,
  age: 30,
  created_at: ~U[2024-01-01 00:00:00Z],
  email: "alice@example.com",
  name: "Alice",
  role: :user
}

# เข้าถึง fields
iex> user.name
"Alice"

iex> User.admin?(user)
false

# อัปเดต struct
iex> admin_user = %{user | role: :admin}
iex> User.admin?(admin_user)
true

# Enforce keys
iex> %User{}  # error เพราะ name และ email ต้องมี
** (ArgumentError) the following keys must also be given when building struct ...

# Pattern matching กับ struct
def greet(%User{name: name, role: :admin}) do
  "Hello, Admin #{name}!"
end

def greet(%User{name: name}) do
  "Hello, #{name}!"
end
```

---

## 11. ตัวอย่างจริง: Plugin System

```elixir
defmodule Plugin do
  @callback name() :: String.t()
  @callback version() :: String.t()
  @callback run(any()) :: {:ok, any()} | {:error, any()}
  @optional_callbacks [cleanup: 0]

  defmacro __using__(_opts) do
    quote do
      @behaviour Plugin

      def cleanup, do: :ok

      defoverridable [cleanup: 0]
    end
  end
end

defmodule Plugin.Logger do
  use Plugin

  @impl Plugin
  def name, do: "Logger Plugin"

  @impl Plugin
  def version, do: "1.0.0"

  @impl Plugin
  def run(data) do
    IO.puts("[LOG] Processing: #{inspect(data)}")
    {:ok, data}
  end
end

defmodule Plugin.Transformer do
  use Plugin

  @impl Plugin
  def name, do: "Transformer Plugin"

  @impl Plugin
  def version, do: "2.0.0"

  @impl Plugin
  def run(data) when is_map(data) do
    transformed = Map.new(data, fn {k, v} ->
      {Atom.to_string(k), to_string(v)}
    end)
    {:ok, transformed}
  end

  def run(_), do: {:error, "Expected a map"}

  @impl Plugin
  def cleanup do
    IO.puts("Transformer cleanup")
    :ok
  end
end

defmodule PluginManager do
  def run_pipeline(data, plugins) do
    Enum.reduce_while(plugins, {:ok, data}, fn plugin, {:ok, current_data} ->
      IO.puts("Running #{plugin.name()} v#{plugin.version()}")
      case plugin.run(current_data) do
        {:ok, result} -> {:cont, {:ok, result}}
        error -> {:halt, error}
      end
    end)
  end

  def cleanup_all(plugins) do
    Enum.each(plugins, fn plugin ->
      if function_exported?(plugin, :cleanup, 0) do
        plugin.cleanup()
      end
    end)
  end
end

# ใช้งาน
plugins = [Plugin.Logger, Plugin.Transformer]
data = %{name: "Alice", age: 30}

case PluginManager.run_pipeline(data, plugins) do
  {:ok, result} -> IO.puts("Result: #{inspect(result)}")
  {:error, reason} -> IO.puts("Error: #{reason}")
end

PluginManager.cleanup_all(plugins)
```

---

## 12. Function Documentation

```elixir
defmodule MathUtils do
  @moduledoc """
  Mathematical utility functions.

  ## Examples

      iex> MathUtils.factorial(5)
      120

      iex> MathUtils.fibonacci(10)
      [0, 1, 1, 2, 3, 5, 8, 13, 21, 34]
  """

  @doc """
  Calculates the factorial of a non-negative integer.

  ## Parameters
    - `n` - non-negative integer

  ## Returns
    - Integer factorial of n

  ## Examples

      iex> MathUtils.factorial(0)
      1

      iex> MathUtils.factorial(5)
      120

      iex> MathUtils.factorial(10)
      3628800
  """
  def factorial(0), do: 1
  def factorial(n) when is_integer(n) and n > 0, do: n * factorial(n - 1)

  @doc """
  Generates Fibonacci sequence up to n terms.

  ## Parameters
    - `n` - number of terms (positive integer)

  ## Examples

      iex> MathUtils.fibonacci(5)
      [0, 1, 1, 2, 3]
  """
  def fibonacci(n) when n > 0 do
    Stream.unfold({0, 1}, fn {a, b} -> {a, {b, a + b}} end)
    |> Enum.take(n)
  end
end
```

---

## แบบฝึกหัด

### Exercise 1: Higher-Order Functions
เขียน function `apply_twice(f, x)` ที่ apply function f กับ x สองครั้ง:
```elixir
apply_twice(&(&1 * 2), 3)  # => 12
apply_twice(&String.upcase/1, "hello")  # => "HELLO" (same, idempotent)
```

### Exercise 2: Closure
เขียน function `make_adder(n)` ที่ return function ที่บวก n:
```elixir
add5 = make_adder(5)
add5.(3)   # => 8
add5.(10)  # => 15
```

### Exercise 3: Behaviour
สร้าง `Shape` behaviour ที่มี:
- `area/1` - คำนวณพื้นที่
- `perimeter/1` - คำนวณเส้นรอบวง

แล้ว implement สำหรับ Circle, Rectangle, Triangle

### Exercise 4: Struct
สร้าง `Product` struct ที่มี:
- fields: id, name, price, quantity, category
- enforce: name, price
- function: `total_value/1`, `discount/2`, `in_stock?/1`

### Exercise 5: Protocol
สร้าง `Serializable` protocol ที่มี `serialize/1` และ `deserialize/1`
Implement สำหรับ String, Integer, Map

---

## เฉลย Exercise 3

```elixir
defmodule Shape do
  @callback area(map()) :: float()
  @callback perimeter(map()) :: float()
end

defmodule Circle do
  @behaviour Shape

  defstruct [:radius]

  def new(radius), do: %__MODULE__{radius: radius}

  @impl Shape
  def area(%{radius: r}), do: :math.pi() * r * r

  @impl Shape
  def perimeter(%{radius: r}), do: 2 * :math.pi() * r
end

defmodule Rectangle do
  @behaviour Shape

  defstruct [:width, :height]

  def new(width, height), do: %__MODULE__{width: width, height: height}

  @impl Shape
  def area(%{width: w, height: h}), do: w * h

  @impl Shape
  def perimeter(%{width: w, height: h}), do: 2 * (w + h)
end

defmodule Triangle do
  @behaviour Shape

  defstruct [:a, :b, :c]

  def new(a, b, c), do: %__MODULE__{a: a, b: b, c: c}

  @impl Shape
  def area(%{a: a, b: b, c: c}) do
    s = (a + b + c) / 2  # semi-perimeter
    :math.sqrt(s * (s - a) * (s - b) * (s - c))  # Heron's formula
  end

  @impl Shape
  def perimeter(%{a: a, b: b, c: c}), do: a + b + c
end
```

---

## สรุป

```
Functions ใน Elixir:

Named Functions (def/defp):
├── def: public
├── defp: private
├── arity: def hello/1 vs hello/2
├── default args: def f(x, y \\ 0)
└── pattern matching ใน params

Anonymous Functions:
├── fn x -> x * 2 end
├── &(&1 * 2) shorthand
└── &Function.name/arity capture

Higher-Order:
├── รับ function เป็น argument
├── ส่ง function เป็น return value
└── Closures

Modules:
├── defmodule MyModule do
├── Module attributes: @name
├── alias, import, use, require
└── Struct: defstruct

OOP-like Features:
├── Behaviours: interface/contract
├── Protocols: polymorphism
└── Struct: typed data
```

---

*ก่อนหน้า: [Part 04](part_04.md) | ต่อไป: [Part 06 - Control Flow](part_06.md)*
