# Part 62: Phoenix Livebook (Interactive Notebooks)

## เป้าหมายการเรียนรู้
- ติดตั้งและใช้งาน Livebook
- ใช้ Smart Cells สำหรับ database, maps, charts
- สร้าง charts ด้วย VegaLite
- ใช้ Kino สำหรับ interactive widgets
- แชร์และ publish notebooks
- Workflow สำหรับ data exploration
- เชื่อม Livebook กับ Elixir projects

---

## 1. Livebook คืออะไรและติดตั้งอย่างไร?

Livebook เป็น interactive notebook สำหรับ Elixir คล้ายกับ Jupyter Notebook แต่สร้างมาโดยเฉพาะสำหรับ Elixir พร้อม features เช่น reactive cells, collaboration, smart cells

```bash
# วิธีติดตั้งและรัน Livebook

# Option 1: ผ่าน Mix
mix do local.livebook, livebook server

# Option 2: Docker
docker run -p 8080:8080 -p 8081:8081 \
  --pull always \
  -u $(id -u):$(id -g) \
  -v $(pwd):/data \
  ghcr.io/livebook-dev/livebook

# Option 3: ดาวน์โหลด Desktop App
# https://livebook.dev/

# Option 4: เพิ่มเข้า Phoenix project สำหรับ dev
# mix.exs
{:livebook, "~> 0.12", only: :dev}
```

เข้าใช้งานที่ `http://localhost:8080`

---

## 2. โครงสร้าง Notebook

Notebook ประกอบด้วย cells หลายประเภท:

```markdown
# Notebook Structure

## Markdown Cells
- อธิบาย context, ทฤษฎี, หรือ instructions
- รองรับ LaTeX สำหรับ math formulas
- รองรับ Mermaid diagrams

## Code Cells
- รัน Elixir code
- แต่ละ cell share bindings กับ cells อื่นๆ ใน section เดียวกัน
- มี auto-complete และ inline documentation

## Smart Cells
- UI เฉพาะสำหรับ common tasks
- Database queries, Map visualizations, Charts
- สร้าง code โดยอัตโนมัติ

## Branch Cells
- รัน code แบบ parallel ใน separate process
- ไม่ block main session
```

---

## 3. Smart Cells - Database Query

Smart Cell สำหรับ database ช่วยสร้าง SQL query ด้วย visual interface

```elixir
# ขั้นแรก: ตั้งค่า database connection
# ใน Setup section ของ Notebook

Mix.install([
  {:kino_db, "~> 0.2"},
  {:postgrex, ">= 0.0.0"}
])
```

Smart Cell จะสร้าง code ดังนี้:

```elixir
# Code ที่ Smart Cell สร้างให้อัตโนมัติ
opts = [
  hostname: "localhost",
  port: 5432,
  username: "postgres",
  password: "postgres",
  database: "my_app_dev"
]

{:ok, conn} = Kino.start_child({Postgrex, opts})
```

```elixir
# SQL Query Smart Cell
result = Postgrex.query!(conn, """
  SELECT 
    date_trunc('day', inserted_at) AS date,
    COUNT(*) AS order_count,
    SUM(total_amount) AS revenue
  FROM orders
  WHERE inserted_at >= NOW() - INTERVAL '30 days'
    AND status = 'completed'
  GROUP BY date
  ORDER BY date DESC
""", [])

Kino.DataTable.new(result.rows,
  keys: Enum.map(result.columns, &String.to_atom/1)
)
```

---

## 4. VegaLite Charts

VegaLite เป็น declarative visualization library ที่ใช้ใน Livebook

```elixir
# Setup
Mix.install([
  {:vega_lite, "~> 0.1"},
  {:kino_vega_lite, "~> 0.1"},
  {:explorer, "~> 0.7"}
])

alias VegaLite, as: Vl
```

```elixir
# Bar Chart: Revenue by month
data = [
  %{month: "Jan", revenue: 42000},
  %{month: "Feb", revenue: 58000},
  %{month: "Mar", revenue: 51000},
  %{month: "Apr", revenue: 67000},
  %{month: "May", revenue: 73000},
  %{month: "Jun", revenue: 88000}
]

Vl.new(width: 600, height: 300, title: "Monthly Revenue 2024")
|> Vl.data_from_values(data)
|> Vl.mark(:bar, color: "#4299E1")
|> Vl.encode_field(:x, "month",
  type: :ordinal,
  axis: [title: "Month"]
)
|> Vl.encode_field(:y, "revenue",
  type: :quantitative,
  axis: [title: "Revenue (THB)", format: "~s"]
)
```

```elixir
# Line Chart with multiple series
sales_data = [
  %{date: "2024-01", product: "Widget A", sales: 120},
  %{date: "2024-01", product: "Widget B", sales: 85},
  %{date: "2024-02", product: "Widget A", sales: 145},
  %{date: "2024-02", product: "Widget B", sales: 92},
  %{date: "2024-03", product: "Widget A", sales: 130},
  %{date: "2024-03", product: "Widget B", sales: 110},
  %{date: "2024-04", product: "Widget A", sales: 160},
  %{date: "2024-04", product: "Widget B", sales: 125}
]

Vl.new(width: 600, height: 300, title: "Product Sales Comparison")
|> Vl.data_from_values(sales_data)
|> Vl.mark(:line, point: true)
|> Vl.encode_field(:x, "date", type: :ordinal)
|> Vl.encode_field(:y, "sales", type: :quantitative)
|> Vl.encode_field(:color, "product", type: :nominal)
```

```elixir
# Scatter Plot with tooltips
user_data = Enum.map(1..100, fn i ->
  %{
    user_id: i,
    age: :rand.uniform(50) + 20,
    purchase_count: :rand.uniform(20),
    total_spent: :rand.uniform(5000) + 100,
    segment: Enum.random(["Premium", "Regular", "Budget"])
  }
end)

Vl.new(width: 500, height: 400, title: "Customer Segmentation")
|> Vl.data_from_values(user_data)
|> Vl.mark(:point, opacity: 0.7)
|> Vl.encode_field(:x, "purchase_count",
  type: :quantitative,
  axis: [title: "Number of Purchases"]
)
|> Vl.encode_field(:y, "total_spent",
  type: :quantitative,
  axis: [title: "Total Spent (THB)"]
)
|> Vl.encode_field(:color, "segment", type: :nominal)
|> Vl.encode_field(:size, "age", type: :quantitative)
|> Vl.encode_field(:tooltip, [
  [field: "user_id", type: :nominal],
  [field: "segment", type: :nominal],
  [field: "total_spent", type: :quantitative]
])
```

```elixir
# Heatmap: Orders by day/hour
heatmap_data = for day <- 0..6, hour <- 0..23 do
  %{
    day: ["Sun", "Mon", "Tue", "Wed", "Thu", "Fri", "Sat"] |> Enum.at(day),
    hour: hour,
    orders: :rand.uniform(50)
  }
end

Vl.new(width: 600, height: 300, title: "Orders Heatmap (Day vs Hour)")
|> Vl.data_from_values(heatmap_data)
|> Vl.mark(:rect)
|> Vl.encode_field(:x, "hour",
  type: :ordinal,
  axis: [title: "Hour of Day"]
)
|> Vl.encode_field(:y, "day",
  type: :ordinal,
  sort: ["Mon", "Tue", "Wed", "Thu", "Fri", "Sat", "Sun"]
)
|> Vl.encode_field(:color, "orders",
  type: :quantitative,
  scale: [scheme: "blues"]
)
|> Vl.config(view: [stroke_width: 0])
```

---

## 5. Kino - Interactive Widgets

Kino เป็น library สำหรับสร้าง interactive elements ใน Livebook

```elixir
# Setup Kino
Mix.install([{:kino, "~> 0.12"}])
```

```elixir
# Input widgets
name_input = Kino.Input.text("Customer Name", default: "Alice")
age_input = Kino.Input.number("Age", default: 25)
segment_input = Kino.Input.select("Segment", [
  {"Premium", :premium},
  {"Regular", :regular},
  {"Budget", :budget}
])
date_input = Kino.Input.date("Start Date")
```

```elixir
# อ่านค่าจาก inputs
name = Kino.Input.read(name_input)
age = Kino.Input.read(age_input)
segment = Kino.Input.read(segment_input)

IO.puts("Name: #{name}, Age: #{age}, Segment: #{segment}")
```

```elixir
# Form widget (รวม inputs ในแบบฟอร์ม)
form = Kino.Control.form([
  name: Kino.Input.text("Name"),
  email: Kino.Input.text("Email"),
  role: Kino.Input.select("Role", [
    {"Admin", :admin},
    {"User", :user},
    {"Viewer", :viewer}
  ]),
  active: Kino.Input.checkbox("Active", default: true)
], submit: "Save User")

Kino.listen(form, fn %{data: data} ->
  IO.inspect(data, label: "Form submitted")
end)

form
```

```elixir
# DataTable - แสดงข้อมูลแบบ interactive table
users = [
  %{id: 1, name: "Alice", email: "alice@example.com", role: "admin"},
  %{id: 2, name: "Bob", email: "bob@example.com", role: "user"},
  %{id: 3, name: "Charlie", email: "charlie@example.com", role: "user"},
  %{id: 4, name: "Diana", email: "diana@example.com", role: "viewer"}
]

Kino.DataTable.new(users, keys: [:id, :name, :email, :role])
```

```elixir
# Kino.Frame - อัพเดต content แบบ dynamic
frame = Kino.Frame.new()

# ใส่ content เริ่มต้น
Kino.Frame.render(frame, Kino.Markdown.new("**Loading...**"))

# อัพเดต frame ทุก 1 วินาที
Task.start(fn ->
  Enum.each(1..10, fn i ->
    Kino.Frame.render(frame, Kino.Markdown.new("**Count: #{i}**"))
    Process.sleep(1_000)
  end)
  Kino.Frame.render(frame, Kino.Markdown.new("**Done!**"))
end)

frame
```

```elixir
# Kino.Layout - จัดวาง widgets
chart1 = Vl.new(width: 300, height: 200)
|> Vl.data_from_values(Enum.map(1..10, &%{x: &1, y: &1 * &1}))
|> Vl.mark(:line)
|> Vl.encode_field(:x, "x", type: :quantitative)
|> Vl.encode_field(:y, "y", type: :quantitative)

chart2 = Vl.new(width: 300, height: 200)
|> Vl.data_from_values(Enum.map(1..10, &%{x: &1, y: :math.sin(&1)}))
|> Vl.mark(:line, color: "red")
|> Vl.encode_field(:x, "x", type: :quantitative)
|> Vl.encode_field(:y, "y", type: :quantitative)

# แสดง side by side
Kino.Layout.grid([chart1, chart2], columns: 2)
```

---

## 6. Explorer สำหรับ DataFrame Operations

Explorer เป็น DataFrame library สำหรับ Elixir คล้าย pandas ใน Python

```elixir
Mix.install([
  {:explorer, "~> 0.7"},
  {:kino_explorer, "~> 0.1"}
])

alias Explorer.DataFrame, as: DF
alias Explorer.Series
```

```elixir
# โหลดข้อมูลจาก CSV
df = DF.from_csv!("sales_data.csv")

# หรือสร้างจาก map
df = DF.new(%{
  date: ["2024-01-01", "2024-01-02", "2024-01-03", "2024-01-04", "2024-01-05"],
  product: ["A", "B", "A", "C", "B"],
  quantity: [10, 5, 8, 12, 3],
  price: [100.0, 250.0, 100.0, 75.0, 250.0],
  region: ["North", "South", "North", "East", "South"]
})

# แสดง DataFrame แบบ interactive ใน Livebook
df
```

```elixir
# Data exploration
IO.puts("Shape: #{inspect(DF.shape(df))}")
IO.puts("Columns: #{inspect(DF.names(df))}")

# Describe statistics
DF.describe(df)
```

```elixir
# Filtering
north_sales = DF.filter(df, region == "North")

# Multiple conditions
high_value = DF.filter(df, price > 100.0 and quantity >= 5)
```

```elixir
# Aggregation
summary = df
|> DF.group_by(["product"])
|> DF.summarise(
  total_quantity: quantity |> Series.sum(),
  total_revenue: Series.multiply(quantity, price) |> Series.sum(),
  avg_price: price |> Series.mean()
)

summary
```

```elixir
# สร้าง computed column
df_with_revenue = DF.mutate(df,
  revenue: Series.multiply(quantity, price)
)

# Sort
df_sorted = DF.arrange_with(df_with_revenue, [desc: revenue])
df_sorted
```

---

## 7. Real-time Data Visualization

```elixir
# Real-time chart ที่อัพเดตตัวเองทุก 500ms
alias VegaLite, as: Vl

# สร้าง chart ด้วย Kino.VegaLite ที่รองรับ dynamic updates
chart = Vl.new(width: 600, height: 300, title: "Live Sensor Data")
|> Vl.mark(:line)
|> Vl.encode_field(:x, "time",
  type: :quantitative,
  axis: [title: "Time (s)"]
)
|> Vl.encode_field(:y, "value",
  type: :quantitative,
  axis: [title: "Sensor Value"],
  scale: [domain: [0, 100]]
)
|> Kino.VegaLite.new()

# เพิ่ม data points แบบ streaming
Task.start(fn ->
  start_time = System.system_time(:second)

  Enum.each(1..60, fn i ->
    time = System.system_time(:second) - start_time
    value = 50 + 30 * :math.sin(time * 0.5) + :rand.normal() * 5

    Kino.VegaLite.push(chart, %{time: time, value: Float.round(value, 2)},
      window: 30  # แสดงแค่ 30 points ล่าสุด
    )

    Process.sleep(500)
  end)
end)

chart
```

---

## 8. เชื่อม Livebook กับ Elixir Projects

```elixir
# ใช้ Mix.install เพื่อโหลด local project
Mix.install([
  {:my_app, path: "/path/to/my_app"},
  {:kino, "~> 0.12"},
  {:vega_lite, "~> 0.1"},
  {:kino_vega_lite, "~> 0.1"}
])

# หรือใช้ App mode (แนะนำ)
# เริ่ม Livebook ภายใน Phoenix project:
# iex -S mix livebook.server
```

```elixir
# ใช้ตรงๆ กับ Application modules
# เมื่อ run ใน app mode, modules ทั้งหมดพร้อมใช้งาน

# ดึงข้อมูลจาก database ผ่าน Repo
orders = MyApp.Repo.all(
  from o in MyApp.Order,
  where: o.status == :completed,
  where: o.inserted_at >= ^DateTime.add(DateTime.utc_now(), -30, :day),
  select: %{
    id: o.id,
    total: o.total_amount,
    date: o.inserted_at
  }
)

# Visualize
Vl.new(width: 600, height: 300, title: "Orders Last 30 Days")
|> Vl.data_from_values(orders)
|> Vl.mark(:point)
|> Vl.encode_field(:x, "date", type: :temporal)
|> Vl.encode_field(:y, "total", type: :quantitative)
```

```elixir
# ทดสอบ business logic ใน notebook
# สะดวกสำหรับ debugging และ exploration

alias MyApp.{Pricing, Discounts}

# ทดสอบ pricing calculation
test_order = %{
  items: [
    %{product_id: "prod-1", quantity: 2, base_price: 150.0},
    %{product_id: "prod-2", quantity: 1, base_price: 300.0}
  ],
  customer_tier: :premium,
  coupon_code: "SAVE20"
}

pricing_result = Pricing.calculate(test_order)
IO.inspect(pricing_result, label: "Pricing Result")
```

---

## 9. Data Exploration Workflow

```elixir
# Workflow สมบูรณ์สำหรับ data analysis

# Step 1: Load data
df = DF.from_csv!("/data/sales_2024.csv")
IO.puts("Loaded #{DF.n_rows(df)} rows")

# Step 2: Inspect data quality
null_counts = Enum.map(DF.names(df), fn col ->
  null_count = df[col] |> Series.nil_count()
  {col, null_count}
end)
IO.inspect(null_counts, label: "Null counts per column")

# Step 3: Clean data
cleaned_df = df
|> DF.drop_nil(:amount)
|> DF.filter(amount > 0)
|> DF.mutate(
  date: Series.cast(date, :date),
  amount: Series.cast(amount, :float)
)

# Step 4: Feature engineering
enriched_df = DF.mutate(cleaned_df,
  month: Series.month(date),
  day_of_week: Series.day_of_week(date),
  is_weekend: Series.day_of_week(date) |> Series.greater(5)
)

# Step 5: Analysis
monthly_stats = enriched_df
|> DF.group_by(["month"])
|> DF.summarise(
  total_sales: amount |> Series.sum(),
  avg_sale: amount |> Series.mean(),
  count: amount |> Series.count()
)

monthly_stats
```

```elixir
# Step 6: Visualize findings
Vl.new(width: 700, height: 400, title: "Monthly Sales Analysis")
|> Vl.data_from_values(DF.to_rows(monthly_stats))
|> Vl.layers([
  Vl.new()
  |> Vl.mark(:bar, opacity: 0.7, color: "#4299E1")
  |> Vl.encode_field(:x, "month", type: :ordinal)
  |> Vl.encode_field(:y, "total_sales",
    type: :quantitative,
    axis: [title: "Total Sales"]
  ),
  Vl.new()
  |> Vl.mark(:line, color: "red", point: true)
  |> Vl.encode_field(:x, "month", type: :ordinal)
  |> Vl.encode_field(:y, "count",
    type: :quantitative,
    axis: [title: "Order Count"]
  )
  |> Vl.resolve_scale(y: :independent)
])
```

---

## 10. Sharing และ Publishing Notebooks

```elixir
# Livebook สามารถ export เป็น:
# 1. .livemd file (Markdown with embedded code)
# 2. .html static page
# 3. PDF

# สร้าง notebook ที่ import ได้
# File > Export > Notebook source (.livemd)
```

```markdown
<!-- ตัวอย่าง .livemd file structure -->

# Sales Analysis Report

```elixir
Mix.install([
  {:explorer, "~> 0.7"},
  {:kino_vega_lite, "~> 0.1"}
])
```

## Loading Data

```elixir
df = Explorer.DataFrame.from_csv!("sales.csv")
```

## Analysis

```elixir
# code cells...
```
```

```elixir
# Livebook Teams: แชร์กับทีม
# - Centralized notebook hosting
# - Access control
# - Audit logs
# - Secrets management

# สำหรับ Elixir deployment
# .livemd สามารถ commit ลง git ได้
# และรันผ่าน CI/CD pipeline

# ตัวอย่าง: รัน notebook ใน CI
# livebook export notebook.livemd output.html
```

---

## สรุป

```
Livebook Ecosystem:

┌─────────────────────────────────────────────┐
│              Livebook Interface              │
│  ┌──────────┐ ┌──────────┐ ┌────────────┐  │
│  │ Markdown │ │  Code    │ │   Smart    │  │
│  │  Cells   │ │  Cells   │ │   Cells    │  │
│  └──────────┘ └──────────┘ └────────────┘  │
└──────────────────────────────────────────────┘
         │              │            │
         ▼              ▼            ▼
   Documentation    Execution   Auto-generated
                                    code
                        │
         ┌──────────────┼──────────────┐
         ▼              ▼              ▼
     VegaLite         Kino          Explorer
    (Charts)       (Widgets)     (DataFrames)

Use Cases:
- Data exploration และ analysis
- ทำ reports ที่ reproducible
- Document business logic ด้วย code
- Interactive dashboards
- Teaching และ tutorials
- Debugging production issues

Key Libraries:
- Kino: interactive widgets, tables, frames
- VegaLite + Kino.VegaLite: charts
- Explorer: DataFrame operations
- Kino.DB: database smart cells
- Kino.MapLibre: map visualizations
```

---

*ก่อนหน้า: [Part 61 - WebAssembly and Nerves IoT](part_61.md) | ต่อไป: [Part 63 - Advanced OTP Patterns](part_63.md)*
