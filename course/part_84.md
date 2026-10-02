# Part 84: Email System Advanced (ระบบ Email ขั้นสูง)

## เป้าหมายการเรียนรู้

หลังจากจบ Part นี้ คุณจะสามารถ:
- สร้าง email templates ที่สวยงาม
- Transactional email ด้วย Resend/SendGrid/SES
- Email tracking (open, click)
- Unsubscribe management

---

## 1. Swoosh Email Templates

```elixir
# mix.exs: {:swoosh, "~> 1.16"}, {:mjml_eex, "~> 0.7"}

defmodule MyApp.Emails.WelcomeEmail do
  import Swoosh.Email

  def build(user) do
    new()
    |> to({user.name, user.email})
    |> from({"MyApp", "hello@myapp.com"})
    |> subject("ยินดีต้อนรับสู่ MyApp!")
    |> html_body(html(user))
    |> text_body(text(user))
  end

  defp html(user) do
    """
    <!DOCTYPE html>
    <html>
    <head>
      <meta charset="utf-8">
      <style>
        body { font-family: sans-serif; color: #333; }
        .container { max-width: 600px; margin: 0 auto; padding: 20px; }
        .button { background: #3b82f6; color: white; padding: 12px 24px;
                  text-decoration: none; border-radius: 6px; display: inline-block; }
        .footer { color: #999; font-size: 12px; margin-top: 40px; }
      </style>
    </head>
    <body>
      <div class="container">
        <h1>สวัสดี #{user.name}!</h1>
        <p>ยินดีต้อนรับสู่ MyApp ขอบคุณที่สมัครสมาชิก</p>
        <p>เริ่มต้นใช้งานได้เลย:</p>
        <p>
          <a href="https://myapp.com/dashboard" class="button">
            ไปยัง Dashboard
          </a>
        </p>
        <p>ถ้ามีข้อสงสัยสามารถตอบกลับ email นี้ได้เลย</p>
        <div class="footer">
          <p>MyApp &bull; <a href="https://myapp.com/unsubscribe?token=#{user.unsubscribe_token}">ยกเลิกรับ email</a></p>
        </div>
      </div>
    </body>
    </html>
    """
  end

  defp text(user) do
    """
    สวัสดี #{user.name}!

    ยินดีต้อนรับสู่ MyApp

    ไปยัง Dashboard: https://myapp.com/dashboard

    ยกเลิกรับ email: https://myapp.com/unsubscribe?token=#{user.unsubscribe_token}
    """
  end
end
```

---

## 2. Mailer Configuration

```elixir
# config/runtime.exs
config :my_app, MyApp.Mailer,
  adapter: Swoosh.Adapters.Resend,
  api_key: System.fetch_env!("RESEND_API_KEY")

# หรือ SendGrid
config :my_app, MyApp.Mailer,
  adapter: Swoosh.Adapters.Sendgrid,
  api_key: System.fetch_env!("SENDGRID_API_KEY")

# lib/my_app/mailer.ex
defmodule MyApp.Mailer do
  use Swoosh.Mailer, otp_app: :my_app
end

# Async email sending
defmodule MyApp.Emails do
  def send_welcome(user) do
    MyApp.Emails.WelcomeEmail.build(user)
    |> MyApp.Mailer.deliver_later()
  end

  def send_password_reset(user, token) do
    MyApp.Emails.PasswordResetEmail.build(user, token)
    |> MyApp.Mailer.deliver_now()  # synchronous for critical emails
  end
end
```

---

## 3. Email Tracking

```elixir
defmodule MyApp.EmailTracker do
  alias MyApp.Repo
  alias MyApp.Email.{Log, TrackPixel}

  def track_email(email, user_id, email_type) do
    {:ok, log} = Repo.insert(%Log{
      user_id: user_id,
      email_type: email_type,
      to: email.to |> List.first() |> elem(1),
      subject: email.subject
    })

    # Add 1px tracking pixel
    tracking_url = MyAppWeb.Router.Helpers.email_tracking_url(
      MyAppWeb.Endpoint, :pixel, log.id
    )

    pixel_html = ~s(<img src="#{tracking_url}" width="1" height="1" style="display:none">)

    html_body = email.html_body <> pixel_html
    %{email | html_body: html_body}
  end

  def record_open(log_id) do
    Repo.get!(Log, log_id)
    |> Ecto.Changeset.change(%{
      opened_at: DateTime.utc_now(),
      open_count: Repo.one(from l in Log, where: l.id == ^log_id, select: l.open_count) + 1
    })
    |> Repo.update()
  end
end

# Controller
defmodule MyAppWeb.EmailTrackingController do
  use MyAppWeb, :controller

  def pixel(conn, %{"id" => id}) do
    MyApp.EmailTracker.record_open(id)

    # Return 1x1 transparent GIF
    pixel = Base.decode64!("R0lGODlhAQABAIAAAAAAAP///yH5BAEAAAAALAAAAAABAAEAAAIBRAA7")

    conn
    |> put_resp_content_type("image/gif")
    |> send_resp(200, pixel)
  end
end
```

---

## 4. Email Queue with Oban

```elixir
defmodule MyApp.Workers.EmailWorker do
  use Oban.Worker, queue: :emails, max_attempts: 3

  @impl Oban.Worker
  def perform(%Oban.Job{args: %{"type" => type} = args}) do
    case type do
      "welcome" ->
        user = MyApp.Accounts.get_user!(args["user_id"])
        MyApp.Emails.WelcomeEmail.build(user)
        |> MyApp.Mailer.deliver()

      "password_reset" ->
        user = MyApp.Accounts.get_user!(args["user_id"])
        MyApp.Emails.PasswordResetEmail.build(user, args["token"])
        |> MyApp.Mailer.deliver()

      "order_confirmation" ->
        order = MyApp.Orders.get_order!(args["order_id"]) |> MyApp.Repo.preload([:user, :items])
        MyApp.Emails.OrderEmail.build(order)
        |> MyApp.Mailer.deliver()
    end
  end

  @impl Oban.Worker
  def timeout(_), do: :timer.seconds(30)
end

# Enqueue email
def send_welcome_async(user) do
  %{"type" => "welcome", "user_id" => user.id}
  |> MyApp.Workers.EmailWorker.new()
  |> Oban.insert!()
end
```

---

## 5. Unsubscribe Management

```elixir
defmodule MyApp.EmailPreferences do
  alias MyApp.Repo
  import Ecto.Query

  @email_types ~w(marketing weekly_digest product_updates)

  def unsubscribe(user_id, type \\ :all) do
    user = Repo.get!(MyApp.User, user_id)

    unsubscribed = case type do
      :all -> @email_types
      specific -> [to_string(specific)]
    end

    user
    |> Ecto.Changeset.change(%{
      unsubscribed_emails: Enum.uniq((user.unsubscribed_emails || []) ++ unsubscribed)
    })
    |> Repo.update()
  end

  def subscribed?(user, email_type) do
    not (to_string(email_type) in (user.unsubscribed_emails || []))
  end

  def generate_unsubscribe_token(user) do
    Phoenix.Token.sign(MyAppWeb.Endpoint, "unsubscribe", user.id, max_age: 30 * 24 * 60 * 60)
  end

  def verify_unsubscribe_token(token) do
    Phoenix.Token.verify(MyAppWeb.Endpoint, "unsubscribe", token, max_age: 30 * 24 * 60 * 60)
  end
end

# Controller
defmodule MyAppWeb.UnsubscribeController do
  use MyAppWeb, :controller

  def unsubscribe(conn, %{"token" => token} = params) do
    case MyApp.EmailPreferences.verify_unsubscribe_token(token) do
      {:ok, user_id} ->
        type = Map.get(params, "type", :all)
        MyApp.EmailPreferences.unsubscribe(user_id, type)
        render(conn, :success, message: "ยกเลิกรับ email เรียบร้อยแล้ว")

      {:error, _} ->
        render(conn, :error, message: "ลิงก์ไม่ถูกต้องหรือหมดอายุ")
    end
  end
end
```

---

## สรุป

```
Email System:
├── Swoosh: universal email library
├── Adapters: Resend, SendGrid, SES, SMTP
├── deliver_now: synchronous
└── deliver_later: async via GenServer

Transactional Emails:
├── Welcome, password reset, receipts
├── HTML + plain text versions
└── Queue with Oban for reliability

Tracking:
├── Pixel tracking: detect opens
├── Click tracking: redirect through app
└── Log opens/clicks in DB

Compliance:
├── Unsubscribe link in every email
├── Token-based unsubscribe
└── Honor preferences before sending
```

---

*ก่อนหน้า: [Part 83](part_83.md) | ต่อไป: [Part 85 - Authentication Advanced](part_85.md)*
