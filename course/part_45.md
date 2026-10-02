# Part 45: Code Organization and Architecture

## เป้าหมายการเรียนรู้

- เข้าใจ Context pattern ใน Phoenix และ bounded contexts
- ใช้ Service objects และ Command pattern
- ประยุกต์ใช้ Domain-driven design กับ Elixir
- สร้าง umbrella projects สำหรับ large applications
- จัดระเบียบ modules อย่างมีหลักการ
- สร้าง dependency injection patterns ใน Elixir
- เข้าใจ event sourcing basics
- เข้าใจ CQRS pattern

---

## 1. Context Pattern ใน Phoenix

Contexts เป็น bounded boundaries ที่แยก business domain ออกจากกัน

```elixir
# โครงสร้างที่ดีสำหรับ Phoenix application
lib/
├── my_app/
│   ├── accounts/          # Accounts context
│   │   ├── user.ex
│   │   ├── profile.ex
│   │   └── user_token.ex
│   ├── accounts.ex        # Public API ของ Accounts context
│   ├── catalog/           # Catalog context
│   │   ├── product.ex
│   │   ├── category.ex
│   │   └── tag.ex
│   ├── catalog.ex
│   ├── orders/            # Orders context
│   │   ├── order.ex
│   │   ├── order_item.ex
│   │   └── cart.ex
│   └── orders.ex
└── my_app_web/
    ├── controllers/
    ├── live/
    └── components/
```

```elixir
# lib/my_app/accounts.ex - Public API ของ Accounts context
defmodule MyApp.Accounts do
  @moduledoc """
  The Accounts context จัดการทุกอย่างเกี่ยวกับ users
  นี่คือ public interface - code ภายนอกควรใช้ functions นี้เท่านั้น
  """

  alias MyApp.Repo
  alias MyApp.Accounts.{User, Profile, UserToken}

  # === User Management ===

  @doc "ดึง user ด้วย id"
  def get_user!(id), do: Repo.get!(User, id)

  @doc "ดึง user ด้วย email"
  def get_user_by_email(email) when is_binary(email) do
    Repo.get_by(User, email: String.downcase(email))
  end

  @doc "สร้าง user ใหม่"
  def register_user(attrs) do
    %User{}
    |> User.registration_changeset(attrs)
    |> Repo.insert()
  end

  @doc "อัปเดต user profile"
  def update_user_profile(%User{} = user, attrs) do
    user
    |> User.profile_changeset(attrs)
    |> Repo.update()
  end

  # === Authentication ===

  @doc "ตรวจสอบ credentials"
  def authenticate_user(email, password) do
    user = get_user_by_email(email)
    User.check_password(user, password)
  end

  @doc "สร้าง session token"
  def generate_user_session_token(user) do
    {token, user_token} = UserToken.build_session_token(user)
    Repo.insert!(user_token)
    token
  end

  # === Boundary: อย่าให้ code ภายนอก access User schema โดยตรง ===
  # ผ่าน function เหล่านี้เท่านั้น
end
```

```elixir
# Context ไม่ควร cross boundaries
# BAD: Orders context access User struct โดยตรง
defmodule MyApp.Orders do
  alias MyApp.Accounts.User  # Cross-context dependency!

  def create_order(%User{id: user_id}, items) do
    # ...
  end
end

# GOOD: ใช้แค่ user_id ที่เป็น primitive value
defmodule MyApp.Orders do
  def create_order(user_id, items) when is_integer(user_id) do
    # ...
  end

  # หรือถ้าต้องการข้อมูล user เพิ่ม ใช้ public API
  def create_order_for_user(user_id, items) do
    user = MyApp.Accounts.get_user!(user_id)
    # ใช้เฉพาะ fields ที่จำเป็น
    create_order(%{user_id: user.id, email: user.email}, items)
  end
end
```

---

## 2. Service Objects / Command Pattern

Service objects แยก complex business logic ออกจาก context

```elixir
# lib/my_app/accounts/commands/register_user.ex
defmodule MyApp.Accounts.Commands.RegisterUser do
  @moduledoc """
  Command สำหรับ register user ใหม่พร้อม dependencies ทั้งหมด
  """

  alias Ecto.Multi
  alias MyApp.{Repo, Mailer}
  alias MyApp.Accounts.{User, Profile}

  defstruct [:attrs]

  @type t :: %__MODULE__{attrs: map()}

  @doc "Execute register user command"
  def execute(%__MODULE__{attrs: attrs}) do
    Multi.new()
    |> Multi.insert(:user, User.registration_changeset(%User{}, attrs))
    |> Multi.insert(:profile, &build_profile/2)
    |> Multi.run(:send_email, &send_welcome_email/2)
    |> Repo.transaction()
    |> handle_result()
  end

  defp build_profile(_repo, %{user: user}) do
    Profile.changeset(%Profile{}, %{
      user_id: user.id,
      display_name: user.name
    })
  end

  defp send_welcome_email(_repo, %{user: user}) do
    case Mailer.send_welcome_email(user) do
      {:ok, _} -> {:ok, :sent}
      error -> error
    end
  end

  defp handle_result({:ok, %{user: user}}), do: {:ok, user}
  defp handle_result({:error, :user, changeset, _}), do: {:error, changeset}
  defp handle_result({:error, step, reason, _}), do: {:error, {step, reason}}
end

# ใช้ใน context
defmodule MyApp.Accounts do
  alias MyApp.Accounts.Commands.RegisterUser

  def register_user(attrs) do
    %RegisterUser{attrs: attrs}
    |> RegisterUser.execute()
  end
end
```

```elixir
# lib/my_app/orders/commands/place_order.ex
defmodule MyApp.Orders.Commands.PlaceOrder do
  @moduledoc "Command สำหรับ place order"

  alias Ecto.Multi
  alias MyApp.{Repo}
  alias MyApp.Orders.{Order, OrderItem}
  alias MyApp.Inventory
  alias MyApp.Payments

  defstruct [:user_id, :cart_items, :payment_method, :shipping_address]

  def execute(%__MODULE__{} = cmd) do
    with :ok <- validate_command(cmd),
         {:ok, result} <- run_transaction(cmd) do
      {:ok, result.order}
    end
  end

  defp validate_command(%{cart_items: []}), do: {:error, :empty_cart}
  defp validate_command(%{payment_method: nil}), do: {:error, :no_payment_method}
  defp validate_command(_), do: :ok

  defp run_transaction(cmd) do
    Multi.new()
    |> Multi.run(:check_inventory, fn _repo, _ ->
      Inventory.check_availability(cmd.cart_items)
    end)
    |> Multi.insert(:order, fn _ ->
      Order.changeset(%Order{}, %{
        user_id: cmd.user_id,
        status: "pending",
        shipping_address: cmd.shipping_address
      })
    end)
    |> Multi.insert_all(:order_items, OrderItem, fn %{order: order} ->
      build_order_items(order.id, cmd.cart_items)
    end)
    |> Multi.run(:reserve_inventory, fn _repo, %{order_items: {_, items}} ->
      Inventory.reserve(items)
    end)
    |> Multi.run(:process_payment, fn _repo, %{order: order} ->
      Payments.charge(cmd.payment_method, order.total)
    end)
    |> Multi.update(:confirm_order, fn %{order: order} ->
      Order.changeset(order, %{status: "confirmed"})
    end)
    |> Repo.transaction()
  end

  defp build_order_items(order_id, cart_items) do
    Enum.map(cart_items, fn item ->
      %{
        order_id: order_id,
        product_id: item.product_id,
        quantity: item.quantity,
        price: item.price,
        inserted_at: DateTime.utc_now(),
        updated_at: DateTime.utc_now()
      }
    end)
  end
end
```

---

## 3. Domain-driven Design กับ Elixir

```elixir
# Value Objects - immutable, equality by value
defmodule MyApp.Domain.Money do
  @enforce_keys [:amount, :currency]
  defstruct [:amount, :currency]

  def new(amount, currency \\ "THB") when is_number(amount) and amount >= 0 do
    %__MODULE__{amount: Decimal.new("#{amount}"), currency: currency}
  end

  def add(%__MODULE__{currency: c} = a, %__MODULE__{currency: c} = b) do
    %__MODULE__{amount: Decimal.add(a.amount, b.amount), currency: c}
  end

  def add(_, _), do: {:error, :currency_mismatch}

  def multiply(%__MODULE__{} = money, factor) when is_number(factor) do
    %__MODULE__{money | amount: Decimal.mult(money.amount, Decimal.new("#{factor}"))}
  end

  def equal?(%__MODULE__{} = a, %__MODULE__{} = b) do
    a.currency == b.currency && Decimal.equal?(a.amount, b.amount)
  end

  def to_string(%__MODULE__{amount: amount, currency: currency}) do
    "#{currency} #{Decimal.to_string(amount)}"
  end
end
```

```elixir
# Entities - has identity (id), mutable state
defmodule MyApp.Domain.Order do
  @enforce_keys [:id, :user_id, :status]
  defstruct [:id, :user_id, :status, :items, :total, :placed_at]

  # Domain events ที่เกิดขึ้นกับ Order
  def place(user_id, items) do
    order = %__MODULE__{
      id: Ecto.UUID.generate(),
      user_id: user_id,
      status: :pending,
      items: items,
      total: calculate_total(items),
      placed_at: DateTime.utc_now()
    }

    {order, [{:order_placed, %{order_id: order.id, user_id: user_id}}]}
  end

  def confirm(%__MODULE__{status: :pending} = order) do
    updated = %{order | status: :confirmed}
    {updated, [{:order_confirmed, %{order_id: order.id}}]}
  end

  def confirm(%__MODULE__{status: status}) do
    {:error, {:invalid_status_transition, status, :confirmed}}
  end

  def cancel(%__MODULE__{status: status} = order) when status in [:pending, :confirmed] do
    updated = %{order | status: :cancelled}
    {updated, [{:order_cancelled, %{order_id: order.id}}]}
  end

  def cancel(%__MODULE__{status: status}) do
    {:error, {:cannot_cancel, status}}
  end

  defp calculate_total(items) do
    items
    |> Enum.map(fn item -> MyApp.Domain.Money.multiply(item.price, item.quantity) end)
    |> Enum.reduce(MyApp.Domain.Money.new(0), &MyApp.Domain.Money.add/2)
  end
end
```

```elixir
# Aggregates - Root entity ที่ control access
defmodule MyApp.Domain.Cart do
  @enforce_keys [:user_id]
  defstruct [:user_id, items: [], coupon: nil]

  def new(user_id), do: %__MODULE__{user_id: user_id}

  def add_item(%__MODULE__{} = cart, product_id, quantity, price) do
    item = %{product_id: product_id, quantity: quantity, price: price}

    case find_item(cart, product_id) do
      nil ->
        %{cart | items: [item | cart.items]}

      existing ->
        updated_items =
          Enum.map(cart.items, fn
            %{product_id: ^product_id} ->
              %{existing | quantity: existing.quantity + quantity}
            other ->
              other
          end)

        %{cart | items: updated_items}
    end
  end

  def remove_item(%__MODULE__{} = cart, product_id) do
    %{cart | items: Enum.reject(cart.items, &(&1.product_id == product_id))}
  end

  def apply_coupon(%__MODULE__{} = cart, coupon) do
    if valid_coupon?(coupon) do
      {:ok, %{cart | coupon: coupon}}
    else
      {:error, :invalid_coupon}
    end
  end

  def checkout(%__MODULE__{items: []}), do: {:error, :empty_cart}
  def checkout(%__MODULE__{} = cart) do
    MyApp.Domain.Order.place(cart.user_id, cart.items)
  end

  defp find_item(cart, product_id) do
    Enum.find(cart.items, &(&1.product_id == product_id))
  end

  defp valid_coupon?(%{active: true}), do: true
  defp valid_coupon?(_), do: false
end
```

---

## 4. Umbrella Projects

```bash
# สร้าง umbrella project
mix new my_platform --umbrella

# โครงสร้างที่ได้
my_platform/
├── apps/
│   ├── # เพิ่ม apps ที่นี่
├── mix.exs     # umbrella mix.exs
└── config/

# สร้าง apps ภายใน umbrella
cd my_platform
mix new apps/core             # Core business logic
mix new apps/web --sup        # Phoenix web app
mix new apps/worker --sup     # Background jobs
mix phx.new apps/api --no-html --no-assets  # API app
```

```elixir
# apps/core/mix.exs - Core app (ไม่มี Phoenix)
defmodule Core.MixProject do
  use Mix.Project

  def project do
    [
      app: :core,
      version: "0.1.0",
      build_path: "../../_build",
      config_path: "../../config/config.exs",
      deps_path: "../../deps",
      lockfile: "../../mix.lock",
      elixir: "~> 1.15",
      start_permanent: Mix.env() == :prod,
      deps: deps()
    ]
  end

  defp deps do
    [
      {:ecto_sql, "~> 3.10"},
      {:postgrex, ">= 0.0.0"}
    ]
  end
end
```

```elixir
# apps/web/mix.exs - Web app depend on core
defmodule Web.MixProject do
  use Mix.Project

  def project do
    [
      app: :web,
      version: "0.1.0",
      # ...
      deps: deps()
    ]
  end

  defp deps do
    [
      {:phoenix, "~> 1.7"},
      {:core, in_umbrella: true}  # depend on core app
    ]
  end
end
```

```elixir
# apps/worker/mix.exs - Background worker
defmodule Worker.MixProject do
  use Mix.Project

  defp deps do
    [
      {:oban, "~> 2.17"},
      {:core, in_umbrella: true}
    ]
  end
end
```

```elixir
# apps/core/lib/core/accounts.ex - Shared business logic
defmodule Core.Accounts do
  # ทุก app ใน umbrella ใช้ Core ร่วมกัน
  def register_user(attrs), do: # ...
  def authenticate(email, password), do: # ...
end

# apps/web/lib/web/controllers/user_controller.ex
defmodule Web.UserController do
  use Web, :controller
  alias Core.Accounts  # ใช้ Core.Accounts

  def create(conn, %{"user" => params}) do
    case Accounts.register_user(params) do
      {:ok, user} -> redirect(conn, to: ~p"/dashboard")
      {:error, changeset} -> render(conn, :new, changeset: changeset)
    end
  end
end
```

---

## 5. Module Organization Best Practices

```elixir
# lib/my_app/orders/order.ex - Well-organized schema module
defmodule MyApp.Orders.Order do
  @moduledoc """
  Schema สำหรับ Order

  An order represents a customer's purchase. Orders go through the following
  lifecycle: pending -> confirmed -> shipped -> delivered (or cancelled)
  """

  use Ecto.Schema
  import Ecto.Changeset

  # Constants อยู่ด้านบน
  @valid_statuses ~w(pending confirmed shipped delivered cancelled)
  @cancellable_statuses ~w(pending confirmed)

  schema "orders" do
    field :status, :string, default: "pending"
    field :total, :decimal
    field :notes, :string

    belongs_to :user, MyApp.Accounts.User
    has_many :items, MyApp.Orders.OrderItem

    timestamps()
  end

  # Public interface - changesets
  def creation_changeset(order, attrs) do
    order
    |> cast(attrs, [:user_id, :notes])
    |> validate_required([:user_id])
    |> foreign_key_constraint(:user_id)
    |> put_change(:status, "pending")
  end

  def status_changeset(order, status) when status in @valid_statuses do
    order
    |> change(status: status)
    |> validate_status_transition(order.status, status)
  end

  # Business logic queries (class methods pattern)
  def cancellable?(%__MODULE__{status: status}) do
    status in @cancellable_statuses
  end

  # Private helpers
  defp validate_status_transition(changeset, from, to) do
    if valid_transition?(from, to) do
      changeset
    else
      add_error(changeset, :status, "cannot transition from #{from} to #{to}")
    end
  end

  defp valid_transition?("pending", "confirmed"), do: true
  defp valid_transition?("confirmed", "shipped"), do: true
  defp valid_transition?("shipped", "delivered"), do: true
  defp valid_transition?(from, "cancelled") when from in @cancellable_statuses, do: true
  defp valid_transition?(_, _), do: false
end
```

---

## 6. Dependency Injection Patterns

```elixir
# Behaviour-based dependency injection
defmodule MyApp.Notifications.Sender do
  @moduledoc "Behaviour สำหรับ notification senders"

  @callback send(recipient :: String.t(), message :: map()) ::
              {:ok, any()} | {:error, any()}
end

defmodule MyApp.Notifications.EmailSender do
  @behaviour MyApp.Notifications.Sender

  def send(email, message) do
    MyApp.Mailer.deliver(%{to: email, subject: message.subject, body: message.body})
  end
end

defmodule MyApp.Notifications.SmsSender do
  @behaviour MyApp.Notifications.Sender

  def send(phone, message) do
    MyApp.SMS.send(%{to: phone, body: message.body})
  end
end

# Null object สำหรับ testing
defmodule MyApp.Notifications.NullSender do
  @behaviour MyApp.Notifications.Sender

  def send(_recipient, _message), do: {:ok, :sent}
end
```

```elixir
# ใช้ Application env เพื่อ inject dependency
# config/config.exs
config :my_app, :notification_sender, MyApp.Notifications.EmailSender

# config/test.exs
config :my_app, :notification_sender, MyApp.Notifications.NullSender

# lib/my_app/notifications.ex
defmodule MyApp.Notifications do
  defp sender do
    Application.get_env(:my_app, :notification_sender)
  end

  def notify_user(user, message) do
    sender().send(user.email, message)
  end
end
```

```elixir
# Mox สำหรับ mock ใน tests
# mix.exs
defp deps do
  [
    {:mox, "~> 1.1", only: :test}
  ]
end

# test/support/mocks.ex
Mox.defmock(MyApp.NotificationSenderMock,
  for: MyApp.Notifications.Sender)

# config/test.exs
config :my_app, :notification_sender, MyApp.NotificationSenderMock

# test/my_app/notifications_test.exs
defmodule MyApp.NotificationsTest do
  use ExUnit.Case, async: true
  import Mox

  setup :verify_on_exit!

  test "sends notification to user" do
    expect(MyApp.NotificationSenderMock, :send, fn email, message ->
      assert email == "user@example.com"
      assert message.subject =~ "Welcome"
      {:ok, :sent}
    end)

    user = %{email: "user@example.com"}
    assert :ok = MyApp.Notifications.notify_user(user, %{subject: "Welcome", body: "..."})
  end
end
```

---

## 7. Event Sourcing Basics

```elixir
# lib/my_app/event_store.ex
defmodule MyApp.EventStore do
  @moduledoc """
  Simple event store สำหรับ event sourcing
  """

  use Ecto.Schema
  import Ecto.Query
  alias MyApp.{Repo}

  schema "events" do
    field :aggregate_id, :string
    field :aggregate_type, :string
    field :event_type, :string
    field :event_data, :map
    field :metadata, :map
    field :version, :integer
    field :occurred_at, :utc_datetime

    timestamps(updated_at: false)
  end

  def append(aggregate_id, aggregate_type, events) when is_list(events) do
    current_version = get_current_version(aggregate_id)

    event_records =
      events
      |> Enum.with_index(current_version + 1)
      |> Enum.map(fn {event, version} ->
        %{
          aggregate_id: aggregate_id,
          aggregate_type: aggregate_type,
          event_type: event.__struct__ |> Module.split() |> List.last(),
          event_data: Map.from_struct(event),
          version: version,
          occurred_at: DateTime.utc_now(),
          inserted_at: DateTime.utc_now()
        }
      end)

    Repo.insert_all(__MODULE__, event_records)
  end

  def load(aggregate_id, from_version \\ 0) do
    from(e in __MODULE__,
      where: e.aggregate_id == ^aggregate_id and e.version > ^from_version,
      order_by: e.version
    )
    |> Repo.all()
    |> Enum.map(&deserialize_event/1)
  end

  defp get_current_version(aggregate_id) do
    from(e in __MODULE__,
      where: e.aggregate_id == ^aggregate_id,
      select: max(e.version)
    )
    |> Repo.one() || 0
  end

  defp deserialize_event(%{event_type: type, event_data: data}) do
    module = Module.concat(MyApp.Events, type)
    struct(module, Enum.map(data, fn {k, v} -> {String.to_atom(k), v} end))
  end
end
```

```elixir
# Events
defmodule MyApp.Events.OrderPlaced do
  defstruct [:order_id, :user_id, :items, :total]
end

defmodule MyApp.Events.OrderConfirmed do
  defstruct [:order_id, :confirmed_at]
end

defmodule MyApp.Events.OrderCancelled do
  defstruct [:order_id, :reason, :cancelled_at]
end

# Aggregate ที่ rebuild จาก events
defmodule MyApp.Aggregates.Order do
  defstruct [:id, :user_id, :status, :items, :total]

  def rebuild(order_id) do
    events = MyApp.EventStore.load(order_id)
    Enum.reduce(events, nil, &apply_event/2)
  end

  defp apply_event(%MyApp.Events.OrderPlaced{} = event, nil) do
    %__MODULE__{
      id: event.order_id,
      user_id: event.user_id,
      status: :pending,
      items: event.items,
      total: event.total
    }
  end

  defp apply_event(%MyApp.Events.OrderConfirmed{}, order) do
    %{order | status: :confirmed}
  end

  defp apply_event(%MyApp.Events.OrderCancelled{}, order) do
    %{order | status: :cancelled}
  end
end
```

---

## 8. CQRS Pattern

```elixir
# Command side - Write operations
defmodule MyApp.Commands do
  defmodule PlaceOrder do
    defstruct [:user_id, :items, :payment_method]
  end

  defmodule ConfirmOrder do
    defstruct [:order_id, :confirmed_by]
  end
end

defmodule MyApp.CommandHandler do
  alias MyApp.{EventStore, Aggregates}
  alias MyApp.Events

  def handle(%MyApp.Commands.PlaceOrder{} = cmd) do
    order_id = Ecto.UUID.generate()
    event = %Events.OrderPlaced{
      order_id: order_id,
      user_id: cmd.user_id,
      items: cmd.items,
      total: calculate_total(cmd.items)
    }

    EventStore.append(order_id, "Order", [event])
    {:ok, order_id}
  end

  def handle(%MyApp.Commands.ConfirmOrder{} = cmd) do
    order = Aggregates.Order.rebuild(cmd.order_id)

    case order.status do
      :pending ->
        event = %Events.OrderConfirmed{
          order_id: cmd.order_id,
          confirmed_at: DateTime.utc_now()
        }
        EventStore.append(cmd.order_id, "Order", [event])
        {:ok, cmd.order_id}

      status ->
        {:error, "Cannot confirm order with status: #{status}"}
    end
  end

  defp calculate_total(items) do
    Enum.sum(Enum.map(items, &(&1.price * &1.quantity)))
  end
end
```

```elixir
# Query side - Read-optimized projections
defmodule MyApp.Projections.OrderSummary do
  use Ecto.Schema

  # Read model ที่ optimize สำหรับ queries
  # อาจเป็น denormalized view
  schema "order_summaries" do
    field :order_id, :string
    field :user_id, :integer
    field :user_email, :string
    field :status, :string
    field :total, :decimal
    field :items_count, :integer
    field :placed_at, :utc_datetime

    timestamps()
  end
end

# Projector - อัปเดต read model เมื่อมี event
defmodule MyApp.Projectors.OrderProjector do
  alias MyApp.{Repo}
  alias MyApp.Projections.OrderSummary
  alias MyApp.Events

  def project(%Events.OrderPlaced{} = event) do
    %OrderSummary{}
    |> Ecto.Changeset.change(%{
      order_id: event.order_id,
      user_id: event.user_id,
      status: "pending",
      total: event.total,
      items_count: length(event.items),
      placed_at: DateTime.utc_now()
    })
    |> Repo.insert()
  end

  def project(%Events.OrderConfirmed{} = event) do
    case Repo.get_by(OrderSummary, order_id: event.order_id) do
      nil -> :ok
      summary ->
        summary
        |> Ecto.Changeset.change(%{status: "confirmed"})
        |> Repo.update()
    end
  end
end

# Query functions ที่ query จาก read model
defmodule MyApp.Queries do
  import Ecto.Query
  alias MyApp.{Repo}
  alias MyApp.Projections.OrderSummary

  def get_user_orders(user_id, opts \\ []) do
    status = Keyword.get(opts, :status)

    OrderSummary
    |> where([o], o.user_id == ^user_id)
    |> then(fn q ->
      if status, do: where(q, [o], o.status == ^status), else: q
    end)
    |> order_by([o], desc: o.placed_at)
    |> Repo.all()
  end

  def get_order_stats(from_date, to_date) do
    from(o in OrderSummary,
      where: o.placed_at >= ^from_date and o.placed_at <= ^to_date,
      group_by: o.status,
      select: %{
        status: o.status,
        count: count(o.id),
        total: sum(o.total)
      }
    )
    |> Repo.all()
  end
end
```

---

## สรุป

```
Architecture Patterns Summary:
┌─────────────────────────────────────────────────────────┐
│ Context Pattern (Phoenix Default)                       │
│   Web Layer → Context (Public API) → Schemas/Queries   │
├─────────────────────────────────────────────────────────┤
│ Command Pattern                                         │
│   Request → Command Struct → Handler → Result          │
├─────────────────────────────────────────────────────────┤
│ DDD                                                     │
│   Value Objects + Entities + Aggregates + Domain Events │
├─────────────────────────────────────────────────────────┤
│ CQRS                                                    │
│   Commands → Write Model → Events → Read Model          │
│                                       ↑                 │
│                               Projectors update it      │
└─────────────────────────────────────────────────────────┘

Module Organization:
lib/
├── my_app/                    # Business logic (no HTTP)
│   ├── accounts/              # Context internals
│   ├── accounts.ex            # Public API
│   ├── commands/              # Command objects
│   ├── events/                # Domain events
│   └── queries/               # Read-side queries
└── my_app_web/                # HTTP/WebSocket layer
    ├── controllers/
    ├── live/
    └── components/

Key Principles:
• Contexts = Bounded Boundaries (ไม่ cross โดยตรง)
• Commands = Atomic operations with full context
• Events = Facts that happened (immutable)
• Queries = Optimized for reads (can be denormalized)
• Behaviours = Dependency injection contracts
• Umbrella = Large apps แยกเป็น sub-applications
```

---

*ก่อนหน้า: [Part 44 - Ecto Advanced](part_44.md) | ต่อไป: Part 46*
