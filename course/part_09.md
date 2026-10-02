# Part 09: Pipe Operator และ Functional Programming

## เป้าหมายการเรียนรู้

หลังจากจบ Part นี้ คุณจะสามารถ:
- ใช้ Pipe Operator อย่างมีประสิทธิภาพ
- เข้าใจหลักการ Functional Programming
- เขียนโค้ดในสไตล์ functional ได้
- Compose functions ได้อย่างสง่างาม

---

## 1. Pipe Operator (|>)

Pipe operator ส่ง output ของ expression ซ้ายเป็น argument **แรก** ของ expression ขวา

```elixir
# โดยไม่มี pipe
result = Enum.sum(Enum.filter(Enum.map([1, 2, 3, 4, 5], &(&1 * 2)), &(&1 > 4)))

# ด้วย pipe (อ่านง่ายกว่ามาก)
result =
  [1, 2, 3, 4, 5]
  |> Enum.map(&(&1 * 2))
  |> Enum.filter(&(&1 > 4))
  |> Enum.sum()

# อ่านได้ว่า: "เริ่มที่ list, map แล้ว filter แล้ว sum"
```

### ตัวอย่าง Pipe

```elixir
# String processing
"  Hello, World!  "
|> String.trim()
|> String.downcase()
|> String.split(", ")
|> Enum.map(&String.capitalize/1)
|> Enum.join(" and ")
# => "Hello and world!"

# User processing
users
|> Enum.filter(&(&1.active))
|> Enum.sort_by(&(&1.name))
|> Enum.map(fn u -> %{id: u.id, display: "#{u.name} (#{u.email})"} end)
|> Enum.take(10)

# Arithmetic pipeline
42
|> :math.sqrt()
|> Float.round(2)
|> to_string()
|> String.pad_leading(10)
```

### Pipe กับ Functions ที่มีหลาย Arguments

```elixir
# Pipe ส่งค่าเป็น argument แรก
"hello world"
|> String.split(" ")     # String.split("hello world", " ")
|> Enum.map(&String.upcase/1)  # Enum.map(["hello", "world"], &String.upcase/1)

# ถ้า argument แรกไม่ใช่ที่ต้องการ ให้ใช้ anonymous function
[1, 2, 3]
|> Enum.reduce(10, fn x, acc -> acc - x end)  # reduce([1,2,3], 10, fn...)

# หรือสร้าง helper function
[1, 2, 3]
|> then(fn list -> Enum.reduce(list, 10, fn x, acc -> acc - x end) end)
```

### then/1 และ tap/1

```elixir
# then/1: apply function กับ value (เหมือน pipe แต่ flexible กว่า)
42
|> then(fn x -> x * 2 end)
|> then(&IO.inspect/1)
|> then(fn x -> x + 1 end)

# tap/1: ทำ side effect แต่ไม่เปลี่ยนค่า
[1, 2, 3]
|> tap(&IO.inspect(&1, label: "Before processing"))
|> Enum.map(&(&1 * 2))
|> tap(&IO.inspect(&1, label: "After processing"))
|> Enum.sum()
```

---

## 2. Function Composition

```elixir
# Compose functions ด้วย pipe (ที่นิยมใน Elixir)
def process_user(user) do
  user
  |> validate_user()
  |> normalize_user()
  |> enrich_user()
  |> save_user()
end

# Manual composition
defmodule Compose do
  def compose(f, g) do
    fn x -> f.(g.(x)) end
  end

  def compose_all(funcs) do
    Enum.reduce(funcs, fn f, acc -> compose(f, acc) end)
  end
end

double = &(&1 * 2)
add_one = &(&1 + 1)
square = &(&1 * &1)

process = Compose.compose(square, Compose.compose(add_one, double))
# process.(3) = square(add_one(double(3))) = square(add_one(6)) = square(7) = 49
```

---

## 3. Pure Functions

Pure function คือ function ที่:
1. Return ค่าเดียวกันเสมอสำหรับ input เดียวกัน
2. ไม่มี side effects

```elixir
# Pure function
def add(a, b), do: a + b
def square(n), do: n * n
def capitalize(str), do: String.capitalize(str)

# Impure function (มี side effects)
def save_user(user) do
  Database.insert(user)  # side effect: เขียนลง DB
  {:ok, user}
end

def get_current_time do
  DateTime.utc_now()  # ไม่ pure เพราะ return ค่าต่างกันทุกครั้ง
end

# ใน Elixir เราจัดการ side effects ด้วย:
# 1. ส่ง side effects ออกไปที่ boundary
# 2. ใช้ Process สำหรับ state
# 3. ทำให้ pure core, impure shell
```

### Pure Core Architecture

```elixir
defmodule OrderProcessor do
  # Pure functions - business logic
  def calculate_total(items) do
    Enum.reduce(items, 0, fn item, acc ->
      acc + (item.price * item.quantity)
    end)
  end

  def apply_discount(total, :none), do: total
  def apply_discount(total, percent) when is_number(percent) do
    total * (1 - percent / 100)
  end

  def calculate_tax(total, rate \\ 0.07) do
    total * rate
  end

  def validate_order(items) when length(items) > 0 do
    case Enum.find(items, fn item -> item.quantity <= 0 end) do
      nil -> {:ok, items}
      item -> {:error, "Invalid quantity for #{item.name}"}
    end
  end
  def validate_order([]), do: {:error, "Empty order"}

  # Impure functions - I/O boundary
  def process_and_save(items, discount \\ :none) do
    with {:ok, validated} <- validate_order(items),
         subtotal <- calculate_total(validated),
         discounted <- apply_discount(subtotal, discount),
         tax <- calculate_tax(discounted),
         total <- discounted + tax,
         {:ok, order} <- save_order(%{items: validated, total: total}) do
      {:ok, order}
    end
  end

  defp save_order(order) do
    # ส่วน impure: write to DB
    {:ok, Map.put(order, :id, :rand.uniform(1000))}
  end
end
```

---

## 4. Immutability

```elixir
# ข้อมูลใน Elixir เป็น immutable
original = [1, 2, 3]
modified = [0 | original]  # สร้าง list ใหม่

IO.inspect(original)   # [1, 2, 3] - ไม่เปลี่ยน
IO.inspect(modified)   # [0, 1, 2, 3]

# Map
user = %{name: "Alice", age: 30}
updated_user = %{user | age: 31}

IO.inspect(user)         # %{name: "Alice", age: 30}
IO.inspect(updated_user) # %{name: "Alice", age: 31}

# ประโยชน์ของ immutability:
# 1. Thread safety (ไม่ต้อง lock)
# 2. Easier reasoning (ค่าไม่เปลี่ยนแปลงโดยไม่คาดคิด)
# 3. Undo/redo ง่าย
# 4. Testing ง่าย
```

---

## 5. Higher-Order Functions และ Partial Application

```elixir
# Currying-like pattern
defmodule Functional do
  # สร้าง function ที่ partially applied
  def multiplier(n) do
    fn x -> x * n end
  end

  def adder(n) do
    fn x -> x + n end
  end

  def divider(divisor) do
    fn dividend -> dividend / divisor end
  end

  # ใช้งาน
  def demo do
    double = multiplier(2)
    triple = multiplier(3)
    add10 = adder(10)
    half = divider(2)

    [1, 2, 3, 4, 5]
    |> Enum.map(double)
    |> Enum.map(add10)
    |> Enum.map(half)
    |> IO.inspect()
  end
end

Functional.demo()
# [6.0, 7.0, 8.0, 9.0, 10.0]
```

---

## 6. ตัวอย่างจริง: Text Processing Pipeline

```elixir
defmodule TextAnalyzer do
  def analyze(text) do
    text
    |> clean()
    |> tokenize()
    |> remove_stopwords()
    |> frequency_count()
    |> top_words(10)
  end

  defp clean(text) do
    text
    |> String.downcase()
    |> String.replace(~r/[^\w\s]/, "")
    |> String.trim()
  end

  defp tokenize(text) do
    String.split(text, ~r/\s+/, trim: true)
  end

  @stopwords ~w(a an the is are was were be been being have has had do does did will would could should may might shall can need dare ought)

  defp remove_stopwords(words) do
    Enum.reject(words, &(&1 in @stopwords))
  end

  defp frequency_count(words) do
    Enum.frequencies(words)
  end

  defp top_words(freq_map, n) do
    freq_map
    |> Enum.sort_by(fn {_, count} -> count end, :desc)
    |> Enum.take(n)
    |> Enum.map(fn {word, count} -> %{word: word, count: count} end)
  end

  # Analysis report
  def report(text) do
    words = tokenize(clean(text))
    total = length(words)
    unique = words |> Enum.uniq() |> length()
    top = analyze(text)

    """
    === Text Analysis Report ===
    Total words: #{total}
    Unique words: #{unique}
    Vocabulary richness: #{Float.round(unique / total * 100, 1)}%

    Top #{length(top)} words:
    #{format_top_words(top)}
    """
  end

  defp format_top_words(words) do
    words
    |> Enum.with_index(1)
    |> Enum.map(fn {%{word: w, count: c}, i} ->
      "  #{i}. #{w} (#{c} times)"
    end)
    |> Enum.join("\n")
  end
end

# ทดสอบ
sample_text = """
Elixir is a dynamic functional language designed for building scalable
and maintainable applications. Elixir leverages the Erlang VM known for
running low-latency distributed and fault-tolerant systems, while also
being successfully used in web development and the embedded software domain.
"""

IO.puts(TextAnalyzer.report(sample_text))
```

---

## 7. ตัวอย่างจริง: Data Transformation

```elixir
defmodule DataTransformer do
  @doc "Transform raw API data เป็น format ที่ต้องการ"
  def transform(raw_data) do
    raw_data
    |> validate_structure()
    |> normalize_fields()
    |> enrich_data()
    |> filter_valid()
  end

  defp validate_structure(data) when is_list(data), do: data
  defp validate_structure(_), do: []

  defp normalize_fields(data) do
    Enum.map(data, fn item ->
      %{
        id: Map.get(item, "id") || Map.get(item, :id),
        name: normalize_name(item),
        email: normalize_email(item),
        created_at: parse_date(item["created_at"] || item[:created_at]),
        metadata: item
      }
    end)
  end

  defp normalize_name(item) do
    case {item["first_name"], item["last_name"], item["name"], item[:name]} do
      {first, last, _, _} when is_binary(first) and is_binary(last) ->
        "#{String.trim(first)} #{String.trim(last)}"
      {_, _, name, _} when is_binary(name) -> String.trim(name)
      {_, _, _, name} when is_binary(name) -> String.trim(name)
      _ -> "Unknown"
    end
  end

  defp normalize_email(item) do
    email = item["email"] || item[:email] || ""
    email |> String.downcase() |> String.trim()
  end

  defp parse_date(nil), do: nil
  defp parse_date(date_str) when is_binary(date_str) do
    case DateTime.from_iso8601(date_str) do
      {:ok, dt, _} -> dt
      _ -> nil
    end
  end

  defp enrich_data(items) do
    Enum.map(items, fn item ->
      Map.merge(item, %{
        valid_email: valid_email?(item.email),
        name_parts: String.split(item.name, " "),
        days_since_created: days_since(item.created_at)
      })
    end)
  end

  defp valid_email?(email) do
    String.match?(email, ~r/^[^\s]+@[^\s]+\.[^\s]+$/)
  end

  defp days_since(nil), do: nil
  defp days_since(datetime) do
    Date.diff(Date.utc_today(), DateTime.to_date(datetime))
  end

  defp filter_valid(items) do
    Enum.filter(items, fn item ->
      item.id != nil and item.valid_email
    end)
  end
end
```

---

## แบบฝึกหัด

### Exercise 1: Pipeline
เขียน pipeline ที่:
1. รับ list ของ strings
2. Filter เอาเฉพาะที่มีความยาว > 5
3. Capitalize ตัวแรก
4. Sort
5. Join ด้วย ", "

### Exercise 2: Function Composition
เขียน `compose/2` function และใช้ compose สร้าง pipeline
สำหรับ: `trim -> downcase -> remove_spaces -> encode_url`

### Exercise 3: Data Pipeline
สร้าง pipeline สำหรับ student grades:
```elixir
students = [
  %{name: "Alice", scores: [85, 90, 78, 92]},
  %{name: "Bob", scores: [70, 65, 80, 75]},
  %{name: "Charlie", scores: [95, 98, 92, 99]},
]
```
คำนวณ: average, grade (A/B/C/D/F), rank

---

## เฉลย Exercise 1

```elixir
def process_strings(strings) do
  strings
  |> Enum.filter(&(String.length(&1) > 5))
  |> Enum.map(&String.capitalize/1)
  |> Enum.sort()
  |> Enum.join(", ")
end
```

## เฉลย Exercise 3

```elixir
defmodule GradeBook do
  def process(students) do
    students
    |> Enum.map(&calculate_average/1)
    |> Enum.map(&assign_grade/1)
    |> Enum.sort_by(& &1.average, :desc)
    |> Enum.with_index(1)
    |> Enum.map(fn {student, rank} -> Map.put(student, :rank, rank) end)
  end

  defp calculate_average(student) do
    avg = Enum.sum(student.scores) / length(student.scores)
    Map.put(student, :average, Float.round(avg, 1))
  end

  defp assign_grade(%{average: avg} = student) do
    grade = cond do
      avg >= 90 -> "A"
      avg >= 80 -> "B"
      avg >= 70 -> "C"
      avg >= 60 -> "D"
      true -> "F"
    end
    Map.put(student, :grade, grade)
  end
end
```

---

## สรุป

```
Pipe Operator (|>):
├── ส่ง value เป็น argument แรก
├── ทำให้โค้ดอ่านจากบนลงล่าง
├── then/1 สำหรับ complex transforms
└── tap/1 สำหรับ debugging

Functional Programming Principles:
├── Pure functions (no side effects)
├── Immutable data
├── Function composition
├── Higher-order functions
└── Declarative over imperative

Best Practices:
├── ใช้ pipe สำหรับ multi-step transformations
├── Extract helper functions ที่ชัดเจน
├── Pure core, impure at boundaries
└── Small, single-purpose functions
```

---

*ก่อนหน้า: [Part 08](part_08.md) | ต่อไป: [Part 10 - Strings, Binaries](part_10.md)*
