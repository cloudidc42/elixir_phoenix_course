# Part 50: Real-world Project - Blog Platform (แพลตฟอร์มบล็อก)

## เป้าหมายการเรียนรู้

หลังจากจบ Part นี้ คุณจะสามารถ:
- สร้าง blog platform ที่สมบูรณ์
- Full CRUD ด้วย LiveView
- Authentication และ Authorization
- SEO-friendly URLs

---

## 1. Project Setup

```bash
mix phx.new blog --database postgres
cd blog

# Auth
mix phx.gen.auth Accounts User users

# Blog context
mix phx.gen.context Blog Post posts \
  title:string \
  slug:string \
  content:text \
  excerpt:text \
  published:boolean \
  published_at:utc_datetime \
  cover_image_url:string \
  user_id:references:users

mix phx.gen.context Blog Category categories \
  name:string slug:string description:text

mix phx.gen.context Blog Comment comments \
  content:text \
  post_id:references:posts \
  user_id:references:users \
  approved:boolean
```

---

## 2. Post Schema with Slugs

```elixir
defmodule Blog.Blog.Post do
  use Ecto.Schema
  import Ecto.Changeset

  schema "posts" do
    field :title, :string
    field :slug, :string
    field :content, :string
    field :excerpt, :string
    field :published, :boolean, default: false
    field :published_at, :utc_datetime
    field :cover_image_url, :string
    field :tags, {:array, :string}, default: []
    field :view_count, :integer, default: 0

    belongs_to :user, Blog.Accounts.User
    belongs_to :category, Blog.Blog.Category
    has_many :comments, Blog.Blog.Comment

    timestamps()
  end

  def changeset(post, attrs) do
    post
    |> cast(attrs, [:title, :content, :excerpt, :published, :published_at,
                    :cover_image_url, :tags, :category_id, :user_id])
    |> validate_required([:title, :content, :user_id])
    |> validate_length(:title, min: 5, max: 200)
    |> validate_length(:excerpt, max: 500)
    |> generate_slug()
    |> unique_constraint(:slug)
  end

  def publish_changeset(post) do
    post
    |> change(published: true, published_at: DateTime.utc_now() |> DateTime.truncate(:second))
    |> validate_required([:excerpt])
  end

  defp generate_slug(changeset) do
    case get_change(changeset, :title) do
      nil -> changeset
      title ->
        slug = title
          |> String.downcase()
          |> String.replace(~r/[^a-z0-9\s-]/, "")
          |> String.replace(~r/\s+/, "-")
          |> String.replace(~r/-+/, "-")
          |> String.trim("-")

        put_change(changeset, :slug, slug)
    end
  end
end
```

---

## 3. Blog Context

```elixir
defmodule Blog.Blog do
  alias Blog.Repo
  alias Blog.Blog.{Post, Comment}
  import Ecto.Query

  def list_published_posts(opts \\ []) do
    category_id = Keyword.get(opts, :category_id)
    tag = Keyword.get(opts, :tag)
    page = Keyword.get(opts, :page, 1)
    per_page = Keyword.get(opts, :per_page, 10)

    query = from(p in Post,
      where: p.published == true,
      order_by: [desc: p.published_at],
      preload: [:user, :category]
    )

    query = if category_id, do: where(query, [p], p.category_id == ^category_id), else: query
    query = if tag, do: where(query, [p], ^tag in p.tags), else: query

    query
    |> offset(^((page - 1) * per_page))
    |> limit(^per_page)
    |> Repo.all()
  end

  def get_post_by_slug!(slug) do
    Repo.get_by!(Post, slug: slug, published: true)
    |> Repo.preload([:user, :category, comments: [:user]])
  end

  def increment_view_count(post) do
    from(p in Post, where: p.id == ^post.id)
    |> Repo.update_all(inc: [view_count: 1])
  end

  def create_post(user, attrs) do
    %Post{user_id: user.id}
    |> Post.changeset(attrs)
    |> Repo.insert()
  end

  def publish_post(post) do
    post
    |> Post.publish_changeset()
    |> Repo.update()
  end

  def create_comment(post, user, attrs) do
    %Comment{post_id: post.id, user_id: user.id}
    |> Comment.changeset(attrs)
    |> Repo.insert()
  end

  def list_user_posts(user_id, opts \\ []) do
    page = Keyword.get(opts, :page, 1)
    per_page = 20

    from(p in Post,
      where: p.user_id == ^user_id,
      order_by: [desc: p.inserted_at],
      preload: [:category]
    )
    |> offset(^((page - 1) * per_page))
    |> limit(^per_page)
    |> Repo.all()
  end

  def search_posts(query_string) do
    term = "%#{query_string}%"
    from(p in Post,
      where: p.published == true
        and (ilike(p.title, ^term) or ilike(p.content, ^term)),
      order_by: [desc: p.published_at],
      limit: 20,
      preload: [:user]
    )
    |> Repo.all()
  end
end
```

---

## 4. Blog LiveView

```elixir
defmodule BlogWeb.PostLive.Index do
  use BlogWeb, :live_view
  alias Blog.Blog

  def mount(params, _session, socket) do
    posts = Blog.list_published_posts(
      category_id: params["category"],
      page: String.to_integer(params["page"] || "1")
    )

    {:ok, assign(socket, posts: posts, page_title: "Blog")}
  end

  def render(assigns) do
    ~H"""
    <div class="max-w-4xl mx-auto py-8 px-4">
      <h1 class="text-4xl font-bold mb-8">บล็อก</h1>

      <div class="space-y-8">
        <%= for post <- @posts do %>
          <article class="border-b pb-8">
            <%= if post.cover_image_url do %>
              <img src={post.cover_image_url} class="w-full h-48 object-cover rounded-lg mb-4" />
            <% end %>

            <div class="flex items-center gap-2 text-sm text-gray-500 mb-2">
              <span><%= post.user.name %></span>
              <span>•</span>
              <span><%= format_date(post.published_at) %></span>
              <%= if post.category do %>
                <span>•</span>
                <span class="text-blue-600"><%= post.category.name %></span>
              <% end %>
            </div>

            <.link navigate={~p"/posts/#{post.slug}"}>
              <h2 class="text-2xl font-semibold hover:text-blue-600">
                <%= post.title %>
              </h2>
            </.link>

            <p class="text-gray-600 mt-2"><%= post.excerpt %></p>

            <div class="flex gap-2 mt-3">
              <%= for tag <- post.tags do %>
                <span class="bg-gray-100 text-gray-700 px-2 py-1 rounded text-xs">
                  #<%= tag %>
                </span>
              <% end %>
            </div>
          </article>
        <% end %>
      </div>
    </div>
    """
  end

  defp format_date(nil), do: ""
  defp format_date(dt), do: Calendar.strftime(dt, "%d %B %Y")
end

defmodule BlogWeb.PostLive.Show do
  use BlogWeb, :live_view
  alias Blog.Blog

  def mount(%{"slug" => slug}, session, socket) do
    post = Blog.get_post_by_slug!(slug)
    Blog.increment_view_count(post)

    current_user = session["current_user"]

    {:ok, assign(socket,
      post: post,
      current_user: current_user,
      page_title: post.title,
      new_comment: ""
    )}
  end

  def handle_event("submit_comment", %{"content" => content}, socket) do
    case socket.assigns.current_user do
      nil ->
        {:noreply, push_navigate(socket, to: "/users/log_in")}

      user ->
        case Blog.create_comment(socket.assigns.post, user, %{content: content}) do
          {:ok, _comment} ->
            post = Blog.get_post_by_slug!(socket.assigns.post.slug)
            {:noreply, assign(socket, post: post, new_comment: "")}

          {:error, _changeset} ->
            {:noreply, put_flash(socket, :error, "ไม่สามารถโพสต์ความคิดเห็น")}
        end
    end
  end

  def render(assigns) do
    ~H"""
    <article class="max-w-3xl mx-auto py-8 px-4">
      <header class="mb-8">
        <h1 class="text-4xl font-bold"><%= @post.title %></h1>
        <div class="flex items-center gap-3 mt-4 text-gray-500">
          <span>โดย <%= @post.user.name %></span>
          <span>•</span>
          <span><%= format_date(@post.published_at) %></span>
          <span>•</span>
          <span><%= @post.view_count %> views</span>
        </div>
      </header>

      <%= if @post.cover_image_url do %>
        <img src={@post.cover_image_url} class="w-full rounded-lg mb-8" />
      <% end %>

      <div class="prose max-w-none">
        <%= raw(Earmark.as_html!(@post.content)) %>
      </div>

      <!-- Comments -->
      <section class="mt-12 border-t pt-8">
        <h3 class="text-xl font-bold mb-6">ความคิดเห็น (<%= length(@post.comments) %>)</h3>

        <%= for comment <- @post.comments do %>
          <div class="mb-6">
            <div class="flex items-center gap-2 mb-2">
              <span class="font-semibold"><%= comment.user.name %></span>
              <span class="text-sm text-gray-500">
                <%= Calendar.strftime(comment.inserted_at, "%d %b %Y %H:%M") %>
              </span>
            </div>
            <p class="text-gray-700"><%= comment.content %></p>
          </div>
        <% end %>

        <%= if @current_user do %>
          <form phx-submit="submit_comment" class="mt-6">
            <textarea
              name="content"
              rows="4"
              class="w-full border rounded-lg p-3"
              placeholder="แสดงความคิดเห็น..."
            ><%= @new_comment %></textarea>
            <button type="submit" class="mt-2 bg-blue-600 text-white px-6 py-2 rounded-lg">
              โพสต์ความคิดเห็น
            </button>
          </form>
        <% else %>
          <p class="text-gray-500 mt-4">
            <.link navigate={~p"/users/log_in"} class="text-blue-600">เข้าสู่ระบบ</.link>
            เพื่อแสดงความคิดเห็น
          </p>
        <% end %>
      </section>
    </article>
    """
  end

  defp format_date(nil), do: ""
  defp format_date(dt), do: Calendar.strftime(dt, "%d %B %Y")
end
```

---

## 5. Admin Dashboard LiveView

```elixir
defmodule BlogWeb.Admin.PostsLive do
  use BlogWeb, :live_view

  alias Blog.Blog
  on_mount {BlogWeb.UserAuth, :require_admin}

  def mount(_params, _session, socket) do
    posts = Blog.list_user_posts(socket.assigns.current_user.id)
    {:ok, assign(socket, posts: posts)}
  end

  def handle_event("publish", %{"id" => id}, socket) do
    post = Blog.get_post!(id)
    {:ok, _} = Blog.publish_post(post)
    posts = Blog.list_user_posts(socket.assigns.current_user.id)
    {:noreply, assign(socket, posts: posts)}
  end

  def handle_event("delete", %{"id" => id}, socket) do
    post = Blog.get_post!(id)
    {:ok, _} = Blog.delete_post(post)
    posts = Blog.list_user_posts(socket.assigns.current_user.id)
    {:noreply, put_flash(assign(socket, posts: posts), :info, "ลบโพสต์แล้ว")}
  end

  def render(assigns) do
    ~H"""
    <div class="p-6">
      <div class="flex justify-between mb-6">
        <h1 class="text-2xl font-bold">โพสต์ของฉัน</h1>
        <.link navigate={~p"/admin/posts/new"} class="bg-blue-600 text-white px-4 py-2 rounded">
          + เขียนโพสต์ใหม่
        </.link>
      </div>

      <table class="w-full">
        <thead>
          <tr class="border-b">
            <th class="text-left py-3">หัวข้อ</th>
            <th class="text-left py-3">สถานะ</th>
            <th class="text-left py-3">วันที่</th>
            <th></th>
          </tr>
        </thead>
        <tbody>
          <%= for post <- @posts do %>
            <tr class="border-b">
              <td class="py-3"><%= post.title %></td>
              <td class="py-3">
                <span class={"px-2 py-1 rounded text-sm #{if post.published, do: "bg-green-100 text-green-800", else: "bg-gray-100 text-gray-600"}"}>
                  <%= if post.published, do: "เผยแพร่", else: "ฉบับร่าง" %>
                </span>
              </td>
              <td class="py-3 text-sm text-gray-500">
                <%= Calendar.strftime(post.inserted_at, "%d/%m/%Y") %>
              </td>
              <td class="py-3 flex gap-2">
                <.link navigate={~p"/admin/posts/#{post.id}/edit"} class="text-blue-600 text-sm">
                  แก้ไข
                </.link>
                <%= unless post.published do %>
                  <button phx-click="publish" phx-value-id={post.id}
                    class="text-green-600 text-sm">
                    เผยแพร่
                  </button>
                <% end %>
                <button phx-click="delete" phx-value-id={post.id}
                  class="text-red-600 text-sm"
                  data-confirm="แน่ใจว่าต้องการลบ?">
                  ลบ
                </button>
              </td>
            </tr>
          <% end %>
        </tbody>
      </table>
    </div>
    """
  end
end
```

---

## สรุป

```
Blog Platform:
├── Authentication: phx.gen.auth
├── Posts: CRUD with slugs and publishing
├── Comments: approval flow
└── Admin: manage own posts

Features:
├── SEO slugs (auto-generated from title)
├── Cover images
├── Tags and categories
├── View counter
└── Markdown content (Earmark)

LiveView:
├── Index: paginated post list
├── Show: post + comments
└── Admin: post management
```

---

*ก่อนหน้า: [Part 49](part_49.md) | ต่อไป: [Part 51 - E-Commerce System](part_51.md)*
