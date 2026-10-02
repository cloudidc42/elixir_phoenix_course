# Part 13: Error Handling และ Fault Tolerance

## เป้าหมายการเรียนรู้

หลังจากจบ Part นี้ คุณจะสามารถ:
- ใช้ try/rescue/catch/after อย่างถูกต้อง
- เข้าใจ "Let it crash" philosophy
- สร้าง custom exceptions
- จัดการ errors แบบ functional ด้วย {:ok, result} และ {:error, reason}

---

## 1. Error Types ใน Elixir

```elixir
# 3 ประเภทหลัก:

# 1. Errors - exceptions ปกติ
raise "Something went wrong"
raise ArgumentError, message: "invalid argument"

# 2. Exits - สัญญาณให้ process หยุดทำงาน
exit(:normal)
exit({:shutdown, reason})

# 3. Throws - ใช้สำหรับ non-local control flow
throw(:done)
throw({:error, :not_found})
```

---

## 2. try/rescue

```elixir
# Rescue จาก exception
def safe_divide(a, b) do
  try do
    a / b
  rescue
    ArithmeticError -> {:error, :division_by_zero}
  end
end

# Pattern matching บน exception
def parse_integer(str) do
  try do
    String.to_integer(str)
  rescue
    e in ArgumentError ->
      {:error, "Cannot parse '#{str}': #{e.message}"}
    _ ->
      {:error, "Unknown error"}
  end
end

# Multiple rescue clauses
def process_file(path) do
  try do
    path
    |> File.read!()
    |> Jason.decode!()
  rescue
    e in File.Error ->
      {:error, {:file_not_found, path, e.message}}
    e in Jason.DecodeError ->
      {:error, {:invalid_json, e.message}}
    e ->
      {:error, {:unexpected, Exception.message(e)}}
  end
end
```

### after Clause

```elixir
def with_resource(resource, func) do
  try do
    func.(resource)
  rescue
    e -> {:error, e}
  after
    # ทำงานเสมอ ไม่ว่าจะ success หรือ fail
    cleanup(resource)
  end
end

def read_file_safely(path) do
  file = File.open!(path)
  try do
    IO.read(file, :all)
  after
    File.close(file)
  end
end
```

---

## 3. try/catch

```elixir
# Catch throws
def find_item(list, target) do
  try do
    Enum.each(list, fn item ->
      if item == target, do: throw({:found, item})
    end)
    {:error, :not_found}
  catch
    {:found, item} -> {:ok, item}
  end
end

# Catch exits
def safe_exit do
  try do
    exit(:boom)
  catch
    :exit, reason ->
      IO.puts("Process would have exited: #{inspect(reason)}")
  end
end

# Catch everything
def catch_all(func) do
  try do
    {:ok, func.()}
  rescue
    e -> {:error, {:exception, e}}
  catch
    :exit, reason -> {:error, {:exit, reason}}
    thrown -> {:error, {:thrown, thrown}}
  end
end
```

---

## 4. Custom Exceptions

```elixir
# สร้าง custom exception
defmodule MyApp.ValidationError do
  defexception [:message, :field, :value]

  def exception(opts) do
    field = Keyword.fetch!(opts, :field)
    value = Keyword.get(opts, :value)
    message = Keyword.get(opts, :message, "Validation failed for #{field}")

    %__MODULE__{
      message: message,
      field: field,
      value: value
    }
  end
end

defmodule MyApp.NotFoundError do
  defexception [:message, :resource, :id]

  def exception(opts) do
    resource = Keyword.fetch!(opts, :resource)
    id = Keyword.fetch!(opts, :id)

    %__MODULE__{
      message: "#{resource} with id #{id} not found",
      resource: resource,
      id: id
    }
  end
end

# ใช้งาน
raise MyApp.ValidationError, field: :email, value: "not-an-email", message: "Invalid email format"
raise MyApp.NotFoundError, resource: "User", id: 123
```

---

## 5. {:ok, result} / {:error, reason} Pattern

```elixir
# Pattern ที่แนะนำใน Elixir: ไม่ raise exception แต่ return tagged tuple
defmodule MyApp.UserService do
  def get_user(id) do
    case Repo.get(User, id) do
      nil -> {:error, :not_found}
      user -> {:ok, user}
    end
  end

  def create_user(attrs) do
    case %User{} |> User.changeset(attrs) |> Repo.insert() do
      {:ok, user} -> {:ok, user}
      {:error, changeset} -> {:error, {:validation_failed, changeset}}
    end
  end

  def update_user(id, attrs) do
    with {:ok, user} <- get_user(id),
         {:ok, updated} <- do_update(user, attrs) do
      {:ok, updated}
    end
  end

  defp do_update(user, attrs) do
    user
    |> User.changeset(attrs)
    |> Repo.update()
  end
end

# ใช้งานด้วย case
case UserService.get_user(123) do
  {:ok, user} -> render_user(user)
  {:error, :not_found} -> send_404()
end

# ใช้งานด้วย with
with {:ok, user} <- UserService.get_user(user_id),
     {:ok, order} <- OrderService.create_order(user, items),
     {:ok, payment} <- PaymentService.charge(user, order.total) do
  {:ok, %{user: user, order: order, payment: payment}}
else
  {:error, :not_found} -> {:error, :user_not_found}
  {:error, {:validation_failed, _}} -> {:error, :invalid_order}
  {:error, :payment_declined} -> {:error, :payment_failed}
end
```

---

## 6. "Let It Crash" Philosophy

```elixir
# ใน Elixir เราไม่ต้องป้องกัน crash ทุกอย่าง
# เพราะ Supervisor จะ restart process ให้

# แทนที่จะเขียนแบบนี้:
def get_config(key) do
  try do
    config = Application.fetch_env!(:my_app, key)
    config
  rescue
    _ -> nil  # ซ่อน error ไว้
  end
end

# เขียนแบบนี้ดีกว่า:
def get_config(key) do
  Application.fetch_env!(:my_app, key)
  # ถ้า missing config -> crash -> supervisor restart -> เห็น error ทันที
end

# หรือถ้าต้องการ default:
def get_config(key, default \\ nil) do
  Application.get_env(:my_app, key, default)
end

# Let it crash ใช้ได้ดีกับ:
# 1. Programming errors (bugs) - ควร crash แล้วแก้ bug
# 2. Initialization failures - ควร crash แล้วแจ้ง ops
# 3. Unexpected state - crash + supervisor restart ดีกว่า silent failure

# Handle อย่างชัดเจนสำหรับ:
# 1. External input validation (user forms, API input)
# 2. External service calls (HTTP, DB)
# 3. File system operations
```

---

## 7. Error Handling ใน Process

```elixir
defmodule SafeWorker do
  use GenServer

  def start_link(opts) do
    GenServer.start_link(__MODULE__, opts, name: __MODULE__)
  end

  def init(opts) do
    Process.flag(:trap_exit, true)  # รับ exit signals แทนที่จะ crash
    {:ok, %{opts: opts, error_count: 0}}
  end

  def handle_call({:process, data}, _from, state) do
    case do_process(data) do
      {:ok, result} ->
        {:reply, {:ok, result}, state}

      {:error, reason} ->
        new_state = %{state | error_count: state.error_count + 1}
        {:reply, {:error, reason}, new_state}
    end
  end

  def handle_info({:EXIT, pid, reason}, state) do
    IO.puts("Linked process #{inspect(pid)} exited: #{inspect(reason)}")
    {:noreply, state}
  end

  defp do_process(data) do
    # Processing logic
    {:ok, data}
  end
end
```

---

## 8. ตัวอย่างจริง: Resilient API Client

```elixir
defmodule MyApp.ApiClient do
  @max_retries 3
  @base_delay 1000  # milliseconds

  def get(url, opts \\ []) do
    do_request(:get, url, nil, opts, 0)
  end

  def post(url, body, opts \\ []) do
    do_request(:post, url, body, opts, 0)
  end

  defp do_request(method, url, body, opts, attempt) do
    result =
      try do
        make_request(method, url, body, opts)
      rescue
        e in [Mint.HTTPError, Mint.TransportError] ->
          {:error, {:network_error, Exception.message(e)}}
        e ->
          {:error, {:unexpected, Exception.message(e)}}
      end

    case {result, attempt} do
      {{:ok, response}, _} ->
        handle_response(response)

      {{:error, {:network_error, _}} = error, attempt} when attempt < @max_retries ->
        delay = @base_delay * :math.pow(2, attempt) |> round()
        Process.sleep(delay)
        do_request(method, url, body, opts, attempt + 1)

      {error, _} ->
        error
    end
  end

  defp make_request(method, url, body, opts) do
    headers = Keyword.get(opts, :headers, [])
    timeout = Keyword.get(opts, :timeout, 5000)

    Req.request!(
      method: method,
      url: url,
      body: body,
      headers: headers,
      receive_timeout: timeout
    )
    |> then(&{:ok, &1})
  end

  defp handle_response(%{status: status, body: body}) when status in 200..299 do
    {:ok, body}
  end

  defp handle_response(%{status: 404}) do
    {:error, :not_found}
  end

  defp handle_response(%{status: 401}) do
    {:error, :unauthorized}
  end

  defp handle_response(%{status: 429, headers: headers}) do
    retry_after = get_retry_after(headers)
    {:error, {:rate_limited, retry_after}}
  end

  defp handle_response(%{status: status, body: body}) when status >= 500 do
    {:error, {:server_error, status, body}}
  end

  defp get_retry_after(headers) do
    case List.keyfind(headers, "retry-after", 0) do
      {_, value} -> String.to_integer(value) * 1000
      nil -> 60_000
    end
  end
end
```

---

## แบบฝึกหัด

### Exercise 1: Result Monad
สร้าง `Result` module ที่มี:
- `ok(value)` - wrap success value
- `error(reason)` - wrap error
- `map(result, func)` - transform success value
- `flat_map(result, func)` - chain operations
- `get_or_else(result, default)` - get value with fallback

### Exercise 2: Validation Pipeline
```elixir
validate_user(%{
  name: "",
  email: "not-email",
  age: -5
})
# ควร return {:error, [name: "can't be blank", email: "invalid", age: "must be positive"]}
```

---

## สรุป

```
Error Handling ใน Elixir:
├── try/rescue - จัดการ exceptions
├── try/catch - จัดการ throws และ exits
├── after - cleanup เสมอ
├── {:ok, result}/{:error, reason} - functional style
└── "Let it crash" + Supervisor - resilience

Custom Exceptions:
├── defexception macro
├── message field ต้องมี
└── exception/1 callback

Best Practices:
├── ใช้ {:ok, _}/{:error, _} สำหรับ expected failures
├── ใช้ rescue สำหรับ unexpected exceptions
├── อย่า rescue ทุก exception แบบ silent
└── Let programming errors crash
```

---

## 14. Result Pipeline Pattern

```elixir
defmodule Pipeline do
  @doc """
  รัน list ของ functions แบบ pipeline
  แต่ละ function รับ {:ok, value} และคืน {:ok, value} หรือ {:error, reason}
  ถ้า step ไหน error -> หยุดทันที
  """
  def run(value, steps) do
    Enum.reduce_while(steps, {:ok, value}, fn step, {:ok, current} ->
      case step.(current) do
        {:ok, result} -> {:cont, {:ok, result}}
        {:error, _} = error -> {:halt, error}
      end
    end)
  end
end

# ตัวอย่าง: Process user registration
steps = [
  fn params -> validate_email(params) end,
  fn params -> validate_password(params) end,
  fn params -> check_duplicate_email(params) end,
  fn params -> create_user(params) end,
  fn user -> send_welcome_email(user) end
]

Pipeline.run(params, steps)
```

---

## 15. Exception ใน OTP Context

```elixir
defmodule RobustWorker do
  use GenServer

  @impl GenServer
  def handle_call({:process, data}, _from, state) do
    result = try do
      process(data)
    rescue
      e ->
        IO.puts("Error in process: #{Exception.message(e)}")
        {:error, :processing_failed}
    end
    {:reply, result, state}
  end

  defp process(data) do
    # อาจ throw exception
    if is_nil(data), do: raise ArgumentError, "data cannot be nil"
    {:ok, data}
  end
end

# GenServer จัดการ unexpected errors เอง
# ถ้าไม่ rescue -> process crash -> supervisor restart
# แต่ถ้าต้องการ log error ก่อน -> rescue
```

---

## 16. Exception Hierarchy

```elixir
# Elixir exception hierarchy
RuntimeError         # ทั่วไป - raise "message"
ArgumentError        # argument ไม่ถูกต้อง
ArithmeticError      # เลขผิดพลาด (1/0)
FunctionClauseError  # pattern match ล้มเหลว
MatchError           # = ไม่ match
KeyError             # key ไม่มีใน map
UndefinedFunctionError # เรียก function ที่ไม่มี
BadArityError        # จำนวน arguments ผิด
ErlangError          # wrapped Erlang errors

# ตัวอย่าง: catch specific hierarchy
try do
  some_operation()
rescue
  e in [ArgumentError, FunctionClauseError] ->
    IO.puts("User error: #{e.message}")
  RuntimeError = e ->
    IO.puts("Runtime: #{e.message}")
  e ->
    IO.puts("Unknown: #{inspect(e)}")
    reraise e, __STACKTRACE__
end
```

---

## 17. Error Logging Best Practices

```elixir
defmodule ErrorLogger do
  require Logger

  def safe_execute(label, fun) do
    result = fun.()
    {:ok, result}
  rescue
    e ->
      Logger.error("#{label} failed",
        error: Exception.message(e),
        stacktrace: Exception.format_stacktrace(__STACKTRACE__)
      )
      {:error, e}
  catch
    :exit, reason ->
      Logger.error("#{label} exited", reason: inspect(reason))
      {:error, {:exit, reason}}
    thrown ->
      Logger.warning("#{label} threw", value: inspect(thrown))
      {:error, {:thrown, thrown}}
  end
end

# ใช้งาน
ErrorLogger.safe_execute("database_query", fn ->
  MyApp.Repo.all(MyApp.User)
end)
```

---

## สรุป

```
Error Handling Approaches:

1. try/rescue/catch/after
   ├── rescue - จับ exceptions (raise)
   ├── catch - จับ thrown values และ exits
   └── after - cleanup (รันเสมอ)

2. {:ok, result} / {:error, reason}
   ├── Idiomatic Elixir pattern
   ├── ใช้กับ case, with
   └── !-version สำหรับ programmer errors

3. with statement
   ├── Chain operations ที่อาจ fail
   ├── else clause สำหรับ error handling
   └── สะอาดกว่า nested case

4. raise vs throw vs exit
   ├── raise - exception สำหรับ bugs
   ├── throw - flow control (ไม่ค่อยใช้)
   └── exit - terminate process (OTP)

Custom Exceptions:
├── defexception macro
├── message field ต้องมี
└── exception/1 callback

Best Practices:
├── ใช้ {:ok, _}/{:error, _} สำหรับ expected failures
├── ใช้ rescue สำหรับ unexpected exceptions
├── อย่า rescue ทุก exception แบบ silent
└── Let programming errors crash
```

---

*ก่อนหน้า: [Part 12](part_12.md) | ต่อไป: [Part 14 - Processes](part_14.md)*
