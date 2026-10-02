# Part 90: Elixir Design Patterns (รูปแบบการออกแบบ Elixir)

## เป้าหมายการเรียนรู้

หลังจากจบ Part นี้ คุณจะสามารถ:
- Functional core, imperative shell
- Builder pattern ใน Elixir
- State machine pattern
- Saga pattern สำหรับ distributed transactions

---

## 1. Functional Core, Imperative Shell

```elixir
# Pure functional core - no side effects
defmodule MyApp.Pricing do
  # Pure function: input -> output, no side effects
  def calculate(base_price, quantity, coupon \\ nil) do
    subtotal = base_price * quantity
    discount = apply_discount(subtotal, coupon)
    tax = calculate_tax(subtotal - discount)
    %{
      subtotal: subtotal,
      discount: discount,
      tax: tax,
      total: subtotal - discount + tax
    }
  end

  defp apply_discount(subtotal, nil), do: Decimal.new(0)
  defp apply_discount(subtotal, %{type: "percent", value: pct}) do
    Decimal.mult(subtotal, Decimal.div(pct, 100))
  end
  defp apply_discount(subtotal, %{type: "fixed", value: amount}) do
    min(amount, subtotal)
  end

  defp calculate_tax(taxable), do: Decimal.mult(taxable, Decimal.new("0.07"))
end

# Imperative shell - handles I/O, state, side effects
defmodule MyApp.Orders do
  def create_order(user_id, cart_id, coupon_code) do
    # I/O operations at the edges
    user = MyApp.Accounts.get_user!(user_id)
    cart = MyApp.Cart.get_cart!(cart_id)
    coupon = if coupon_code, do: MyApp.Coupons.get_valid(coupon_code), else: nil

    # Pure calculation
    pricing = MyApp.Pricing.calculate(cart.total, 1, coupon)

    # I/O operations again
    Repo.transaction(fn ->
      order = Repo.insert!(%Order{
        user_id: user_id,
        subtotal: pricing.subtotal,
        discount: pricing.discount,
        tax: pricing.tax,
        total: pricing.total
      })
      MyApp.Cart.clear(cart_id)
      if coupon, do: MyApp.Coupons.mark_used(coupon.id, user_id)
      order
    end)
  end
end
```

---

## 2. State Machine Pattern

```elixir
defmodule MyApp.Order.StateMachine do
  @transitions %{
    pending: [:processing, :cancelled],
    processing: [:shipped, :cancelled, :failed],
    shipped: [:delivered, :returned],
    delivered: [:returned],
    failed: [:pending],
    cancelled: [],
    returned: []
  }

  def transition(order, new_state) do
    current = order.status |> String.to_atom()
    allowed = @transitions[current] || []

    if new_state in allowed do
      {:ok, execute_transition(order, new_state)}
    else
      {:error, "Invalid transition from #{current} to #{new_state}"}
    end
  end

  defp execute_transition(order, :processing) do
    MyApp.Orders.update_status(order, "processing")
    MyApp.Emails.send_processing_confirmation(order)
    order
  end

  defp execute_transition(order, :shipped) do
    {:ok, updated} = MyApp.Orders.update_status(order, "shipped")
    MyApp.Emails.send_shipping_notification(updated)
    MyApp.Analytics.track("order_shipped", %{order_id: order.id})
    updated
  end

  defp execute_transition(order, :cancelled) do
    {:ok, updated} = MyApp.Orders.update_status(order, "cancelled")
    if order.payment_intent_id, do: MyApp.Billing.Refunds.refund_order(updated)
    updated
  end

  defp execute_transition(order, new_state) do
    {:ok, updated} = MyApp.Orders.update_status(order, to_string(new_state))
    updated
  end
end
```

---

## 3. Saga Pattern for Distributed Transactions

```elixir
defmodule MyApp.Sagas.CreateOrder do
  # Saga: sequence of steps with compensating transactions

  def execute(params) do
    with {:ok, order} <- create_order(params),
         {:ok, payment} <- charge_payment(order, params.payment_method),
         {:ok, _} <- reserve_inventory(order),
         {:ok, _} <- send_confirmation(order) do
      {:ok, order}
    else
      {:error, :payment_failed, reason} ->
        cancel_order(params[:created_order_id])
        {:error, {:payment_failed, reason}}

      {:error, :inventory_unavailable} ->
        refund_payment(params[:payment_id])
        cancel_order(params[:created_order_id])
        {:error, :inventory_unavailable}

      error ->
        rollback_all(params)
        error
    end
  end

  # Forward steps
  defp create_order(params) do
    case MyApp.Orders.create(params) do
      {:ok, order} ->
        # Store for potential rollback
        Process.put(:created_order_id, order.id)
        {:ok, order}
      error -> error
    end
  end

  defp charge_payment(order, payment_method) do
    case MyApp.Billing.charge(order, payment_method) do
      {:ok, payment} ->
        Process.put(:payment_id, payment.id)
        {:ok, payment}
      {:error, reason} ->
        {:error, :payment_failed, reason}
    end
  end

  defp reserve_inventory(order) do
    case MyApp.Inventory.reserve(order.items) do
      :ok -> {:ok, :reserved}
      {:error, :insufficient} -> {:error, :inventory_unavailable}
    end
  end

  defp send_confirmation(order) do
    MyApp.Emails.order_confirmation(order) |> MyApp.Mailer.deliver()
    {:ok, :sent}
  end

  # Compensating transactions
  defp cancel_order(nil), do: :ok
  defp cancel_order(order_id) do
    order = MyApp.Orders.get_order!(order_id)
    MyApp.Order.StateMachine.transition(order, :cancelled)
  end

  defp refund_payment(nil), do: :ok
  defp refund_payment(payment_id) do
    payment = MyApp.Payments.get!(payment_id)
    MyApp.Billing.Refunds.refund(payment)
  end

  defp rollback_all(params) do
    refund_payment(Process.get(:payment_id))
    cancel_order(Process.get(:created_order_id))
  end
end
```

---

## 4. Builder Pattern

```elixir
defmodule MyApp.QueryBuilder do
  defstruct filters: [], sorts: [], includes: [], page: 1, per_page: 20

  def new, do: %__MODULE__{}

  def filter(builder, key, value) do
    %{builder | filters: [{key, value} | builder.filters]}
  end

  def sort(builder, field, direction \\ :asc) do
    %{builder | sorts: [{field, direction} | builder.sorts]}
  end

  def include(builder, assoc) when is_atom(assoc) do
    %{builder | includes: [assoc | builder.includes]}
  end

  def paginate(builder, page, per_page \\ 20) do
    %{builder | page: page, per_page: per_page}
  end

  def build(builder, queryable) do
    import Ecto.Query

    queryable
    |> apply_filters(builder.filters)
    |> apply_sorts(builder.sorts)
    |> preload(^builder.includes)
    |> then(fn q ->
      offset = (builder.page - 1) * builder.per_page
      from(q, limit: ^builder.per_page, offset: ^offset)
    end)
  end

  defp apply_filters(query, filters) do
    import Ecto.Query
    Enum.reduce(filters, query, fn {key, value}, q ->
      where(q, [r], field(r, ^key) == ^value)
    end)
  end

  defp apply_sorts(query, sorts) do
    import Ecto.Query
    Enum.reduce(sorts, query, fn {field, direction}, q ->
      order_by(q, [r], [{^direction, field(r, ^field)}])
    end)
  end
end

# Usage:
articles =
  MyApp.QueryBuilder.new()
  |> MyApp.QueryBuilder.filter(:status, "published")
  |> MyApp.QueryBuilder.filter(:category_id, 5)
  |> MyApp.QueryBuilder.sort(:inserted_at, :desc)
  |> MyApp.QueryBuilder.include(:author)
  |> MyApp.QueryBuilder.paginate(2, 15)
  |> MyApp.QueryBuilder.build(MyApp.Article)
  |> MyApp.Repo.all()
```

---

## สรุป

```
Elixir Design Patterns:
├── Functional core: pure logic, testable
├── Imperative shell: I/O at edges
├── State machine: explicit transitions
├── Saga: distributed transaction rollback
└── Builder: fluent API construction

State Machine Benefits:
├── Invalid transitions prevented
├── Side effects in execute_transition
└── Clear audit trail

Saga Benefits:
├── Compensating transactions
├── Partial failure recovery
└── Explicit rollback steps
```

---

*ก่อนหน้า: [Part 89](part_89.md) | ต่อไป: [Part 91 - Umbrella Projects](part_91.md)*
