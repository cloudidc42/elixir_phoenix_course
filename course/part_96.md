# Part 96: CI/CD Pipeline (การ Deploy อัตโนมัติ)

## เป้าหมายการเรียนรู้

หลังจากจบ Part นี้ คุณจะสามารถ:
- GitHub Actions สำหรับ Elixir
- Automated testing pipeline
- Docker build และ push
- Deploy to Fly.io

---

## 1. GitHub Actions CI

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

env:
  MIX_ENV: test
  ELIXIR_VERSION: "1.16"
  OTP_VERSION: "26.2"

jobs:
  test:
    name: Test
    runs-on: ubuntu-latest

    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_USER: postgres
          POSTGRES_PASSWORD: postgres
          POSTGRES_DB: my_app_test
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432

    steps:
      - uses: actions/checkout@v4

      - name: Set up Elixir
        uses: erlef/setup-beam@v1
        with:
          elixir-version: ${{ env.ELIXIR_VERSION }}
          otp-version: ${{ env.OTP_VERSION }}

      - name: Restore deps cache
        uses: actions/cache@v4
        with:
          path: |
            deps
            _build
          key: ${{ runner.os }}-mix-${{ hashFiles('**/mix.lock') }}
          restore-keys: ${{ runner.os }}-mix-

      - name: Install dependencies
        run: mix deps.get

      - name: Check formatting
        run: mix format --check-formatted

      - name: Run Credo
        run: mix credo --strict

      - name: Compile (warnings as errors)
        run: mix compile --warnings-as-errors

      - name: Run database migrations
        run: mix ecto.create && mix ecto.migrate
        env:
          DATABASE_URL: postgres://postgres:postgres@localhost/my_app_test

      - name: Run tests with coverage
        run: mix test --cover
        env:
          DATABASE_URL: postgres://postgres:postgres@localhost/my_app_test

      - name: Upload coverage
        uses: codecov/codecov-action@v4
        with:
          token: ${{ secrets.CODECOV_TOKEN }}

  dialyzer:
    name: Dialyzer
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Set up Elixir
        uses: erlef/setup-beam@v1
        with:
          elixir-version: ${{ env.ELIXIR_VERSION }}
          otp-version: ${{ env.OTP_VERSION }}

      - name: Restore PLT cache
        uses: actions/cache@v4
        with:
          path: priv/plts
          key: ${{ runner.os }}-dialyzer-${{ hashFiles('**/mix.lock') }}

      - name: Install deps
        run: mix deps.get

      - name: Run Dialyzer
        run: mix dialyzer --format github
```

---

## 2. Docker Multi-stage Build

```dockerfile
# Dockerfile
ARG ELIXIR_VERSION=1.16
ARG OTP_VERSION=26.2
ARG DEBIAN_VERSION=bookworm-20240701-slim

ARG BUILDER_IMAGE="hexpm/elixir:${ELIXIR_VERSION}-erlang-${OTP_VERSION}-debian-${DEBIAN_VERSION}"
ARG RUNNER_IMAGE="debian:${DEBIAN_VERSION}"

FROM ${BUILDER_IMAGE} AS builder

# Install build dependencies
RUN apt-get update -y && \
    apt-get install -y build-essential git nodejs npm && \
    apt-get clean && rm -rf /var/lib/apt/lists/*

WORKDIR /app

ENV MIX_ENV=prod

# Install hex and rebar
RUN mix local.hex --force && mix local.rebar --force

# Copy and install deps
COPY mix.exs mix.lock ./
RUN mix deps.get --only prod

# Copy config
COPY config config

# Compile deps
RUN mix deps.compile

# Copy assets and compile
COPY assets assets
RUN cd assets && npm ci --progress=false --no-audit
RUN mix assets.deploy

# Copy source
COPY lib lib
COPY priv priv

# Compile app
RUN mix compile

# Build release
COPY rel rel
RUN mix release

# ---- Runner image ----
FROM ${RUNNER_IMAGE} AS runner

RUN apt-get update -y && \
    apt-get install -y libstdc++6 openssl libncurses5 locales ca-certificates && \
    apt-get clean && rm -rf /var/lib/apt/lists/*

RUN sed -i '/en_US.UTF-8/s/^# //g' /etc/locale.gen && locale-gen
ENV LANG=en_US.UTF-8 LANGUAGE=en_US:en LC_ALL=en_US.UTF-8

WORKDIR /app
RUN chown nobody /app

ENV MIX_ENV=prod

# Copy release from builder
COPY --from=builder --chown=nobody:root /app/_build/prod/rel/my_app ./

USER nobody

CMD ["/app/bin/server"]
```

---

## 3. Deploy to Fly.io

```toml
# fly.toml
app = "my-app-prod"
primary_region = "sin"  # Singapore

[build]
  dockerfile = "Dockerfile"

[env]
  PHX_HOST = "my-app-prod.fly.dev"
  PORT = "8080"
  POOL_SIZE = "10"

[http_service]
  internal_port = 8080
  force_https = true
  auto_stop_machines = true
  auto_start_machines = true
  min_machines_running = 1

  [http_service.concurrency]
    type = "connections"
    hard_limit = 1000
    soft_limit = 800

[[vm]]
  memory = "512mb"
  cpu_kind = "shared"
  cpus = 1

[checks]
  [checks.health]
    grace_period = "30s"
    interval = "15s"
    method = "GET"
    path = "/health/live"
    timeout = "5s"
    type = "http"
```

```yaml
# .github/workflows/deploy.yml
name: Deploy to Fly.io

on:
  push:
    branches: [main]

jobs:
  deploy:
    name: Deploy
    runs-on: ubuntu-latest
    needs: [test]  # Only deploy if tests pass

    steps:
      - uses: actions/checkout@v4

      - name: Set up Flyctl
        uses: superfly/flyctl-actions/setup-flyctl@master

      - name: Run DB migrations
        run: flyctl ssh console -a my-app-prod -C "/app/bin/my_app eval 'MyApp.Release.migrate()'"
        env:
          FLY_API_TOKEN: ${{ secrets.FLY_API_TOKEN }}

      - name: Deploy
        run: flyctl deploy --remote-only
        env:
          FLY_API_TOKEN: ${{ secrets.FLY_API_TOKEN }}
```

---

## 4. Release Migration Module

```elixir
# lib/my_app/release.ex
defmodule MyApp.Release do
  @app :my_app

  def migrate do
    load_app()

    for repo <- repos() do
      {:ok, _, _} = Ecto.Migrator.with_repo(repo, &Ecto.Migrator.run(&1, :up, all: true))
    end
  end

  def rollback(repo, version) do
    load_app()
    {:ok, _, _} = Ecto.Migrator.with_repo(repo, &Ecto.Migrator.run(&1, :down, to: version))
  end

  defp repos do
    Application.fetch_env!(@app, :ecto_repos)
  end

  defp load_app do
    Application.load(@app)
  end
end

# Run manually: /app/bin/my_app eval "MyApp.Release.migrate()"
```

---

## 5. Environment Secrets Setup

```bash
# Set Fly.io secrets
flyctl secrets set \
  SECRET_KEY_BASE=$(mix phx.gen.secret) \
  DATABASE_URL=postgres://... \
  STRIPE_SECRET_KEY=sk_live_... \
  SENTRY_DSN=https://...

# List set secrets (values hidden)
flyctl secrets list

# Rotate a secret
flyctl secrets set SECRET_KEY_BASE=$(mix phx.gen.secret)
```

---

## สรุป

```
CI/CD Pipeline:
├── Test: format, credo, compile, test, cover
├── Dialyzer: static type analysis
├── Build: Docker multi-stage
└── Deploy: Fly.io with blue-green

GitHub Actions:
├── Services: Postgres for integration tests
├── Cache: deps, _build, PLT files
└── Secrets: API tokens, deploy keys

Release:
├── Distillery/Mix release: self-contained
├── Runtime config: runtime.exs
└── Migration: run before deploy

Fly.io:
├── fly.toml: app config
├── Secrets: flyctl secrets set
├── Health checks: /health/live
└── Scale: fly scale vm/count
```

---

*ก่อนหน้า: [Part 95](part_95.md) | ต่อไป: [Part 97 - WebRTC และ Real-time Media](part_97.md)*
