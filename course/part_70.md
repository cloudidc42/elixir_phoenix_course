# Part 70: Advanced Authentication Patterns

## เป้าหมายการเรียนรู้

- สร้าง passwordless auth ด้วย magic links และ passkeys
- รวม social login (Google, GitHub, Facebook) ด้วย Ueberauth
- ตั้งค่า Enterprise SSO ด้วย SAML และ LDAP
- หมุน API keys อย่างปลอดภัย
- จัดการ sessions และยกเลิกได้จากทุก device
- ติดตาม sessions หลาย devices
- เก็บ audit log สำหรับ security events

---

## 1. Passwordless Authentication

### 1.1 Magic Links

Magic link คือ one-time token ที่ส่งผ่าน email ให้ผู้ใช้คลิกเพื่อ login โดยไม่ต้องใช้รหัสผ่าน

```elixir
# migration
defmodule MyApp.Repo.Migrations.CreateMagicLinks do
  use Ecto.Migration

  def change do
    create table(:magic_links) do
      add :user_id, references(:users, on_delete: :delete_all), null: false
      add :token_hash, :string, null: false
      add :used_at, :utc_datetime
      add :expires_at, :utc_datetime, null: false
      add :ip_address, :string
      add :user_agent, :string
      timestamps()
    end

    create unique_index(:magic_links, [:token_hash])
    create index(:magic_links, [:user_id])
    create index(:magic_links, [:expires_at])
  end
end
```

```elixir
defmodule MyApp.Auth.MagicLink do
  alias MyApp.{Repo, Accounts}
  alias MyApp.Auth.MagicLink, as: ML
  import Ecto.Query

  @token_length 32
  @expiry_minutes 15

  schema "magic_links" do
    belongs_to :user, MyApp.Accounts.User
    field :token_hash, :string
    field :used_at, :utc_datetime
    field :expires_at, :utc_datetime
    field :ip_address, :string
    field :user_agent, :string
    timestamps()
  end

  # สร้าง magic link สำหรับ email
  def create_for_email(email, opts \\ []) do
    case Accounts.get_user_by_email(email) do
      nil ->
        # ไม่บอกว่า email ไม่มีอยู่ (security)
        {:ok, :email_sent}
      user ->
        {:ok, link} = create_link(user, opts)
        send_magic_link_email(user, link)
        {:ok, :email_sent}
    end
  end

  defp create_link(user, opts) do
    # ลบ links เก่าที่ยังไม่ได้ใช้
    from(ml in ML, where: ml.user_id == ^user.id and is_nil(ml.used_at))
    |> Repo.delete_all()

    raw_token = :crypto.strong_rand_bytes(@token_length) |> Base.url_encode64(padding: false)
    token_hash = hash_token(raw_token)
    expires_at = DateTime.add(DateTime.utc_now(), @expiry_minutes * 60, :second)

    attrs = %{
      user_id: user.id,
      token_hash: token_hash,
      expires_at: expires_at |> DateTime.truncate(:second),
      ip_address: Keyword.get(opts, :ip_address),
      user_agent: Keyword.get(opts, :user_agent)
    }

    case Repo.insert(struct(ML, attrs)) do
      {:ok, _link} -> {:ok, raw_token}
      error -> error
    end
  end

  # ตรวจสอบ magic link token
  def verify_token(raw_token) do
    token_hash = hash_token(raw_token)
    now = DateTime.utc_now()

    case from(ml in ML,
           where: ml.token_hash == ^token_hash
             and is_nil(ml.used_at)
             and ml.expires_at > ^now,
           preload: [:user]
         )
         |> Repo.one() do
      nil ->
        {:error, :invalid_or_expired}

      link ->
        # Mark as used
        Repo.update_all(
          from(ml in ML, where: ml.id == ^link.id),
          set: [used_at: DateTime.utc_now() |> DateTime.truncate(:second)]
        )
        {:ok, link.user}
    end
  end

  defp hash_token(token) do
    :crypto.hash(:sha256, token) |> Base.encode16(case: :lower)
  end

  defp send_magic_link_email(user, token) do
    url = "#{MyAppWeb.Endpoint.url()}/auth/magic?token=#{token}"

    %{}
    |> MyApp.Workers.SendMagicLinkEmail.new(%{
      email: user.email,
      name: user.name,
      url: url
    })
    |> Oban.insert()
  end
end
```

### 1.2 Passkeys (WebAuthn)

```elixir
# mix.exs
defp deps do
  [
    {:wax_, "~> 0.6"},
    # ...
  ]
end
```

```elixir
defmodule MyApp.Auth.Passkey do
  alias MyApp.Repo
  import Ecto.Query

  # Schema
  schema "passkeys" do
    belongs_to :user, MyApp.Accounts.User
    field :credential_id, :binary
    field :public_key_cose, :binary
    field :sign_count, :integer, default: 0
    field :device_name, :string
    field :last_used_at, :utc_datetime
    timestamps()
  end

  # Registration: step 1 - สร้าง challenge
  def registration_challenge(user) do
    challenge = :crypto.strong_rand_bytes(32)

    options = %{
      challenge: challenge,
      rp: %{name: "MyApp", id: "myapp.com"},
      user: %{
        id: :crypto.hash(:sha256, "user:#{user.id}"),
        name: user.email,
        display_name: user.name
      },
      pub_key_cred_params: [
        %{type: "public-key", alg: -7},   # ES256
        %{type: "public-key", alg: -257}  # RS256
      ],
      authenticator_selection: %{
        resident_key: "preferred",
        user_verification: "preferred"
      },
      timeout: 60_000,
      attestation: "none"
    }

    # เก็บ challenge ใน session
    Phoenix.Token.sign(MyAppWeb.Endpoint, "passkey_registration", %{
      user_id: user.id,
      challenge: Base.encode64(challenge)
    })
    |> then(fn token -> {token, options} end)
  end

  # Registration: step 2 - verify และบันทึก
  def complete_registration(user, credential, session_token) do
    with {:ok, %{user_id: user_id, challenge: challenge_b64}} <-
           Phoenix.Token.verify(MyAppWeb.Endpoint, "passkey_registration", session_token,
             max_age: 300),
         true <- user_id == user.id,
         challenge <- Base.decode64!(challenge_b64),
         {:ok, result} <- Wax.register(credential, challenge, origin: "https://myapp.com") do

      attrs = %{
        user_id: user.id,
        credential_id: result.credential_id,
        public_key_cose: result.public_key_cose,
        sign_count: result.sign_count || 0,
        device_name: credential["deviceName"] || "Device"
      }

      Repo.insert(struct(__MODULE__, attrs))
    else
      _ -> {:error, :registration_failed}
    end
  end

  # Authentication challenge
  def authentication_challenge(email) do
    user = MyApp.Accounts.get_user_by_email!(email)
    passkeys = list_user_passkeys(user.id)
    challenge = :crypto.strong_rand_bytes(32)

    token = Phoenix.Token.sign(MyAppWeb.Endpoint, "passkey_auth", %{
      user_id: user.id,
      challenge: Base.encode64(challenge)
    })

    options = %{
      challenge: challenge,
      timeout: 60_000,
      rp_id: "myapp.com",
      allow_credentials: Enum.map(passkeys, fn pk ->
        %{type: "public-key", id: pk.credential_id}
      end),
      user_verification: "preferred"
    }

    {token, options}
  end

  def complete_authentication(credential, session_token) do
    with {:ok, %{user_id: user_id, challenge: challenge_b64}} <-
           Phoenix.Token.verify(MyAppWeb.Endpoint, "passkey_auth", session_token,
             max_age: 300),
         challenge <- Base.decode64!(challenge_b64),
         passkey <- find_passkey(credential["id"]),
         {:ok, result} <- Wax.authenticate(
           credential,
           challenge,
           passkey.public_key_cose,
           passkey.sign_count,
           origin: "https://myapp.com"
         ) do

      update_sign_count(passkey.id, result.sign_count)
      user = MyApp.Accounts.get_user!(user_id)
      {:ok, user}
    else
      _ -> {:error, :authentication_failed}
    end
  end

  defp list_user_passkeys(user_id) do
    from(pk in __MODULE__, where: pk.user_id == ^user_id)
    |> Repo.all()
  end

  defp find_passkey(credential_id) do
    from(pk in __MODULE__, where: pk.credential_id == ^credential_id)
    |> Repo.one()
  end

  defp update_sign_count(id, count) do
    from(pk in __MODULE__, where: pk.id == ^id)
    |> Repo.update_all(set: [sign_count: count, last_used_at: DateTime.utc_now() |> DateTime.truncate(:second)])
  end
end
```

---

## 2. Social Login ด้วย Ueberauth

### 2.1 ติดตั้ง

```elixir
# mix.exs
defp deps do
  [
    {:ueberauth, "~> 0.10"},
    {:ueberauth_google, "~> 0.10"},
    {:ueberauth_github, "~> 0.8"},
    {:ueberauth_facebook, "~> 0.9"},
    # ...
  ]
end
```

```elixir
# config/runtime.exs
config :ueberauth, Ueberauth,
  providers: [
    google: {Ueberauth.Strategy.Google, [
      default_scope: "email profile"
    ]},
    github: {Ueberauth.Strategy.Github, [
      default_scope: "user:email"
    ]},
    facebook: {Ueberauth.Strategy.Facebook, [
      default_scope: "email,public_profile"
    ]}
  ]

config :ueberauth, Ueberauth.Strategy.Google.OAuth,
  client_id: System.get_env("GOOGLE_CLIENT_ID"),
  client_secret: System.get_env("GOOGLE_CLIENT_SECRET"),
  redirect_uri: System.get_env("GOOGLE_REDIRECT_URI")

config :ueberauth, Ueberauth.Strategy.Github.OAuth,
  client_id: System.get_env("GITHUB_CLIENT_ID"),
  client_secret: System.get_env("GITHUB_CLIENT_SECRET")
```

### 2.2 Auth Controller

```elixir
defmodule MyAppWeb.AuthController do
  use MyAppWeb, :controller
  plug Ueberauth

  alias MyApp.Auth.SocialAuth
  alias MyApp.{Accounts, AuditLog}

  # Redirect ไปยัง provider
  def request(conn, _params), do: conn

  # Callback จาก provider
  def callback(%{assigns: %{ueberauth_failure: failure}} = conn, _params) do
    conn
    |> put_flash(:error, "เข้าสู่ระบบไม่สำเร็จ: #{format_failure(failure)}")
    |> redirect(to: ~p"/login")
  end

  def callback(%{assigns: %{ueberauth_auth: auth}} = conn, _params) do
    case SocialAuth.find_or_create_user(auth) do
      {:ok, user} ->
        AuditLog.log(user.id, "login", %{
          method: "oauth",
          provider: to_string(auth.provider),
          ip: get_client_ip(conn)
        })

        conn
        |> put_session(:user_id, user.id)
        |> put_flash(:info, "เข้าสู่ระบบสำเร็จ")
        |> redirect(to: ~p"/dashboard")

      {:error, :email_taken} ->
        conn
        |> put_flash(:error, "Email นี้ถูกใช้ด้วยวิธีอื่นแล้ว กรุณาเข้าสู่ระบบด้วยวิธีเดิม")
        |> redirect(to: ~p"/login")

      {:error, reason} ->
        conn
        |> put_flash(:error, "เกิดข้อผิดพลาด: #{inspect(reason)}")
        |> redirect(to: ~p"/login")
    end
  end

  defp format_failure(%{errors: errors}) do
    errors |> Enum.map(& &1.message) |> Enum.join(", ")
  end
end

defmodule MyApp.Auth.SocialAuth do
  alias MyApp.{Repo, Accounts}
  alias MyApp.Accounts.{User, UserIdentity}
  import Ecto.Query

  def find_or_create_user(auth) do
    uid = to_string(auth.uid)
    provider = to_string(auth.provider)
    email = get_email(auth)

    # ค้นหา identity ที่มีอยู่
    case find_identity(provider, uid) do
      %UserIdentity{} = identity ->
        {:ok, Repo.preload(identity, :user).user}

      nil ->
        # ค้นหา user จาก email
        case email && Accounts.get_user_by_email(email) do
          %User{} = existing_user ->
            # Link identity กับ user ที่มีอยู่
            create_identity(existing_user, auth, provider, uid)
            {:ok, existing_user}

          nil ->
            # สร้าง user ใหม่
            create_user_with_identity(auth, provider, uid, email)
        end
    end
  end

  defp find_identity(provider, uid) do
    from(i in UserIdentity,
      where: i.provider == ^provider and i.uid == ^uid
    )
    |> Repo.one()
  end

  defp create_identity(user, auth, provider, uid) do
    Repo.insert!(%UserIdentity{
      user_id: user.id,
      provider: provider,
      uid: uid,
      access_token: auth.credentials.token,
      refresh_token: auth.credentials.refresh_token,
      token_expires_at: token_expiry(auth)
    })
  end

  defp create_user_with_identity(auth, provider, uid, email) do
    Repo.transaction(fn ->
      {:ok, user} = Accounts.create_user(%{
        email: email,
        name: get_name(auth),
        avatar_url: get_avatar(auth),
        email_verified: true  # OAuth email ถือว่า verified
      })

      create_identity(user, auth, provider, uid)
      user
    end)
  end

  defp get_email(%{info: %{email: email}}) when not is_nil(email), do: email
  defp get_email(_), do: nil

  defp get_name(%{info: %{name: name}}) when not is_nil(name), do: name
  defp get_name(%{info: %{nickname: nickname}}), do: nickname
  defp get_name(_), do: "User"

  defp get_avatar(%{info: %{image: image}}), do: image
  defp get_avatar(_), do: nil

  defp token_expiry(%{credentials: %{expires_at: exp}}) when not is_nil(exp) do
    DateTime.from_unix!(exp)
  end
  defp token_expiry(_), do: nil
end
```

---

## 3. API Key Management

```elixir
defmodule MyApp.Auth.APIKeys do
  alias MyApp.Repo
  import Ecto.Query

  @key_prefix "myapp"
  @key_length 32

  schema "api_keys" do
    belongs_to :user, MyApp.Accounts.User
    field :name, :string
    field :key_prefix, :string    # แสดงให้ user เห็นได้
    field :key_hash, :string      # hash ของ key จริง
    field :scopes, {:array, :string}, default: []
    field :last_used_at, :utc_datetime
    field :expires_at, :utc_datetime
    field :revoked_at, :utc_datetime
    field :ip_whitelist, {:array, :string}, default: []
    timestamps()
  end

  # สร้าง API key ใหม่
  def create(user, name, opts \\ []) do
    scopes = Keyword.get(opts, :scopes, ["read"])
    expires_in_days = Keyword.get(opts, :expires_in_days, 365)
    ip_whitelist = Keyword.get(opts, :ip_whitelist, [])

    # สร้าง raw key: myapp_<prefix>_<random>
    random = :crypto.strong_rand_bytes(@key_length) |> Base.url_encode64(padding: false)
    prefix = :crypto.strong_rand_bytes(4) |> Base.encode16(case: :lower)
    raw_key = "#{@key_prefix}_#{prefix}_#{random}"

    # Hash key สำหรับเก็บ DB
    key_hash = hash_key(raw_key)
    expires_at = DateTime.add(DateTime.utc_now(), expires_in_days * 86400, :second)

    attrs = %{
      user_id: user.id,
      name: name,
      key_prefix: "#{@key_prefix}_#{prefix}",
      key_hash: key_hash,
      scopes: scopes,
      expires_at: expires_at |> DateTime.truncate(:second),
      ip_whitelist: ip_whitelist
    }

    case Repo.insert(struct(__MODULE__, attrs)) do
      {:ok, api_key} ->
        # คืน raw key ครั้งเดียวเท่านั้น
        {:ok, raw_key, api_key}
      error -> error
    end
  end

  # ตรวจสอบ API key
  def verify(raw_key, opts \\ []) do
    required_scope = Keyword.get(opts, :scope)
    client_ip = Keyword.get(opts, :ip)
    key_hash = hash_key(raw_key)
    now = DateTime.utc_now()

    case from(k in __MODULE__,
           where: k.key_hash == ^key_hash
             and is_nil(k.revoked_at)
             and (is_nil(k.expires_at) or k.expires_at > ^now),
           preload: [:user]
         )
         |> Repo.one() do
      nil -> {:error, :invalid_key}

      key ->
        with :ok <- check_ip(key, client_ip),
             :ok <- check_scope(key, required_scope) do
          # อัพเดท last_used_at
          update_last_used(key.id)
          {:ok, key.user}
        end
    end
  end

  # Rotate API key (สร้างใหม่, ยกเลิกเก่า)
  def rotate(api_key_id, user) do
    with key when not is_nil(key) <- Repo.get(__MODULE__, api_key_id),
         true <- key.user_id == user.id do
      Repo.transaction(fn ->
        # Revoke key เก่า
        revoke(api_key_id)

        # สร้าง key ใหม่ด้วย settings เดียวกัน
        {:ok, raw_key, new_key} = create(user, "#{key.name} (rotated)",
          scopes: key.scopes,
          ip_whitelist: key.ip_whitelist
        )
        {raw_key, new_key}
      end)
    else
      nil -> {:error, :not_found}
      false -> {:error, :unauthorized}
    end
  end

  def revoke(api_key_id) do
    from(k in __MODULE__, where: k.id == ^api_key_id)
    |> Repo.update_all(set: [revoked_at: DateTime.utc_now() |> DateTime.truncate(:second)])
  end

  defp check_ip(key, nil), do: :ok
  defp check_ip(%{ip_whitelist: []}, _), do: :ok
  defp check_ip(%{ip_whitelist: whitelist}, ip) do
    if ip in whitelist, do: :ok, else: {:error, :ip_not_allowed}
  end

  defp check_scope(_, nil), do: :ok
  defp check_scope(%{scopes: scopes}, required) do
    if required in scopes, do: :ok, else: {:error, :insufficient_scope}
  end

  defp hash_key(key) do
    :crypto.hash(:sha256, key) |> Base.encode16(case: :lower)
  end

  defp update_last_used(id) do
    from(k in __MODULE__, where: k.id == ^id)
    |> Repo.update_all(set: [last_used_at: DateTime.utc_now() |> DateTime.truncate(:second)])
  end
end
```

---

## 4. Session Management

### 4.1 Multi-Device Session Tracking

```elixir
defmodule MyApp.Auth.SessionManager do
  alias MyApp.Repo
  import Ecto.Query

  schema "user_sessions" do
    belongs_to :user, MyApp.Accounts.User
    field :token_hash, :string
    field :device_name, :string
    field :device_type, :string    # "desktop", "mobile", "tablet"
    field :browser, :string
    field :os, :string
    field :ip_address, :string
    field :location, :string       # country/city จาก IP geolocation
    field :last_active_at, :utc_datetime
    field :expires_at, :utc_datetime
    field :revoked_at, :utc_datetime
    timestamps()
  end

  @session_duration_days 30

  def create_session(user, conn_info) do
    raw_token = :crypto.strong_rand_bytes(48) |> Base.url_encode64(padding: false)
    token_hash = hash_token(raw_token)
    expires_at = DateTime.add(DateTime.utc_now(), @session_duration_days * 86400, :second)

    ua = UAParser.parse(conn_info[:user_agent] || "")

    attrs = %{
      user_id: user.id,
      token_hash: token_hash,
      device_type: classify_device(ua),
      browser: "#{ua.family} #{ua.version}",
      os: "#{ua.os.family}",
      ip_address: conn_info[:ip_address],
      last_active_at: DateTime.utc_now() |> DateTime.truncate(:second),
      expires_at: expires_at |> DateTime.truncate(:second)
    }

    case Repo.insert(struct(__MODULE__, attrs)) do
      {:ok, session} -> {:ok, raw_token, session}
      error -> error
    end
  end

  def verify_session(raw_token) do
    token_hash = hash_token(raw_token)
    now = DateTime.utc_now()

    case from(s in __MODULE__,
           where: s.token_hash == ^token_hash
             and is_nil(s.revoked_at)
             and s.expires_at > ^now,
           preload: [:user]
         )
         |> Repo.one() do
      nil -> {:error, :invalid_session}
      session ->
        update_last_active(session.id)
        {:ok, session.user, session}
    end
  end

  def list_user_sessions(user_id) do
    now = DateTime.utc_now()
    from(s in __MODULE__,
      where: s.user_id == ^user_id
        and is_nil(s.revoked_at)
        and s.expires_at > ^now,
      order_by: [desc: s.last_active_at]
    )
    |> Repo.all()
  end

  def revoke_session(session_id) do
    from(s in __MODULE__, where: s.id == ^session_id)
    |> Repo.update_all(set: [revoked_at: DateTime.utc_now() |> DateTime.truncate(:second)])
  end

  def revoke_all_sessions(user_id, except_session_id \\ nil) do
    query = from(s in __MODULE__,
      where: s.user_id == ^user_id and is_nil(s.revoked_at)
    )

    query =
      if except_session_id,
        do: from(s in query, where: s.id != ^except_session_id),
        else: query

    query
    |> Repo.update_all(set: [revoked_at: DateTime.utc_now() |> DateTime.truncate(:second)])
  end

  defp update_last_active(session_id) do
    from(s in __MODULE__, where: s.id == ^session_id)
    |> Repo.update_all(set: [last_active_at: DateTime.utc_now() |> DateTime.truncate(:second)])
  end

  defp hash_token(token) do
    :crypto.hash(:sha256, token) |> Base.encode16(case: :lower)
  end

  defp classify_device(ua) do
    cond do
      ua.device.family in ["iPhone", "Android"] -> "mobile"
      ua.device.family in ["iPad"] -> "tablet"
      true -> "desktop"
    end
  end
end
```

### 4.2 Session LiveView Component

```elixir
defmodule MyAppWeb.SessionsLive do
  use MyAppWeb, :live_view

  alias MyApp.Auth.SessionManager
  alias MyApp.AuditLog

  def mount(_params, session, socket) do
    user = socket.assigns.current_user
    sessions = SessionManager.list_user_sessions(user.id)
    current_session_id = session["session_id"]

    {:ok, assign(socket,
      sessions: sessions,
      current_session_id: current_session_id
    )}
  end

  def handle_event("revoke_session", %{"id" => id}, socket) do
    id = String.to_integer(id)
    user = socket.assigns.current_user

    if id != socket.assigns.current_session_id do
      SessionManager.revoke_session(id)
      AuditLog.log(user.id, "session_revoked", %{session_id: id})
      sessions = SessionManager.list_user_sessions(user.id)
      {:noreply, assign(socket, :sessions, sessions)}
    else
      {:noreply, put_flash(socket, :error, "ไม่สามารถยกเลิก session ปัจจุบันได้")}
    end
  end

  def handle_event("revoke_all", _params, socket) do
    user = socket.assigns.current_user
    current_id = socket.assigns.current_session_id

    SessionManager.revoke_all_sessions(user.id, current_id)
    AuditLog.log(user.id, "all_sessions_revoked", %{})

    sessions = SessionManager.list_user_sessions(user.id)
    {:noreply, assign(socket, sessions: sessions)}
  end

  def render(assigns) do
    ~H"""
    <div class="max-w-2xl mx-auto p-6">
      <div class="flex justify-between items-center mb-6">
        <h1 class="text-xl font-bold">อุปกรณ์ที่เข้าสู่ระบบ</h1>
        <button phx-click="revoke_all" class="btn btn-sm btn-error btn-outline"
          data-confirm="ออกจากระบบทุกอุปกรณ์ (ยกเว้นอุปกรณ์นี้)?">
          ออกจากระบบทุกอุปกรณ์
        </button>
      </div>

      <div class="space-y-3">
        <%= for session <- @sessions do %>
          <div class={"bg-white rounded-lg border p-4 #{if session.id == @current_session_id, do: "border-blue-500 bg-blue-50"}"}>
            <div class="flex justify-between items-start">
              <div>
                <div class="flex items-center gap-2">
                  <span class="text-xl"><%= device_icon(session.device_type) %></span>
                  <span class="font-medium"><%= session.browser %></span>
                  <%= if session.id == @current_session_id do %>
                    <span class="text-xs bg-blue-100 text-blue-700 px-2 py-0.5 rounded">
                      อุปกรณ์นี้
                    </span>
                  <% end %>
                </div>
                <div class="text-sm text-gray-500 mt-1">
                  <span><%= session.os %></span>
                  <span class="mx-1">·</span>
                  <span><%= session.ip_address %></span>
                  <span class="mx-1">·</span>
                  <span>ใช้งานล่าสุด: <%= format_time(session.last_active_at) %></span>
                </div>
              </div>
              <%= if session.id != @current_session_id do %>
                <button
                  phx-click="revoke_session"
                  phx-value-id={session.id}
                  class="btn btn-xs btn-error btn-outline"
                >
                  ออกจากระบบ
                </button>
              <% end %>
            </div>
          </div>
        <% end %>
      </div>
    </div>
    """
  end

  defp device_icon("mobile"), do: "📱"
  defp device_icon("tablet"), do: "📱"
  defp device_icon(_), do: "💻"

  defp format_time(nil), do: "ไม่ทราบ"
  defp format_time(dt) do
    diff = DateTime.diff(DateTime.utc_now(), dt, :second)
    cond do
      diff < 60 -> "เมื่อสักครู่"
      diff < 3600 -> "#{div(diff, 60)} นาทีที่แล้ว"
      diff < 86400 -> "#{div(diff, 3600)} ชั่วโมงที่แล้ว"
      true -> "#{div(diff, 86400)} วันที่แล้ว"
    end
  end
end
```

---

## 5. Audit Log สำหรับ Security Events

```elixir
defmodule MyApp.AuditLog do
  alias MyApp.Repo
  import Ecto.Query

  @security_events ~w(
    login login_failed logout
    password_changed email_changed
    two_factor_enabled two_factor_disabled
    api_key_created api_key_revoked
    session_revoked all_sessions_revoked
    account_locked account_unlocked
    permission_changed
    suspicious_login
  )

  schema "audit_logs" do
    belongs_to :user, MyApp.Accounts.User
    field :action, :string, null: false
    field :actor_id, :integer           # ใครทำ (อาจเป็น admin)
    field :resource_type, :string
    field :resource_id, :integer
    field :metadata, :map, default: %{}
    field :ip_address, :string
    field :user_agent, :string
    field :severity, :string, default: "info"  # info, warning, critical
    timestamps(updated_at: false)
  end

  def log(user_id, action, metadata \\ %{}, opts \\ []) do
    severity = if action in security_critical_events(), do: "critical", else: determine_severity(action)

    attrs = %{
      user_id: user_id,
      action: action,
      actor_id: Keyword.get(opts, :actor_id, user_id),
      resource_type: Keyword.get(opts, :resource_type),
      resource_id: Keyword.get(opts, :resource_id),
      metadata: metadata,
      ip_address: Keyword.get(opts, :ip),
      user_agent: Keyword.get(opts, :user_agent),
      severity: severity,
      inserted_at: DateTime.utc_now() |> DateTime.truncate(:second)
    }

    Repo.insert_all("audit_logs", [attrs])

    # Alert สำหรับ critical events
    if severity == "critical" do
      notify_security_team(user_id, action, metadata)
    end

    :ok
  end

  def list_user_events(user_id, opts \\ []) do
    limit = Keyword.get(opts, :limit, 50)
    action = Keyword.get(opts, :action)

    from(a in __MODULE__,
      where: a.user_id == ^user_id
    )
    |> maybe_filter_action(action)
    |> order_by([a], desc: a.inserted_at)
    |> limit(^limit)
    |> Repo.all()
  end

  # ตรวจจับ suspicious activity
  def check_suspicious_logins(user_id) do
    # หา failed logins มากกว่า 5 ครั้งใน 10 นาที
    ten_minutes_ago = DateTime.add(DateTime.utc_now(), -600, :second)

    failed_count =
      from(a in __MODULE__,
        where: a.user_id == ^user_id
          and a.action == "login_failed"
          and a.inserted_at >= ^ten_minutes_ago,
        select: count(a.id)
      )
      |> Repo.one()

    if failed_count >= 5 do
      log(user_id, "suspicious_login", %{
        failed_attempts: failed_count,
        reason: "Multiple failed login attempts"
      })
      {:warning, :multiple_failed_attempts}
    else
      :ok
    end
  end

  # Security report
  def security_summary(user_id) do
    last_30_days = DateTime.add(DateTime.utc_now(), -30 * 86400, :second)

    from(a in __MODULE__,
      where: a.user_id == ^user_id
        and a.inserted_at >= ^last_30_days,
      group_by: a.action,
      select: {a.action, count(a.id)}
    )
    |> Repo.all()
    |> Map.new()
  end

  defp security_critical_events do
    ~w(password_changed email_changed two_factor_disabled account_locked suspicious_login)
  end

  defp determine_severity("login_failed"), do: "warning"
  defp determine_severity("suspicious_login"), do: "critical"
  defp determine_severity("account_locked"), do: "critical"
  defp determine_severity(_), do: "info"

  defp maybe_filter_action(query, nil), do: query
  defp maybe_filter_action(query, action), do: from(a in query, where: a.action == ^action)

  defp notify_security_team(user_id, action, metadata) do
    # Async notification
    Task.start(fn ->
      %{user_id: user_id, action: action, metadata: metadata}
      |> MyApp.Workers.SecurityAlert.new()
      |> Oban.insert()
    end)
  end
end
```

---

## 6. Enterprise SSO (SAML Skeleton)

```elixir
# mix.exs - ใช้ samly library
defp deps do
  [
    {:samly, "~> 1.3"},
    # ...
  ]
end
```

```elixir
# config/runtime.exs
config :samly, Samly.Provider,
  identity_providers: [
    %{
      id: "enterprise_idp",
      sp_id: "myapp",
      base_url: System.get_env("APP_URL"),
      metadata_file: "priv/saml/idp_metadata.xml",
      sign_requests: true,
      sign_metadata: true,
      signed_assertion_in_resp: true,
      signed_envelopes_in_resp: true,
      allow_idp_initiated_flow: false
    }
  ]
```

```elixir
defmodule MyAppWeb.SSO.SAMLController do
  use MyAppWeb, :controller

  def consume(conn, _params) do
    case Samly.get_active_assertion(conn) do
      nil ->
        conn
        |> put_flash(:error, "SSO authentication failed")
        |> redirect(to: ~p"/login")

      assertion ->
        email = Samly.Assertion.attribute(assertion, "email")
        name = Samly.Assertion.attribute(assertion, "displayName")
        groups = Samly.Assertion.attribute(assertion, "groups") || []

        case MyApp.Auth.SSOAuth.find_or_provision_user(email, name, groups) do
          {:ok, user} ->
            MyApp.AuditLog.log(user.id, "login", %{method: "saml"})
            conn
            |> put_session(:user_id, user.id)
            |> redirect(to: ~p"/dashboard")

          {:error, _} ->
            conn
            |> put_flash(:error, "ไม่สามารถเข้าสู่ระบบได้")
            |> redirect(to: ~p"/login")
        end
    end
  end
end
```

---

## สรุป

```
Authentication Strategy Matrix
═════════════════════════════════════════════════════════
Method           | Use case              | Security
─────────────────────────────────────────────────────────
Password         | General users         | Medium
Magic Link       | Low-friction login    | High
Passkey          | Modern browsers       | Very High
Google/GitHub    | Developer tools       | High
Facebook         | Consumer apps         | Medium-High
SAML/SSO         | Enterprise            | Very High
LDAP             | Corporate networks    | High
API Key          | Machine-to-machine    | High (w/ rotation)

Session Security
  ├── Stored as hash (never plaintext)
  ├── Per-device tracking
  ├── Force logout from any device
  ├── Automatic expiry (30 days default)
  └── Suspicious activity detection

API Key Best Practices
  ├── Never store raw key (hash only)
  ├── Show raw key ONCE at creation
  ├── Support key rotation with overlap period
  ├── Scope-limited (read, write, admin)
  ├── IP whitelist support
  └── Expiry date

Audit Log Events
  INFO:     login, logout, api_key_used
  WARNING:  login_failed, unknown_device
  CRITICAL: password_changed, 2FA disabled
            suspicious_login, account_locked
═════════════════════════════════════════════════════════
```

---

*ก่อนหน้า: [Part 69 - Microservices Communication](part_69.md) | ต่อไป: [Part 71 - Testing Advanced Patterns](part_71.md)*
