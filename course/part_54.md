# Part 54: API Platform (REST + Webhooks)

## เป้าหมายการเรียนรู้

หลังจากจบ Part นี้ คุณจะสามารถ:
- สร้าง production-ready REST API
- API key management
- Webhooks delivery system
- API analytics and monitoring

---

## 1. API Key Management

```elixir
defmodule ApiPlatform.ApiKeys do
  alias ApiPlatform.Repo
  alias ApiPlatform.ApiKeys.ApiKey
  import Ecto.Query

  def create_key(user_id, name, scopes \\ ["read"]) do
    key = generate_key()
    hashed = hash_key(key)

    %ApiKey{}
    |> ApiKey.changeset(%{
      name: name,
      key_prefix: String.slice(key, 0, 8),
      key_hash: hashed,
      scopes: scopes,
      user_id: user_id
    })
    |> Repo.insert()
    |> case do
      {:ok, api_key} -> {:ok, api_key, key}  # return plain key only once
      error -> error
    end
  end

  def authenticate(key) do
    prefix = String.slice(key, 0, 8)

    case Repo.get_by(ApiKey, key_prefix: prefix, revoked: false) do
      nil -> {:error, :invalid_key}
      api_key ->
        if Bcrypt.verify_pass(key, api_key.key_hash) do
          update_last_used(api_key)
          {:ok, api_key}
        else
          {:error, :invalid_key}
        end
    end
  end

  def revoke_key(key_id, user_id) do
    Repo.get_by!(ApiKey, id: key_id, user_id: user_id)
    |> ApiKey.changeset(%{revoked: true, revoked_at: DateTime.utc_now()})
    |> Repo.update()
  end

  defp generate_key do
    "apk_" <> (:crypto.strong_rand_bytes(32) |> Base.encode64(padding: false))
  end

  defp hash_key(key) do
    Bcrypt.hash_pwd_salt(key)
  end

  defp update_last_used(api_key) do
    api_key
    |> ApiKey.changeset(%{
      last_used_at: DateTime.utc_now(),
      usage_count: api_key.usage_count + 1
    })
    |> Repo.update()
  end
end
```

---

## 2. API Authentication Plug

```elixir
defmodule ApiPlatformWeb.Plugs.ApiAuth do
  import Plug.Conn
  alias ApiPlatform.ApiKeys

  def init(opts), do: opts

  def call(conn, opts) do
    required_scopes = Keyword.get(opts, :scopes, [])

    case get_api_key(conn) do
      {:ok, key} ->
        case ApiKeys.authenticate(key) do
          {:ok, api_key} ->
            if has_scopes?(api_key, required_scopes) do
              conn
              |> assign(:current_api_key, api_key)
              |> assign(:current_user_id, api_key.user_id)
            else
              unauthorized(conn, "Insufficient scopes")
            end

          {:error, :invalid_key} ->
            unauthorized(conn, "Invalid API key")
        end

      {:error, :missing} ->
        unauthorized(conn, "API key required")
    end
  end

  defp get_api_key(conn) do
    case get_req_header(conn, "authorization") do
      ["Bearer " <> key] -> {:ok, key}
      ["ApiKey " <> key] -> {:ok, key}
      _ ->
        case conn.params["api_key"] do
          nil -> {:error, :missing}
          key -> {:ok, key}
        end
    end
  end

  defp has_scopes?(api_key, required) when required == [], do: true
  defp has_scopes?(api_key, required) do
    Enum.all?(required, &(&1 in api_key.scopes))
  end

  defp unauthorized(conn, message) do
    conn
    |> put_status(:unauthorized)
    |> Phoenix.Controller.json(%{error: message})
    |> halt()
  end
end
```

---

## 3. Rate Limiting ขั้นสูง

```elixir
defmodule ApiPlatformWeb.Plugs.RateLimit do
  import Plug.Conn

  # 100 requests/minute per API key
  @limit 100
  @window 60_000  # 1 minute in ms

  def init(opts), do: opts

  def call(conn, _opts) do
    key = rate_limit_key(conn)
    {allowed, remaining, reset_at} = check_rate_limit(key)

    conn = conn
    |> put_resp_header("x-ratelimit-limit", to_string(@limit))
    |> put_resp_header("x-ratelimit-remaining", to_string(remaining))
    |> put_resp_header("x-ratelimit-reset", to_string(reset_at))

    if allowed do
      conn
    else
      conn
      |> put_status(:too_many_requests)
      |> Phoenix.Controller.json(%{
        error: "Rate limit exceeded",
        retry_after: reset_at
      })
      |> halt()
    end
  end

  defp rate_limit_key(conn) do
    case conn.assigns[:current_api_key] do
      nil -> "ip:#{format_ip(conn.remote_ip)}"
      key -> "api_key:#{key.id}"
    end
  end

  defp check_rate_limit(key) do
    now = System.system_time(:millisecond)
    window_start = now - @window

    case :ets.lookup(:rate_limits, key) do
      [{^key, requests, window}] when window > window_start ->
        if length(requests) < @limit do
          new_requests = [now | requests]
          :ets.insert(:rate_limits, {key, new_requests, window})
          {true, @limit - length(new_requests), div(window + @window, 1000)}
        else
          {false, 0, div(window + @window, 1000)}
        end

      _ ->
        :ets.insert(:rate_limits, {key, [now], now})
        {true, @limit - 1, div(now + @window, 1000)}
    end
  end

  defp format_ip({a, b, c, d}), do: "#{a}.#{b}.#{c}.#{d}"
  defp format_ip(ip), do: to_string(ip)
end
```

---

## 4. Webhooks System

```elixir
defmodule ApiPlatform.Webhooks do
  alias ApiPlatform.Repo
  alias ApiPlatform.Webhooks.{Endpoint, Delivery}
  import Ecto.Query

  def register_endpoint(user_id, url, events) do
    secret = generate_secret()

    %Endpoint{}
    |> Endpoint.changeset(%{
      url: url,
      events: events,
      secret: secret,
      user_id: user_id,
      active: true
    })
    |> Repo.insert()
  end

  def dispatch_event(event_type, payload, user_id \\ nil) do
    endpoints = get_active_endpoints(event_type, user_id)

    Enum.each(endpoints, fn endpoint ->
      schedule_delivery(endpoint, event_type, payload)
    end)
  end

  defp schedule_delivery(endpoint, event_type, payload) do
    delivery_data = %{
      endpoint_id: endpoint.id,
      event_type: event_type,
      payload: payload
    }

    %ApiPlatform.Workers.WebhookDelivery{}
    |> Oban.insert!(delivery_data)
  end

  def verify_signature(payload, signature, secret) do
    expected = "sha256=" <> (:crypto.mac(:hmac, :sha256, secret, payload) |> Base.encode16(case: :lower))
    Plug.Crypto.secure_compare(expected, signature)
  end

  defp get_active_endpoints(event_type, user_id) do
    query = from(e in Endpoint,
      where: e.active == true
        and ^event_type in e.events
    )

    query = if user_id, do: where(query, [e], e.user_id == ^user_id), else: query
    Repo.all(query)
  end

  defp generate_secret do
    :crypto.strong_rand_bytes(32) |> Base.encode16(case: :lower)
  end
end
```

---

## 5. Webhook Delivery Worker

```elixir
defmodule ApiPlatform.Workers.WebhookDelivery do
  use Oban.Worker,
    queue: :webhooks,
    max_attempts: 5,
    priority: 2

  alias ApiPlatform.Repo
  alias ApiPlatform.Webhooks.{Endpoint, Delivery}

  @impl Oban.Worker
  def perform(%Oban.Job{
    args: %{"endpoint_id" => endpoint_id, "event_type" => event_type, "payload" => payload},
    attempt: attempt
  }) do
    endpoint = Repo.get!(Endpoint, endpoint_id)
    event_id = Ecto.UUID.generate()
    body = Jason.encode!(%{
      id: event_id,
      type: event_type,
      data: payload,
      created_at: DateTime.utc_now()
    })

    signature = sign_payload(body, endpoint.secret)

    case deliver(endpoint.url, body, signature) do
      {:ok, status, response_body} ->
        log_delivery(endpoint_id, event_type, event_id, :success, status, response_body)
        :ok

      {:error, reason} ->
        log_delivery(endpoint_id, event_type, event_id, :failed, nil, inspect(reason))

        # Exponential backoff: 2^attempt seconds
        backoff = :math.pow(2, attempt) |> round()
        {:snooze, backoff}
    end
  end

  defp deliver(url, body, signature) do
    headers = [
      {"Content-Type", "application/json"},
      {"X-Webhook-Signature", signature},
      {"X-Webhook-Timestamp", to_string(System.system_time(:second))},
      {"User-Agent", "ApiPlatform-Webhook/1.0"}
    ]

    case Finch.build(:post, url, headers, body) |> Finch.request(ApiPlatform.Finch, receive_timeout: 10_000) do
      {:ok, %{status: status, body: response_body}} when status in 200..299 ->
        {:ok, status, response_body}

      {:ok, %{status: status, body: response_body}} ->
        {:error, "HTTP #{status}: #{response_body}"}

      {:error, exception} ->
        {:error, exception}
    end
  end

  defp sign_payload(body, secret) do
    hmac = :crypto.mac(:hmac, :sha256, secret, body)
    "sha256=" <> Base.encode16(hmac, case: :lower)
  end

  defp log_delivery(endpoint_id, event_type, event_id, status, http_status, response) do
    %Delivery{}
    |> Delivery.changeset(%{
      endpoint_id: endpoint_id,
      event_type: event_type,
      event_id: event_id,
      status: status,
      http_status: http_status,
      response_body: response
    })
    |> Repo.insert()
  end
end
```

---

## 6. API Versioning

```elixir
# router.ex
scope "/api" do
  pipe_through [:api, :api_auth]

  scope "/v1" do
    resources "/users", V1.UsersController, only: [:index, :show]
    resources "/products", V1.ProductsController
    post "/webhooks/endpoints", V1.WebhooksController, :create
  end

  scope "/v2" do
    resources "/users", V2.UsersController, only: [:index, :show]
    resources "/products", V2.ProductsController
    resources "/webhooks", V2.WebhooksController, only: [:create, :index, :delete]
  end
end
```

---

## สรุป

```
API Platform:
├── API Keys: generate, authenticate, revoke
├── Rate Limiting: per-key, sliding window
├── Webhooks: register endpoints, dispatch events
└── Versioning: /v1, /v2

Security:
├── HMAC signing for webhooks
├── Bcrypt hashing for API keys
├── Secure comparison (timing attack prevention)
└── Rate limiting to prevent abuse

Webhook Delivery:
├── Oban background jobs
├── Retry with exponential backoff
├── Delivery logging
└── Signature verification
```

---

*ก่อนหน้า: [Part 53](part_53.md) | ต่อไป: [Part 55 - Monitoring and Observability](part_55.md)*
