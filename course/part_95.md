# Part 95: Security Best Practices (แนวทางความปลอดภัย)

## เป้าหมายการเรียนรู้

หลังจากจบ Part นี้ คุณจะสามารถ:
- OWASP Top 10 prevention ใน Elixir/Phoenix
- SQL injection prevention
- XSS protection
- CSRF, Content Security Policy
- Secrets management

---

## 1. SQL Injection Prevention

```elixir
# BAD: Never interpolate into queries
def bad_search(term) do
  Repo.query!("SELECT * FROM users WHERE name = '#{term}'")
end

# GOOD: Always use parameterized queries via Ecto
def good_search(term) do
  from(u in User, where: u.name == ^term) |> Repo.all()
end

# GOOD: Raw SQL with parameters
def raw_search(term) do
  Repo.query!("SELECT * FROM users WHERE name = $1", [term])
end

# Fragment with parameters
def search_by_expression(col, val) do
  import Ecto.Query
  from(u in User,
    where: fragment("lower(?) LIKE lower(?)", field(u, ^col), ^"%#{val}%")
  )
end

# Dynamic queries - safe way
def build_filter(filters) do
  import Ecto.Query
  Enum.reduce(filters, from(u in User), fn
    {:name, val}, query -> where(query, [u], ilike(u.name, ^"%#{val}%"))
    {:role, val}, query -> where(query, [u], u.role == ^val)
    {:min_age, val}, query -> where(query, [u], u.age >= ^val)
    _, query -> query
  end)
end
```

---

## 2. XSS Prevention

```elixir
# Phoenix templates auto-escape by default
# In .heex templates:
def render(assigns) do
  ~H"""
  <p><%= @user_input %></p>  <!-- Auto-escaped: safe -->
  <p><%= raw(@safe_html) %></p>  <!-- Use only for trusted HTML -->
  """
end

# Never use raw() with user input
# WRONG:
~H"<%= raw(@user.bio) %>"

# SAFE: Sanitize if you must render HTML
def sanitize_html(html) do
  # Use HtmlSanitizeEx or similar
  HtmlSanitizeEx.basic_html(html)
end

# Content Security Policy header
defmodule MyAppWeb.Plugs.ContentSecurityPolicy do
  import Plug.Conn

  @nonce_header "x-csp-nonce"

  def init(opts), do: opts

  def call(conn, _opts) do
    nonce = :crypto.strong_rand_bytes(16) |> Base.url_encode64(padding: false)

    conn
    |> assign(:csp_nonce, nonce)
    |> put_resp_header("content-security-policy", build_csp(nonce))
  end

  defp build_csp(nonce) do
    [
      "default-src 'self'",
      "script-src 'self' 'nonce-#{nonce}'",
      "style-src 'self' 'unsafe-inline' fonts.googleapis.com",
      "img-src 'self' data: https:",
      "font-src 'self' fonts.gstatic.com",
      "connect-src 'self' wss:",
      "frame-ancestors 'none'",
      "base-uri 'self'"
    ]
    |> Enum.join("; ")
  end
end
```

---

## 3. CSRF Protection

```elixir
# Phoenix includes CSRF protection by default
# endpoint.ex
plug Plug.Session, ...
plug :fetch_session
plug :protect_from_forgery  # CSRF token required on POST/PUT/DELETE

# In forms - Phoenix automatically includes CSRF token
~H"""
<.form for={@form} action={~p"/settings"}>
  <!-- Phoenix adds hidden _csrf_token field automatically -->
  <.input field={@form[:name]} />
  <.button>Save</.button>
</.form>
"""

# For API endpoints - verify CSRF token manually or use token auth
# router.ex
pipeline :api do
  plug :accepts, ["json"]
  # No CSRF for API - use Bearer token auth instead
end

pipeline :browser do
  plug :accepts, ["html"]
  plug :fetch_session
  plug :fetch_live_flash
  plug :put_root_layout, html: {MyAppWeb.Layouts, :root}
  plug :protect_from_forgery   # CSRF enabled
  plug :put_secure_browser_headers
end
```

---

## 4. Secrets Management

```elixir
# NEVER commit secrets to git
# config/runtime.exs - read at runtime, not compile time
import Config

config :my_app, MyApp.Repo,
  url: System.fetch_env!("DATABASE_URL")

config :my_app, :stripe,
  secret_key: System.fetch_env!("STRIPE_SECRET_KEY"),
  webhook_secret: System.fetch_env!("STRIPE_WEBHOOK_SECRET")

config :my_app, MyAppWeb.Endpoint,
  secret_key_base: System.fetch_env!("SECRET_KEY_BASE")

# Generate secrets:
# mix phx.gen.secret     # 64-char base64 secret
# openssl rand -hex 32   # 32-byte hex secret

# .gitignore
# .env
# config/*.secret.exs

# Verify secrets exist on startup
defmodule MyApp.Config do
  @required_secrets ~w[
    DATABASE_URL
    SECRET_KEY_BASE
    STRIPE_SECRET_KEY
  ]

  def validate! do
    missing = Enum.filter(@required_secrets, fn key ->
      System.get_env(key) in [nil, ""]
    end)

    if missing != [] do
      raise "Missing required environment variables: #{Enum.join(missing, ", ")}"
    end
  end
end

# In application.ex
def start(_type, _args) do
  MyApp.Config.validate!()
  # ...
end
```

---

## 5. Mass Assignment Protection

```elixir
defmodule MyApp.Accounts.User do
  use Ecto.Schema
  import Ecto.Changeset

  schema "users" do
    field :email, :string
    field :name, :string
    field :password, :string, virtual: true
    field :password_hash, :string
    field :role, :string, default: "user"   # never let users set this
    field :is_admin, :boolean, default: false  # never let users set this
    timestamps()
  end

  # Public changeset - limited fields
  def changeset(user, attrs) do
    user
    |> cast(attrs, [:email, :name, :password])  # only safe fields
    |> validate_required([:email])
    |> validate_length(:password, min: 8)
    |> hash_password()
  end

  # Admin-only changeset
  def admin_changeset(user, attrs) do
    user
    |> cast(attrs, [:email, :name, :role, :is_admin])
    |> validate_inclusion(:role, ["user", "moderator", "admin"])
  end
end

# Controller: use the right changeset
def update(conn, %{"user" => user_params}) do
  user = conn.assigns.current_user
  # Use public changeset, not admin_changeset
  changeset = User.changeset(user, user_params)
  # ...
end
```

---

## 6. Timing Attack Prevention

```elixir
defmodule MyApp.Security do
  # Use constant-time comparison for secrets
  def secure_compare(a, b) when is_binary(a) and is_binary(b) do
    # Plug.Crypto.secure_compare/2 is constant-time
    Plug.Crypto.secure_compare(a, b)
  end

  # Hash comparison - always compare hashes, never raw values
  def verify_api_key(provided_key, stored_hash) do
    Argon2.verify_pass(provided_key, stored_hash)
    # Argon2 is constant-time
  end

  # Webhook signature verification
  def verify_stripe_signature(payload, sig_header, secret) do
    [_, timestamp] = String.split(sig_header, "t=")
    [ts, _] = String.split(timestamp, ",")

    computed = :crypto.mac(:hmac, :sha256, secret, "#{ts}.#{payload}")
                |> Base.encode16(case: :lower)

    # Extract signature from header
    [_, received] = String.split(sig_header, "v1=")
    [sig | _] = String.split(received, ",")

    Plug.Crypto.secure_compare(computed, sig)
  end
end
```

---

## สรุป

```
OWASP Top 10 in Phoenix:
├── Injection: Ecto params, never string interp
├── XSS: HEEx auto-escape, CSP headers
├── CSRF: plug :protect_from_forgery
├── Broken Auth: phx.gen.auth, Argon2
├── Sensitive Data: runtime.exs, encryption
├── Security Misconfig: put_secure_browser_headers
├── Mass Assignment: explicit cast field lists
└── Timing Attacks: secure_compare/2

Security Headers (Phoenix default):
├── x-frame-options: SAMEORIGIN
├── x-xss-protection: 1; mode=block
├── x-content-type-options: nosniff
└── x-download-options: noopen

Never:
├── Commit .env files
├── Log passwords or tokens
├── Use raw() with user input
└── Trust user-provided IDs without auth check
```

---

*ก่อนหน้า: [Part 94](part_94.md) | ต่อไป: [Part 96 - CI/CD Pipeline](part_96.md)*
