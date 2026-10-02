# Part 19: Mix: Build Tool

## เป้าหมายการเรียนรู้

หลังจากจบ Part นี้ คุณจะสามารถ:
- เข้าใจโครงสร้างของ `mix.exs`
- สร้าง custom Mix tasks
- จัดการ Environments (dev/test/prod)
- จัดการ dependencies
- สร้าง CLI tool ด้วย Mix

---

## 1. Mix คืออะไร?

Mix คือ build tool ที่มาพร้อม Elixir

```
Mix Commands:
┌──────────────────────────────────────────────────────────────┐
│  mix new          - สร้าง project ใหม่                      │
│  mix compile      - compile โค้ด                            │
│  mix test         - รัน tests                               │
│  mix deps.get     - ดาวน์โหลด dependencies                 │
│  mix deps.compile - compile dependencies                     │
│  mix run          - รัน script                              │
│  mix release      - สร้าง production release               │
│  mix format       - format โค้ด                             │
│  mix docs         - สร้าง documentation                     │
└──────────────────────────────────────────────────────────────┘
```

---

## 2. mix.exs

```elixir
# mix.exs - heart of a Mix project
defmodule MyApp.MixProject do
  use Mix.Project

  def project do
    [
      app: :my_app,           # application name (atom)
      version: "0.1.0",       # semantic versioning
      elixir: "~> 1.14",      # required Elixir version
      start_permanent: Mix.env() == :prod,
      deps: deps(),
      description: "My awesome app",
      package: package(),
      aliases: aliases(),
      test_coverage: [tool: ExCoveralls],
      preferred_cli_env: [
        "test.coverage": :test,
        coveralls: :test
      ]
    ]
  end

  def application do
    [
      extra_applications: [:logger, :crypto],
      mod: {MyApp.Application, []}
    ]
  end

  defp deps do
    [
      {:jason, "~> 1.4"},
      {:httpoison, "~> 2.0"},
      {:ecto, "~> 3.10"},
      {:ecto_sql, "~> 3.10"},
      {:postgrex, ">= 0.0.0"},
      {:phoenix, "~> 1.7", only: [:dev, :test]},
      {:ex_doc, "~> 0.30", only: :dev, runtime: false},
      {:credo, "~> 1.7", only: [:dev, :test], runtime: false},
      {:dialyxir, "~> 1.3", only: [:dev], runtime: false},
      {:mox, "~> 1.0", only: :test},
      {:excoveralls, "~> 0.16", only: :test}
    ]
  end

  defp package do
    [
      licenses: ["MIT"],
      links: %{"GitHub" => "https://github.com/myorg/my_app"}
    ]
  end

  defp aliases do
    [
      setup: ["deps.get", "ecto.setup"],
      "ecto.setup": ["ecto.create", "ecto.migrate", "run priv/repo/seeds.exs"],
      "ecto.reset": ["ecto.drop", "ecto.setup"],
      test: ["ecto.create --quiet", "ecto.migrate --quiet", "test"]
    ]
  end
end
```

---

## 3. Environments

```elixir
# Mix.env() คืน :dev, :test, หรือ :prod
Mix.env()

# config/config.exs - ทุก environments
import Config

config :my_app,
  ecto_repos: [MyApp.Repo]

config :logger,
  level: :info

# Import environment-specific config
import_config "#{config_env()}.exs"

# config/dev.exs
import Config

config :my_app, MyApp.Repo,
  username: "postgres",
  password: "postgres",
  hostname: "localhost",
  database: "my_app_dev"

config :logger, :console,
  format: "[$level] $message\n",
  metadata: [:request_id]

# config/test.exs
import Config

config :my_app, MyApp.Repo,
  username: "postgres",
  password: "postgres",
  hostname: "localhost",
  database: "my_app_test#{System.get_env("MIX_TEST_PARTITION")}",
  pool: Ecto.Adapters.SQL.Sandbox,
  pool_size: 10

config :logger, level: :warning

# config/prod.exs
import Config

config :my_app, MyApp.Repo,
  url: System.get_env("DATABASE_URL"),
  pool_size: String.to_integer(System.get_env("POOL_SIZE") || "10")

# config/runtime.exs - evaluated at runtime (ไม่ใช่ compile time)
import Config

if config_env() == :prod do
  database_url =
    System.get_env("DATABASE_URL") ||
      raise "environment variable DATABASE_URL is missing"

  config :my_app, MyApp.Repo, url: database_url

  secret_key_base =
    System.get_env("SECRET_KEY_BASE") ||
      raise "environment variable SECRET_KEY_BASE is missing"

  config :my_app, MyAppWeb.Endpoint,
    secret_key_base: secret_key_base
end
```

---

## 4. Custom Mix Tasks

```elixir
# lib/mix/tasks/hello.ex
defmodule Mix.Tasks.Hello do
  use Mix.Task

  @shortdoc "Say hello"

  @moduledoc """
  Prints hello message.

  ## Usage

      mix hello
      mix hello --name World

  ## Options

    * `--name` - the name to greet (default: "World")
  """

  def run(args) do
    {opts, _args, _invalid} = OptionParser.parse(args,
      switches: [name: :string],
      aliases: [n: :name]
    )

    name = Keyword.get(opts, :name, "World")
    Mix.shell().info("Hello, #{name}!")
  end
end
```

### Mix Task ที่ซับซ้อนขึ้น

```elixir
# lib/mix/tasks/db.seed.ex
defmodule Mix.Tasks.Db.Seed do
  use Mix.Task

  @shortdoc "Seed the database with sample data"

  @moduledoc """
  Seeds the database with sample data.

      mix db.seed
      mix db.seed --count 100
      mix db.seed --env production
  """

  def run(args) do
    {opts, _args, _} = OptionParser.parse(args,
      switches: [count: :integer, env: :string, truncate: :boolean],
      aliases: [c: :count, e: :env, t: :truncate]
    )

    count = Keyword.get(opts, :count, 10)
    should_truncate = Keyword.get(opts, :truncate, false)

    # Start the application
    Mix.Task.run("app.start")

    if should_truncate do
      Mix.shell().info("Truncating tables...")
      truncate_tables()
    end

    Mix.shell().info("Seeding #{count} users...")
    seed_users(count)

    Mix.shell().info("Seeding products...")
    seed_products(count * 5)

    Mix.shell().info("Done! Seeded #{count} users and #{count * 5} products.")
  end

  defp truncate_tables do
    # ลบ data เก่า
    MyApp.Repo.delete_all(MyApp.User)
    MyApp.Repo.delete_all(MyApp.Product)
  end

  defp seed_users(count) do
    Enum.map(1..count, fn i ->
      %{
        name: "User #{i}",
        email: "user#{i}@example.com",
        inserted_at: NaiveDateTime.utc_now() |> NaiveDateTime.truncate(:second),
        updated_at: NaiveDateTime.utc_now() |> NaiveDateTime.truncate(:second)
      }
    end)
    |> then(fn users ->
      MyApp.Repo.insert_all(MyApp.User, users)
    end)
  end

  defp seed_products(count) do
    categories = [:electronics, :clothing, :food, :books, :sports]
    Enum.map(1..count, fn i ->
      %{
        name: "Product #{i}",
        price: (:rand.uniform(10000) / 100),
        category: Enum.random(categories),
        stock: :rand.uniform(100),
        inserted_at: NaiveDateTime.utc_now() |> NaiveDateTime.truncate(:second),
        updated_at: NaiveDateTime.utc_now() |> NaiveDateTime.truncate(:second)
      }
    end)
    |> then(fn products ->
      MyApp.Repo.insert_all(MyApp.Product, products)
    end)
  end
end
```

---

## 5. Mix Task Helpers

```elixir
# Mix.shell() - output functions
Mix.shell().info("Regular message")
Mix.shell().error("Error message")
Mix.shell().warn("Warning message")

# ถามผู้ใช้
if Mix.shell().yes?("Are you sure?") do
  do_dangerous_thing()
end

# Mix.raise - raise error
Mix.raise("Something went wrong")

# Mix.Task.run - รัน task อื่น
Mix.Task.run("deps.get")
Mix.Task.run("compile")
Mix.Task.run("ecto.migrate")
```

---

## 6. Dependencies Management

```elixir
# Version requirements
{:dep, "~> 1.0"}    # >= 1.0.0, < 2.0.0
{:dep, "~> 1.1.0"}  # >= 1.1.0, < 1.2.0
{:dep, ">= 1.0"}    # 1.0 or higher
{:dep, "== 1.0.0"}  # exactly 1.0.0
{:dep, "~> 1.0 and < 1.5"} # range

# Source options
{:dep, "~> 1.0", only: :dev}           # dev only
{:dep, "~> 1.0", only: [:dev, :test]}  # dev + test
{:dep, "~> 1.0", runtime: false}       # compile only

# Git dependency
{:dep, git: "https://github.com/user/repo.git", tag: "v1.0.0"}
{:dep, git: "https://github.com/user/repo.git", branch: "main"}
{:dep, git: "https://github.com/user/repo.git", ref: "abc123"}

# Path dependency (local)
{:dep, path: "../my_other_project"}

# Hex.pm organization
{:dep, "~> 1.0", organization: "my_org"}
```

### Mix commands สำหรับ deps

```bash
mix deps.get           # ดาวน์โหลด deps
mix deps.update dep    # update specific dep
mix deps.update --all  # update ทุก deps
mix deps.clean dep     # ลบ compiled dep
mix deps.clean --all   # ลบทุก compiled deps
mix deps.unlock dep    # unlock version
mix deps.unlock --all  # unlock ทุก versions
mix deps.audit         # ตรวจ security vulnerabilities
```

---

## 7. Mix Aliases

```elixir
# mix.exs
defp aliases do
  [
    # รัน multiple tasks
    setup: ["deps.get", "compile", "ecto.setup"],

    # Custom function
    "test.all": [&run_all_tests/1],

    # ส่ง arguments
    "gen.schema": ["ecto.gen.migration", "phx.gen.schema"],

    # Run with specific env
    "test.integration": ["test --only integration"],

    # Development shortcuts
    dev: ["phx.server"],

    # Deployment helpers
    deploy: [
      "deps.get --only prod",
      "compile",
      "assets.deploy",
      "phx.digest",
      "release"
    ]
  ]
end

defp run_all_tests(args) do
  Mix.shell().info("Running all tests...")
  Mix.Task.run("test", args ++ ["--cover"])
  Mix.shell().info("Tests complete!")
end
```

---

## 8. ตัวอย่างจริง: CLI Tool

```elixir
# สร้าง CLI tool สำหรับ manage tasks

defmodule Mix.Tasks.Tasks do
  use Mix.Task

  @shortdoc "Manage TODO tasks"

  @moduledoc """
  A simple task manager from the command line.

  ## Commands

      mix tasks list              # List all tasks
      mix tasks list --pending    # List pending only
      mix tasks add "Task title"  # Add a new task
      mix tasks done <id>         # Mark task as done
      mix tasks delete <id>       # Delete a task

  """

  @task_file "tasks.json"

  def run(args) do
    case args do
      ["list" | rest] -> list_tasks(rest)
      ["add" | rest] -> add_task(rest)
      ["done", id] -> mark_done(id)
      ["delete", id] -> delete_task(id)
      [] -> help()
      _ -> Mix.shell().error("Unknown command. Run `mix tasks` for help.")
    end
  end

  defp list_tasks(opts) do
    {parsed, _, _} = OptionParser.parse(opts, switches: [pending: :boolean, completed: :boolean])
    tasks = load_tasks()

    filtered = case parsed do
      [pending: true] -> Enum.filter(tasks, &(not &1["completed"]))
      [completed: true] -> Enum.filter(tasks, & &1["completed"])
      _ -> tasks
    end

    if filtered == [] do
      Mix.shell().info("No tasks found.")
    else
      Mix.shell().info("\nTasks:")
      Mix.shell().info("------")
      Enum.each(filtered, fn task ->
        status = if task["completed"], do: "[x]", else: "[ ]"
        Mix.shell().info("#{status} #{task["id"]}. #{task["title"]}")
      end)
      Mix.shell().info("")
    end
  end

  defp add_task([]) do
    Mix.shell().error("Please provide a task title.")
  end

  defp add_task(words) do
    title = Enum.join(words, " ")
    tasks = load_tasks()
    next_id = (tasks |> Enum.map(& &1["id"]) |> Enum.max(fn -> 0 end)) + 1

    new_task = %{
      "id" => next_id,
      "title" => title,
      "completed" => false,
      "created_at" => DateTime.utc_now() |> DateTime.to_iso8601()
    }

    save_tasks([new_task | tasks])
    Mix.shell().info("Added task ##{next_id}: #{title}")
  end

  defp mark_done(id_str) do
    case Integer.parse(id_str) do
      {id, ""} ->
        tasks = load_tasks()
        case Enum.find_index(tasks, fn t -> t["id"] == id end) do
          nil ->
            Mix.shell().error("Task ##{id} not found.")
          index ->
            updated = List.update_at(tasks, index, fn t ->
              Map.put(t, "completed", true)
            end)
            save_tasks(updated)
            Mix.shell().info("Task ##{id} marked as done!")
        end
      _ ->
        Mix.shell().error("Invalid task ID: #{id_str}")
    end
  end

  defp delete_task(id_str) do
    case Integer.parse(id_str) do
      {id, ""} ->
        tasks = load_tasks()
        task = Enum.find(tasks, fn t -> t["id"] == id end)
        if task do
          if Mix.shell().yes?("Delete task '#{task["title"]}'?") do
            save_tasks(Enum.reject(tasks, fn t -> t["id"] == id end))
            Mix.shell().info("Task ##{id} deleted.")
          end
        else
          Mix.shell().error("Task ##{id} not found.")
        end
      _ ->
        Mix.shell().error("Invalid task ID: #{id_str}")
    end
  end

  defp help do
    Mix.shell().info("""

    Task Manager - manage your TODO tasks from the command line

    Usage:
      mix tasks list              List all tasks
      mix tasks list --pending    List pending tasks only
      mix tasks list --completed  List completed tasks only
      mix tasks add "task title"  Add a new task
      mix tasks done <id>         Mark task #id as done
      mix tasks delete <id>       Delete task #id

    """)
  end

  defp load_tasks do
    case File.read(@task_file) do
      {:ok, content} ->
        case Jason.decode(content) do
          {:ok, tasks} -> tasks
          _ -> []
        end
      {:error, _} -> []
    end
  end

  defp save_tasks(tasks) do
    sorted = Enum.sort_by(tasks, & &1["id"])
    File.write!(@task_file, Jason.encode!(sorted, pretty: true))
  end
end
```

### ตัวอย่าง Report Generator

```elixir
defmodule Mix.Tasks.Reports.Generate do
  use Mix.Task

  @shortdoc "Generate project reports"

  def run(args) do
    {opts, types, _} = OptionParser.parse(args,
      switches: [
        format: :string,
        output: :string,
        from: :string,
        to: :string
      ]
    )

    Mix.Task.run("app.start")

    format = Keyword.get(opts, :format, "text")
    output = Keyword.get(opts, :output, "report.#{format}")
    report_types = if types == [], do: ["summary"], else: types

    Mix.shell().info("Generating reports: #{Enum.join(report_types, ", ")}")

    report_data = Enum.reduce(report_types, %{}, fn type, acc ->
      Mix.shell().info("  - #{type}...")
      Map.put(acc, type, generate_report(type, opts))
    end)

    case format do
      "json" ->
        File.write!(output, Jason.encode!(report_data, pretty: true))
      "text" ->
        content = format_as_text(report_data)
        File.write!(output, content)
      _ ->
        Mix.raise("Unknown format: #{format}. Use 'json' or 'text'")
    end

    Mix.shell().info("Report saved to #{output}")
  end

  defp generate_report("summary", _opts) do
    # ใน production จะ query DB จริงๆ
    %{
      total_users: 1250,
      active_users: 890,
      new_users_today: 45,
      generated_at: DateTime.utc_now()
    }
  end

  defp generate_report("revenue", opts) do
    from = Keyword.get(opts, :from, "2024-01-01")
    to = Keyword.get(opts, :to, Date.to_string(Date.utc_today()))
    %{
      period: "#{from} to #{to}",
      total: 125_000.00,
      transactions: 3_420
    }
  end

  defp generate_report(type, _) do
    Mix.shell().warn("Unknown report type: #{type}")
    %{error: "unknown report type"}
  end

  defp format_as_text(data) do
    Enum.map(data, fn {type, content} ->
      """
      === #{String.upcase(to_string(type))} ===
      #{format_map(content)}
      """
    end)
    |> Enum.join("\n")
  end

  defp format_map(map) when is_map(map) do
    Enum.map(map, fn {k, v} -> "  #{k}: #{v}" end) |> Enum.join("\n")
  end
end
```

---

## 9. mix release

```elixir
# mix.exs
def project do
  [
    ...
    releases: [
      my_app: [
        include_executables_for: [:unix],
        applications: [runtime_tools: :permanent],
        steps: [:assemble, :tar]
      ]
    ]
  ]
end
```

```bash
# สร้าง release
MIX_ENV=prod mix release

# รัน release
_build/prod/rel/my_app/bin/my_app start
_build/prod/rel/my_app/bin/my_app daemon  # background
_build/prod/rel/my_app/bin/my_app stop
```

---

## 10. Exercises

### Exercise 1: Custom Compile Task

```elixir
# สร้าง task ที่:
# 1. ตรวจสอบ code quality ก่อน compile
# 2. แสดง dependency versions
# 3. รัน credo ถ้า available

defmodule Mix.Tasks.Build do
  use Mix.Task

  @shortdoc "Build with quality checks"
  # TODO: implement
end
```

### Exercise 2: Database Backup Task

```elixir
# สร้าง task ที่:
# 1. Backup database ไปยัง JSON file
# 2. Compress ถ้าใส่ --compress flag
# 3. ชื่อไฟล์มี timestamp

defmodule Mix.Tasks.Db.Backup do
  use Mix.Task
  @shortdoc "Backup database to file"
  # TODO: implement
end
```

### Exercise 3: Health Check Task

```elixir
# สร้าง task ที่ตรวจสอบ:
# 1. Database connection
# 2. External API reachability
# 3. Disk space
# 4. Memory usage
# แสดงผลสี (green/red) สำหรับแต่ละ check

defmodule Mix.Tasks.Health do
  use Mix.Task
  @shortdoc "Check application health"
  # TODO: implement
end
```

---

## เฉลย Exercises

### เฉลย Exercise 3: Health Check

```elixir
defmodule Mix.Tasks.Health do
  use Mix.Task

  @shortdoc "Check application health"

  @green "\e[32m"
  @red "\e[31m"
  @yellow "\e[33m"
  @reset "\e[0m"

  def run(_args) do
    Mix.Task.run("app.start")

    Mix.shell().info("\nApplication Health Check")
    Mix.shell().info("========================")

    checks = [
      {"Database", &check_database/0},
      {"Memory", &check_memory/0},
      {"Disk Space", &check_disk/0},
      {"Process Count", &check_processes/0}
    ]

    results = Enum.map(checks, fn {name, check_fn} ->
      result = check_fn.()
      print_result(name, result)
      result
    end)

    Mix.shell().info("\n========================")
    failed = Enum.count(results, fn {status, _} -> status == :fail end)
    if failed == 0 do
      Mix.shell().info("#{@green}All checks passed!#{@reset}")
    else
      Mix.shell().error("#{@red}#{failed} check(s) failed!#{@reset}")
      System.halt(1)
    end
  end

  defp check_database do
    try do
      MyApp.Repo.query!("SELECT 1")
      {:ok, "Connected"}
    rescue
      e -> {:fail, "Cannot connect: #{Exception.message(e)}"}
    end
  end

  defp check_memory do
    mem = :erlang.memory()
    total_mb = div(mem[:total], 1_048_576)
    if total_mb < 500 do
      {:ok, "#{total_mb} MB"}
    else
      {:warn, "#{total_mb} MB (high)"}
    end
  end

  defp check_disk do
    case System.cmd("df", ["-h", "/"]) do
      {output, 0} ->
        line = output |> String.split("\n") |> Enum.at(1)
        {:ok, String.trim(line)}
      {_, _} ->
        {:warn, "Cannot check disk"}
    end
  end

  defp check_processes do
    count = length(Process.list())
    if count < 10_000 do
      {:ok, "#{count} processes"}
    else
      {:warn, "#{count} processes (high)"}
    end
  end

  defp print_result(name, {:ok, msg}) do
    Mix.shell().info("#{@green}✓#{@reset} #{name}: #{msg}")
  end
  defp print_result(name, {:warn, msg}) do
    Mix.shell().info("#{@yellow}!#{@reset} #{name}: #{msg}")
  end
  defp print_result(name, {:fail, msg}) do
    Mix.shell().error("#{@red}✗#{@reset} #{name}: #{msg}")
  end
end
```

---

## สรุป

```
mix.exs Structure:
├── project/0 - project metadata
├── application/0 - OTP application config
├── deps/0 - dependencies
└── aliases/0 - custom shortcuts

Environments:
├── config/config.exs - all envs
├── config/dev.exs
├── config/test.exs
├── config/prod.exs
└── config/runtime.exs - runtime evaluation

Custom Tasks:
├── defmodule Mix.Tasks.Name
├── use Mix.Task
├── @shortdoc "one-liner"
├── @moduledoc "full docs"
└── def run(args)

Dependencies:
├── Version: ~>, >=, ==
├── only: :dev | :test | [:dev, :test]
├── runtime: false (compile only)
└── git: / path: for non-hex deps
```

---

*ก่อนหน้า: [Part 18](part_18.md) | ต่อไป: [Part 20 - ExUnit: Testing Framework](part_20.md)*
