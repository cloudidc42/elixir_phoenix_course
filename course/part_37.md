# Part 37: Email ด้วย Swoosh

## เป้าหมายการเรียนรู้

หลังจากจบ Part นี้ คุณจะสามารถ:
- ส่ง email ด้วย Swoosh
- สร้าง email templates
- Test emails
- ใช้ email services (SendGrid, Mailgun, SES)

---

## 1. ติดตั้ง Swoosh

```elixir
# mix.exs
{:swoosh, "~> 1.16"},
{:finch, "~> 0.18"}  # HTTP adapter

# config/config.exs
config :my_app, MyApp.Mailer,
  adapter: Swoosh.Adapters.Local

config :swoosh, :api_client, Swoosh.ApiClient.Finch

# config/prod.exs
config :my_app, MyApp.Mailer,
  adapter: Swoosh.Adapters.Mailgun,
  api_key: System.get_env("MAILGUN_API_KEY"),
  domain: System.get_env("MAILGUN_DOMAIN")

# หรือ SendGrid
config :my_app, MyApp.Mailer,
  adapter: Swoosh.Adapters.Sendgrid,
  api_key: System.get_env("SENDGRID_API_KEY")

# หรือ AWS SES
config :my_app, MyApp.Mailer,
  adapter: Swoosh.Adapters.AmazonSES,
  region: "ap-southeast-1",
  access_key: System.get_env("AWS_ACCESS_KEY_ID"),
  secret: System.get_env("AWS_SECRET_ACCESS_KEY")

# lib/my_app/mailer.ex
defmodule MyApp.Mailer do
  use Swoosh.Mailer, otp_app: :my_app
end
```

---

## 2. สร้าง Email

```elixir
# lib/my_app/emails/user_emails.ex
defmodule MyApp.Emails.UserEmails do
  use Swoosh.Email

  import Swoosh.Email

  @from_email {"My App", "noreply@myapp.com"}

  def welcome_email(user) do
    new()
    |> from(@from_email)
    |> to({user.name, user.email})
    |> subject("Welcome to My App, #{user.name}!")
    |> text_body("""
    Hi #{user.name},

    Welcome to My App! Your account has been created.

    You can log in at: https://myapp.com/login

    Best regards,
    The My App Team
    """)
    |> html_body("""
    <!DOCTYPE html>
    <html>
    <body>
      <h1>Welcome to My App!</h1>
      <p>Hi #{user.name},</p>
      <p>Your account has been created successfully.</p>
      <a href="https://myapp.com/login">Log In Now</a>
    </body>
    </html>
    """)
  end

  def password_reset_email(user, reset_url) do
    new()
    |> from(@from_email)
    |> to({user.name, user.email})
    |> subject("Reset your My App password")
    |> text_body("Reset your password: #{reset_url}")
    |> html_body("""
    <p>Click <a href="#{reset_url}">here</a> to reset your password.</p>
    <p>This link expires in 1 hour.</p>
    """)
  end

  def order_confirmation_email(user, order) do
    new()
    |> from(@from_email)
    |> to({user.name, user.email})
    |> subject("Order ##{order.id} Confirmed")
    |> add_attachment(generate_pdf_invoice(order))
    |> html_body(order_html(order))
  end

  defp order_html(order) do
    """
    <h1>Order Confirmed!</h1>
    <p>Order ##{order.id}</p>
    <table>
      <tr><th>Item</th><th>Qty</th><th>Price</th></tr>
      #{Enum.map_join(order.items, "\n", fn item ->
        "<tr><td>#{item.name}</td><td>#{item.quantity}</td><td>#{item.price}</td></tr>"
      end)}
    </table>
    <p>Total: #{order.total}</p>
    """
  end

  defp generate_pdf_invoice(_order) do
    # ใช้ Chromic PDF หรือ PDFKit
    %Swoosh.Attachment{filename: "invoice.pdf", data: <<>>, content_type: "application/pdf"}
  end
end
```

---

## 3. Email Templates ด้วย HEEx

```elixir
defmodule MyApp.Emails.UserNotifier do
  import Swoosh.Email

  use Phoenix.Swoosh,
    view: MyAppWeb.EmailView,
    layout: {MyAppWeb.LayoutView, :email}

  @from {"My App", "noreply@myapp.com"}

  def deliver_welcome_instructions(user) do
    new()
    |> from(@from)
    |> to({user.name, user.email})
    |> subject("Welcome to My App!")
    |> render_body("welcome.html", %{user: user})
    |> deliver()
  end

  def deliver_confirmation_instructions(user, url) do
    new()
    |> from(@from)
    |> to({user.name, user.email})
    |> subject("Confirm your My App account")
    |> render_body("confirm_account.html", %{user: user, url: url})
    |> deliver()
  end

  defp deliver(email) do
    MyApp.Mailer.deliver(email)
  end
end
```

```heex
<%# lib/my_app_web/templates/email/welcome.html.heex %>
<!DOCTYPE html>
<html>
<head>
  <style>
    body { font-family: Arial, sans-serif; }
    .container { max-width: 600px; margin: 0 auto; }
    .button {
      background: #4CAF50;
      color: white;
      padding: 12px 24px;
      text-decoration: none;
      border-radius: 4px;
    }
  </style>
</head>
<body>
  <div class="container">
    <h1>Welcome to My App!</h1>
    <p>Hi <%= @user.name %>,</p>
    <p>Thank you for signing up. We're excited to have you!</p>

    <a href="<%= @url %>" class="button">
      Get Started
    </a>

    <p>If the button doesn't work, copy this URL:</p>
    <p><%= @url %></p>

    <hr>
    <p style="color: #888; font-size: 12px;">
      You received this email because you signed up for My App.
    </p>
  </div>
</body>
</html>
```

---

## 4. Sending Emails

```elixir
# ส่งทันที
MyApp.Emails.UserEmails.welcome_email(user)
|> MyApp.Mailer.deliver()

# ส่งด้วย Oban (แนะนำสำหรับ production)
defmodule MyApp.Workers.WelcomeEmail do
  use Oban.Worker, queue: :emails, max_attempts: 3

  @impl true
  def perform(%Oban.Job{args: %{"user_id" => user_id}}) do
    user = MyApp.Accounts.get_user!(user_id)

    case MyApp.Emails.UserNotifier.deliver_welcome_instructions(user) do
      {:ok, _metadata} -> :ok
      {:error, {_status, %{"message" => message}}} -> {:error, message}
    end
  end
end

# หลัง register user
{:ok, user} = MyApp.Accounts.register_user(attrs)
%{user_id: user.id}
|> MyApp.Workers.WelcomeEmail.new()
|> Oban.insert!()
```

---

## 5. Testing Emails

```elixir
# config/test.exs
config :my_app, MyApp.Mailer,
  adapter: Swoosh.Adapters.Test

defmodule MyApp.Emails.UserNotifierTest do
  use MyApp.DataCase

  import Swoosh.TestAssertions

  alias MyApp.Emails.UserNotifier
  alias MyApp.Factory

  test "delivers welcome email" do
    user = insert(:user)

    {:ok, _email} = UserNotifier.deliver_welcome_instructions(user)

    assert_email_sent(
      to: [{"#{user.name}", user.email}],
      subject: "Welcome to My App!"
    )
  end

  test "delivers password reset email" do
    user = insert(:user)
    url = "https://myapp.com/reset/abc123"

    {:ok, _} = UserNotifier.deliver_reset_password_instructions(user, url)

    assert_email_sent fn email ->
      assert email.subject =~ "Reset"
      assert email.html_body =~ url
      assert email.to == [{user.name, user.email}]
    end
  end
end
```

---

## 6. Preview Emails (Development)

Phoenix และ Swoosh มี local inbox ให้ดู emails ใน dev:

```elixir
# lib/my_app_web/router.ex
if Mix.env() == :dev do
  scope "/dev" do
    pipe_through :browser
    forward "/mailbox", Plug.Swoosh.MailboxPreview
  end
end
```

เปิด http://localhost:4000/dev/mailbox เพื่อดู emails ที่ส่งใน dev mode

---

## สรุป

```
Swoosh:
├── Unified email API
├── หลาย adapters: Mailgun, SendGrid, SES, etc.
├── Local adapter สำหรับ development
└── Test adapter สำหรับ testing

Email Best Practices:
├── ส่งผ่าน background jobs
├── Retry ด้วย exponential backoff
├── Text + HTML body
└── Track delivery status

Templates:
├── HEEx templates
├── Layout สำหรับ consistent styling
└── Inline CSS (email clients ต้องการ)
```

---

*ก่อนหน้า: [Part 36](part_36.md) | ต่อไป: [Part 38 - WebSockets และ Real-time](part_38.md)*
