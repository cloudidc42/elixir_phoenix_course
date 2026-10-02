# Part 10: Strings, Binaries, และ Charlists

## เป้าหมายการเรียนรู้

หลังจากจบ Part นี้ คุณจะสามารถ:
- เข้าใจว่า String ใน Elixir คืออะไร
- จัดการ Strings อย่างมีประสิทธิภาพ
- ใช้ Regex ได้
- เข้าใจ Binary และ Bitstring

---

## 1. String คืออะไรใน Elixir?

```elixir
# String ใน Elixir = UTF-8 encoded binary
iex> "hello"
"hello"

iex> is_binary("hello")
true

iex> byte_size("hello")
5

iex> String.length("hello")
5

# Unicode characters
iex> "สวัสดี"
"สวัสดี"

iex> byte_size("สวัสดี")
21  # UTF-8: ภาษาไทย 3 bytes per char

iex> String.length("สวัสดี")
6   # 6 graphemes

# Emoji
iex> byte_size("🎉")
4

iex> String.length("🎉")
1
```

---

## 2. String Operations

### Basic Operations

```elixir
# Length
iex> String.length("hello world")
11

# Uppercase/Lowercase
iex> String.upcase("hello")
"HELLO"

iex> String.downcase("HELLO")
"hello"

iex> String.capitalize("hello world")
"Hello world"

# Trim
iex> String.trim("  hello  ")
"hello"

iex> String.trim_leading("  hello  ")
"hello  "

iex> String.trim_trailing("  hello  ")
"  hello"

iex> String.trim("xxhelloxx", "x")
"hello"

# Reverse
iex> String.reverse("hello")
"olleh"

# Repeat
iex> String.duplicate("ha", 3)
"hahaha"

# Pad
iex> String.pad_leading("42", 5)
"   42"

iex> String.pad_leading("42", 5, "0")
"00042"

iex> String.pad_trailing("hello", 10, ".")
"hello....."
```

### Searching and Checking

```elixir
str = "Hello, World!"

# Contains
iex> String.contains?(str, "World")
true

iex> String.contains?(str, ["foo", "World"])
true  # any match

# Starts/Ends with
iex> String.starts_with?(str, "Hello")
true

iex> String.ends_with?(str, "!")
true

# Match regex
iex> String.match?(str, ~r/\d+/)
false

# Find index
iex> String.length(str)
13

# Count occurrences
iex> String.split(str, "l") |> length() |> Kernel.-(1)
3  # จำนวน "l" ใน string
```

### Splitting and Joining

```elixir
# Split
iex> String.split("hello world foo", " ")
["hello", "world", "foo"]

iex> String.split("a,b,,c", ",", trim: true)
["a", "b", "c"]

iex> String.split("hello", "", trim: true)
["h", "e", "l", "l", "o"]

iex> String.split("abc", ~r/b/)
["a", "c"]

# Split into max parts
iex> String.split("a.b.c.d", ".", parts: 2)
["a", "b.c.d"]

# Join
iex> Enum.join(["hello", "world"], " ")
"hello world"

iex> Enum.join([1, 2, 3], ", ")
"1, 2, 3"
```

### Extracting

```elixir
str = "Hello, World!"

# Slice
iex> String.slice(str, 0, 5)
"Hello"

iex> String.slice(str, 7, 5)
"World"

iex> String.slice(str, -6, 6)
"orld!"

# At (character at position)
iex> String.at(str, 0)
"H"

iex> String.at(str, -1)
"!"

# First/Last
iex> String.first(str)
"H"

iex> String.last(str)
"!"

# Graphemes (unicode safe)
iex> String.graphemes("abc")
["a", "b", "c"]

iex> String.graphemes("สวัสดี")
["ส", "ว", "ั", "ส", "ด", "ี"]
```

### Replacing

```elixir
str = "Hello, World! Hello!"

# Replace first
iex> String.replace(str, "Hello", "Hi", global: false)
"Hi, World! Hello!"

# Replace all
iex> String.replace(str, "Hello", "Hi")
"Hi, World! Hi!"

# Replace with regex
iex> String.replace("2024-01-15", ~r/(\d{4})-(\d{2})-(\d{2})/, "\\3/\\2/\\1")
"15/01/2024"

# Replace using function
iex> String.replace("hello world", ~r/\w+/, fn word ->
...>   String.capitalize(word)
...> end)
"Hello World"
```

---

## 3. String Interpolation

```elixir
name = "Alice"
age = 30

# Basic interpolation
iex> "Hello, #{name}!"
"Hello, Alice!"

# Expression interpolation
iex> "In 10 years, #{name} will be #{age + 10}"
"In 10 years, Alice will be 40"

# Any expression
iex> "Result: #{if true, do: "yes", else: "no"}"
"Result: yes"

# Escaped hash (no interpolation)
iex> "Price: \#{price}"
"Price: \#{price}"

# Heredoc
iex> """
...> Hello, #{name}!
...> You are #{age} years old.
...> """
"Hello, Alice!\nYou are 30 years old.\n"
```

---

## 4. Regex

```elixir
# สร้าง Regex
regex = ~r/hello/i  # case insensitive

# Match
iex> Regex.match?(~r/\d+/, "abc123")
true

iex> Regex.match?(~r/^\d+$/, "abc123")
false

# Named captures
iex> Regex.named_captures(~r/(?<year>\d{4})-(?<month>\d{2})-(?<day>\d{2})/, "2024-01-15")
%{"day" => "15", "month" => "01", "year" => "2024"}

# Scan (find all matches)
iex> Regex.scan(~r/\d+/, "abc 123 def 456")
[["123"], ["456"]]

# Named scan
iex> Regex.scan(~r/(?<word>\w+)/, "hello world")
[["hello", "hello"], ["world", "world"]]

# Run (first match)
iex> Regex.run(~r/(\d+)-(\d+)/, "2024-01")
["2024-01", "2024", "01"]

# Replace
iex> Regex.replace(~r/\d+/, "abc123def456", fn match -> "[#{match}]" end)
"abc[123]def[456]"

# Split
iex> Regex.split(~r/\s+/, "hello   world  foo")
["hello", "world", "foo"]
```

---

## 5. Binary

```elixir
# Binary คือ sequence of bytes
iex> <<72, 101, 108, 108, 111>>
"Hello"

iex> <<0, 1, 2, 3>>
<<0, 1, 2, 3>>

# String เป็น binary
iex> "hello" == <<104, 101, 108, 108, 111>>
true

# Pattern matching บน binary
iex> <<first, rest::binary>> = "Hello"
"Hello"

iex> first
72  # 'H'

iex> rest
"ello"

# ดึง bytes ที่ต้องการ
iex> <<a, b, c>> = <<1, 2, 3>>
<<1, 2, 3>>

iex> a
1

# Bitstrings
iex> <<1::1, 0::1, 1::1>>  # 3 bits
<<5::size(3)>>

# Binary matching
def parse_binary(<<length::8, data::binary-size(length), rest::binary>>) do
  {data, rest}
end

# Match specific bytes
def is_png?(<<0x89, "PNG", _::binary>>), do: true
def is_png?(_), do: false
```

---

## 6. Charlists

```elixir
# Charlist = list of Unicode codepoints
iex> 'hello'
'hello'

iex> [104, 101, 108, 108, 111]
'hello'

iex> is_list('hello')
true

# เปรียบเทียบ
iex> 'hello' == "hello"
false  # ต่างชนิด

# ใช้ใน Erlang interop
iex> :io.format("Hello ~p~n", ["world"])
Hello "world"
:ok

iex> :io.format("Hello ~s~n", ['world'])
Hello world
:ok

# Convert
iex> to_charlist("hello")
'hello'

iex> to_string('hello')
"hello"

iex> List.to_string([72, 101, 108, 108, 111])
"Hello"
```

---

## 7. ตัวอย่างจริง: Template Engine

```elixir
defmodule SimpleTemplate do
  def render(template, vars) do
    Regex.replace(~r/\{\{(\w+)\}\}/, template, fn _, key ->
      case Map.fetch(vars, key) do
        {:ok, value} -> to_string(value)
        :error -> "{{#{key}}}"  # ถ้าไม่พบ ปล่อยไว้
      end
    end)
  end

  def render_file(path, vars) do
    path
    |> File.read!()
    |> render(vars)
  end
end

template = "Hello, {{name}}! You have {{count}} messages."
vars = %{"name" => "Alice", "count" => 5}

iex> SimpleTemplate.render(template, vars)
"Hello, Alice! You have 5 messages."
```

---

## 8. ตัวอย่างจริง: CSV Parser

```elixir
defmodule CSVParser do
  def parse(content, opts \\ []) do
    separator = Keyword.get(opts, :separator, ",")
    has_header = Keyword.get(opts, :header, true)

    lines =
      content
      |> String.split("\n", trim: true)
      |> Enum.map(&String.trim/1)
      |> Enum.reject(&(&1 == ""))

    if has_header and length(lines) > 0 do
      [header_line | data_lines] = lines
      headers = parse_line(header_line, separator)

      Enum.map(data_lines, fn line ->
        values = parse_line(line, separator)
        Enum.zip(headers, values) |> Map.new()
      end)
    else
      Enum.map(lines, &parse_line(&1, separator))
    end
  end

  defp parse_line(line, separator) do
    # Handle quoted fields
    line
    |> split_respecting_quotes(separator)
    |> Enum.map(&clean_field/1)
  end

  defp split_respecting_quotes(line, separator) do
    # Simple implementation
    String.split(line, separator)
  end

  defp clean_field(field) do
    field = String.trim(field)
    if String.starts_with?(field, "\"") and String.ends_with?(field, "\"") do
      field |> String.slice(1..-2//1) |> String.replace("\"\"", "\"")
    else
      field
    end
  end

  def to_csv(data, headers \\ nil) do
    headers = headers || (data |> List.first() |> Map.keys() |> Enum.map(&to_string/1))

    rows = Enum.map(data, fn row ->
      headers
      |> Enum.map(fn h -> escape_csv_field(to_string(row[h] || row[String.to_atom(h)] || "")) end)
      |> Enum.join(",")
    end)

    ([Enum.join(headers, ",")] ++ rows) |> Enum.join("\n")
  end

  defp escape_csv_field(field) do
    if String.contains?(field, [",", "\"", "\n"]) do
      "\"#{String.replace(field, "\"", "\"\"")}\""
    else
      field
    end
  end
end

# ทดสอบ
csv = """
name,age,email
Alice,30,alice@example.com
Bob,25,bob@example.com
Charlie,35,charlie@example.com
"""

data = CSVParser.parse(csv)
# [%{"age" => "30", "email" => "alice@example.com", "name" => "Alice"}, ...]

# กลับเป็น CSV
CSVParser.to_csv(data)
```

---

## แบบฝึกหัด

### Exercise 1: String Manipulation
1. เขียน `slugify/1` ที่แปลง "Hello World! 123" → "hello-world-123"
2. เขียน `word_wrap/2` ที่ wrap text ที่ความยาวที่กำหนด

### Exercise 2: Regex
1. Extract ทุก email จาก text
2. Validate Thai phone number (0XX-XXX-XXXX)
3. Extract ทุก URL จาก HTML

### Exercise 3: Binary
1. Parse binary protocol: `<<type::8, length::16, data::binary-size(length)>>`
2. Encode/decode simple binary format

---

## เฉลย Exercise 1

```elixir
defmodule StringUtils do
  def slugify(text) do
    text
    |> String.downcase()
    |> String.replace(~r/[^\w\s-]/, "")
    |> String.replace(~r/[\s_]+/, "-")
    |> String.trim("-")
  end

  def word_wrap(text, width) do
    text
    |> String.split(" ")
    |> Enum.reduce({[], []}, fn word, {lines, current} ->
      current_length = Enum.join(current, " ") |> String.length()
      new_length = current_length + String.length(word) + if(current == [], do: 0, else: 1)

      if new_length > width and current != [] do
        {[Enum.join(current, " ") | lines], [word]}
      else
        {lines, current ++ [word]}
      end
    end)
    |> then(fn {lines, last} ->
      all = if last != [], do: [Enum.join(last, " ") | lines], else: lines
      all |> Enum.reverse() |> Enum.join("\n")
    end)
  end
end
```

---

## สรุป

```
Strings ใน Elixir:
├── UTF-8 encoded binary
├── String module: upcase, split, replace, ...
├── Regex: ~r/pattern/flags
└── Immutable

Binary:
├── << >> notation
├── Pattern matching บน bytes
├── Bitstring operations
└── Erlang interop

Charlists:
├── ' ' single quotes
├── List of codepoints
└── ใช้สำหรับ Erlang interop
```

---

*ก่อนหน้า: [Part 09](part_09.md) | ต่อไป: [Part 11 - Structs และ Protocols](part_11.md)*
