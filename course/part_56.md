# Part 56: Multi-tenancy (ระบบหลาย Tenants)

## เป้าหมายการเรียนรู้

หลังจากจบ Part นี้ คุณจะสามารถ:
- สร้างระบบ multi-tenant ด้วย Ecto prefix (schema per tenant)
- Row-level isolation
- Tenant-aware queries
- Tenant provisioning

---

## 1. Schema-based Multi-tenancy (PostgreSQL Schemas)

```elixir
# ทุก tenant มี PostgreSQL schema แยก
# e.g., tenant_acme.users, tenant_acme.orders

defmodule MultiTenant.Tenants.Tenant do
  use Ecto.Schema

  schema "tenants" do
    field :name, :string
    field :slug, :string  # used as schema prefix
    field :status, :string, default: "active"
    field :plan, :string, default: "free"

    timestamps()
  end
end

defmodule MultiTenant.Tenants do
  alias MultiTenant.Repo
  alias MultiTenant.Tenants.Tenant

  def provision_tenant(attrs) do
    Ecto.Multi.new()
    |> Ecto.Multi.insert(:tenant, Tenant.changeset(%Tenant{}, attrs))
    |> Ecto.Multi.run(:create_schema, fn _repo, %{tenant: tenant} ->
      create_tenant_schema(tenant.slug)
    end)
    |> Ecto.Multi.run(:run_migrations, fn _repo, %{tenant: tenant} ->
      run_tenant_migrations(tenant.slug)
    end)
    |> Repo.transaction()
  end

  defp create_tenant_schema(slug) do
    case Repo.query("CREATE SCHEMA IF NOT EXISTS \"tenant_#{slug}\"") do
      {:ok, _} -> {:ok, slug}
      {:error, e} -> {:error, e}
    end
  end

  defp run_tenant_migrations(slug) do
    prefix = "tenant_#{slug}"
    Ecto.Migrator.run(Repo, :up, all: true, prefix: prefix)
    {:ok, :migrated}
  end

  def delete_tenant(tenant) do
    Ecto.Multi.new()
    |> Ecto.Multi.run(:drop_schema, fn _repo, _changes ->
      Repo.query("DROP SCHEMA IF EXISTS \"tenant_#{tenant.slug}\" CASCADE")
    end)
    |> Ecto.Multi.delete(:tenant, tenant)
    |> Repo.transaction()
  end
end
```

---

## 2. Tenant Context (Process Dictionary Pattern)

```elixir
defmodule MultiTenant.TenantContext do
  @tenant_key :current_tenant_prefix

  def set_tenant(prefix) when is_binary(prefix) do
    Process.put(@tenant_key, prefix)
  end

  def get_tenant do
    Process.get(@tenant_key)
  end

  def clear_tenant do
    Process.delete(@tenant_key)
  end

  def with_tenant(prefix, fun) do
    set_tenant(prefix)
    try do
      fun.()
    after
      clear_tenant()
    end
  end
end

# Ecto Repo wrapper ที่ inject prefix อัตโนมัติ
defmodule MultiTenant.TenantRepo do
  alias MultiTenant.Repo
  alias MultiTenant.TenantContext

  def all(queryable, opts \\ []) do
    Repo.all(queryable, prefix_opts(opts))
  end

  def get(queryable, id, opts \\ []) do
    Repo.get(queryable, id, prefix_opts(opts))
  end

  def get!(queryable, id, opts \\ []) do
    Repo.get!(queryable, id, prefix_opts(opts))
  end

  def insert(struct_or_changeset, opts \\ []) do
    Repo.insert(struct_or_changeset, prefix_opts(opts))
  end

  def update(changeset, opts \\ []) do
    Repo.update(changeset, prefix_opts(opts))
  end

  def delete(struct_or_changeset, opts \\ []) do
    Repo.delete(struct_or_changeset, prefix_opts(opts))
  end

  def one(queryable, opts \\ []) do
    Repo.one(queryable, prefix_opts(opts))
  end

  def transaction(fun_or_multi, opts \\ []) do
    Repo.transaction(fun_or_multi, prefix_opts(opts))
  end

  defp prefix_opts(opts) do
    case TenantContext.get_tenant() do
      nil -> opts
      prefix -> Keyword.put_new(opts, :prefix, prefix)
    end
  end
end
```

---

## 3. Tenant Resolution Plug

```elixir
defmodule MultiTenantWeb.Plugs.ResolveTenant do
  import Plug.Conn
  alias MultiTenant.Tenants
  alias MultiTenant.TenantContext

  def init(opts), do: opts

  def call(conn, _opts) do
    case resolve_tenant(conn) do
      {:ok, tenant} ->
        prefix = "tenant_#{tenant.slug}"
        TenantContext.set_tenant(prefix)

        conn
        |> assign(:current_tenant, tenant)
        |> assign(:tenant_prefix, prefix)
        |> register_before_send(fn conn ->
          TenantContext.clear_tenant()
          conn
        end)

      {:error, :not_found} ->
        conn
        |> put_status(404)
        |> Phoenix.Controller.json(%{error: "Tenant not found"})
        |> halt()
    end
  end

  # Resolve จาก subdomain: acme.myapp.com
  defp resolve_tenant(conn) do
    host = conn.host

    cond do
      # Subdomain pattern
      String.contains?(host, ".") ->
        [subdomain | _] = String.split(host, ".")
        Tenants.get_tenant_by_slug(subdomain)

      # Custom domain (ต้องมี tenant_domains table)
      true ->
        Tenants.get_tenant_by_domain(host)
    end
  end
end
```

---

## 4. Row-level Multi-tenancy (ใน Schema เดียว)

```elixir
# ถ้าไม่ใช้ PostgreSQL schema แยก
# ใส่ tenant_id ในทุก table

defmodule MultiTenant.BaseSchema do
  defmacro __using__(_opts) do
    quote do
      use Ecto.Schema
      import Ecto.Changeset

      def base_changeset(struct, attrs) do
        struct
        |> cast(attrs, [:tenant_id])
        |> validate_required([:tenant_id])
      end
    end
  end
end

defmodule MultiTenant.Post do
  use MultiTenant.BaseSchema

  schema "posts" do
    field :title, :string
    field :content, :text
    field :tenant_id, :integer
    timestamps()
  end

  def changeset(post, attrs) do
    post
    |> base_changeset(attrs)
    |> cast(attrs, [:title, :content])
    |> validate_required([:title, :tenant_id])
  end
end

# Query scope สำหรับ tenant
defmodule MultiTenant.ScopedQuery do
  import Ecto.Query

  def for_tenant(queryable, tenant_id) do
    where(queryable, [q], q.tenant_id == ^tenant_id)
  end
end

# ใช้งาน
import MultiTenant.ScopedQuery

posts =
  Post
  |> for_tenant(current_tenant.id)
  |> Repo.all()
```

---

## 5. Tenant-aware LiveView

```elixir
defmodule MultiTenantWeb.PostsLive do
  use MultiTenantWeb, :live_view
  alias MultiTenant.{TenantContext, Blog}

  def mount(_params, %{"tenant_prefix" => prefix}, socket) do
    # Set tenant context ใน LiveView process
    TenantContext.set_tenant(prefix)
    posts = Blog.list_posts()

    {:ok, assign(socket, posts: posts, tenant_prefix: prefix)}
  end

  def handle_event("create_post", %{"title" => title}, socket) do
    # TenantContext ถูก set แล้วจาก mount
    {:ok, post} = Blog.create_post(%{title: title})
    {:noreply, update(socket, :posts, &[post | &1])}
  end
end
```

---

## 6. Cross-tenant Admin Queries

```elixir
defmodule MultiTenant.Admin do
  alias MultiTenant.Repo
  import Ecto.Query

  # Query ข้ามทุก tenants
  def global_stats do
    tenants = Repo.all(MultiTenant.Tenants.Tenant)

    stats = Enum.map(tenants, fn tenant ->
      prefix = "tenant_#{tenant.slug}"
      user_count = Repo.one(from(u in MultiTenant.User, select: count(u.id)), prefix: prefix)
      post_count = Repo.one(from(p in MultiTenant.Post, select: count(p.id)), prefix: prefix)

      %{
        tenant: tenant.name,
        users: user_count || 0,
        posts: post_count || 0
      }
    end)

    %{
      total_tenants: length(tenants),
      tenant_stats: stats
    }
  end

  def migrate_all_tenants do
    tenants = Repo.all(MultiTenant.Tenants.Tenant)

    Enum.each(tenants, fn tenant ->
      prefix = "tenant_#{tenant.slug}"
      Ecto.Migrator.run(Repo, :up, all: true, prefix: prefix)
      IO.puts("Migrated tenant: #{tenant.name}")
    end)
  end
end
```

---

## 7. Tenant Provisioning Flow

```elixir
defmodule MultiTenantWeb.OnboardingController do
  use MultiTenantWeb, :controller

  def create(conn, %{"company" => company_attrs}) do
    slug = Slug.slugify(company_attrs["name"])

    case MultiTenant.Tenants.provision_tenant(%{
      name: company_attrs["name"],
      slug: slug,
      plan: "free"
    }) do
      {:ok, %{tenant: tenant}} ->
        # Create admin user
        {:ok, user} = MultiTenant.TenantRepo.insert(
          MultiTenant.User.changeset(%MultiTenant.User{}, %{
            email: company_attrs["admin_email"],
            role: "admin"
          }),
          prefix: "tenant_#{slug}"
        )

        # Send welcome email
        MultiTenant.Emails.welcome(user, tenant)
        |> MultiTenant.Mailer.deliver()

        conn
        |> put_status(:created)
        |> json(%{
          tenant: %{slug: slug, name: tenant.name},
          redirect_url: "https://#{slug}.myapp.com"
        })

      {:error, :tenant, changeset, _} ->
        conn
        |> put_status(:unprocessable_entity)
        |> json(%{errors: format_errors(changeset)})
    end
  end
end
```

---

## สรุป

```
Multi-tenancy Approaches:
├── Schema per tenant (PostgreSQL schemas) - strongest isolation
├── Row-level (tenant_id column) - simplest
└── Database per tenant - most isolated but expensive

Schema-based Flow:
1. Create tenant record
2. CREATE SCHEMA tenant_{slug}
3. Run migrations on that schema
4. Set prefix in Process dictionary
5. All queries auto-use prefix

Process isolation:
├── TenantContext.set_tenant(prefix)
├── TenantRepo wraps Repo with prefix
└── clear_tenant() after request
```

---

*ก่อนหน้า: [Part 55](part_55.md) | ต่อไป: [Part 57 - Event Sourcing](part_57.md)*
