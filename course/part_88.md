# Part 88: API Rate Limiting (การจำกัดอัตราการเรียก API)

## เป้าหมายการเรียนรู้

หลังจากจบ Part นี้ คุณจะสามารถ:
- Token bucket algorithm
- Sliding window rate limiting
- Rate limiting ด้วย Redis
- Tiered limits (free/pro/enterprise)

---

## 1. Token Bucket Algorithm

```elixir
defmodule MyApp.RateLimit.TokenBucket do
  @ets_table :rate_limit_buckets

  def setup do
    :ets.new(@ets_table, [:named_table, :set, :public,
      read_concurrency: true, write_concurrency: true])
  end

  def check(key, capacity, refill_rate_per_second) do
    now = System.monotonic_time(:millisecond)

    case :ets.lookup(@ets_table, key) do
      [] ->
        # First request
        :ets.insert(@ets_table, {key, capacity - 1, now})
        {:ok, capacity - 1}

      [{^key, tokens, last_refill}] ->
        # Calculate new tokens since last refill
        elapsed = (now - last_refill) / 1000  # seconds
        new_tokens = min(capacity, tokens + elapsed * refill_rate_per_second)

        if new_tokens >= 1 do
          :ets.insert(@ets_table, {key, new_tokens - 1, now})
          {:ok, floor(new_tokens - 1)}
        else
          {:error, :rate_limited, floor(1 / refill_rate_per_second * 1000)}
        end
    end
  end
end
```

---

## 2. Sliding Window with Redis

```elixir
defmodule MyApp.RateLimit.SlidingWindow do
  @redis_pool :redix

  def check(key, limit, window_seconds) do
    now = System.os_time(:millisecond)
    window_start = now - window_seconds * 1000

    redis_key = "rate_limit:#{key}"

    case Redix.pipeline(@redis_pool, [
      ["ZREMRANGEBYSCORE", redis_key, "-inf", window_start],
      ["ZCARD", redis_key],
      ["ZADD", redis_key, now, now],
      ["EXPIRE", redis_key, window_seconds + 1]
    ]) do
      {:ok, [_, count, _, _]} when count < limit ->
        {:ok, limit - count - 1}

      {:ok, [_, count, _, _]} ->
        retry_after = window_seconds  # simplified
        {:error, :rate_limited, retry_after}

      {:error, reason} ->
        # Fail open on Redis error
        {:ok, :unknown}
    end
  end
end
```

---

## 3. Rate Limiting Plug

```elixir
defmodule MyAppWeb.Plugs.RateLimit do
  import Plug.Conn
  alias MyApp.RateLimit.SlidingWindow

  def init(opts), do: opts

  def call(conn, opts) do
    limit = Keyword.get(opts, :limit, 100)
    window = Keyword.get(opts, :window, 60)
    key_fn = Keyword.get(opts, :key, &default_key/1)

    key = key_fn.(conn)

    case SlidingWindow.check(key, limit, window) do
      {:ok, remaining} ->
        conn
        |> put_resp_header("x-ratelimit-limit", to_string(limit))
        |> put_resp_header("x-ratelimit-remaining", to_string(remaining))
        |> put_resp_header("x-ratelimit-reset", to_string(System.os_time(:second) + window))

      {:error, :rate_limited, retry_after} ->
        conn
        |> put_status(429)
        |> put_resp_header("retry-after", to_string(retry_after))
        |> put_resp_header("x-ratelimit-limit", to_string(limit))
        |> put_resp_header("x-ratelimit-remaining", "0")
        |> Phoenix.Controller.json(%{
          error: "Rate limit exceeded",
          retry_after: retry_after
        })
        |> halt()
    end
  end

  defp default_key(conn) do
    user_id = conn.assigns[:current_user]?.id
    if user_id, do: "user:#{user_id}", else: "ip:#{:inet.ntoa(conn.remote_ip)}"
  end
end

# Router usage
pipeline :api_with_rate_limit do
  plug MyAppWeb.Plugs.RateLimit, limit: 100, window: 60
end

pipeline :strict_rate_limit do
  plug MyAppWeb.Plugs.RateLimit, limit: 10, window: 60
end
```

---

## 4. Tiered Rate Limits

```elixir
defmodule MyApp.RateLimit.Tiered do
  @tiers %{
    "free" =>       %{requests_per_minute: 60,    requests_per_day: 1_000},
    "pro" =>        %{requests_per_minute: 600,   requests_per_day: 50_000},
    "enterprise" => %{requests_per_minute: 6_000, requests_per_day: 500_000}
  }

  def check(user) do
    tier = get_user_tier(user)
    limits = @tiers[tier]

    with {:ok, _} <- check_window("#{user.id}:minute", limits.requests_per_minute, 60),
         {:ok, _} <- check_window("#{user.id}:day", limits.requests_per_day, 86400) do
      :ok
    end
  end

  defp check_window(key, limit, window) do
    MyApp.RateLimit.SlidingWindow.check("rate:#{key}", limit, window)
  end

  defp get_user_tier(user) do
    case MyApp.Billing.get_active_subscription(user) do
      %{plan: %{name: "Enterprise"}} -> "enterprise"
      %{plan: %{name: "Pro"}} -> "pro"
      _ -> "free"
    end
  end

  def get_limits(user) do
    tier = get_user_tier(user)
    @tiers[tier] |> Map.put(:tier, tier)
  end
end
```

---

## 5. Rate Limit Headers and Client Handling

```elixir
defmodule MyAppWeb.Plugs.APIRateLimit do
  import Plug.Conn

  def init(opts), do: opts

  def call(conn, _opts) do
    case conn.assigns[:current_api_key] do
      nil ->
        conn

      api_key ->
        limits = MyApp.RateLimit.Tiered.get_limits(api_key.user)

        register_before_send(conn, fn conn ->
          case MyApp.RateLimit.Tiered.check(api_key.user) do
            :ok ->
              conn
              |> put_resp_header("x-ratelimit-tier", limits.tier)
              |> put_resp_header("x-ratelimit-limit-minute", to_string(limits.requests_per_minute))
              |> put_resp_header("x-ratelimit-limit-day", to_string(limits.requests_per_day))

            {:error, :rate_limited, _} ->
              %{conn |
                status: 429,
                resp_body: Jason.encode!(%{
                  error: "Rate limit exceeded",
                  limits: limits,
                  upgrade_url: "/pricing"
                })
              }
          end
        end)
    end
  end
end
```

---

## สรุป

```
Rate Limiting Algorithms:
├── Token bucket: burst-friendly, refills over time
├── Sliding window: accurate, Redis-backed
├── Fixed window: simple, less accurate
└── Leaky bucket: constant output rate

Implementation:
├── ETS: fast local (single node)
├── Redis: distributed (multi-node)
└── Cache fallback: fail open on Redis error

Headers (RFC 6585):
├── X-RateLimit-Limit: max requests
├── X-RateLimit-Remaining: left in window
├── X-RateLimit-Reset: when window resets
└── Retry-After: when limit exceeded (429)

Tiers:
├── Free: 60/min, 1K/day
├── Pro: 600/min, 50K/day
└── Enterprise: 6K/min, 500K/day
```

---

*ก่อนหน้า: [Part 87](part_87.md) | ต่อไป: [Part 89 - GraphQL Subscriptions](part_89.md)*
