# Part 33: Authentication ด้วย Phoenix

## เป้าหมายการเรียนรู้

หลังจากจบ Part นี้ คุณจะสามารถ:
- สร้าง authentication system ด้วย mix phx.gen.auth
- เข้าใจ session-based authentication
- สร้าง JWT authentication สำหรับ API
- Implement OAuth

---

## 1. mix phx.gen.auth

Phoenix มี generator ที่สร้าง authentication ให้อัตโนมัติ:

```bash
mix phx.gen.auth Accounts User users
mix deps.get
mix ecto.migrate
```

สิ่งที่ถูกสร้าง:
- `Accounts` context
- `User` schema
- Migration
- Controllers, Views, Templates
- Session handling
- Email confirmation

---

## 2. โครงสร้าง Authentication

### User Schema

```elixir
# lib/my_app/accounts/user.ex
defmodule MyApp.Accounts.User do
  use Ecto.Schema
  import Ecto.Changeset

  schema "users" do
    field :email, :string
    field :password, :string, virtual: true, redact: true
    field :hashed_password, :string, redact: true
    field :confirmed_at, :naive_datetime
    field :role, :string, default: "user"
    field :name, :string

    timestamps()
  end

  def registration_changeset(user, attrs, opts \\ []) do
    user
    |> cast(attrs, [:email, :password, :name])
    |> validate_required([:email, :password])
    |> validate_email(opts)
    |> validate_password(opts)
  end

  defp validate_email(changeset, opts) do
    changeset
    |> validate_required([:email])
    |> validate_format(:email, ~r/^[^\s]+@[^\s]+$/, message: "must have the @ sign and no spaces")
    |> validate_length(:email, max: 160)
    |> maybe_validate_unique_email(opts)
  end

  defp validate_password(changeset, opts) do
    changeset
    |> validate_required([:password])
    |> validate_length(:password, min: 12, max: 72)
    |> maybe_hash_password(opts)
  end

  defp maybe_hash_password(changeset, opts) do
    hash_password? = Keyword.get(opts, :hash_password, true)
    password = get_change(changeset, :password)

    if hash_password? && password && changeset.valid? do
      changeset
      |> validate_length(:password, max: 72, count: :bytes)
      |> put_change(:hashed_password, Bcrypt.hash_pwd_salt(password))
      |> delete_change(:password)
    else
      changeset
    end
  end

  def valid_password?(%MyApp.Accounts.User{hashed_password: hashed_password}, password)
      when is_binary(hashed_password) and byte_size(password) > 0 do
    Bcrypt.verify_pass(password, hashed_password)
  end
end
```

### Accounts Context

```elixir
# lib/my_app/accounts.ex
defmodule MyApp.Accounts do
  alias MyApp.Accounts.{User, UserToken}
  alias MyApp.Repo

  def get_user_by_email_and_password(email, password) when is_binary(email) and is_binary(password) do
    user = Repo.get_by(User, email: email)
    if User.valid_password?(user, password), do: user
  end

  def register_user(attrs) do
    %User{}
    |> User.registration_changeset(attrs)
    |> Repo.insert()
  end

  def generate_user_session_token(user) do
    {token, user_token} = UserToken.build_session_token(user)
    Repo.insert!(user_token)
    token
  end

  def get_user_by_session_token(token) do
    {:ok, query} = UserToken.verify_session_token_query(token)
    Repo.one(query)
  end

  def delete_user_session_token(token) do
    Repo.delete_all(UserToken.by_token_and_context_query(token, "session"))
    :ok
  end
end
```

### Session Controller

```elixir
# lib/my_app_web/controllers/user_session_controller.ex
defmodule MyAppWeb.UserSessionController do
  use MyAppWeb, :controller

  alias MyApp.Accounts
  alias MyAppWeb.UserAuth

  def new(conn, _params) do
    render(conn, :new, error_message: nil)
  end

  def create(conn, %{"user" => user_params}) do
    %{"email" => email, "password" => password} = user_params

    if user = Accounts.get_user_by_email_and_password(email, password) do
      conn
      |> put_flash(:info, "Welcome back!")
      |> UserAuth.log_in_user(user)
    else
      conn
      |> put_flash(:error, "Invalid email or password")
      |> put_status(401)
      |> render(:new, error_message: "Invalid email or password")
    end
  end

  def delete(conn, _params) do
    conn
    |> put_flash(:info, "Logged out successfully.")
    |> UserAuth.log_out_user()
  end
end
```

### UserAuth Plug

```elixir
# lib/my_app_web/user_auth.ex
defmodule MyAppWeb.UserAuth do
  use MyAppWeb, :verified_routes

  import Plug.Conn
  import Phoenix.Controller

  alias MyApp.Accounts

  @max_age 60 * 60 * 24 * 60  # 60 days
  @remember_me_cookie "_my_app_web_user_remember_me"
  @remember_me_options [sign: true, max_age: @max_age, same_site: "Lax"]

  def log_in_user(conn, user, params \\ %{}) do
    token = Accounts.generate_user_session_token(user)
    user_return_to = get_session(conn, :user_return_to)

    conn
    |> renew_session()
    |> put_token_in_session(token)
    |> maybe_write_remember_me_cookie(token, params)
    |> redirect(to: user_return_to || signed_in_path(conn))
  end

  def log_out_user(conn) do
    user_token = get_session(conn, :user_token)
    user_token && Accounts.delete_user_session_token(user_token)

    if live_socket_id = get_session(conn, :live_socket_id) do
      MyAppWeb.Endpoint.broadcast(live_socket_id, "disconnect", %{})
    end

    conn
    |> renew_session()
    |> delete_resp_cookie(@remember_me_cookie)
    |> redirect(to: ~p"/")
  end

  def fetch_current_user(conn, _opts) do
    {user_token, conn} = ensure_user_token(conn)
    user = user_token && Accounts.get_user_by_session_token(user_token)
    assign(conn, :current_user, user)
  end

  def require_authenticated_user(conn, _opts) do
    if conn.assigns[:current_user] do
      conn
    else
      conn
      |> put_flash(:error, "You must log in to access this page.")
      |> maybe_store_return_to()
      |> redirect(to: ~p"/users/log_in")
      |> halt()
    end
  end

  def redirect_if_user_is_authenticated(conn, _opts) do
    if conn.assigns[:current_user] do
      conn
      |> redirect(to: signed_in_path(conn))
      |> halt()
    else
      conn
    end
  end

  defp ensure_user_token(conn) do
    if token = get_session(conn, :user_token) do
      {token, conn}
    else
      conn = fetch_cookies(conn, signed: [@remember_me_cookie])
      if token = conn.cookies[@remember_me_cookie] do
        {token, put_token_in_session(conn, token)}
      else
        {nil, conn}
      end
    end
  end

  defp maybe_write_remember_me_cookie(conn, token, %{"remember_me" => "true"}) do
    put_resp_cookie(conn, @remember_me_cookie, token, @remember_me_options)
  end
  defp maybe_write_remember_me_cookie(conn, _token, _params), do: conn

  defp renew_session(conn) do
    delete_csrf_token()
    conn
    |> configure_session(renew: true)
    |> clear_session()
  end

  defp put_token_in_session(conn, token) do
    conn
    |> put_session(:user_token, token)
    |> put_session(:live_socket_id, "users_sessions:#{Base.url_encode64(token)}")
  end

  defp maybe_store_return_to(%{method: "GET"} = conn) do
    put_session(conn, :user_return_to, current_path(conn))
  end
  defp maybe_store_return_to(conn), do: conn

  defp signed_in_path(_conn), do: ~p"/"
end
```

---

## 3. JWT Authentication สำหรับ API

```elixir
# mix.exs
{:joken, "~> 2.6"}

# lib/my_app/auth/token.ex
defmodule MyApp.Auth.Token do
  use Joken.Config

  @impl Joken.Config
  def token_config do
    [exp: 86400]  # 24 hours
    |> add_claim("iss", fn -> "my_app" end, &(&1 == "my_app"))
    |> add_claim("iat", fn -> DateTime.utc_now() |> DateTime.to_unix() end)
  end

  def generate(user) do
    claims = %{
      "sub" => user.id,
      "email" => user.email,
      "role" => user.role
    }

    with {:ok, token, _claims} <- generate_and_sign(claims) do
      {:ok, token}
    end
  end

  def verify(token) do
    with {:ok, claims} <- verify_and_validate(token) do
      {:ok, claims}
    end
  end
end
```

### API Authentication Plug

```elixir
defmodule MyAppWeb.Plugs.ApiAuth do
  import Plug.Conn
  import Phoenix.Controller

  alias MyApp.Auth.Token
  alias MyApp.Accounts

  def init(opts), do: opts

  def call(conn, _opts) do
    with {:ok, token} <- extract_token(conn),
         {:ok, claims} <- Token.verify(token),
         {:ok, user} <- get_user(claims["sub"]) do
      assign(conn, :current_user, user)
    else
      {:error, _reason} ->
        conn
        |> put_status(401)
        |> json(%{error: "Unauthorized"})
        |> halt()
    end
  end

  defp extract_token(conn) do
    case get_req_header(conn, "authorization") do
      ["Bearer " <> token] -> {:ok, token}
      _ -> {:error, :no_token}
    end
  end

  defp get_user(user_id) do
    case Accounts.get_user(user_id) do
      nil -> {:error, :not_found}
      user -> {:ok, user}
    end
  end
end
```

### API Auth Controller

```elixir
defmodule MyAppWeb.Api.AuthController do
  use MyAppWeb, :controller

  alias MyApp.Accounts
  alias MyApp.Auth.Token

  def login(conn, %{"email" => email, "password" => password}) do
    case Accounts.get_user_by_email_and_password(email, password) do
      nil ->
        conn
        |> put_status(401)
        |> json(%{error: "Invalid credentials"})

      user ->
        {:ok, token} = Token.generate(user)
        refresh_token = generate_refresh_token(user)

        conn
        |> json(%{
          access_token: token,
          refresh_token: refresh_token,
          user: %{id: user.id, email: user.email, name: user.name}
        })
    end
  end

  def refresh(conn, %{"refresh_token" => refresh_token}) do
    case verify_refresh_token(refresh_token) do
      {:ok, user} ->
        {:ok, token} = Token.generate(user)
        conn |> json(%{access_token: token})

      {:error, _} ->
        conn
        |> put_status(401)
        |> json(%{error: "Invalid refresh token"})
    end
  end

  defp generate_refresh_token(user) do
    Phoenix.Token.sign(MyAppWeb.Endpoint, "refresh", user.id,
      max_age: 60 * 60 * 24 * 30)  # 30 days
  end

  defp verify_refresh_token(token) do
    case Phoenix.Token.verify(MyAppWeb.Endpoint, "refresh", token, max_age: 60 * 60 * 24 * 30) do
      {:ok, user_id} -> {:ok, Accounts.get_user!(user_id)}
      {:error, reason} -> {:error, reason}
    end
  end
end
```

---

## 4. Role-Based Access Control (RBAC)

```elixir
defmodule MyApp.Auth.Policy do
  @roles_hierarchy %{
    super_admin: [:admin, :moderator, :user],
    admin: [:moderator, :user],
    moderator: [:user],
    user: []
  }

  def can?(%{role: role}, action, resource) do
    check_permission(String.to_atom(role), action, resource)
  end

  def can?(nil, _action, _resource), do: false

  defp check_permission(:super_admin, _action, _resource), do: true

  defp check_permission(:admin, action, :user) do
    action in [:read, :update, :delete, :create]
  end

  defp check_permission(:admin, action, :post) do
    action in [:read, :update, :delete, :create, :publish]
  end

  defp check_permission(:moderator, action, :post) do
    action in [:read, :update, :delete]
  end

  defp check_permission(:user, action, :post) when action in [:read, :create], do: true

  defp check_permission(:user, action, {:own_post, owner_id}, user_id) do
    action in [:update, :delete] and owner_id == user_id
  end

  defp check_permission(_role, _action, _resource), do: false
end

# Plug สำหรับตรวจสอบ permission
defmodule MyAppWeb.Plugs.Authorize do
  import Plug.Conn
  import Phoenix.Controller

  alias MyApp.Auth.Policy

  def init(opts), do: opts

  def call(conn, [action: action, resource: resource]) do
    user = conn.assigns[:current_user]

    if Policy.can?(user, action, resource) do
      conn
    else
      conn
      |> put_status(403)
      |> json(%{error: "Forbidden"})
      |> halt()
    end
  end
end
```

---

## 5. OAuth Integration (ด้วย Ueberauth)

```elixir
# mix.exs
{:ueberauth, "~> 0.10"},
{:ueberauth_google, "~> 0.10"},
{:ueberauth_github, "~> 0.8"},

# config/config.exs
config :ueberauth, Ueberauth,
  providers: [
    google: {Ueberauth.Strategy.Google, []},
    github: {Ueberauth.Strategy.Github, []}
  ]

config :ueberauth, Ueberauth.Strategy.Google.OAuth,
  client_id: System.get_env("GOOGLE_CLIENT_ID"),
  client_secret: System.get_env("GOOGLE_CLIENT_SECRET")
```

```elixir
defmodule MyAppWeb.AuthController do
  use MyAppWeb, :controller
  plug Ueberauth

  alias MyApp.Accounts

  def request(conn, _params), do: conn

  def callback(%{assigns: %{ueberauth_auth: auth}} = conn, _params) do
    user_params = %{
      email: auth.info.email,
      name: auth.info.name,
      provider: to_string(auth.provider),
      provider_id: to_string(auth.uid)
    }

    case Accounts.find_or_create_oauth_user(user_params) do
      {:ok, user} ->
        conn
        |> put_flash(:info, "Welcome, #{user.name}!")
        |> MyAppWeb.UserAuth.log_in_user(user)

      {:error, _reason} ->
        conn
        |> put_flash(:error, "Authentication failed")
        |> redirect(to: "/login")
    end
  end

  def callback(%{assigns: %{ueberauth_failure: _failure}} = conn, _params) do
    conn
    |> put_flash(:error, "Authentication failed")
    |> redirect(to: "/login")
  end
end
```

---

## 6. Two-Factor Authentication

```elixir
defmodule MyApp.Auth.TwoFactor do
  alias MyApp.Accounts.User
  alias MyApp.Repo

  def generate_secret do
    :crypto.strong_rand_bytes(20) |> Base.encode32(padding: false)
  end

  def generate_totp(secret) do
    :pot.totp(secret)
  end

  def verify_totp(secret, code) do
    :pot.valid_totp(code, secret)
  end

  def enable_2fa(user, secret) do
    user
    |> Ecto.Changeset.change(%{totp_secret: secret, totp_enabled: true})
    |> Repo.update()
  end

  def generate_qr_uri(user, secret) do
    issuer = "MyApp"
    label = URI.encode("#{issuer}:#{user.email}")
    "otpauth://totp/#{label}?secret=#{secret}&issuer=#{URI.encode(issuer)}"
  end
end
```

---

## แบบฝึกหัด

### Exercise 1: Password Reset
Implement password reset flow:
1. User request password reset (กรอก email)
2. ส่ง reset email ที่มี token
3. User คลิก link ใน email
4. แสดง form รีเซ็ต password
5. อัปเดต password

### Exercise 2: API Rate Limiting
เพิ่ม rate limiting สำหรับ API:
- Max 100 requests per minute per user
- ใช้ ETS หรือ Redis เก็บ state

---

## สรุป

```
Authentication ใน Phoenix:
├── Session-based: phx.gen.auth
├── JWT: Joken library
├── OAuth: Ueberauth
└── 2FA: TOTP

Key Components:
├── User schema + password hashing
├── Token generation/verification
├── Plug pipeline integration
└── Role-based access control

Security Best Practices:
├── bcrypt/argon2 สำหรับ passwords
├── Short-lived JWT tokens
├── Refresh token rotation
└── Rate limiting
```

---

*ก่อนหน้า: [Part 32](part_32.md) | ต่อไป: [Part 34 - API Development](part_34.md)*
