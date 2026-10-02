# Part 26: Phoenix Routing

## เป้าหมายการเรียนรู้

หลังจากจบ Part นี้ คุณจะสามารถ:
- เข้าใจ router.ex และกำหนด routes ได้
- ใช้ `resources`, `scope`, `pipeline` ได้
- ใช้ path helpers ได้
- กำหนด wildcard routes ได้
- สร้าง API routes สำหรับ Blog app

---

## 1. Router Overview

```elixir
# lib/blog_web/router.ex
defmodule BlogWeb.Router do
  use BlogWeb, :router

  # Pipelines กำหนด middleware
  pipeline :browser do
    plug :accepts, ["html"]
    plug :fetch_session
    plug :fetch_live_flash
    plug :put_root_layout, html: {BlogWeb.Layouts, :root}
    plug :protect_from_forgery
    plug :put_secure_browser_headers
  end

  pipeline :api do
    plug :accepts, ["json"]
  end

  # Browser routes
  scope "/", BlogWeb do
    pipe_through :browser

    get "/", PageController, :home
    get "/about", PageController, :about
    resources "/posts", PostController
  end

  # API routes
  scope "/api", BlogWeb do
    pipe_through :api

    resources "/posts", PostController, only: [:index, :show]
  end
end
```

---

## 2. HTTP Methods และ Routes

### Basic Route Verbs

```elixir
scope "/", BlogWeb do
  pipe_through :browser

  # HTTP verb เดียว
  get "/",           PageController, :index
  post "/contact",   ContactController, :create
  put "/posts/:id",  PostController, :update
  patch "/posts/:id", PostController, :update
  delete "/posts/:id", PostController, :delete

  # ดู current routes
  # mix phx.routes
end
```

### resources/2 - RESTful routes อัตโนมัติ

```elixir
resources "/posts", PostController
# เทียบเท่ากับ:
# GET    /posts           PostController :index
# GET    /posts/new       PostController :new
# POST   /posts           PostController :create
# GET    /posts/:id       PostController :show
# GET    /posts/:id/edit  PostController :edit
# PUT    /posts/:id       PostController :update
# PATCH  /posts/:id       PostController :update
# DELETE /posts/:id       PostController :delete
```

### resources options

```elixir
# only: เลือกเฉพาะ actions ที่ต้องการ
resources "/posts", PostController, only: [:index, :show]

# except: ยกเว้น actions บางอัน
resources "/posts", PostController, except: [:delete]

# สร้าง routes บางส่วน
resources "/users", UserController, only: [:index, :create, :delete]

# ดู routes ทั้งหมด
# mix phx.routes BlogWeb.Router
```

---

## 3. Nested Resources

```elixir
scope "/", BlogWeb do
  pipe_through :browser

  resources "/posts", PostController do
    # /posts/:post_id/comments/...
    resources "/comments", CommentController, only: [:index, :create, :delete]
  end
end

# Routes ที่ได้:
# GET    /posts/:post_id/comments
# POST   /posts/:post_id/comments
# DELETE /posts/:post_id/comments/:id
```

```elixir
# Controller สำหรับ nested route
defmodule BlogWeb.CommentController do
  use BlogWeb, :controller

  def index(conn, %{"post_id" => post_id}) do
    comments = Blog.list_comments_for_post(post_id)
    render(conn, :index, comments: comments)
  end

  def create(conn, %{"post_id" => post_id, "comment" => comment_params}) do
    case Blog.create_comment(post_id, comment_params) do
      {:ok, comment} ->
        conn
        |> put_flash(:info, "Comment created")
        |> redirect(to: ~p"/posts/#{post_id}")
      {:error, changeset} ->
        conn
        |> put_flash(:error, "Error creating comment")
        |> redirect(to: ~p"/posts/#{post_id}")
    end
  end
end
```

---

## 4. Pipelines

Pipeline คือ chain ของ Plugs ที่ทำงานก่อน request ถึง controller

```elixir
defmodule BlogWeb.Router do
  use BlogWeb, :router

  # Browser pipeline
  pipeline :browser do
    plug :accepts, ["html"]
    plug :fetch_session
    plug :fetch_live_flash
    plug :put_root_layout, html: {BlogWeb.Layouts, :root}
    plug :protect_from_forgery
    plug :put_secure_browser_headers
  end

  # API pipeline
  pipeline :api do
    plug :accepts, ["json"]
  end

  # Authentication pipeline
  pipeline :authenticated do
    plug BlogWeb.Plugs.RequireAuth
  end

  # Admin pipeline
  pipeline :admin do
    plug BlogWeb.Plugs.RequireAuth
    plug BlogWeb.Plugs.RequireAdmin
  end

  # Public routes
  scope "/", BlogWeb do
    pipe_through :browser

    get "/", PageController, :home
    get "/posts", PostController, :index
    get "/posts/:id", PostController, :show
  end

  # Routes ที่ต้อง login
  scope "/", BlogWeb do
    pipe_through [:browser, :authenticated]

    get "/posts/new", PostController, :new
    post "/posts", PostController, :create
    get "/posts/:id/edit", PostController, :edit
    put "/posts/:id", PostController, :update
    delete "/posts/:id", PostController, :delete
  end

  # Admin routes
  scope "/admin", BlogWeb.Admin do
    pipe_through [:browser, :admin]

    get "/", DashboardController, :index
    resources "/users", UserController
    resources "/posts", PostController
  end
end
```

### สร้าง Custom Pipeline Plug

```elixir
# lib/blog_web/plugs/require_auth.ex
defmodule BlogWeb.Plugs.RequireAuth do
  import Plug.Conn
  import Phoenix.Controller

  def init(opts), do: opts

  def call(conn, _opts) do
    case get_session(conn, :user_id) do
      nil ->
        conn
        |> put_flash(:error, "You must be logged in to access this page.")
        |> redirect(to: "/login")
        |> halt()
      user_id ->
        user = Blog.Accounts.get_user!(user_id)
        assign(conn, :current_user, user)
    end
  end
end

# lib/blog_web/plugs/require_admin.ex
defmodule BlogWeb.Plugs.RequireAdmin do
  import Plug.Conn
  import Phoenix.Controller

  def init(opts), do: opts

  def call(%{assigns: %{current_user: user}} = conn, _opts) do
    if user.role == :admin do
      conn
    else
      conn
      |> put_flash(:error, "You don't have permission to access this page.")
      |> redirect(to: "/")
      |> halt()
    end
  end
end
```

---

## 5. Scopes

Scopes ใช้สำหรับ group routes เข้าด้วยกัน

```elixir
defmodule BlogWeb.Router do
  use BlogWeb, :router

  # Web routes
  scope "/", BlogWeb do
    pipe_through :browser

    get "/", PageController, :home
    resources "/posts", PostController
  end

  # API v1
  scope "/api/v1", BlogWeb.API.V1 do
    pipe_through :api

    resources "/posts", PostController, except: [:new, :edit]
    resources "/users", UserController, only: [:index, :show, :create]
  end

  # API v2
  scope "/api/v2", BlogWeb.API.V2 do
    pipe_through :api

    resources "/posts", PostController
    resources "/users", UserController
  end

  # Admin
  scope "/admin", BlogWeb.Admin, as: :admin do
    pipe_through [:browser, :authenticate_admin]

    resources "/users", UserController
    resources "/posts", PostController
    get "/dashboard", DashboardController, :index
  end
end
```

---

## 6. Path Helpers

Phoenix สร้าง path helpers อัตโนมัติจาก routes

```elixir
# ใช้ ~p sigil (Phoenix 1.7+)
~p"/posts"                # => "/posts"
~p"/posts/#{id}"          # => "/posts/123"
~p"/posts/#{post.id}/edit" # => "/posts/123/edit"
~p"/api/v1/posts"         # => "/api/v1/posts"

# ใน controller
defmodule BlogWeb.PostController do
  use BlogWeb, :controller

  def create(conn, %{"post" => post_params}) do
    case Blog.create_post(post_params) do
      {:ok, post} ->
        conn
        |> put_flash(:info, "Post created!")
        |> redirect(to: ~p"/posts/#{post}")
      {:error, changeset} ->
        render(conn, :new, changeset: changeset)
    end
  end
end

# ใน template (.heex)
<.link href={~p"/posts/#{@post}"}>View Post</.link>
<.link href={~p"/posts/new"}>New Post</.link>
<.link href={~p"/posts/#{@post}/edit"}>Edit</.link>
```

### URL Helpers

```elixir
# สร้าง full URL (ต้องการ conn หรือ endpoint)
BlogWeb.Endpoint.url()
# => "http://localhost:4000"

url(~p"/posts/#{post}")
# => "http://localhost:4000/posts/123"

# ใน controller
conn
|> put_flash(:info, "Check #{url(conn, ~p"/posts/#{post}")}")
```

---

## 7. Query Parameters และ Path Parameters

```elixir
# Route: GET /posts/:id
# URL: /posts/123?format=json&page=2

defmodule BlogWeb.PostController do
  use BlogWeb, :controller

  def show(conn, %{"id" => id} = params) do
    format = Map.get(params, "format", "html")
    page = Map.get(params, "page", "1") |> String.to_integer()

    post = Blog.get_post!(id)
    render(conn, :show, post: post)
  end

  def index(conn, params) do
    page = Map.get(params, "page", "1") |> String.to_integer()
    per_page = Map.get(params, "per_page", "10") |> String.to_integer()
    search = Map.get(params, "q", "")
    category = Map.get(params, "category", nil)

    posts = Blog.list_posts(%{
      page: page,
      per_page: per_page,
      search: search,
      category: category
    })

    render(conn, :index, posts: posts, page: page)
  end
end
```

---

## 8. Wildcard Routes

```elixir
scope "/", BlogWeb do
  pipe_through :browser

  # Catch-all route
  get "/*path", ErrorController, :not_found
end

# หรือ
get "/docs/*path", DocsController, :show
# URL: /docs/getting-started/installation
# params: %{"path" => ["getting-started", "installation"]}
```

```elixir
defmodule BlogWeb.DocsController do
  use BlogWeb, :controller

  def show(conn, %{"path" => path_parts}) do
    doc_path = Enum.join(path_parts, "/")

    case Blog.Docs.find(doc_path) do
      {:ok, doc} ->
        render(conn, :show, doc: doc)
      {:error, :not_found} ->
        conn
        |> put_status(:not_found)
        |> render(:not_found)
    end
  end
end
```

---

## 9. Forward Routes

```elixir
# Forward requests ไปยัง Plug หรือ Router อื่น
defmodule BlogWeb.Router do
  use BlogWeb, :router

  # Forward ไปยัง Phoenix.LiveDashboard
  import Phoenix.LiveDashboard.Router

  scope "/dev" do
    pipe_through :browser
    live_dashboard "/dashboard", metrics: BlogWeb.Telemetry
  end

  # Forward ไปยัง custom router
  scope "/webhook" do
    forward "/github", BlogWeb.WebhookRouter
    forward "/stripe", BlogWeb.StripeWebhookPlug
  end
end

defmodule BlogWeb.WebhookRouter do
  use Plug.Router

  plug :match
  plug :dispatch

  post "/push" do
    # handle github webhook
    send_resp(conn, 200, "OK")
  end

  match _ do
    send_resp(conn, 404, "Not Found")
  end
end
```

---

## 10. ตัวอย่างจริง: API Routes สำหรับ Blog

```elixir
# lib/blog_web/router.ex
defmodule BlogWeb.Router do
  use BlogWeb, :router

  # ============= Pipelines =============

  pipeline :browser do
    plug :accepts, ["html"]
    plug :fetch_session
    plug :fetch_live_flash
    plug :put_root_layout, html: {BlogWeb.Layouts, :root}
    plug :protect_from_forgery
    plug :put_secure_browser_headers
    plug BlogWeb.Plugs.FetchCurrentUser  # ดึง current user จาก session
  end

  pipeline :api do
    plug :accepts, ["json"]
    plug BlogWeb.Plugs.AuthToken         # Bearer token authentication
  end

  pipeline :require_auth do
    plug BlogWeb.Plugs.RequireAuth
  end

  pipeline :require_admin do
    plug BlogWeb.Plugs.RequireAdmin
  end

  # ============= Browser Routes =============

  # Public
  scope "/", BlogWeb do
    pipe_through :browser

    get "/",           PageController, :home
    get "/about",      PageController, :about
    get "/search",     SearchController, :index

    # Auth
    get  "/login",     SessionController, :new
    post "/login",     SessionController, :create
    delete "/logout",  SessionController, :delete
    get  "/register",  UserController, :new
    post "/register",  UserController, :create

    # Blog
    get "/posts",         PostController, :index
    get "/posts/:id",     PostController, :show
    get "/tags/:name",    TagController, :show
    get "/categories/:slug", CategoryController, :show

    # Feed
    get "/feed.xml",  FeedController, :index
    get "/sitemap.xml", SitemapController, :index
  end

  # Authenticated user routes
  scope "/", BlogWeb do
    pipe_through [:browser, :require_auth]

    get  "/profile",        UserController, :show
    get  "/profile/edit",   UserController, :edit
    put  "/profile",        UserController, :update
    delete "/profile",      UserController, :delete

    resources "/posts", PostController, except: [:index, :show] do
      resources "/comments", CommentController, only: [:create]
    end

    resources "/comments", CommentController, only: [:delete]
  end

  # Admin routes
  scope "/admin", BlogWeb.Admin do
    pipe_through [:browser, :require_auth, :require_admin]

    get "/",              DashboardController, :index
    resources "/users",   UserController
    resources "/posts",   PostController
    resources "/tags",    TagController
    resources "/categories", CategoryController
  end

  # ============= API Routes =============

  # Public API
  scope "/api/v1", BlogWeb.API.V1 do
    pipe_through :api

    get "/health",      HealthController, :index

    resources "/posts",   PostController, only: [:index, :show]
    resources "/tags",    TagController, only: [:index, :show]
    resources "/categories", CategoryController, only: [:index, :show]

    # Search
    get "/search",      SearchController, :index

    # Auth
    post "/register",   UserController, :create
    post "/login",      SessionController, :create
  end

  # Authenticated API
  scope "/api/v1", BlogWeb.API.V1 do
    pipe_through [:api, :require_auth]

    get "/me",              UserController, :me
    put "/me",              UserController, :update

    resources "/posts",     PostController, except: [:index, :show, :new, :edit]
    resources "/comments",  CommentController, except: [:new, :edit]

    post "/logout",         SessionController, :delete
  end

  # Admin API
  scope "/api/v1/admin", BlogWeb.API.V1.Admin do
    pipe_through [:api, :require_auth, :require_admin]

    resources "/users",     UserController
    resources "/analytics", AnalyticsController, only: [:index]
  end

  # ============= Webhook Routes =============

  scope "/webhooks" do
    post "/stripe",   BlogWeb.WebhookController, :stripe
    post "/sendgrid", BlogWeb.WebhookController, :sendgrid
  end

  # ============= Dev Routes =============

  if Application.compile_env(:blog, :dev_routes) do
    import Phoenix.LiveDashboard.Router

    scope "/dev" do
      pipe_through :browser
      live_dashboard "/dashboard", metrics: BlogWeb.Telemetry
      forward "/mailbox", Plug.Swoosh.MailboxPreview
    end
  end
end
```

---

## 11. ดู Routes

```bash
# ดู routes ทั้งหมด
mix phx.routes

# Output:
#         page_path  GET   /               BlogWeb.PageController :home
#         post_path  GET   /posts          BlogWeb.PostController :index
#         post_path  GET   /posts/new      BlogWeb.PostController :new
#         post_path  GET   /posts/:id      BlogWeb.PostController :show
#         post_path  POST  /posts          BlogWeb.PostController :create
#         post_path  GET   /posts/:id/edit BlogWeb.PostController :edit
#         post_path  PATCH /posts/:id      BlogWeb.PostController :update
#         post_path  PUT   /posts/:id      BlogWeb.PostController :update
#         post_path  DELETE /posts/:id     BlogWeb.PostController :delete

# ดูเฉพาะ routes ที่ match pattern
mix phx.routes --info /api
```

---

## 12. Exercises

### Exercise 1: สร้าง API Version Negotiation

สร้าง middleware ที่ support multiple API versions

**เฉลย:**

```elixir
defmodule BlogWeb.Plugs.APIVersion do
  import Plug.Conn

  def init(default_version), do: default_version

  def call(conn, default_version) do
    version = get_version(conn, default_version)
    assign(conn, :api_version, version)
  end

  defp get_version(conn, default) do
    # ดูจาก Accept header: application/vnd.blog.v2+json
    case get_req_header(conn, "accept") do
      [accept | _] ->
        case Regex.run(~r/vnd\.blog\.v(\d+)/, accept) do
          [_, version] -> String.to_integer(version)
          _ -> check_query_param(conn, default)
        end
      [] ->
        check_query_param(conn, default)
    end
  end

  defp check_query_param(conn, default) do
    conn.query_params["version"] |> parse_version(default)
  end

  defp parse_version(nil, default), do: default
  defp parse_version(v, _default) do
    case Integer.parse(v) do
      {n, ""} -> n
      _ -> 1
    end
  end
end

# ใน router
pipeline :api do
  plug :accepts, ["json"]
  plug BlogWeb.Plugs.APIVersion, 1
end
```

### Exercise 2: Rate Limiting Plug

สร้าง rate limiting สำหรับ API

**เฉลย:**

```elixir
defmodule BlogWeb.Plugs.RateLimit do
  import Plug.Conn

  @default_limit 100       # requests
  @default_window 60_000   # 1 minute in ms

  def init(opts) do
    %{
      limit: Keyword.get(opts, :limit, @default_limit),
      window: Keyword.get(opts, :window, @default_window)
    }
  end

  def call(conn, %{limit: limit, window: window}) do
    client_ip = get_client_ip(conn)
    key = "rate_limit:#{client_ip}"

    case check_rate(key, limit, window) do
      {:ok, count} ->
        conn
        |> put_resp_header("x-ratelimit-limit", to_string(limit))
        |> put_resp_header("x-ratelimit-remaining", to_string(limit - count))

      {:error, :too_many_requests} ->
        conn
        |> put_status(:too_many_requests)
        |> Phoenix.Controller.json(%{error: "Rate limit exceeded"})
        |> halt()
    end
  end

  defp get_client_ip(conn) do
    case get_req_header(conn, "x-forwarded-for") do
      [ip | _] -> ip
      [] ->
        {a, b, c, d} = conn.remote_ip
        "#{a}.#{b}.#{c}.#{d}"
    end
  end

  defp check_rate(key, limit, window) do
    # ใช้ ETS หรือ Redis สำหรับ production
    # ตัวอย่างนี้ใช้ Agent อย่างง่าย
    count = RateLimiter.increment(key, window)
    if count > limit do
      {:error, :too_many_requests}
    else
      {:ok, count}
    end
  end
end
```

---

## สรุป

```
Phoenix Routing:
├── Routes พื้นฐาน
│   ├── get, post, put, patch, delete
│   ├── resources/2 - RESTful routes ทั้งหมด
│   └── resources options: only:, except:
├── Nested Resources
│   └── resources "/posts", PostController do
│       resources "/comments", CommentController
├── Pipelines
│   ├── browser - HTML requests
│   ├── api - JSON requests
│   └── custom plugs
├── Scopes
│   ├── group routes ด้วย path prefix
│   ├── specify module namespace
│   └── as: :admin สำหรับ route name prefix
├── Path Helpers
│   ├── ~p"/posts/#{id}" - sigil (recommended)
│   └── type-checked at compile time
└── Wildcard: get "/*path", ...

Best Practices:
├── แยก browser และ api routes ชัดเจน
├── ใช้ pipeline สำหรับ common middleware
├── ใช้ scope เพื่อ organize routes
└── ใส่ authentication ใน pipeline
```

---

*ก่อนหน้า: [Part 25 - Phoenix Framework](part_25.md) | ต่อไป: [Part 27 - Phoenix Controllers](part_27.md)*
