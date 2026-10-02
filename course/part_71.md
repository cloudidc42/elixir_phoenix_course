# Part 71: SaaS Platform Architecture (สถาปัตยกรรม SaaS)

## เป้าหมายการเรียนรู้

หลังจากจบ Part นี้ คุณจะสามารถ:
- ออกแบบ SaaS platform ที่สมบูรณ์
- Subscription management
- Usage-based billing
- Feature flags

---

## 1. Subscription Plans

```elixir
defmodule SaaS.Billing do
  use Ecto.Schema

  schema "subscription_plans" do
    field :name, :string           # "Free", "Pro", "Enterprise"
    field :stripe_price_id, :string
    field :price_monthly, :decimal
    field :price_yearly, :decimal
    field :features, {:array, :string}
    field :limits, :map            # %{users: 5, storage_gb: 10, api_calls: 1000}
    field :active, :boolean, default: true
    timestamps()
  end
end

defmodule SaaS.Subscriptions.Subscription do
  use Ecto.Schema

  schema "subscriptions" do
    field :status, :string  # trialing, active, past_due, cancelled
    field :plan_id, :integer
    field :stripe_subscription_id, :string
    field :current_period_start, :utc_datetime
    field :current_period_end, :utc_datetime
    field :trial_end, :utc_datetime
    field :cancel_at_period_end, :boolean, default: false

    belongs_to :organization, SaaS.Organizations.Organization
    belongs_to :plan, SaaS.Billing.Plan

    timestamps()
  end
end
```

---

## 2. Stripe Subscription Integration

```elixir
defmodule SaaS.Billing.StripeService do
  @stripe_secret Application.compile_env(:saas, :stripe_secret_key)

  def create_customer(org) do
    Stripe.Customer.create(%{
      email: org.billing_email,
      name: org.name,
      metadata: %{org_id: org.id}
    })
  end

  def create_subscription(customer_id, price_id, opts \\ []) do
    trial_days = Keyword.get(opts, :trial_days, 14)

    Stripe.Subscription.create(%{
      customer: customer_id,
      items: [%{price: price_id}],
      trial_period_days: trial_days,
      payment_behavior: "default_incomplete",
      expand: ["latest_invoice.payment_intent"]
    })
  end

  def cancel_subscription(subscription_id, opts \\ []) do
    at_period_end = Keyword.get(opts, :at_period_end, true)

    if at_period_end do
      Stripe.Subscription.update(subscription_id, %{cancel_at_period_end: true})
    else
      Stripe.Subscription.cancel(subscription_id)
    end
  end

  def upgrade_plan(subscription_id, new_price_id) do
    subscription = Stripe.Subscription.retrieve!(subscription_id)
    item_id = hd(subscription.items.data).id

    Stripe.Subscription.update(subscription_id, %{
      items: [%{id: item_id, price: new_price_id}],
      proration_behavior: "create_prorations"
    })
  end

  def get_invoices(customer_id) do
    Stripe.Invoice.list(%{customer: customer_id, limit: 10})
  end
end
```

---

## 3. Stripe Webhooks Handler

```elixir
defmodule SaaSWeb.StripeWebhookController do
  use SaaSWeb, :controller

  alias SaaS.Billing.Events

  def handle(conn, _params) do
    payload = conn.assigns[:raw_body]
    sig_header = get_req_header(conn, "stripe-signature") |> List.first()
    webhook_secret = Application.get_env(:saas, :stripe_webhook_secret)

    case Stripe.Webhook.construct_event(payload, sig_header, webhook_secret) do
      {:ok, event} ->
        process_event(event)
        json(conn, %{ok: true})

      {:error, _} ->
        conn |> put_status(400) |> json(%{error: "Invalid signature"})
    end
  end

  defp process_event(%{type: "customer.subscription.created"} = event) do
    sub = event.data.object
    Events.subscription_created(sub)
  end

  defp process_event(%{type: "customer.subscription.updated"} = event) do
    sub = event.data.object
    Events.subscription_updated(sub)
  end

  defp process_event(%{type: "customer.subscription.deleted"} = event) do
    sub = event.data.object
    Events.subscription_cancelled(sub)
  end

  defp process_event(%{type: "invoice.payment_failed"} = event) do
    invoice = event.data.object
    Events.payment_failed(invoice)
  end

  defp process_event(%{type: "invoice.payment_succeeded"} = event) do
    invoice = event.data.object
    Events.payment_succeeded(invoice)
  end

  defp process_event(_event), do: :ok
end
```

---

## 4. Feature Flags

```elixir
defmodule SaaS.Features do
  @features %{
    "free" => [:basic_analytics, :email_support, :api_access],
    "pro" => [:basic_analytics, :advanced_analytics, :priority_support,
              :api_access, :custom_domain, :team_collaboration],
    "enterprise" => [:all]
  }

  def has_feature?(org, feature) do
    plan = get_current_plan(org)

    if "all" in @features[plan] do
      true
    else
      feature in @features[plan]
    end
  end

  def require_feature(org, feature) do
    if has_feature?(org, feature) do
      :ok
    else
      {:error, {:feature_unavailable, feature, current_plan: get_current_plan(org)}}
    end
  end

  defp get_current_plan(org) do
    case SaaS.Billing.get_active_subscription(org) do
      nil -> "free"
      sub -> sub.plan.name |> String.downcase()
    end
  end
end

# Plug for feature gating
defmodule SaaSWeb.Plugs.RequireFeature do
  import Plug.Conn
  alias SaaS.Features

  def init(feature), do: feature

  def call(conn, feature) do
    org = conn.assigns[:current_organization]

    case Features.require_feature(org, feature) do
      :ok ->
        conn

      {:error, {:feature_unavailable, _, current_plan: plan}} ->
        conn
        |> put_status(402)
        |> Phoenix.Controller.json(%{
          error: "Feature requires Pro plan",
          current_plan: plan,
          upgrade_url: "/billing/upgrade"
        })
        |> halt()
    end
  end
end
```

---

## 5. Usage Tracking

```elixir
defmodule SaaS.Usage do
  alias SaaS.Repo
  alias SaaS.Usage.UsageRecord
  import Ecto.Query

  def track(org_id, metric, amount \\ 1) do
    date = Date.utc_today()

    # Upsert daily usage
    %UsageRecord{}
    |> UsageRecord.changeset(%{
      org_id: org_id,
      metric: metric,
      date: date,
      amount: amount
    })
    |> Repo.insert(
      on_conflict: [inc: [amount: amount]],
      conflict_target: [:org_id, :metric, :date]
    )
  end

  def get_usage(org_id, metric, period \\ :current_month) do
    {start_date, end_date} = date_range(period)

    Repo.one(
      from(u in UsageRecord,
        where: u.org_id == ^org_id
          and u.metric == ^metric
          and u.date >= ^start_date
          and u.date <= ^end_date,
        select: sum(u.amount)
      )
    ) || 0
  end

  def check_limit(org_id, metric, limit) do
    current = get_usage(org_id, metric)
    if current < limit do
      {:ok, current}
    else
      {:error, :limit_exceeded}
    end
  end

  defp date_range(:current_month) do
    today = Date.utc_today()
    start_date = Date.new!(today.year, today.month, 1)
    end_date = today
    {start_date, end_date}
  end
end

# ใช้งาน
def send_email(org, to, subject, body) do
  case SaaS.Usage.check_limit(org.id, "emails_sent", plan_limit(org, :emails)) do
    {:ok, _} ->
      SaaS.Usage.track(org.id, "emails_sent")
      deliver_email(to, subject, body)

    {:error, :limit_exceeded} ->
      {:error, "Email limit reached for this month"}
  end
end
```

---

## 6. Organization Onboarding

```elixir
defmodule SaaS.Onboarding do
  def onboard_organization(attrs, plan \\ "free") do
    Ecto.Multi.new()
    |> Ecto.Multi.insert(:org, Organization.changeset(%Organization{}, attrs))
    |> Ecto.Multi.run(:stripe_customer, fn _repo, %{org: org} ->
      SaaS.Billing.StripeService.create_customer(org)
    end)
    |> Ecto.Multi.run(:update_stripe_id, fn repo, %{org: org, stripe_customer: customer} ->
      org
      |> Organization.changeset(%{stripe_customer_id: customer.id})
      |> repo.update()
    end)
    |> Ecto.Multi.run(:start_trial, fn _repo, %{update_stripe_id: org} ->
      start_trial_subscription(org, plan)
    end)
    |> Ecto.Multi.run(:create_admin, fn repo, %{org: org} ->
      create_admin_user(repo, org, attrs)
    end)
    |> Ecto.Multi.run(:send_welcome, fn _repo, %{create_admin: user, org: org} ->
      SaaS.Emails.welcome(user, org) |> SaaS.Mailer.deliver()
    end)
    |> SaaS.Repo.transaction()
  end
end
```

---

## สรุป

```
SaaS Architecture:
├── Subscription Plans (Free/Pro/Enterprise)
├── Stripe integration for payments
├── Webhook handling for subscription events
├── Feature flags per plan
└── Usage tracking and limits

Key Patterns:
├── Stripe webhooks for async payment events
├── Feature gates as Plugs
├── Usage tracking with daily aggregates
└── Multi-step onboarding with Ecto.Multi
```

---

*ก่อนหน้า: [Part 70](part_70.md) | ต่อไป: [Part 72 - Admin Panel](part_72.md)*
