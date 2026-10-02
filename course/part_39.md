# Part 39: Deployment ด้วย Fly.io และ Docker

## เป้าหมายการเรียนรู้

หลังจากจบ Part นี้ คุณจะสามารถ:
- Build Phoenix release
- Deploy ด้วย Docker
- Deploy บน Fly.io
- จัดการ secrets และ environment variables

---

## 1. Phoenix Release

```elixir
# mix.exs
def project do
  [
    releases: [
      my_app: [
        include_executables_for: [:unix],
        applications: [runtime_tools: :permanent]
      ]
    ]
  ]
end
```

```bash
# Build release
MIX_ENV=prod mix assets.deploy
MIX_ENV=prod mix release

# Release อยู่ที่
_build/prod/rel/my_app/

# รัน
SECRET_KEY_BASE=xxx DATABASE_URL=postgres://... PORT=4000 \
  _build/prod/rel/my_app/bin/my_app start
```

---

## 2. Dockerfile

```dockerfile
# Dockerfile
ARG ELIXIR_VERSION=1.16.0
ARG OTP_VERSION=26.2.1
ARG DEBIAN_VERSION=bullseye-20231009-slim

ARG BUILDER_IMAGE="hexpm/elixir:${ELIXIR_VERSION}-erlang-${OTP_VERSION}-debian-${DEBIAN_VERSION}"
ARG RUNNER_IMAGE="debian:${DEBIAN_VERSION}"

FROM ${BUILDER_IMAGE} as builder

# Install hex + rebar
RUN mix local.hex --force && \
    mix local.rebar --force

WORKDIR /app

# Install mix dependencies
COPY mix.exs mix.lock ./
RUN mix deps.get --only $MIX_ENV

# Copy compile config files
COPY config/config.exs config/${MIX_ENV}.exs config/
RUN mix deps.compile

# Build assets
COPY priv priv
COPY assets assets
RUN mix assets.deploy

# Compile and build release
COPY lib lib
RUN mix compile

COPY config/runtime.exs config/
RUN mix release

# Start a new build stage
FROM ${RUNNER_IMAGE}

RUN apt-get update -y && \
  apt-get install -y libstdc++6 openssl libncurses5 locales ca-certificates \
  && apt-get clean && rm -f /var/lib/apt/lists/*_*

RUN sed -i '/en_US.UTF-8/s/^# //g' /etc/locale.gen && locale-gen

ENV LANG en_US.UTF-8
ENV LANGUAGE en_US:en
ENV LC_ALL en_US.UTF-8

WORKDIR "/app"
RUN chown nobody /app

# Only copy the final release
COPY --from=builder --chown=nobody:root /app/_build/prod/rel/my_app ./

USER nobody

CMD ["/app/bin/server"]
```

```bash
# Build Docker image
docker build --build-arg MIX_ENV=prod -t my-app .

# รัน Docker container
docker run -p 4000:4000 \
  -e SECRET_KEY_BASE=xxx \
  -e DATABASE_URL=postgres://... \
  -e PHX_HOST=myapp.com \
  my-app
```

---

## 3. docker-compose.yml

```yaml
version: '3.8'

services:
  app:
    build:
      context: .
      args:
        MIX_ENV: prod
    image: my-app:latest
    depends_on:
      db:
        condition: service_healthy
    ports:
      - "4000:4000"
    environment:
      DATABASE_URL: "postgres://postgres:postgres@db/my_app_prod"
      SECRET_KEY_BASE: "${SECRET_KEY_BASE}"
      PHX_HOST: "${PHX_HOST:-localhost}"
      PORT: "4000"
      PHX_SERVER: "true"
    restart: unless-stopped

  db:
    image: postgres:15-alpine
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: my_app_prod
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 5s
      retries: 5
    restart: unless-stopped

  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/conf.d/default.conf
      - certbot_conf:/etc/letsencrypt
    depends_on:
      - app
    restart: unless-stopped

volumes:
  postgres_data:
  certbot_conf:
```

---

## 4. Deploy ด้วย Fly.io

```bash
# ติดตั้ง flyctl
curl -L https://fly.io/install.sh | sh

# Login
fly auth login

# สร้าง app
fly launch

# fly.toml ที่ถูกสร้าง
```

```toml
# fly.toml
app = "my-app"
primary_region = "sin"  # Singapore

[build]

[env]
  PHX_HOST = "my-app.fly.dev"
  PORT = "8080"

[http_service]
  internal_port = 8080
  force_https = true
  auto_stop_machines = true
  auto_start_machines = true
  min_machines_running = 0

  [http_service.concurrency]
    type = "connections"
    hard_limit = 1000
    soft_limit = 750

[[vm]]
  cpu_kind = "shared"
  cpus = 1
  memory_mb = 512
```

```bash
# สร้าง PostgreSQL database
fly postgres create --name my-app-db --region sin

# Attach database
fly postgres attach --app my-app my-app-db

# Set secrets
fly secrets set SECRET_KEY_BASE=$(mix phx.gen.secret)
fly secrets set PHX_HOST=my-app.fly.dev

# Deploy
fly deploy

# Run migrations
fly ssh console
> /app/bin/migrate

# หรือ
fly ssh console -C "/app/bin/my_app eval 'MyApp.Release.migrate'"

# Logs
fly logs

# Scale
fly scale count 2         # 2 instances
fly scale vm shared-cpu-2x  # เพิ่ม CPU

# Open app
fly open
```

---

## 5. Release Tasks (Migrations)

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

  def seed do
    load_app()
    for repo <- repos() do
      {:ok, _, _} = Ecto.Migrator.with_repo(repo, fn _repo ->
        seed_path = Application.app_dir(@app, "priv/repo/seeds.exs")
        Code.eval_file(seed_path)
      end)
    end
  end

  defp repos do
    Application.fetch_env!(@app, :ecto_repos)
  end

  defp load_app do
    Application.load(@app)
  end
end
```

```elixir
# rel/overlays/bin/migrate
#!/bin/sh
cd -P -- "$(dirname -- "$0")/.."
exec ./bin/my_app eval "MyApp.Release.migrate"
```

---

## 6. Environment Configuration

```elixir
# config/runtime.exs
import Config

if config_env() == :prod do
  database_url =
    System.get_env("DATABASE_URL") ||
      raise """
      environment variable DATABASE_URL is missing.
      For example: ecto://USER:PASS@HOST/DATABASE
      """

  maybe_ipv6 = if System.get_env("ECTO_IPV6") in ~w(true 1), do: [:inet6], else: []

  config :my_app, MyApp.Repo,
    url: database_url,
    pool_size: String.to_integer(System.get_env("POOL_SIZE") || "10"),
    socket_options: maybe_ipv6

  secret_key_base =
    System.get_env("SECRET_KEY_BASE") ||
      raise "SECRET_KEY_BASE environment variable is missing."

  host = System.get_env("PHX_HOST") || "example.com"
  port = String.to_integer(System.get_env("PORT") || "4000")

  config :my_app, :dns_cluster_query, System.get_env("DNS_CLUSTER_QUERY")

  config :my_app, MyAppWeb.Endpoint,
    url: [host: host, port: 443, scheme: "https"],
    http: [
      ip: {0, 0, 0, 0, 0, 0, 0, 0},
      port: port
    ],
    secret_key_base: secret_key_base,
    server: true
end
```

---

## 7. CI/CD ด้วย GitHub Actions

```yaml
# .github/workflows/deploy.yml
name: Deploy

on:
  push:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest

    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_PASSWORD: postgres
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432

    steps:
      - uses: actions/checkout@v4
      - uses: erlef/setup-beam@v1
        with:
          elixir-version: '1.16'
          otp-version: '26'

      - name: Cache deps
        uses: actions/cache@v3
        with:
          path: deps
          key: ${{ runner.os }}-mix-${{ hashFiles('**/mix.lock') }}

      - run: mix deps.get
      - run: mix compile
      - run: mix ecto.create
        env:
          MIX_ENV: test
          DATABASE_URL: postgres://postgres:postgres@localhost/my_app_test
      - run: mix ecto.migrate
        env:
          MIX_ENV: test
          DATABASE_URL: postgres://postgres:postgres@localhost/my_app_test
      - run: mix test
        env:
          DATABASE_URL: postgres://postgres:postgres@localhost/my_app_test

  deploy:
    needs: test
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4
      - uses: superfly/flyctl-actions/setup-flyctl@master
      - run: flyctl deploy --remote-only
        env:
          FLY_API_TOKEN: ${{ secrets.FLY_API_TOKEN }}
```

---

## สรุป

```
Deployment:
├── mix release - build self-contained app
├── Docker - containerization
├── Fly.io - PaaS ที่รองรับ Elixir
└── GitHub Actions - CI/CD

Environment:
├── config/runtime.exs สำหรับ runtime config
├── fly secrets set สำหรับ secrets
└── DATABASE_URL, SECRET_KEY_BASE, PHX_HOST

Migrations:
├── MyApp.Release.migrate
├── รันก่อน start app
└── fly ssh console สำหรับ manual migration
```

---

*ก่อนหน้า: [Part 38](part_38.md) | ต่อไป: [Part 40 - Performance Optimization](part_40.md)*
