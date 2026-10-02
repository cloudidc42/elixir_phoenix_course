# Part 64: Database Optimization Advanced

## เป้าหมายการเรียนรู้

- ใช้งาน PostgreSQL-specific features ผ่าน Ecto ได้อย่างมีประสิทธิภาพ
- จัดการข้อมูล JSONB และทำ full-text search ด้วย tsvector
- สร้างและใช้งาน Materialized Views เพื่อเพิ่มประสิทธิภาพ query
- เข้าใจ database partitioning patterns
- ตั้งค่า PgBouncer สำหรับ connection pooling
- ทำ zero-downtime migrations อย่างปลอดภัย

---

## 1. PostgreSQL-Specific Features ใน Ecto

Ecto รองรับ PostgreSQL features พิเศษผ่าน `Ecto.Adapters.Postgres` ซึ่งช่วยให้เราใช้ความสามารถเต็มของ PostgreSQL ได้โดยตรง

### 1.1 Array Type

```elixir
# migration
defmodule MyApp.Repo.Migrations.AddTagsToArticles do
  use Ecto.Migration

  def change do
    alter table(:articles) do
      add :tags, {:array, :string}, default: []
      add :scores, {:array, :integer}, default: []
    end

    # สร้าง GIN index สำหรับ array operations
    create index(:articles, [:tags], using: :gin)
  end
end
```

```elixir
# schema
defmodule MyApp.Articles.Article do
  use Ecto.Schema
  import Ecto.Changeset

  schema "articles" do
    field :title, :string
    field :tags, {:array, :string}, default: []
    field :scores, {:array, :integer}, default: []
    timestamps()
  end

  def changeset(article, attrs) do
    article
    |> cast(attrs, [:title, :tags, :scores])
    |> validate_required([:title])
  end
end
```

```elixir
# query operations
import Ecto.Query

# ค้นหา articles ที่มี tag "elixir"
Article
|> where(fragment("? @> ?", field(a, :tags), ^["elixir"]))
|> Repo.all()

# ค้นหา articles ที่มี tag อย่างน้อย 1 อันจากที่กำหนด (overlap)
Article
|> where(fragment("? && ?", field(a, :tags), ^["elixir", "phoenix"]))
|> Repo.all()

# เพิ่ม tag เข้า array (PostgreSQL array_append)
from(a in Article, where: a.id == ^article_id)
|> Repo.update_all(set: [tags: fragment("array_append(tags, ?)", "new_tag")])
```

### 1.2 Composite Types และ Range Types

```elixir
# ใช้ daterange สำหรับช่วงเวลา
defmodule MyApp.Repo.Migrations.AddEventSchedule do
  use Ecto.Migration

  def change do
    alter table(:events) do
      add :schedule, :daterange
      add :price_range, :numrange
    end

    # สร้าง index บน range type
    create index(:events, [:schedule], using: :gist)
  end
end
```

```elixir
# query ด้วย range operators
# @> = contains, && = overlaps, << = strictly left of
Event
|> where(fragment("? @> ?::date", e.schedule, ^Date.utc_today()))
|> Repo.all()
```

---

## 2. JSONB Operations ใน Ecto

JSONB เป็น binary format ของ JSON ใน PostgreSQL ที่ query ได้เร็วกว่า JSON ธรรมดา

### 2.1 Schema และ Migration สำหรับ JSONB

```elixir
# migration
defmodule MyApp.Repo.Migrations.AddMetadataToProducts do
  use Ecto.Migration

  def change do
    alter table(:products) do
      add :metadata, :map, default: %{}
      add :attributes, :map, default: %{}
    end

    # GIN index สำหรับ JSONB
    create index(:products, [:metadata], using: :gin)

    # Expression index สำหรับ specific key
    execute(
      "CREATE INDEX products_metadata_brand_idx ON products ((metadata->>'brand'))",
      "DROP INDEX products_metadata_brand_idx"
    )
  end
end
```

```elixir
# schema
defmodule MyApp.Catalog.Product do
  use Ecto.Schema

  schema "products" do
    field :name, :string
    field :price, :decimal
    field :metadata, :map, default: %{}
    field :attributes, :map, default: %{}
    timestamps()
  end
end
```

### 2.2 JSONB Query Operations

```elixir
defmodule MyApp.Catalog do
  import Ecto.Query

  # ค้นหาด้วย JSONB key
  def find_by_brand(brand) do
    from(p in Product,
      where: fragment("?->>'brand' = ?", p.metadata, ^brand)
    )
    |> Repo.all()
  end

  # ค้นหาด้วย nested JSONB
  def find_by_color(color) do
    from(p in Product,
      where: fragment("?->'attributes'->>'color' = ?", p.metadata, ^color)
    )
    |> Repo.all()
  end

  # ตรวจสอบ key existence
  def find_with_discount() do
    from(p in Product,
      where: fragment("? ? ?", p.metadata, "discount")
    )
    |> Repo.all()
  end

  # JSONB @> (contains)
  def find_featured_products() do
    from(p in Product,
      where: fragment("? @> ?", p.metadata, ^%{"featured" => true})
    )
    |> Repo.all()
  end

  # Update specific JSONB field ด้วย jsonb_set
  def update_product_rating(product_id, rating) do
    from(p in Product, where: p.id == ^product_id)
    |> Repo.update_all(
      set: [
        metadata: fragment(
          "jsonb_set(metadata, '{rating}', ?::jsonb)",
          ^to_string(rating)
        )
      ]
    )
  end

  # Aggregate JSONB data
  def get_brand_stats() do
    from(p in Product,
      group_by: fragment("?->>'brand'", p.metadata),
      select: {
        fragment("?->>'brand'", p.metadata),
        count(p.id),
        avg(p.price)
      }
    )
    |> Repo.all()
  end
end
```

### 2.3 Custom Ecto Type สำหรับ JSONB

```elixir
defmodule MyApp.Types.ProductMetadata do
  use Ecto.Type

  @type t :: %{
    brand: String.t() | nil,
    rating: float() | nil,
    featured: boolean(),
    tags: [String.t()]
  }

  def type, do: :map

  def cast(%{brand: _, rating: _, featured: _, tags: _} = value), do: {:ok, value}
  def cast(value) when is_map(value) do
    {:ok, %{
      brand: Map.get(value, "brand") || Map.get(value, :brand),
      rating: Map.get(value, "rating") || Map.get(value, :rating),
      featured: Map.get(value, "featured", false),
      tags: Map.get(value, "tags", [])
    }}
  end
  def cast(_), do: :error

  def load(value) when is_map(value), do: {:ok, cast_keys(value)}
  def load(_), do: :error

  def dump(value) when is_map(value), do: {:ok, value}
  def dump(_), do: :error

  defp cast_keys(map) do
    Map.new(map, fn {k, v} -> {String.to_atom(k), v} end)
  end
end
```

---

## 3. Full-Text Search ด้วย tsvector

PostgreSQL มี built-in full-text search ที่ทรงพลัง ใช้ `tsvector` เก็บ document และ `tsquery` สำหรับ search

### 3.1 ตั้งค่า Full-Text Search

```elixir
# migration สร้าง tsvector column และ trigger
defmodule MyApp.Repo.Migrations.AddFullTextSearchToArticles do
  use Ecto.Migration

  def up do
    # เพิ่ม tsvector column
    alter table(:articles) do
      add :search_vector, :tsvector
    end

    # สร้าง GIN index บน tsvector
    execute "CREATE INDEX articles_search_vector_idx ON articles USING GIN(search_vector)"

    # สร้าง function สำหรับ update search vector
    execute """
    CREATE OR REPLACE FUNCTION update_articles_search_vector()
    RETURNS trigger AS $$
    BEGIN
      NEW.search_vector :=
        setweight(to_tsvector('thai', coalesce(NEW.title, '')), 'A') ||
        setweight(to_tsvector('thai', coalesce(NEW.body, '')), 'B') ||
        setweight(to_tsvector('english', coalesce(NEW.title, '')), 'A') ||
        setweight(to_tsvector('english', coalesce(NEW.body, '')), 'B');
      RETURN NEW;
    END
    $$ LANGUAGE plpgsql;
    """

    # สร้าง trigger
    execute """
    CREATE TRIGGER articles_search_vector_update
    BEFORE INSERT OR UPDATE ON articles
    FOR EACH ROW EXECUTE FUNCTION update_articles_search_vector();
    """

    # Update existing records
    execute "UPDATE articles SET search_vector = NULL"
  end

  def down do
    execute "DROP TRIGGER IF EXISTS articles_search_vector_update ON articles"
    execute "DROP FUNCTION IF EXISTS update_articles_search_vector()"
    execute "DROP INDEX IF EXISTS articles_search_vector_idx"
    alter table(:articles) do
      remove :search_vector
    end
  end
end
```

### 3.2 Search Query

```elixir
defmodule MyApp.Search do
  import Ecto.Query

  def search_articles(query_string, opts \\ []) do
    limit = Keyword.get(opts, :limit, 20)
    offset = Keyword.get(opts, :offset, 0)

    # แปลง query string เป็น tsquery
    tsquery = format_tsquery(query_string)

    from(a in Article,
      where: fragment(
        "search_vector @@ to_tsquery('thai', ?)",
        ^tsquery
      ),
      order_by: fragment(
        "ts_rank(search_vector, to_tsquery('thai', ?)) DESC",
        ^tsquery
      ),
      select: %{
        id: a.id,
        title: a.title,
        published_at: a.published_at,
        # สร้าง headline (snippet with highlights)
        headline: fragment(
          "ts_headline('thai', ?, to_tsquery('thai', ?), 'MaxFragments=2,MaxWords=30')",
          a.body,
          ^tsquery
        ),
        rank: fragment(
          "ts_rank(search_vector, to_tsquery('thai', ?))",
          ^tsquery
        )
      },
      limit: ^limit,
      offset: ^offset
    )
    |> Repo.all()
  end

  # นับผลลัพธ์ทั้งหมด
  def count_search_results(query_string) do
    tsquery = format_tsquery(query_string)

    from(a in Article,
      where: fragment(
        "search_vector @@ to_tsquery('thai', ?)",
        ^tsquery
      ),
      select: count(a.id)
    )
    |> Repo.one()
  end

  # แปลง search term เป็น tsquery format
  defp format_tsquery(query_string) do
    query_string
    |> String.split(~r/\s+/)
    |> Enum.filter(&(String.length(&1) > 0))
    |> Enum.map(&(&1 <> ":*"))  # prefix matching
    |> Enum.join(" & ")
  end
end
```

---

## 4. Materialized Views

Materialized Views เก็บผลลัพธ์ของ query ไว้จริงๆ ช่วยให้ query ซับซ้อนทำงานเร็วขึ้นมาก

### 4.1 สร้าง Materialized View

```elixir
defmodule MyApp.Repo.Migrations.CreateProductStatsMaterializedView do
  use Ecto.Migration

  def up do
    execute """
    CREATE MATERIALIZED VIEW product_stats AS
    SELECT
      p.category_id,
      c.name as category_name,
      COUNT(p.id) as product_count,
      AVG(p.price) as avg_price,
      MIN(p.price) as min_price,
      MAX(p.price) as max_price,
      SUM(oi.quantity) as total_sold,
      SUM(oi.quantity * p.price) as total_revenue
    FROM products p
    JOIN categories c ON c.id = p.category_id
    LEFT JOIN order_items oi ON oi.product_id = p.id
    GROUP BY p.category_id, c.name
    WITH DATA;
    """

    execute "CREATE UNIQUE INDEX ON product_stats (category_id)"
    execute "CREATE INDEX ON product_stats (total_revenue DESC)"
  end

  def down do
    execute "DROP MATERIALIZED VIEW IF EXISTS product_stats"
  end
end
```

### 4.2 Schema สำหรับ Materialized View

```elixir
defmodule MyApp.Analytics.ProductStats do
  use Ecto.Schema

  # ไม่ต้องการ migration schema สำหรับ view
  @primary_key false
  schema "product_stats" do
    field :category_id, :integer
    field :category_name, :string
    field :product_count, :integer
    field :avg_price, :decimal
    field :min_price, :decimal
    field :max_price, :decimal
    field :total_sold, :integer
    field :total_revenue, :decimal
  end
end
```

### 4.3 Refresh Strategy

```elixir
defmodule MyApp.Analytics do
  import Ecto.Query

  # Refresh แบบ CONCURRENT (ไม่ lock table)
  def refresh_product_stats(concurrent \\ true) do
    if concurrent do
      Repo.query!("REFRESH MATERIALIZED VIEW CONCURRENTLY product_stats")
    else
      Repo.query!("REFRESH MATERIALIZED VIEW product_stats")
    end
  end

  # ตั้งเวลา refresh ด้วย Oban
  def schedule_refresh do
    %{view: "product_stats"}
    |> MyApp.Workers.RefreshMaterializedView.new(schedule_in: 0)
    |> Oban.insert()
  end
end
```

```elixir
# Oban Worker สำหรับ refresh
defmodule MyApp.Workers.RefreshMaterializedView do
  use Oban.Worker, queue: :analytics, max_attempts: 3

  @impl Oban.Worker
  def perform(%Oban.Job{args: %{"view" => view}}) do
    case view do
      "product_stats" ->
        MyApp.Repo.query!("REFRESH MATERIALIZED VIEW CONCURRENTLY product_stats")
        :ok
      _ ->
        {:error, "Unknown view: #{view}"}
    end
  end
end
```

---

## 5. Database Partitioning

Partitioning ช่วยจัดการตารางขนาดใหญ่โดยแบ่งข้อมูลออกเป็น partitions ย่อย

### 5.1 Range Partitioning (ตามเวลา)

```sql
-- สร้าง parent table
CREATE TABLE events (
  id BIGSERIAL,
  user_id INTEGER NOT NULL,
  event_type VARCHAR(50) NOT NULL,
  payload JSONB,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
) PARTITION BY RANGE (created_at);

-- สร้าง partitions รายเดือน
CREATE TABLE events_2024_01
  PARTITION OF events
  FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');

CREATE TABLE events_2024_02
  PARTITION OF events
  FOR VALUES FROM ('2024-02-01') TO ('2024-03-01');

-- สร้าง index บนแต่ละ partition
CREATE INDEX ON events_2024_01 (user_id, created_at);
CREATE INDEX ON events_2024_02 (user_id, created_at);
```

```elixir
# Auto-create partitions ด้วย Elixir
defmodule MyApp.Partitioning do
  def create_monthly_partition(table, year, month) do
    start_date = Date.new!(year, month, 1)
    end_date = Date.add(start_date, Date.days_in_month(start_date))
    partition_name = "#{table}_#{year}_#{String.pad_leading("#{month}", 2, "0")}"

    sql = """
    CREATE TABLE IF NOT EXISTS #{partition_name}
      PARTITION OF #{table}
      FOR VALUES FROM ('#{start_date}') TO ('#{end_date}');
    """

    MyApp.Repo.query!(sql)

    # สร้าง index บน partition ใหม่
    index_sql = "CREATE INDEX IF NOT EXISTS ON #{partition_name} (user_id, created_at)"
    MyApp.Repo.query!(index_sql)

    {:ok, partition_name}
  end

  # สร้าง partitions ล่วงหน้า 3 เดือน
  def ensure_future_partitions(table) do
    today = Date.utc_today()
    Enum.each(0..2, fn offset ->
      future_date = Date.add(today, offset * 30)
      create_monthly_partition(table, future_date.year, future_date.month)
    end)
  end
end
```

### 5.2 Hash Partitioning (กระจาย load)

```sql
-- Hash partitioning สำหรับ high-write tables
CREATE TABLE user_activities (
  id BIGSERIAL,
  user_id INTEGER NOT NULL,
  activity_type VARCHAR(50),
  created_at TIMESTAMPTZ DEFAULT NOW()
) PARTITION BY HASH (user_id);

-- แบ่งเป็น 8 partitions
CREATE TABLE user_activities_0 PARTITION OF user_activities
  FOR VALUES WITH (MODULUS 8, REMAINDER 0);
CREATE TABLE user_activities_1 PARTITION OF user_activities
  FOR VALUES WITH (MODULUS 8, REMAINDER 1);
-- ... ต่อไปจนถึง 7
```

---

## 6. PgBouncer Connection Pooling

PgBouncer ช่วยจัดการ database connections อย่างมีประสิทธิภาพ ลด overhead ของการ connect/disconnect

### 6.1 ตั้งค่า PgBouncer

```ini
# /etc/pgbouncer/pgbouncer.ini
[databases]
myapp_prod = host=127.0.0.1 port=5432 dbname=myapp_production

[pgbouncer]
listen_port = 6432
listen_addr = 127.0.0.1
auth_type = md5
auth_file = /etc/pgbouncer/userlist.txt

# Pool mode: session, transaction, statement
pool_mode = transaction

# Connection limits
max_client_conn = 1000
default_pool_size = 20
min_pool_size = 5
reserve_pool_size = 5
reserve_pool_timeout = 3

# Timeouts
server_connect_timeout = 15
server_idle_timeout = 600
client_idle_timeout = 0
query_timeout = 0

# Logging
log_connections = 1
log_disconnections = 1
log_pooler_errors = 1
stats_period = 60
```

### 6.2 ตั้งค่า Phoenix กับ PgBouncer

```elixir
# config/runtime.exs
config :my_app, MyApp.Repo,
  url: System.get_env("DATABASE_URL"),
  # เชื่อมต่อผ่าน PgBouncer port 6432
  hostname: System.get_env("PGBOUNCER_HOST", "localhost"),
  port: String.to_integer(System.get_env("PGBOUNCER_PORT", "6432")),
  pool_size: String.to_integer(System.get_env("POOL_SIZE", "10")),
  # สำคัญมาก: ปิด prepare statements เมื่อใช้ transaction mode
  prepare: :unnamed,
  # ปิด advisory locks (ไม่รองรับใน transaction mode)
  parameters: [application_name: "myapp"]
```

```elixir
# สำหรับ Ecto migration กับ PgBouncer
# ใช้ connection โดยตรงไปยัง PostgreSQL ไม่ผ่าน PgBouncer
defmodule MyApp.MigrationRepo do
  use Ecto.Repo,
    otp_app: :my_app,
    adapter: Ecto.Adapters.Postgres
end

# config/config.exs
config :my_app, MyApp.MigrationRepo,
  hostname: System.get_env("DB_HOST", "localhost"),
  port: 5432,  # direct PostgreSQL port
  database: "myapp_production",
  username: System.get_env("DB_USER"),
  password: System.get_env("DB_PASSWORD"),
  pool_size: 2
```

---

## 7. Zero-Downtime Migrations

การทำ migration โดยไม่หยุด service ต้องใช้เทคนิคพิเศษ

### 7.1 หลักการ Expand-Contract Pattern

```elixir
# PHASE 1: Expand - เพิ่ม column ใหม่ (backward compatible)
defmodule MyApp.Repo.Migrations.Phase1AddNewColumn do
  use Ecto.Migration

  # ไม่ lock transaction เพื่อ zero-downtime
  @disable_ddl_transaction true
  @disable_migration_lock true

  def change do
    alter table(:users) do
      # เพิ่ม column ใหม่ที่ nullable
      add_if_not_exists :full_name, :string
    end
  end
end
```

```elixir
# PHASE 2: Migrate data (backfill)
defmodule MyApp.Repo.Migrations.Phase2BackfillFullName do
  use Ecto.Migration

  @disable_ddl_transaction true
  @disable_migration_lock true

  def up do
    # Batch update เพื่อไม่ lock table นานเกินไป
    repo().query!("""
    DO $$
    DECLARE
      batch_size INTEGER := 1000;
      last_id INTEGER := 0;
      max_id INTEGER;
    BEGIN
      SELECT MAX(id) INTO max_id FROM users;

      WHILE last_id < max_id LOOP
        UPDATE users
        SET full_name = first_name || ' ' || last_name
        WHERE id > last_id
          AND id <= last_id + batch_size
          AND full_name IS NULL;

        last_id := last_id + batch_size;
        PERFORM pg_sleep(0.01); -- หน่วงเล็กน้อยเพื่อลด load
      END LOOP;
    END $$;
    """)
  end

  def down, do: :ok
end
```

```elixir
# PHASE 3: Contract - ลบ column เก่า (หลังจาก deploy code ใหม่แล้ว)
defmodule MyApp.Repo.Migrations.Phase3RemoveOldColumns do
  use Ecto.Migration

  @disable_ddl_transaction true
  @disable_migration_lock true

  def change do
    alter table(:users) do
      remove_if_exists :first_name, :string
      remove_if_exists :last_name, :string
    end
  end
end
```

### 7.2 Safe Index Creation

```elixir
defmodule MyApp.Repo.Migrations.AddIndexConcurrently do
  use Ecto.Migration

  # ต้องปิด DDL transaction เพื่อใช้ CONCURRENTLY
  @disable_ddl_transaction true
  @disable_migration_lock true

  def change do
    # CONCURRENTLY ไม่ lock table ขณะสร้าง index
    create index(:orders, [:user_id, :status], concurrently: true)
    create index(:orders, [:created_at], concurrently: true)
  end
end
```

### 7.3 Renaming Column อย่างปลอดภัย

```elixir
# วิธีที่ถูกต้อง: ใช้ virtual column (generated column)
defmodule MyApp.Repo.Migrations.SafeRenameColumn do
  use Ecto.Migration

  def up do
    # Step 1: เพิ่ม column ใหม่
    alter table(:products) do
      add :product_name, :string
    end

    # Step 2: Copy data
    execute "UPDATE products SET product_name = name WHERE product_name IS NULL"

    # Step 3: ทำให้ column ใหม่เป็น NOT NULL (หลัง backfill)
    alter table(:products) do
      modify :product_name, :string, null: false
    end

    # Step 4: สร้าง trigger เพื่อ sync ระหว่าง deploy
    execute """
    CREATE OR REPLACE FUNCTION sync_product_name()
    RETURNS TRIGGER AS $$
    BEGIN
      IF TG_OP = 'INSERT' OR NEW.name IS DISTINCT FROM OLD.name THEN
        NEW.product_name := NEW.name;
      END IF;
      IF TG_OP = 'INSERT' OR NEW.product_name IS DISTINCT FROM OLD.product_name THEN
        NEW.name := NEW.product_name;
      END IF;
      RETURN NEW;
    END;
    $$ LANGUAGE plpgsql;
    """

    execute """
    CREATE TRIGGER sync_product_name_trigger
    BEFORE INSERT OR UPDATE ON products
    FOR EACH ROW EXECUTE FUNCTION sync_product_name();
    """
  end

  def down do
    execute "DROP TRIGGER IF EXISTS sync_product_name_trigger ON products"
    execute "DROP FUNCTION IF EXISTS sync_product_name()"
    alter table(:products) do
      remove :product_name
    end
  end
end
```

---

## 8. Query Performance Monitoring

```elixir
defmodule MyApp.QueryMonitor do
  require Logger

  # ติดตาม slow queries ผ่าน Ecto telemetry
  def setup do
    :telemetry.attach(
      "slow-query-monitor",
      [:my_app, :repo, :query],
      &handle_query_event/4,
      nil
    )
  end

  def handle_query_event(_event, measurements, metadata, _config) do
    duration_ms = System.convert_time_unit(measurements.total_time, :native, :millisecond)

    if duration_ms > 100 do
      Logger.warning("""
      Slow query detected: #{duration_ms}ms
      Query: #{metadata.query}
      Params: #{inspect(metadata.params)}
      Source: #{metadata.source}
      """)

      # ส่ง metric ไปยัง monitoring system
      :telemetry.execute(
        [:my_app, :slow_query],
        %{duration: duration_ms},
        %{query: metadata.query}
      )
    end
  end
end
```

```elixir
# EXPLAIN ANALYZE ผ่าน Ecto
defmodule MyApp.QueryAnalyzer do
  def explain(query) do
    {sql, params} = Ecto.Adapters.SQL.to_sql(:all, MyApp.Repo, query)

    explain_sql = "EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT) #{sql}"
    %{rows: rows} = MyApp.Repo.query!(explain_sql, params)
    rows |> List.flatten() |> Enum.join("\n")
  end
end

# ใช้งาน:
# Article |> where([a], a.status == "published") |> MyApp.QueryAnalyzer.explain() |> IO.puts()
```

---

## สรุป

```
Database Optimization Stack
════════════════════════════════════════════════════════
Application Layer (Elixir/Phoenix)
  ├── Ecto Queries (type-safe, composable)
  └── Telemetry (query monitoring)

Connection Layer
  └── PgBouncer (transaction pooling, 6432)
        └── PostgreSQL (5432)

PostgreSQL Features
  ├── Data Types
  │     ├── JSONB (flexible schema, GIN indexed)
  │     ├── Array (multi-value fields)
  │     └── Range (date/number ranges)
  ├── Full-Text Search
  │     ├── tsvector (indexed document)
  │     ├── tsquery (search expression)
  │     └── ts_rank (relevance scoring)
  ├── Materialized Views
  │     ├── Pre-computed aggregates
  │     └── CONCURRENT refresh (no lock)
  └── Partitioning
        ├── Range (time-based)
        └── Hash (write distribution)

Migration Best Practices
  ├── Expand: Add nullable column
  ├── Backfill: Batch update data
  ├── Contract: Remove old column
  └── Index: CREATE CONCURRENTLY
════════════════════════════════════════════════════════
```

---

*ก่อนหน้า: [Part 63](part_63.md) | ต่อไป: [Part 65 - Real-time Notifications System](part_65.md)*
