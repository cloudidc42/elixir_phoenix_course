# Part 44: Ecto Advanced (Ecto ขั้นสูง)

## เป้าหมายการเรียนรู้

- ใช้ Ecto.Multi สำหรับ complex transactions ที่ต้องการ atomicity
- สร้าง Custom Ecto types สำหรับ domain-specific data
- ใช้ Embedded schemas สำหรับ structured data ที่ไม่ต้องการ table แยก
- สร้าง multi-tenancy patterns ด้วย prefix-based approach
- ใช้ database constraints และ check constraints อย่างถูกต้อง
- สร้าง soft delete pattern
- ทำ upserts และ bulk operations อย่างมีประสิทธิภาพ
- Compose queries อย่างเป็นระบบ

---

## 1. Ecto.Multi สำหรับ Complex Transactions

Ecto.Multi ช่วยให้เราทำหลายๆ database operations ใน single transaction และจัดการ error ได้ง่าย

```elixir
# lib/my_app/accounts.ex
defmodule MyApp.Accounts do
  alias Ecto.Multi
  alias MyApp.{Repo, Mailer}
  alias MyApp.Accounts.{User, Profile, WelcomeEmail}

  @doc """
  สร้าง user พร้อม profile และส่ง welcome email
  ทั้งหมดใน single transaction
  """
  def register_user(attrs) do
    Multi.new()
    |> Multi.insert(:user, User.registration_changeset(%User{}, attrs))
    |> Multi.insert(:profile, fn %{user: user} ->
      Profile.changeset(%Profile{}, %{user_id: user.id, display_name: user.name})
    end)
    |> Multi.run(:send_welcome_email, fn _repo, %{user: user} ->
      case Mailer.send_welcome(user) do
        {:ok, _} -> {:ok, user}
        {:error, reason} -> {:error, reason}
      end
    end)
    |> Multi.update(:mark_email_sent, fn %{user: user} ->
      User.changeset(user, %{welcome_email_sent: true})
    end)
    |> Repo.transaction()
    |> case do
      {:ok, %{user: user}} -> {:ok, user}
      {:error, :user, changeset, _} -> {:error, changeset}
      {:error, :profile, changeset, _} -> {:error, changeset}
      {:error, :send_welcome_email, reason, _} -> {:error, reason}
    end
  end

  @doc "Transfer credits ระหว่าง users"
  def transfer_credits(from_user, to_user, amount) do
    Multi.new()
    |> Multi.run(:check_balance, fn repo, _ ->
      from = repo.reload(from_user)
      if from.credits >= amount do
        {:ok, from}
      else
        {:error, :insufficient_credits}
      end
    end)
    |> Multi.update(:debit_sender, fn %{check_balance: from} ->
      User.changeset(from, %{credits: from.credits - amount})
    end)
    |> Multi.update(:credit_receiver, fn _ ->
      User.changeset(to_user, %{credits: to_user.credits + amount})
    end)
    |> Multi.insert(:transaction_log, fn %{check_balance: from} ->
      TransactionLog.changeset(%TransactionLog{}, %{
        from_user_id: from.id,
        to_user_id: to_user.id,
        amount: amount,
        type: "transfer"
      })
    end)
    |> Repo.transaction()
    |> case do
      {:ok, _} -> :ok
      {:error, :check_balance, :insufficient_credits, _} ->
        {:error, "Insufficient credits"}
      {:error, step, reason, _} ->
        {:error, "Transfer failed at #{step}: #{inspect(reason)}"}
    end
  end
end
```

```elixir
# Multi.merge สำหรับ conditional operations
def process_order(order_attrs) do
  base_multi =
    Multi.new()
    |> Multi.insert(:order, Order.changeset(%Order{}, order_attrs))

  # เพิ่ม inventory check เฉพาะเมื่อ product มี inventory tracking
  multi =
    if order_attrs[:product_id] do
      Multi.merge(base_multi, fn %{order: order} ->
        Multi.new()
        |> Multi.run(:check_inventory, fn repo, _ ->
          product = repo.get!(Product, order.product_id)
          if product.stock >= order.quantity do
            {:ok, product}
          else
            {:error, :out_of_stock}
          end
        end)
        |> Multi.update(:decrement_stock, fn %{check_inventory: product} ->
          Product.changeset(product, %{stock: product.stock - order.quantity})
        end)
      end)
    else
      base_multi
    end

  Repo.transaction(multi)
end
```

---

## 2. Custom Ecto Types

Custom types ช่วยให้ทำงานกับ domain-specific data ได้สะดวก

```elixir
# lib/my_app/types/money.ex
defmodule MyApp.Types.Money do
  @moduledoc """
  Custom Ecto type สำหรับ money values
  เก็บเป็น integer (satangs) ใน DB แต่ใช้ Decimal ใน Elixir
  """
  use Ecto.Type

  def type, do: :integer

  def cast(value) when is_integer(value) do
    {:ok, Decimal.new(value)}
  end

  def cast(value) when is_float(value) do
    {:ok, Decimal.from_float(value)}
  end

  def cast(%Decimal{} = value), do: {:ok, value}

  def cast(value) when is_binary(value) do
    case Decimal.parse(value) do
      {decimal, ""} -> {:ok, decimal}
      _ -> :error
    end
  end

  def cast(_), do: :error

  # แปลงจาก Elixir value เป็น DB value (satangs)
  def dump(%Decimal{} = value) do
    {:ok, Decimal.to_integer(Decimal.mult(value, 100))}
  end

  def dump(value) when is_integer(value), do: {:ok, value}
  def dump(_), do: :error

  # แปลงจาก DB value เป็น Elixir value (baht)
  def load(value) when is_integer(value) do
    {:ok, Decimal.div(Decimal.new(value), 100)}
  end

  def load(_), do: :error
end
```

```elixir
# lib/my_app/types/encrypted_string.ex
defmodule MyApp.Types.EncryptedString do
  @moduledoc "Ecto type ที่ encrypt/decrypt อัตโนมัติ"
  use Ecto.Type

  def type, do: :binary

  def cast(value) when is_binary(value), do: {:ok, value}
  def cast(_), do: :error

  def dump(value) when is_binary(value) do
    encrypted = MyApp.Crypto.encrypt(value)
    {:ok, encrypted}
  end

  def dump(_), do: :error

  def load(value) when is_binary(value) do
    decrypted = MyApp.Crypto.decrypt(value)
    {:ok, decrypted}
  end

  def load(_), do: :error
end
```

```elixir
# lib/my_app/types/slug.ex
defmodule MyApp.Types.Slug do
  @moduledoc "Auto-generate URL-friendly slugs"
  use Ecto.Type

  def type, do: :string

  def cast(value) when is_binary(value) do
    slug =
      value
      |> String.downcase()
      |> String.replace(~r/[^a-z0-9\s-]/, "")
      |> String.replace(~r/\s+/, "-")
      |> String.trim("-")

    {:ok, slug}
  end

  def cast(_), do: :error
  def dump(value) when is_binary(value), do: {:ok, value}
  def dump(_), do: :error
  def load(value) when is_binary(value), do: {:ok, value}
  def load(_), do: :error
end
```

```elixir
# ใช้ custom types ใน schema
defmodule MyApp.Shop.Product do
  use Ecto.Schema
  import Ecto.Changeset

  schema "products" do
    field :name, :string
    field :slug, MyApp.Types.Slug       # auto-generate slug
    field :price, MyApp.Types.Money     # store as integer, use as Decimal
    field :secret_key, MyApp.Types.EncryptedString  # encrypted at rest

    timestamps()
  end

  def changeset(product, attrs) do
    product
    |> cast(attrs, [:name, :slug, :price, :secret_key])
    |> validate_required([:name, :price])
    |> put_slug_from_name()
  end

  defp put_slug_from_name(changeset) do
    case get_change(changeset, :name) do
      nil -> changeset
      name -> put_change(changeset, :slug, name)  # types.Slug.cast จะแปลงให้
    end
  end
end
```

---

## 3. Embedded Schemas

```elixir
# lib/my_app/accounts/address.ex
defmodule MyApp.Accounts.Address do
  use Ecto.Schema
  import Ecto.Changeset

  # embedded_schema ไม่มี table ใน DB
  embedded_schema do
    field :street, :string
    field :city, :string
    field :state, :string
    field :zip, :string
    field :country, :string, default: "TH"
  end

  def changeset(address, attrs) do
    address
    |> cast(attrs, [:street, :city, :state, :zip, :country])
    |> validate_required([:street, :city, :country])
    |> validate_length(:zip, min: 5, max: 10)
  end
end
```

```elixir
# lib/my_app/accounts/user.ex - ใช้ embeds_one
defmodule MyApp.Accounts.User do
  use Ecto.Schema
  import Ecto.Changeset

  schema "users" do
    field :name, :string
    field :email, :string

    # address เก็บใน users table เป็น JSONB column
    embeds_one :address, MyApp.Accounts.Address, on_replace: :update

    # สามารถมีหลาย embedded schemas ได้
    embeds_many :previous_addresses, MyApp.Accounts.Address, on_replace: :delete

    timestamps()
  end

  def changeset(user, attrs) do
    user
    |> cast(attrs, [:name, :email])
    |> cast_embed(:address, required: true)
    |> cast_embed(:previous_addresses)
    |> validate_required([:name, :email])
  end
end
```

```elixir
# Embedded schema สำหรับ product variants
defmodule MyApp.Shop.ProductVariant do
  use Ecto.Schema
  import Ecto.Changeset

  embedded_schema do
    field :sku, :string
    field :color, :string
    field :size, :string
    field :stock, :integer, default: 0
    field :extra_price, :decimal, default: 0
  end

  def changeset(variant, attrs) do
    variant
    |> cast(attrs, [:sku, :color, :size, :stock, :extra_price])
    |> validate_required([:sku, :stock])
    |> validate_number(:stock, greater_than_or_equal_to: 0)
  end
end

defmodule MyApp.Shop.Product do
  use Ecto.Schema

  schema "products" do
    field :name, :string
    field :base_price, :decimal

    # เก็บ variants ทั้งหมดใน products table
    embeds_many :variants, MyApp.Shop.ProductVariant, on_replace: :delete

    timestamps()
  end

  def changeset(product, attrs) do
    product
    |> cast(attrs, [:name, :base_price])
    |> cast_embed(:variants, required: true)
  end
end
```

---

## 4. Multi-tenancy Patterns

```elixir
# Prefix-based multi-tenancy - แต่ละ tenant มี schema ของตัวเองใน PostgreSQL
defmodule MyApp.Tenants do
  alias MyApp.Repo

  @doc "สร้าง schema สำหรับ tenant ใหม่"
  def create_tenant_schema(tenant_id) do
    schema_name = tenant_schema_name(tenant_id)

    Repo.query!("CREATE SCHEMA IF NOT EXISTS #{schema_name}")
    Repo.query!("SET search_path TO #{schema_name}, public")

    # รัน migrations สำหรับ schema ใหม่
    Ecto.Migrator.run(Repo, :up, prefix: schema_name)

    :ok
  end

  def tenant_schema_name(tenant_id), do: "tenant_#{tenant_id}"
end
```

```elixir
# lib/my_app_web/plugs/tenant_plug.ex
defmodule MyAppWeb.Plugs.TenantPlug do
  import Plug.Conn

  def init(opts), do: opts

  def call(conn, _opts) do
    # ดึง tenant จาก subdomain หรือ header
    tenant_id = extract_tenant_id(conn)

    case MyApp.Tenants.get_tenant(tenant_id) do
      nil ->
        conn
        |> put_status(:not_found)
        |> Phoenix.Controller.json(%{error: "Tenant not found"})
        |> halt()

      tenant ->
        conn
        |> assign(:current_tenant, tenant)
        |> assign(:tenant_prefix, MyApp.Tenants.tenant_schema_name(tenant.id))
    end
  end

  defp extract_tenant_id(conn) do
    # Option 1: Subdomain
    case conn.host do
      host ->
        host
        |> String.split(".")
        |> List.first()
    end

    # Option 2: Header
    # get_req_header(conn, "x-tenant-id") |> List.first()
  end
end
```

```elixir
# ใช้ prefix ใน queries
defmodule MyApp.TenantRepo do
  @doc "ทำ query ภายใน tenant schema"
  def all(queryable, tenant_id, opts \\ []) do
    prefix = MyApp.Tenants.tenant_schema_name(tenant_id)
    MyApp.Repo.all(queryable, prefix: prefix, opts)
  end

  def get(queryable, id, tenant_id) do
    prefix = MyApp.Tenants.tenant_schema_name(tenant_id)
    MyApp.Repo.get(queryable, id, prefix: prefix)
  end

  def insert(changeset, tenant_id) do
    prefix = MyApp.Tenants.tenant_schema_name(tenant_id)
    MyApp.Repo.insert(changeset, prefix: prefix)
  end
end

# ใน context function
defmodule MyApp.Posts do
  import Ecto.Query
  alias MyApp.{Repo, TenantRepo}
  alias MyApp.Posts.Post

  def list_posts(tenant_id) do
    Post
    |> order_by([p], desc: p.inserted_at)
    |> TenantRepo.all(tenant_id)
  end

  def create_post(attrs, tenant_id) do
    %Post{}
    |> Post.changeset(attrs)
    |> TenantRepo.insert(tenant_id)
  end
end
```

---

## 5. Database Constraints

```elixir
# Migration สำหรับ constraints
defmodule MyApp.Repo.Migrations.AddConstraints do
  use Ecto.Migration

  def change do
    # Check constraint - ตรวจสอบ business rules ระดับ DB
    create constraint("products", :price_must_be_positive,
      check: "price > 0"
    )

    create constraint("orders", :quantity_must_be_positive,
      check: "quantity > 0"
    )

    create constraint("users", :age_must_be_valid,
      check: "age >= 0 AND age <= 150"
    )

    # Exclusion constraint (PostgreSQL) - ป้องกัน overlapping
    create constraint("bookings", :no_overlapping_bookings,
      exclude: ~s(gist (room_id WITH =, daterange(start_date, end_date) WITH &&))
    )
  end
end
```

```elixir
# Handle constraint errors ใน changeset
defmodule MyApp.Shop.Product do
  use Ecto.Schema
  import Ecto.Changeset

  schema "products" do
    field :name, :string
    field :price, :decimal
    field :sku, :string
    timestamps()
  end

  def changeset(product, attrs) do
    product
    |> cast(attrs, [:name, :price, :sku])
    |> validate_required([:name, :price, :sku])
    |> validate_number(:price, greater_than: 0)
    |> unique_constraint(:sku)
    # Handle check constraint error
    |> check_constraint(:price, name: :price_must_be_positive,
        message: "must be greater than 0")
  end
end
```

```elixir
# Foreign key constraints
defmodule MyApp.Orders.Order do
  use Ecto.Schema
  import Ecto.Changeset

  schema "orders" do
    belongs_to :user, MyApp.Accounts.User
    belongs_to :product, MyApp.Shop.Product
    field :quantity, :integer
    field :status, :string
    timestamps()
  end

  def changeset(order, attrs) do
    order
    |> cast(attrs, [:user_id, :product_id, :quantity, :status])
    |> validate_required([:user_id, :product_id, :quantity])
    |> validate_number(:quantity, greater_than: 0)
    # foreign_key_constraint จะดัก error จาก DB
    |> foreign_key_constraint(:user_id)
    |> foreign_key_constraint(:product_id)
    |> check_constraint(:quantity, name: :quantity_must_be_positive)
  end
end
```

---

## 6. Soft Delete Pattern

```elixir
# Migration เพิ่ม deleted_at column
defmodule MyApp.Repo.Migrations.AddSoftDelete do
  use Ecto.Migration

  def change do
    alter table(:posts) do
      add :deleted_at, :utc_datetime
    end

    # Index เพื่อ performance
    create index(:posts, [:deleted_at])
  end
end
```

```elixir
# lib/my_app/soft_delete.ex - Reusable soft delete module
defmodule MyApp.SoftDelete do
  import Ecto.Query
  alias MyApp.Repo

  @doc "Mark record as deleted (soft delete)"
  def soft_delete(record) do
    record
    |> Ecto.Changeset.change(deleted_at: DateTime.utc_now() |> DateTime.truncate(:second))
    |> Repo.update()
  end

  @doc "Restore soft-deleted record"
  def restore(record) do
    record
    |> Ecto.Changeset.change(deleted_at: nil)
    |> Repo.update()
  end

  @doc "Filter out soft-deleted records"
  def not_deleted(query) do
    from(q in query, where: is_nil(q.deleted_at))
  end

  @doc "Get only soft-deleted records"
  def only_deleted(query) do
    from(q in query, where: not is_nil(q.deleted_at))
  end

  @doc "Permanently delete old soft-deleted records"
  def cleanup_deleted(queryable, days_old \\ 30) do
    cutoff = DateTime.add(DateTime.utc_now(), -days_old * 24 * 3600, :second)

    from(q in queryable, where: q.deleted_at < ^cutoff)
    |> Repo.delete_all()
  end
end
```

```elixir
# ใช้ใน schema และ context
defmodule MyApp.Posts.Post do
  use Ecto.Schema

  schema "posts" do
    field :title, :string
    field :body, :string
    field :deleted_at, :utc_datetime
    belongs_to :user, MyApp.Accounts.User
    timestamps()
  end

  def deleted?(post), do: not is_nil(post.deleted_at)
end

defmodule MyApp.Posts do
  import Ecto.Query
  alias MyApp.{Repo, SoftDelete}
  alias MyApp.Posts.Post

  # Default query ที่กรอง deleted records ออก
  def list_posts do
    Post
    |> SoftDelete.not_deleted()
    |> order_by([p], desc: p.inserted_at)
    |> Repo.all()
  end

  def list_deleted_posts do
    Post
    |> SoftDelete.only_deleted()
    |> Repo.all()
  end

  def delete_post(post) do
    SoftDelete.soft_delete(post)
  end

  def restore_post(post) do
    SoftDelete.restore(post)
  end

  def permanent_delete_post(post) do
    Repo.delete(post)
  end
end
```

---

## 7. Upserts และ Bulk Operations

```elixir
defmodule MyApp.Analytics do
  import Ecto.Query
  alias MyApp.{Repo}
  alias MyApp.Analytics.PageView

  @doc "Upsert page view count"
  def track_page_view(page_path, date \\ Date.utc_today()) do
    %PageView{}
    |> PageView.changeset(%{page_path: page_path, date: date, views: 1})
    |> Repo.insert(
      on_conflict: [inc: [views: 1]],
      conflict_target: [:page_path, :date]
    )
  end

  @doc "Bulk insert analytics events"
  def bulk_insert_events(events) do
    now = NaiveDateTime.utc_now() |> NaiveDateTime.truncate(:second)

    events_with_timestamps =
      Enum.map(events, fn event ->
        event
        |> Map.put(:inserted_at, now)
        |> Map.put(:updated_at, now)
      end)

    Repo.insert_all(
      "analytics_events",
      events_with_timestamps,
      on_conflict: :nothing,    # skip duplicates
      conflict_target: :event_id
    )
  end

  @doc "Upsert user settings (create or update)"
  def upsert_user_settings(user_id, settings) do
    %UserSettings{}
    |> UserSettings.changeset(%{user_id: user_id, settings: settings})
    |> Repo.insert(
      on_conflict: {:replace, [:settings, :updated_at]},
      conflict_target: :user_id
    )
  end

  @doc "Bulk update สำหรับ batch processing"
  def bulk_update_status(ids, new_status) do
    from(e in Event, where: e.id in ^ids)
    |> Repo.update_all(set: [status: new_status, updated_at: DateTime.utc_now()])
  end

  @doc "Insert หลาย records พร้อมกัน"
  def bulk_create_notifications(notifications) do
    now = DateTime.utc_now() |> DateTime.truncate(:second)

    entries =
      Enum.map(notifications, fn %{user_id: uid, message: msg} ->
        %{
          user_id: uid,
          message: msg,
          read: false,
          inserted_at: now,
          updated_at: now
        }
      end)

    {count, _} = Repo.insert_all("notifications", entries)
    {:ok, count}
  end
end
```

---

## 8. Query Composition Patterns

```elixir
# lib/my_app/query_helpers.ex
defmodule MyApp.QueryHelpers do
  import Ecto.Query

  @doc "Paginate query"
  def paginate(query, page, per_page \\ 20) do
    offset = (page - 1) * per_page

    query
    |> limit(^per_page)
    |> offset(^offset)
  end

  @doc "Sort query dynamically"
  def sort(query, field, direction \\ :asc)
      when field in [:name, :email, :inserted_at, :updated_at]
      when direction in [:asc, :desc] do
    order_by(query, [{^direction, ^field}])
  end

  def sort(query, _field, _direction), do: query  # ignore invalid fields

  @doc "Filter by date range"
  def in_date_range(query, field, from_date, to_date) do
    query
    |> where([q], field(q, ^field) >= ^from_date)
    |> where([q], field(q, ^field) <= ^to_date)
  end

  @doc "Search across multiple text fields"
  def search(query, fields, term) when is_list(fields) and is_binary(term) do
    pattern = "%#{term}%"

    conditions =
      Enum.reduce(fields, false, fn field, acc ->
        dynamic([q], ilike(field(q, ^field), ^pattern) or ^acc)
      end)

    where(query, ^conditions)
  end
end
```

```elixir
# Composable query functions ใน Context
defmodule MyApp.Posts do
  import Ecto.Query
  alias MyApp.{Repo, QueryHelpers}
  alias MyApp.Posts.Post

  # Base query
  defp base_query do
    from(p in Post, preload: [:user, :tags])
  end

  # Filter functions
  defp filter_by_status(query, nil), do: query
  defp filter_by_status(query, status) do
    where(query, [p], p.status == ^status)
  end

  defp filter_by_user(query, nil), do: query
  defp filter_by_user(query, user_id) do
    where(query, [p], p.user_id == ^user_id)
  end

  defp filter_by_tags(query, []), do: query
  defp filter_by_tags(query, tags) do
    where(query, [p, t], t.name in ^tags)
    |> join(:inner, [p], t in assoc(p, :tags), as: :tags)
    |> distinct(true)
  end

  @doc "Flexible post listing ด้วย options"
  def list_posts(opts \\ []) do
    status = Keyword.get(opts, :status)
    user_id = Keyword.get(opts, :user_id)
    tags = Keyword.get(opts, :tags, [])
    search_term = Keyword.get(opts, :search)
    page = Keyword.get(opts, :page, 1)
    per_page = Keyword.get(opts, :per_page, 20)
    sort_by = Keyword.get(opts, :sort_by, :inserted_at)
    sort_dir = Keyword.get(opts, :sort_dir, :desc)

    base_query()
    |> filter_by_status(status)
    |> filter_by_user(user_id)
    |> filter_by_tags(tags)
    |> then(fn q ->
      if search_term do
        QueryHelpers.search(q, [:title, :body], search_term)
      else
        q
      end
    end)
    |> QueryHelpers.sort(sort_by, sort_dir)
    |> QueryHelpers.paginate(page, per_page)
    |> Repo.all()
  end

  @doc "Count posts ด้วย same filters (สำหรับ pagination)"
  def count_posts(opts \\ []) do
    status = Keyword.get(opts, :status)
    user_id = Keyword.get(opts, :user_id)

    base_query()
    |> filter_by_status(status)
    |> filter_by_user(user_id)
    |> select([p], count(p.id))
    |> Repo.one()
  end
end
```

```elixir
# Named queries สำหรับ reusability
defmodule MyApp.Posts.Query do
  import Ecto.Query
  alias MyApp.Posts.Post

  def published(query \\ Post) do
    where(query, [p], p.status == "published")
  end

  def recent(query \\ Post, days \\ 7) do
    cutoff = DateTime.add(DateTime.utc_now(), -days * 24 * 3600, :second)
    where(query, [p], p.inserted_at >= ^cutoff)
  end

  def with_user(query \\ Post) do
    preload(query, :user)
  end

  def with_comments_count(query \\ Post) do
    query
    |> join(:left, [p], c in assoc(p, :comments), as: :comments)
    |> group_by([p], p.id)
    |> select_merge([p, comments: c], %{comments_count: count(c.id)})
  end

  def top_by_views(query \\ Post, limit \\ 10) do
    query
    |> order_by([p], desc: p.views_count)
    |> limit(^limit)
  end
end

# ใช้ named queries
defmodule MyApp.Posts do
  alias MyApp.{Repo}
  alias MyApp.Posts.Query

  def list_recent_published_posts do
    Query.published()
    |> Query.recent(30)
    |> Query.with_user()
    |> Query.with_comments_count()
    |> Repo.all()
  end

  def trending_posts do
    Query.published()
    |> Query.recent(7)
    |> Query.top_by_views(20)
    |> Repo.all()
  end
end
```

---

## สรุป

```
Ecto Advanced Patterns:
┌────────────────────┬─────────────────────────────────────┐
│ Feature            │ Use Case                            │
├────────────────────┼─────────────────────────────────────┤
│ Ecto.Multi         │ Multiple operations ใน 1 transaction│
│ Custom Types       │ Domain-specific data transformations│
│ Embedded Schemas   │ Structured JSON fields              │
│ Multi-tenancy      │ Schema-per-tenant isolation         │
│ Constraints        │ Business rules enforcement ที่ DB   │
│ Soft Delete        │ Recoverable deletion                │
│ Upserts            │ Insert-or-update patterns           │
│ Query Composition  │ Reusable, composable queries        │
└────────────────────┴─────────────────────────────────────┘

Ecto.Multi Flow:
Multi.new()
  → Multi.insert(:step1, ...)
  → Multi.run(:step2, fn repo, %{step1: result1} -> ... end)
  → Multi.update(:step3, fn %{step2: result2} -> ... end)
  → Repo.transaction()
  → {:ok, %{step1: ..., step2: ..., step3: ...}}
  → {:error, failed_step, reason, changes_so_far}

Query Composition:
BaseQuery
  |> filter_1()
  |> filter_2()
  |> sort()
  |> paginate()
  |> Repo.all()
```

---

*ก่อนหน้า: [Part 43 - Phoenix PubSub and Real-time Patterns](part_43.md) | ต่อไป: [Part 45 - Code Organization and Architecture](part_45.md)*
