# Part 45: Code Organization and Architecture (การจัดโครงสร้าง Code)

## เป้าหมายการเรียนรู้

หลังจากจบ Part นี้ คุณจะสามารถ:
- จัดโครงสร้าง Phoenix app ด้วย Contexts
- สร้าง service objects และ pipelines
- Umbrella projects
- Module naming conventions

---

## 1. Phoenix Contexts (Bounded Contexts)

```
lib/my_app/
├── accounts/          # Context: user accounts
│   ├── accounts.ex   # public API
│   ├── user.ex
│   └── credential.ex
├── blog/              # Context: blog
│   ├── blog.ex
│   ├── post.ex
│   └── comment.ex
├── notifications/     # Context: notifications
│   ├── notifications.ex
│   └── email_sender.ex
└── application.ex
```

```elixir
defmodule MyApp.Accounts do
  # Context เป็น public interface ให้ส่วนอื่นเรียกใช้
  # ซ่อน implementation details ไว้ข้างใน

  alias MyApp.Accounts.{User, Credential}
  alias MyApp.Repo

  # Public API
  def get_user(id), do: Repo.get(User, id)
  def get_user!(id), do: Repo.get!(User, id)
  def get_user_by_email(email), do: Repo.get_by(User, email: email)

  def create_user(attrs) do
    %User{}
    |> User.changeset(attrs)
    |> Repo.insert()
  end

  def authenticate(email, password) do
    user = get_user_by_email(email)
    Credential.verify_password(user, password)
  end
end
```

---

## 2. Service Objects

```elixir
# Service แยกออกมาเมื่อ logic ซับซ้อน
defmodule MyApp.Accounts.RegisterUser do
  alias MyApp.{Accounts, Repo}
  alias MyApp.Notifications

  def call(attrs) do
    with {:ok, user} <- Accounts.create_user(attrs),
         {:ok, _} <- create_profile(user),
         {:ok, token} <- generate_confirmation_token(user),
         {:ok, _} <- Notifications.send_confirmation_email(user, token) do
      {:ok, user}
    end
  end

  defp create_profile(user) do
    %MyApp.Profiles.Profile{}
    |> MyApp.Profiles.Profile.changeset(%{user_id: user.id})
    |> Repo.insert()
  end

  defp generate_confirmation_token(user) do
    {:ok, Phoenix.Token.sign(MyAppWeb.Endpoint, "confirm", user.id)}
  end
end

# ใช้งาน
case MyApp.Accounts.RegisterUser.call(params) do
  {:ok, user} -> redirect(conn, to: "/welcome")
  {:error, changeset} -> render(conn, :new, changeset: changeset)
end
```

---

## 3. Pipeline Pattern

```elixir
defmodule MyApp.Pipeline do
  def pipe(initial, steps) do
    Enum.reduce_while(steps, {:ok, initial}, fn step, {:ok, value} ->
      case step.(value) do
        {:ok, result} -> {:cont, {:ok, result}}
        {:error, _} = error -> {:halt, error}
      end
    end)
  end
end

defmodule MyApp.Orders.PlaceOrder do
  alias MyApp.Pipeline

  def call(order_params, user) do
    Pipeline.pipe(order_params, [
      &validate_order/1,
      fn params -> check_inventory(params) end,
      fn params -> calculate_shipping(params, user) end,
      fn params -> apply_discounts(params, user) end,
      fn params -> create_order(params, user) end,
      fn order -> charge_payment(order) end,
      fn order -> send_confirmation(order, user) end,
    ])
  end

  defp validate_order(params) do
    changeset = Order.changeset(%Order{}, params)
    if changeset.valid?, do: {:ok, params}, else: {:error, changeset}
  end

  defp check_inventory(params) do
    case MyApp.Inventory.reserve(params.items) do
      {:ok, reserved} -> {:ok, Map.put(params, :reserved, reserved)}
      {:error, out_of_stock} -> {:error, {:out_of_stock, out_of_stock}}
    end
  end

  # ... other steps
end
```

---

## 4. Umbrella Projects

```bash
# สร้าง Umbrella project
mix new my_platform --umbrella
cd my_platform

# เพิ่ม apps
cd apps
mix phx.new web --no-install
mix new core --no-install
mix new workers --no-install
```

```
my_platform/
├── apps/
│   ├── core/              # Business logic, schemas
│   │   ├── lib/core/
│   │   └── mix.exs
│   ├── web/               # Phoenix app
│   │   ├── lib/web/
│   │   └── mix.exs
│   └── workers/           # Background jobs
│       ├── lib/workers/
│       └── mix.exs
├── mix.exs
└── config/
```

```elixir
# apps/web/mix.exs
defp deps do
  [
    {:core, in_umbrella: true},
    {:workers, in_umbrella: true},
    {:phoenix, "~> 1.7"},
  ]
end

# apps/core/lib/core/accounts.ex
defmodule Core.Accounts do
  def create_user(attrs), do: # ...
end

# apps/web/lib/web/controllers/user_controller.ex
defmodule Web.UserController do
  use Web, :controller
  alias Core.Accounts  # cross-app reference

  def create(conn, params) do
    case Accounts.create_user(params) do
      {:ok, user} -> json(conn, user)
      {:error, cs} -> json(conn, %{errors: cs})
    end
  end
end
```

---

## 5. Module Naming Conventions

```
# Good naming:
MyApp.Accounts           # Context
MyApp.Accounts.User      # Schema
MyApp.Accounts.UserQuery # Query helpers
MyApp.Blog.PostPolicy    # Authorization policy
MyApp.Blog.PostFormatter # Formatting logic

MyAppWeb.UserController   # Controller
MyAppWeb.UserJSON         # JSON renderer
MyAppWeb.UserLive         # LiveView
MyAppWeb.UserLive.Index   # Nested LiveView

# Bad naming:
MyApp.UserService         # ไม่ได้บอกว่าทำอะไร
MyApp.Helper              # too generic
MyApp.Utils               # catch-all ไม่ดี
```

---

## 6. Protocol for Polymorphism

```elixir
defprotocol MyApp.Notifiable do
  def notify(entity, message)
  def contact_info(entity)
end

defimpl MyApp.Notifiable, for: MyApp.Accounts.User do
  def notify(user, message) do
    MyApp.Email.send(user.email, message)
  end

  def contact_info(user), do: user.email
end

defimpl MyApp.Notifiable, for: MyApp.Accounts.Team do
  def notify(team, message) do
    Enum.each(team.members, &MyApp.Email.send(&1.email, message))
  end

  def contact_info(team), do: "team:#{team.name}"
end

# ใช้งาน - polymorphic, ไม่ต้อง if/else
def send_notification(entity, message) do
  MyApp.Notifiable.notify(entity, message)
end
```

---

## 7. Behaviour for Abstraction

```elixir
defmodule MyApp.PaymentGateway do
  @callback charge(amount :: integer, token :: String.t()) ::
    {:ok, String.t()} | {:error, String.t()}

  @callback refund(transaction_id :: String.t(), amount :: integer) ::
    {:ok, map()} | {:error, String.t()}
end

defmodule MyApp.Gateways.Stripe do
  @behaviour MyApp.PaymentGateway

  def charge(amount, token) do
    # Stripe API call
    {:ok, "stripe_txn_123"}
  end

  def refund(transaction_id, amount) do
    {:ok, %{refund_id: "re_123"}}
  end
end

defmodule MyApp.Gateways.Omise do
  @behaviour MyApp.PaymentGateway

  def charge(amount, token), do: {:ok, "omise_chrg_123"}
  def refund(transaction_id, amount), do: {:ok, %{refund_id: "omise_ref_123"}}
end

# Config-driven gateway selection
defmodule MyApp.Payment do
  def gateway do
    Application.get_env(:my_app, :payment_gateway, MyApp.Gateways.Stripe)
  end

  def charge(amount, token) do
    gateway().charge(amount, token)
  end
end
```

---

## สรุป

```
Code Organization:
├── Contexts: bounded domains with public APIs
├── Services: complex multi-step operations
├── Pipelines: composable step-by-step processing
└── Umbrella: separate apps in one monorepo

Abstraction:
├── Protocols: polymorphism on data types
├── Behaviours: define interface, swap implementations
└── Dependency injection via config

Naming:
├── Context.Schema - data model
├── Context.Function - public operations
└── Web.Controller/LiveView - HTTP layer
```

---

*ก่อนหน้า: [Part 44](part_44.md) | ต่อไป: [Part 46 - LiveView Advanced](part_46.md)*
