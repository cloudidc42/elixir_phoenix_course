# Part 04: Pattern Matching พื้นฐาน

## เป้าหมายการเรียนรู้

หลังจากจบ Part นี้ คุณจะสามารถ:
- เข้าใจว่า Pattern Matching คืออะไรและทำงานอย่างไร
- ใช้ Pattern Matching กับข้อมูลแต่ละชนิดได้
- ใช้ Guards ร่วมกับ Pattern Matching
- ใช้ Pattern Matching เพื่อ control flow

---

## 1. Pattern Matching คืออะไร?

Pattern Matching เป็นหนึ่งในคุณสมบัติที่ทรงพลังที่สุดของ Elixir

```elixir
# ใน Python/JavaScript: = คือ assignment
x = 5  # กำหนดให้ x = 5

# ใน Elixir: = คือ match operator
x = 5  # ลอง match ซ้ายกับขวา
        # x ยังไม่มีค่า ดังนั้น bind x = 5

# Pattern matching ต่างออกไป:
{a, b, c} = {1, 2, 3}  # match tuple และ bind ค่า
# a = 1, b = 2, c = 3

[head | tail] = [1, 2, 3]  # match list
# head = 1, tail = [2, 3]
```

### กฎของ Pattern Matching

```
ซ้าย (Pattern)  =  ขวา (Data)
     ↓                ↓
  โครงสร้าง      ข้อมูลจริง

Rules:
1. ถ้า pattern ตรงกับ data → match สำเร็จ
2. ตัวแปรใน pattern จะถูก bind กับค่าที่ตรงกัน
3. _ ใน pattern = wildcard (match อะไรก็ได้ ไม่ bind)
4. ถ้า match ไม่ตรง → MatchError
```

---

## 2. Pattern Matching กับ Types ต่างๆ

### Integers และ Floats

```elixir
# Match กับค่าตรงๆ
iex> 5 = 5
5

iex> 5 = 6
** (MatchError) no match of right hand side value: 6

# Match variable
iex> x = 42
42

iex> x
42

# Match กลับ (เมื่อ x มีค่าแล้ว)
iex> 42 = x
42

iex> 100 = x
** (MatchError) no match of right hand side value: 42
```

### Strings

```elixir
iex> "hello" = "hello"
"hello"

iex> greeting = "Hello"
"Hello"

iex> "Hello" = greeting
"Hello"

# String prefix matching
iex> "Hello " <> name = "Hello Alice"
"Hello Alice"

iex> name
"Alice"

# ซับซ้อนกว่า
iex> <<"Hello", rest::binary>> = "Hello World"
"Hello World"

iex> rest
" World"
```

### Atoms

```elixir
iex> :ok = :ok
:ok

iex> :error = :ok
** (MatchError) no match of right hand side value: :ok

# ใช้บ่อยมากกับ tagged tuples
iex> {:ok, value} = {:ok, 42}
{:ok, 42}

iex> value
42

iex> {:ok, value} = {:error, "not found"}
** (MatchError) no match of right hand side value: {:error, "not found"}
```

### Tuples

```elixir
# Match tuple structure
iex> {a, b} = {1, 2}
{1, 2}

iex> a
1

iex> b
2

# Match ขนาดต้องตรงกัน
iex> {a, b} = {1, 2, 3}
** (MatchError) no match of right hand side value: {1, 2, 3}

# Match ค่าบางตัว ignore บางตัว
iex> {_, second, _} = {1, 2, 3}
{1, 2, 3}

iex> second
2

# Nested tuple matching
iex> {{a, b}, c} = {{1, 2}, 3}
{{1, 2}, 3}

iex> a
1

iex> b
2

iex> c
3

# Match ค่าที่รู้ค่าแล้ว
iex> {1, x} = {1, 99}
{1, 99}

iex> x
99

iex> {2, x} = {1, 99}
** (MatchError) no match of right hand side value: {1, 99}

# Tagged tuple pattern
iex> {:ok, user} = {:ok, %{name: "Alice", age: 30}}
{:ok, %{age: 30, name: "Alice"}}

iex> user.name
"Alice"
```

### Lists

```elixir
# Match list ทั้งหมด
iex> [a, b, c] = [1, 2, 3]
[1, 2, 3]

# Match ขนาดต้องตรงกัน
iex> [a, b] = [1, 2, 3]
** (MatchError) no match of right hand side value: [1, 2, 3]

# Head/Tail matching
iex> [head | tail] = [1, 2, 3, 4, 5]
[1, 2, 3, 4, 5]

iex> head
1

iex> tail
[2, 3, 4, 5]

# Match หลาย elements
iex> [first, second | rest] = [1, 2, 3, 4, 5]
[1, 2, 3, 4, 5]

iex> first
1

iex> second
2

iex> rest
[3, 4, 5]

# Match list ว่าง
iex> [] = []
[]

iex> [] = [1]
** (MatchError) no match of right hand side value: [1]

# Match list ที่มีค่าเฉพาะ
iex> [1, second, 3] = [1, 99, 3]
[1, 99, 3]

iex> second
99

# Nested list
iex> [[a, b], c] = [[1, 2], 3]
[[1, 2], 3]
```

### Maps

```elixir
# Map matching ไม่ต้อง match ทุก key
iex> %{name: name} = %{name: "Alice", age: 30, email: "alice@example.com"}
%{age: 30, email: "alice@example.com", name: "Alice"}

iex> name
"Alice"

# Match หลาย keys
iex> %{name: name, age: age} = %{name: "Alice", age: 30}
%{age: 30, name: "Alice"}

# Match ค่าที่รู้ค่าแล้ว
iex> %{status: :active, user: user} = %{status: :active, user: "Alice"}
%{status: :active, user: "Alice"}

iex> user
"Alice"

# Match ค่าที่ไม่ตรงจะ error
iex> %{status: :active} = %{status: :inactive}
** (MatchError) no match of right hand side value: %{status: :inactive}

# Nested map
iex> %{user: %{name: name}} = %{user: %{name: "Alice", age: 30}}
%{user: %{age: 30, name: "Alice"}}

iex> name
"Alice"

# String keys
iex> %{"name" => name} = %{"name" => "Alice"}
%{"name" => "Alice"}
```

---

## 3. Wildcard Pattern (_)

```elixir
# _ match อะไรก็ได้ และไม่ bind
iex> {_, b, _} = {1, 2, 3}
{1, 2, 3}

iex> b
2

iex> _
# ใช้ _ ไม่ได้ (unbound)

# _name สำหรับ documentation (match และไม่ใช้)
iex> {_first, second} = {1, 2}
{1, 2}

iex> second
2
# _first บอกว่า "ตัวแปรนี้ไม่ใช้"
```

---

## 4. Pin Operator (^)

```elixir
# ^ ใช้เพื่อ match กับค่าที่มีอยู่แล้ว (ไม่ rebind)
iex> x = 5
5

# โดยไม่มี ^ : rebind x = 10
iex> x = 10
10

iex> x
10

# ด้วย ^ : match x กับ 10 (x ยังเป็น 10 อยู่ - match สำเร็จ)
iex> ^x = 10
10

# ด้วย ^ : match x กับ 99 - ล้มเหลว!
iex> ^x = 99
** (MatchError) no match of right hand side value: 99

# ใช้งานจริง
iex> expected = :ok
:ok

iex> {:ok, result} = {:ok, 42}
{:ok, 42}

iex> ^expected = :ok    # ตรวจสอบว่าเป็น :ok
:ok

iex> ^expected = :error  # error เพราะไม่ตรง
** (MatchError) no match of right hand side value: :error
```

### Pin Operator ใน Function Patterns

```elixir
defmodule Validator do
  @expected_version "1.0"

  def validate(%{version: @expected_version} = data) do
    {:ok, data}
  end

  def validate(%{version: version}) do
    {:error, "Invalid version: #{version}, expected #{@expected_version}"}
  end
end
```

---

## 5. Pattern Matching ใน Function Definitions

นี่คือการใช้ Pattern Matching ที่ทรงพลังที่สุด!

```elixir
defmodule Greeting do
  # Match atom :english
  def greet(:english, name) do
    "Hello, #{name}!"
  end

  # Match atom :thai
  def greet(:thai, name) do
    "สวัสดี, #{name}!"
  end

  # Match atom :japanese
  def greet(:japanese, name) do
    "こんにちは、#{name}!"
  end

  # Default case
  def greet(_, name) do
    "Hi, #{name}!"
  end
end

iex> Greeting.greet(:thai, "Alice")
"สวัสดี, Alice!"

iex> Greeting.greet(:spanish, "Bob")
"Hi, Bob!"
```

### Match Tuples ใน Functions

```elixir
defmodule ResultHandler do
  def handle({:ok, value}) do
    IO.puts("Success: #{inspect(value)}")
    value
  end

  def handle({:error, reason}) do
    IO.puts("Error: #{reason}")
    nil
  end

  def handle({:loading, percent}) do
    IO.puts("Loading: #{percent}%")
    nil
  end
end

iex> ResultHandler.handle({:ok, 42})
Success: 42
42

iex> ResultHandler.handle({:error, "not found"})
Error: not found
nil

iex> ResultHandler.handle({:loading, 75})
Loading: 75%
nil
```

### Match ใน Recursive Functions

```elixir
defmodule ListOps do
  # Base case: empty list
  def sum([]), do: 0

  # Recursive case: head + sum of tail
  def sum([head | tail]) do
    head + sum(tail)
  end

  # Length
  def length([]), do: 0
  def length([_ | tail]), do: 1 + length(tail)

  # Map
  def map([], _func), do: []
  def map([head | tail], func) do
    [func.(head) | map(tail, func)]
  end

  # Filter
  def filter([], _func), do: []
  def filter([head | tail], func) do
    if func.(head) do
      [head | filter(tail, func)]
    else
      filter(tail, func)
    end
  end

  # Reduce
  def reduce([], acc, _func), do: acc
  def reduce([head | tail], acc, func) do
    reduce(tail, func.(acc, head), func)
  end
end

iex> ListOps.sum([1, 2, 3, 4, 5])
15

iex> ListOps.length([1, 2, 3])
3

iex> ListOps.map([1, 2, 3], fn x -> x * 2 end)
[2, 4, 6]

iex> ListOps.filter([1, 2, 3, 4, 5], fn x -> rem(x, 2) == 0 end)
[2, 4]

iex> ListOps.reduce([1, 2, 3, 4, 5], 0, fn acc, x -> acc + x end)
15
```

---

## 6. Guards

Guards เป็นเงื่อนไขเพิ่มเติมที่ใช้กับ Pattern Matching

```elixir
# Syntax พื้นฐาน
def function_name(pattern) when guard_expression do
  # body
end
```

### Guards พื้นฐาน

```elixir
defmodule NumberCheck do
  def describe(n) when is_integer(n) and n > 0 do
    "positive integer: #{n}"
  end

  def describe(n) when is_integer(n) and n < 0 do
    "negative integer: #{n}"
  end

  def describe(0) do
    "zero"
  end

  def describe(n) when is_float(n) do
    "float: #{n}"
  end

  def describe(_) do
    "unknown"
  end
end

iex> NumberCheck.describe(42)
"positive integer: 42"

iex> NumberCheck.describe(-5)
"negative integer: -5"

iex> NumberCheck.describe(0)
"zero"

iex> NumberCheck.describe(3.14)
"float: 3.14"

iex> NumberCheck.describe("hello")
"unknown"
```

### Guard Expressions ที่ใช้ได้

```elixir
# Type checks
is_integer(x)
is_float(x)
is_number(x)
is_atom(x)
is_binary(x)
is_list(x)
is_tuple(x)
is_map(x)
is_function(x)
is_nil(x)
is_boolean(x)
is_pid(x)
is_port(x)
is_reference(x)

# Comparison
x > 0
x >= 0
x < 100
x == :ok
x != nil

# Size/Length
length(list) > 0
map_size(map) == 0
tuple_size(tuple) == 2
byte_size(binary) < 100

# Math
rem(x, 2) == 0
abs(x) > 10
div(x, 10) == 5

# Logical
is_integer(x) and x > 0
is_atom(x) or is_binary(x)
not is_nil(x)
```

### Guards ใน case และ cond

```elixir
def categorize(x) do
  case x do
    n when is_integer(n) and n > 0 -> :positive_integer
    n when is_integer(n) and n < 0 -> :negative_integer
    0 -> :zero
    n when is_float(n) -> :float
    _ -> :other
  end
end

# cond ไม่ใช้ guards โดยตรง แต่ใช้ boolean expressions
def grade(score) do
  cond do
    score >= 90 -> "A"
    score >= 80 -> "B"
    score >= 70 -> "C"
    score >= 60 -> "D"
    true -> "F"
  end
end
```

### Custom Guards

```elixir
defmodule Guards do
  # defguard สร้าง custom guard
  defguard is_positive(n) when is_number(n) and n > 0
  defguard is_adult(age) when is_integer(age) and age >= 18
  defguard is_valid_email(email) when is_binary(email) and byte_size(email) > 3
end

defmodule User do
  import Guards

  def create(name, age) when is_adult(age) do
    {:ok, %{name: name, age: age}}
  end

  def create(_, age) do
    {:error, "Age #{age} is below minimum (18)"}
  end

  def set_score(score) when is_positive(score) do
    {:ok, score}
  end

  def set_score(_) do
    {:error, "Score must be positive"}
  end
end

iex> User.create("Alice", 25)
{:ok, %{age: 25, name: "Alice"}}

iex> User.create("Bob", 16)
{:error, "Age 16 is below minimum (18)"}
```

---

## 7. case Statement

```elixir
# case ใช้ Pattern Matching เพื่อ control flow
def process_result(result) do
  case result do
    {:ok, data} ->
      IO.puts("Got data: #{inspect(data)}")
      data

    {:error, :not_found} ->
      IO.puts("Resource not found")
      nil

    {:error, reason} ->
      IO.puts("Error: #{reason}")
      nil

    _ ->
      IO.puts("Unexpected result")
      nil
  end
end
```

### case กับ Guards

```elixir
def process_number(n) do
  case n do
    x when is_integer(x) and x > 100 ->
      "Large integer: #{x}"

    x when is_integer(x) and x > 0 ->
      "Small positive integer: #{x}"

    x when is_integer(x) ->
      "Non-positive integer: #{x}"

    x when is_float(x) ->
      "Float: #{x}"

    _ ->
      "Not a number"
  end
end
```

### case กับ Nested Patterns

```elixir
def process_user_action(action) do
  case action do
    {:login, %{user: user, password: pass}} when is_binary(user) and is_binary(pass) ->
      authenticate(user, pass)

    {:logout, %{session_id: id}} when is_binary(id) ->
      close_session(id)

    {:update_profile, %{user_id: id, changes: changes}} when is_map(changes) ->
      update_user(id, changes)

    _ ->
      {:error, "Invalid action"}
  end
end
```

---

## 8. with Statement

`with` ใช้สำหรับ chaining หลาย pattern matches

```elixir
# ถ้าทุก match สำเร็จ ทำ do block
# ถ้ามีอันไหนไม่ตรง ส่ง value นั้นกลับ

def create_account(params) do
  with {:ok, name} <- validate_name(params[:name]),
       {:ok, email} <- validate_email(params[:email]),
       {:ok, age} <- validate_age(params[:age]),
       {:ok, user} <- save_user(name, email, age) do
    {:ok, user}
  else
    {:error, :invalid_name} -> {:error, "Name is invalid"}
    {:error, :invalid_email} -> {:error, "Email is invalid"}
    {:error, :underage} -> {:error, "User must be 18+"}
    error -> error
  end
end

# เทียบกับการเขียนแบบ nested case:
def create_account_nested(params) do
  case validate_name(params[:name]) do
    {:ok, name} ->
      case validate_email(params[:email]) do
        {:ok, email} ->
          case validate_age(params[:age]) do
            {:ok, age} ->
              save_user(name, email, age)
            error -> error
          end
        error -> error
      end
    error -> error
  end
end
# with ทำให้โค้ดอ่านง่ายกว่ามาก!
```

### with ที่ซับซ้อนขึ้น

```elixir
def process_order(order_id) do
  with {:ok, order} <- fetch_order(order_id),
       :ok <- validate_order_status(order),
       {:ok, inventory} <- check_inventory(order.items),
       {:ok, payment} <- process_payment(order.total),
       {:ok, _} <- update_inventory(inventory),
       {:ok, shipped} <- create_shipment(order) do
    {:ok, %{order: order, payment: payment, shipment: shipped}}
  end
end
```

---

## 9. Pattern Matching ใน Comprehensions

```elixir
# กรอง patterns ที่ match เท่านั้น
users = [
  {:user, "Alice", 30},
  {:admin, "Bob", 25},
  {:user, "Charlie", 35},
  {:admin, "Dave", 28}
]

# เอาเฉพาะ user ที่มี age > 28
iex> for {:user, name, age} when age > 28 <- users do
...>   {name, age}
...> end
[{"Alice", 30}, {"Charlie", 35}]

# Match map ใน comprehension
data = [
  %{type: :success, value: 10},
  %{type: :error, message: "oops"},
  %{type: :success, value: 20},
]

iex> for %{type: :success, value: v} <- data, do: v
[10, 20]
```

---

## 10. ตัวอย่างจริง: HTTP Response Handler

```elixir
defmodule HttpHandler do
  @moduledoc """
  จัดการ HTTP responses ด้วย Pattern Matching
  """

  defstruct [:status, :body, :headers]

  def handle(%__MODULE__{status: 200, body: body}) do
    {:ok, parse_body(body)}
  end

  def handle(%__MODULE__{status: 201, body: body, headers: headers}) do
    location = get_header(headers, "Location")
    {:created, parse_body(body), location}
  end

  def handle(%__MODULE__{status: 400, body: body}) do
    {:bad_request, parse_error(body)}
  end

  def handle(%__MODULE__{status: 401}) do
    {:unauthorized, "Authentication required"}
  end

  def handle(%__MODULE__{status: 403}) do
    {:forbidden, "Access denied"}
  end

  def handle(%__MODULE__{status: 404}) do
    {:not_found, "Resource not found"}
  end

  def handle(%__MODULE__{status: status, body: body}) when status >= 500 do
    {:server_error, status, parse_error(body)}
  end

  def handle(%__MODULE__{status: status}) do
    {:unknown, "Unhandled status: #{status}"}
  end

  defp parse_body(body) when is_binary(body) do
    case Jason.decode(body) do
      {:ok, data} -> data
      _ -> body
    end
  end

  defp parse_error(body) when is_binary(body) do
    case Jason.decode(body) do
      {:ok, %{"message" => msg}} -> msg
      {:ok, %{"error" => err}} -> err
      _ -> body
    end
  end

  defp get_header(headers, name) do
    case List.keyfind(headers, name, 0) do
      {_, value} -> value
      nil -> nil
    end
  end
end
```

---

## 11. ตัวอย่างจริง: State Machine

```elixir
defmodule OrderStateMachine do
  @moduledoc """
  State machine สำหรับ order workflow
  """

  # กำหนด transitions ที่ valid
  def transition(:pending, :confirm), do: {:ok, :confirmed}
  def transition(:confirmed, :ship), do: {:ok, :shipped}
  def transition(:shipped, :deliver), do: {:ok, :delivered}
  def transition(:pending, :cancel), do: {:ok, :cancelled}
  def transition(:confirmed, :cancel), do: {:ok, :cancelled}

  # Invalid transitions
  def transition(from, action) do
    {:error, "Cannot #{action} an order in #{from} state"}
  end

  # Execute action
  def execute(order, action) do
    case transition(order.status, action) do
      {:ok, new_status} ->
        {:ok, %{order | status: new_status, updated_at: DateTime.utc_now()}}

      {:error, reason} ->
        {:error, reason}
    end
  end

  # Batch transitions
  def execute_all(order, actions) do
    Enum.reduce_while(actions, {:ok, order}, fn action, {:ok, current_order} ->
      case execute(current_order, action) do
        {:ok, updated} -> {:cont, {:ok, updated}}
        error -> {:halt, error}
      end
    end)
  end
end

# ใช้งาน
order = %{id: 1, status: :pending, created_at: DateTime.utc_now()}

iex> {:ok, confirmed} = OrderStateMachine.execute(order, :confirm)
iex> confirmed.status
:confirmed

iex> {:ok, shipped} = OrderStateMachine.execute(confirmed, :ship)
iex> shipped.status
:shipped

iex> OrderStateMachine.execute(order, :deliver)
{:error, "Cannot deliver an order in pending state"}

# Batch
iex> {:ok, final} = OrderStateMachine.execute_all(order, [:confirm, :ship, :deliver])
iex> final.status
:delivered
```

---

## แบบฝึกหัด

### Exercise 1: Basic Pattern Matching
```elixir
# 1. Match tuple {name, age, role} จาก {"Alice", 30, :admin}
# 2. Match list [first, _, last] จาก [1, 2, 3]
# 3. Match map %{name: name, score: score} จาก %{name: "Bob", score: 95, level: :gold}
# 4. Match {:ok, %{id: id, data: data}} จาก {:ok, %{id: 1, data: "test", meta: nil}}
```

### Exercise 2: Function Pattern Matching
เขียน function `describe_list` ที่:
- `[]` → "empty list"
- `[_]` → "single element list"
- `[_, _]` → "two element list"
- `[_ | _]` → "list with 3+ elements"

### Exercise 3: Guards
เขียน function `classify_temperature` ที่:
- `temp <= 0` → `:freezing`
- `0 < temp <= 15` → `:cold`
- `15 < temp <= 25` → `:comfortable`
- `25 < temp <= 35` → `:warm`
- `temp > 35` → `:hot`

### Exercise 4: with Statement
เขียน function `register_user` ที่:
1. Validate name (ต้องไม่ว่าง)
2. Validate email (ต้องมี @)
3. Validate password (ต้องยาวอย่างน้อย 8 ตัว)
4. ถ้าทุกอย่างผ่าน สร้าง user map

### Exercise 5: State Machine
สร้าง traffic light state machine:
- `:red` → `:green` (action: :go)
- `:green` → `:yellow` (action: :slow)
- `:yellow` → `:red` (action: :stop)

---

## เฉลย Exercise 2

```elixir
defmodule ListDescriber do
  def describe([]), do: "empty list"
  def describe([_]), do: "single element list"
  def describe([_, _]), do: "two element list"
  def describe([_ | _]), do: "list with 3+ elements"
end
```

## เฉลย Exercise 3

```elixir
defmodule Temperature do
  def classify(temp) when temp <= 0, do: :freezing
  def classify(temp) when temp <= 15, do: :cold
  def classify(temp) when temp <= 25, do: :comfortable
  def classify(temp) when temp <= 35, do: :warm
  def classify(_temp), do: :hot
end
```

## เฉลย Exercise 4

```elixir
defmodule UserRegistration do
  def register_user(params) do
    with {:ok, name} <- validate_name(params[:name]),
         {:ok, email} <- validate_email(params[:email]),
         {:ok, password} <- validate_password(params[:password]) do
      {:ok, %{name: name, email: email, password_hash: hash_password(password)}}
    end
  end

  defp validate_name(nil), do: {:error, "Name is required"}
  defp validate_name(""), do: {:error, "Name cannot be empty"}
  defp validate_name(name) when is_binary(name), do: {:ok, String.trim(name)}

  defp validate_email(nil), do: {:error, "Email is required"}
  defp validate_email(email) when is_binary(email) do
    if String.contains?(email, "@") do
      {:ok, String.downcase(email)}
    else
      {:error, "Invalid email format"}
    end
  end

  defp validate_password(nil), do: {:error, "Password is required"}
  defp validate_password(pass) when is_binary(pass) and byte_size(pass) >= 8 do
    {:ok, pass}
  end
  defp validate_password(_), do: {:error, "Password must be at least 8 characters"}

  defp hash_password(password) do
    # ในจริงใช้ bcrypt หรือ argon2
    :crypto.hash(:sha256, password) |> Base.encode16(case: :lower)
  end
end
```

---

## สรุป

```
Pattern Matching ใน Elixir:

= Operator:
├── Match ค่าตรงๆ: 5 = 5
├── Bind variables: x = 5
└── Destructure: {a, b} = {1, 2}

Special Patterns:
├── _ Wildcard: match อะไรก็ได้
├── ^ Pin: match ค่าที่มีอยู่
└── [h|t] List: head/tail

Guards (when):
├── Type checks: is_integer, is_binary, ...
├── Comparisons: >, <, ==, ...
├── Size: length, map_size, ...
└── Logic: and, or, not

Control Flow:
├── case: match หลาย patterns
├── cond: multiple conditions
└── with: chain หลาย matches

Functions:
├── Multiple clauses กับ patterns
├── Guards บน function params
└── Pattern matching ใน params
```

---

*ก่อนหน้า: [Part 03](part_03.md) | ต่อไป: [Part 05 - Functions และ Modules](part_05.md)*
