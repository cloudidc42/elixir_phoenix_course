# Part 27: Phoenix Controllers

## เป้าหมายการเรียนรู้

หลังจากจบ Part นี้ คุณจะสามารถ:
- เข้าใจ Controller actions และ conn struct
- ใช้ Plug pipeline ใน controller
- Render responses ในรูปแบบต่างๆ
- Handle redirects, flash messages
- สร้าง CRUD controller สำหรับ Blog

---

## 1. Controller พื้นฐาน

Controller รับ HTTP request และส่ง response กลับไป

```elixir
defmodule BlogWeb.PageController do
  use BlogWeb, :controller

  # action รับ conn และ params
  def home(conn, _params) do
    # render view
    render(conn, :home)
  end

  def about(conn, _params) do
    render(conn, :about, company: "My Blog Inc.")
  end
end
```

### conn struct

`conn` คือ struct ที่เก็บข้อมูลทั้งหมดของ HTTP request/response

```elixir
%Plug.Conn{
  # Request
  method: "GET",
  path_info: ["posts", "123"],
  query_string: "page=1&q=elixir",
  query_params: %{"page" => "1", "q" => "elixir"},
  params: %{"id" => "123", "page" => "1"},
  req_headers: [{"accept", "text/html"}, ...],
  body_params: %{},

  # Response
  status: nil,           # HTTP status code
  resp_headers: [],
  resp_body: "",

  # State
  assigns: %{},          # custom data สำหรับ request lifecycle
  halted: false,

  # Misc
  host: "localhost",
  port: 4000,
  scheme: :http,
  remote_ip: {127, 0, 0, 1}
}
```

---

## 2. Rendering Responses

### render/2 และ render/3

```elixir
defmodule BlogWeb.PostController do
  use BlogWeb, :controller

  def index(conn, _params) do
    posts = Blog.list_posts()

    # render :index action ใน PostHTML view
    render(conn, :index, posts: posts)

    # or render explicit template
    # render(conn, "index.html", posts: posts)
  end

  def show(conn, %{"id" => id}) do
    post = Blog.get_post!(id)
    render(conn, :show, post: post)
  end
end
```

### JSON Response

```elixir
defmodule BlogWeb.API.V1.PostController do
  use BlogWeb, :controller

  def index(conn, _params) do
    posts = Blog.list_posts()

    # json/2 - render JSON
    json(conn, %{
      data: Enum.map(posts, &post_json/1),
      meta: %{total: length(posts)}
    })
  end

  def show(conn, %{"id" => id}) do
    case Blog.get_post(id) do
      nil ->
        conn
        |> put_status(:not_found)
        |> json(%{error: "Post not found"})

      post ->
        json(conn, %{data: post_json(post)})
    end
  end

  defp post_json(post) do
    %{
      id: post.id,
      title: post.title,
      body: post.body,
      author: post.author,
      published_at: post.published_at,
      url: ~p"/api/v1/posts/#{post.id}"
    }
  end
end
```

### Text Response

```elixir
def robots(conn, _params) do
  text(conn, """
  User-agent: *
  Disallow: /admin
  Sitemap: https://myblog.com/sitemap.xml
  """)
end

# Plain text
def health(conn, _params) do
  conn
  |> put_resp_content_type("text/plain")
  |> send_resp(200, "OK")
end
```

### Custom Status Codes

```elixir
def create(conn, params) do
  case Blog.create_post(params) do
    {:ok, post} ->
      conn
      |> put_status(:created)       # 201
      |> json(%{data: post_json(post)})

    {:error, changeset} ->
      conn
      |> put_status(:unprocessable_entity)  # 422
      |> json(%{errors: format_errors(changeset)})
  end
end

# หรือใช้ integer
conn |> put_status(200) |> json(%{ok: true})
conn |> put_status(404) |> json(%{error: "Not found"})
conn |> put_status(500) |> json(%{error: "Internal server error"})
```

---

## 3. Redirects

```elixir
defmodule BlogWeb.PostController do
  use BlogWeb, :controller

  def create(conn, %{"post" => post_params}) do
    case Blog.create_post(post_params) do
      {:ok, post} ->
        conn
        |> put_flash(:info, "Post created successfully!")
        |> redirect(to: ~p"/posts/#{post}")

      {:error, changeset} ->
        render(conn, :new, changeset: changeset)
    end
  end

  def delete(conn, %{"id" => id}) do
    post = Blog.get_post!(id)
    {:ok, _} = Blog.delete_post(post)

    conn
    |> put_flash(:info, "Post deleted.")
    |> redirect(to: ~p"/posts")
  end

  # Redirect ไปยัง external URL
  def external_redirect(conn, _params) do
    redirect(conn, external: "https://elixir-lang.org")
  end
end
```

### Flash Messages

```elixir
# ใส่ flash message
conn
|> put_flash(:info, "Operation successful!")
|> put_flash(:error, "Something went wrong!")
|> put_flash(:warning, "Please check your input")

# ใน template อ่านด้วย
# @flash["info"] หรือ
# Phoenix.Flash.get(@flash, :info)
```

---

## 4. Controller Plugs

```elixir
defmodule BlogWeb.PostController do
  use BlogWeb, :controller

  # Plug ทุก actions
  plug :authenticate

  # Plug เฉพาะ actions ที่ระบุ
  plug :load_post when action in [:show, :edit, :update, :delete]
  plug :authorize_post when action in [:edit, :update, :delete]

  def index(conn, _params) do
    posts = Blog.list_published_posts()
    render(conn, :index, posts: posts)
  end

  def show(conn, _params) do
    # @post ถูก load แล้วใน plug
    render(conn, :show)
  end

  def edit(conn, _params) do
    changeset = Blog.change_post(conn.assigns.post)
    render(conn, :edit, changeset: changeset)
  end

  def update(conn, %{"post" => post_params}) do
    case Blog.update_post(conn.assigns.post, post_params) do
      {:ok, post} ->
        conn
        |> put_flash(:info, "Post updated.")
        |> redirect(to: ~p"/posts/#{post}")

      {:error, changeset} ->
        render(conn, :edit, changeset: changeset)
    end
  end

  def delete(conn, _params) do
    Blog.delete_post(conn.assigns.post)

    conn
    |> put_flash(:info, "Post deleted.")
    |> redirect(to: ~p"/posts")
  end

  # ========= Private Plugs =========

  defp authenticate(conn, _opts) do
    case get_session(conn, :user_id) do
      nil ->
        conn
        |> put_flash(:error, "Please log in.")
        |> redirect(to: ~p"/login")
        |> halt()
      user_id ->
        user = Blog.Accounts.get_user!(user_id)
        assign(conn, :current_user, user)
    end
  end

  defp load_post(conn, _opts) do
    post = Blog.get_post!(conn.params["id"])
    assign(conn, :post, post)
  end

  defp authorize_post(conn, _opts) do
    %{current_user: user, post: post} = conn.assigns

    if post.author_id == user.id || user.role == :admin do
      conn
    else
      conn
      |> put_flash(:error, "You don't have permission.")
      |> redirect(to: ~p"/posts")
      |> halt()
    end
  end
end
```

---

## 5. params, body_params, query_params

```elixir
defmodule BlogWeb.SearchController do
  use BlogWeb, :controller

  def index(conn, params) do
    # params รวม query string + body + path params
    query = Map.get(params, "q", "")
    page = Map.get(params, "page", "1") |> String.to_integer()
    per_page = Map.get(params, "per_page", "10") |> String.to_integer()
    category = Map.get(params, "category")
    tags = Map.get(params, "tags", [])

    # Pattern matching บน params
    search_opts = %{
      query: query,
      page: page,
      per_page: per_page,
      category: category,
      tags: tags
    }

    results = Blog.search(search_opts)

    render(conn, :index,
      results: results,
      query: query,
      page: page,
      total: results.total
    )
  end
end

defmodule BlogWeb.API.PostController do
  use BlogWeb, :controller

  def create(conn, %{"post" => %{"title" => title, "body" => body} = post_params}) do
    # Pattern matching บน nested params
    case Blog.create_post(post_params) do
      {:ok, post} ->
        conn
        |> put_status(:created)
        |> put_resp_header("location", ~p"/api/v1/posts/#{post}")
        |> json(%{data: post})

      {:error, changeset} ->
        conn
        |> put_status(:unprocessable_entity)
        |> json(%{errors: format_errors(changeset)})
    end
  end

  defp format_errors(changeset) do
    Ecto.Changeset.traverse_errors(changeset, fn {msg, opts} ->
      Regex.replace(~r"%{(\w+)}", msg, fn _, key ->
        opts |> Keyword.get(String.to_existing_atom(key), key) |> to_string()
      end)
    end)
  end
end
```

---

## 6. Content Negotiation

```elixir
defmodule BlogWeb.PostController do
  use BlogWeb, :controller

  def show(conn, %{"id" => id}) do
    post = Blog.get_post!(id)

    conn
    |> put_status(200)
    |> render(:show, post: post)
  end
end

# ใน router pipeline เลือกตาม Accept header:
pipeline :browser do
  plug :accepts, ["html"]
end

pipeline :api do
  plug :accepts, ["json"]
end
```

---

## 7. File Downloads

```elixir
defmodule BlogWeb.ExportController do
  use BlogWeb, :controller

  def download_csv(conn, _params) do
    posts = Blog.list_posts()
    csv_content = generate_csv(posts)

    conn
    |> put_resp_content_type("text/csv")
    |> put_resp_header("content-disposition", ~s(attachment; filename="posts.csv"))
    |> send_resp(200, csv_content)
  end

  def download_pdf(conn, %{"id" => id}) do
    post = Blog.get_post!(id)
    pdf_path = Blog.generate_pdf(post)

    conn
    |> put_resp_content_type("application/pdf")
    |> put_resp_header("content-disposition", ~s(attachment; filename="post-#{id}.pdf"))
    |> send_file(200, pdf_path)
  end

  defp generate_csv(posts) do
    header = "ID,Title,Author,Published At\n"
    rows = Enum.map_join(posts, "\n", fn post ->
      "#{post.id},\"#{post.title}\",#{post.author},#{post.published_at}"
    end)
    header <> rows
  end
end
```

---

## 8. ตัวอย่างจริง: CRUD Controller สำหรับ Blog

```elixir
# lib/blog_web/controllers/post_controller.ex
defmodule BlogWeb.PostController do
  use BlogWeb, :controller

  alias Blog.Posts
  alias Blog.Posts.Post

  # Plugs
  plug :authenticate when action in [:new, :create, :edit, :update, :delete]
  plug :load_post when action in [:show, :edit, :update, :delete]
  plug :authorize_post when action in [:edit, :update, :delete]

  # ============= Actions =============

  @doc "List all published posts with pagination"
  def index(conn, params) do
    page = Map.get(params, "page", "1") |> String.to_integer()
    per_page = 10
    filter = build_filter(params)

    {posts, total} = Posts.list_posts(page: page, per_page: per_page, filter: filter)
    total_pages = ceil(total / per_page)

    render(conn, :index,
      posts: posts,
      page: page,
      total_pages: total_pages,
      total: total,
      filter: filter
    )
  end

  @doc "Show a single post"
  def show(conn, %{"id" => _id}) do
    post = conn.assigns.post

    # Increment view count
    Posts.increment_views(post)

    # Get related posts
    related = Posts.related_posts(post, limit: 3)

    # Get comments
    comments = Blog.Comments.list_comments(post_id: post.id)

    render(conn, :show,
      post: post,
      related: related,
      comments: comments,
      comment_changeset: Blog.Comments.change_comment(%Blog.Comment{})
    )
  end

  @doc "Render new post form"
  def new(conn, _params) do
    changeset = Posts.change_post(%Post{})
    categories = Blog.Categories.list_categories()
    tags = Blog.Tags.list_tags()

    render(conn, :new,
      changeset: changeset,
      categories: categories,
      tags: tags
    )
  end

  @doc "Create a new post"
  def create(conn, %{"post" => post_params}) do
    # เพิ่ม current_user_id
    post_params = Map.put(post_params, "author_id", conn.assigns.current_user.id)

    case Posts.create_post(post_params) do
      {:ok, post} ->
        conn
        |> put_flash(:info, "Post '#{post.title}' created successfully.")
        |> redirect(to: ~p"/posts/#{post}")

      {:error, %Ecto.Changeset{} = changeset} ->
        categories = Blog.Categories.list_categories()
        tags = Blog.Tags.list_tags()

        conn
        |> put_flash(:error, "Please fix the errors below.")
        |> render(:new,
          changeset: changeset,
          categories: categories,
          tags: tags
        )
    end
  end

  @doc "Render edit form"
  def edit(conn, _params) do
    post = conn.assigns.post
    changeset = Posts.change_post(post)
    categories = Blog.Categories.list_categories()
    tags = Blog.Tags.list_tags()

    render(conn, :edit,
      post: post,
      changeset: changeset,
      categories: categories,
      tags: tags
    )
  end

  @doc "Update a post"
  def update(conn, %{"post" => post_params}) do
    post = conn.assigns.post

    case Posts.update_post(post, post_params) do
      {:ok, post} ->
        conn
        |> put_flash(:info, "Post updated successfully.")
        |> redirect(to: ~p"/posts/#{post}")

      {:error, %Ecto.Changeset{} = changeset} ->
        categories = Blog.Categories.list_categories()
        tags = Blog.Tags.list_tags()

        conn
        |> put_flash(:error, "Please fix the errors below.")
        |> render(:edit,
          post: post,
          changeset: changeset,
          categories: categories,
          tags: tags
        )
    end
  end

  @doc "Delete a post"
  def delete(conn, _params) do
    post = conn.assigns.post

    case Posts.delete_post(post) do
      {:ok, _} ->
        conn
        |> put_flash(:info, "Post '#{post.title}' deleted.")
        |> redirect(to: ~p"/posts")

      {:error, _} ->
        conn
        |> put_flash(:error, "Could not delete post.")
        |> redirect(to: ~p"/posts/#{post}")
    end
  end

  @doc "Publish/unpublish a post"
  def toggle_publish(conn, _params) do
    post = conn.assigns.post

    case Posts.toggle_publish(post) do
      {:ok, updated_post} ->
        status = if updated_post.is_published, do: "published", else: "unpublished"
        conn
        |> put_flash(:info, "Post #{status}.")
        |> redirect(to: ~p"/posts/#{post}")

      {:error, _} ->
        conn
        |> put_flash(:error, "Could not update post status.")
        |> redirect(to: ~p"/posts/#{post}")
    end
  end

  # ============= Private Helpers =============

  defp build_filter(params) do
    %{
      search: Map.get(params, "q", ""),
      category: Map.get(params, "category"),
      tag: Map.get(params, "tag"),
      author: Map.get(params, "author"),
      published: Map.get(params, "published", "true") == "true"
    }
  end

  defp authenticate(conn, _opts) do
    case conn.assigns[:current_user] do
      nil ->
        conn
        |> put_flash(:error, "You must be logged in.")
        |> redirect(to: ~p"/login")
        |> halt()
      _user ->
        conn
    end
  end

  defp load_post(conn, _opts) do
    case Posts.get_post(conn.params["id"]) do
      nil ->
        conn
        |> put_status(:not_found)
        |> put_flash(:error, "Post not found.")
        |> redirect(to: ~p"/posts")
        |> halt()
      post ->
        assign(conn, :post, post)
    end
  end

  defp authorize_post(conn, _opts) do
    %{current_user: user, post: post} = conn.assigns

    cond do
      user.role == :admin ->
        conn
      post.author_id == user.id ->
        conn
      true ->
        conn
        |> put_flash(:error, "You don't have permission to edit this post.")
        |> redirect(to: ~p"/posts/#{conn.assigns.post}")
        |> halt()
    end
  end
end
```

---

## 9. API Controller พร้อม Error Handling

```elixir
# lib/blog_web/controllers/api/v1/post_controller.ex
defmodule BlogWeb.API.V1.PostController do
  use BlogWeb, :controller

  alias Blog.Posts
  alias Blog.Posts.Post

  action_fallback BlogWeb.FallbackController

  def index(conn, params) do
    page = Map.get(params, "page", "1") |> String.to_integer()
    per_page = Map.get(params, "per_page", "20") |> min(100) |> String.to_integer()

    {posts, meta} = Posts.list_posts_paginated(page: page, per_page: per_page)

    conn
    |> put_status(:ok)
    |> render(:index, posts: posts, meta: meta)
  end

  def show(conn, %{"id" => id}) do
    with {:ok, post} <- Posts.fetch_post(id) do
      render(conn, :show, post: post)
    end
  end

  def create(conn, %{"post" => post_params}) do
    current_user = conn.assigns.current_user

    with {:ok, post} <- Posts.create_post(Map.put(post_params, "author_id", current_user.id)) do
      conn
      |> put_status(:created)
      |> put_resp_header("location", ~p"/api/v1/posts/#{post}")
      |> render(:show, post: post)
    end
  end

  def update(conn, %{"id" => id, "post" => post_params}) do
    with {:ok, post} <- Posts.fetch_post(id),
         :ok <- authorize(conn.assigns.current_user, post),
         {:ok, updated_post} <- Posts.update_post(post, post_params) do
      render(conn, :show, post: updated_post)
    end
  end

  def delete(conn, %{"id" => id}) do
    with {:ok, post} <- Posts.fetch_post(id),
         :ok <- authorize(conn.assigns.current_user, post),
         {:ok, _} <- Posts.delete_post(post) do
      send_resp(conn, :no_content, "")
    end
  end

  defp authorize(%{role: :admin}, _post), do: :ok
  defp authorize(%{id: user_id}, %{author_id: user_id}), do: :ok
  defp authorize(_, _), do: {:error, :forbidden}
end

# Fallback controller สำหรับ with/1 errors
defmodule BlogWeb.FallbackController do
  use Phoenix.Controller

  def call(conn, {:error, :not_found}) do
    conn
    |> put_status(:not_found)
    |> json(%{error: "Resource not found"})
  end

  def call(conn, {:error, :forbidden}) do
    conn
    |> put_status(:forbidden)
    |> json(%{error: "Access denied"})
  end

  def call(conn, {:error, %Ecto.Changeset{} = changeset}) do
    conn
    |> put_status(:unprocessable_entity)
    |> json(%{errors: format_errors(changeset)})
  end

  defp format_errors(changeset) do
    Ecto.Changeset.traverse_errors(changeset, fn {msg, opts} ->
      Regex.replace(~r"%{(\w+)}", msg, fn _, key ->
        opts |> Keyword.get(String.to_existing_atom(key), key) |> to_string()
      end)
    end)
  end
end
```

---

## 10. Exercises

### Exercise 1: สร้าง Session Controller

สร้าง controller สำหรับ login/logout

**เฉลย:**

```elixir
defmodule BlogWeb.SessionController do
  use BlogWeb, :controller

  alias Blog.Accounts

  def new(conn, _params) do
    render(conn, :new)
  end

  def create(conn, %{"session" => %{"email" => email, "password" => password}}) do
    case Accounts.authenticate(email, password) do
      {:ok, user} ->
        conn
        |> put_session(:user_id, user.id)
        |> put_flash(:info, "Welcome back, #{user.name}!")
        |> redirect(to: ~p"/")

      {:error, :invalid_credentials} ->
        conn
        |> put_flash(:error, "Invalid email or password.")
        |> render(:new)
    end
  end

  def delete(conn, _params) do
    conn
    |> clear_session()
    |> put_flash(:info, "Logged out successfully.")
    |> redirect(to: ~p"/")
  end
end
```

### Exercise 2: Upload Controller

สร้าง controller สำหรับ upload รูปภาพ

**เฉลย:**

```elixir
defmodule BlogWeb.UploadController do
  use BlogWeb, :controller

  def create(conn, %{"upload" => %Plug.Upload{} = upload}) do
    case handle_upload(upload) do
      {:ok, url} ->
        json(conn, %{url: url})

      {:error, reason} ->
        conn
        |> put_status(:unprocessable_entity)
        |> json(%{error: reason})
    end
  end

  defp handle_upload(%Plug.Upload{
    content_type: content_type,
    filename: filename,
    path: path
  }) do
    # ตรวจสอบ file type
    unless content_type in ["image/jpeg", "image/png", "image/webp", "image/gif"] do
      {:error, "Only image files are allowed"}
    else
      # สร้าง unique filename
      ext = Path.extname(filename)
      unique_name = "#{System.unique_integer([:positive])}#{ext}"
      dest = Path.join(["priv", "static", "uploads", unique_name])

      # Copy file
      File.copy!(path, dest)

      {:ok, "/uploads/#{unique_name}"}
    end
  end
end
```

---

## สรุป

```
Phoenix Controllers:
├── use BlogWeb, :controller
├── Actions: def action(conn, params) do ... end
├── conn struct:
│   ├── .params - all params
│   ├── .assigns - custom data
│   ├── .session - session data
│   └── .req_headers - request headers
├── Responses:
│   ├── render/2,3 - render template
│   ├── json/2 - JSON response
│   ├── text/2 - plain text
│   ├── redirect/2 - HTTP redirect
│   └── send_resp/3 - custom response
├── Plugs:
│   ├── plug :name when action in [...]
│   └── จัดการ authentication, authorization
└── action_fallback: handle with/1 errors

Best Practices:
├── Keep controllers thin
├── Business logic ไปอยู่ใน Context
├── ใช้ plug สำหรับ shared middleware
└── ใช้ action_fallback สำหรับ error handling
```

---

*ก่อนหน้า: [Part 26 - Phoenix Routing](part_26.md) | ต่อไป: [Part 28 - Phoenix Templates (HEEx)](part_28.md)*
