# Part 08: Enum และ Stream

## เป้าหมายการเรียนรู้

หลังจากจบ Part นี้ คุณจะสามารถ:
- ใช้ Enum module สำหรับ collection operations ได้อย่างคล่องแคล่ว
- เข้าใจความแตกต่างระหว่าง Enum และ Stream
- สร้าง lazy pipelines ด้วย Stream
- เลือกใช้ Enum หรือ Stream ได้ถูกต้อง

---

## 1. Enum Module

Enum ใช้กับ Enumerable (List, Map, Range, etc.)

### map, filter, reduce

```elixir
# map: แปลงทุก element
iex> Enum.map([1, 2, 3, 4, 5], fn x -> x * 2 end)
[2, 4, 6, 8, 10]

iex> Enum.map([1, 2, 3], &(&1 * &1))  # shorthand
[1, 4, 9]

# filter: กรอง elements
iex> Enum.filter([1, 2, 3, 4, 5, 6], &(rem(&1, 2) == 0))
[2, 4, 6]

# reject: ตรงข้ามกับ filter
iex> Enum.reject([1, 2, 3, 4, 5], &(rem(&1, 2) == 0))
[1, 3, 5]

# reduce: สรุปเป็นค่าเดียว
iex> Enum.reduce([1, 2, 3, 4, 5], 0, fn x, acc -> acc + x end)
15

# reduce ไม่มี initial value (ใช้ element แรกเป็น acc)
iex> Enum.reduce([1, 2, 3, 4, 5], fn x, acc -> acc + x end)
15
```

### Aggregations

```elixir
numbers = [3, 1, 4, 1, 5, 9, 2, 6, 5, 3, 5]

# Sum
iex> Enum.sum(numbers)
44

# Count
iex> Enum.count(numbers)
11

# Count ที่ตรงกับ condition
iex> Enum.count(numbers, &(&1 > 4))
4

# Min/Max
iex> Enum.min(numbers)
1

iex> Enum.max(numbers)
9

iex> {Enum.min(numbers), Enum.max(numbers)}
{1, 9}

iex> Enum.min_max(numbers)
{1, 9}

# Min/Max ด้วย custom function
iex> Enum.min_by(["apple", "banana", "cherry"], &String.length/1)
"apple"

iex> Enum.max_by(["apple", "banana", "cherry"], &String.length/1)
"banana"

# Average
iex> Enum.sum(numbers) / Enum.count(numbers)
4.0
```

### Searching

```elixir
users = [
  %{name: "Alice", age: 30, active: true},
  %{name: "Bob", age: 25, active: false},
  %{name: "Charlie", age: 35, active: true}
]

# find: หา element แรกที่ตรง
iex> Enum.find(users, fn u -> u.age > 28 end)
%{active: true, age: 30, name: "Alice"}

# find กับ default value
iex> Enum.find(users, :not_found, fn u -> u.age > 100 end)
:not_found

# member?: ตรวจสอบ
iex> Enum.member?([1, 2, 3], 2)
true

# any?: มีสักตัวที่ตรงไหม
iex> Enum.any?(users, fn u -> u.active end)
true

# all?: ทุกตัวตรงไหม
iex> Enum.all?(users, fn u -> u.active end)
false

# none?: ไม่มีตัวที่ตรงเลยไหม
iex> Enum.none?(users, fn u -> u.age > 100 end)
true

# find_index
iex> Enum.find_index(users, fn u -> u.name == "Bob" end)
1
```

### Sorting

```elixir
# Sort ascending (default)
iex> Enum.sort([3, 1, 4, 1, 5, 9])
[1, 1, 3, 4, 5, 9]

# Sort descending
iex> Enum.sort([3, 1, 4, 1, 5, 9], :desc)
[9, 5, 4, 3, 1, 1]

# Sort by custom function
users = [%{name: "Charlie"}, %{name: "Alice"}, %{name: "Bob"}]

iex> Enum.sort_by(users, & &1.name)
[%{name: "Alice"}, %{name: "Bob"}, %{name: "Charlie"}]

iex> Enum.sort_by(users, & &1.name, :desc)
[%{name: "Charlie"}, %{name: "Bob"}, %{name: "Alice"}]

# Sort stable (elements เท่ากันรักษา order เดิม)
iex> Enum.sort_by([%{age: 25, name: "Bob"}, %{age: 25, name: "Alice"}], & &1.age, {:asc, Date})
```

### Grouping

```elixir
words = ["apple", "banana", "cherry", "avocado", "blueberry"]

# Group by first letter
iex> Enum.group_by(words, fn word -> String.first(word) end)
%{"a" => ["apple", "avocado"], "b" => ["banana", "blueberry"], "c" => ["cherry"]}

# Group by length
iex> Enum.group_by(words, &String.length/1)
%{5 => ["apple"], 6 => ["banana", "cherry"], 7 => ["avocado"], 9 => ["blueberry"]}

# Frequencies (count occurrences)
iex> Enum.frequencies(["a", "b", "a", "c", "b", "a"])
%{"a" => 3, "b" => 2, "c" => 1}

iex> Enum.frequencies_by(["apple", "banana", "avocado"], &String.first/1)
%{"a" => 2, "b" => 1}
```

### Transformations

```elixir
# flat_map: map แล้ว flatten
iex> Enum.flat_map([1, 2, 3], fn x -> [x, x * 2] end)
[1, 2, 2, 4, 3, 6]

# flatten
iex> Enum.concat([[1, 2], [3, 4], [5]])
[1, 2, 3, 4, 5]

# zip: รวมสอง lists
iex> Enum.zip([1, 2, 3], [:a, :b, :c])
[{1, :a}, {2, :b}, {3, :c}]

# unzip: แยก
iex> Enum.unzip([{1, :a}, {2, :b}, {3, :c}])
{[1, 2, 3], [:a, :b, :c]}

# with_index: เพิ่ม index
iex> Enum.with_index(["a", "b", "c"])
[{"a", 0}, {"b", 1}, {"c", 2}]

iex> Enum.with_index(["a", "b", "c"], 1)  # start from 1
[{"a", 1}, {"b", 2}, {"c", 3}]

# chunk_every: แบ่งเป็นกลุ่ม
iex> Enum.chunk_every([1, 2, 3, 4, 5, 6], 2)
[[1, 2], [3, 4], [5, 6]]

iex> Enum.chunk_every([1, 2, 3, 4, 5], 2, 1, :discard)
[[1, 2], [2, 3], [3, 4], [4, 5]]

# take และ drop
iex> Enum.take([1, 2, 3, 4, 5], 3)
[1, 2, 3]

iex> Enum.drop([1, 2, 3, 4, 5], 3)
[4, 5]

# split
iex> Enum.split([1, 2, 3, 4, 5], 3)
{[1, 2, 3], [4, 5]}

# scan (คล้าย reduce แต่เก็บทุก step)
iex> Enum.scan([1, 2, 3, 4, 5], 0, &(&1 + &2))
[1, 3, 6, 10, 15]

# take_while, drop_while
iex> Enum.take_while([1, 2, 3, 4, 5], &(&1 < 3))
[1, 2]

iex> Enum.drop_while([1, 2, 3, 4, 5], &(&1 < 3))
[3, 4, 5]

# Dedup
iex> Enum.dedup([1, 1, 2, 2, 3, 1, 1])
[1, 2, 3, 1]  # ลบแค่ consecutive

iex> Enum.uniq([1, 1, 2, 2, 3, 1, 1])
[1, 2, 3]  # ลบทั้งหมด
```

### Reduce ขั้นสูง

```elixir
# reduce_while: หยุดได้กลางทาง
iex> Enum.reduce_while(1..100, 0, fn x, acc ->
...>   if acc + x > 20 do
...>     {:halt, acc}  # หยุด
...>   else
...>     {:cont, acc + x}  # ทำต่อ
...>   end
...> end)
21

# map_reduce: map และ reduce พร้อมกัน
iex> {mapped, total} = Enum.map_reduce([1, 2, 3, 4], 0, fn x, acc ->
...>   {x * 2, acc + x}
...> end)
{[2, 4, 6, 8], 10}
```

---

## 2. Stream Module

Stream สร้าง lazy enumerables - ไม่ประมวลผลจนกว่าจะ materialize

### Enum vs Stream

```elixir
# Enum: eager (ประมวลผลทันที)
[1, 2, 3, 4, 5]
|> Enum.map(&(&1 * 2))    # สร้าง list ใหม่ [2,4,6,8,10]
|> Enum.filter(&(&1 > 4)) # สร้าง list ใหม่ [6,8,10]
|> Enum.take(2)           # สร้าง list ใหม่ [6,8]
# สร้าง 3 intermediate lists!

# Stream: lazy (ประมวลผลเมื่อจำเป็น)
[1, 2, 3, 4, 5]
|> Stream.map(&(&1 * 2))    # สร้าง lazy operation
|> Stream.filter(&(&1 > 4)) # สร้าง lazy operation
|> Enum.take(2)             # materialize!
# ประมวลผลทีละ element จนได้ 2 ตัว

# ผลลัพธ์เหมือนกัน แต่ Stream efficient กว่า
```

### Creating Streams

```elixir
# Stream.map, filter, etc.
stream = Stream.map(1..10, &(&1 * 2))
# ยังไม่ประมวลผล!

# Materialize ด้วย Enum functions
iex> Enum.to_list(stream)
[2, 4, 6, 8, 10, 12, 14, 16, 18, 20]

# Stream.cycle: วนซ้ำ infinite
iex> Stream.cycle([1, 2, 3]) |> Enum.take(7)
[1, 2, 3, 1, 2, 3, 1]

# Stream.repeatedly: เรียก function ซ้ำๆ
iex> Stream.repeatedly(fn -> :rand.uniform(10) end) |> Enum.take(5)
[7, 2, 9, 1, 5]

# Stream.iterate: เริ่มจากค่า apply function ไปเรื่อยๆ
iex> Stream.iterate(1, &(&1 * 2)) |> Enum.take(10)
[1, 2, 4, 8, 16, 32, 64, 128, 256, 512]

# Stream.unfold: สร้าง stream จาก state
iex> Stream.unfold(0, fn n -> {n, n + 1} end) |> Enum.take(5)
[0, 1, 2, 3, 4]

# Fibonacci stream
fib = Stream.unfold({0, 1}, fn {a, b} -> {a, {b, a + b}} end)
iex> Enum.take(fib, 10)
[0, 1, 1, 2, 3, 5, 8, 13, 21, 34]
```

### Stream สำหรับ File Reading

```elixir
# อ่านไฟล์ขนาดใหญ่โดยไม่ load ทั้งหมดเข้า memory
def count_lines(file_path) do
  file_path
  |> File.stream!()
  |> Enum.count()
end

def find_matching_lines(file_path, pattern) do
  file_path
  |> File.stream!()
  |> Stream.filter(fn line -> String.contains?(line, pattern) end)
  |> Stream.map(&String.trim/1)
  |> Enum.to_list()
end

def process_large_file(file_path) do
  file_path
  |> File.stream!()
  |> Stream.map(&String.trim/1)
  |> Stream.reject(&(&1 == ""))
  |> Stream.chunk_every(100)  # process 100 lines at a time
  |> Enum.each(fn batch ->
    process_batch(batch)
  end)
end
```

### Stream.resource

```elixir
# สร้าง stream จาก external resource
def read_from_database(query) do
  Stream.resource(
    fn -> open_cursor(query) end,          # start: setup
    fn cursor ->
      case fetch_next(cursor) do
        nil -> {:halt, cursor}              # stop: no more data
        row -> {[row], cursor}             # emit: {elements, new_state}
      end
    end,
    fn cursor -> close_cursor(cursor) end  # cleanup
  )
end

# ใช้งาน
read_from_database("SELECT * FROM users")
|> Stream.filter(fn user -> user.active end)
|> Stream.map(fn user -> transform(user) end)
|> Enum.each(fn user -> process(user) end)
```

---

## 3. เปรียบเทียบ Performance

```elixir
# วัด performance
defmodule Benchmark do
  def run do
    large_list = Enum.to_list(1..1_000_000)

    # Enum approach
    {enum_time, _} = :timer.tc(fn ->
      large_list
      |> Enum.map(&(&1 * 2))
      |> Enum.filter(&(rem(&1, 6) == 0))
      |> Enum.take(10)
    end)

    # Stream approach
    {stream_time, _} = :timer.tc(fn ->
      large_list
      |> Stream.map(&(&1 * 2))
      |> Stream.filter(&(rem(&1, 6) == 0))
      |> Enum.take(10)
    end)

    IO.puts("Enum: #{enum_time / 1000}ms")
    IO.puts("Stream: #{stream_time / 1000}ms")
  end
end

# Stream เร็วกว่ามากเพราะ:
# 1. ไม่สร้าง intermediate lists
# 2. หยุดได้เมื่อได้ข้อมูลครบ
```

---

## 4. ตัวอย่างจริง: Data Pipeline

```elixir
defmodule DataPipeline do
  def process_sales_data(csv_file) do
    csv_file
    |> File.stream!()
    |> Stream.drop(1)  # skip header
    |> Stream.map(&parse_csv_line/1)
    |> Stream.filter(&valid_record?/1)
    |> Stream.map(&normalize_record/1)
    |> Stream.chunk_every(1000)  # batch insert
    |> Stream.each(&insert_batch/1)
    |> Stream.run()
  end

  defp parse_csv_line(line) do
    [date, product, quantity, price] =
      line
      |> String.trim()
      |> String.split(",")

    %{
      date: Date.from_iso8601!(date),
      product: product,
      quantity: String.to_integer(quantity),
      price: String.to_float(price)
    }
  rescue
    _ -> nil
  end

  defp valid_record?(nil), do: false
  defp valid_record?(%{quantity: q, price: p}) when q > 0 and p > 0, do: true
  defp valid_record?(_), do: false

  defp normalize_record(record) do
    %{record |
      product: String.trim(record.product),
      total: record.quantity * record.price
    }
  end

  defp insert_batch(batch) do
    IO.puts("Inserting #{length(batch)} records...")
    # ใส่ลง database
  end
end
```

---

## 5. Enum กับ Map

```elixir
products = %{
  "apple" => %{price: 10, stock: 100},
  "banana" => %{price: 5, stock: 200},
  "cherry" => %{price: 30, stock: 50}
}

# Map ผ่าน map entries
iex> Enum.map(products, fn {name, info} ->
...>   {name, info.price * info.stock}
...> end)
[{"apple", 1000}, {"banana", 1000}, {"cherry", 1500}]

# Filter map
iex> Enum.filter(products, fn {_, info} -> info.price > 8 end)
[{"apple", %{price: 10, stock: 100}}, {"cherry", %{price: 30, stock: 50}}]

# Reduce map เป็นค่าเดียว
iex> Enum.reduce(products, 0, fn {_, info}, total ->
...>   total + info.price * info.stock
...> end)
3500
```

---

## แบบฝึกหัด

### Exercise 1: Data Analysis
```elixir
transactions = [
  %{type: :income, amount: 5000, category: "salary"},
  %{type: :expense, amount: 1200, category: "rent"},
  %{type: :income, amount: 2000, category: "freelance"},
  %{type: :expense, amount: 300, category: "food"},
  %{type: :expense, amount: 150, category: "transport"},
  %{type: :income, amount: 500, category: "bonus"},
  %{type: :expense, amount: 800, category: "utilities"},
]

# 1. คำนวณรายรับรวม
# 2. คำนวณรายจ่ายรวม
# 3. คำนวณยอดคงเหลือ
# 4. หา category ที่จ่ายมากที่สุด
# 5. Group transactions ตาม type
```

### Exercise 2: Stream Processing
เขียน function ที่:
1. อ่านตัวเลข 1 ถึง 1,000,000 (ใช้ Stream)
2. กรองเฉพาะที่หารด้วย 3 ได้
3. Map เป็น square
4. หาค่า 100 ตัวแรก
5. คำนวณ sum

### Exercise 3: Custom Enum
เขียน function เหล่านี้โดยไม่ใช้ Enum:
- `my_zip/2` - zip สอง lists
- `my_chunk_every/2` - แบ่ง list เป็นกลุ่ม
- `my_flat_map/2` - flat_map

---

## เฉลย Exercise 1

```elixir
transactions = [...]

# 1. รายรับรวม
income = transactions
  |> Enum.filter(&(&1.type == :income))
  |> Enum.map(& &1.amount)
  |> Enum.sum()

# 2. รายจ่ายรวม
expenses = transactions
  |> Enum.filter(&(&1.type == :expense))
  |> Enum.map(& &1.amount)
  |> Enum.sum()

# 3. ยอดคงเหลือ
balance = income - expenses

# 4. Category ที่จ่ายมากที่สุด
top_category = transactions
  |> Enum.filter(&(&1.type == :expense))
  |> Enum.group_by(& &1.category)
  |> Enum.map(fn {cat, items} ->
    {cat, Enum.sum(Enum.map(items, & &1.amount))}
  end)
  |> Enum.max_by(fn {_, total} -> total end)

# 5. Group by type
grouped = Enum.group_by(transactions, & &1.type)
```

## เฉลย Exercise 2

```elixir
result =
  Stream.iterate(1, &(&1 + 1))
  |> Stream.filter(&(rem(&1, 3) == 0))
  |> Stream.map(&(&1 * &1))
  |> Enum.take(100)
  |> Enum.sum()
```

---

## สรุป

```
Enum vs Stream:

Enum (Eager):
├── ประมวลผลทันที
├── สร้าง intermediate results
├── เหมาะกับ small-medium collections
└── Simpler code

Stream (Lazy):
├── ประมวลผลเมื่อ materialize
├── ไม่สร้าง intermediate results
├── เหมาะกับ large files/infinite sequences
└── More efficient for early termination

Enum Functions ที่ใช้บ่อย:
├── map, filter, reduce
├── find, any?, all?, none?
├── sort, sort_by
├── group_by, frequencies
├── flat_map, concat
├── zip, with_index
└── chunk_every, take, drop

Stream Functions:
├── Stream.map, filter
├── Stream.cycle, repeatedly, iterate
├── Stream.unfold, resource
└── File.stream!
```

---

*ก่อนหน้า: [Part 07](part_07.md) | ต่อไป: [Part 09 - Pipe Operator](part_09.md)*
