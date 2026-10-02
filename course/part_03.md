# Part 03: โครงสร้างข้อมูล - List, Tuple, Map, Keyword List

## เป้าหมายการเรียนรู้

หลังจากจบ Part นี้ คุณจะสามารถ:
- ใช้งาน List, Tuple, Map, Keyword List ได้
- เลือกโครงสร้างข้อมูลที่เหมาะสมกับงาน
- Manipulate ข้อมูลในแต่ละโครงสร้างได้
- เข้าใจ performance characteristics ของแต่ละโครงสร้าง

---

## 1. List

List คือ linked list ที่เก็บข้อมูลหลายชิ้น โดยสามารถมีชนิดข้อมูลต่างกันได้

### การสร้าง List

```elixir
# List ธรรมดา
iex> [1, 2, 3]
[1, 2, 3]

# List แบบผสมชนิดข้อมูล
iex> [1, "hello", :atom, true, nil]
[1, "hello", :atom, true, nil]

# List ว่าง
iex> []
[]

# List ซ้อนกัน (Nested list)
iex> [[1, 2], [3, 4], [5, 6]]
[[1, 2], [3, 4], [5, 6]]

# สร้างด้วย Range
iex> Enum.to_list(1..5)
[1, 2, 3, 4, 5]
```

### โครงสร้างภายใน

```
List [1, 2, 3] ใน memory:
┌───┬───┐   ┌───┬───┐   ┌───┬────┐
│ 1 │ • ├──>│ 2 │ • ├──>│ 3 │ [] │
└───┴───┘   └───┴───┘   └───┴────┘
head      tail of head  tail of tail

List ใน Elixir เป็น Linked List ดังนั้น:
- การเข้าถึง head: O(1)
- การเข้าถึง element ที่ n: O(n)
- การ prepend: O(1)
- การ append: O(n)
```

### Head และ Tail

```elixir
# head = element แรก
iex> hd([1, 2, 3])
1

# tail = ส่วนที่เหลือ (เป็น list)
iex> tl([1, 2, 3])
[2, 3]

# Pattern matching ที่ใช้บ่อย
iex> [head | tail] = [1, 2, 3]
[1, 2, 3]

iex> head
1

iex> tail
[2, 3]

# ใช้ _ สำหรับส่วนที่ไม่ใช้
iex> [first | _] = [1, 2, 3]
[1, 2, 3]

iex> first
1

# Destructuring หลาย element
iex> [a, b | rest] = [1, 2, 3, 4, 5]
[1, 2, 3, 4, 5]

iex> a
1

iex> b
2

iex> rest
[3, 4, 5]
```

### การ Prepend (เพิ่มที่ต้น)

```elixir
# Prepend ด้วย | operator - O(1)
iex> [0 | [1, 2, 3]]
[0, 1, 2, 3]

iex> list = [1, 2, 3]
[1, 2, 3]

iex> [0 | list]
[0, 1, 2, 3]

# list เดิมไม่เปลี่ยน
iex> list
[1, 2, 3]
```

### List Operations ที่ใช้บ่อย

```elixir
# ความยาว
iex> length([1, 2, 3, 4, 5])
5

# Concatenation
iex> [1, 2, 3] ++ [4, 5, 6]
[1, 2, 3, 4, 5, 6]

# Subtraction
iex> [1, 2, 3, 4, 5] -- [2, 4]
[1, 3, 5]

# ตรวจสอบสมาชิก
iex> 3 in [1, 2, 3]
true

iex> 6 in [1, 2, 3]
false

# Reverse
iex> Enum.reverse([1, 2, 3])
[3, 2, 1]

# Sort
iex> Enum.sort([3, 1, 4, 1, 5, 9])
[1, 1, 3, 4, 5, 9]

# Unique
iex> Enum.uniq([1, 2, 2, 3, 3, 3])
[1, 2, 3]

# Flatten
iex> List.flatten([[1, 2], [3, [4, 5]]])
[1, 2, 3, 4, 5]

# Last element
iex> List.last([1, 2, 3])
3

# Access by index
iex> Enum.at([10, 20, 30], 1)
20

iex> Enum.at([10, 20, 30], 10, :not_found)
:not_found
```

### List Comprehensions

```elixir
# สร้าง list ใหม่จาก list เดิม
iex> for x <- [1, 2, 3, 4, 5], do: x * 2
[2, 4, 6, 8, 10]

# กรองด้วย conditions
iex> for x <- 1..10, rem(x, 2) == 0, do: x
[2, 4, 6, 8, 10]

# Nested
iex> for x <- 1..3, y <- 1..3, do: {x, y}
[{1, 1}, {1, 2}, {1, 3}, {2, 1}, {2, 2}, {2, 3}, {3, 1}, {3, 2}, {3, 3}]

# เก็บผลเป็น map
iex> for {k, v} <- [a: 1, b: 2, c: 3], into: %{} do
...>   {k, v * 10}
...> end
%{a: 10, b: 20, c: 30}
```

---

## 2. Tuple

Tuple คือ collection ที่มีขนาดคงที่ ใช้เก็บข้อมูลที่มีความสัมพันธ์กัน

### การสร้าง Tuple

```elixir
# Tuple ธรรมดา
iex> {1, 2, 3}
{1, 2, 3}

# Tuple แบบผสม
iex> {"Alice", 30, :developer}
{"Alice", 30, :developer}

# Tuple ว่าง
iex> {}
{}

# Tuple เดี่ยว
iex> {42}
{42}
```

### โครงสร้างภายใน

```
Tuple {1, 2, 3} ใน memory:
┌───┬───┬───┐
│ 1 │ 2 │ 3 │  <- Contiguous memory (เหมือน Array)
└───┴───┴───┘
  0   1   2   <- Index

Tuple ดีกว่า List สำหรับ:
- Random access: O(1)
- ขนาดคงที่ (ทราบล่วงหน้า)
- Return value หลาย values จาก function
```

### การเข้าถึง Element

```elixir
# ด้วย elem/2
iex> tuple = {"Alice", 30, :developer}
{"Alice", 30, :developer}

iex> elem(tuple, 0)
"Alice"

iex> elem(tuple, 1)
30

iex> elem(tuple, 2)
:developer

# Pattern matching (วิธีที่แนะนำ)
iex> {name, age, role} = {"Alice", 30, :developer}
{"Alice", 30, :developer}

iex> name
"Alice"

iex> age
30
```

### การแก้ไข Tuple

```elixir
# put_elem สร้าง tuple ใหม่
iex> tuple = {1, 2, 3}
{1, 2, 3}

iex> put_elem(tuple, 1, 99)
{1, 99, 3}

# tuple เดิมไม่เปลี่ยน
iex> tuple
{1, 2, 3}

# ขนาดของ tuple
iex> tuple_size({1, 2, 3})
3
```

### Pattern ที่ใช้บ่อย: Tagged Tuples

```elixir
# {:ok, value} และ {:error, reason}
iex> {:ok, "result"}
{:ok, "result"}

iex> {:error, "not found"}
{:error, "not found"}

# การใช้งาน
def find_user(id) do
  case Database.get(id) do
    nil -> {:error, "User not found"}
    user -> {:ok, user}
  end
end

# ใช้ Pattern Matching กับ result
case find_user(123) do
  {:ok, user} -> IO.puts("Found: #{user.name}")
  {:error, reason} -> IO.puts("Error: #{reason}")
end
```

---

## 3. Map

Map คือ key-value store ที่ใช้บ่อยที่สุดใน Elixir

### การสร้าง Map

```elixir
# Map ด้วย atom keys
iex> %{name: "Alice", age: 30}
%{name: "Alice", age: 30}

# Map ด้วย string keys
iex> %{"name" => "Alice", "age" => 30}
%{"age" => 30, "name" => "Alice"}

# Map ด้วย mixed keys
iex> %{:name => "Alice", "email" => "alice@example.com", 1 => "one"}
%{1 => "one", :name => "Alice", "email" => "alice@example.com"}

# Map ว่าง
iex> %{}
%{}
```

### การเข้าถึงข้อมูล

```elixir
iex> user = %{name: "Alice", age: 30, email: "alice@example.com"}

# ด้วย dot notation (atom keys เท่านั้น)
iex> user.name
"Alice"

iex> user.age
30

# ถ้า key ไม่มี จะ error
iex> user.phone
** (KeyError) key :phone not found in: %{age: 30, email: "alice@example.com", name: "Alice"}

# ด้วย [] notation (ทุก key type)
iex> user[:name]
"Alice"

# ถ้า key ไม่มี จะ return nil
iex> user[:phone]
nil

# Map.get ระบุ default ได้
iex> Map.get(user, :phone, "N/A")
"N/A"

# Map.fetch ให้ tagged tuple
iex> Map.fetch(user, :name)
{:ok, "Alice"}

iex> Map.fetch(user, :phone)
:error

# Map.fetch! (raise error ถ้าไม่พบ)
iex> Map.fetch!(user, :phone)
** (KeyError) key :phone not found in: ...
```

### การแก้ไข Map

```elixir
# Map.put - เพิ่มหรืออัปเดต key
iex> user = %{name: "Alice", age: 30}
iex> Map.put(user, :email, "alice@example.com")
%{age: 30, email: "alice@example.com", name: "Alice"}

# Syntax สั้น (atom keys เท่านั้น)
iex> %{user | age: 31}
%{age: 31, name: "Alice"}

# ถ้า key ไม่มีจะ error
iex> %{user | phone: "123"}
** (KeyError) key :phone not found in: %{age: 30, name: "Alice"}

# Map.update - อัปเดตด้วย function
iex> Map.update(user, :age, 0, fn age -> age + 1 end)
%{age: 31, name: "Alice"}

# Map.update! (raise error ถ้าไม่พบ key)
iex> Map.update!(user, :age, fn age -> age + 1 end)
%{age: 31, name: "Alice"}

# Map.delete
iex> Map.delete(user, :age)
%{name: "Alice"}

# Map.drop (ลบหลาย keys)
iex> Map.drop(user, [:age, :name])
%{}
```

### Map Operations ที่ใช้บ่อย

```elixir
user = %{name: "Alice", age: 30, email: "alice@example.com"}

# ตรวจสอบ key
iex> Map.has_key?(user, :name)
true

iex> Map.has_key?(user, :phone)
false

# ดู keys ทั้งหมด
iex> Map.keys(user)
[:age, :email, :name]

# ดู values ทั้งหมด
iex> Map.values(user)
[30, "alice@example.com", "Alice"]

# Map.to_list
iex> Map.to_list(user)
[age: 30, email: "alice@example.com", name: "Alice"]

# Map.merge (merge สอง maps)
iex> Map.merge(%{a: 1, b: 2}, %{b: 3, c: 4})
%{a: 1, b: 3, c: 4}  # key ซ้ำใช้ค่าจาก map ที่สอง

# Map.merge ด้วย function
iex> Map.merge(%{a: 1, b: 2}, %{b: 3, c: 4}, fn _k, v1, v2 -> v1 + v2 end)
%{a: 1, b: 5, c: 4}

# Map.filter
iex> Map.filter(user, fn {_k, v} -> is_binary(v) end)
%{email: "alice@example.com", name: "Alice"}

# Map size
iex> map_size(user)
3
```

### Nested Maps

```elixir
user = %{
  name: "Alice",
  address: %{
    street: "123 Main St",
    city: "Bangkok",
    country: "Thailand"
  }
}

# เข้าถึง nested value
iex> user.address.city
"Bangkok"

iex> user[:address][:city]
"Bangkok"

iex> get_in(user, [:address, :city])
"Bangkok"

# อัปเดต nested value
iex> put_in(user, [:address, :city], "Chiang Mai")
%{address: %{city: "Chiang Mai", country: "Thailand", street: "123 Main St"}, name: "Alice"}

# อัปเดตด้วย function
iex> update_in(user, [:address, :city], &String.upcase/1)
%{address: %{city: "BANGKOK", country: "Thailand", street: "123 Main St"}, name: "Alice"}

# get_and_update_in
iex> get_and_update_in(user, [:address, :city], fn city ->
...>   {city, String.upcase(city)}
...> end)
{"Bangkok", %{address: %{city: "BANGKOK", ...}, name: "Alice"}}
```

### Pattern Matching กับ Map

```elixir
# Match specific keys
iex> %{name: name} = %{name: "Alice", age: 30}
%{name: "Alice", age: 30}

iex> name
"Alice"

# Map matching ไม่ต้อง match ทุก key
iex> %{name: name, age: age} = %{name: "Alice", age: 30, email: "alice@example.com"}
%{name: "Alice", age: 30, email: "alice@example.com"}

iex> name
"Alice"
```

---

## 4. Keyword List

Keyword List คือ List ของ Tuples `{atom, value}` ใช้สำหรับ options

### การสร้าง Keyword List

```elixir
# Syntax ย่อ
iex> [name: "Alice", age: 30]
[name: "Alice", age: 30]

# เทียบเท่ากับ
iex> [{:name, "Alice"}, {:age, 30}]
[{:name, "Alice"}, {:age, 30}]

# Keyword list สามารถมี key ซ้ำได้!
iex> [a: 1, a: 2, b: 3]
[a: 1, a: 2, b: 3]
```

### การเข้าถึงข้อมูล

```elixir
opts = [name: "Alice", age: 30]

# ด้วย [] notation
iex> opts[:name]
"Alice"

# Keyword.get
iex> Keyword.get(opts, :name)
"Alice"

# ถ้ามี key ซ้ำ จะได้ค่าแรก
iex> opts = [a: 1, a: 2]
iex> opts[:a]
1

iex> Keyword.get_values(opts, :a)
[1, 2]

# Keyword.fetch
iex> Keyword.fetch(opts, :name)
{:ok, "Alice"}

iex> Keyword.fetch(opts, :phone)
:error
```

### Keyword List Operations

```elixir
opts = [name: "Alice", age: 30]

# เพิ่ม
iex> Keyword.put(opts, :email, "alice@example.com")
[name: "Alice", age: 30, email: "alice@example.com"]

# ลบ
iex> Keyword.delete(opts, :age)
[name: "Alice"]

# ตรวจสอบ
iex> Keyword.has_key?(opts, :name)
true

# Merge
iex> Keyword.merge([a: 1, b: 2], [b: 3, c: 4])
[a: 1, b: 3, c: 4]

# Keys/Values
iex> Keyword.keys(opts)
[:name, :age]

iex> Keyword.values(opts)
["Alice", 30]
```

### เมื่อใช้ Keyword List

```elixir
# ใช้เป็น function options
def create_user(name, opts \\ []) do
  age = Keyword.get(opts, :age, 0)
  email = Keyword.get(opts, :email, "")
  %{name: name, age: age, email: email}
end

create_user("Alice")
create_user("Alice", age: 30)
create_user("Alice", age: 30, email: "alice@example.com")

# ใช้เป็น options สำหรับ library functions
Enum.sort([3, 1, 2], order: :desc)

String.split("hello world", " ", trim: true)
```

---

## 5. MapSet

MapSet คือ Set ที่เก็บค่าไม่ซ้ำ

```elixir
# สร้าง MapSet
iex> MapSet.new([1, 2, 3, 2, 1])
MapSet.new([1, 2, 3])

iex> MapSet.new(["a", "b", "c"])
MapSet.new(["a", "b", "c"])

# เพิ่มสมาชิก
iex> set = MapSet.new([1, 2, 3])
iex> MapSet.put(set, 4)
MapSet.new([1, 2, 3, 4])

iex> MapSet.put(set, 2)  # ไม่เพิ่มถ้ามีอยู่แล้ว
MapSet.new([1, 2, 3])

# ตรวจสอบสมาชิก
iex> MapSet.member?(set, 2)
true

iex> MapSet.member?(set, 5)
false

# Set operations
iex> a = MapSet.new([1, 2, 3, 4])
iex> b = MapSet.new([3, 4, 5, 6])

# Union
iex> MapSet.union(a, b)
MapSet.new([1, 2, 3, 4, 5, 6])

# Intersection
iex> MapSet.intersection(a, b)
MapSet.new([3, 4])

# Difference
iex> MapSet.difference(a, b)
MapSet.new([1, 2])

# ขนาด
iex> MapSet.size(set)
3

# แปลงเป็น list
iex> MapSet.to_list(set)
[1, 2, 3]
```

---

## 6. เปรียบเทียบโครงสร้างข้อมูล

```
┌──────────────────┬─────────────┬────────────────────────────────────┐
│  Structure       │  เมื่อใช้   │  คุณสมบัติ                         │
├──────────────────┼─────────────┼────────────────────────────────────┤
│  List [1,2,3]    │  Collection │  - Linked list O(n) access         │
│                  │  ของข้อมูล  │  - O(1) prepend                    │
│                  │  ที่เกี่ยว  │  - สามารถเป็น recursive pattern    │
│                  │  กัน        │                                    │
├──────────────────┼─────────────┼────────────────────────────────────┤
│  Tuple {1,2,3}   │  Fixed-size │  - O(1) access by index            │
│                  │  data       │  - ใช้เป็น return value            │
│                  │  ขนาดทราบ  │  - Pattern matching ดีมาก          │
│                  │  ล่วงหน้า   │                                    │
├──────────────────┼─────────────┼────────────────────────────────────┤
│  Map %{}         │  Key-value  │  - O(log n) access                 │
│                  │  store      │  - Keys ไม่ซ้ำ                     │
│                  │             │  - ใช้บ่อยที่สุด                   │
├──────────────────┼─────────────┼────────────────────────────────────┤
│  Keyword List    │  Options    │  - Ordered                         │
│  [k: v]          │  สำหรับ     │  - Keys ซ้ำได้                     │
│                  │  functions  │  - ใช้เป็น options เท่านั้น        │
├──────────────────┼─────────────┼────────────────────────────────────┤
│  MapSet          │  Unique     │  - No duplicates                   │
│                  │  values     │  - Set operations                  │
│                  │             │  - O(log n) access                 │
└──────────────────┴─────────────┴────────────────────────────────────┘
```

---

## 7. ตัวอย่างจริง: Contact Book

```elixir
defmodule ContactBook do
  @moduledoc """
  สมุดโทรศัพท์อย่างง่าย
  """

  # สร้าง contact ใหม่
  def new_contact(name, phone, opts \\ []) do
    %{
      name: name,
      phone: phone,
      email: Keyword.get(opts, :email, ""),
      tags: Keyword.get(opts, :tags, []),
      created_at: DateTime.utc_now()
    }
  end

  # สร้าง contact book ว่าง
  def new_book do
    %{}
  end

  # เพิ่ม contact
  def add_contact(book, contact) do
    Map.put(book, contact.name, contact)
  end

  # ค้นหา contact
  def find_contact(book, name) do
    case Map.fetch(book, name) do
      {:ok, contact} -> {:ok, contact}
      :error -> {:error, "Contact '#{name}' not found"}
    end
  end

  # อัปเดต contact
  def update_phone(book, name, new_phone) do
    case Map.has_key?(book, name) do
      true ->
        updated_book = update_in(book, [name, :phone], fn _ -> new_phone end)
        {:ok, updated_book}
      false ->
        {:error, "Contact '#{name}' not found"}
    end
  end

  # ลบ contact
  def remove_contact(book, name) do
    Map.delete(book, name)
  end

  # ค้นหาด้วย tag
  def find_by_tag(book, tag) do
    book
    |> Map.values()
    |> Enum.filter(fn contact -> tag in contact.tags end)
  end

  # แสดงรายชื่อทั้งหมด
  def list_contacts(book) do
    book
    |> Map.values()
    |> Enum.sort_by(& &1.name)
  end

  # สถิติ
  def stats(book) do
    contacts = Map.values(book)
    total_tags = contacts
      |> Enum.flat_map(& &1.tags)
      |> MapSet.new()
      |> MapSet.size()

    %{
      total_contacts: length(contacts),
      unique_tags: total_tags,
      contacts_with_email: Enum.count(contacts, fn c -> c.email != "" end)
    }
  end
end
```

การใช้งาน:

```elixir
# สร้าง contact book
book = ContactBook.new_book()

# เพิ่ม contacts
alice = ContactBook.new_contact("Alice", "081-234-5678",
  email: "alice@example.com",
  tags: [:work, :friend]
)

bob = ContactBook.new_contact("Bob", "082-345-6789",
  email: "bob@example.com",
  tags: [:work]
)

charlie = ContactBook.new_contact("Charlie", "083-456-7890",
  tags: [:family]
)

book = book
  |> ContactBook.add_contact(alice)
  |> ContactBook.add_contact(bob)
  |> ContactBook.add_contact(charlie)

# ค้นหา
{:ok, contact} = ContactBook.find_contact(book, "Alice")
IO.puts("Found: #{contact.name}, Phone: #{contact.phone}")

# อัปเดต
{:ok, book} = ContactBook.update_phone(book, "Alice", "081-999-9999")

# ค้นหาด้วย tag
work_contacts = ContactBook.find_by_tag(book, :work)
IO.puts("Work contacts: #{length(work_contacts)}")

# สถิติ
stats = ContactBook.stats(book)
IO.puts("Total contacts: #{stats.total_contacts}")

# รายชื่อทั้งหมด
ContactBook.list_contacts(book)
|> Enum.each(fn c ->
  IO.puts("#{c.name}: #{c.phone}")
end)
```

---

## 8. ตัวอย่างจริง: Shopping Cart

```elixir
defmodule ShoppingCart do
  # สร้าง cart ว่าง
  def new do
    %{
      items: [],
      discount_codes: MapSet.new()
    }
  end

  # เพิ่มสินค้า
  def add_item(cart, product_id, name, price, quantity \\ 1) do
    item = %{
      product_id: product_id,
      name: name,
      price: price,
      quantity: quantity
    }

    # ตรวจสอบว่ามีสินค้านี้แล้วหรือยัง
    case Enum.find_index(cart.items, fn i -> i.product_id == product_id end) do
      nil ->
        %{cart | items: [item | cart.items]}
      index ->
        updated_items = List.update_at(cart.items, index, fn existing ->
          %{existing | quantity: existing.quantity + quantity}
        end)
        %{cart | items: updated_items}
    end
  end

  # ลบสินค้า
  def remove_item(cart, product_id) do
    %{cart | items: Enum.reject(cart.items, fn i -> i.product_id == product_id end)}
  end

  # อัปเดตจำนวน
  def update_quantity(cart, product_id, quantity) when quantity > 0 do
    case Enum.find_index(cart.items, fn i -> i.product_id == product_id end) do
      nil -> {:error, "Product not in cart"}
      index ->
        updated_items = List.update_at(cart.items, index, fn item ->
          %{item | quantity: quantity}
        end)
        {:ok, %{cart | items: updated_items}}
    end
  end

  # เพิ่ม discount code
  def add_discount(cart, code) do
    %{cart | discount_codes: MapSet.put(cart.discount_codes, code)}
  end

  # คำนวณราคา
  def subtotal(cart) do
    cart.items
    |> Enum.reduce(0, fn item, acc ->
      acc + (item.price * item.quantity)
    end)
  end

  def total(cart, opts \\ []) do
    tax_rate = Keyword.get(opts, :tax_rate, 0.07)
    subtotal = subtotal(cart)
    discount = calculate_discount(cart, subtotal)
    tax = (subtotal - discount) * tax_rate

    %{
      subtotal: subtotal,
      discount: discount,
      tax: Float.round(tax, 2),
      total: Float.round(subtotal - discount + tax, 2)
    }
  end

  defp calculate_discount(cart, subtotal) do
    base_discount = if MapSet.member?(cart.discount_codes, "SAVE10"), do: subtotal * 0.1, else: 0
    if MapSet.size(cart.discount_codes) > 1, do: base_discount + 50, else: base_discount
  end

  # แสดงรายการสินค้า
  def item_count(cart) do
    cart.items
    |> Enum.reduce(0, fn item, acc -> acc + item.quantity end)
  end

  def display(cart) do
    IO.puts("\n=== Shopping Cart ===")
    Enum.each(cart.items, fn item ->
      IO.puts("#{item.name} x#{item.quantity} = ฿#{item.price * item.quantity}")
    end)

    totals = total(cart)
    IO.puts("\nSubtotal: ฿#{totals.subtotal}")
    IO.puts("Discount: -฿#{totals.discount}")
    IO.puts("Tax (7%): +฿#{totals.tax}")
    IO.puts("Total: ฿#{totals.total}")
    IO.puts("===================\n")
  end
end
```

การใช้งาน:

```elixir
cart = ShoppingCart.new()
  |> ShoppingCart.add_item("P001", "Elixir Book", 499, 2)
  |> ShoppingCart.add_item("P002", "Phoenix Book", 599, 1)
  |> ShoppingCart.add_item("P003", "OTP Guide", 399, 1)
  |> ShoppingCart.add_discount("SAVE10")

ShoppingCart.display(cart)
# === Shopping Cart ===
# Elixir Book x2 = ฿998
# Phoenix Book x1 = ฿599
# OTP Guide x1 = ฿399
#
# Subtotal: ฿1996
# Discount: -฿199.6
# Tax (7%): +฿125.77
# Total: ฿1922.17
# ===================

IO.puts("Items in cart: #{ShoppingCart.item_count(cart)}")
```

---

## 9. Access Module

Access module ให้ใช้ `[]` syntax กับ data structures หลายชนิด

```elixir
# กับ Map
iex> user = %{name: "Alice", address: %{city: "Bangkok"}}
iex> get_in(user, [:address, :city])
"Bangkok"

# กับ List
iex> users = [%{name: "Alice"}, %{name: "Bob"}]
iex> get_in(users, [Access.at(0), :name])
"Alice"

# กับ filter
iex> users = [%{name: "Alice", active: true}, %{name: "Bob", active: false}]
iex> get_in(users, [Access.filter(& &1.active), :name])
["Alice"]

# update_in กับ complex structures
data = %{users: [%{name: "Alice", score: 0}, %{name: "Bob", score: 0}]}
updated = update_in(data, [:users, Access.all(), :score], &(&1 + 10))
# %{users: [%{name: "Alice", score: 10}, %{name: "Bob", score: 10}]}
```

---

## แบบฝึกหัด

### Exercise 1: List Operations
```elixir
numbers = [3, 1, 4, 1, 5, 9, 2, 6, 5, 3, 5]

# 1. หา unique numbers
# 2. Sort จากมากไปน้อย
# 3. กรองเฉพาะ numbers > 3
# 4. คำนวณ sum ของ numbers ทั้งหมด
# 5. หา max และ min
```

### Exercise 2: Map Operations
```elixir
students = [
  %{name: "Alice", grade: 85, subject: "Math"},
  %{name: "Bob", grade: 92, subject: "Science"},
  %{name: "Alice", grade: 78, subject: "English"},
  %{name: "Charlie", grade: 88, subject: "Math"},
]

# 1. หา student ที่มี grade สูงสุด
# 2. Group ตาม name
# 3. คำนวณ average grade ของแต่ละ student
# 4. หา students ที่มี grade >= 85
```

### Exercise 3: Keyword List
เขียน function `parse_query_string` ที่:
- รับ string เช่น `"name=Alice&age=30&active=true"`
- Return เป็น Keyword list เช่น `[name: "Alice", age: "30", active: "true"]`

### Exercise 4: สร้าง Inventory System
สร้าง module `Inventory` ที่มี:
- `new/0` - สร้าง inventory ว่าง
- `add_product/4` - เพิ่มสินค้า (id, name, price, quantity)
- `update_stock/3` - อัปเดต quantity
- `get_product/2` - ดึงข้อมูลสินค้า
- `low_stock/2` - หาสินค้าที่ stock ต่ำกว่า threshold
- `total_value/1` - คำนวณมูลค่ารวม

---

## เฉลย Exercise 1

```elixir
numbers = [3, 1, 4, 1, 5, 9, 2, 6, 5, 3, 5]

unique = Enum.uniq(numbers)            # [3, 1, 4, 5, 9, 2, 6]
sorted_desc = Enum.sort(numbers, :desc) # [9, 6, 5, 5, 5, 4, 3, 3, 2, 1, 1]
filtered = Enum.filter(numbers, &(&1 > 3))  # [4, 5, 9, 6, 5, 5]
total = Enum.sum(numbers)              # 44
{min_val, max_val} = {Enum.min(numbers), Enum.max(numbers)}  # {1, 9}
```

## เฉลย Exercise 3

```elixir
defmodule QueryString do
  def parse(query_string) do
    query_string
    |> String.split("&")
    |> Enum.map(fn pair ->
      [key, value] = String.split(pair, "=", parts: 2)
      {String.to_atom(key), value}
    end)
  end
end

iex> QueryString.parse("name=Alice&age=30&active=true")
[name: "Alice", age: "30", active: "true"]
```

---

## สรุป

```
โครงสร้างข้อมูลใน Elixir:

List []:
├── Linked list
├── Pattern matching [head | tail]
├── เหมาะกับ collection ที่ต้อง iterate
└── O(1) prepend, O(n) access

Tuple {}:
├── Fixed-size array
├── Pattern matching {a, b, c}
├── ใช้เป็น {:ok, value} / {:error, reason}
└── O(1) access by index

Map %{}:
├── Key-value store
├── Keys ไม่ซ้ำ
├── ใช้มากที่สุด
└── O(log n) access

Keyword List []:
├── List ของ {atom, value}
├── Keys ซ้ำได้
├── ใช้เป็น function options
└── Ordered

MapSet:
├── ไม่มี duplicates
├── Set operations (union, intersection, difference)
└── O(log n) operations
```

---

*ก่อนหน้า: [Part 02](part_02.md) | ต่อไป: [Part 04 - Pattern Matching](part_04.md)*
