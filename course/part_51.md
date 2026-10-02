# Part 51: Real-world Project - E-Commerce (ระบบร้านค้าออนไลน์)

## เป้าหมายการเรียนรู้

หลังจากจบ Part นี้ คุณจะสามารถ:
- สร้าง e-commerce system ที่สมบูรณ์
- จัดการ products, cart, orders
- ระบบ payment integration
- Inventory management

---

## 1. โครงสร้างโปรเจกต์

```elixir
# สร้าง Phoenix project
mix phx.new shop --database postgres
cd shop

# สร้าง contexts หลัก
mix phx.gen.context Catalog Product products \
  name:string \
  description:text \
  price:decimal \
  sku:string \
  stock:integer \
  published:boolean

mix phx.gen.context Orders Cart carts \
  user_id:references:users \
  status:string

mix phx.gen.context Orders CartItem cart_items \
  cart_id:references:carts \
  product_id:references:products \
  quantity:integer \
  price:decimal
```

---

## 2. Product Schema

```elixir
defmodule Shop.Catalog.Product do
  use Ecto.Schema
  import Ecto.Changeset

  schema "products" do
    field :name, :string
    field :description, :string
    field :price, :decimal
    field :sku, :string
    field :stock, :integer, default: 0
    field :published, :boolean, default: false
    field :images, {:array, :string}, default: []

    belongs_to :category, Shop.Catalog.Category
    has_many :cart_items, Shop.Orders.CartItem
    has_many :order_items, Shop.Orders.OrderItem

    timestamps()
  end

  def changeset(product, attrs) do
    product
    |> cast(attrs, [:name, :description, :price, :sku, :stock, :published, :category_id])
    |> validate_required([:name, :price, :sku])
    |> validate_number(:price, greater_than: 0)
    |> validate_number(:stock, greater_than_or_equal_to: 0)
    |> unique_constraint(:sku)
  end

  def publish_changeset(product) do
    product
    |> change(published: true)
    |> validate_required([:description])
  end
end
```

---

## 3. Shopping Cart

```elixir
defmodule Shop.Orders do
  alias Shop.Repo
  alias Shop.Orders.{Cart, CartItem}
  alias Shop.Catalog.Product
  import Ecto.Query

  def get_or_create_cart(user_id) do
    case Repo.get_by(Cart, user_id: user_id, status: "active") do
      nil -> create_cart(user_id)
      cart -> {:ok, cart}
    end
  end

  def add_to_cart(cart, product_id, quantity \\ 1) do
    product = Repo.get!(Product, product_id)

    if product.stock < quantity do
      {:error, :insufficient_stock}
    else
      case Repo.get_by(CartItem, cart_id: cart.id, product_id: product_id) do
        nil ->
          %CartItem{}
          |> CartItem.changeset(%{
            cart_id: cart.id,
            product_id: product_id,
            quantity: quantity,
            price: product.price
          })
          |> Repo.insert()

        existing ->
          existing
          |> CartItem.changeset(%{quantity: existing.quantity + quantity})
          |> Repo.update()
      end
    end
  end

  def remove_from_cart(cart_item_id) do
    cart_item = Repo.get!(CartItem, cart_item_id)
    Repo.delete(cart_item)
  end

  def cart_total(cart) do
    cart = Repo.preload(cart, :cart_items)

    Enum.reduce(cart.cart_items, Decimal.new(0), fn item, total ->
      Decimal.add(total, Decimal.mult(item.price, item.quantity))
    end)
  end

  def checkout(cart, payment_info) do
    Ecto.Multi.new()
    |> Ecto.Multi.run(:validate_stock, fn _repo, _changes ->
      validate_cart_stock(cart)
    end)
    |> Ecto.Multi.run(:create_order, fn repo, _changes ->
      total = cart_total(cart)
      %Shop.Orders.Order{}
      |> Shop.Orders.Order.changeset(%{
        user_id: cart.user_id,
        total: total,
        status: "pending"
      })
      |> repo.insert()
    end)
    |> Ecto.Multi.run(:create_order_items, fn repo, %{create_order: order} ->
      cart = repo.preload(cart, :cart_items)
      items = Enum.map(cart.cart_items, fn item ->
        %{
          order_id: order.id,
          product_id: item.product_id,
          quantity: item.quantity,
          price: item.price
        }
      end)
      {count, _} = repo.insert_all(Shop.Orders.OrderItem, items)
      {:ok, count}
    end)
    |> Ecto.Multi.run(:reserve_stock, fn repo, _changes ->
      cart = repo.preload(cart, :cart_items)
      Enum.each(cart.cart_items, fn item ->
        from(p in Product, where: p.id == ^item.product_id)
        |> repo.update_all(inc: [stock: -item.quantity])
      end)
      {:ok, :reserved}
    end)
    |> Ecto.Multi.run(:process_payment, fn _repo, %{create_order: order} ->
      process_payment(order, payment_info)
    end)
    |> Ecto.Multi.update(:complete_order, fn %{create_order: order} ->
      Shop.Orders.Order.changeset(order, %{status: "paid"})
    end)
    |> Ecto.Multi.update(:close_cart, fn _changes ->
      Cart.changeset(cart, %{status: "completed"})
    end)
    |> Repo.transaction()
  end

  defp validate_cart_stock(cart) do
    cart = Repo.preload(cart, cart_items: :product)
    errors = Enum.filter(cart.cart_items, fn item ->
      item.product.stock < item.quantity
    end)

    if Enum.empty?(errors) do
      {:ok, :valid}
    else
      products = Enum.map(errors, & &1.product.name)
      {:error, "Insufficient stock for: #{Enum.join(products, ", ")}"}
    end
  end

  defp process_payment(order, _payment_info) do
    # ส่ง payment ไป payment gateway (Stripe, Omise)
    # นี่คือ mock
    {:ok, %{transaction_id: "txn_#{order.id}_#{System.unique_integer()}"}}
  end
end
```

---

## 4. LiveView Cart

```elixir
defmodule ShopWeb.CartLive do
  use ShopWeb, :live_view

  alias Shop.Orders

  def mount(_params, %{"user_id" => user_id}, socket) do
    {:ok, cart} = Orders.get_or_create_cart(user_id)
    cart = Shop.Repo.preload(cart, cart_items: :product)

    {:ok, assign(socket, cart: cart, total: Orders.cart_total(cart))}
  end

  def handle_event("remove_item", %{"id" => id}, socket) do
    {:ok, _} = Orders.remove_from_cart(id)
    cart = Shop.Repo.preload(socket.assigns.cart, [cart_items: :product], force: true)

    {:noreply, assign(socket, cart: cart, total: Orders.cart_total(cart))}
  end

  def handle_event("update_quantity", %{"id" => id, "quantity" => qty}, socket) do
    item = Shop.Repo.get!(Shop.Orders.CartItem, id)
    {:ok, _} = item
      |> Shop.Orders.CartItem.changeset(%{quantity: String.to_integer(qty)})
      |> Shop.Repo.update()

    cart = Shop.Repo.preload(socket.assigns.cart, [cart_items: :product], force: true)
    {:noreply, assign(socket, cart: cart, total: Orders.cart_total(cart))}
  end

  def handle_event("checkout", _params, socket) do
    case Orders.checkout(socket.assigns.cart, %{}) do
      {:ok, %{create_order: order}} ->
        {:noreply, push_navigate(socket, to: "/orders/#{order.id}/confirmation")}

      {:error, :process_payment, reason, _} ->
        {:noreply, put_flash(socket, :error, "Payment failed: #{reason}")}

      {:error, _, reason, _} ->
        {:noreply, put_flash(socket, :error, reason)}
    end
  end

  def render(assigns) do
    ~H"""
    <div class="max-w-2xl mx-auto p-4">
      <h1 class="text-2xl font-bold mb-6">ตะกร้าสินค้า</h1>

      <%= if Enum.empty?(@cart.cart_items) do %>
        <p class="text-gray-500">ตะกร้าของคุณว่างเปล่า</p>
      <% else %>
        <div class="space-y-4">
          <%= for item <- @cart.cart_items do %>
            <div class="flex items-center gap-4 p-4 border rounded">
              <div class="flex-1">
                <p class="font-semibold"><%= item.product.name %></p>
                <p class="text-gray-600">฿<%= item.price %></p>
              </div>
              <input
                type="number"
                value={item.quantity}
                min="1"
                class="w-20 border rounded p-1"
                phx-change="update_quantity"
                phx-value-id={item.id}
                name="quantity"
              />
              <button
                phx-click="remove_item"
                phx-value-id={item.id}
                class="text-red-500 hover:text-red-700"
              >
                ลบ
              </button>
            </div>
          <% end %>
        </div>

        <div class="mt-6 pt-6 border-t">
          <div class="flex justify-between text-xl font-bold">
            <span>รวม:</span>
            <span>฿<%= @total %></span>
          </div>
          <button
            phx-click="checkout"
            class="mt-4 w-full bg-blue-600 text-white py-3 rounded-lg hover:bg-blue-700"
          >
            ชำระเงิน
          </button>
        </div>
      <% end %>
    </div>
    """
  end
end
```

---

## 5. Product Listing with Search

```elixir
defmodule ShopWeb.ProductsLive do
  use ShopWeb, :live_view

  alias Shop.Catalog

  def mount(_params, _session, socket) do
    {:ok, assign(socket, products: Catalog.list_products(), search: "")}
  end

  def handle_event("search", %{"query" => query}, socket) do
    products = Catalog.search_products(query)
    {:noreply, assign(socket, products: products, search: query)}
  end

  def handle_event("add_to_cart", %{"product_id" => product_id}, socket) do
    user_id = socket.assigns.current_user.id
    {:ok, cart} = Shop.Orders.get_or_create_cart(user_id)

    case Shop.Orders.add_to_cart(cart, product_id) do
      {:ok, _} ->
        {:noreply, put_flash(socket, :info, "เพิ่มในตะกร้าแล้ว")}

      {:error, :insufficient_stock} ->
        {:noreply, put_flash(socket, :error, "สินค้าไม่เพียงพอ")}
    end
  end

  def render(assigns) do
    ~H"""
    <div>
      <form phx-change="search" class="mb-6">
        <input
          type="text"
          name="query"
          value={@search}
          placeholder="ค้นหาสินค้า..."
          class="w-full border rounded-lg p-3"
          phx-debounce="300"
        />
      </form>

      <div class="grid grid-cols-3 gap-6">
        <%= for product <- @products do %>
          <div class="border rounded-lg p-4">
            <h3 class="font-semibold"><%= product.name %></h3>
            <p class="text-gray-600 text-sm"><%= product.description %></p>
            <div class="flex justify-between items-center mt-4">
              <span class="text-lg font-bold">฿<%= product.price %></span>
              <button
                phx-click="add_to_cart"
                phx-value-product_id={product.id}
                class="bg-blue-600 text-white px-4 py-2 rounded"
              >
                เพิ่มในตะกร้า
              </button>
            </div>
          </div>
        <% end %>
      </div>
    </div>
    """
  end
end
```

---

## 6. Order Management

```elixir
defmodule Shop.Orders.Order do
  use Ecto.Schema
  import Ecto.Changeset

  schema "orders" do
    field :status, :string, default: "pending"
    field :total, :decimal
    field :shipping_address, :map
    field :payment_method, :string
    field :transaction_id, :string

    belongs_to :user, Shop.Accounts.User
    has_many :order_items, Shop.Orders.OrderItem

    timestamps()
  end

  def changeset(order, attrs) do
    order
    |> cast(attrs, [:status, :total, :shipping_address, :payment_method, :transaction_id, :user_id])
    |> validate_required([:total, :user_id])
    |> validate_inclusion(:status, ~w(pending paid shipped delivered cancelled))
  end
end

# Order status transitions
defmodule Shop.Orders.OrderStateMachine do
  @transitions %{
    "pending" => ["paid", "cancelled"],
    "paid" => ["shipped", "cancelled"],
    "shipped" => ["delivered"],
    "delivered" => [],
    "cancelled" => []
  }

  def can_transition?(from, to) do
    to in Map.get(@transitions, from, [])
  end

  def transition(order, new_status) do
    if can_transition?(order.status, new_status) do
      order
      |> Shop.Orders.Order.changeset(%{status: new_status})
      |> Shop.Repo.update()
    else
      {:error, "Invalid status transition from #{order.status} to #{new_status}"}
    end
  end
end
```

---

## 7. Payment Integration (Stripe)

```elixir
# mix.exs: {:stripity_stripe, "~> 2.17"}
defmodule Shop.Payment.Stripe do
  def create_payment_intent(amount_satang, metadata \\ %{}) do
    Stripe.PaymentIntent.create(%{
      amount: amount_satang,  # Stripe ใช้ satang (smallest unit)
      currency: "thb",
      metadata: metadata,
      automatic_payment_methods: %{enabled: true}
    })
  end

  def confirm_payment(payment_intent_id) do
    case Stripe.PaymentIntent.retrieve(payment_intent_id) do
      {:ok, %{status: "succeeded"} = intent} ->
        {:ok, intent}

      {:ok, %{status: status}} ->
        {:error, "Payment status: #{status}"}

      {:error, error} ->
        {:error, error.message}
    end
  end

  def refund(payment_intent_id, amount_satang \\ nil) do
    params = %{payment_intent: payment_intent_id}
    params = if amount_satang, do: Map.put(params, :amount, amount_satang), else: params
    Stripe.Refund.create(params)
  end
end
```

---

## สรุป

```
E-Commerce System:
├── Catalog: products, categories, inventory
├── Orders: cart, checkout, order management
├── Payment: Stripe integration
└── LiveView: real-time cart updates

Checkout Flow:
1. User adds items to cart
2. Validate stock availability
3. Create order record
4. Create order items
5. Reserve/decrement stock
6. Process payment
7. Complete order (Ecto.Multi)
```

---

*ก่อนหน้า: [Part 50](part_50.md) | ต่อไป: [Part 52 - Real-time Chat App](part_52.md)*
