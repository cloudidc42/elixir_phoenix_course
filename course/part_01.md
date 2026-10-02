# Part 01: บทนำและการติดตั้ง Elixir

## เป้าหมายการเรียนรู้

หลังจากจบ Part นี้ คุณจะสามารถ:
- เข้าใจว่า Elixir คืออะไรและทำไมต้องใช้
- ติดตั้ง Elixir และ Erlang บนเครื่องของคุณ
- ใช้ IEx (Interactive Elixir Shell) ได้
- สร้างโปรเจ็กต์ด้วย Mix ได้

---

## 1. Elixir คืออะไร?

Elixir เป็นภาษาโปรแกรมที่สร้างขึ้นบน Erlang VM (BEAM) โดย **José Valim** ในปี 2011

### คุณสมบัติเด่น

```
┌─────────────────────────────────────────────────────────┐
│                    ELIXIR FEATURES                       │
├─────────────────────────────────────────────────────────┤
│  Functional     │  โค้ดเป็น Functions บริสุทธิ์          │
│  Concurrent     │  รองรับล้าน Processes พร้อมกัน         │
│  Distributed    │  กระจายงานข้ามหลายเครื่อง              │
│  Fault-tolerant │  ระบบไม่ล้มแม้มี Error                │
│  Hot-reloading  │  อัปเดตโค้ดโดยไม่ต้องหยุดระบบ          │
└─────────────────────────────────────────────────────────┘
```

### ทำไมต้องเรียน Elixir?

1. **ประสิทธิภาพสูง** - WhatsApp รองรับ 2 พันล้านผู้ใช้ด้วยทีมงานเล็ก
2. **Real-time Applications** - เหมาะสำหรับ Chat, Live Dashboard, IoT
3. **ความเสถียร** - Erlang VM ออกแบบมาเพื่อ 99.9999% uptime
4. **Phoenix Framework** - Web framework ที่เร็วกว่า Rails 10 เท่า
5. **Developer Experience** - ไวยากรณ์สวยงาม อ่านง่าย

### บริษัทที่ใช้ Elixir

- **Discord** - รองรับ 5 ล้าน concurrent users
- **Pinterest** - จัดการ notifications
- **Moz** - SEO Platform
- **Bleacher Report** - Sports media
- **Toyota Connected** - Connected car platform
- **Pepsi** - Global CMS

---

## 2. Erlang VM (BEAM)

Elixir ทำงานบน Erlang VM หรือ BEAM (Bogdan's Erlang Abstract Machine)

```
┌─────────────────────────────────────┐
│         BEAM Architecture           │
├─────────────────────────────────────┤
│  ┌──────────┐  ┌──────────────────┐ │
│  │  Elixir  │  │     Erlang       │ │
│  └────┬─────┘  └────────┬─────────┘ │
│       │                 │           │
│  ┌────▼─────────────────▼─────────┐ │
│  │         BEAM (Erlang VM)        │ │
│  │  ┌─────────────────────────┐   │ │
│  │  │   Scheduler (per CPU)   │   │ │
│  │  ├─────────────────────────┤   │ │
│  │  │   GC per Process        │   │ │
│  │  ├─────────────────────────┤   │ │
│  │  │   Message Passing       │   │ │
│  │  └─────────────────────────┘   │ │
│  └─────────────────────────────────┘ │
└─────────────────────────────────────┘
```

### กระบวนการ (Processes) ใน BEAM

- แต่ละ Process แยกอิสระจากกัน (Isolated)
- Communication ผ่าน Message Passing เท่านั้น
- Process เบาและเร็ว (spawn ได้ภายใน microseconds)
- GC ทำงานแยกต่างหากในแต่ละ Process

---

## 3. การติดตั้ง Elixir

### บน macOS

**วิธีที่ 1: ใช้ Homebrew (แนะนำ)**

```bash
# ติดตั้ง Homebrew ก่อน (ถ้ายังไม่มี)
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# ติดตั้ง Elixir (จะติดตั้ง Erlang ด้วยอัตโนมัติ)
brew install elixir

# ตรวจสอบการติดตั้ง
elixir --version
erl -version
```

**วิธีที่ 2: ใช้ asdf (สำหรับจัดการหลาย version)**

```bash
# ติดตั้ง asdf
git clone https://github.com/asdf-vm/asdf.git ~/.asdf --branch v0.14.0

# เพิ่มใน ~/.zshrc หรือ ~/.bashrc
echo '. "$HOME/.asdf/asdf.sh"' >> ~/.zshrc
source ~/.zshrc

# เพิ่ม plugin
asdf plugin add erlang
asdf plugin add elixir

# ติดตั้ง Erlang ก่อน
asdf install erlang 26.2.1
asdf global erlang 26.2.1

# ติดตั้ง Elixir
asdf install elixir 1.16.0
asdf global elixir 1.16.0
```

### บน Ubuntu/Debian

```bash
# วิธีที่ 1: ใช้ apt
sudo apt update
sudo apt install elixir erlang

# วิธีที่ 2: ใช้ Erlang Solutions repository (แนะนำ - ได้ version ล่าสุด)
wget https://packages.erlang-solutions.com/erlang-solutions_2.0_all.deb
sudo dpkg -i erlang-solutions_2.0_all.deb
sudo apt update
sudo apt install esl-erlang elixir

# ตรวจสอบ
elixir --version
```

### บน Windows

**วิธีที่ 1: ใช้ Windows Package Manager (winget)**

```powershell
# เปิด PowerShell ในฐานะ Administrator
winget install Erlang.OTP
winget install Elixir
```

**วิธีที่ 2: ดาวน์โหลด Installer**

1. ไปที่ https://elixir-lang.org/install.html
2. ดาวน์โหลด Windows installer
3. ติดตั้งตามขั้นตอน

**วิธีที่ 3: ใช้ WSL2 (Windows Subsystem for Linux)**

```bash
# ใน WSL2 terminal (Ubuntu)
sudo apt update
sudo apt install elixir erlang
```

### ตรวจสอบการติดตั้ง

```bash
# ตรวจสอบ Elixir version
elixir --version
# Output: Elixir 1.16.0 (compiled with Erlang/OTP 26)

# ตรวจสอบ Erlang version
erl -version
# Output: Erlang (SMP,ASYNC_THREADS) (BEAM) emulator version 14.2.1

# ตรวจสอบ Mix version
mix --version
# Output: Mix 1.16.0 (Elixir 1.16.0)

# ตรวจสอบ IEx
iex --version
# Output: IEx 1.16.0 (Elixir 1.16.0)
```

---

## 4. IEx: Interactive Elixir Shell

IEx คือ REPL (Read-Eval-Print Loop) ของ Elixir ใช้สำหรับทดสอบโค้ดแบบ interactive

### เริ่มต้น IEx

```bash
iex
```

คุณจะเห็น:

```
Erlang/OTP 26 [erts-14.2.1] [source] [64-bit] [smp:8:8] [ds:8:8:10] [async-threads:1]

Interactive Elixir (1.16.0) - press Ctrl+C to exit (type h() ENTER for help)
iex(1)>
```

### คำสั่งพื้นฐานใน IEx

```elixir
# แสดง help
iex(1)> h()

# ดู documentation ของ function
iex(1)> h String.upcase

# ดู documentation ของ module
iex(1)> h Enum

# ดูข้อมูลตัวแปร
iex(1)> i "hello"

# ดูประเภทของข้อมูล
iex(1)> is_integer(42)
true

iex(1)> is_string("hello")
** (UndefinedFunctionError) function :is_string/1 is undefined

# ถูกต้องคือ is_binary
iex(1)> is_binary("hello")
true
```

### การใช้งาน IEx เบื้องต้น

```elixir
# คำนวณง่ายๆ
iex(1)> 2 + 2
4

iex(2)> 10 / 3
3.3333333333333335

iex(3)> div(10, 3)
3

iex(4)> rem(10, 3)
1

# String operations
iex(5)> "Hello, " <> "World!"
"Hello, World!"

iex(6)> String.upcase("hello")
"HELLO"

iex(7)> String.length("hello")
5

# List operations
iex(8)> [1, 2, 3] ++ [4, 5, 6]
[1, 2, 3, 4, 5, 6]

iex(9)> length([1, 2, 3])
3

iex(10)> hd([1, 2, 3])
1

iex(11)> tl([1, 2, 3])
[2, 3]
```

### คำสั่ง IEx ที่มีประโยชน์

```elixir
# เคลียร์ประวัติ
iex(1)> IEx.Helpers.clear()

# Reload module
iex(1)> r MyModule

# Compile file
iex(1)> c "path/to/file.ex"

# ดู loaded modules
iex(1)> :code.all_loaded()

# ออกจาก IEx
# กด Ctrl+C แล้วกด a (abort)
# หรือ
iex(1)> System.halt()
```

### IEx Configuration

สร้างไฟล์ `.iex.exs` ใน home directory หรือใน project directory:

```elixir
# ~/.iex.exs
# Aliases ที่ใช้บ่อย
alias MyApp.Repo
alias MyApp.User

# Helper functions
defmodule IExHelpers do
  def reload! do
    Mix.Tasks.Compile.run([])
    :ok
  end
end

# แสดง message ตอนเริ่ม
IO.puts("Welcome to IEx! Type h() for help.")
```

---

## 5. Mix: Build Tool

Mix เป็น build tool ที่มาพร้อมกับ Elixir ใช้สำหรับ:
- สร้างโปรเจ็กต์
- จัดการ dependencies
- Compile โค้ด
- Run tests
- Generate tasks

### สร้างโปรเจ็กต์แรก

```bash
# สร้างโปรเจ็กต์ใหม่
mix new hello_world

# Output:
* creating README.md
* creating .formatter.exs
* creating .gitignore
* creating mix.exs
* creating lib/
* creating lib/hello_world.ex
* creating test/
* creating test/test_helper.exs
* creating test/hello_world_test.exs

Your Mix project was created successfully.
You can use "mix" to compile it, test it, and more:

    cd hello_world
    mix test

Run "mix help" for more commands.
```

### โครงสร้างโปรเจ็กต์

```
hello_world/
├── .formatter.exs      # Code formatting config
├── .gitignore          # Git ignore rules
├── mix.exs             # Project configuration
├── README.md           # Project documentation
├── lib/
│   └── hello_world.ex  # Main module
└── test/
    ├── test_helper.exs # Test configuration
    └── hello_world_test.exs  # Tests
```

### ไฟล์ mix.exs

```elixir
# hello_world/mix.exs
defmodule HelloWorld.MixProject do
  use Mix.Project

  def project do
    [
      app: :hello_world,
      version: "0.1.0",
      elixir: "~> 1.16",
      start_permanent: Mix.env() == :prod,
      deps: deps()
    ]
  end

  # OTP Application configuration
  def application do
    [
      extra_applications: [:logger]
    ]
  end

  # Dependencies
  defp deps do
    [
      # เพิ่ม dependencies ตรงนี้
    ]
  end
end
```

### คำสั่ง Mix พื้นฐาน

```bash
# Compile โปรเจ็กต์
mix compile

# Run tests
mix test

# เริ่ม IEx พร้อม project loaded
mix iex -S mix
# หรือ
iex -S mix

# ดูคำสั่งทั้งหมด
mix help

# สร้าง release
mix release

# Format code
mix format

# ดู dependencies
mix deps

# ติดตั้ง dependencies
mix deps.get

# อัปเดต dependencies
mix deps.update --all

# ดู documentation
mix docs
```

### ไฟล์ lib/hello_world.ex

```elixir
# hello_world/lib/hello_world.ex
defmodule HelloWorld do
  @moduledoc """
  Documentation for `HelloWorld`.
  """

  @doc """
  Hello world.

  ## Examples

      iex> HelloWorld.hello()
      :world

  """
  def hello do
    :world
  end
end
```

### ทดสอบโปรเจ็กต์

```bash
cd hello_world
mix test
```

Output:
```
Compiling 1 file (.ex)
Generated hello_world app
Running ExUnit with seed: 123456, max_cases: 8

.

Finished in 0.01 seconds (0.00s async, 0.01s sync)
1 test, 0 failures
```

---

## 6. เขียนโค้ด Hello World แรก

### แก้ไข lib/hello_world.ex

```elixir
defmodule HelloWorld do
  @moduledoc """
  Module สำหรับบทเรียน Hello World
  """

  def hello do
    IO.puts("Hello, World!")
  end

  def greet(name) do
    IO.puts("Hello, #{name}!")
  end

  def greet_many(names) do
    Enum.each(names, fn name ->
      IO.puts("Hello, #{name}!")
    end)
  end
end
```

### ทดสอบใน IEx

```bash
# เริ่ม IEx พร้อม project
iex -S mix
```

```elixir
iex(1)> HelloWorld.hello()
Hello, World!
:ok

iex(2)> HelloWorld.greet("Elixir Developer")
Hello, Elixir Developer!
:ok

iex(3)> HelloWorld.greet_many(["Alice", "Bob", "Charlie"])
Hello, Alice!
Hello, Bob!
Hello, Charlie!
:ok
```

---

## 7. Code Formatting

Elixir มี built-in code formatter:

```bash
# Format ไฟล์เดียว
mix format lib/hello_world.ex

# Format ทั้งโปรเจ็กต์
mix format

# ตรวจสอบว่า format ถูกต้องหรือไม่ (ไม่แก้ไข)
mix format --check-formatted
```

### .formatter.exs

```elixir
# .formatter.exs
[
  inputs: ["{mix,.formatter}.exs", "{config,lib,test}/**/*.{ex,exs}"]
]
```

---

## 8. Editor Setup

### VS Code

ติดตั้ง Extensions:
1. **ElixirLS** - Elixir Language Server
2. **Elixir Formatter** - Code formatting
3. **Phoenix** - Phoenix framework support

```json
// settings.json
{
  "[elixir]": {
    "editor.formatOnSave": true,
    "editor.defaultFormatter": "JakeBecker.elixir-ls"
  },
  "elixirLS.suggestSpecs": true,
  "elixirLS.dialyzerEnabled": true
}
```

### Neovim

```lua
-- LSP configuration
require('lspconfig').elixirls.setup({
  cmd = { "elixir-ls" },
  settings = {
    elixirLS = {
      dialyzerEnabled = true,
      fetchDeps = false,
    }
  }
})
```

### Emacs

```elisp
;; Install elixir-mode and lsp-mode
(use-package elixir-mode
  :ensure t)

(use-package lsp-mode
  :ensure t
  :hook (elixir-mode . lsp))
```

---

## 9. Elixir Ecosystem

### Package Manager: Hex

```bash
# ค้นหา package
mix hex.search jason

# ดู info ของ package
mix hex.info jason

# เพิ่มใน mix.exs
defp deps do
  [
    {:jason, "~> 1.4"}
  ]
end

# ติดตั้ง
mix deps.get
```

### Popular Libraries

| Library | ใช้ทำอะไร |
|---------|-----------|
| Phoenix | Web Framework |
| Ecto | Database ORM |
| Jason | JSON encoding/decoding |
| Plug | HTTP middleware |
| Tesla | HTTP client |
| Oban | Background jobs |
| Absinthe | GraphQL |
| Ash | Resource framework |
| Nx | Numerical computing |
| Bumblebee | Machine learning |

---

## 10. แหล่งเรียนรู้เพิ่มเติม

### Official Resources
- **Documentation**: https://hexdocs.pm/elixir
- **Getting Started**: https://elixir-lang.org/getting-started
- **Hex Packages**: https://hex.pm

### Books
1. **Programming Elixir** by Dave Thomas
2. **Elixir in Action** by Saša Jurić
3. **The Little Elixir & OTP Guidebook** by Benjamin Tan
4. **Programming Phoenix** by Chris McCord

### Online Courses
- Elixir School (https://elixirschool.com)
- Exercism Elixir Track
- Pragmatic Studio Elixir/OTP

### Community
- **ElixirForum**: https://elixirforum.com
- **Elixir Slack**: https://elixir-slackin.herokuapp.com
- **Reddit**: r/elixir

---

## 11. สรุป Concepts สำคัญ

```
Elixir Key Concepts:
┌─────────────────────────────────────────────────────────┐
│                                                         │
│  1. Functional Programming                              │
│     - Functions as first-class citizens                 │
│     - Immutable data                                    │
│     - No side effects (pure functions)                  │
│                                                         │
│  2. Pattern Matching                                    │
│     - = is match operator, not assignment               │
│     - Powerful for control flow                         │
│                                                         │
│  3. Concurrency                                         │
│     - Lightweight processes                             │
│     - Message passing                                   │
│     - No shared memory                                  │
│                                                         │
│  4. OTP (Open Telecom Platform)                         │
│     - GenServer                                         │
│     - Supervisor                                        │
│     - Application                                       │
│                                                         │
│  5. BEAM Virtual Machine                                │
│     - Garbage collection per process                    │
│     - Preemptive scheduling                             │
│     - Hot code reloading                                │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

---

## แบบฝึกหัด

### Exercise 1: ติดตั้งและทดสอบ
1. ติดตั้ง Elixir บนเครื่องของคุณ
2. รัน `elixir --version` และบันทึก version ที่ได้
3. เปิด IEx และคำนวณ `2 ** 10`

### Exercise 2: สร้างโปรเจ็กต์
1. สร้างโปรเจ็กต์ใหม่ชื่อ `my_first_app`
2. แก้ไข module ให้มี function `add(a, b)` ที่บวกตัวเลข
3. ทดสอบใน IEx

### Exercise 3: IEx Exploration
ใน IEx ลองรันคำสั่งเหล่านี้:
```elixir
# 1. ดู documentation ของ String module
h String

# 2. ดู documentation ของ String.split
h String.split

# 3. ทดสอบ String.split
String.split("hello world", " ")

# 4. ดู info ของ string
i "hello"

# 5. ลอง tab completion โดยพิมพ์ "Str" แล้วกด Tab
```

### Exercise 4: Mix Commands
```bash
# 1. รัน mix help และดูคำสั่งทั้งหมด
mix help

# 2. ใน project ของคุณ รัน
mix compile
mix test

# 3. ลอง format code
mix format
```

---

## คำถามทบทวน

1. Elixir ทำงานบน VM ชื่ออะไร?
2. IEx ย่อมาจากอะไร?
3. Mix ใช้สำหรับทำอะไร?
4. ทำไม Elixir ถึงเหมาะกับ concurrent applications?
5. คำสั่งอะไรที่ใช้สร้างโปรเจ็กต์ใหม่?

---

## เฉลย

1. BEAM (Bogdan's Erlang Abstract Machine)
2. Interactive Elixir
3. Build tool: compile, test, manage dependencies, create projects
4. เพราะ BEAM รองรับ millions of lightweight processes ที่ทำงานพร้อมกัน
5. `mix new <project_name>`

---

*ต่อไป: [Part 02 - ไวยากรณ์พื้นฐาน](part_02.md)*
