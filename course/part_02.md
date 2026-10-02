# Part 02: ไวยากรณ์พื้นฐาน - ตัวแปร ชนิดข้อมูล และ Operators

## เป้าหมายการเรียนรู้

หลังจากจบ Part นี้ คุณจะสามารถ:
- เข้าใจชนิดข้อมูลพื้นฐานของ Elixir
- ใช้งาน operators ต่างๆ ได้
- เข้าใจความแตกต่างของ Elixir กับภาษาอื่น
- เขียนโค้ดพื้นฐานได้อย่างถูกต้อง

---

## 1. ตัวแปร (Variables)

### การประกาศตัวแปร

ใน Elixir ไม่มี `var`, `let`, หรือ `const` เพียงแค่เขียนชื่อตัวแปรและ bind ค่า:

```elixir
iex> name = "Alice"
"Alice"

iex> age = 30
30

iex> is_student = true
true
```

### กฎการตั้งชื่อตัวแปร

```elixir
# ถูกต้อง - ขึ้นต้นด้วยตัวพิมพ์เล็กหรือ underscore
name = "Alice"
_unused = "ignored"
user_name = "alice123"
camelCase_isOk = true  # แต่ snake_case แนะนำกว่า
count2 = 42

# ผิด - ขึ้นต้นด้วยตัวพิมพ์ใหญ่ (นั่นคือ atom หรือ module)
Name = "Alice"  # นี่คือ Pattern matching ไม่ใช่ variable
```

### Convention ในการตั้งชื่อ

```elixir
# snake_case สำหรับตัวแปรและ function
user_name = "alice"
max_retry_count = 3

# ขึ้นต้นด้วย _ สำหรับตัวแปรที่ไม่ใช้
def process(_unused_param, value) do
  value * 2
end

# CamelCase สำหรับ module names
defmodule MyModule do
end

defmodule UserAccount do
end
```

### Immutability ใน Elixir

```elixir
# ใน Elixir ข้อมูลเป็น immutable
iex> list = [1, 2, 3]
[1, 2, 3]

iex> new_list = [0 | list]  # สร้าง list ใหม่
[0, 1, 2, 3]

iex> list  # list เดิมไม่เปลี่ยน
[1, 2, 3]
```

### Rebinding ตัวแปร

```elixir
# คุณสามารถ rebind ตัวแปรได้ (แต่ค่าเดิมไม่เปลี่ยน)
iex> x = 1
1

iex> x = 2  # rebind x ไปยัง 2
2

iex> x
2
```

---

## 2. ชนิดข้อมูลพื้นฐาน (Basic Types)

### 2.1 Integers (จำนวนเต็ม)

```elixir
# Integer ธรรมดา
iex> 42
42

iex> -100
-100

# Large numbers (ใช้ _ เป็น separator)
iex> 1_000_000
1000000

iex> 1_000_000_000
1000000000

# เลขฐาน 16 (Hexadecimal)
iex> 0xFF
255

iex> 0x1F
31

# เลขฐาน 8 (Octal)
iex> 0o777
511

# เลขฐาน 2 (Binary)
iex> 0b1010
10

# ตรวจสอบชนิดข้อมูล
iex> is_integer(42)
true

iex> is_integer(3.14)
false
```

### 2.2 Floats (จำนวนทศนิยม)

```elixir
# Float ธรรมดา
iex> 3.14
3.14

iex> -2.5
-2.5

# Scientific notation
iex> 1.0e10
1.0e10

iex> 1.5e-3
0.0015

# ความแม่นยำของ Float
iex> 0.1 + 0.2
0.30000000000000004  # Floating point precision issue

# การแปลง
iex> round(3.7)
4

iex> trunc(3.7)
3

iex> Float.round(3.14159, 2)
3.14

# ตรวจสอบชนิดข้อมูล
iex> is_float(3.14)
true
```

### 2.3 Booleans

```elixir
iex> true
true

iex> false
false

# Boolean operators
iex> true and false
false

iex> true or false
true

iex> not true
false

# short-circuit evaluation
iex> false and raise("This won't run")
false

iex> true or raise("This won't run")
true

# ตรวจสอบ
iex> is_boolean(true)
true

iex> is_boolean(1)
false
```

### 2.4 Atoms

Atoms คือ constant ที่ค่าของมันคือชื่อตัวมันเอง คล้าย Symbol ใน Ruby

```elixir
# Atom พื้นฐาน
iex> :ok
:ok

iex> :error
:error

iex> :hello
:hello

# Atom ที่มีช่องว่างหรือ special chars ต้องใช้ quotes
iex> :"hello world"
:"hello world"

iex> :"user-name"
:"user-name"

# Boolean เป็น Atom ด้วย!
iex> is_atom(true)
true

iex> is_atom(false)
true

iex> is_atom(nil)
true

# nil ก็เป็น Atom
iex> nil == :nil
true

# Atoms ใช้ใน tuples เพื่อ indicate results
iex> {:ok, "result"}
{:ok, "result"}

iex> {:error, "not found"}
{:error, "not found"}

# Module names ก็เป็น Atoms
iex> is_atom(String)
true

iex> String == :"Elixir.String"
true
```

### 2.5 Strings

```elixir
# String ใช้ double quotes
iex> "Hello, World!"
"Hello, World!"

# String interpolation
iex> name = "Alice"
"Alice"

iex> "Hello, #{name}!"
"Hello, Alice!"

# String กับ expressions
iex> "2 + 2 = #{2 + 2}"
"2 + 2 = 4"

# Multiline strings
iex> """
...> Line 1
...> Line 2
...> Line 3
...> """
"Line 1\nLine 2\nLine 3\n"

# Escape sequences
iex> "Tab:\tEnd"
"Tab:\tEnd"

iex> "Newline:\nEnd"
"Newline:\nEnd"

iex> "Quote: \""
"Quote: \""

# String เป็น binary ใน Elixir
iex> is_binary("hello")
true

# String operations
iex> String.length("hello")
5

iex> String.upcase("hello")
"HELLO"

iex> String.downcase("HELLO")
"hello"

iex> String.reverse("hello")
"olleh"

iex> String.contains?("hello world", "world")
true

iex> String.replace("hello world", "world", "Elixir")
"hello Elixir"

iex> String.split("hello world foo", " ")
["hello", "world", "foo"]

iex> String.trim("  hello  ")
"hello"

# String concatenation
iex> "Hello" <> " " <> "World"
"Hello World"
```

### 2.6 Charlists

```elixir
# Charlist ใช้ single quotes (list ของ integers)
iex> 'hello'
'hello'

iex> is_list('hello')
true

# แต่ละ element เป็น integer (Unicode codepoint)
iex> 'hello' |> Enum.map(&IO.inspect/1)
104
101
108
108
111

# แปลง string <-> charlist
iex> String.to_charlist("hello")
'hello'

iex> List.to_string('hello')
"hello"

# ในทางปฏิบัติ ใช้ String (double quotes) มากกว่า
```

### 2.7 nil

```elixir
# nil คือ "ไม่มีค่า"
iex> nil
nil

iex> is_nil(nil)
true

iex> is_nil(0)
false

iex> is_nil("")
false

iex> is_nil(false)
false

# nil เป็น falsy value
iex> if nil, do: "yes", else: "no"
"no"

# เฉพาะ nil และ false เป็น falsy ใน Elixir
iex> if 0, do: "yes", else: "no"
"yes"

iex> if "", do: "yes", else: "no"
"yes"
```

---

## 3. Operators

### 3.1 Arithmetic Operators

```elixir
# บวก
iex> 5 + 3
8

# ลบ
iex> 10 - 4
6

# คูณ
iex> 4 * 5
20

# หาร (ได้ float เสมอ)
iex> 10 / 3
3.3333333333333335

# Integer division
iex> div(10, 3)
3

# Modulo (เศษจากการหาร)
iex> rem(10, 3)
1

# ยกกำลัง
iex> :math.pow(2, 10)
1024.0

# Integer arithmetic
iex> Integer.pow(2, 10)
1024

# Absolute value
iex> abs(-42)
42

# Max/Min
iex> max(10, 20)
20

iex> min(10, 20)
10
```

### 3.2 Comparison Operators

```elixir
# เท่ากัน
iex> 1 == 1
true

iex> 1 == 1.0
true  # == เปรียบเทียบค่า

# เท่ากันทั้งค่าและชนิด
iex> 1 === 1
true

iex> 1 === 1.0
false  # === เปรียบเทียบทั้งค่าและชนิด

# ไม่เท่ากัน
iex> 1 != 2
true

iex> 1 !== 1.0
true

# มากกว่า น้อยกว่า
iex> 5 > 3
true

iex> 5 < 3
false

iex> 5 >= 5
true

iex> 5 <= 4
false

# Elixir สามารถเปรียบเทียบชนิดข้อมูลต่างกันได้!
iex> 1 < :atom
true  # ordering: number < atom < reference < function < port < pid < tuple < map < list < bitstring
```

### 3.3 Boolean/Logical Operators

```elixir
# Strict boolean operators (ต้องเป็น boolean เท่านั้น)
iex> true and false
false

iex> true or false
true

iex> not true
false

# ถ้าใส่ non-boolean จะ error
iex> 1 and true
** (BadBooleanError) expected a boolean on left-side of "and", got: 1

# Relaxed operators (ทำงานกับทุก type)
iex> 1 && 2
2

iex> nil && 2
nil

iex> 1 || false
1

iex> nil || false
false

iex> !nil
true

iex> !false
true

iex> !0
false  # 0 เป็น truthy ใน Elixir!
```

### 3.4 String Operators

```elixir
# Concatenation
iex> "Hello" <> " " <> "World"
"Hello World"

# String comparison
iex> "abc" == "abc"
true

iex> "abc" < "abd"
true  # เปรียบเทียบ character by character

iex> "abc" < "abcd"
true  # string สั้นกว่า < string ยาวกว่า
```

### 3.5 List Operators

```elixir
# List concatenation
iex> [1, 2, 3] ++ [4, 5, 6]
[1, 2, 3, 4, 5, 6]

# List subtraction
iex> [1, 2, 3, 4, 5] -- [2, 4]
[1, 3, 5]

# Prepend (สร้าง list ใหม่)
iex> [0 | [1, 2, 3]]
[0, 1, 2, 3]
```

### 3.6 Pin Operator (^)

```elixir
# ปกติ = ทำ pattern matching และ bind
iex> x = 1
1

iex> x = 2  # rebind x
2

# ^ ป้องกันไม่ให้ rebind
iex> x = 1
1

iex> ^x = 1  # match x กับ 1 (ไม่ rebind)
1

iex> ^x = 2  # error เพราะ x = 1 ไม่ตรงกับ 2
** (MatchError) no match of right hand side value: 2
```

### 3.7 Pipe Operator (|>)

```elixir
# ส่ง output ของ function ซ้ายไปเป็น argument แรกของ function ขวา
iex> "hello world" |> String.split(" ") |> Enum.map(&String.upcase/1)
["HELLO", "WORLD"]

# เทียบกับแบบที่ไม่ใช้ pipe
iex> Enum.map(String.split("hello world", " "), &String.upcase/1)
["HELLO", "WORLD"]

# ตัวอย่างที่ซับซ้อนกว่า
iex> 1..10
     |> Enum.filter(fn x -> rem(x, 2) == 0 end)
     |> Enum.map(fn x -> x * x end)
     |> Enum.sum()
220
```

---

## 4. Type Checking Functions

```elixir
# ตรวจสอบชนิดข้อมูล
iex> is_integer(42)
true

iex> is_float(3.14)
true

iex> is_boolean(true)
true

iex> is_atom(:hello)
true

iex> is_binary("hello")  # String เป็น binary
true

iex> is_list([1, 2, 3])
true

iex> is_tuple({1, 2, 3})
true

iex> is_map(%{key: "value"})
true

iex> is_nil(nil)
true

iex> is_function(fn x -> x end)
true

iex> is_number(42)
true

iex> is_number(3.14)
true
```

---

## 5. Type Conversion

```elixir
# Integer ไป Float
iex> 42 / 1  # division ให้ float
42.0

iex> :erlang.float(42)
42.0

# Float ไป Integer
iex> trunc(3.7)
3

iex> round(3.7)
4

iex> floor(3.7)
3

iex> ceil(3.2)
4

# String ไป Integer/Float
iex> String.to_integer("42")
42

iex> String.to_float("3.14")
3.14

iex> Integer.parse("42abc")
{42, "abc"}  # {value, remaining_string}

# Integer/Float ไป String
iex> Integer.to_string(42)
"42"

iex> Float.to_string(3.14)
"3.14"

iex> to_string(42)
"42"

iex> to_string(:hello)
"hello"

# Atom ไป String
iex> Atom.to_string(:hello)
"hello"

# String ไป Atom (ระวัง! อย่าทำกับ user input)
iex> String.to_atom("hello")
:hello

iex> String.to_existing_atom("ok")  # ปลอดภัยกว่า - atom ต้องมีอยู่แล้ว
:ok
```

---

## 6. Special Values

### Infinity และ NaN

```elixir
# Elixir/Erlang ไม่มี infinity ใน standard syntax
# แต่ใช้งานได้ผ่าน :math
iex> :math.inf()
:infinity

iex> :math.nan()
:nan

# Division by zero
iex> 1 / 0
** (ArithmeticError) bad argument in arithmetic expression

# Float operations
iex> 1.0e308 * 2
:infinity  # ไม่เสมอไป ขึ้นกับ implementation
```

---

## 7. Sigils

Sigils เป็น shorthand syntax สำหรับสร้างข้อมูล

```elixir
# ~s - String sigil
iex> ~s(Hello "World")
"Hello \"World\""  # ไม่ต้อง escape quotes

# ~S - String sigil (ไม่ process interpolation)
iex> ~S(Hello #{name})
"Hello \#{name}"

# ~c - Charlist sigil
iex> ~c(hello)
'hello'

# ~r - Regex sigil
iex> ~r/hello/i
~r/hello/i

iex> Regex.match?(~r/hello/i, "Hello World")
true

# ~w - Word list sigil
iex> ~w(foo bar baz)
["foo", "bar", "baz"]

iex> ~w(foo bar baz)a  # a = atoms
[:foo, :bar, :baz]

iex> ~w(foo bar baz)c  # c = charlists
['foo', 'bar', 'baz']

# ~D - Date sigil
iex> ~D[2024-01-01]
~D[2024-01-01]

# ~T - Time sigil
iex> ~T[12:00:00]
~T[12:00:00]

# ~N - NaiveDateTime sigil
iex> ~N[2024-01-01 12:00:00]
~N[2024-01-01 12:00:00]

# ~U - UTC DateTime sigil
iex> ~U[2024-01-01 12:00:00Z]
~U[2024-01-01 12:00:00Z]
```

---

## 8. Module Attributes

```elixir
defmodule Config do
  # Module attribute คือ constant
  @app_name "MyApp"
  @version "1.0.0"
  @max_retries 3

  def app_info do
    "#{@app_name} v#{@version}"
  end

  def max_retries do
    @max_retries
  end
end

# ใน IEx
iex> Config.app_info()
"MyApp v1.0.0"

iex> Config.max_retries()
3
```

---

## 9. ตัวอย่างจริง: Simple Calculator

```elixir
defmodule Calculator do
  @moduledoc """
  เครื่องคิดเลขพื้นฐาน
  """

  @doc "บวกเลขสองจำนวน"
  def add(a, b) when is_number(a) and is_number(b) do
    a + b
  end

  @doc "ลบเลขสองจำนวน"
  def subtract(a, b) when is_number(a) and is_number(b) do
    a - b
  end

  @doc "คูณเลขสองจำนวน"
  def multiply(a, b) when is_number(a) and is_number(b) do
    a * b
  end

  @doc "หารเลขสองจำนวน"
  def divide(_a, 0), do: {:error, "Cannot divide by zero"}
  def divide(a, b) when is_number(a) and is_number(b) do
    {:ok, a / b}
  end

  @doc "คำนวณเปอร์เซ็นต์"
  def percentage(value, total) when total != 0 do
    (value / total) * 100
  end

  @doc "ปัดเศษ"
  def round_to(number, decimal_places) do
    Float.round(number * 1.0, decimal_places)
  end
end
```

ทดสอบ:

```elixir
iex> Calculator.add(5, 3)
8

iex> Calculator.subtract(10, 4)
6

iex> Calculator.multiply(6, 7)
42

iex> Calculator.divide(10, 3)
{:ok, 3.3333333333333335}

iex> Calculator.divide(10, 0)
{:error, "Cannot divide by zero"}

iex> Calculator.percentage(25, 100)
25.0

iex> Calculator.round_to(3.14159, 2)
3.14
```

---

## 10. ตัวอย่างจริง: String Utilities

```elixir
defmodule StringUtils do
  @doc "ตรวจสอบว่าเป็น palindrome"
  def palindrome?(str) do
    clean = str |> String.downcase() |> String.replace(~r/[^a-z0-9]/, "")
    clean == String.reverse(clean)
  end

  @doc "นับจำนวนคำ"
  def word_count(str) do
    str
    |> String.split(~r/\s+/, trim: true)
    |> length()
  end

  @doc "Capitalize ทุกคำ"
  def title_case(str) do
    str
    |> String.split(" ")
    |> Enum.map(&String.capitalize/1)
    |> Enum.join(" ")
  end

  @doc "ตัด string ให้สั้นลง"
  def truncate(str, max_length, suffix \\ "...") do
    if String.length(str) > max_length do
      String.slice(str, 0, max_length - String.length(suffix)) <> suffix
    else
      str
    end
  end

  @doc "แปลง snake_case เป็น camelCase"
  def snake_to_camel(str) do
    [first | rest] = String.split(str, "_")
    rest_capitalized = Enum.map(rest, &String.capitalize/1)
    Enum.join([first | rest_capitalized])
  end
end
```

ทดสอบ:

```elixir
iex> StringUtils.palindrome?("racecar")
true

iex> StringUtils.palindrome?("A man a plan a canal Panama")
true

iex> StringUtils.palindrome?("hello")
false

iex> StringUtils.word_count("Hello World Foo")
3

iex> StringUtils.title_case("hello world")
"Hello World"

iex> StringUtils.truncate("Hello, World!", 8)
"Hello..."

iex> StringUtils.snake_to_camel("hello_world_foo")
"helloWorldFoo"
```

---

## 11. ข้อสังเกตสำคัญ

### Elixir vs ภาษาอื่น

```elixir
# ใน JavaScript/Python: = คือ assignment
# x = 5 หมายถึง "กำหนดให้ x มีค่า 5"

# ใน Elixir: = คือ match operator
# x = 5 หมายถึง "match ค่า 5 กับ pattern x"

# ดังนั้น:
iex> 5 = 5   # match สำเร็จ
5

iex> x = 5   # bind x ไปยัง 5 (เพราะ x ยังไม่มีค่า)
5

iex> 5 = x   # match 5 กับ x (x = 5 แล้ว) - สำเร็จ
5

iex> 6 = x   # error! 6 ไม่ตรงกับ 5
** (MatchError) no match of right hand side value: 5
```

### Truthy/Falsy

```elixir
# ใน Elixir มีแค่ false และ nil เท่านั้นที่เป็น falsy
# ทุกอย่างอื่นเป็น truthy

iex> if 0, do: "truthy", else: "falsy"
"truthy"  # 0 เป็น truthy!

iex> if "", do: "truthy", else: "falsy"
"truthy"  # "" เป็น truthy!

iex> if [], do: "truthy", else: "falsy"
"truthy"  # [] เป็น truthy!

iex> if nil, do: "truthy", else: "falsy"
"falsy"   # nil เป็น falsy

iex> if false, do: "truthy", else: "falsy"
"falsy"   # false เป็น falsy
```

---

## แบบฝึกหัด

### Exercise 1: ทดสอบ Types
รันใน IEx:
```elixir
# 1. สร้างตัวแปรแต่ละชนิด
integer_val = 42
float_val = 3.14
bool_val = true
atom_val = :hello
string_val = "world"

# 2. ใช้ is_* function ตรวจสอบแต่ละตัวแปร
# 3. ลอง type conversion
```

### Exercise 2: Operators
```elixir
# คำนวณสมการเหล่านี้:
# 1. (10 + 5) * 2 / 3
# 2. rem(100, 7)
# 3. div(100, 7)
# 4. "Hello" <> ", " <> "World!"
# 5. [1,2,3] ++ [4,5,6] -- [3,4]
```

### Exercise 3: String Operations
เขียน function ที่:
1. รับ string และแปลงเป็น uppercase
2. นับจำนวน vowels (a,e,i,o,u)
3. ตรวจสอบว่า string ขึ้นต้นด้วย "Hello" หรือไม่

### Exercise 4: Calculator
เพิ่ม functions ใน Calculator module:
1. `power(base, exponent)` - ยกกำลัง
2. `square_root(n)` - รากที่สอง
3. `absolute(n)` - ค่าสัมบูรณ์
4. `factorial(n)` - แฟกทอเรียล (ลองใช้ recursion)

---

## เฉลย Exercise 3

```elixir
defmodule StringOps do
  def to_uppercase(str), do: String.upcase(str)

  def count_vowels(str) do
    str
    |> String.downcase()
    |> String.graphemes()
    |> Enum.count(fn c -> c in ["a", "e", "i", "o", "u"] end)
  end

  def starts_with_hello?(str) do
    String.starts_with?(str, "Hello")
  end
end
```

## เฉลย Exercise 4

```elixir
defmodule Calculator do
  # ... functions เดิม ...

  def power(base, exponent) do
    :math.pow(base, exponent)
  end

  def square_root(n) when n >= 0 do
    {:ok, :math.sqrt(n)}
  end
  def square_root(_n) do
    {:error, "Cannot take square root of negative number"}
  end

  def absolute(n), do: abs(n)

  def factorial(0), do: 1
  def factorial(n) when n > 0 do
    n * factorial(n - 1)
  end
  def factorial(_n), do: {:error, "Factorial requires non-negative integer"}
end
```

---

## สรุป

```
ชนิดข้อมูลใน Elixir:
├── Numbers
│   ├── Integer: 42, -100, 0xFF, 0b1010
│   └── Float: 3.14, 1.0e10
├── Boolean: true, false
├── Atom: :ok, :error, :hello
├── String: "hello" (binary UTF-8)
├── Charlist: 'hello' (list of integers)
├── nil: nil (falsy)
└── Complex (จะเรียนใน Part 03)
    ├── List: [1, 2, 3]
    ├── Tuple: {1, 2, 3}
    ├── Map: %{key: value}
    └── Keyword List: [key: value]

Operators:
├── Arithmetic: +, -, *, /, div, rem
├── Comparison: ==, !=, ===, !==, >, <, >=, <=
├── Boolean: and, or, not, &&, ||, !
├── String: <>
├── List: ++, --
├── Pin: ^
└── Pipe: |>
```

---

*ก่อนหน้า: [Part 01](part_01.md) | ต่อไป: [Part 03 - โครงสร้างข้อมูล](part_03.md)*
