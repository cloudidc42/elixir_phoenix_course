# Part 25: Phoenix Framework - ติดตั้งและโครงสร้าง

## เป้าหมายการเรียนรู้

หลังจากจบ Part นี้ คุณจะสามารถ:
- ติดตั้ง Phoenix Framework ได้
- เข้าใจโครงสร้าง Phoenix project
- เข้าใจ Application Supervisor tree
- กำหนด Configuration ได้
- สร้าง Blog Application เป็นตัวอย่าง

---

## 1. Phoenix คืออะไร?

Phoenix เป็น web framework สำหรับ Elixir ที่ออกแบบมาเพื่อ:
- **Performance**: รองรับ request ล้านๆ ต่อวินาที
- **Productivity**: มี conventions ที่ชัดเจน
- **Real-time**: LiveView, Channels (WebSocket)
- **Scalability**: สร้างบน Erlang/OTP

### เปรียบเทียบกับ frameworks อื่น

```
Phoenix vs Rails vs Django:
├── Phoenix: Elixir/Erlang, functional, real-time first
├── Rails: Ruby, MVC, full-featured
└── Django: Python, batteries included

Performance:
├── Phoenix: ~1-2ms response, ~1M concurrent users
├── Rails: ~10-50ms response
└── Django: ~5-30ms response
```

---

## 2. ติดตั้ง Phoenix

### Prerequisites

```bash
# ต้องมี Elixir 1.14+
elixir --version

# ต้องมี Erlang/OTP 24+
erl --version

# Node.js สำหรับ assets (optional)
node --version

# PostgreSQL (default database)
psql --version
```

### ติดตั้ง Phoenix Generator

```bash
# ติดตั้ง phx_new archive
mix archive.install hex phx_new

# ตรวจสอบ
mix phx.new --version
```

---

## 3. สร้าง Phoenix Project

```bash
# สร้าง project ใหม่
mix phx.new blog

# สร้างโดยไม่มี Ecto (ไม่ใช้ database)
mix phx.new blog --no-ecto

# สร้าง API-only project (ไม่มี HTML)
mix phx.new blog_api --no-html --no-assets

# สร้าง project พร้อม LiveView
mix phx.new blog --live
```

```bash
# เข้าไปใน project
cd blog

# สร้าง database
mix ecto.create

# รัน development server
mix phx.server

# หรือรันใน IEx
iex -S mix phx.server
```

เปิด browser ไปที่ http://localhost:4000

---

## 4. โครงสร้าง Phoenix Project

```
blog/
├── assets/              # Frontend assets (JS, CSS)
│   ├── css/
│   │   └── app.css
│   ├── js/
│   │   └── app.js
│   └── vendor/
├── config/              # Configuration files
│   ├── config.exs       # Base config
│   ├── dev.exs          # Development config
│   ├── prod.exs         # Production config
│   ├── runtime.exs      # Runtime config
│   └── test.exs         # Test config
├── lib/
│   ├── blog/            # Business logic (context)
│   │   ├── posts.ex     # Posts context
│   │   └── accounts.ex  # Accounts context
│   └── blog_web/        # Web layer
│       ├── controllers/ # HTTP controllers
│       ├── views/       # View modules (Phoenix 1.6)
│       ├── components/  # Components (Phoenix 1.7+)
│       ├── templates/   # HEEx templates
│       ├── live/        # LiveView modules
│       ├── router.ex    # Routes definition
│       ├── endpoint.ex  # HTTP endpoint config
│       └── telemetry.ex # Metrics/telemetry
├── priv/
│   ├── repo/
│   │   └── migrations/  # Database migrations
│   └── static/          # Static files
├── test/                # Tests
├── mix.exs              # Dependencies
└── mix.lock             # Lock file
```

---

## 5. Application Supervisor Tree

```elixir
# lib/blog/application.ex
defmodule Blog.Application do
  use Application

  @impl true
  def start(_type, _args) do
    children = [
      # Ecto Repo (database connection pool)
      Blog.Repo,

      # DNS clustering (สำหรับ distributed deployments)
      {DNSCluster, query: Application.get_env(:blog, :dns_cluster_query) || :ignore},

      # Phoenix PubSub
      {Phoenix.PubSub, name: Blog.PubSub},

      # Finch HTTP client
      {Finch, name: Blog.Finch},

      # Phoenix Endpoint (web server)
      BlogWeb.Endpoint
    ]

    opts = [strategy: :one_for_one, name: Blog.Supervisor]
    Supervisor.start_link(children, opts)
  end

  @impl true
  def config_change(changed, _new, removed) do
    BlogWeb.Endpoint.config_change(changed, removed)
    :ok
  end
end
```

```
Supervisor Tree:
Blog.Supervisor
├── Blog.Repo              # DB connection pool
├── Blog.PubSub            # Phoenix PubSub
├── Blog.Finch             # HTTP client
└── BlogWeb.Endpoint       # Web server
    ├── Plug.Cowboy        # HTTP adapter
    └── Phoenix.CodeReloader # Dev hot-reload
```

---

## 6. Endpoint

Endpoint คือ entry point ของทุก HTTP request

```elixir
# lib/blog_web/endpoint.ex
defmodule BlogWeb.Endpoint do
  use Phoenix.Endpoint, otp_app: :blog

  # Session
  @session_options [
    store: :cookie,
    key: "_blog_key",
    signing_salt: "abc123",
    same_site: "Lax"
  ]

  socket "/live", Phoenix.LiveView.Socket,
    websocket: [connect_info: [session: @session_options]],
    longpoll: [connect_info: [session: @session_options]]

  # Static files
  plug Plug.Static,
    at: "/",
    from: :blog,
    gzip: false,
    only: BlogWeb.static_paths()

  # Code reloading (dev only)
  if code_reloading? do
    socket "/phoenix/live_reload/socket", Phoenix.LiveReloader.Socket
    plug Phoenix.LiveReloader
    plug Phoenix.CodeReloader
    plug Phoenix.Ecto.CheckRepoStatus, otp_app: :blog
  end

  plug Plug.RequestId
  plug Plug.Telemetry, event_prefix: [:phoenix, :endpoint]

  plug Plug.Parsers,
    parsers: [:urlencoded, :multipart, :json],
    pass: ["*/*"],
    json_decoder: Phoenix.json_library()

  plug Plug.MethodOverride
  plug Plug.Head
  plug Plug.Session, @session_options

  # Router
  plug BlogWeb.Router
end
```

---

## 7. Configuration

### config/config.exs - Base Configuration

```elixir
# config/config.exs
import Config

config :blog,
  ecto_repos: [Blog.Repo],
  generators: [timestamp_type: :utc_datetime]

config :blog, BlogWeb.Endpoint,
  url: [host: "localhost"],
  adapter: Bandit.PhoenixAdapter,
  render_errors: [
    formats: [html: BlogWeb.ErrorHTML, json: BlogWeb.ErrorJSON],
    layout: false
  ],
  pubsub_server: Blog.PubSub,
  live_view: [signing_salt: "abc123"]

config :logger, :console,
  format: "$time $metadata[$level] $message\n",
  metadata: [:request_id]

config :phoenix, :json_library, Jason

# Import environment-specific config
import_config "#{config_env()}.exs"
```

### config/dev.exs - Development

```elixir
# config/dev.exs
import Config

# Database
config :blog, Blog.Repo,
  username: "postgres",
  password: "postgres",
  hostname: "localhost",
  database: "blog_dev",
  stacktrace: true,
  show_sensitive_data_on_connection_error: true,
  pool_size: 10

# Endpoint
config :blog, BlogWeb.Endpoint,
  http: [ip: {127, 0, 0, 1}, port: 4000],
  check_origin: false,
  code_reloader: true,
  debug_errors: true,
  secret_key_base: "dev_secret_key_base_abc123...",
  watchers: [
    esbuild: {Esbuild, :install_and_run, [:blog, ~w(--sourcemap=inline --watch)]},
    tailwind: {Tailwind, :install_and_run, [:blog, ~w(--watch)]}
  ]

# Live reload
config :blog, BlogWeb.Endpoint,
  live_reload: [
    patterns: [
      ~r"priv/static/(?!uploads/).*(js|css|png|jpeg|jpg|gif|svg)$",
      ~r"priv/gettext/.*(po)$",
      ~r"lib/blog_web/(controllers|live|components)/.*(ex|heex)$"
    ]
  ]

config :logger, :console, format: "[$level] $message\n"

config :phoenix, :stacktrace_depth, 20
config :phoenix, :plug_init_mode, :runtime
```

### config/prod.exs - Production

```elixir
# config/prod.exs
import Config

config :blog, BlogWeb.Endpoint,
  cache_static_manifest: "priv/static/cache_manifest.json"

config :logger, level: :info

config :phoenix, :serve_endpoints, true
```

### config/runtime.exs - Runtime (12-factor app)

```elixir
# config/runtime.exs - อ่าน environment variables ตอน runtime
import Config

if config_env() == :prod do
  database_url =
    System.get_env("DATABASE_URL") ||
      raise """
      environment variable DATABASE_URL is missing.
      For example: ecto://USER:PASS@HOST/DATABASE
      """

  maybe_ipv6 = if System.get_env("ECTO_IPV6") in ~w(true 1), do: [:inet6], else: []

  config :blog, Blog.Repo,
    url: database_url,
    pool_size: String.to_integer(System.get_env("POOL_SIZE") || "10"),
    socket_options: maybe_ipv6

  secret_key_base =
    System.get_env("SECRET_KEY_BASE") ||
      raise """
      environment variable SECRET_KEY_BASE is missing.
      You can generate one by calling: mix phx.gen.secret
      """

  host = System.get_env("PHX_HOST") || "example.com"
  port = String.to_integer(System.get_env("PORT") || "4000")

  config :blog, BlogWeb.Endpoint,
    url: [host: host, port: 443, scheme: "https"],
    http: [
      ip: {0, 0, 0, 0, 0, 0, 0, 0},
      port: port
    ],
    secret_key_base: secret_key_base
end
```

---

## 8. Mix Tasks ที่มีประโยชน์

```bash
# สร้าง project ใหม่
mix phx.new APP_NAME

# สร้าง Database
mix ecto.create

# ลบ Database
mix ecto.drop

# สร้าง migration
mix ecto.gen.migration create_posts

# รัน migrations
mix ecto.migrate

# Rollback migration
mix ecto.rollback

# เปิด DB console
mix ecto.dump

# Generate context + schema + migration
mix phx.gen.context Blog Post posts title:string body:text

# Generate full HTML CRUD
mix phx.gen.html Blog Post posts title:string body:text

# Generate JSON API
mix phx.gen.json Blog Post posts title:string body:text

# Generate LiveView
mix phx.gen.live Blog Post posts title:string body:text

# Generate Schema only
mix phx.gen.schema Blog.Post posts title:string body:text

# รัน server
mix phx.server

# สร้าง secret key
mix phx.gen.secret

# ดู routes
mix phx.routes

# Run tests
mix test

# Deploy release
mix phx.gen.release
mix release
```

---

## 9. ตัวอย่างจริง: สร้าง Blog App

### Step 1: สร้าง project

```bash
mix phx.new blog --database postgres
cd blog
mix ecto.create
```

### Step 2: สร้าง Post schema

```bash
mix phx.gen.html Blog Post posts \
  title:string \
  body:text \
  author:string \
  published_at:utc_datetime \
  is_published:boolean \
  view_count:integer \
  slug:string:unique
```

Output จาก generator:
```
* creating lib/blog/blog.ex
* creating lib/blog_web/controllers/post_controller.ex
* creating lib/blog_web/controllers/post_html.ex
* creating lib/blog_web/controllers/post_html/edit.html.heex
* creating lib/blog_web/controllers/post_html/index.html.heex
* creating lib/blog_web/controllers/post_html/new.html.heex
* creating lib/blog_web/controllers/post_html/show.html.heex
* creating test/blog_web/controllers/post_controller_test.exs
* creating lib/blog/blog/post.ex
* creating priv/repo/migrations/20240101000000_create_posts.exs
```

### Step 3: เพิ่ม routes

```elixir
# lib/blog_web/router.ex
scope "/", BlogWeb do
  pipe_through :browser

  get "/", PageController, :home
  resources "/posts", PostController
end
```

### Step 4: รัน migration

```bash
mix ecto.migrate
```

### Step 5: ทดสอบ

```bash
mix phx.server
# เปิด http://localhost:4000/posts
```

---

## 10. Custom Mix Task

```elixir
# lib/mix/tasks/blog.seed.ex
defmodule Mix.Tasks.Blog.Seed do
  use Mix.Task

  @shortdoc "Seed the database with sample data"

  @moduledoc """
  Seeds the database with sample blog posts.

  ## Usage

      mix blog.seed
      mix blog.seed --count 20
  """

  def run(args) do
    # Parse options
    {opts, _, _} = OptionParser.parse(args, strict: [count: :integer])
    count = Keyword.get(opts, :count, 10)

    # Start the app
    Mix.Task.run("app.start")

    # Seed data
    IO.puts("Seeding #{count} posts...")

    for i <- 1..count do
      {:ok, post} = Blog.create_post(%{
        title: "Post #{i}: #{Faker.Lorem.sentence()}",
        body: Faker.Lorem.paragraphs(3) |> Enum.join("\n\n"),
        author: Faker.Name.name(),
        slug: "post-#{i}-#{System.unique_integer([:positive])}",
        is_published: Enum.random([true, false]),
        published_at: DateTime.utc_now(),
        view_count: Enum.random(0..1000)
      })

      IO.puts("Created: #{post.title}")
    end

    IO.puts("Done!")
  end
end
```

---

## 11. Exercises

### Exercise 1: เพิ่ม Category ให้ Blog

สร้าง Category context และ associate กับ Post

```bash
# Generate category
mix phx.gen.context Blog Category categories name:string:unique slug:string:unique

# สร้าง migration เพิ่ม category_id ใน posts
mix ecto.gen.migration add_category_to_posts
```

**เฉลย (migration):**

```elixir
# priv/repo/migrations/TIMESTAMP_add_category_to_posts.exs
defmodule Blog.Repo.Migrations.AddCategoryToPosts do
  use Ecto.Migration

  def change do
    alter table(:posts) do
      add :category_id, references(:categories, on_delete: :nilify_all)
    end

    create index(:posts, [:category_id])
  end
end
```

### Exercise 2: เพิ่ม Comment System

สร้าง Comments ที่ belong to Post

**เฉลย:**

```bash
mix phx.gen.context Blog Comment comments \
  body:text \
  author_name:string \
  author_email:string \
  post_id:references:posts
```

```elixir
# lib/blog/blog/comment.ex
defmodule Blog.Comment do
  use Ecto.Schema
  import Ecto.Changeset

  schema "comments" do
    field :body, :string
    field :author_name, :string
    field :author_email, :string

    belongs_to :post, Blog.Post

    timestamps(type: :utc_datetime)
  end

  def changeset(comment, attrs) do
    comment
    |> cast(attrs, [:body, :author_name, :author_email, :post_id])
    |> validate_required([:body, :author_name, :post_id])
    |> validate_length(:body, min: 10, max: 2000)
    |> validate_format(:author_email, ~r/@/)
    |> foreign_key_constraint(:post_id)
  end
end
```

---

## สรุป

```
Phoenix Framework:
├── ติดตั้ง: mix archive.install hex phx_new
├── สร้าง project: mix phx.new APP_NAME
├── โครงสร้าง:
│   ├── lib/APP/ - Business logic (contexts)
│   ├── lib/APP_web/ - Web layer
│   └── config/ - Configuration
├── Application Supervisor:
│   ├── Repo (DB pool)
│   ├── PubSub
│   └── Endpoint (web server)
└── Configuration:
    ├── config.exs - Base
    ├── dev.exs - Development
    ├── prod.exs - Production
    └── runtime.exs - Environment variables

สำคัญ:
├── Contexts คือ boundaries ของ domain
├── Web layer แยกจาก business logic
└── Convention over configuration
```

---

*ก่อนหน้า: [Part 24 - Distributed Elixir](part_24.md) | ต่อไป: [Part 26 - Phoenix Routing](part_26.md)*
