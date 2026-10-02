# Part 07: Recursion และ Tail Call Optimization

## เป้าหมายการเรียนรู้

หลังจากจบ Part นี้ คุณจะสามารถ:
- เขียน recursive functions ใน Elixir ได้
- เข้าใจ tail call optimization (TCO)
- เปลี่ยน recursion ธรรมดาเป็น tail recursive
- ใช้ accumulator pattern

---

## 1. Recursion พื้นฐาน

Elixir ไม่มี loop แบบ imperative (for/while) แต่ใช้ recursion แทน

```elixir
# Recursion ต้องมี:
# 1. Base case (กรณีหยุด)
# 2. Recursive case (เรียกตัวเอง)

defmodule Basics do
  # Factorial
  def factorial(0), do: 1                              # base case
  def factorial(n) when n > 0, do: n * factorial(n-1) # recursive case

  # Sum of list
  def sum([]), do: 0                         # base case: empty list
  def sum([head | tail]), do: head + sum(tail) # recursive case

  # Length of list
  def length([]), do: 0
  def length([_ | tail]), do: 1 + length(tail)

  # Reverse a list
  def reverse([]), do: []
  def reverse([head | tail]), do: reverse(tail) ++ [head]
end

iex> Basics.factorial(5)
120

iex> Basics.sum([1, 2, 3, 4, 5])
15

iex> Basics.length([1, 2, 3])
3

iex> Basics.reverse([1, 2, 3])
[3, 2, 1]
```

### Call Stack Visualization

```
factorial(4)
= 4 * factorial(3)
= 4 * (3 * factorial(2))
= 4 * (3 * (2 * factorial(1)))
= 4 * (3 * (2 * (1 * factorial(0))))
= 4 * (3 * (2 * (1 * 1)))
= 4 * (3 * (2 * 1))
= 4 * (3 * 2)
= 4 * 6
= 24

stack ใหญ่ขึ้นเรื่อยๆ จนถึง base case แล้วค่อย unwind
```

---

## 2. Tail Call Optimization (TCO)

ปัญหาของ recursion ธรรมดา: stack overflow เมื่อ n ใหญ่มาก

```elixir
# นี่ไม่ใช่ tail call เพราะต้องทำ * หลังจาก recursive call กลับมา
def factorial(n), do: n * factorial(n-1)
# stack: factorial(1000000) จะทำให้ stack overflow
```

### Tail Call คืออะไร?

Tail call คือเมื่อ recursive call เป็น **operation สุดท้าย** ของ function

```elixir
# ไม่ใช่ tail call (ต้องทำ * หลัง recursive call)
def factorial(n), do: n * factorial(n - 1)
#                     ^^^
#                     operation นี้ทำหลัง recursive call

# เป็น tail call (recursive call เป็น operation สุดท้าย)
def factorial(n, acc), do: factorial(n - 1, n * acc)
#                         ^^^^^^^^^
#                         recursive call เป็น operation สุดท้าย
```

### Tail Recursive Factorial

```elixir
defmodule TailRecursive do
  # Public interface - accumulator เริ่มที่ 1
  def factorial(n), do: factorial(n, 1)

  # Private tail recursive helper
  defp factorial(0, acc), do: acc
  defp factorial(n, acc) when n > 0 do
    factorial(n - 1, n * acc)
  end
end

# Stack visualization:
# factorial(4, 1)
# factorial(3, 4)   ← ไม่ต้อง remember ว่า n = 4
# factorial(2, 12)  ← ไม่ต้อง remember ว่า n = 3
# factorial(1, 24)  ← ไม่ต้อง remember ว่า n = 2
# factorial(0, 24)  ← return 24
# Stack ขนาดเท่าเดิมตลอด!

iex> TailRecursive.factorial(1_000_000)  # จะทำงานได้ ไม่ crash
# ตัวเลขใหญ่มาก...
```

### Accumulator Pattern

```elixir
defmodule ListOps do
  # Sum ด้วย accumulator
  def sum(list), do: sum(list, 0)
  defp sum([], acc), do: acc
  defp sum([head | tail], acc), do: sum(tail, acc + head)

  # Length ด้วย accumulator
  def length(list), do: length(list, 0)
  defp length([], acc), do: acc
  defp length([_ | tail], acc), do: length(tail, acc + 1)

  # Reverse ด้วย accumulator (efficient!)
  def reverse(list), do: reverse(list, [])
  defp reverse([], acc), do: acc
  defp reverse([head | tail], acc), do: reverse(tail, [head | acc])

  # Map ด้วย accumulator (สังเกต: ต้อง reverse ตอนท้าย)
  def map(list, func), do: map(list, func, [])
  defp map([], _func, acc), do: Enum.reverse(acc)
  defp map([head | tail], func, acc), do: map(tail, func, [func.(head) | acc])

  # Filter ด้วย accumulator
  def filter(list, pred), do: filter(list, pred, [])
  defp filter([], _pred, acc), do: Enum.reverse(acc)
  defp filter([head | tail], pred, acc) do
    if pred.(head) do
      filter(tail, pred, [head | acc])
    else
      filter(tail, pred, acc)
    end
  end

  # Flatten
  def flatten(list), do: flatten(list, [])
  defp flatten([], acc), do: Enum.reverse(acc)
  defp flatten([head | tail], acc) when is_list(head) do
    flatten(head ++ tail, acc)
  end
  defp flatten([head | tail], acc) do
    flatten(tail, [head | acc])
  end
end

iex> ListOps.sum([1, 2, 3, 4, 5])
15

iex> ListOps.reverse([1, 2, 3, 4, 5])
[5, 4, 3, 2, 1]

iex> ListOps.map([1, 2, 3], &(&1 * 2))
[2, 4, 6]

iex> ListOps.filter([1, 2, 3, 4, 5], &(rem(&1, 2) == 0))
[2, 4]

iex> ListOps.flatten([[1, [2, 3]], 4, [5, 6]])
[1, 2, 3, 4, 5, 6]
```

---

## 3. Mutual Recursion

```elixir
defmodule EvenOdd do
  def even?(0), do: true
  def even?(n) when n > 0, do: odd?(n - 1)
  def even?(n) when n < 0, do: even?(-n)

  def odd?(0), do: false
  def odd?(n) when n > 0, do: even?(n - 1)
  def odd?(n) when n < 0, do: odd?(-n)
end

iex> EvenOdd.even?(4)
true

iex> EvenOdd.odd?(3)
true
```

---

## 4. Tree Recursion

```elixir
defmodule Tree do
  # Binary tree: {:node, value, left, right} หรือ :leaf

  def insert(:leaf, value) do
    {:node, value, :leaf, :leaf}
  end

  def insert({:node, v, left, right}, value) when value < v do
    {:node, v, insert(left, value), right}
  end

  def insert({:node, v, left, right}, value) when value > v do
    {:node, v, left, insert(right, value)}
  end

  def insert({:node, v, _, _} = node, v), do: node  # duplicate

  def member?(:leaf, _), do: false
  def member?({:node, v, _, _}, v), do: true
  def member?({:node, v, left, _}, value) when value < v do
    member?(left, value)
  end
  def member?({:node, _, _, right}, value) do
    member?(right, value)
  end

  def to_sorted_list(:leaf), do: []
  def to_sorted_list({:node, v, left, right}) do
    to_sorted_list(left) ++ [v] ++ to_sorted_list(right)
  end

  def height(:leaf), do: 0
  def height({:node, _, left, right}) do
    1 + max(height(left), height(right))
  end
end

# สร้าง BST
tree = Enum.reduce([5, 3, 7, 1, 4, 6, 8], :leaf, &Tree.insert(&2, &1))

iex> Tree.member?(tree, 4)
true

iex> Tree.to_sorted_list(tree)
[1, 3, 4, 5, 6, 7, 8]

iex> Tree.height(tree)
3
```

---

## 5. Fibonacci ต่างๆ

```elixir
defmodule Fibonacci do
  # Version 1: Simple (แต่ O(2^n) - ช้ามาก!)
  def naive(0), do: 0
  def naive(1), do: 1
  def naive(n), do: naive(n-1) + naive(n-2)

  # Version 2: Tail recursive (O(n))
  def tail(n), do: tail(n, 0, 1)
  defp tail(0, a, _b), do: a
  defp tail(n, a, b), do: tail(n-1, b, a+b)

  # Version 3: Stream (lazy, infinite sequence)
  def stream do
    Stream.unfold({0, 1}, fn {a, b} -> {a, {b, a+b}} end)
  end

  # Version 4: With memoization (cache)
  def memoized(n) do
    cache = :ets.new(:fib_cache, [:set])
    result = memo_fib(n, cache)
    :ets.delete(cache)
    result
  end

  defp memo_fib(0, _cache), do: 0
  defp memo_fib(1, _cache), do: 1
  defp memo_fib(n, cache) do
    case :ets.lookup(cache, n) do
      [{^n, result}] -> result
      [] ->
        result = memo_fib(n-1, cache) + memo_fib(n-2, cache)
        :ets.insert(cache, {n, result})
        result
    end
  end
end

iex> Fibonacci.tail(10)
55

iex> Fibonacci.stream() |> Enum.take(10)
[0, 1, 1, 2, 3, 5, 8, 13, 21, 34]

iex> Fibonacci.memoized(50)
12586269025
```

---

## 6. Recursive Data Processing

```elixir
defmodule JSONProcessor do
  @doc "Deep transform keys ใน nested map/list"
  def transform_keys(map, func) when is_map(map) do
    Map.new(map, fn {k, v} ->
      {func.(k), transform_keys(v, func)}
    end)
  end

  def transform_keys(list, func) when is_list(list) do
    Enum.map(list, &transform_keys(&1, func))
  end

  def transform_keys(value, _func), do: value

  @doc "Deep merge สอง maps"
  def deep_merge(map1, map2) when is_map(map1) and is_map(map2) do
    Map.merge(map1, map2, fn _key, v1, v2 ->
      deep_merge(v1, v2)
    end)
  end

  def deep_merge(_v1, v2), do: v2

  @doc "Flatten nested map เป็น flat map ด้วย dot notation keys"
  def flatten_map(map, prefix \\ "") do
    Enum.flat_map(map, fn {k, v} ->
      full_key = if prefix == "", do: to_string(k), else: "#{prefix}.#{k}"

      if is_map(v) do
        flatten_map(v, full_key)
      else
        [{full_key, v}]
      end
    end)
    |> Map.new()
  end
end

# ตัวอย่าง
data = %{
  "firstName" => "Alice",
  "lastName" => "Smith",
  "address" => %{
    "streetAddress" => "123 Main St",
    "city" => "Bangkok"
  }
}

# Transform keys เป็น snake_case
snake_case_data = JSONProcessor.transform_keys(data, fn k ->
  k
  |> Macro.underscore()
  |> String.to_atom()
end)

# Flatten
flattened = JSONProcessor.flatten_map(data)
# %{"firstName" => "Alice", "address.city" => "Bangkok", ...}
```

---

## 7. Practical Examples

### Quick Sort

```elixir
defmodule Sort do
  def quicksort([]), do: []
  def quicksort([pivot | rest]) do
    smaller = Enum.filter(rest, &(&1 <= pivot))
    larger = Enum.filter(rest, &(&1 > pivot))
    quicksort(smaller) ++ [pivot] ++ quicksort(larger)
  end

  # Merge Sort
  def mergesort([]), do: []
  def mergesort([x]), do: [x]
  def mergesort(list) do
    mid = div(length(list), 2)
    {left, right} = Enum.split(list, mid)
    merge(mergesort(left), mergesort(right))
  end

  defp merge([], right), do: right
  defp merge(left, []), do: left
  defp merge([lh | lt], [rh | _] = right) when lh <= rh do
    [lh | merge(lt, right)]
  end
  defp merge(left, [rh | rt]) do
    [rh | merge(left, rt)]
  end
end

iex> Sort.quicksort([3, 1, 4, 1, 5, 9, 2, 6])
[1, 1, 2, 3, 4, 5, 6, 9]

iex> Sort.mergesort([3, 1, 4, 1, 5, 9, 2, 6])
[1, 1, 2, 3, 4, 5, 6, 9]
```

### Power Set

```elixir
defmodule Sets do
  def power_set([]), do: [[]]
  def power_set([head | tail]) do
    sub_sets = power_set(tail)
    sub_sets ++ Enum.map(sub_sets, fn set -> [head | set] end)
  end

  def permutations([]), do: [[]]
  def permutations(list) do
    for head <- list,
        tail <- permutations(list -- [head]) do
      [head | tail]
    end
  end
end

iex> Sets.power_set([1, 2, 3])
[[], [3], [2], [2, 3], [1], [1, 3], [1, 2], [1, 2, 3]]

iex> Sets.permutations([1, 2, 3])
[[1, 2, 3], [1, 3, 2], [2, 1, 3], [2, 3, 1], [3, 1, 2], [3, 2, 1]]
```

---

## แบบฝึกหัด

### Exercise 1: Tail Recursive Sum
แปลง function นี้ให้เป็น tail recursive:
```elixir
def sum([]), do: 0
def sum([h | t]), do: h + sum(t)
```

### Exercise 2: Flatten Deep
เขียน `deep_flatten/1` ที่ flatten list ได้ทุก level:
```elixir
deep_flatten([1, [2, [3, [4]]], 5])  # => [1, 2, 3, 4, 5]
```

### Exercise 3: Tree
เพิ่ม functions ใน Tree module:
- `delete/2` - ลบ node
- `count/1` - นับจำนวน nodes
- `map/2` - apply function กับทุก value

### Exercise 4: GCD และ LCM
เขียน recursive function สำหรับ:
- `gcd(a, b)` - Greatest Common Divisor (ใช้ Euclidean algorithm)
- `lcm(a, b)` - Least Common Multiple

---

## เฉลย Exercise 1

```elixir
def sum(list), do: sum(list, 0)
defp sum([], acc), do: acc
defp sum([h | t], acc), do: sum(t, acc + h)
```

## เฉลย Exercise 4

```elixir
defmodule NumberTheory do
  def gcd(a, 0), do: abs(a)
  def gcd(a, b), do: gcd(b, rem(a, b))

  def lcm(0, _), do: 0
  def lcm(_, 0), do: 0
  def lcm(a, b), do: div(abs(a * b), gcd(a, b))
end

iex> NumberTheory.gcd(48, 18)
6

iex> NumberTheory.lcm(4, 6)
12
```

---

## สรุป

```
Recursion ใน Elixir:

พื้นฐาน:
├── Base case (กรณีหยุด)
├── Recursive case (เรียกตัวเอง)
└── ต้องมีทั้งคู่

Tail Call Optimization:
├── Tail call = recursive call เป็น operation สุดท้าย
├── BEAM optimizes tail calls → ไม่ใช้ stack เพิ่ม
└── ใช้ accumulator pattern

Accumulator Pattern:
├── เพิ่ม parameter สำหรับ accumulate ผลลัพธ์
├── Public API ไม่เปิด accumulator
└── เหมาะกับ list operations

เมื่อใช้ recursion:
├── ข้อมูลที่มีโครงสร้าง recursive (lists, trees)
├── ปัญหาที่แบ่งเป็น sub-problems ได้
└── ใช้ Enum/Stream สำหรับ simple iterations
```

---

*ก่อนหน้า: [Part 06](part_06.md) | ต่อไป: [Part 08 - Enum และ Stream](part_08.md)*
