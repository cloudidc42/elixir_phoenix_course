# Part 34: API Development และ JSON

## เป้าหมายการเรียนรู้

หลังจากจบ Part นี้ คุณจะสามารถ:
- สร้าง REST API ด้วย Phoenix
- ใช้ Jason สำหรับ JSON
- Implement API versioning
- สร้าง API documentation

---

## 1. JSON Controller

```elixir
# lib/my_app_web/controllers/api/v1/user_controller.ex
defmodule MyAppWeb.Api.V1.UserController do
  use MyAppWeb, :controller

  alias MyApp.Accounts
  alias MyApp.Accounts.User

  action_fallback MyAppWeb.FallbackController

  def index(conn, params) do
    users = Accounts.list_users(params)
    render(conn, :index, users: users)
  end

  def show(conn, %{"id" => id}) do
    with {:ok, user} <- Accounts.get_user(id) do
      render(conn, :show, user: user)
    end
  end

  def create(conn, %{"user" => user_params}) do
    with {:ok, %User{} = user} <- Accounts.register_user(user_params) do
      conn
      |> put_status(:created)
      |> put_resp_header("location", ~p"/api/v1/users/#{user}")
      |> render(:show, user: user)
    end
  end

  def update(conn, %{"id" => id, "user" => user_params}) do
    with {:ok, user} <- Accounts.get_user(id),
         {:ok, %User{} = updated} <- Accounts.update_user(user, user_params) do
      render(conn, :show, user: updated)
    end
  end

  def delete(conn, %{"id" => id}) do
    with {:ok, user} <- Accounts.get_user(id),
         {:ok, %User{}} <- Accounts.delete_user(user) do
      send_resp(conn, :no_content, "")
    end
  end
end
```

---

## 2. JSON Views (Phoenix 1.7+)

```elixir
# lib/my_app_web/controllers/api/v1/user_json.ex
defmodule MyAppWeb.Api.V1.UserJSON do
  alias MyApp.Accounts.User

  def index(%{users: users}) do
    %{data: for(user <- users, do: data(user))}
  end

  def show(%{user: user}) do
    %{data: data(user)}
  end

  defp data(%User{} = user) do
    %{
      id: user.id,
      name: user.name,
      email: user.email,
      role: user.role,
      active: user.active,
      inserted_at: user.inserted_at,
      updated_at: user.updated_at
    }
  end

  # Paginated response
  def index(%{users: users, page: page, total: total}) do
    %{
      data: for(user <- users, do: data(user)),
      meta: %{
        page: page.page_number,
        page_size: page.page_size,
        total_entries: total,
        total_pages: page.total_pages
      }
    }
  end
end
```

---

## 3. FallbackController

```elixir
defmodule MyAppWeb.FallbackController do
  use MyAppWeb, :controller

  def call(conn, {:error, :not_found}) do
    conn
    |> put_status(:not_found)
    |> put_view(json: MyAppWeb.ErrorJSON)
    |> render(:"404")
  end

  def call(conn, {:error, :unauthorized}) do
    conn
    |> put_status(:unauthorized)
    |> put_view(json: MyAppWeb.ErrorJSON)
    |> render(:"401")
  end

  def call(conn, {:error, :forbidden}) do
    conn
    |> put_status(:forbidden)
    |> put_view(json: MyAppWeb.ErrorJSON)
    |> render(:"403")
  end

  def call(conn, {:error, %Ecto.Changeset{} = changeset}) do
    conn
    |> put_status(:unprocessable_entity)
    |> put_view(json: MyAppWeb.ChangesetJSON)
    |> render(:error, changeset: changeset)
  end

  def call(conn, {:error, {:rate_limited, retry_after}}) do
    conn
    |> put_resp_header("retry-after", to_string(div(retry_after, 1000)))
    |> put_status(429)
    |> put_view(json: MyAppWeb.ErrorJSON)
    |> render(:"429")
  end
end

# lib/my_app_web/controllers/changeset_json.ex
defmodule MyAppWeb.ChangesetJSON do
  def error(%{changeset: changeset}) do
    %{errors: translate_errors(changeset)}
  end

  defp translate_errors(changeset) do
    Ecto.Changeset.traverse_errors(changeset, fn {msg, opts} ->
      Regex.replace(~r"%{(\w+)}", msg, fn _, key ->
        opts |> Keyword.get(String.to_existing_atom(key), key) |> to_string()
      end)
    end)
  end
end
```

---

## 4. API Routing

```elixir
# lib/my_app_web/router.ex
defmodule MyAppWeb.Router do
  use MyAppWeb, :router

  pipeline :api do
    plug :accepts, ["json"]
    plug :fetch_session
    plug MyAppWeb.Plugs.ApiAuth
  end

  pipeline :api_public do
    plug :accepts, ["json"]
  end

  scope "/api", MyAppWeb do
    pipe_through :api_public

    scope "/v1", Api.V1 do
      post "/auth/login", AuthController, :login
      post "/auth/refresh", AuthController, :refresh
      post "/users", UserController, :create
    end
  end

  scope "/api", MyAppWeb do
    pipe_through :api

    scope "/v1", Api.V1 do
      resources "/users", UserController, except: [:new, :edit]
      resources "/posts", PostController, except: [:new, :edit] do
        resources "/comments", CommentController, except: [:new, :edit]
      end
    end
  end
end
```

---

## 5. Pagination

```elixir
# mix.exs
{:scrivener_ecto, "~> 2.7"}

# lib/my_app/accounts.ex
defmodule MyApp.Accounts do
  alias MyApp.Repo
  alias MyApp.Accounts.User
  import Ecto.Query

  def list_users(params \\ %{}) do
    page = Map.get(params, "page", "1") |> String.to_integer()
    page_size = Map.get(params, "page_size", "20") |> String.to_integer() |> min(100)

    User
    |> apply_filters(params)
    |> order_by([u], asc: u.inserted_at)
    |> Repo.paginate(page: page, page_size: page_size)
  end

  defp apply_filters(query, params) do
    Enum.reduce(params, query, fn
      {"search", q}, query when byte_size(q) > 0 ->
        where(query, [u], ilike(u.name, ^"%#{q}%") or ilike(u.email, ^"%#{q}%"))

      {"role", role}, query when role != "" ->
        where(query, [u], u.role == ^role)

      _, query ->
        query
    end)
  end
end

# JSON view ที่มี pagination
def index(%{page: page}) do
  %{
    data: Enum.map(page.entries, &data/1),
    meta: %{
      page: page.page_number,
      page_size: page.page_size,
      total_entries: page.total_entries,
      total_pages: page.total_pages
    },
    links: %{
      self: "/api/v1/users?page=#{page.page_number}",
      prev: if(page.page_number > 1, do: "/api/v1/users?page=#{page.page_number - 1}"),
      next: if(page.page_number < page.total_pages, do: "/api/v1/users?page=#{page.page_number + 1}")
    }
  }
end
```

---

## 6. API Versioning

```elixir
# Strategy 1: URL versioning (แนะนำ)
scope "/api/v1", MyAppWeb.Api.V1 do
  resources "/users", UserController
end

scope "/api/v2", MyAppWeb.Api.V2 do
  resources "/users", UserController
end

# Strategy 2: Header versioning
defmodule MyAppWeb.Plugs.ApiVersion do
  import Plug.Conn

  def init(opts), do: opts

  def call(conn, _opts) do
    version = get_req_header(conn, "x-api-version")
    |> List.first()
    |> parse_version()

    assign(conn, :api_version, version)
  end

  defp parse_version("v2"), do: :v2
  defp parse_version(_), do: :v1
end

# Controller ที่รองรับหลาย versions
defmodule MyAppWeb.Api.UserController do
  use MyAppWeb, :controller

  def show(conn, %{"id" => id}) do
    with {:ok, user} <- MyApp.Accounts.get_user(id) do
      case conn.assigns.api_version do
        :v2 -> render(conn, :show_v2, user: user)
        _ -> render(conn, :show, user: user)
      end
    end
  end
end
```

---

## 7. Rate Limiting สำหรับ API

```elixir
defmodule MyAppWeb.Plugs.RateLimit do
  import Plug.Conn
  import Phoenix.Controller

  alias MyApp.RateLimiter

  def init(opts), do: opts

  def call(conn, _opts) do
    key = rate_limit_key(conn)

    case RateLimiter.check(key) do
      {:ok, remaining} ->
        conn
        |> put_resp_header("x-ratelimit-limit", "100")
        |> put_resp_header("x-ratelimit-remaining", to_string(remaining))

      {:error, {:rate_limited, retry_after}} ->
        conn
        |> put_resp_header("retry-after", to_string(div(retry_after, 1000)))
        |> put_status(429)
        |> json(%{error: "Too many requests"})
        |> halt()
    end
  end

  defp rate_limit_key(conn) do
    case conn.assigns[:current_user] do
      nil -> "ip:#{conn.remote_ip |> :inet.ntoa() |> to_string()}"
      user -> "user:#{user.id}"
    end
  end
end
```

---

## 8. API Documentation

```elixir
# mix.exs
{:open_api_spex, "~> 3.18"}

# lib/my_app_web/api_spec.ex
defmodule MyAppWeb.ApiSpec do
  alias OpenApiSpex.{Info, OpenApi, Server}

  @behaviour OpenApi

  @impl OpenApi
  def spec do
    %OpenApi{
      servers: [%Server{url: "https://api.myapp.com"}],
      info: %Info{
        title: "My App API",
        version: "1.0",
        description: "REST API for My App"
      },
      paths: OpenApiSpex.Paths.from_router(MyAppWeb.Router)
    }
    |> OpenApiSpex.resolve_schema_modules()
  end
end
```

---

## แบบฝึกหัด

### Exercise 1: Blog API
สร้าง REST API สำหรับ Blog:
- GET /api/v1/posts - list posts (paginated)
- GET /api/v1/posts/:id - get post
- POST /api/v1/posts - create post (auth required)
- PUT /api/v1/posts/:id - update (owner only)
- DELETE /api/v1/posts/:id - delete (owner only)

### Exercise 2: Search API
สร้าง search endpoint:
- GET /api/v1/search?q=keyword&type=user|post
- Return mixed results

---

## สรุป

```
REST API ใน Phoenix:
├── Controller: action_fallback
├── View (JSON): render functions
├── FallbackController: error handling
└── Router: versioned scopes

Best Practices:
├── Consistent response format
├── Proper HTTP status codes
├── Pagination metadata
├── Rate limiting headers
└── API versioning

Tools:
├── Jason สำหรับ JSON
├── Scrivener สำหรับ pagination
└── OpenApiSpex สำหรับ documentation
```

---

*ก่อนหน้า: [Part 33](part_33.md) | ต่อไป: [Part 35 - File Uploads](part_35.md)*
