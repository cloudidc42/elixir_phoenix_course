# Part 63: Livebook - Elixir Notebooks (สมุดบันทึกแบบ Interactive)

## เป้าหมายการเรียนรู้

หลังจากจบ Part นี้ คุณจะสามารถ:
- ใช้ Livebook สำหรับ data analysis
- Smart cells สำหรับ database, charts, maps
- Deploy Livebook บน production
- ใช้ Livebook เป็น internal tool

---

## 1. Livebook Setup

```bash
# ติดตั้งผ่าน Mix
mix escript.install hex livebook

# รัน Livebook
livebook server --port 8080

# หรือผ่าน Docker
docker run -p 8080:8080 ghcr.io/livebook-dev/livebook

# Livebook Desktop App
# ดาวน์โหลดจาก https://livebook.dev
```

---

## 2. Data Analysis Notebook

```elixir
# Cell 1: Dependencies
Mix.install([
  {:explorer, "~> 0.8"},
  {:kino, "~> 0.12"},
  {:kino_vega_lite, "~> 0.1"},
])

# Cell 2: Load Data
alias Explorer.DataFrame
alias Explorer.Series

df = DataFrame.from_csv!("/path/to/sales_data.csv")
DataFrame.head(df)

# Cell 3: Basic Statistics
DataFrame.describe(df)

# Cell 4: Filter and transform
top_products =
  df
  |> DataFrame.filter(col("revenue") > 10_000)
  |> DataFrame.group_by("product_category")
  |> DataFrame.summarise(
    total_revenue: sum(col("revenue")),
    avg_revenue: mean(col("revenue")),
    count: count(col("id"))
  )
  |> DataFrame.sort_by(desc: col("total_revenue"))

# Cell 5: Visualize
VegaLite.new(width: 600, height: 400)
|> VegaLite.data_from_values(DataFrame.to_rows(top_products))
|> VegaLite.mark(:bar)
|> VegaLite.encode_field(:x, "product_category", type: :nominal)
|> VegaLite.encode_field(:y, "total_revenue", type: :quantitative)
|> VegaLite.encode_field(:color, "product_category")
|> Kino.render()
```

---

## 3. Database Smart Cell

```elixir
# ใช้ Smart Cell สำหรับ database
# ใน Livebook คลิก "+ Smart cell" -> Database connection

# หลังเชื่อมต่อแล้ว สามารถ query ได้
# SQL Smart Cell จะ generate code:
result = Kino.DB.query!(conn, """
  SELECT
    DATE_TRUNC('month', created_at) AS month,
    COUNT(*) AS signups,
    SUM(revenue) AS total_revenue
  FROM users
  WHERE created_at >= NOW() - INTERVAL '6 months'
  GROUP BY 1
  ORDER BY 1
""")

Kino.DataTable.new(result)
```

---

## 4. Interactive Widgets

```elixir
# Form input
form = Kino.Control.form([
  start_date: Kino.Input.date("Start Date", default: ~D[2024-01-01]),
  end_date: Kino.Input.date("End Date", default: Date.utc_today()),
  category: Kino.Input.select("Category", [
    "All": nil,
    "Electronics": "electronics",
    "Clothing": "clothing"
  ])
], submit: "Apply Filter")

# React to form submission
Kino.listen(form, fn %{data: data} ->
  filtered = filter_data(data.start_date, data.end_date, data.category)
  chart = render_chart(filtered)
  Kino.render(chart)
end)
```

---

## 5. Live Process Monitoring

```elixir
# Monitor running processes in production
defmodule ProcessMonitor do
  def top_processes(n \\ 10) do
    Process.list()
    |> Enum.map(fn pid ->
      info = Process.info(pid, [:memory, :message_queue_len, :current_function, :registered_name])
      %{
        pid: inspect(pid),
        name: info[:registered_name],
        memory_kb: div(info[:memory], 1024),
        mq_len: info[:message_queue_len],
        current_fn: inspect(info[:current_function])
      }
    end)
    |> Enum.sort_by(& &1.memory_kb, :desc)
    |> Enum.take(n)
  end
end

# Auto-refresh table every 2 seconds
frame = Kino.Frame.new()

Kino.animate(frame, 2_000, fn _ ->
  data = ProcessMonitor.top_processes()
  Kino.DataTable.new(data)
end)
```

---

## 6. Livebook as Internal Tool

```elixir
# app.livemd - internal admin tool

# Cell 1: Connect to production DB
# (use Kino.DB smart cell)

# Cell 2: User management
user_input = Kino.Input.text("Email")
Kino.render(user_input)

button = Kino.Control.button("Find User")
Kino.listen(button, fn _event ->
  email = Kino.Input.read(user_input)
  user = MyApp.Accounts.get_user_by_email(email)
  Kino.render(Kino.DataTable.new([user]))
end)

# Cell 3: Bulk operations
# Backfill missing data, run migrations, etc.
defmodule BackfillJob do
  def run do
    MyApp.Repo.all(MyApp.User)
    |> Enum.filter(& is_nil(&1.slug))
    |> Enum.each(fn user ->
      slug = Slug.slugify(user.name)
      MyApp.Accounts.update_user(user, %{slug: slug})
    end)
  end
end

# Preview first before running
IO.puts("Found #{MyApp.Repo.aggregate(MyApp.User, :count)} users to process")
# BackfillJob.run()  # Uncomment to execute
```

---

## สรุป

```
Livebook Use Cases:
├── Data analysis: Explorer + VegaLite
├── Database queries: Smart cells
├── Interactive widgets: Kino.Control
├── Process monitoring: live views
└── Internal admin tools

Key Libraries:
├── Explorer: DataFrame API
├── Kino: widgets, controls, tables
├── VegaLite: charts
└── Kino.VegaLite: chart smart cell
```

---

*ก่อนหน้า: [Part 62](part_62.md) | ต่อไป: [Part 64 - Database Optimization](part_64.md)*
