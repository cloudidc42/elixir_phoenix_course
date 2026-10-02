# Part 41: Security (ความปลอดภัย)

## เป้าหมายการเรียนรู้

- เข้าใจการป้องกัน CSRF ใน Phoenix และวิธีที่ framework จัดการให้อัตโนมัติ
- รู้จักการป้องกัน XSS ผ่าน HEEx auto-escaping
- ป้องกัน SQL injection ด้วย Ecto parameterized queries
- สร้าง rate limiting ด้วย Plug
- กำหนด secure headers สำหรับ production
- Hash password อย่างปลอดภัยด้วย bcrypt และ Argon2
- Validate และ sanitize input อย่างถูกต้อง
- เข้าใจ OWASP Top 10 ในบริบทของ Elixir/Phoenix

---

## 1. CSRF Protection ใน Phoenix

CSRF (Cross-Site Request Forgery) คือการโจมตีที่ทำให้ผู้ใช้ส่ง request ที่ไม่ต้องการโดยไม่รู้ตัว Phoenix มีการป้องกันในตัวผ่าน `Phoenix.Controller` และ `Plug.CSRFProtection`

```elixir
# lib/my_app_web/router.ex
defmodule MyAppWeb.Router do
  use MyAppWeb, :router

  pipeline :browser do
    plug :accepts, ["html"]
    plug :fetch_session
    plug :fetch_live_flash
    plug :put_root_layout, html: {MyAppWeb.Layouts, :root}
    plug :protect_from_forgery        # <-- CSRF protection อยู่ที่นี่
    plug :put_secure_browser_headers
  end

  pipeline :api do
    plug :accepts, ["json"]
    # API ไม่ใช้ CSRF แต่ควรใช้ token authentication แทน
  end
end
```

```elixir
# ใน form CSRF token จะถูกเพิ่มอัตโนมัติโดย Phoenix.HTML
# templates/user/new.html.heex
<.form :let={f} for={@changeset} action={~p"/users"}>
  <%# Phoenix เพิ่ม _csrf_token hidden field ให้อัตโนมัติ %>
  <.input field={f[:email]} type="email" label="Email" />
  <.input field={f[:password]} type="password" label="Password" />
  <.button>Register</.button>
</.form>
```

```elixir
# สำหรับ AJAX requests ต้องส่ง CSRF token ใน header
# assets/js/app.js
const csrfToken = document.querySelector("meta[name='csrf-token']").getAttribute("content");
fetch("/api/data", {
  method: "POST",
  headers: {
    "Content-Type": "application/json",
    "X-CSRF-Token": csrfToken
  },
  body: JSON.stringify({ data: "value" })
});
```

```elixir
# ตรวจสอบ CSRF token ด้วยตนเองสำหรับ custom endpoints
defmodule MyAppWeb.WebhookController do
  use MyAppWeb, :controller

  # Webhook จาก external service ไม่ต้องการ CSRF
  # แต่ต้องตรวจสอบ signature แทน
  plug :verify_webhook_signature when action in [:receive]

  def receive(conn, params) do
    # ประมวลผล webhook
    json(conn, %{status: "ok"})
  end

  defp verify_webhook_signature(conn, _opts) do
    signature = get_req_header(conn, "x-signature") |> List.first()
    secret = Application.get_env(:my_app, :webhook_secret)

    expected = :crypto.mac(:hmac, :sha256, secret, conn.assigns[:raw_body])
                |> Base.encode16(case: :lower)

    if Plug.Crypto.secure_compare("sha256=#{expected}", signature || "") do
      conn
    else
      conn
      |> put_status(:unauthorized)
      |> json(%{error: "Invalid signature"})
      |> halt()
    end
  end
end
```

---

## 2. XSS Prevention ด้วย HEEx Auto-escaping

HEEx (HTML+EEx) template engine ของ Phoenix มีการ escape HTML entities อัตโนมัติ ป้องกัน XSS attacks

```elixir
# HEEx escape อัตโนมัติ - ปลอดภัย
# ถ้า @user_input = "<script>alert('XSS')</script>"
# จะแสดงเป็น text ไม่ใช่ execute script
<p><%= @user_input %></p>
# Output: <p>&lt;script&gt;alert('XSS')&lt;/script&gt;</p>
```

```elixir
# Phoenix.HTML.raw/1 - ใช้เมื่อต้องการ render HTML โดยไม่ escape
# ระวัง! ใช้เฉพาะกับ content ที่เชื่อถือได้เท่านั้น
<p><%= Phoenix.HTML.raw(@trusted_html_content) %></p>

# วิธีที่ดีกว่าคือ sanitize ก่อน render
defmodule MyAppWeb.Helpers do
  def safe_html(content) do
    content
    |> HtmlSanitizeEx.sanitize()          # ใช้ library HtmlSanitizeEx
    |> Phoenix.HTML.raw()
  end
end
```

```elixir
# เพิ่ม HtmlSanitizeEx ใน mix.exs
# mix.exs
defp deps do
  [
    {:html_sanitize_ex, "~> 1.4"}
  ]
end
```

```elixir
# กำหนด allowed tags สำหรับ rich text content
defmodule MyApp.Sanitizer do
  @doc """
  Sanitize HTML content โดยอนุญาตเฉพาะ tags ที่ปลอดภัย
  """
  def sanitize_rich_text(html) do
    html
    |> HtmlSanitizeEx.Scrubber.scrub(MyApp.Scrubber)
  end

  def sanitize_plain_text(text) do
    HtmlSanitizeEx.strip_tags(text)
  end
end

# กำหนด custom scrubber
defmodule MyApp.Scrubber do
  require HtmlSanitizeEx.Scrubber.Meta
  alias HtmlSanitizeEx.Scrubber.Meta

  # อนุญาต tags เหล่านี้
  Meta.allow_tag_with_these_attributes("p", [])
  Meta.allow_tag_with_these_attributes("br", [])
  Meta.allow_tag_with_these_attributes("strong", [])
  Meta.allow_tag_with_these_attributes("em", [])
  Meta.allow_tag_with_these_attributes("ul", [])
  Meta.allow_tag_with_these_attributes("ol", [])
  Meta.allow_tag_with_these_attributes("li", [])
  Meta.allow_tag_with_these_attributes("a", ["href", "title"])

  # ไม่อนุญาต script, iframe, object
  Meta.strip_everything_not_covered()
end
```

```elixir
# Content Security Policy ป้องกัน XSS ระดับ browser
# lib/my_app_web/plugs/content_security_policy.ex
defmodule MyAppWeb.Plugs.ContentSecurityPolicy do
  import Plug.Conn

  def init(opts), do: opts

  def call(conn, _opts) do
    conn
    |> put_resp_header("content-security-policy", build_csp())
  end

  defp build_csp do
    [
      "default-src 'self'",
      "script-src 'self' 'nonce-#{generate_nonce()}'",
      "style-src 'self' 'unsafe-inline'",
      "img-src 'self' data: https:",
      "font-src 'self'",
      "connect-src 'self' wss:",
      "frame-ancestors 'none'",
      "base-uri 'self'",
      "form-action 'self'"
    ]
    |> Enum.join("; ")
  end

  defp generate_nonce do
    :crypto.strong_rand_bytes(16) |> Base.encode64()
  end
end
```

---

## 3. SQL Injection Prevention ด้วย Ecto

Ecto ใช้ parameterized queries โดยอัตโนมัติ ทำให้ป้องกัน SQL injection ได้

```elixir
# ปลอดภัย - Ecto parameterize query อัตโนมัติ
defmodule MyApp.Accounts do
  import Ecto.Query
  alias MyApp.Repo
  alias MyApp.Accounts.User

  def get_user_by_email(email) do
    Repo.get_by(User, email: email)
    # SQL: SELECT * FROM users WHERE email = $1
    # parameter $1 = email (ไม่มีการ interpolate โดยตรง)
  end

  def search_users(search_term) do
    pattern = "%#{search_term}%"

    from(u in User,
      where: ilike(u.name, ^pattern) or ilike(u.email, ^pattern),
      order_by: u.name
    )
    |> Repo.all()
    # SQL: SELECT * FROM users WHERE name ILIKE $1 OR email ILIKE $1
  end
end
```

```elixir
# อันตราย! อย่าทำแบบนี้ - Direct string interpolation
defmodule MyApp.UnsafeQuery do
  alias MyApp.Repo

  # อันตราย - SQL injection ได้
  def unsafe_search(term) do
    Repo.query("SELECT * FROM users WHERE name = '#{term}'")
  end
end

# ถูกต้อง - ใช้ parameterized query
defmodule MyApp.SafeQuery do
  alias MyApp.Repo

  def safe_search(term) do
    Repo.query("SELECT * FROM users WHERE name = $1", [term])
  end
end
```

```elixir
# Fragment ที่ปลอดภัยสำหรับ dynamic SQL
defmodule MyApp.Reports do
  import Ecto.Query

  def get_stats(table_name, column) when table_name in ["orders", "products"] do
    # Whitelist table names - ไม่เอา user input โดยตรง
    from(u in User,
      join: o in ^table_name, on: o.user_id == u.id,
      select: {u.id, fragment("COUNT(?)", field(o, ^column))}
    )
    |> Repo.all()
  end

  # ตรวจสอบ column names ผ่าน atom (ปลอดภัยกว่า string)
  def order_by_safe(query, :name), do: order_by(query, [u], u.name)
  def order_by_safe(query, :email), do: order_by(query, [u], u.email)
  def order_by_safe(query, :inserted_at), do: order_by(query, [u], u.inserted_at)
  def order_by_safe(query, _), do: query  # default - ignore unknown fields
end
```

---

## 4. Rate Limiting ด้วย Plug

Rate limiting ป้องกัน brute force attacks และ DoS attacks

```elixir
# เพิ่ม dependencies
# mix.exs
defp deps do
  [
    {:hammer, "~> 6.1"},
    {:hammer_backend_redis, "~> 6.1"}  # optional - สำหรับ distributed rate limiting
  ]
end
```

```elixir
# config/config.exs
config :hammer,
  backend: {Hammer.Backend.ETS, [expiry_ms: 60_000 * 60 * 4, cleanup_interval_ms: 60_000]}

# สำหรับ Redis backend
# config :hammer,
#   backend: {Hammer.Backend.Redis, [redix_config: [host: "localhost"]]}
```

```elixir
# lib/my_app_web/plugs/rate_limiter.ex
defmodule MyAppWeb.Plugs.RateLimiter do
  import Plug.Conn
  require Logger

  def init(opts) do
    %{
      limit: Keyword.get(opts, :limit, 100),
      period_ms: Keyword.get(opts, :period_ms, 60_000),
      bucket_name: Keyword.get(opts, :bucket_name, "default")
    }
  end

  def call(conn, %{limit: limit, period_ms: period_ms, bucket_name: bucket_name}) do
    client_ip = get_client_ip(conn)
    bucket_key = "#{bucket_name}:#{client_ip}"

    case Hammer.check_rate(bucket_key, period_ms, limit) do
      {:allow, _count} ->
        conn

      {:deny, _limit} ->
        Logger.warning("Rate limit exceeded for IP: #{client_ip}")

        conn
        |> put_resp_header("retry-after", "60")
        |> put_status(:too_many_requests)
        |> Phoenix.Controller.json(%{error: "Too many requests. Please try again later."})
        |> halt()
    end
  end

  defp get_client_ip(conn) do
    # ตรวจสอบ X-Forwarded-For header สำหรับ load balancer
    case get_req_header(conn, "x-forwarded-for") do
      [forwarded | _] ->
        forwarded |> String.split(",") |> List.first() |> String.trim()
      [] ->
        conn.remote_ip |> Tuple.to_list() |> Enum.join(".")
    end
  end
end
```

```elixir
# ใช้ rate limiter ใน router
defmodule MyAppWeb.Router do
  use MyAppWeb, :router

  pipeline :api do
    plug :accepts, ["json"]
    plug MyAppWeb.Plugs.RateLimiter, limit: 100, period_ms: 60_000
  end

  pipeline :auth do
    plug :accepts, ["json"]
    # เข้มงวดกว่าสำหรับ authentication endpoints
    plug MyAppWeb.Plugs.RateLimiter, limit: 5, period_ms: 60_000, bucket_name: "auth"
  end

  scope "/api", MyAppWeb do
    pipe_through :api

    resources "/posts", PostController
  end

  scope "/auth", MyAppWeb do
    pipe_through :auth

    post "/login", SessionController, :create
    post "/register", RegistrationController, :create
  end
end
```

```elixir
# Rate limiter ระดับ per-user (หลัง authentication)
defmodule MyAppWeb.Plugs.UserRateLimiter do
  import Plug.Conn

  def init(opts), do: opts

  def call(conn, opts) do
    user_id = conn.assigns[:current_user]?.id

    if user_id do
      limit = Keyword.get(opts, :limit, 1000)
      period_ms = Keyword.get(opts, :period_ms, 60_000)

      case Hammer.check_rate("user:#{user_id}", period_ms, limit) do
        {:allow, _count} -> conn
        {:deny, _limit} ->
          conn
          |> put_status(:too_many_requests)
          |> Phoenix.Controller.json(%{error: "Rate limit exceeded"})
          |> halt()
      end
    else
      conn
    end
  end
end
```

---

## 5. Secure Headers

```elixir
# lib/my_app_web/plugs/secure_headers.ex
defmodule MyAppWeb.Plugs.SecureHeaders do
  import Plug.Conn

  def init(opts), do: opts

  def call(conn, _opts) do
    conn
    # ป้องกัน clickjacking
    |> put_resp_header("x-frame-options", "DENY")
    # ป้องกัน MIME type sniffing
    |> put_resp_header("x-content-type-options", "nosniff")
    # XSS protection (legacy browsers)
    |> put_resp_header("x-xss-protection", "1; mode=block")
    # ให้ใช้ HTTPS เสมอ
    |> put_resp_header("strict-transport-security", "max-age=31536000; includeSubDomains")
    # Referrer policy
    |> put_resp_header("referrer-policy", "strict-origin-when-cross-origin")
    # Permissions policy
    |> put_resp_header("permissions-policy", "camera=(), microphone=(), geolocation=()")
  end
end
```

```elixir
# Phoenix มี put_secure_browser_headers/2 ที่ตั้งค่า headers พื้นฐาน
# แต่เราสามารถ override ได้
defmodule MyAppWeb.Router do
  pipeline :browser do
    plug :accepts, ["html"]
    plug :fetch_session
    plug :fetch_live_flash
    plug :put_root_layout, html: {MyAppWeb.Layouts, :root}
    plug :protect_from_forgery
    plug :put_secure_browser_headers, %{
      "content-security-policy" =>
        "default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline'"
    }
    plug MyAppWeb.Plugs.SecureHeaders
  end
end
```

```elixir
# config/prod.exs - HTTPS configuration
config :my_app, MyAppWeb.Endpoint,
  https: [
    port: 443,
    cipher_suite: :strong,
    keyfile: System.get_env("SSL_KEY_PATH"),
    certfile: System.get_env("SSL_CERT_PATH")
  ],
  force_ssl: [rewrite_on: [:x_forwarded_proto]]
```

---

## 6. Password Hashing

```elixir
# mix.exs - เลือก bcrypt หรือ argon2
defp deps do
  [
    # Option 1: bcrypt (เร็วกว่า, ใช้งานง่าย)
    {:bcrypt_elixir, "~> 3.0"},

    # Option 2: argon2 (แนะนำกว่าสำหรับ new projects)
    {:argon2_elixir, "~> 4.0"}
  ]
end
```

```elixir
# lib/my_app/accounts/user.ex
defmodule MyApp.Accounts.User do
  use Ecto.Schema
  import Ecto.Changeset

  schema "users" do
    field :email, :string
    field :password, :string, virtual: true  # ไม่ save ลง DB
    field :password_confirmation, :string, virtual: true
    field :hashed_password, :string

    timestamps()
  end

  def registration_changeset(user, attrs) do
    user
    |> cast(attrs, [:email, :password, :password_confirmation])
    |> validate_required([:email, :password])
    |> validate_email()
    |> validate_password()
    |> hash_password()
  end

  def password_changeset(user, attrs) do
    user
    |> cast(attrs, [:password, :password_confirmation])
    |> validate_required([:password])
    |> validate_password()
    |> hash_password()
  end

  defp validate_email(changeset) do
    changeset
    |> validate_format(:email, ~r/^[^\s]+@[^\s]+\.[^\s]+$/, message: "must have the @ sign and no spaces")
    |> validate_length(:email, max: 254)
    |> unsafe_validate_unique(:email, MyApp.Repo)
    |> unique_constraint(:email)
    |> update_change(:email, &String.downcase/1)
  end

  defp validate_password(changeset) do
    changeset
    |> validate_required([:password])
    |> validate_length(:password, min: 12, max: 72)
    |> validate_format(:password, ~r/[0-9]/, message: "must contain at least one number")
    |> validate_format(:password, ~r/[A-Z]/, message: "must contain at least one uppercase letter")
    |> validate_format(:password, ~r/[a-z]/, message: "must contain at least one lowercase letter")
    |> validate_confirmation(:password, message: "does not match password")
  end

  defp hash_password(changeset) do
    password = get_change(changeset, :password)

    if password && changeset.valid? do
      changeset
      # ใช้ Argon2 สำหรับ hashing
      |> put_change(:hashed_password, Argon2.hash_pwd_salt(password))
      # ลบ plain text password ออกจาก changeset
      |> delete_change(:password)
      |> delete_change(:password_confirmation)
    else
      changeset
    end
  end

  @doc "ตรวจสอบ password"
  def valid_password?(%__MODULE__{hashed_password: hashed_password}, password)
      when is_binary(hashed_password) and byte_size(password) > 0 do
    Argon2.verify_pass(password, hashed_password)
  end

  def valid_password?(_, _) do
    # Dummy check เพื่อป้องกัน timing attacks
    Argon2.no_user_verify()
    false
  end
end
```

```elixir
# bcrypt version
defmodule MyApp.Accounts.UserBcrypt do
  defp hash_password(changeset) do
    password = get_change(changeset, :password)

    if password && changeset.valid? do
      changeset
      |> put_change(:hashed_password, Bcrypt.hash_pwd_salt(password))
      |> delete_change(:password)
    else
      changeset
    end
  end

  def valid_password?(%{hashed_password: hash}, password) do
    Bcrypt.verify_pass(password, hash)
  end

  def valid_password?(_, _) do
    Bcrypt.no_user_verify()
    false
  end
end
```

---

## 7. Input Validation and Sanitization

```elixir
# lib/my_app/accounts/user.ex - comprehensive validation
defmodule MyApp.Inputs do
  import Ecto.Changeset

  @doc "Validate และ sanitize username"
  def validate_username(changeset, field) do
    changeset
    |> validate_required([field])
    |> validate_length(field, min: 3, max: 30)
    # อนุญาตเฉพาะ alphanumeric และ underscore
    |> validate_format(field, ~r/^[a-zA-Z0-9_]+$/,
        message: "can only contain letters, numbers, and underscores")
    |> update_change(field, &String.downcase/1)
  end

  @doc "Validate URL"
  def validate_url(changeset, field) do
    changeset
    |> validate_change(field, fn _, url ->
      case URI.parse(url) do
        %URI{scheme: scheme, host: host}
        when scheme in ["http", "https"] and not is_nil(host) ->
          []
        _ ->
          [{field, "must be a valid HTTP/HTTPS URL"}]
      end
    end)
  end

  @doc "Validate phone number (Thai format)"
  def validate_phone(changeset, field) do
    changeset
    |> validate_format(field, ~r/^(\+66|0)[0-9]{8,9}$/,
        message: "must be a valid Thai phone number")
    |> update_change(field, &normalize_phone/1)
  end

  defp normalize_phone("0" <> rest), do: "+66" <> rest
  defp normalize_phone(phone), do: phone

  @doc "Sanitize HTML ให้ปลอดภัย"
  def sanitize_html_input(changeset, field, opts \\ []) do
    changeset
    |> update_change(field, fn value ->
      if value do
        case Keyword.get(opts, :allow_html, false) do
          true -> HtmlSanitizeEx.sanitize(value)
          false -> HtmlSanitizeEx.strip_tags(value)
        end
      end
    end)
  end
end
```

```elixir
# ป้องกัน mass assignment ด้วยการระบุ fields ที่อนุญาตเท่านั้น
defmodule MyApp.Accounts.Profile do
  use Ecto.Schema
  import Ecto.Changeset

  schema "profiles" do
    field :display_name, :string
    field :bio, :string
    field :website, :string
    field :is_admin, :boolean  # ไม่ควร allow user อัปเดต
    field :verified, :boolean  # ไม่ควร allow user อัปเดต
    belongs_to :user, MyApp.Accounts.User
    timestamps()
  end

  # User changeset - ระบุเฉพาะ fields ที่ user อัปเดตได้
  def user_changeset(profile, attrs) do
    profile
    |> cast(attrs, [:display_name, :bio, :website])  # ไม่รวม :is_admin, :verified
    |> validate_required([:display_name])
    |> validate_length(:display_name, min: 1, max: 100)
    |> validate_length(:bio, max: 500)
    |> MyApp.Inputs.validate_url(:website)
  end

  # Admin changeset - สามารถอัปเดต fields เพิ่มเติมได้
  def admin_changeset(profile, attrs) do
    profile
    |> cast(attrs, [:display_name, :bio, :website, :is_admin, :verified])
    |> validate_required([:display_name])
  end
end
```

---

## 8. OWASP Top 10 สำหรับ Elixir/Phoenix

```elixir
# A01: Broken Access Control
# ใช้ authorization middleware ตรวจสอบทุก request

defmodule MyAppWeb.Plugs.RequireAuth do
  import Plug.Conn
  import Phoenix.Controller

  def init(opts), do: opts

  def call(conn, _opts) do
    if conn.assigns[:current_user] do
      conn
    else
      conn
      |> put_flash(:error, "You must be logged in to access this page.")
      |> redirect(to: "/login")
      |> halt()
    end
  end
end

defmodule MyAppWeb.Plugs.RequireAdmin do
  import Plug.Conn
  import Phoenix.Controller

  def init(opts), do: opts

  def call(conn, _opts) do
    case conn.assigns[:current_user] do
      %{is_admin: true} -> conn
      _ ->
        conn
        |> put_status(:forbidden)
        |> json(%{error: "Access denied"})
        |> halt()
    end
  end
end
```

```elixir
# A02: Cryptographic Failures
# ใช้ strong encryption สำหรับ sensitive data

defmodule MyApp.Crypto do
  @aad "AES256GCM"

  def encrypt(plaintext, key \\ get_key()) do
    iv = :crypto.strong_rand_bytes(16)
    {ciphertext, tag} = :crypto.crypto_one_time_aead(
      :aes_256_gcm, key, iv, plaintext, @aad, true
    )
    iv <> tag <> ciphertext
  end

  def decrypt(ciphertext, key \\ get_key()) do
    <<iv::binary-16, tag::binary-16, ciphertext::binary>> = ciphertext
    :crypto.crypto_one_time_aead(
      :aes_256_gcm, key, iv, ciphertext, @aad, tag, false
    )
  end

  defp get_key do
    Application.get_env(:my_app, :encryption_key)
    |> Base.decode64!()
  end
end
```

```elixir
# A03: Injection - ใช้ Ecto parameterized queries (ดูหัวข้อ 3)

# A05: Security Misconfiguration
# config/prod.exs
config :my_app, MyAppWeb.Endpoint,
  secret_key_base: System.fetch_env!("SECRET_KEY_BASE"),  # ใช้ env var
  # อย่า hardcode secrets ใน config files

# ตรวจสอบ secrets ไม่ถูก commit ด้วย mix deps.compile --force
# หรือใช้ tools เช่น git-secrets, detect-secrets

# A06: Vulnerable Components
# อัปเดต dependencies เป็นประจำ
# mix hex.audit   # ตรวจสอบ security advisories
# mix deps.update --all

# A07: Identification and Authentication Failures
defmodule MyApp.Accounts do
  def authenticate_user(email, password) do
    user = get_user_by_email(email)

    # ใช้ constant-time comparison ป้องกัน timing attacks
    cond do
      user && MyApp.Accounts.User.valid_password?(user, password) ->
        {:ok, user}
      true ->
        # ต้องทำ dummy check แม้ user ไม่มีอยู่
        Argon2.no_user_verify()
        {:error, :invalid_credentials}
    end
  end

  def generate_session_token(user) do
    # ใช้ secure random token
    token = :crypto.strong_rand_bytes(32) |> Base.url_encode64()
    # บันทึก token hash ลง DB
    hashed = :crypto.hash(:sha256, token) |> Base.encode16()

    UserToken.create_token!(user, "session", hashed)
    token
  end
end
```

```elixir
# A09: Security Logging and Monitoring
defmodule MyApp.SecurityLogger do
  require Logger

  def log_login_attempt(email, success, ip) do
    level = if success, do: :info, else: :warning

    Logger.log(level, "Login attempt",
      email: email,
      success: success,
      ip: ip,
      timestamp: DateTime.utc_now()
    )
  end

  def log_permission_denied(user_id, resource, action, ip) do
    Logger.warning("Permission denied",
      user_id: user_id,
      resource: resource,
      action: action,
      ip: ip
    )
  end

  def log_suspicious_activity(description, metadata) do
    Logger.error("Suspicious activity: #{description}",
      Keyword.merge(metadata, [timestamp: DateTime.utc_now()])
    )
    # ส่ง alert ไปยัง monitoring system (Sentry, PagerDuty, etc.)
  end
end
```

---

## สรุป

```
Security Layers ใน Phoenix Application:
┌─────────────────────────────────────────────────────────┐
│ Network Layer                                           │
│   • HTTPS/TLS (force_ssl)                              │
│   • HSTS headers                                       │
├─────────────────────────────────────────────────────────┤
│ Application Layer (Plugs)                               │
│   • CSRF Protection (protect_from_forgery)             │
│   • Rate Limiting (Hammer)                             │
│   • Secure Headers (CSP, X-Frame-Options, etc.)        │
│   • Authentication (session/token)                     │
│   • Authorization (role-based)                         │
├─────────────────────────────────────────────────────────┤
│ Business Logic Layer                                    │
│   • Input Validation (Ecto Changesets)                 │
│   • HTML Sanitization (HtmlSanitizeEx)                 │
│   • Password Hashing (Argon2/Bcrypt)                   │
├─────────────────────────────────────────────────────────┤
│ Data Layer                                              │
│   • SQL Injection Prevention (Ecto parameterized)      │
│   • Encryption at rest (AES-256-GCM)                   │
│   • Audit logging                                      │
└─────────────────────────────────────────────────────────┘

Security Checklist:
✓ CSRF tokens ใน forms
✓ HEEx auto-escaping สำหรับ XSS prevention
✓ Ecto parameterized queries สำหรับ SQL injection
✓ Rate limiting บน sensitive endpoints
✓ Secure HTTP headers
✓ Argon2/Bcrypt สำหรับ password hashing
✓ Input validation ใน changesets
✓ Authorization checks ทุก endpoint
✓ Secrets ใน environment variables
✓ HTTPS enforced ใน production
```

---

*ก่อนหน้า: [Part 40](part_40.md) | ต่อไป: [Part 42 - Testing Advanced](part_42.md)*
