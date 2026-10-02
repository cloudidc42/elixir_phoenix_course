# Part 78: Advanced Caching Strategies (กลยุทธ์ Cache ขั้นสูง)

## เป้าหมายการเรียนรู้

หลังจากจบ Part นี้ คุณจะสามารถ:
- Multi-level caching (L1 ETS, L2 Redis)
- Cache stampede prevention
- Cache invalidation strategies
- Distributed cache coordination

---

## 1. Multi-level Cache Architecture

```elixir
defmodule MyApp.Cache do
  @l1_ttl :timer.minutes(5)    # ETS: fast, local
  @l2_ttl :timer.minutes(60)   # Redis: shared, persistent

  def get(key, opts \\ [], fetcher) do
    case get_l1(key) do
      {:hit, value} ->
        value

      :miss ->
        case get_l2(key) do
          {:hit, value} ->
            # Warm L1
            put_l1(key, value, @l1_ttl)
            value

          :miss ->
            # Fetch from source
            value = prevent_stampede(key, fetcher, opts)
            put_l1(key, value, @l1_ttl)
            put_l2(key, value, @l2_ttl)
            value
        end
    end
  end

  defp get_l1(key) do
    case :ets.lookup(:app_cache, key) do
      [{^key, value, expires_at}] when expires_at > System.monotonic_time(:millisecond) ->
        {:hit, value}
      _ ->
        :miss
    end
  end

  defp put_l1(key, value, ttl_ms) do
    expires_at = System.monotonic_time(:millisecond) + ttl_ms
    :ets.insert(:app_cache, {key, value, expires_at})
  end

  defp get_l2(key) do
    case Redix.command(:redix, ["GET", "cache:#{key}"]) do
      {:ok, nil} -> :miss
      {:ok, binary} -> {:hit, :erlang.binary_to_term(binary)}
      {:error, _} -> :miss
    end
  end

  defp put_l2(key, value, ttl_ms) do
    ttl_sec = div(ttl_ms, 1000)
    binary = :erlang.term_to_binary(value)
    Redix.command(:redix, ["SET", "cache:#{key}", binary, "EX", ttl_sec])
  end
end
```

---

## 2. Cache Stampede Prevention

```elixir
defmodule MyApp.Cache.Mutex do
  @lock_ttl 10_000  # 10 seconds max

  def prevent_stampede(key, fetcher, _opts) do
    lock_key = "lock:#{key}"

    case acquire_lock(lock_key) do
      :ok ->
        try do
          value = fetcher.()
          value
        after
          release_lock(lock_key)
        end

      :locked ->
        # Wait and retry
        Process.sleep(100)
        case get_l2(key) do
          {:hit, value} -> value
          :miss -> prevent_stampede(key, fetcher, [])
        end
    end
  end

  defp acquire_lock(lock_key) do
    case Redix.command(:redix, ["SET", lock_key, "1", "NX", "PX", @lock_ttl]) do
      {:ok, "OK"} -> :ok
      {:ok, nil} -> :locked
    end
  end

  defp release_lock(lock_key) do
    Redix.command(:redix, ["DEL", lock_key])
  end
end
```

---

## 3. Tag-based Cache Invalidation

```elixir
defmodule MyApp.Cache.Tags do
  @redis_prefix "cache_tags:"

  # Store value with tags
  def put(key, value, tags, ttl_sec) do
    Redix.pipeline(:redix, [
      ["SET", "cache:#{key}", :erlang.term_to_binary(value), "EX", ttl_sec],
      # Register this key under each tag
      | Enum.map(tags, fn tag ->
        ["SADD", "#{@redis_prefix}#{tag}", key]
      end)
    ])
  end

  # Invalidate all keys with a specific tag
  def invalidate_tag(tag) do
    tag_key = "#{@redis_prefix}#{tag}"

    case Redix.command(:redix, ["SMEMBERS", tag_key]) do
      {:ok, keys} when keys != [] ->
        cache_keys = Enum.map(keys, &"cache:#{&1}")
        Redix.pipeline(:redix, [
          ["DEL" | cache_keys],
          ["DEL", tag_key]
        ])
      _ -> :ok
    end
  end
end

# ใช้งาน
def get_article(id) do
  MyApp.Cache.get("article:#{id}", fn ->
    article = MyApp.Blog.get_article!(id)
    MyApp.Cache.Tags.put(
      "article:#{id}",
      article,
      ["articles", "article:#{id}", "user:#{article.author_id}"],
      3600
    )
    article
  end)
end

# เมื่อ article อัปเดต
def update_article(article, attrs) do
  {:ok, updated} = MyApp.Blog.update_article(article, attrs)
  MyApp.Cache.Tags.invalidate_tag("article:#{article.id}")
  {:ok, updated}
end
```

---

## 4. Fragment Caching in LiveView

```elixir
defmodule MyAppWeb.ArticleLive do
  use MyAppWeb, :live_view

  def render(assigns) do
    ~H"""
    <article>
      <!-- Cache expensive fragments -->
      <%= cached_render("article:#{@article.id}:header", fn -> %>
        <header>
          <h1><%= @article.title %></h1>
          <.avatar user={@article.author} />
        </header>
      <% end) %>

      <div class="content">
        <%= raw(@article.content) %>
      </div>

      <!-- Comments are dynamic, not cached -->
      <.live_component module={CommentsComponent}
        id="comments" article_id={@article.id} />
    </article>
    """
  end

  defp cached_render(key, fun) do
    case MyApp.Cache.get(key) do
      {:hit, html} -> html
      :miss ->
        html = fun.() |> Phoenix.HTML.safe_to_string()
        MyApp.Cache.put(key, html, ttl: :timer.hours(1))
        html
    end
  end
end
```

---

## 5. Cache Warming

```elixir
defmodule MyApp.CacheWarmer do
  use GenServer

  def start_link(_) do
    GenServer.start_link(__MODULE__, %{}, name: __MODULE__)
  end

  def init(_) do
    schedule_warm()
    {:ok, %{}}
  end

  def handle_info(:warm, state) do
    warm_popular_articles()
    warm_homepage_data()
    schedule_warm()
    {:noreply, state}
  end

  defp warm_popular_articles do
    MyApp.Blog.list_popular_articles(limit: 50)
    |> Task.async_stream(fn article ->
      MyApp.Cache.put("article:#{article.id}", article, ttl: :timer.hours(2))
    end, max_concurrency: 10)
    |> Stream.run()
  end

  defp warm_homepage_data do
    data = %{
      featured_articles: MyApp.Blog.featured_articles(),
      popular_tags: MyApp.Blog.popular_tags(20),
      recent_articles: MyApp.Blog.recent_articles(10)
    }
    MyApp.Cache.put("homepage_data", data, ttl: :timer.minutes(30))
  end

  defp schedule_warm do
    Process.send_after(self(), :warm, :timer.minutes(15))
  end
end
```

---

## สรุป

```
Cache Architecture:
├── L1 (ETS): microsecond reads, local process
├── L2 (Redis): millisecond reads, distributed
└── Source (DB): ground truth

Invalidation Strategies:
├── TTL-based: expire after N seconds
├── Tag-based: invalidate by entity group
└── Event-driven: invalidate on write

Stampede Prevention:
├── Redis distributed lock
├── Wait and retry
└── Background refresh (stale-while-revalidate)

Cache Warming:
├── Pre-populate on startup
├── Refresh popular content periodically
└── Background worker
```

---

*ก่อนหน้า: [Part 77](part_77.md) | ต่อไป: [Part 79 - Advanced Ecto Queries](part_79.md)*
