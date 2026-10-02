# Part 77: Phoenix Deployment Patterns (รูปแบบการ Deploy)

## เป้าหมายการเรียนรู้

หลังจากจบ Part นี้ คุณจะสามารถ:
- Zero-downtime deployment
- Blue-green deployment
- Database migration strategies
- Rollback strategies

---

## 1. Release Configuration

```elixir
# config/runtime.exs
import Config

config :my_app, MyApp.Repo,
  url: System.fetch_env!("DATABASE_URL"),
  pool_size: String.to_integer(System.get_env("POOL_SIZE", "10")),
  ssl: System.get_env("DB_SSL", "false") == "true"

config :my_app, MyAppWeb.Endpoint,
  http: [ip: {0, 0, 0, 0, 0, 0, 0, 0}, port: String.to_integer(System.get_env("PORT", "4000"))],
  secret_key_base: System.fetch_env!("SECRET_KEY_BASE"),
  url: [host: System.fetch_env!("PHX_HOST"), port: 443, scheme: "https"],
  server: true

# mix.exs
defp releases do
  [
    my_app: [
      version: "0.1.0",
      applications: [runtime_tools: :permanent],
      include_executables_for: [:unix],
      steps: [:assemble, :tar]
    ]
  ]
end
```

---

## 2. Dockerfile for Production

```dockerfile
# Multi-stage build
FROM hexpm/elixir:1.16.1-erlang-26.2.3-debian-bookworm-20240130 AS builder

WORKDIR /app

# Install build tools
RUN apt-get update && apt-get install -y build-essential git nodejs npm

# Install Hex and Rebar
RUN mix local.hex --force && mix local.rebar --force

# Set build environment
ENV MIX_ENV=prod

# Install dependencies
COPY mix.exs mix.lock ./
RUN mix deps.get --only prod
RUN mix deps.compile

# Compile assets
COPY assets assets
RUN mix assets.deploy

# Compile app
COPY config config
COPY lib lib
COPY priv priv
RUN mix compile

# Build release
RUN mix release

# --- Runtime stage ---
FROM debian:bookworm-slim

WORKDIR /app

RUN apt-get update && apt-get install -y libstdc++6 openssl libncurses5 locales \
  && rm -rf /var/lib/apt/lists/*

# Set locale
RUN sed -i '/en_US.UTF-8/s/^# //g' /etc/locale.gen && locale-gen
ENV LANG=en_US.UTF-8

# Copy release
COPY --from=builder /app/_build/prod/rel/my_app ./

# Create non-root user
RUN groupadd -r app && useradd -r -g app app && chown -R app:app /app
USER app

EXPOSE 4000

CMD ["bin/my_app", "start"]
```

---

## 3. Zero-downtime Migration

```elixir
# Zero-downtime migration strategy:
# 1. Add column as nullable first
# 2. Backfill existing data
# 3. Add constraint after backfill

defmodule MyApp.Repo.Migrations.AddUserTypeColumn do
  use Ecto.Migration

  # Phase 1: Add nullable column (safe, no lock)
  def up do
    alter table(:users) do
      add :user_type, :string, null: true
    end
  end

  def down do
    alter table(:users) do
      remove :user_type
    end
  end
end

# Phase 2: Backfill script (run separately)
defmodule MyApp.Migrations.BackfillUserType do
  alias MyApp.Repo
  import Ecto.Query

  def run(batch_size \\ 1000) do
    from(u in MyApp.User, where: is_nil(u.user_type))
    |> batch_update(batch_size)
  end

  defp batch_update(query, batch_size) do
    ids = Repo.all(from(u in query, select: u.id, limit: ^batch_size))

    if ids == [] do
      :done
    else
      Repo.update_all(
        from(u in MyApp.User, where: u.id in ^ids),
        set: [user_type: "regular"]
      )
      batch_update(query, batch_size)
    end
  end
end

# Phase 3: Add constraint (after backfill)
defmodule MyApp.Repo.Migrations.AddUserTypeConstraint do
  use Ecto.Migration

  def change do
    alter table(:users) do
      modify :user_type, :string, null: false, default: "regular"
    end
  end
end
```

---

## 4. Blue-Green Deployment Script

```bash
#!/bin/bash
# deploy.sh - Blue-green deployment

CURRENT=$(cat /etc/nginx/current-color 2>/dev/null || echo "blue")
NEW=$([ "$CURRENT" = "blue" ] && echo "green" || echo "blue")

echo "Deploying to $NEW (current: $CURRENT)"

# Deploy new version
docker pull my-registry/my-app:$VERSION
docker-compose -f docker-compose.$NEW.yml up -d

# Wait for health check
MAX_RETRIES=30
for i in $(seq 1 $MAX_RETRIES); do
  STATUS=$(curl -s -o /dev/null -w "%{http_code}" http://localhost:$(get_port $NEW)/health)
  if [ "$STATUS" = "200" ]; then
    echo "New version healthy"
    break
  fi
  echo "Waiting for health check... ($i/$MAX_RETRIES)"
  sleep 2

  if [ $i -eq $MAX_RETRIES ]; then
    echo "Health check failed, rolling back"
    docker-compose -f docker-compose.$NEW.yml down
    exit 1
  fi
done

# Switch nginx to new color
sed -i "s/upstream app_server.*/upstream app_server { server localhost:$(get_port $NEW); }/" /etc/nginx/nginx.conf
nginx -s reload

# Record current color
echo $NEW > /etc/nginx/current-color

# Stop old version after traffic drained
sleep 10
docker-compose -f docker-compose.$CURRENT.yml down

echo "Deployment complete. Running: $NEW"
```

---

## 5. Health Check Endpoint

```elixir
defmodule MyAppWeb.HealthController do
  use MyAppWeb, :controller

  def liveness(conn, _params) do
    # Simple check: app is running
    json(conn, %{status: "ok", timestamp: DateTime.utc_now()})
  end

  def readiness(conn, _params) do
    # Thorough check: dependencies available
    checks = %{
      database: check_database(),
      cache: check_cache()
    }

    all_ok = Enum.all?(checks, fn {_, status} -> status == :ok end)

    status = if all_ok, do: 200, else: 503
    conn
    |> put_status(status)
    |> json(%{
      status: if(all_ok, do: "ok", else: "degraded"),
      checks: Map.new(checks, fn {k, v} -> {k, to_string(v)} end),
      timestamp: DateTime.utc_now()
    })
  end

  defp check_database do
    case Ecto.Adapters.SQL.query(MyApp.Repo, "SELECT 1", []) do
      {:ok, _} -> :ok
      _ -> :error
    end
  end

  defp check_cache do
    case Cachex.get(:app_cache, "_health_check") do
      {:ok, _} -> :ok
      _ -> :error
    end
  rescue
    _ -> :error
  end
end

# router.ex
scope "/health" do
  get "/live", HealthController, :liveness
  get "/ready", HealthController, :readiness
end
```

---

## สรุป

```
Deployment Strategies:
├── Rolling: gradual instance replacement
├── Blue-green: full switch with rollback
└── Canary: gradual traffic shift

Zero-downtime DB Migrations:
├── Phase 1: Add nullable column
├── Phase 2: Backfill data
└── Phase 3: Add constraint

Health Checks:
├── /health/live: process is alive
└── /health/ready: all deps connected

Kubernetes probes:
├── livenessProbe: /health/live
└── readinessProbe: /health/ready
```

---

*ก่อนหน้า: [Part 76](part_76.md) | ต่อไป: [Part 78 - Performance Profiling](part_78.md)*
