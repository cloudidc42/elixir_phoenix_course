# Part 11: Structs และ Protocols ขั้นสูง

## เป้าหมายการเรียนรู้

หลังจากจบ Part นี้ คุณจะสามารถ:
- สร้างและใช้งาน Struct ได้อย่างถูกต้อง
- กำหนด default values, enforce keys, และ derive protocols
- นิยาม Protocol และ implement ให้กับ data types ต่างๆ
- ใช้ built-in protocols อย่าง Inspect, String.Chars, Enumerable, Collectable
- สร้างระบบ Shape และ JSON encoder จากโค้ดจริง

---

## 1. Struct คืออะไร?

Struct คือ Map ชนิดพิเศษที่มีชื่อและ keys ที่กำหนดไว้ล่วงหน้า

```elixir
# Map ธรรมดา - ไม่รู้ว่าต้องมี keys อะไร
user_map = %{name: "Alice", age: 30}

# Struct - รู้ชัดเจนว่าต้องมี fields อะไร
defmodule User do
  defstruct name: nil, age: nil, email: nil
end

user_struct = %User{name: "Alice", age: 30, email: "alice@example.com"}
```

---

## 2. การสร้าง Struct

### Basic Struct

```elixir
defmodule Point do
  defstruct x: 0, y: 0
end

# สร้าง instance
iex> p1 = %Point{}
%Point{x: 0, y: 0}

iex> p2 = %Point{x: 10, y: 20}
%Point{x: 10, y: 20}

# Access fields
iex> p2.x
10

iex> p2.y
20

# Pattern matching
iex> %Point{x: x, y: y} = p2
iex> x
10
iex> y
20
```

### Struct ที่มี Default Values

```elixir
defmodule Config do
  defstruct [
    host: "localhost",
    port: 4000,
    debug: false,
    max_connections: 100,
    timeout: 5000
  ]
end

iex> cfg = %Config{}
%Config{host: "localhost", port: 4000, debug: false, max_connections: 100, timeout: 5000}

iex> cfg_dev = %Config{debug: true, port: 4001}
%Config{host: "localhost", port: 4001, debug: true, max_connections: 100, timeout: 5000}
```

### Updating Struct

```elixir
iex> p = %Point{x: 1, y: 2}

# Update syntax (เหมือน Map)
iex> p2 = %{p | x: 10}
%Point{x: 10, y: 2}

iex> p3 = %{p | x: 5, y: 15}
%Point{x: 5, y: 15}

# Struct ยังคงเป็น Struct ชนิดเดิม
iex> p3.__struct__
Point
```

---

## 3. @enforce_keys

ใช้บังคับให้ต้องกำหนด keys บางตัวเมื่อสร้าง Struct

```elixir
defmodule User do
  @enforce_keys [:name, :email]
  defstruct [:name, :email, age: nil, role: :user]
end

# ต้องระบุ name และ email
iex> %User{name: "Alice", email: "alice@example.com"}
%User{name: "Alice", email: "alice@example.com", age: nil, role: :user}

# ถ้าไม่ระบุ -> error
iex> %User{name: "Bob"}
** (ArgumentError) the following keys must also be given when building struct User: [:email]
```

### ตัวอย่าง: Product Struct

```elixir
defmodule Product do
  @enforce_keys [:id, :name, :price]
  defstruct [
    :id,
    :name,
    :price,
    description: "",
    stock: 0,
    category: :general,
    active: true
  ]

  def new(id, name, price, opts \\ []) do
    %__MODULE__{
      id: id,
      name: name,
      price: price,
      description: Keyword.get(opts, :description, ""),
      stock: Keyword.get(opts, :stock, 0),
      category: Keyword.get(opts, :category, :general)
    }
  end

  def discount(product, percent) when percent >= 0 and percent <= 100 do
    new_price = product.price * (1 - percent / 100)
    %{product | price: Float.round(new_price, 2)}
  end
end

iex> p = Product.new(1, "Laptop", 999.99, stock: 10, category: :electronics)
%Product{id: 1, name: "Laptop", price: 999.99, stock: 10, ...}

iex> Product.discount(p, 10)
%Product{price: 899.99, ...}
```

---

## 4. @derive

ใช้ auto-implement protocols ให้กับ Struct

```elixir
defmodule Point do
  @derive [Inspect]
  defstruct x: 0, y: 0
end

# ที่มักใช้บ่อย
defmodule User do
  @derive {Inspect, only: [:name, :email]}  # แสดงเฉพาะบาง fields
  defstruct [:name, :email, :password_hash]
end

iex> %User{name: "Alice", email: "alice@example.com", password_hash: "secret"}
#User<name: "Alice", email: "alice@example.com", ...>
# password_hash ไม่แสดง!
```

### @derive Jason.Encoder (กับ Jason library)

```elixir
defmodule Article do
  @derive {Jason.Encoder, only: [:id, :title, :body, :published_at]}
  defstruct [:id, :title, :body, :published_at, :internal_notes]
end
```

---

## 5. Protocols

Protocol คือ mechanism สำหรับ polymorphism ใน Elixir

### นิยาม Protocol

```elixir
defprotocol Printable do
  @doc "แปลงค่าเป็น string สำหรับ print"
  def to_string(value)
end

defprotocol Serializable do
  @doc "แปลงเป็น Map สำหรับ serialize"
  def serialize(value)

  @doc "คืนค่า type ของ data"
  def type(value)
end
```

### Implement Protocol

```elixir
# Implement สำหรับ Integer
defimpl Printable, for: Integer do
  def to_string(value), do: "Int(#{value})"
end

# Implement สำหรับ Float
defimpl Printable, for: Float do
  def to_string(value), do: "Float(#{Float.round(value, 2)})"
end

# Implement สำหรับ List
defimpl Printable, for: List do
  def to_string(value) do
    items = Enum.map(value, &Printable.to_string/1)
    "[#{Enum.join(items, ", ")}]"
  end
end

# Implement สำหรับ Map
defimpl Printable, for: Map do
  def to_string(value) do
    pairs = Enum.map(value, fn {k, v} ->
      "#{k}: #{Printable.to_string(v)}"
    end)
    "{#{Enum.join(pairs, ", ")}}"
  end
end

iex> Printable.to_string(42)
"Int(42)"

iex> Printable.to_string(3.14159)
"Float(3.14)"

iex> Printable.to_string([1, 2, 3])
"[Int(1), Int(2), Int(3)]"
```

---

## 6. ตัวอย่างจริง: Shape System

```elixir
# กำหนด Protocol
defprotocol Shape do
  @doc "คำนวณพื้นที่"
  def area(shape)

  @doc "คำนวณเส้นรอบรูป"
  def perimeter(shape)

  @doc "คืน string อธิบาย shape"
  def describe(shape)
end

# Circle Struct
defmodule Circle do
  @enforce_keys [:radius]
  defstruct [:radius]

  def new(radius) when radius > 0 do
    {:ok, %__MODULE__{radius: radius}}
  end
  def new(_), do: {:error, "radius must be positive"}
end

# Rectangle Struct
defmodule Rectangle do
  @enforce_keys [:width, :height]
  defstruct [:width, :height]

  def new(width, height) when width > 0 and height > 0 do
    {:ok, %__MODULE__{width: width, height: height}}
  end
  def new(_, _), do: {:error, "dimensions must be positive"}
end

# Triangle Struct
defmodule Triangle do
  @enforce_keys [:a, :b, :c]
  defstruct [:a, :b, :c]

  def new(a, b, c) when a > 0 and b > 0 and c > 0 do
    # ตรวจสอบ triangle inequality
    if a + b > c and b + c > a and a + c > b do
      {:ok, %__MODULE__{a: a, b: b, c: c}}
    else
      {:error, "invalid triangle sides"}
    end
  end
  def new(_, _, _), do: {:error, "sides must be positive"}
end

# Implement Shape protocol สำหรับ Circle
defimpl Shape, for: Circle do
  @pi :math.pi()

  def area(%Circle{radius: r}), do: @pi * r * r

  def perimeter(%Circle{radius: r}), do: 2 * @pi * r

  def describe(%Circle{radius: r}) do
    "Circle with radius #{r} (area: #{Float.round(area(%Circle{radius: r}), 2)})"
  end
end

# Implement Shape protocol สำหรับ Rectangle
defimpl Shape, for: Rectangle do
  def area(%Rectangle{width: w, height: h}), do: w * h

  def perimeter(%Rectangle{width: w, height: h}), do: 2 * (w + h)

  def describe(%Rectangle{width: w, height: h}) do
    "Rectangle #{w}x#{h} (area: #{area(%Rectangle{width: w, height: h})})"
  end
end

# Implement Shape protocol สำหรับ Triangle (Heron's formula)
defimpl Shape, for: Triangle do
  def area(%Triangle{a: a, b: b, c: c}) do
    s = (a + b + c) / 2
    :math.sqrt(s * (s - a) * (s - b) * (s - c))
  end

  def perimeter(%Triangle{a: a, b: b, c: c}), do: a + b + c

  def describe(triangle) do
    "Triangle (#{triangle.a}, #{triangle.b}, #{triangle.c}) " <>
    "(area: #{Float.round(area(triangle), 2)})"
  end
end

# ใช้งาน
iex> {:ok, circle} = Circle.new(5)
iex> Shape.area(circle)
78.53981633974483

iex> Shape.perimeter(circle)
31.41592653589793

iex> Shape.describe(circle)
"Circle with radius 5 (area: 78.54)"

iex> {:ok, rect} = Rectangle.new(4, 6)
iex> Shape.area(rect)
24

iex> {:ok, tri} = Triangle.new(3, 4, 5)
iex> Shape.area(tri)
6.0

# Polymorphism - ใช้ function เดียวกับ shapes ทุกชนิด
shapes = [
  elem(Circle.new(3), 1),
  elem(Rectangle.new(4, 5), 1),
  elem(Triangle.new(3, 4, 5), 1)
]

Enum.each(shapes, fn shape ->
  IO.puts(Shape.describe(shape))
end)
# Circle with radius 3 (area: 28.27)
# Rectangle 4x5 (area: 20)
# Triangle (3, 4, 5) (area: 6.0)

# คำนวณพื้นที่รวม
total_area = shapes
  |> Enum.map(&Shape.area/1)
  |> Enum.sum()
  |> Float.round(2)
# 54.27
```

---

## 7. Built-in Protocol: Inspect

```elixir
defmodule BankAccount do
  defstruct [:account_number, :owner, :balance]

  defimpl Inspect do
    def inspect(%BankAccount{account_number: num, owner: owner, balance: bal}, opts) do
      # ซ่อน account number บางส่วน
      masked = String.slice(num, 0, 4) <> "****" <> String.slice(num, -4, 4)
      Inspect.Algebra.concat([
        "#BankAccount<",
        "owner: #{owner}, ",
        "account: #{masked}, ",
        "balance: #{bal}>",
      ])
    end
  end
end

iex> %BankAccount{account_number: "1234567890123456", owner: "Alice", balance: 1000.00}
#BankAccount<owner: Alice, account: 1234****3456, balance: 1000.0>
```

---

## 8. Built-in Protocol: String.Chars

ใช้เมื่อต้องการแปลง struct เป็น string ด้วย `to_string/1` หรือ string interpolation

```elixir
defmodule Temperature do
  defstruct [:value, :unit]

  def new(value, unit \\ :celsius), do: %__MODULE__{value: value, unit: unit}

  def to_celsius(%__MODULE__{value: v, unit: :celsius}), do: v
  def to_celsius(%__MODULE__{value: v, unit: :fahrenheit}), do: (v - 32) * 5 / 9
  def to_celsius(%__MODULE__{value: v, unit: :kelvin}), do: v - 273.15

  defimpl String.Chars do
    def to_string(%Temperature{value: v, unit: :celsius}),
      do: "#{v}°C"
    def to_string(%Temperature{value: v, unit: :fahrenheit}),
      do: "#{v}°F"
    def to_string(%Temperature{value: v, unit: :kelvin}),
      do: "#{v}K"
  end
end

iex> t = Temperature.new(100)
iex> "The temperature is #{t}"
"The temperature is 100°C"

iex> t2 = Temperature.new(212, :fahrenheit)
iex> to_string(t2)
"212°F"
```

---

## 9. Built-in Protocol: Enumerable

ทำให้ custom struct ใช้กับ `Enum` และ `Stream` ได้

```elixir
defmodule NumberRange do
  defstruct [:first, :last, :step]

  def new(first, last, step \\ 1) do
    %__MODULE__{first: first, last: last, step: step}
  end

  defimpl Enumerable do
    def count(%NumberRange{first: f, last: l, step: s}) do
      {:ok, max(0, div(l - f, s) + 1)}
    end

    def member?(%NumberRange{first: f, last: l, step: s}, value) do
      {:ok, value >= f and value <= l and rem(value - f, s) == 0}
    end

    def slice(%NumberRange{}) do
      {:error, __MODULE__}
    end

    def reduce(_, {:halt, acc}, _fun), do: {:halted, acc}
    def reduce(range, {:suspend, acc}, fun), do: {:suspended, acc, &reduce(range, &1, fun)}
    def reduce(%NumberRange{first: f, last: l, step: s}, {:cont, acc}, fun) when f <= l do
      reduce(%NumberRange{first: f + s, last: l, step: s}, fun.(f, acc), fun)
    end
    def reduce(%NumberRange{}, {:cont, acc}, _fun), do: {:done, acc}
  end
end

iex> range = NumberRange.new(1, 10, 2)
iex> Enum.to_list(range)
[1, 3, 5, 7, 9]

iex> Enum.sum(range)
25

iex> Enum.map(range, &(&1 * 2))
[2, 6, 10, 14, 18]

iex> Enum.member?(range, 5)
true

iex> Enum.member?(range, 4)
false
```

---

## 10. Built-in Protocol: Collectable

ทำให้ struct รับค่าจาก `Enum.into/2` ได้

```elixir
defmodule Bag do
  defstruct items: []

  def new, do: %__MODULE__{}

  defimpl Collectable do
    def into(bag) do
      collector_fn = fn
        acc, {:cont, value} -> %{acc | items: [value | acc.items]}
        acc, :done -> %{acc | items: Enum.reverse(acc.items)}
        _acc, :halt -> :ok
      end
      {bag, collector_fn}
    end
  end
end

iex> Enum.into([1, 2, 3, 4, 5], Bag.new())
%Bag{items: [1, 2, 3, 4, 5]}

iex> Enum.into(1..5, Bag.new())
%Bag{items: [1, 2, 3, 4, 5]}
```

---

## 11. ตัวอย่างจริง: JSON Encoder

```elixir
defprotocol JSONEncoder do
  @doc "แปลงค่าเป็น JSON string"
  def encode(value)
end

# String
defimpl JSONEncoder, for: BitString do
  def encode(str) do
    escaped = str
      |> String.replace("\\", "\\\\")
      |> String.replace("\"", "\\\"")
      |> String.replace("\n", "\\n")
      |> String.replace("\t", "\\t")
    "\"#{escaped}\""
  end
end

# Integer
defimpl JSONEncoder, for: Integer do
  def encode(n), do: Integer.to_string(n)
end

# Float
defimpl JSONEncoder, for: Float do
  def encode(f), do: Float.to_string(f)
end

# Boolean
defimpl JSONEncoder, for: Atom do
  def encode(true), do: "true"
  def encode(false), do: "false"
  def encode(nil), do: "null"
  def encode(atom), do: JSONEncoder.encode(Atom.to_string(atom))
end

# List -> JSON Array
defimpl JSONEncoder, for: List do
  def encode([]), do: "[]"
  def encode(list) do
    items = Enum.map(list, &JSONEncoder.encode/1)
    "[#{Enum.join(items, ",")}]"
  end
end

# Map -> JSON Object
defimpl JSONEncoder, for: Map do
  def encode(map) when map_size(map) == 0, do: "{}"
  def encode(map) do
    pairs = Enum.map(map, fn {k, v} ->
      key = cond do
        is_atom(k) -> JSONEncoder.encode(Atom.to_string(k))
        is_binary(k) -> JSONEncoder.encode(k)
        true -> JSONEncoder.encode(to_string(k))
      end
      "#{key}:#{JSONEncoder.encode(v)}"
    end)
    "{#{Enum.join(pairs, ",")}}"
  end
end

# ตัวอย่างการใช้งาน
iex> JSONEncoder.encode("hello")
"\"hello\""

iex> JSONEncoder.encode(42)
"42"

iex> JSONEncoder.encode(true)
"true"

iex> JSONEncoder.encode(nil)
"null"

iex> JSONEncoder.encode([1, "two", true, nil])
"[1,\"two\",true,null]"

iex> JSONEncoder.encode(%{name: "Alice", age: 30, active: true})
"{\"name\":\"Alice\",\"age\":30,\"active\":true}"
```

### เพิ่ม Struct Support

```elixir
defmodule Person do
  defstruct [:name, :age, :email]

  defimpl JSONEncoder do
    def encode(%Person{name: name, age: age, email: email}) do
      JSONEncoder.encode(%{
        "name" => name,
        "age" => age,
        "email" => email
      })
    end
  end
end

iex> person = %Person{name: "Alice", age: 30, email: "alice@example.com"}
iex> JSONEncoder.encode(person)
"{\"name\":\"Alice\",\"age\":30,\"email\":\"alice@example.com\"}"
```

---

## 12. Protocol Consolidation

ใน production, Elixir consolidates protocols เพื่อ performance

```elixir
# mix.exs
def project do
  [
    app: :my_app,
    version: "0.1.0",
    elixir: "~> 1.14",
    consolidate_protocols: Mix.env() != :test,
    ...
  ]
end
```

---

## 13. Any Protocol Implementation

```elixir
defprotocol Serializable do
  @fallback_to_any true
  def serialize(value)
end

# Fallback สำหรับ types ที่ไม่ได้ implement
defimpl Serializable, for: Any do
  def serialize(value), do: inspect(value)
end

# Specific implementations
defimpl Serializable, for: Integer do
  def serialize(n), do: %{type: "integer", value: n}
end

defimpl Serializable, for: BitString do
  def serialize(s), do: %{type: "string", value: s}
end

iex> Serializable.serialize(42)
%{type: "integer", value: 42}

iex> Serializable.serialize("hello")
%{type: "string", value: "hello"}

iex> Serializable.serialize(:some_atom)
":some_atom"  # ใช้ Any fallback
```

---

## 14. Exercises

### Exercise 1: Vehicle Protocol System

สร้าง Protocol สำหรับยานพาหนะ:

```elixir
# TODO: สร้าง Structs และ implement protocol
defprotocol Vehicle do
  def fuel_type(vehicle)
  def max_speed(vehicle)
  def range(vehicle)
  def describe(vehicle)
end

defmodule Car do
  defstruct [:make, :model, :fuel_capacity, :efficiency]
  # efficiency = km per liter
end

defmodule ElectricCar do
  defstruct [:make, :model, :battery_kwh, :range_per_kwh]
end

defmodule Bicycle do
  defstruct [:make, :type]  # type: :road, :mountain, :hybrid
end
```

### Exercise 2: Comparable Protocol

```elixir
defprotocol Comparable do
  def compare(a, b)
  # คืน :lt, :eq, หรือ :gt
end

# Implement สำหรับ Temperature struct จากตัวอย่างก่อนหน้า
# ต้องแปลงเป็น celsius ก่อนเปรียบเทียบ
```

### Exercise 3: Custom Collection

```elixir
# สร้าง Stack struct ที่:
# 1. Implement Enumerable (enumerate จาก top ลง bottom)
# 2. Implement Collectable (push items เข้า stack)
# 3. Implement Inspect (แสดงเป็น #Stack<[1, 2, 3]>)

defmodule Stack do
  defstruct items: []

  def new, do: %__MODULE__{}
  def push(stack, item), do: %{stack | items: [item | stack.items]}
  def pop(%Stack{items: []}), do: {:error, :empty}
  def pop(%Stack{items: [h | t]}), do: {:ok, h, %Stack{items: t}}
  def peek(%Stack{items: []}), do: {:error, :empty}
  def peek(%Stack{items: [h | _]}), do: {:ok, h}
end
```

---

## เฉลย Exercises

### เฉลย Exercise 1

```elixir
defimpl Vehicle, for: Car do
  def fuel_type(_), do: :gasoline
  def max_speed(_), do: 200  # km/h
  def range(%Car{fuel_capacity: cap, efficiency: eff}), do: cap * eff
  def describe(%Car{make: make, model: model, fuel_capacity: cap, efficiency: eff}) do
    "#{make} #{model} - Gasoline, #{cap}L tank, range: #{cap * eff}km"
  end
end

defimpl Vehicle, for: ElectricCar do
  def fuel_type(_), do: :electric
  def max_speed(_), do: 250  # km/h
  def range(%ElectricCar{battery_kwh: batt, range_per_kwh: r}), do: batt * r
  def describe(%ElectricCar{make: make, model: model, battery_kwh: batt, range_per_kwh: r}) do
    "#{make} #{model} - Electric, #{batt}kWh battery, range: #{batt * r}km"
  end
end

defimpl Vehicle, for: Bicycle do
  def fuel_type(_), do: :human_power
  def max_speed(%Bicycle{type: :road}), do: 50
  def max_speed(%Bicycle{type: :mountain}), do: 40
  def max_speed(_), do: 30
  def range(_), do: :unlimited  # depends on rider!
  def describe(%Bicycle{make: make, type: type}) do
    "#{make} #{type} bicycle - Human powered"
  end
end

# Test
car = %Car{make: "Toyota", model: "Camry", fuel_capacity: 60, efficiency: 15}
ev = %ElectricCar{make: "Tesla", model: "Model 3", battery_kwh: 75, range_per_kwh: 6}
bike = %Bicycle{make: "Trek", type: :road}

vehicles = [car, ev, bike]
Enum.each(vehicles, fn v -> IO.puts(Vehicle.describe(v)) end)
```

### เฉลย Exercise 2

```elixir
defimpl Comparable, for: Temperature do
  def compare(a, b) do
    a_celsius = Temperature.to_celsius(a)
    b_celsius = Temperature.to_celsius(b)
    cond do
      a_celsius < b_celsius -> :lt
      a_celsius > b_celsius -> :gt
      true -> :eq
    end
  end
end

# Test
t1 = Temperature.new(100, :celsius)
t2 = Temperature.new(212, :fahrenheit)  # = 100°C
t3 = Temperature.new(373.15, :kelvin)   # = 100°C
t4 = Temperature.new(50, :celsius)

Comparable.compare(t1, t4)  # :gt
Comparable.compare(t1, t2)  # :eq
Comparable.compare(t4, t1)  # :lt
```

### เฉลย Exercise 3

```elixir
defimpl Enumerable, for: Stack do
  def count(%Stack{items: items}), do: {:ok, length(items)}

  def member?(%Stack{items: items}, value), do: {:ok, value in items}

  def slice(%Stack{}), do: {:error, __MODULE__}

  def reduce(_, {:halt, acc}, _), do: {:halted, acc}
  def reduce(stack, {:suspend, acc}, fun), do: {:suspended, acc, &reduce(stack, &1, fun)}
  def reduce(%Stack{items: []}, {:cont, acc}, _), do: {:done, acc}
  def reduce(%Stack{items: [h | t]}, {:cont, acc}, fun) do
    reduce(%Stack{items: t}, fun.(h, acc), fun)
  end
end

defimpl Collectable, for: Stack do
  def into(stack) do
    {stack, fn
      acc, {:cont, value} -> Stack.push(acc, value)
      acc, :done -> acc
      _, :halt -> :ok
    end}
  end
end

defimpl Inspect, for: Stack do
  def inspect(%Stack{items: items}, _opts) do
    "#Stack<#{inspect(items)}>"
  end
end

# Test
stack = Enum.into([1, 2, 3], Stack.new())
# #Stack<[3, 2, 1]>  (push order: 1, 2, 3 → top is 3)

Enum.to_list(stack)
# [3, 2, 1]

Enum.sum(stack)
# 6
```

---

## สรุป

```
Structs:
├── defstruct [:field, field: default]
├── @enforce_keys [:required_field]
├── @derive [Protocol] หรือ {Protocol, only: [...]}
└── Update: %{struct | field: new_value}

Protocols:
├── defprotocol Name do ... end
├── defimpl Protocol, for: Type do ... end
├── @fallback_to_any true + defimpl for: Any
└── Consolidation ใน production

Built-in Protocols:
├── Inspect     - inspect() และ IEx display
├── String.Chars - to_string() และ #{interpolation}
├── Enumerable  - ใช้กับ Enum/Stream
└── Collectable - Enum.into()
```

---

*ก่อนหน้า: [Part 10](part_10.md) | ต่อไป: [Part 12 - Behaviours และ Callbacks](part_12.md)*
