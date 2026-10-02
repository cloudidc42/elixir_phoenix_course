# Part 06: Control Flow - if, cond, case, with

## เป้าหมายการเรียนรู้

หลังจากจบ Part นี้ คุณจะสามารถ:
- ใช้ if/unless, cond, case, with ได้อย่างถูกต้อง
- เลือก control flow ที่เหมาะสมกับสถานการณ์
- เขียน idiomatic Elixir ด้วย Pattern Matching

---

## 1. if และ unless

```elixir
# if พื้นฐาน
if condition do
  # execute when true
end

# if/else
if condition do
  # true branch
else
  # false branch
end

# Inline (single line)
if condition, do: true_value, else: false_value

# unless (ตรงข้ามกับ if)
unless condition do
  # execute when false
end
```

### ตัวอย่าง if

```elixir
defmodule AgeCheck do
  def can_vote?(age) do
    if age >= 18 do
      "You can vote!"
    else
      "You cannot vote yet. Come back in #{18 - age} years."
    end
  end

  def status(user) do
    status = if user.active, do: "Active", else: "Inactive"
    premium = if user.premium, do: " (Premium)", else: ""
    status <> premium
  end
end

iex> AgeCheck.can_vote?(20)
"You can vote!"

iex> AgeCheck.can_vote?(15)
"You cannot vote yet. Come back in 3 years."
```

### if เป็น Expression

```elixir
# if คือ expression ใน Elixir (return value)
x = if true, do: 1, else: 2
# x = 1

# ถ้าไม่มี else และ condition เป็น false → return nil
result = if false do
  "yes"
end
# result = nil

# ดังนั้นควรเสมอมี else ถ้าต้องการ value
```

### unless

```elixir
def process(data) do
  unless is_nil(data) do
    handle(data)
  end
end

# เทียบเท่ากับ
def process(data) do
  if not is_nil(data) do
    handle(data)
  end
end

# unless กับ else (แต่ไม่แนะนำ อ่านสับสน)
unless condition do
  # false branch
else
  # true branch
end
```

---

## 2. cond

`cond` ใช้เมื่อมีหลาย conditions ที่ต้องตรวจสอบ (คล้าย else if)

```elixir
cond do
  condition1 -> result1
  condition2 -> result2
  condition3 -> result3
  true -> default_result  # default case
end
```

### ตัวอย่าง cond

```elixir
defmodule Grader do
  def grade(score) do
    cond do
      score >= 90 -> "A"
      score >= 80 -> "B"
      score >= 70 -> "C"
      score >= 60 -> "D"
      true -> "F"
    end
  end

  def fizzbuzz(n) do
    cond do
      rem(n, 15) == 0 -> "FizzBuzz"
      rem(n, 3) == 0 -> "Fizz"
      rem(n, 5) == 0 -> "Buzz"
      true -> to_string(n)
    end
  end
end

iex> Grader.grade(95)
"A"

iex> Grader.grade(55)
"F"

iex> Enum.map(1..20, &Grader.fizzbuzz/1)
["1", "2", "Fizz", "4", "Buzz", "Fizz", "7", "8", "Fizz", "Buzz",
 "11", "Fizz", "13", "14", "FizzBuzz", "16", "17", "Fizz", "19", "Buzz"]
```

### cond vs if vs case

```elixir
# if: binary condition (true/false)
result = if x > 0, do: :positive, else: :non_positive

# cond: หลาย conditions เป็นตัวแปร
result = cond do
  x > 0 -> :positive
  x < 0 -> :negative
  true -> :zero
end

# case: match patterns
result = case x do
  n when n > 0 -> :positive
  n when n < 0 -> :negative
  0 -> :zero
end
```

---

## 3. case

`case` ใช้ Pattern Matching เพื่อ branch

```elixir
case value do
  pattern1 -> result1
  pattern2 -> result2
  _ -> default
end
```

### ตัวอย่าง case พื้นฐาน

```elixir
defmodule HttpStatus do
  def describe(status) do
    case status do
      200 -> "OK"
      201 -> "Created"
      400 -> "Bad Request"
      401 -> "Unauthorized"
      403 -> "Forbidden"
      404 -> "Not Found"
      500 -> "Internal Server Error"
      _ -> "Unknown Status #{status}"
    end
  end
end

iex> HttpStatus.describe(200)
"OK"

iex> HttpStatus.describe(404)
"Not Found"

iex> HttpStatus.describe(418)
"Unknown Status 418"
```

### case กับ Pattern Matching

```elixir
defmodule ResultProcessor do
  def process(result) do
    case result do
      {:ok, data} ->
        IO.puts("Success!")
        process_data(data)

      {:error, :not_found} ->
        IO.puts("Not found, using default")
        default_value()

      {:error, reason} when is_binary(reason) ->
        IO.puts("Error: #{reason}")
        nil

      {:loading, percent} when percent >= 0 and percent <= 100 ->
        IO.puts("Loading: #{percent}%")
        nil

      _ ->
        IO.puts("Unexpected result: #{inspect(result)}")
        nil
    end
  end

  defp process_data(data), do: data
  defp default_value, do: %{}
end
```

### case กับ Complex Patterns

```elixir
defmodule UserService do
  def authorize(%{role: :admin} = user, _action) do
    {:authorized, user}
  end

  def authorize(%{role: :user, active: true} = user, action) do
    case action do
      :read -> {:authorized, user}
      :write when user.verified -> {:authorized, user}
      :write -> {:unauthorized, "Email verification required"}
      _ -> {:unauthorized, "Action not permitted"}
    end
  end

  def authorize(%{active: false}, _action) do
    {:unauthorized, "Account is inactive"}
  end

  def authorize(_, _) do
    {:unauthorized, "Invalid user"}
  end
end

# เรียกใช้
user = %{role: :user, active: true, verified: true}
case UserService.authorize(user, :write) do
  {:authorized, u} -> perform_action(u)
  {:unauthorized, reason} -> send_error(reason)
end
```

---

## 4. with

`with` ใช้สำหรับ chaining หลาย operations ที่อาจ fail

```elixir
with pattern1 <- expression1,
     pattern2 <- expression2,
     pattern3 <- expression3 do
  success_body
else
  failed_pattern -> handle_failure
end
```

### with พื้นฐาน

```elixir
defmodule Registration do
  def register(params) do
    with {:ok, name} <- validate_name(params["name"]),
         {:ok, email} <- validate_email(params["email"]),
         {:ok, password} <- validate_password(params["password"]),
         {:ok, user} <- create_user(name, email, password),
         {:ok, _token} <- send_verification_email(user) do
      {:ok, user}
    else
      {:error, :empty_name} -> {:error, "Name cannot be empty"}
      {:error, :invalid_email} -> {:error, "Invalid email format"}
      {:error, :weak_password} -> {:error, "Password is too weak"}
      {:error, :email_taken} -> {:error, "Email already registered"}
      error -> error
    end
  end

  defp validate_name(nil), do: {:error, :empty_name}
  defp validate_name(""), do: {:error, :empty_name}
  defp validate_name(name), do: {:ok, String.trim(name)}

  defp validate_email(email) when is_binary(email) do
    if String.match?(email, ~r/^[^\s]+@[^\s]+\.[^\s]+$/) do
      {:ok, String.downcase(email)}
    else
      {:error, :invalid_email}
    end
  end
  defp validate_email(_), do: {:error, :invalid_email}

  defp validate_password(pass) when byte_size(pass) >= 8, do: {:ok, pass}
  defp validate_password(_), do: {:error, :weak_password}

  defp create_user(name, email, password) do
    # สมมติ save ลง DB
    {:ok, %{name: name, email: email, id: :rand.uniform(1000)}}
  end

  defp send_verification_email(_user) do
    # ส่ง email
    {:ok, "token123"}
  end
end
```

### with ไม่มี else

```elixir
# ถ้าไม่มี else pattern ที่ match ไม่ได้จะ propagate ออกไป
def process(data) do
  with {:ok, parsed} <- parse(data),
       {:ok, validated} <- validate(parsed),
       {:ok, result} <- execute(validated) do
    {:ok, result}
  end
  # ถ้า parse/validate/execute return {:error, ...}
  # มันจะ return ค่านั้นออกมาตรงๆ
end
```

### with กับ Guards

```elixir
def process_user(id) do
  with {:ok, user} <- fetch_user(id),
       true <- user.active || {:error, :inactive},
       {:ok, profile} <- fetch_profile(user.id),
       true <- profile.complete || {:error, :incomplete_profile} do
    {:ok, %{user: user, profile: profile}}
  end
end
```

---

## 5. Comprehensions

```elixir
# for comprehension
for x <- 1..5, do: x * 2
# [2, 4, 6, 8, 10]

# กับ filter (condition)
for x <- 1..20, rem(x, 3) == 0, do: x
# [3, 6, 9, 12, 15, 18]

# หลาย generators (Cartesian product)
for x <- 1..3, y <- 1..3 do
  {x, y}
end

# into: เพื่อเก็บผลลัพธ์เป็น type อื่น
for {key, value} <- %{a: 1, b: 2, c: 3}, into: %{} do
  {key, value * 10}
end
# %{a: 10, b: 20, c: 30}

# uniq: ลบค่าซ้ำ
for x <- [1, 2, 1, 3, 2], uniq: true, do: x
# [1, 2, 3]
```

---

## 6. receive และ after

ใช้ใน process context (จะเรียนเพิ่มเติมใน Part 14)

```elixir
# receive รอรับ message
receive do
  {:msg, data} -> handle(data)
  :stop -> exit(:normal)
end

# receive กับ timeout
receive do
  {:msg, data} -> handle(data)
after
  5000 ->  # timeout หลัง 5 วินาที
    IO.puts("Timeout!")
    :timeout
end
```

---

## 7. try/rescue/catch

```elixir
# try/rescue จัดการ exceptions
try do
  risky_operation()
rescue
  e in RuntimeError ->
    IO.puts("Runtime error: #{e.message}")
    {:error, :runtime}

  e in [ArgumentError, FunctionClauseError] ->
    IO.puts("Argument error: #{inspect(e)}")
    {:error, :argument}

  _ ->
    IO.puts("Unknown error")
    {:error, :unknown}
end

# try/catch จัดการ throw
try do
  throw(:my_error)
catch
  :throw, value ->
    IO.puts("Caught: #{inspect(value)}")
end

# try/after (ensure cleanup)
try do
  open_file("file.txt")
rescue
  e -> handle_error(e)
after
  close_file()  # ทำเสมอไม่ว่า error หรือไม่
end
```

---

## 8. Idiomatic Elixir Control Flow

### ใช้ Pattern Matching แทน if/else

```elixir
# ไม่ดี (imperative style)
def process(user) do
  if user.role == :admin do
    if user.active do
      do_admin_action(user)
    else
      {:error, :inactive}
    end
  else
    {:error, :unauthorized}
  end
end

# ดีกว่า (pattern matching style)
def process(%{role: :admin, active: true} = user) do
  do_admin_action(user)
end

def process(%{role: :admin, active: false}) do
  {:error, :inactive}
end

def process(_user) do
  {:error, :unauthorized}
end
```

### Pipeline แทน Nested Logic

```elixir
# ไม่ดี (nested)
def process_data(input) do
  parsed = parse(input)
  if parsed != nil do
    validated = validate(parsed)
    if validated != nil do
      transform(validated)
    else
      nil
    end
  else
    nil
  end
end

# ดีกว่า (with)
def process_data(input) do
  with {:ok, parsed} <- parse(input),
       {:ok, validated} <- validate(parsed),
       {:ok, result} <- transform(validated) do
    {:ok, result}
  end
end
```

---

## 9. ตัวอย่างจริง: Payment Processing

```elixir
defmodule PaymentProcessor do
  @moduledoc """
  ระบบประมวลผลการชำระเงิน
  """

  def process_payment(payment_params) do
    with {:ok, amount} <- validate_amount(payment_params[:amount]),
         {:ok, currency} <- validate_currency(payment_params[:currency]),
         {:ok, card} <- validate_card(payment_params[:card]),
         {:ok, auth} <- authorize_payment(amount, currency, card),
         {:ok, charge} <- capture_payment(auth) do
      {:ok, %{
        transaction_id: charge.id,
        amount: amount,
        currency: currency,
        status: :completed,
        timestamp: DateTime.utc_now()
      }}
    else
      {:error, :invalid_amount} -> {:error, "Amount must be positive"}
      {:error, :unsupported_currency} -> {:error, "Currency not supported"}
      {:error, :invalid_card} -> {:error, "Invalid card details"}
      {:error, :card_declined} -> {:error, "Card was declined"}
      {:error, :insufficient_funds} -> {:error, "Insufficient funds"}
      {:error, reason} -> {:error, "Payment failed: #{reason}"}
    end
  end

  defp validate_amount(nil), do: {:error, :invalid_amount}
  defp validate_amount(amount) when is_number(amount) and amount > 0 do
    {:ok, Float.round(amount * 1.0, 2)}
  end
  defp validate_amount(_), do: {:error, :invalid_amount}

  defp validate_currency(currency) do
    supported = ~w(THB USD EUR GBP JPY)
    if currency in supported do
      {:ok, currency}
    else
      {:error, :unsupported_currency}
    end
  end

  defp validate_card(%{number: num, expiry: exp, cvv: cvv})
       when byte_size(num) == 16 and byte_size(cvv) == 3 do
    case validate_expiry(exp) do
      :ok -> {:ok, %{number: num, expiry: exp, cvv: cvv}}
      :expired -> {:error, :invalid_card}
    end
  end
  defp validate_card(_), do: {:error, :invalid_card}

  defp validate_expiry(%{month: m, year: y}) do
    now = Date.utc_today()
    card_date = Date.new!(y, m, 1)
    if Date.compare(card_date, now) != :lt, do: :ok, else: :expired
  end

  defp authorize_payment(amount, currency, card) do
    # จำลอง authorization
    cond do
      rem(String.to_integer(String.last(card.number)), 2) == 0 ->
        {:ok, %{auth_code: "AUTH_#{:rand.uniform(999999)}", amount: amount, currency: currency}}
      amount > 100_000 ->
        {:error, :insufficient_funds}
      true ->
        {:error, :card_declined}
    end
  end

  defp capture_payment(%{auth_code: auth_code} = auth) do
    # จำลอง capture
    {:ok, %{id: "TXN_#{auth_code}", amount: auth.amount, status: :captured}}
  end
end

# ทดสอบ
result = PaymentProcessor.process_payment(%{
  amount: 1500.00,
  currency: "THB",
  card: %{
    number: "4111111111111112",
    expiry: %{month: 12, year: 2026},
    cvv: "123"
  }
})

case result do
  {:ok, transaction} ->
    IO.puts("Payment successful! Transaction ID: #{transaction.transaction_id}")
  {:error, reason} ->
    IO.puts("Payment failed: #{reason}")
end
```

---

## 10. ตัวอย่างจริง: Request Router

```elixir
defmodule Router do
  def handle(%{method: "GET", path: "/"}) do
    {:ok, 200, "Welcome to the API!"}
  end

  def handle(%{method: "GET", path: "/users"}) do
    users = get_all_users()
    {:ok, 200, users}
  end

  def handle(%{method: "GET", path: "/users/" <> id}) do
    case get_user(id) do
      {:ok, user} -> {:ok, 200, user}
      {:error, :not_found} -> {:ok, 404, "User not found"}
    end
  end

  def handle(%{method: "POST", path: "/users", body: body}) do
    with {:ok, params} <- parse_json(body),
         {:ok, user} <- create_user(params) do
      {:ok, 201, user}
    else
      {:error, :invalid_json} -> {:ok, 400, "Invalid JSON"}
      {:error, :validation, errors} -> {:ok, 422, errors}
      _ -> {:ok, 500, "Internal server error"}
    end
  end

  def handle(%{method: "PUT", path: "/users/" <> id, body: body}) do
    with {:ok, params} <- parse_json(body),
         {:ok, user} <- update_user(id, params) do
      {:ok, 200, user}
    else
      {:error, :not_found} -> {:ok, 404, "User not found"}
      {:error, :invalid_json} -> {:ok, 400, "Invalid JSON"}
      _ -> {:ok, 500, "Internal server error"}
    end
  end

  def handle(%{method: "DELETE", path: "/users/" <> id}) do
    case delete_user(id) do
      :ok -> {:ok, 204, ""}
      {:error, :not_found} -> {:ok, 404, "User not found"}
    end
  end

  def handle(%{method: method, path: path}) do
    {:ok, 404, "#{method} #{path} not found"}
  end

  # Helper functions
  defp get_all_users, do: []
  defp get_user(_id), do: {:error, :not_found}
  defp create_user(_params), do: {:ok, %{}}
  defp update_user(_id, _params), do: {:ok, %{}}
  defp delete_user(_id), do: :ok
  defp parse_json(_body), do: {:ok, %{}}
end
```

---

## แบบฝึกหัด

### Exercise 1: Rewrite with Pattern Matching
```elixir
# แก้ไข code นี้ให้ใช้ Pattern Matching แทน if/else
def process(value) do
  if is_integer(value) do
    if value > 0 do
      "positive: #{value}"
    else
      "non-positive: #{value}"
    end
  else
    if is_binary(value) do
      "string: #{value}"
    else
      "other: #{inspect(value)}"
    end
  end
end
```

### Exercise 2: cond
เขียน function `bmi_category` ที่รับ BMI และ return category:
- < 18.5: "Underweight"
- 18.5-24.9: "Normal weight"
- 25-29.9: "Overweight"
- >= 30: "Obese"

### Exercise 3: with
เขียน function `login` ที่:
1. ค้นหา user จาก username
2. ตรวจสอบ password
3. ตรวจสอบว่า account active
4. สร้าง session token
ทุกขั้นตอนสามารถ fail ได้

### Exercise 4: State Machine
สร้าง vending machine โดยใช้ case:
- states: :idle, :selecting, :dispensing, :collecting_change
- events: :insert_coin, :select_item, :dispense, :collect_change, :cancel

---

## เฉลย Exercise 1

```elixir
def process(value) when is_integer(value) and value > 0, do: "positive: #{value}"
def process(value) when is_integer(value), do: "non-positive: #{value}"
def process(value) when is_binary(value), do: "string: #{value}"
def process(value), do: "other: #{inspect(value)}"
```

## เฉลย Exercise 2

```elixir
defmodule BMI do
  def category(bmi) do
    cond do
      bmi < 18.5 -> "Underweight"
      bmi < 25.0 -> "Normal weight"
      bmi < 30.0 -> "Overweight"
      true -> "Obese"
    end
  end

  def calculate(weight_kg, height_m) when height_m > 0 do
    bmi = weight_kg / (height_m * height_m)
    {Float.round(bmi, 1), category(bmi)}
  end
end
```

## เฉลย Exercise 3

```elixir
defmodule Auth do
  def login(username, password) do
    with {:ok, user} <- find_user(username),
         :ok <- verify_password(user, password),
         :ok <- check_active(user),
         {:ok, token} <- create_session(user) do
      {:ok, %{user: user, token: token}}
    else
      {:error, :not_found} -> {:error, "User not found"}
      {:error, :wrong_password} -> {:error, "Invalid credentials"}
      {:error, :inactive} -> {:error, "Account is inactive"}
      error -> error
    end
  end

  defp find_user("alice"), do: {:ok, %{id: 1, name: "Alice", active: true}}
  defp find_user(_), do: {:error, :not_found}

  defp verify_password(_user, "correct_password"), do: :ok
  defp verify_password(_, _), do: {:error, :wrong_password}

  defp check_active(%{active: true}), do: :ok
  defp check_active(_), do: {:error, :inactive}

  defp create_session(user) do
    token = :crypto.strong_rand_bytes(32) |> Base.url_encode64()
    {:ok, %{token: token, user_id: user.id, expires_at: DateTime.add(DateTime.utc_now(), 3600)}}
  end
end
```

---

## สรุป

```
Control Flow ใน Elixir:

if/unless:
├── binary conditions
├── expression (มี return value)
└── ไม่แนะนำสำหรับ complex logic

cond:
├── หลาย conditions
├── ทดแทน if/else if/else if
└── ต้องมี true -> default

case:
├── Pattern matching
├── ทรงพลังมาก
└── ใช้บ่อยที่สุด

with:
├── Chain หลาย operations
├── จัดการ error elegantly
└── แทน nested case/if

Best Practices:
├── ใช้ Pattern Matching แทน if/else ทุกเมื่อที่ทำได้
├── ใช้ with สำหรับ pipelines ที่อาจ fail
├── ให้ case เป็น single responsibility
└── หลีกเลี่ยง nested control flow
```

---

*ก่อนหน้า: [Part 05](part_05.md) | ต่อไป: [Part 07 - Recursion](part_07.md)*
