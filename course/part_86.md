# Part 86: Payment Processing Advanced (การประมวลผลการชำระเงิน)

## เป้าหมายการเรียนรู้

หลังจากจบ Part นี้ คุณจะสามารถ:
- Stripe checkout session
- Refunds และ disputes
- Billing portal
- Idempotent payment processing

---

## 1. Stripe Checkout Session

```elixir
defmodule MyApp.Billing.Checkout do
  @stripe_key Application.compile_env(:my_app, :stripe_secret_key)

  def create_checkout_session(user, items) do
    line_items = Enum.map(items, fn item ->
      %{
        price: item.stripe_price_id,
        quantity: item.quantity
      }
    end)

    Stripe.Session.create(%{
      customer: user.stripe_customer_id,
      payment_method_types: ["card", "promptpay"],  # PromptPay for Thailand
      line_items: line_items,
      mode: "payment",
      success_url: "#{base_url()}/orders/success?session_id={CHECKOUT_SESSION_ID}",
      cancel_url: "#{base_url()}/cart",
      metadata: %{
        user_id: user.id,
        order_type: "one_time"
      }
    })
  end

  def create_subscription_session(user, price_id) do
    Stripe.Session.create(%{
      customer: user.stripe_customer_id,
      payment_method_types: ["card"],
      line_items: [%{price: price_id, quantity: 1}],
      mode: "subscription",
      subscription_data: %{
        trial_period_days: 14
      },
      success_url: "#{base_url()}/billing/success?session_id={CHECKOUT_SESSION_ID}",
      cancel_url: "#{base_url()}/pricing"
    })
  end

  defp base_url, do: MyAppWeb.Endpoint.url()
end

# Controller
defmodule MyAppWeb.CheckoutController do
  use MyAppWeb, :controller

  def create(conn, %{"items" => items}) do
    user = conn.assigns.current_user

    case MyApp.Billing.Checkout.create_checkout_session(user, items) do
      {:ok, session} ->
        json(conn, %{checkout_url: session.url})

      {:error, reason} ->
        conn |> put_status(422) |> json(%{error: reason})
    end
  end

  def success(conn, %{"session_id" => session_id}) do
    case Stripe.Session.retrieve(session_id) do
      {:ok, session} when session.payment_status == "paid" ->
        MyApp.Orders.fulfill(session.metadata.user_id, session_id)
        render(conn, :success)

      _ ->
        redirect(conn, to: ~p"/cart")
    end
  end
end
```

---

## 2. Refund Processing

```elixir
defmodule MyApp.Billing.Refunds do
  def refund_order(order, opts \\ []) do
    amount = Keyword.get(opts, :amount)  # nil = full refund
    reason = Keyword.get(opts, :reason, "requested_by_customer")

    refund_params = %{
      payment_intent: order.stripe_payment_intent_id,
      reason: reason
    }
    refund_params = if amount, do: Map.put(refund_params, :amount, amount), else: refund_params

    case Stripe.Refund.create(refund_params) do
      {:ok, refund} ->
        # Record refund in DB
        MyApp.Repo.insert(%MyApp.Refund{
          order_id: order.id,
          stripe_refund_id: refund.id,
          amount: refund.amount,
          reason: reason,
          status: refund.status
        })

        # Update order status
        MyApp.Orders.update_status(order, "refunded")

        {:ok, refund}

      {:error, reason} ->
        {:error, reason}
    end
  end

  def handle_dispute(dispute_id) do
    {:ok, dispute} = Stripe.Dispute.retrieve(dispute_id)

    evidence = %{
      customer_email_address: dispute.charge.billing_details.email,
      product_description: "Digital product access",
      access_activity_log: build_access_log(dispute.charge)
    }

    Stripe.Dispute.update(dispute_id, %{evidence: evidence})
  end

  defp build_access_log(charge) do
    user_id = charge.metadata["user_id"]
    events = MyApp.Analytics.user_events(user_id, limit: 50)
    Enum.map_join(events, "\n", &"#{&1.inserted_at}: #{&1.name}")
  end
end
```

---

## 3. Billing Portal

```elixir
defmodule MyApp.Billing.Portal do
  def create_portal_session(user) do
    Stripe.BillingPortal.Session.create(%{
      customer: user.stripe_customer_id,
      return_url: "#{MyAppWeb.Endpoint.url()}/settings/billing"
    })
  end
end

# Controller
def billing_portal(conn, _params) do
  user = conn.assigns.current_user

  case MyApp.Billing.Portal.create_portal_session(user) do
    {:ok, session} ->
      redirect(conn, external: session.url)

    {:error, _} ->
      conn
      |> put_flash(:error, "ไม่สามารถเปิด billing portal ได้")
      |> redirect(to: ~p"/settings")
  end
end
```

---

## 4. Idempotent Payment

```elixir
defmodule MyApp.Billing.IdempotentPayment do
  alias MyApp.Repo
  alias MyApp.Payment

  def charge(order, amount, currency \\ "thb") do
    idempotency_key = "order_#{order.id}_#{amount}"

    # Check if already charged
    case Repo.get_by(Payment, idempotency_key: idempotency_key) do
      %Payment{status: "succeeded"} = payment ->
        {:ok, payment}

      nil ->
        do_charge(order, amount, currency, idempotency_key)
    end
  end

  defp do_charge(order, amount, currency, idempotency_key) do
    stripe_opts = [idempotency_key: idempotency_key]

    case Stripe.PaymentIntent.create(%{
      amount: amount,
      currency: currency,
      customer: order.user.stripe_customer_id,
      payment_method: order.user.default_payment_method_id,
      confirm: true,
      metadata: %{order_id: order.id}
    }, stripe_opts) do
      {:ok, pi} when pi.status == "succeeded" ->
        {:ok, record_payment(order, pi, idempotency_key)}

      {:ok, pi} ->
        {:error, "Payment status: #{pi.status}"}

      {:error, reason} ->
        {:error, reason}
    end
  end

  defp record_payment(order, payment_intent, idempotency_key) do
    Repo.insert!(%Payment{
      order_id: order.id,
      stripe_payment_intent_id: payment_intent.id,
      amount: payment_intent.amount,
      currency: payment_intent.currency,
      status: "succeeded",
      idempotency_key: idempotency_key
    })
  end
end
```

---

## สรุป

```
Payment Flows:
├── Checkout Session: redirect to Stripe
├── Payment Intent: in-app payment
├── Subscriptions: recurring billing
└── Billing Portal: customer self-service

Reliability:
├── Idempotency keys: prevent double charges
├── Webhooks: async event handling
├── Retry logic: on network failure
└── Reconciliation: verify with Stripe

Refunds:
├── Full refund: no amount specified
├── Partial refund: specify amount
└── Dispute evidence: access logs
```

---

*ก่อนหน้า: [Part 85](part_85.md) | ต่อไป: [Part 87 - Monitoring Production](part_87.md)*
