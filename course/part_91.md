# Part 91: Umbrella Projects (โปรเจกต์แบบ Umbrella)

## เป้าหมายการเรียนรู้

หลังจากจบ Part นี้ คุณจะสามารถ:
- สร้างและจัดการ Umbrella project
- แบ่งโค้ดเป็น apps ย่อย
- การ share code ระหว่าง apps
- Dependency management ใน umbrella

---

## 1. สร้าง Umbrella Project

```bash
# สร้าง umbrella project ใหม่
mix new my_platform --umbrella

# โครงสร้าง:
# my_platform/
#   apps/
#   config/
#     config.exs
#     dev.exs
#     prod.exs
#   mix.exs

# สร้าง apps ภายใน umbrella
cd my_platform
mix new apps/core
mix new apps/web --sup
mix phx.new apps/web --no-install
mix new apps/workers --sup
```

---

## 2. โครงสร้าง Umbrella

```
my_platform/
├── apps/
│   ├── core/           # Business logic, schemas
│   │   ├── lib/
│   │   │   └── core/
│   │   │       ├── accounts/
│   │   │       ├── orders/
│   │   │       └── repo.ex
│   │   └── mix.exs
│   ├── web/            # Phoenix web layer
│   │   ├── lib/
│   │   │   └── web/
│   │   │       ├── controllers/
│   │   │       ├── live/
│   │   │       └── endpoint.ex
│   │   └── mix.exs
│   └── workers/        # Background jobs
│       ├── lib/
│       │   └── workers/
│       │       ├── email_worker.ex
│       │       └── report_worker.ex
│       └── mix.exs
├── config/
└── mix.exs
```

---

## 3. Dependencies ระหว่าง Apps

```elixir
# apps/web/mix.exs
defp deps do
  [
    {:phoenix, "~> 1.7"},
    {:core, in_umbrella: true},  # depend on sibling app
    {:workers, in_umbrella: true}
  ]
end

# apps/workers/mix.exs
defp deps do
  [
    {:oban, "~> 2.17"},
    {:core, in_umbrella: true}  # workers use core schemas
  ]
end

# apps/core/mix.exs
defp deps do
  [
    {:ecto_sql, "~> 3.11"},
    {:postgrex, "~> 0.17"}
    # core has no umbrella deps - lowest layer
  ]
end
```

---

## 4. Shared Config

```elixir
# config/config.exs (root umbrella config)
import Config

# Config shared across all apps
config :core, Core.Repo,
  database: "my_platform_dev",
  username: "postgres",
  hostname: "localhost"

# Import per-app configs
import_config "../apps/core/config/config.exs"
import_config "../apps/web/config/config.exs"

# config/dev.exs
import Config
config :core, Core.Repo, pool_size: 10

# apps/core/config/config.exs (app-specific)
import Config
config :core, ecto_repos: [Core.Repo]
```

---

## 5. Core App - Shared Business Logic

```elixir
# apps/core/lib/core/accounts.ex
defmodule Core.Accounts do
  alias Core.Repo
  alias Core.Accounts.User

  def get_user(id), do: Repo.get(User, id)
  def get_user!(id), do: Repo.get!(User, id)

  def create_user(attrs) do
    %User{}
    |> User.changeset(attrs)
    |> Repo.insert()
  end

  def authenticate_user(email, password) do
    user = Repo.get_by(User, email: email)
    User.check_password(user, password)
  end
end

# apps/core/lib/core/accounts/user.ex
defmodule Core.Accounts.User do
  use Ecto.Schema
  import Ecto.Changeset

  schema "users" do
    field :email, :string
    field :name, :string
    field :password_hash, :string
    field :password, :string, virtual: true
    timestamps()
  end

  def changeset(user, attrs) do
    user
    |> cast(attrs, [:email, :name, :password])
    |> validate_required([:email, :password])
    |> validate_format(:email, ~r/@/)
    |> unique_constraint(:email)
    |> hash_password()
  end

  defp hash_password(%{valid?: false} = changeset), do: changeset
  defp hash_password(changeset) do
    password = get_change(changeset, :password)
    put_change(changeset, :password_hash, Argon2.hash_pwd_salt(password))
  end

  def check_password(nil, _), do: {:error, :not_found}
  def check_password(user, password) do
    if Argon2.verify_pass(password, user.password_hash) do
      {:ok, user}
    else
      {:error, :invalid_password}
    end
  end
end
```

---

## 6. Web App ใช้งาน Core

```elixir
# apps/web/lib/web/controllers/session_controller.ex
defmodule Web.SessionController do
  use Web, :controller

  # ใช้ Core.Accounts จาก apps/core
  alias Core.Accounts

  def create(conn, %{"email" => email, "password" => password}) do
    case Accounts.authenticate_user(email, password) do
      {:ok, user} ->
        conn
        |> put_session(:user_id, user.id)
        |> redirect(to: ~p"/dashboard")

      {:error, _} ->
        conn
        |> put_flash(:error, "Invalid credentials")
        |> render(:new)
    end
  end
end
```

---

## 7. Running the Umbrella

```bash
# Run all apps together
mix phx.server                   # from root

# Run tests for all apps
mix test

# Run tests for specific app
mix test apps/core/

# Start IEx with all apps
iex -S mix

# Create migration in core
cd apps/core && mix ecto.gen.migration create_users

# Run migration from root
mix ecto.migrate
```

---

## สรุป

```
Umbrella Project Structure:
├── core: schemas, business logic, repo
├── web: Phoenix, controllers, LiveView
└── workers: Oban jobs, background tasks

Benefits:
├── Clear separation of concerns
├── Independent compilation
├── Shared dependencies via core
└── Team parallel development

When to use:
├── Multiple interfaces (web + API + CLI)
├── Complex domain with many bounded contexts
└── Large team with specialized areas

When NOT to use:
└── Small apps - overkill overhead
```

---

*ก่อนหน้า: [Part 90](part_90.md) | ต่อไป: [Part 92 - Distributed Elixir](part_92.md)*
