# Part 21: Macros และ Metaprogramming

## เป้าหมายการเรียนรู้

หลังจากจบ Part นี้ คุณจะสามารถ:
- เข้าใจว่า Macro คืออะไรและทำงานอย่างไร
- ใช้ `quote` และ `unquote` ได้
- เขียน `defmacro` ได้
- เข้าใจ AST (Abstract Syntax Tree) ของ Elixir
- ใช้ `__using__` macro ได้
- สร้าง DSL (Domain Specific Language) อย่างง่าย

---

## 1. Metaprogramming คืออะไร?

Metaprogramming คือการเขียนโค้ดที่ **สร้างโค้ดอื่น** ขึ้นมาอีกที ใน Elixir เราทำสิ่งนี้ผ่าน **Macros**

ความแตกต่างระหว่าง Function กับ Macro:
- **Function**: รับ values แล้วคืน value
- **Macro**: รับ **AST** แล้วคืน **AST** (ทำงานตอน compile time)

```elixir
# Function ทำงานตอน runtime
def add(a, b), do: a + b

# Macro ทำงานตอน compile time
defmacro my_if(condition, do: body) do
  quote do
    case unquote(condition) do
      true -> unquote(body)
      false -> nil
    end
  end
end
```

---

## 2. AST (Abstract Syntax Tree)

ทุกโค้ด Elixir ถูกแปลงเป็น AST ก่อน compile ใช้ `quote` เพื่อดู AST

```elixir
iex> quote do: 1 + 2
{:+, [context: Elixir, imports: [{2, Kernel}]], [1, 2]}

iex> quote do: IO.puts("hello")
{{:., [], [{:__aliases__, [alias: false], [:IO]}, :puts]}, [], ["hello"]}

iex> quote do: x = 1 + 2
{:=, [], [{:x, [], Elixir}, {:+, [context: Elixir, imports: [{2, Kernel}]], [1, 2]}]}
```

### โครงสร้างของ AST Node

AST node มีรูปแบบ: `{atom_or_tuple, metadata, args}`

```elixir
# {function_name, metadata, arguments}
{:+, [], [1, 2]}

# Atom ธรรมดา
:hello

# Integer
42

# String
"hello"

# Variable: {var_name, metadata, context}
{:x, [], Elixir}

# Function call: {function_name, metadata, args}
{:foo, [], [1, 2, 3]}

# Module function call:
# {{:., meta, [module, function]}, meta, args}
{{:., [], [{:__aliases__, [], [:String]}, :upcase]}, [], ["hello"]}
```

---

## 3. quote/unquote

### quote - แปลงโค้ดเป็น AST

```elixir
iex> quoted = quote do
...>   x + y
...> end
{:+, [context: Elixir, imports: [{2, Kernel}]], [{:x, [], Elixir}, {:y, [], Elixir}]}

# quote ทำให้ Elixir ไม่ evaluate expression
iex> quote do: 1 + 1
{:+, [context: Elixir, imports: [{2, Kernel}]], [1, 1]}
# ไม่ใช่ 2 แต่เป็น AST ของ 1 + 1
```

### unquote - inject ค่าเข้าไปใน quoted expression

```elixir
iex> a = 42
42

iex> quote do
...>   x + unquote(a)
...> end
{:+, [context: Elixir, imports: [{2, Kernel}]], [{:x, [], Elixir}, 42]}

# unquote ทำให้ a ถูก evaluate แล้ว inject ค่าเข้าไป
```

### unquote_splicing - inject list

```elixir
iex> args = [1, 2, 3]
[1, 2, 3]

iex> quote do
...>   my_function(unquote_splicing(args))
...> end
{:my_function, [], [1, 2, 3]}
```

### ตัวอย่างการใช้ quote/unquote

```elixir
defmodule MyMath do
  # สร้าง macro ที่ generate code
  defmacro squared(x) do
    quote do
      unquote(x) * unquote(x)
    end
  end

  defmacro cubed(x) do
    quote do
      unquote(x) * unquote(x) * unquote(x)
    end
  end
end

# การใช้งาน
import MyMath

IO.puts(squared(4))   # => 16
IO.puts(cubed(3))     # => 27

# macro จะ expand เป็น:
# squared(4) -> 4 * 4
# cubed(3)   -> 3 * 3 * 3
```

---

## 4. defmacro

### Macro พื้นฐาน

```elixir
defmodule ControlFlow do
  # สร้าง unless macro (ตรงข้ามกับ if)
  defmacro unless(condition, do: body) do
    quote do
      if !unquote(condition) do
        unquote(body)
      end
    end
  end

  # สร้าง while loop
  defmacro while(condition, do: body) do
    quote do
      Stream.repeatedly(fn -> unquote(condition) end)
      |> Enum.reduce_while(nil, fn cond, _acc ->
        if cond do
          unquote(body)
          {:cont, nil}
        else
          {:halt, nil}
        end
      end)
    end
  end

  # retry macro - retry N times
  defmacro retry(n, do: body) do
    quote do
      Enum.reduce_while(1..unquote(n), :error, fn _i, _acc ->
        try do
          result = unquote(body)
          {:halt, {:ok, result}}
        rescue
          _ -> {:cont, :error}
        end
      end)
    end
  end
end
```

```elixir
import ControlFlow

x = 5

unless x > 10 do
  IO.puts("x is not greater than 10")
end
# => "x is not greater than 10"

result = retry 3 do
  # ลอง connect
  {:ok, "connected"}
end
# => {:ok, "connected"}
```

### Macro ที่รับ block หลายอัน

```elixir
defmodule MyControl do
  defmacro my_if(condition, clauses) do
    do_clause = Keyword.get(clauses, :do, nil)
    else_clause = Keyword.get(clauses, :else, nil)

    quote do
      case unquote(condition) do
        x when x in [false, nil] -> unquote(else_clause)
        _ -> unquote(do_clause)
      end
    end
  end
end

import MyControl

my_if true do
  IO.puts("truthy!")
else
  IO.puts("falsy!")
end
# => "truthy!"
```

### Macro Hygiene

Elixir macros เป็น **hygienic** - variable ใน macro ไม่ leak ออกมา

```elixir
defmodule HygieneExample do
  defmacro create_var do
    quote do
      x = 42  # x นี้จะไม่ leak ออกไป
    end
  end
end

import HygieneExample

x = 1
create_var()
IO.puts(x)  # => 1 (ไม่ใช่ 42!)
```

ถ้าต้องการให้ variable leak ออกมา ใช้ `var!`

```elixir
defmodule LeakyMacro do
  defmacro set_x(value) do
    quote do
      var!(x) = unquote(value)  # ใช้ var! เพื่อ inject ลงใน caller's context
    end
  end
end

import LeakyMacro

set_x(100)
IO.puts(x)  # => 100
```

---

## 5. __using__ Macro

`__using__` เป็น macro พิเศษที่ถูกเรียกเมื่อใช้ `use ModuleName`

```elixir
defmodule Greetable do
  defmacro __using__(opts) do
    greeting = Keyword.get(opts, :greeting, "Hello")

    quote do
      def greet(name) do
        "#{unquote(greeting)}, #{name}!"
      end

      def greet_many(names) do
        Enum.map(names, &greet/1)
      end
    end
  end
end

defmodule EnglishGreeter do
  use Greetable, greeting: "Hello"
end

defmodule ThaiGreeter do
  use Greetable, greeting: "สวัสดี"
end

IO.puts(EnglishGreeter.greet("Alice"))         # => "Hello, Alice!"
IO.puts(ThaiGreeter.greet("สมชาย"))             # => "สวัสดี, สมชาย!"
IO.inspect(EnglishGreeter.greet_many(["Alice", "Bob"]))  # => ["Hello, Alice!", "Hello, Bob!"]
```

### ตัวอย่าง: Behaviour with __using__

```elixir
defmodule Worker do
  @callback process(term()) :: {:ok, term()} | {:error, String.t()}

  defmacro __using__(_opts) do
    quote do
      @behaviour Worker

      def run(data) do
        case process(data) do
          {:ok, result} ->
            IO.puts("Success: #{inspect(result)}")
            {:ok, result}
          {:error, reason} ->
            IO.puts("Error: #{reason}")
            {:error, reason}
        end
      end

      defoverridable run: 1
    end
  end
end

defmodule EmailWorker do
  use Worker

  @impl Worker
  def process(%{email: email, subject: subject}) do
    # จำลองการส่งอีเมล
    IO.puts("Sending email to #{email}: #{subject}")
    {:ok, %{sent_at: DateTime.utc_now()}}
  end
end

defmodule DataWorker do
  use Worker

  @impl Worker
  def process(data) when is_list(data) do
    result = Enum.map(data, &(&1 * 2))
    {:ok, result}
  end

  def process(_), do: {:error, "Data must be a list"}
end

EmailWorker.run(%{email: "test@example.com", subject: "Hello"})
DataWorker.run([1, 2, 3, 4, 5])
# => Success: [2, 4, 6, 8, 10]
```

---

## 6. AST Manipulation

### วิเคราะห์และแปลง AST

```elixir
defmodule ASTInspector do
  # ดูว่า AST มีรูปร่างอย่างไร
  def inspect_ast(ast) do
    Macro.to_string(ast)
  end

  # นับจำนวน function calls ใน AST
  def count_calls(ast) do
    {_, count} = Macro.prewalk(ast, 0, fn
      {name, _meta, args} = node, acc when is_atom(name) and is_list(args) ->
        {node, acc + 1}
      node, acc ->
        {node, acc}
    end)
    count
  end

  # แทนที่ + ด้วย - ใน AST
  def negate_additions(ast) do
    Macro.postwalk(ast, fn
      {:+, meta, args} -> {:-, meta, args}
      node -> node
    end)
  end
end
```

```elixir
# ทดสอบ
ast = quote do: (1 + 2) * (3 + 4)

IO.puts(ASTInspector.inspect_ast(ast))
# => "(1 + 2) * (3 + 4)"

negated = ASTInspector.negate_additions(ast)
IO.puts(ASTInspector.inspect_ast(negated))
# => "(1 - 2) * (3 - 4)"
```

### Macro.prewalk และ Macro.postwalk

```elixir
# prewalk: เดิน AST จาก root ลงไป child
# postwalk: เดิน AST จาก leaf ขึ้นมา root

defmodule Transform do
  def add_logging(ast) do
    Macro.prewalk(ast, fn
      {:def, meta, [head | body]} ->
        {func_name, _, _} = head
        logged_body = quote do
          IO.puts("Calling: #{unquote(to_string(func_name))}")
          unquote_splicing(body)
        end
        {:def, meta, [head, [do: logged_body]]}
      node ->
        node
    end)
  end
end
```

---

## 7. ตัวอย่างจริง: HTML DSL

```elixir
defmodule HTML do
  @doc """
  HTML DSL สำหรับสร้าง HTML markup
  """

  defmacro html(do: block) do
    quote do
      "<html>#{unquote(block)}</html>"
    end
  end

  defmacro head(do: block) do
    quote do
      "<head>#{unquote(block)}</head>"
    end
  end

  defmacro body(do: block) do
    quote do
      "<body>#{unquote(block)}</body>"
    end
  end

  defmacro div(attrs \\ [], do: block) do
    class = Keyword.get(attrs, :class, "")
    class_attr = if class != "", do: ~s( class="#{class}"), else: ""

    quote do
      "<div#{unquote(class_attr)}>#{unquote(block)}</div>"
    end
  end

  defmacro p(do: block) do
    quote do
      "<p>#{unquote(block)}</p>"
    end
  end

  defmacro h1(do: block) do
    quote do
      "<h1>#{unquote(block)}</h1>"
    end
  end

  defmacro h2(do: block) do
    quote do
      "<h2>#{unquote(block)}</h2>"
    end
  end

  defmacro a(href, do: block) do
    quote do
      "<a href=\"#{unquote(href)}\">#{unquote(block)}</a>"
    end
  end

  defmacro ul(do: block) do
    quote do
      "<ul>#{unquote(block)}</ul>"
    end
  end

  defmacro li(do: block) do
    quote do
      "<li>#{unquote(block)}</li>"
    end
  end

  defmacro span(text) do
    quote do
      "<span>#{unquote(text)}</span>"
    end
  end
end
```

```elixir
import HTML

page = html do
  head do
    "<title>My Page</title>"
  end <>
  body do
    div class: "container" do
      h1 do
        "Welcome to My Site"
      end <>
      p do
        "This is built with Elixir macros!"
      end <>
      ul do
        li do
          a "https://elixir-lang.org" do
            "Elixir"
          end
        end <>
        li do
          a "https://phoenixframework.org" do
            "Phoenix"
          end
        end
      end
    end
  end
end

IO.puts(page)
```

Output:
```html
<html><head><title>My Page</title></head><body><div class="container"><h1>Welcome to My Site</h1><p>This is built with Elixir macros!</p><ul><li><a href="https://elixir-lang.org">Elixir</a></li><li><a href="https://phoenixframework.org">Phoenix</a></li></ul></div></body></html>
```

---

## 8. ตัวอย่างจริง: Validation DSL

```elixir
defmodule Validation do
  @doc """
  DSL สำหรับ data validation
  """

  defmacro __using__(_opts) do
    quote do
      import Validation
      Module.register_attribute(__MODULE__, :validations, accumulate: true)
      @before_compile Validation
    end
  end

  defmacro validates(field, opts) do
    quote do
      @validations {unquote(field), unquote(opts)}
    end
  end

  defmacro __before_compile__(env) do
    validations = Module.get_attribute(env.module, :validations)

    validation_clauses = Enum.map(validations, fn {field, opts} ->
      build_validation(field, opts)
    end)

    quote do
      def validate(data) do
        errors =
          unquote(validation_clauses)
          |> List.flatten()
          |> Enum.reject(&is_nil/1)

        if Enum.empty?(errors) do
          {:ok, data}
        else
          {:error, errors}
        end
      end

      defp run_validations(data) do
        unquote(validation_clauses)
      end
    end
  end

  def build_validation(field, opts) do
    Enum.map(opts, fn
      {:required, true} ->
        quote do
          case Map.get(data, unquote(field)) do
            nil -> "#{unquote(field)} is required"
            "" -> "#{unquote(field)} is required"
            _ -> nil
          end
        end

      {:min_length, min} ->
        quote do
          value = Map.get(data, unquote(field), "")
          if String.length(to_string(value)) < unquote(min) do
            "#{unquote(field)} must be at least #{unquote(min)} characters"
          else
            nil
          end
        end

      {:max_length, max} ->
        quote do
          value = Map.get(data, unquote(field), "")
          if String.length(to_string(value)) > unquote(max) do
            "#{unquote(field)} must be at most #{unquote(max)} characters"
          else
            nil
          end
        end

      {:format, regex} ->
        quote do
          value = to_string(Map.get(data, unquote(field), ""))
          if Regex.match?(unquote(Macro.escape(regex)), value) do
            nil
          else
            "#{unquote(field)} has invalid format"
          end
        end

      _ ->
        quote do: nil
    end)
  end
end
```

```elixir
defmodule UserValidator do
  use Validation

  validates :name, required: true, min_length: 2, max_length: 50
  validates :email, required: true, format: ~r/^[^\s@]+@[^\s@]+\.[^\s@]+$/
  validates :password, required: true, min_length: 8
end

# ทดสอบ
valid_data = %{name: "Alice", email: "alice@example.com", password: "secret123"}
IO.inspect(UserValidator.validate(valid_data))
# => {:ok, %{name: "Alice", email: "alice@example.com", password: "secret123"}}

invalid_data = %{name: "A", email: "not-an-email", password: "123"}
IO.inspect(UserValidator.validate(invalid_data))
# => {:error, ["name must be at least 2 characters", "email has invalid format", "password must be at least 8 characters"]}

missing_data = %{name: "Bob"}
IO.inspect(UserValidator.validate(missing_data))
# => {:error, ["email is required", "password is required"]}
```

---

## 9. Macro กับ Module Attributes

```elixir
defmodule RouteDefiner do
  defmacro __using__(_opts) do
    quote do
      Module.register_attribute(__MODULE__, :routes, accumulate: true)
      import RouteDefiner
      @before_compile RouteDefiner
    end
  end

  defmacro get(path, do: handler) do
    quote do
      @routes {:get, unquote(path), fn conn -> unquote(handler) end}
    end
  end

  defmacro post(path, do: handler) do
    quote do
      @routes {:post, unquote(path), fn conn -> unquote(handler) end}
    end
  end

  defmacro __before_compile__(env) do
    routes = Module.get_attribute(env.module, :routes)

    route_clauses = Enum.map(routes, fn {method, path, handler} ->
      quote do
        def dispatch({unquote(method), unquote(path)}, conn) do
          unquote(handler).(conn)
        end
      end
    end)

    quote do
      unquote_splicing(route_clauses)

      def dispatch(_, _conn), do: {:error, :not_found}

      def routes do
        unquote(Enum.map(routes, fn {method, path, _} -> {method, path} end))
      end
    end
  end
end

defmodule MyApp.Router do
  use RouteDefiner

  get "/" do
    "Welcome to the home page"
  end

  get "/about" do
    "About us page"
  end

  post "/users" do
    "Create user handler"
  end
end

IO.inspect(MyApp.Router.dispatch({:get, "/"}, %{}))
# => "Welcome to the home page"

IO.inspect(MyApp.Router.dispatch({:post, "/users"}, %{}))
# => "Create user handler"

IO.inspect(MyApp.Router.dispatch({:get, "/missing"}, %{}))
# => {:error, :not_found}

IO.inspect(MyApp.Router.routes())
# => [{:get, "/"}, {:get, "/about"}, {:post, "/users"}]
```

---

## 10. Exercises

### Exercise 1: สร้าง `debug` macro

สร้าง macro `debug` ที่พิมพ์ทั้ง expression และ value ของมัน:

```elixir
x = 42
debug(x + 1)
# ควร print: "x + 1 = 43"
```

**เฉลย:**

```elixir
defmodule Debug do
  defmacro debug(expr) do
    expr_string = Macro.to_string(expr)

    quote do
      result = unquote(expr)
      IO.puts("#{unquote(expr_string)} = #{inspect(result)}")
      result
    end
  end
end

import Debug

x = 42
debug(x + 1)
# => "x + 1 = 43"

debug(String.upcase("hello"))
# => "String.upcase(\"hello\") = \"HELLO\""

debug([1, 2, 3] |> Enum.sum())
# => "[1, 2, 3] |> Enum.sum() = 6"
```

### Exercise 2: สร้าง `timed` macro

สร้าง macro ที่วัดเวลา execution:

```elixir
timed do
  :timer.sleep(100)
  "done"
end
# ควร print: "Execution took: 100ms"
# และคืน "done"
```

**เฉลย:**

```elixir
defmodule Timer do
  defmacro timed(do: block) do
    quote do
      start = System.monotonic_time(:millisecond)
      result = unquote(block)
      elapsed = System.monotonic_time(:millisecond) - start
      IO.puts("Execution took: #{elapsed}ms")
      result
    end
  end
end

import Timer

result = timed do
  :timer.sleep(50)
  1 + 1
end

IO.puts("Result: #{result}")
# => "Execution took: ~50ms"
# => "Result: 2"
```

### Exercise 3: สร้าง Config DSL

สร้าง module ที่ใช้ macro สร้าง configuration DSL:

```elixir
defmodule AppConfig do
  use Config.DSL

  set :database_url, "postgres://localhost/myapp"
  set :port, 4000
  set :debug, true
end

AppConfig.get(:port)  # => 4000
AppConfig.all()       # => %{database_url: "...", port: 4000, debug: true}
```

**เฉลย:**

```elixir
defmodule Config.DSL do
  defmacro __using__(_opts) do
    quote do
      import Config.DSL
      Module.register_attribute(__MODULE__, :config_values, accumulate: true)
      @before_compile Config.DSL
    end
  end

  defmacro set(key, value) do
    quote do
      @config_values {unquote(key), unquote(value)}
    end
  end

  defmacro __before_compile__(env) do
    config_values = Module.get_attribute(env.module, :config_values)
    config_map = Map.new(config_values)

    get_clauses = Enum.map(config_values, fn {key, value} ->
      quote do
        def get(unquote(key)), do: unquote(value)
      end
    end)

    quote do
      unquote_splicing(get_clauses)

      def get(_key), do: nil

      def all(), do: unquote(Macro.escape(config_map))
    end
  end
end

defmodule AppConfig do
  use Config.DSL

  set :database_url, "postgres://localhost/myapp"
  set :port, 4000
  set :debug, true
end

IO.puts(AppConfig.get(:port))    # => 4000
IO.inspect(AppConfig.all())      # => %{database_url: "postgres://localhost/myapp", debug: true, port: 4000}
```

---

## สรุป

```
Metaprogramming ใน Elixir:
├── AST: โครงสร้างข้อมูลที่แทน Elixir code
│   ├── {atom, meta, args}
│   ├── literals: integers, strings, atoms
│   └── nested structure
├── quote: แปลง code เป็น AST
│   └── unquote: inject value เข้าใน quoted expression
├── defmacro: สร้าง macro
│   ├── รับ AST, คืน AST
│   ├── ทำงาน compile time
│   └── Hygienic โดย default
├── __using__: macro สำหรับ use ModuleName
│   ├── inject code เข้า caller module
│   └── ใช้กับ @before_compile
└── Use cases:
    ├── DSL สร้าง HTML, routing, validation
    ├── Code generation
    ├── Logging/debugging tools
    └── Testing helpers
```

---

*ก่อนหน้า: [Part 20](part_20.md) | ต่อไป: [Part 22 - GenStage และ Flow](part_22.md)*
