# Part 28: Phoenix Templates (HEEx)

## เป้าหมายการเรียนรู้

หลังจากจบ Part นี้ คุณจะสามารถ:
- เขียน HEEx templates ได้
- สร้างและใช้ Components ได้
- กำหนด Layouts ได้
- ใช้ Form helpers ได้
- สร้าง Blog template system ที่ครบถ้วน

---

## 1. HEEx คืออะไร?

HEEx (HTML + EEx with Extensions) คือ template language ของ Phoenix ที่:
- **Type-safe**: ตรวจสอบ HTML structure ตอน compile
- **XSS-safe**: escape HTML entities อัตโนมัติ
- **Components**: รองรับ function components และ slot
- **Phoenix-specific**: มี features พิเศษสำหรับ Phoenix

```heex
<%# EEx comment %>

<%# Output expression (HTML-escaped) %>
<%= @post.title %>

<%# Unescaped output (ระวัง XSS!) %>
<%- @post.body %>

<%# Phoenix HEEx expression %>
{@post.title}

<%# Conditional %>
<%= if @post.published do %>
  <span class="published">Published</span>
<% else %>
  <span class="draft">Draft</span>
<% end %>

<%# Loop %>
<%= for post <- @posts do %>
  <div><%= post.title %></div>
<% end %>
```

---

## 2. โครงสร้าง Templates

```
lib/blog_web/
├── components/           # Reusable components
│   ├── core_components.ex  # Phoenix default components
│   ├── layouts.ex          # Layout components
│   └── blog_components.ex  # Custom blog components
├── controllers/
│   └── post_html/         # Post HTML views
│       ├── index.html.heex
│       ├── show.html.heex
│       ├── new.html.heex
│       └── edit.html.heex
└── layouts/
    ├── root.html.heex     # Root HTML structure
    └── app.html.heex      # App layout
```

---

## 3. Layout System

### Root Layout (root.html.heex)

```heex
<!DOCTYPE html>
<html lang="en" class="[scrollbar-gutter:stable]">
  <head>
    <meta charset="utf-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1" />
    <meta name="csrf-token" content={get_csrf_token()} />
    <.live_title suffix=" · My Blog">
      <%= assigns[:page_title] || "My Blog" %>
    </.live_title>
    <link phx-track-static rel="stylesheet" href={~p"/assets/app.css"} />
    <script defer phx-track-static type="text/javascript" src={~p"/assets/app.js"}>
    </script>
  </head>
  <body class="bg-white antialiased">
    {@inner_content}
  </body>
</html>
```

### App Layout (app.html.heex)

```heex
<header class="bg-gray-900 text-white">
  <nav class="max-w-7xl mx-auto px-4 py-4 flex items-center justify-between">
    <div class="flex items-center space-x-4">
      <a href={~p"/"} class="text-xl font-bold">My Blog</a>
      <a href={~p"/posts"} class="hover:text-gray-300">Posts</a>
      <a href={~p"/about"} class="hover:text-gray-300">About</a>
    </div>

    <div class="flex items-center space-x-4">
      <%= if @current_user do %>
        <span class="text-gray-400">Hello, <%= @current_user.name %></span>
        <a href={~p"/posts/new"} class="bg-blue-600 px-3 py-1 rounded">New Post</a>
        <.link href={~p"/logout"} method="delete" class="text-gray-400 hover:text-white">
          Logout
        </.link>
      <% else %>
        <a href={~p"/login"} class="hover:text-gray-300">Login</a>
        <a href={~p"/register"} class="bg-blue-600 px-3 py-1 rounded">Register</a>
      <% end %>
    </div>
  </nav>
</header>

<main class="max-w-7xl mx-auto px-4 py-8">
  <.flash_group flash={@flash} />
  {@inner_content}
</main>

<footer class="bg-gray-100 mt-16 py-8">
  <div class="max-w-7xl mx-auto px-4 text-center text-gray-500">
    <p>&copy; 2024 My Blog. Built with Phoenix Framework.</p>
  </div>
</footer>
```

---

## 4. Function Components

Components ใน Phoenix 1.7+ เป็น function ที่ return HTML

```elixir
# lib/blog_web/components/blog_components.ex
defmodule BlogWeb.BlogComponents do
  use Phoenix.Component

  import BlogWeb.CoreComponents
  alias Phoenix.LiveView.JS

  # ============= Post Card =============

  @doc """
  Renders a post card for listing pages.

  ## Examples

      <.post_card post={@post} />
      <.post_card post={@post} show_author={true} />
  """
  attr :post, :map, required: true
  attr :show_author, :boolean, default: true
  attr :class, :string, default: ""

  def post_card(assigns) do
    ~H"""
    <article class={"bg-white rounded-lg shadow-sm border hover:shadow-md transition-shadow #{@class}"}>
      <%= if @post.cover_image do %>
        <img
          src={@post.cover_image}
          alt={@post.title}
          class="w-full h-48 object-cover rounded-t-lg"
        />
      <% end %>

      <div class="p-6">
        <div class="flex items-center gap-2 mb-3">
          <%= if @post.category do %>
            <.category_badge category={@post.category} />
          <% end %>
          <time class="text-sm text-gray-500" datetime={to_string(@post.published_at)}>
            <%= format_date(@post.published_at) %>
          </time>
        </div>

        <h2 class="text-xl font-semibold mb-2">
          <a href={~p"/posts/#{@post}"} class="hover:text-blue-600">
            <%= @post.title %>
          </a>
        </h2>

        <%= if @post.excerpt do %>
          <p class="text-gray-600 text-sm line-clamp-3">
            <%= @post.excerpt %>
          </p>
        <% end %>

        <div class="flex items-center justify-between mt-4">
          <%= if @show_author do %>
            <.author_info author={@post.author} size={:small} />
          <% end %>

          <div class="flex items-center gap-3 text-sm text-gray-500">
            <span>
              <.icon name="hero-eye" class="w-4 h-4 inline" />
              <%= @post.view_count %> views
            </span>
            <span>
              <.icon name="hero-chat-bubble-left" class="w-4 h-4 inline" />
              <%= length(@post.comments) %> comments
            </span>
          </div>
        </div>
      </div>
    </article>
    """
  end

  # ============= Category Badge =============

  attr :category, :map, required: true
  attr :size, :atom, default: :medium, values: [:small, :medium, :large]

  def category_badge(assigns) do
    ~H"""
    <a
      href={~p"/categories/#{@category.slug}"}
      class={[
        "inline-flex items-center rounded-full font-medium",
        "bg-blue-100 text-blue-800",
        size_class(@size)
      ]}
    >
      <%= @category.name %>
    </a>
    """
  end

  defp size_class(:small), do: "text-xs px-2 py-0.5"
  defp size_class(:medium), do: "text-sm px-3 py-1"
  defp size_class(:large), do: "text-base px-4 py-1.5"

  # ============= Author Info =============

  attr :author, :map, required: true
  attr :size, :atom, default: :medium, values: [:small, :medium, :large]

  def author_info(assigns) do
    ~H"""
    <div class="flex items-center gap-2">
      <img
        src={@author.avatar || "/images/default-avatar.png"}
        alt={@author.name}
        class={avatar_size(@size)}
      />
      <div>
        <p class={["font-medium", author_text_size(@size)]}><%= @author.name %></p>
        <%= if @size != :small do %>
          <p class="text-xs text-gray-500"><%= @author.bio %></p>
        <% end %>
      </div>
    </div>
    """
  end

  defp avatar_size(:small), do: "w-6 h-6 rounded-full"
  defp avatar_size(:medium), do: "w-10 h-10 rounded-full"
  defp avatar_size(:large), do: "w-16 h-16 rounded-full"

  defp author_text_size(:small), do: "text-sm"
  defp author_text_size(:medium), do: "text-base"
  defp author_text_size(:large), do: "text-lg"

  # ============= Pagination =============

  attr :page, :integer, required: true
  attr :total_pages, :integer, required: true
  attr :path, :string, required: true

  def pagination(assigns) do
    ~H"""
    <%= if @total_pages > 1 do %>
      <nav class="flex items-center justify-center gap-1 mt-8" aria-label="Pagination">
        <.link
          href={"#{@path}?page=#{@page - 1}"}
          class={[
            "px-3 py-2 rounded border",
            if(@page <= 1, do: "opacity-50 cursor-not-allowed", else: "hover:bg-gray-100")
          ]}
          aria-disabled={@page <= 1}
        >
          Previous
        </.link>

        <%= for p <- visible_pages(@page, @total_pages) do %>
          <%= if p == :ellipsis do %>
            <span class="px-3 py-2">...</span>
          <% else %>
            <.link
              href={"#{@path}?page=#{p}"}
              class={[
                "px-3 py-2 rounded border",
                if(p == @page, do: "bg-blue-600 text-white border-blue-600", else: "hover:bg-gray-100")
              ]}
            >
              <%= p %>
            </.link>
          <% end %>
        <% end %>

        <.link
          href={"#{@path}?page=#{@page + 1}"}
          class={[
            "px-3 py-2 rounded border",
            if(@page >= @total_pages, do: "opacity-50 cursor-not-allowed", else: "hover:bg-gray-100")
          ]}
          aria-disabled={@page >= @total_pages}
        >
          Next
        </.link>
      </nav>
    <% end %>
    """
  end

  defp visible_pages(current, total) do
    cond do
      total <= 7 -> Enum.to_list(1..total)
      current <= 3 -> [1, 2, 3, 4, 5, :ellipsis, total]
      current >= total - 2 -> [1, :ellipsis, total-4, total-3, total-2, total-1, total]
      true -> [1, :ellipsis, current-1, current, current+1, :ellipsis, total]
    end
  end

  # ============= Helpers =============

  defp format_date(nil), do: "N/A"
  defp format_date(datetime) do
    Calendar.strftime(datetime, "%B %d, %Y")
  end
end
```

---

## 5. Slots ใน Components

```elixir
defmodule BlogWeb.UI do
  use Phoenix.Component

  # ============= Card Component with Slots =============

  slot :inner_block, required: true
  slot :header
  slot :footer
  attr :class, :string, default: ""

  def card(assigns) do
    ~H"""
    <div class={"bg-white rounded-lg shadow #{@class}"}>
      <%= if @header != [] do %>
        <div class="border-b px-6 py-4">
          <%= render_slot(@header) %>
        </div>
      <% end %>

      <div class="px-6 py-4">
        <%= render_slot(@inner_block) %>
      </div>

      <%= if @footer != [] do %>
        <div class="border-t px-6 py-4 bg-gray-50">
          <%= render_slot(@footer) %>
        </div>
      <% end %>
    </div>
    """
  end

  # ============= Modal Component =============

  attr :id, :string, required: true
  attr :show, :boolean, default: false
  attr :title, :string, default: nil
  slot :inner_block, required: true

  def modal(assigns) do
    ~H"""
    <div
      id={@id}
      class={["fixed inset-0 z-50", unless @show do "hidden" end]}
      phx-remove={hide_modal(@id)}
    >
      <%# Backdrop %>
      <div
        class="fixed inset-0 bg-black bg-opacity-50"
        phx-click={hide_modal(@id)}
      />

      <%# Modal content %>
      <div class="fixed inset-0 flex items-center justify-center p-4">
        <div class="bg-white rounded-lg shadow-xl max-w-lg w-full max-h-screen overflow-y-auto">
          <%= if @title do %>
            <div class="flex items-center justify-between p-6 border-b">
              <h2 class="text-xl font-semibold"><%= @title %></h2>
              <button phx-click={hide_modal(@id)} class="text-gray-400 hover:text-gray-600">
                <.icon name="hero-x-mark" class="w-6 h-6" />
              </button>
            </div>
          <% end %>

          <div class="p-6">
            <%= render_slot(@inner_block) %>
          </div>
        </div>
      </div>
    </div>
    """
  end

  defp hide_modal(id) do
    %JS{}
    |> JS.hide(to: "##{id}")
  end

  # ============= Table Component =============

  attr :rows, :list, required: true
  attr :id, :string, required: true
  slot :col, required: true do
    attr :label, :string
    attr :class, :string
  end
  slot :action

  def table(assigns) do
    ~H"""
    <div class="overflow-x-auto">
      <table class="min-w-full divide-y divide-gray-200" id={@id}>
        <thead class="bg-gray-50">
          <tr>
            <%= for col <- @col do %>
              <th class={"px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider #{col[:class]}"}>
                <%= col[:label] %>
              </th>
            <% end %>
            <%= if @action != [] do %>
              <th class="relative px-6 py-3">
                <span class="sr-only">Actions</span>
              </th>
            <% end %>
          </tr>
        </thead>
        <tbody class="bg-white divide-y divide-gray-200">
          <%= for row <- @rows do %>
            <tr>
              <%= for col <- @col do %>
                <td class={"px-6 py-4 #{col[:class]}"}>
                  <%= render_slot(col, row) %>
                </td>
              <% end %>
              <%= if @action != [] do %>
                <td class="px-6 py-4 text-right text-sm">
                  <%= render_slot(@action, row) %>
                </td>
              <% end %>
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

## 6. Form Helpers

```heex
<%# lib/blog_web/controllers/post_html/new.html.heex %>

<div class="max-w-3xl mx-auto">
  <h1 class="text-3xl font-bold mb-8">New Post</h1>

  <.form
    for={@changeset}
    action={~p"/posts"}
    multipart={true}
    class="space-y-6"
  >
    <%# Title field %>
    <div>
      <.label for="post_title">Title</.label>
      <.input
        field={@changeset[:title]}
        type="text"
        placeholder="Enter post title..."
        class="mt-1"
      />
    </div>

    <%# Slug field %>
    <div>
      <.label for="post_slug">Slug</.label>
      <.input
        field={@changeset[:slug]}
        type="text"
        placeholder="my-post-slug"
      />
      <p class="text-sm text-gray-500 mt-1">
        URL: example.com/posts/<span id="slug-preview"></span>
      </p>
    </div>

    <%# Category select %>
    <div>
      <.label for="post_category_id">Category</.label>
      <.input
        field={@changeset[:category_id]}
        type="select"
        options={[{"-- Select Category --", nil} | Enum.map(@categories, &{&1.name, &1.id})]}
      />
    </div>

    <%# Body textarea %>
    <div>
      <.label for="post_body">Content</.label>
      <.input
        field={@changeset[:body]}
        type="textarea"
        rows="15"
        placeholder="Write your post content in Markdown..."
        class="font-mono"
      />
    </div>

    <%# Excerpt %>
    <div>
      <.label for="post_excerpt">Excerpt (optional)</.label>
      <.input
        field={@changeset[:excerpt]}
        type="textarea"
        rows="3"
        placeholder="Short summary of your post..."
      />
    </div>

    <%# Tags %>
    <div>
      <.label>Tags</.label>
      <div class="flex flex-wrap gap-2 mt-2">
        <%= for tag <- @tags do %>
          <label class="flex items-center gap-2 cursor-pointer">
            <input
              type="checkbox"
              name="post[tag_ids][]"
              value={tag.id}
              checked={tag.id in (Ecto.Changeset.get_field(@changeset, :tag_ids) || [])}
              class="rounded"
            />
            <span class="text-sm"><%= tag.name %></span>
          </label>
        <% end %>
      </div>
    </div>

    <%# Cover image %>
    <div>
      <.label for="post_cover_image">Cover Image</.label>
      <.input
        field={@changeset[:cover_image]}
        type="file"
        accept="image/*"
      />
    </div>

    <%# Published checkbox %>
    <div class="flex items-center gap-3">
      <.input
        field={@changeset[:is_published]}
        type="checkbox"
      />
      <.label for="post_is_published">Publish immediately</.label>
    </div>

    <%# Submit buttons %>
    <div class="flex items-center gap-4">
      <.button type="submit" class="bg-blue-600 text-white px-6 py-2 rounded hover:bg-blue-700">
        Create Post
      </.button>
      <.link href={~p"/posts"} class="text-gray-600 hover:text-gray-800">
        Cancel
      </.link>
    </div>
  </.form>
</div>
```

---

## 7. ตัวอย่างจริง: Blog Template System

### Index Template

```heex
<%# lib/blog_web/controllers/post_html/index.html.heex %>

<div class="max-w-7xl mx-auto">
  <%# Header %>
  <div class="flex items-center justify-between mb-8">
    <div>
      <h1 class="text-3xl font-bold">Blog Posts</h1>
      <p class="text-gray-600 mt-1"><%= @total %> posts</p>
    </div>

    <%= if @current_user do %>
      <.link href={~p"/posts/new"} class="bg-blue-600 text-white px-4 py-2 rounded hover:bg-blue-700">
        + New Post
      </.link>
    <% end %>
  </div>

  <%# Search & Filters %>
  <div class="bg-gray-50 rounded-lg p-4 mb-8">
    <form action={~p"/posts"} method="get" class="flex flex-wrap gap-4">
      <input
        type="text"
        name="q"
        value={@filter.search}
        placeholder="Search posts..."
        class="border rounded px-3 py-2 flex-1"
      />

      <select name="category" class="border rounded px-3 py-2">
        <option value="">All Categories</option>
        <%= for cat <- @categories do %>
          <option
            value={cat.slug}
            selected={@filter.category == cat.slug}
          >
            <%= cat.name %>
          </option>
        <% end %>
      </select>

      <button type="submit" class="bg-gray-800 text-white px-4 py-2 rounded">
        Filter
      </button>

      <%= if @filter.search != "" || @filter.category do %>
        <.link href={~p"/posts"} class="text-gray-600 underline py-2">
          Clear
        </.link>
      <% end %>
    </form>
  </div>

  <%# Posts Grid %>
  <%= if Enum.empty?(@posts) do %>
    <div class="text-center py-16">
      <p class="text-2xl text-gray-400">No posts found</p>
      <p class="text-gray-500 mt-2">
        <%= if @filter.search != "" do %>
          Try a different search term
        <% else %>
          Be the first to write a post!
        <% end %>
      </p>
    </div>
  <% else %>
    <div class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-6">
      <%= for post <- @posts do %>
        <BlogWeb.BlogComponents.post_card post={post} />
      <% end %>
    </div>

    <BlogWeb.BlogComponents.pagination
      page={@page}
      total_pages={@total_pages}
      path={~p"/posts"}
    />
  <% end %>
</div>
```

### Show Template

```heex
<%# lib/blog_web/controllers/post_html/show.html.heex %>

<article class="max-w-4xl mx-auto">
  <%# Post header %>
  <header class="mb-8">
    <div class="flex items-center gap-2 mb-4">
      <%= if @post.category do %>
        <BlogWeb.BlogComponents.category_badge category={@post.category} />
      <% end %>
      <time class="text-gray-500" datetime={to_string(@post.published_at)}>
        <%= Calendar.strftime(@post.published_at || DateTime.utc_now(), "%B %d, %Y") %>
      </time>
      <span class="text-gray-400">·</span>
      <span class="text-gray-500"><%= @post.view_count %> views</span>
    </div>

    <h1 class="text-4xl font-bold mb-4"><%= @post.title %></h1>

    <%= if @post.excerpt do %>
      <p class="text-xl text-gray-600"><%= @post.excerpt %></p>
    <% end %>

    <div class="flex items-center gap-4 mt-6">
      <BlogWeb.BlogComponents.author_info author={@post.author} size={:medium} />

      <%= if @current_user && (@current_user.id == @post.author_id || @current_user.role == :admin) do %>
        <div class="ml-auto flex items-center gap-2">
          <.link href={~p"/posts/#{@post}/edit"} class="text-sm text-blue-600 hover:underline">
            Edit
          </.link>
          <.link
            href={~p"/posts/#{@post}"}
            method="delete"
            data-confirm="Are you sure?"
            class="text-sm text-red-600 hover:underline"
          >
            Delete
          </.link>
        </div>
      <% end %>
    </div>
  </header>

  <%# Cover image %>
  <%= if @post.cover_image do %>
    <img
      src={@post.cover_image}
      alt={@post.title}
      class="w-full h-96 object-cover rounded-lg mb-8"
    />
  <% end %>

  <%# Post body %>
  <div class="prose prose-lg max-w-none mb-12">
    <%= raw(Earmark.as_html!(@post.body)) %>
  </div>

  <%# Tags %>
  <%= if @post.tags != [] do %>
    <div class="flex flex-wrap gap-2 mb-8">
      <%= for tag <- @post.tags do %>
        <.link
          href={~p"/tags/#{tag.slug}"}
          class="px-3 py-1 bg-gray-100 rounded-full text-sm hover:bg-gray-200"
        >
          #<%= tag.name %>
        </.link>
      <% end %>
    </div>
  <% end %>

  <%# Share buttons %>
  <div class="border-t border-b py-4 my-8">
    <p class="text-sm text-gray-500 mb-3">Share this post:</p>
    <div class="flex gap-3">
      <a
        href={"https://twitter.com/intent/tweet?text=#{URI.encode(@post.title)}&url=#{url(~p"/posts/#{@post}")}"}
        target="_blank"
        class="px-4 py-2 bg-sky-500 text-white rounded text-sm hover:bg-sky-600"
      >
        Share on Twitter
      </a>
      <a
        href={"https://www.facebook.com/sharer/sharer.php?u=#{url(~p"/posts/#{@post}")}"}
        target="_blank"
        class="px-4 py-2 bg-blue-600 text-white rounded text-sm hover:bg-blue-700"
      >
        Share on Facebook
      </a>
    </div>
  </div>

  <%# Related posts %>
  <%= if @related != [] do %>
    <section class="mb-12">
      <h2 class="text-2xl font-bold mb-6">Related Posts</h2>
      <div class="grid grid-cols-1 md:grid-cols-3 gap-4">
        <%= for related <- @related do %>
          <BlogWeb.BlogComponents.post_card post={related} />
        <% end %>
      </div>
    </section>
  <% end %>

  <%# Comments section %>
  <section id="comments">
    <h2 class="text-2xl font-bold mb-6">
      Comments (<%= length(@comments) %>)
    </h2>

    <%# Comment list %>
    <%= if Enum.empty?(@comments) do %>
      <p class="text-gray-500 mb-8">Be the first to comment!</p>
    <% else %>
      <div class="space-y-6 mb-8">
        <%= for comment <- @comments do %>
          <div class="bg-gray-50 rounded-lg p-4">
            <div class="flex items-center justify-between mb-3">
              <div class="flex items-center gap-2">
                <div class="w-8 h-8 bg-gray-300 rounded-full flex items-center justify-center">
                  <%= String.first(comment.author_name) |> String.upcase() %>
                </div>
                <span class="font-medium"><%= comment.author_name %></span>
              </div>
              <time class="text-sm text-gray-500">
                <%= Calendar.strftime(comment.inserted_at, "%B %d, %Y") %>
              </time>
            </div>
            <p class="text-gray-700"><%= comment.body %></p>
          </div>
        <% end %>
      </div>
    <% end %>

    <%# New comment form %>
    <div class="bg-white border rounded-lg p-6">
      <h3 class="text-lg font-semibold mb-4">Leave a Comment</h3>

      <.form
        for={@comment_changeset}
        action={~p"/posts/#{@post}/comments"}
        class="space-y-4"
      >
        <div class="grid grid-cols-2 gap-4">
          <div>
            <.label for="comment_author_name">Name *</.label>
            <.input field={@comment_changeset[:author_name]} type="text" required />
          </div>
          <div>
            <.label for="comment_author_email">Email</.label>
            <.input field={@comment_changeset[:author_email]} type="email" />
          </div>
        </div>

        <div>
          <.label for="comment_body">Comment *</.label>
          <.input field={@comment_changeset[:body]} type="textarea" rows="4" required />
        </div>

        <.button type="submit" class="bg-blue-600 text-white px-6 py-2 rounded hover:bg-blue-700">
          Post Comment
        </.button>
      </.form>
    </div>
  </section>
</article>
```

---

## 8. Component View Module

```elixir
# lib/blog_web/controllers/post_html.ex
defmodule BlogWeb.PostHTML do
  use BlogWeb, :html

  # Import custom components
  import BlogWeb.BlogComponents
  import BlogWeb.UI

  # Embed templates จาก folder
  embed_templates "post_html/*"

  # Helpers สำหรับ templates
  def format_date(nil), do: "Not published"
  def format_date(datetime) do
    Calendar.strftime(datetime, "%B %d, %Y at %I:%M %p")
  end

  def reading_time(body) when is_binary(body) do
    word_count = body |> String.split() |> length()
    minutes = ceil(word_count / 200)  # 200 words per minute
    "#{minutes} min read"
  end

  def truncate(text, max_length \\ 150) when is_binary(text) do
    if String.length(text) > max_length do
      String.slice(text, 0, max_length) <> "..."
    else
      text
    end
  end
end
```

---

## 9. Exercises

### Exercise 1: สร้าง Breadcrumb Component

**เฉลย:**

```elixir
defmodule BlogWeb.UI.Breadcrumb do
  use Phoenix.Component

  attr :items, :list, required: true
  # items: [%{label: "Home", href: "/"}, %{label: "Posts", href: "/posts"}, %{label: "Current"}]

  def breadcrumb(assigns) do
    ~H"""
    <nav aria-label="Breadcrumb" class="flex items-center gap-2 text-sm text-gray-500 mb-6">
      <%= for {item, index} <- Enum.with_index(@items) do %>
        <%= if index > 0 do %>
          <span>/</span>
        <% end %>

        <%= if item[:href] do %>
          <a href={item.href} class="hover:text-gray-700 hover:underline">
            <%= item.label %>
          </a>
        <% else %>
          <span class="text-gray-900 font-medium" aria-current="page">
            <%= item.label %>
          </span>
        <% end %>
      <% end %>
    </nav>
    """
  end
end
```

```heex
<%# ใช้งาน %>
<.breadcrumb items={[
  %{label: "Home", href: ~p"/"},
  %{label: "Posts", href: ~p"/posts"},
  %{label: @post.title}
]} />
```

### Exercise 2: สร้าง Alert Component

**เฉลย:**

```elixir
defmodule BlogWeb.UI.Alert do
  use Phoenix.Component

  attr :type, :atom, default: :info, values: [:info, :success, :warning, :error]
  attr :title, :string, default: nil
  attr :dismissible, :boolean, default: false
  slot :inner_block, required: true

  def alert(assigns) do
    ~H"""
    <div
      role="alert"
      class={[
        "rounded-lg p-4 mb-4",
        type_class(@type)
      ]}
    >
      <div class="flex items-start">
        <div class="flex-shrink-0">
          <.icon name={type_icon(@type)} class="w-5 h-5" />
        </div>
        <div class="ml-3 flex-1">
          <%= if @title do %>
            <h3 class="text-sm font-medium mb-1"><%= @title %></h3>
          <% end %>
          <div class="text-sm">
            <%= render_slot(@inner_block) %>
          </div>
        </div>
        <%= if @dismissible do %>
          <button
            class="ml-auto -mx-1.5 -my-1.5 rounded-lg p-1.5 inline-flex h-8 w-8 opacity-50 hover:opacity-100"
            onclick="this.parentElement.parentElement.remove()"
          >
            <.icon name="hero-x-mark" class="w-5 h-5" />
          </button>
        <% end %>
      </div>
    </div>
    """
  end

  defp type_class(:info), do: "bg-blue-50 text-blue-800"
  defp type_class(:success), do: "bg-green-50 text-green-800"
  defp type_class(:warning), do: "bg-yellow-50 text-yellow-800"
  defp type_class(:error), do: "bg-red-50 text-red-800"

  defp type_icon(:info), do: "hero-information-circle"
  defp type_icon(:success), do: "hero-check-circle"
  defp type_icon(:warning), do: "hero-exclamation-triangle"
  defp type_icon(:error), do: "hero-x-circle"
end
```

---

## สรุป

```
HEEx Templates:
├── Syntax:
│   ├── <%= expr %> - output (escaped)
│   ├── <% expr %> - execute (no output)
│   ├── {@variable} - Phoenix sigil
│   └── <%# comment %>
├── Components:
│   ├── attr - declare attributes
│   ├── slot - declare slots
│   ├── render_slot/1,2 - render slot content
│   └── ~H""" - component body
├── Forms:
│   ├── <.form for={@changeset} action={...}>
│   ├── <.input field={@changeset[:field]} type={...}>
│   ├── <.label for="field_id">
│   └── <.button type="submit">
└── Layouts:
    ├── root.html.heex - HTML shell
    ├── app.html.heex - App layout
    └── {@inner_content} - content injection

Best Practices:
├── Keep templates simple, logic ไปอยู่ใน component
├── Reuse components แทนการ copy code
├── ใช้ attr declarations เพื่อ document
└── slot สำหรับ flexible compositions
```

---

*ก่อนหน้า: [Part 27 - Phoenix Controllers](part_27.md) | ต่อไป: [Part 29 - Ecto Schema และ Migrations](part_29.md)*
