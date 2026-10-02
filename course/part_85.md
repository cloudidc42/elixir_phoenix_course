# Part 85: Authentication Advanced (การยืนยันตัวตนขั้นสูง)

## เป้าหมายการเรียนรู้

หลังจากจบ Part นี้ คุณจะสามารถ:
- Two-factor authentication (TOTP)
- OAuth 2.0 / OpenID Connect
- Passkeys / WebAuthn
- Session management ขั้นสูง

---

## 1. Two-Factor Authentication (TOTP)

```elixir
# mix.exs: {:ex_totp, "~> 0.2"}

defmodule MyApp.TwoFactor do
  alias MyApp.Repo

  def setup_totp(user) do
    secret = NimbleTOTP.secret()
    uri = NimbleTOTP.otpauth_uri("MyApp:#{user.email}", secret, issuer: "MyApp")

    # Generate QR code URL
    qr_url = "https://api.qrserver.com/v1/create-qr-code/?size=200x200&data=#{URI.encode(uri)}"

    {:ok, %{secret: Base.encode32(secret), qr_url: qr_url, uri: uri}}
  end

  def enable_totp(user, secret_base32, code) do
    secret = Base.decode32!(secret_base32)

    if NimbleTOTP.valid?(secret, code) do
      # Generate backup codes
      backup_codes = generate_backup_codes()

      user
      |> Ecto.Changeset.change(%{
        totp_secret: secret_base32,
        totp_enabled: true,
        totp_backup_codes: hash_backup_codes(backup_codes)
      })
      |> Repo.update()
      |> case do
        {:ok, user} -> {:ok, user, backup_codes}
        error -> error
      end
    else
      {:error, :invalid_code}
    end
  end

  def verify_totp(user, code) do
    if user.totp_enabled do
      secret = Base.decode32!(user.totp_secret)

      cond do
        NimbleTOTP.valid?(secret, code) ->
          :ok

        verify_backup_code(user, code) ->
          # Burn the backup code
          remove_backup_code(user, code)
          :ok

        true ->
          {:error, :invalid_code}
      end
    else
      :ok  # TOTP not required
    end
  end

  defp generate_backup_codes do
    Enum.map(1..8, fn _ ->
      :crypto.strong_rand_bytes(5)
      |> Base.encode16()
      |> String.downcase()
      |> then(&"#{String.slice(&1, 0, 4)}-#{String.slice(&1, 4, 4)}")
    end)
  end

  defp hash_backup_codes(codes) do
    Enum.map(codes, &Bcrypt.hash_pwd_salt/1)
  end

  defp verify_backup_code(user, code) do
    Enum.any?(user.totp_backup_codes || [], &Bcrypt.verify_pass(code, &1))
  end

  defp remove_backup_code(user, code) do
    remaining = Enum.reject(user.totp_backup_codes, &Bcrypt.verify_pass(code, &1))
    user |> Ecto.Changeset.change(%{totp_backup_codes: remaining}) |> Repo.update()
  end
end
```

---

## 2. OAuth 2.0 with Ueberauth

```elixir
# mix.exs
{:ueberauth, "~> 0.10"},
{:ueberauth_google, "~> 0.10"},
{:ueberauth_github, "~> 0.8"}

# config/config.exs
config :ueberauth, Ueberauth,
  providers: [
    google: {Ueberauth.Strategy.Google, [
      default_scope: "email profile"
    ]},
    github: {Ueberauth.Strategy.Github, [
      default_scope: "user:email"
    ]}
  ]

config :ueberauth, Ueberauth.Strategy.Google.OAuth,
  client_id: System.get_env("GOOGLE_CLIENT_ID"),
  client_secret: System.get_env("GOOGLE_CLIENT_SECRET")

# router.ex
scope "/auth", MyAppWeb do
  pipe_through :browser

  get "/:provider", AuthController, :request
  get "/:provider/callback", AuthController, :callback
end

# controllers/auth_controller.ex
defmodule MyAppWeb.AuthController do
  use MyAppWeb, :controller
  plug Ueberauth

  def callback(%{assigns: %{ueberauth_auth: auth}} = conn, _params) do
    case MyApp.Accounts.find_or_create_oauth_user(auth) do
      {:ok, user} ->
        conn
        |> MyAppWeb.UserAuth.log_in_user(user)
        |> redirect(to: ~p"/dashboard")

      {:error, reason} ->
        conn
        |> put_flash(:error, "Authentication failed: #{reason}")
        |> redirect(to: ~p"/login")
    end
  end

  def callback(%{assigns: %{ueberauth_failure: failure}} = conn, _params) do
    conn
    |> put_flash(:error, "Failed: #{inspect(failure.errors)}")
    |> redirect(to: ~p"/login")
  end
end

defmodule MyApp.Accounts do
  def find_or_create_oauth_user(%Ueberauth.Auth{} = auth) do
    provider = to_string(auth.provider)
    uid = auth.uid

    case Repo.get_by(OAuthIdentity, provider: provider, uid: uid) do
      %OAuthIdentity{} = identity ->
        {:ok, Repo.preload(identity, :user).user}

      nil ->
        create_oauth_user(auth)
    end
  end

  defp create_oauth_user(auth) do
    email = auth.info.email
    name = auth.info.name

    Ecto.Multi.new()
    |> Ecto.Multi.run(:user, fn repo, _ ->
      case repo.get_by(User, email: email) do
        nil ->
          repo.insert(%User{email: email, name: name, confirmed_at: DateTime.utc_now()})
        user ->
          {:ok, user}
      end
    end)
    |> Ecto.Multi.insert(:identity, fn %{user: user} ->
      %OAuthIdentity{
        user_id: user.id,
        provider: to_string(auth.provider),
        uid: to_string(auth.uid),
        token: auth.credentials.token
      }
    end)
    |> Repo.transaction()
    |> case do
      {:ok, %{user: user}} -> {:ok, user}
      {:error, _, changeset, _} -> {:error, changeset}
    end
  end
end
```

---

## 3. Session Management

```elixir
defmodule MyApp.Sessions do
  alias MyApp.Repo
  alias MyApp.Session
  import Ecto.Query

  def create_session(user, conn) do
    ip = conn.remote_ip |> :inet.ntoa() |> to_string()
    user_agent = get_req_header(conn, "user-agent") |> List.first()

    %Session{}
    |> Session.changeset(%{
      user_id: user.id,
      token: :crypto.strong_rand_bytes(32) |> Base.url_encode64(),
      ip_address: ip,
      user_agent: user_agent,
      expires_at: DateTime.add(DateTime.utc_now(), 30 * 24 * 60 * 60)  # 30 days
    })
    |> Repo.insert()
  end

  def list_active(user_id) do
    from(s in Session,
      where: s.user_id == ^user_id and s.expires_at > ^DateTime.utc_now(),
      order_by: [desc: s.last_used_at]
    )
    |> Repo.all()
  end

  def revoke_session(session_id, user_id) do
    from(s in Session,
      where: s.id == ^session_id and s.user_id == ^user_id
    )
    |> Repo.delete_all()
  end

  def revoke_all_sessions(user_id, except_token \\ nil) do
    query = from(s in Session, where: s.user_id == ^user_id)
    query = if except_token do
      where(query, [s], s.token != ^except_token)
    else
      query
    end
    Repo.delete_all(query)
  end
end
```

---

## สรุป

```
Auth Features:
├── TOTP: time-based one-time passwords
├── Backup codes: recovery when phone lost
├── OAuth: Google, GitHub, Apple
└── Session management: multi-device

Security:
├── Never store TOTP codes after use
├── Rate limit verification attempts
├── Burn backup codes after use
└── Alert user on new device login

OAuth Flow:
├── /auth/google → redirect to Google
├── Google → /auth/google/callback
├── Find or create user + identity
└── Log in and redirect
```

---

*ก่อนหน้า: [Part 84](part_84.md) | ต่อไป: [Part 86 - Payment Processing](part_86.md)*
