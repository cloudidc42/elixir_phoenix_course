# Part 67: Full-text Search System (ระบบค้นหาข้อความ)

## เป้าหมายการเรียนรู้

หลังจากจบ Part นี้ คุณจะสามารถ:
- Full-text search ด้วย PostgreSQL tsvector
- Search ด้วย Elasticsearch/Opensearch
- Typeahead/autocomplete
- Search analytics

---

## 1. PostgreSQL Full-text Search

```elixir
# migration
defmodule MyApp.Repo.Migrations.AddSearchToArticles do
  use Ecto.Migration

  def change do
    alter table(:articles) do
      add :search_vector, :tsvector
    end

    # Create GIN index for fast text search
    create index(:articles, [:search_vector], using: :gin)

    # Auto-update search_vector with trigger
    execute """
    CREATE FUNCTION articles_search_vector_trigger() RETURNS trigger AS $$
    begin
      new.search_vector :=
        setweight(to_tsvector('thai', coalesce(new.title, '')), 'A') ||
        setweight(to_tsvector('simple', coalesce(new.title, '')), 'A') ||
        setweight(to_tsvector('simple', coalesce(new.body, '')), 'B') ||
        setweight(to_tsvector('simple', coalesce(new.tags::text, '')), 'C');
      return new;
    end
    $$ LANGUAGE plpgsql;
    """, "DROP FUNCTION articles_search_vector_trigger()"

    execute """
    CREATE TRIGGER articles_search_vector_update
    BEFORE INSERT OR UPDATE ON articles
    FOR EACH ROW EXECUTE FUNCTION articles_search_vector_trigger();
    """, "DROP TRIGGER articles_search_vector_update ON articles"
  end
end
```

---

## 2. Search Context

```elixir
defmodule MyApp.Search do
  alias MyApp.Repo
  import Ecto.Query

  def search_articles(query, opts \\ []) do
    limit = Keyword.get(opts, :limit, 20)
    offset = Keyword.get(opts, :offset, 0)

    search_query = format_query(query)

    from(a in MyApp.Article,
      where: fragment(
        "search_vector @@ plainto_tsquery('simple', ?)",
        ^search_query
      ),
      order_by: [
        desc: fragment(
          "ts_rank(search_vector, plainto_tsquery('simple', ?))",
          ^search_query
        )
      ],
      select_merge: %{
        highlight: fragment(
          "ts_headline('simple', ?, plainto_tsquery('simple', ?), 'MaxFragments=2,MaxWords=30,MinWords=10')",
          a.body, ^search_query
        )
      },
      where: a.published == true,
      limit: ^limit,
      offset: ^offset,
      preload: [:author, :category]
    )
    |> Repo.all()
  end

  def search_users(query) do
    search_query = "%#{query}%"
    from(u in MyApp.User,
      where: ilike(u.name, ^search_query) or ilike(u.email, ^search_query),
      limit: 10
    )
    |> Repo.all()
  end

  def global_search(query) do
    %{
      articles: search_articles(query, limit: 5),
      users: search_users(query),
    }
  end

  defp format_query(query) do
    query
    |> String.trim()
    |> String.split()
    |> Enum.join(" & ")
  end
end
```

---

## 3. Typeahead with LiveView

```elixir
defmodule MyAppWeb.SearchLive do
  use MyAppWeb, :live_view
  alias MyApp.Search

  def mount(_params, _session, socket) do
    {:ok, assign(socket, query: "", results: [], loading: false)}
  end

  def handle_event("search", %{"query" => query}, socket) when byte_size(query) < 2 do
    {:noreply, assign(socket, query: query, results: [])}
  end

  def handle_event("search", %{"query" => query}, socket) do
    # Cancel previous timer
    if timer = socket.assigns[:debounce_timer], do: Process.cancel_timer(timer)

    timer = Process.send_after(self(), {:do_search, query}, 300)
    {:noreply, assign(socket, query: query, loading: true, debounce_timer: timer)}
  end

  def handle_info({:do_search, query}, socket) do
    results = Search.global_search(query)
    {:noreply, assign(socket, results: results, loading: false)}
  end

  def render(assigns) do
    ~H"""
    <div class="relative" id="search-container" phx-click-away={JS.push("clear_results")}>
      <input
        type="text"
        value={@query}
        placeholder="ค้นหา..."
        phx-input="search"
        phx-debounce="0"
        class="w-full border rounded-lg px-4 py-2 pr-10"
      />
      <%= if @loading do %>
        <div class="absolute right-3 top-3">
          <div class="animate-spin w-4 h-4 border-2 border-blue-500 rounded-full border-t-transparent"></div>
        </div>
      <% end %>

      <%= if @query != "" and @results != [] do %>
        <div class="absolute top-full mt-1 w-full bg-white rounded-lg shadow-lg border z-50">
          <%= if @results[:articles] != [] do %>
            <div class="p-2 border-b">
              <p class="text-xs text-gray-500 font-semibold px-2 mb-1">บทความ</p>
              <%= for article <- @results[:articles] do %>
                <.link navigate={~p"/articles/#{article.slug}"}
                  class="block px-2 py-2 hover:bg-gray-50 rounded">
                  <p class="text-sm font-medium"><%= article.title %></p>
                  <%= if article[:highlight] do %>
                    <p class="text-xs text-gray-500 truncate">
                      <%= raw(article.highlight) %>
                    </p>
                  <% end %>
                </.link>
              <% end %>
            </div>
          <% end %>

          <%= if @results[:users] != [] do %>
            <div class="p-2">
              <p class="text-xs text-gray-500 font-semibold px-2 mb-1">ผู้ใช้</p>
              <%= for user <- @results[:users] do %>
                <.link navigate={~p"/users/#{user.id}"}
                  class="flex items-center gap-2 px-2 py-2 hover:bg-gray-50 rounded">
                  <img src={user.avatar_url || "/images/default_avatar.png"}
                    class="w-6 h-6 rounded-full" />
                  <span class="text-sm"><%= user.name %></span>
                </.link>
              <% end %>
            </div>
          <% end %>
        </div>
      <% end %>
    </div>
    """
  end
end
```

---

## 4. Elasticsearch Integration

```elixir
# mix.exs: {:elasticsearch, "~> 1.0"}

defmodule MyApp.Search.ElasticsearchCluster do
  use Elasticsearch.Cluster, otp_app: :my_app
end

# config/config.exs
config :my_app, MyApp.Search.ElasticsearchCluster,
  url: "http://localhost:9200",
  api: Elasticsearch.API.HTTP,
  json_library: Jason

defmodule MyApp.Search.ArticleIndex do
  alias MyApp.Search.ElasticsearchCluster, as: Cluster

  def index_article(article) do
    doc = %{
      title: article.title,
      body: article.body,
      author: article.author.name,
      tags: article.tags,
      published_at: article.published_at,
      category: article.category.name
    }

    Elasticsearch.put_document(Cluster, article, "articles")
  end

  def search(query, opts \\ []) do
    body = %{
      query: %{
        multi_match: %{
          query: query,
          fields: ["title^3", "body", "tags^2"],
          fuzziness: "AUTO"
        }
      },
      highlight: %{
        fields: %{
          title: %{},
          body: %{fragment_size: 150, number_of_fragments: 2}
        }
      },
      size: opts[:limit] || 10,
      from: opts[:offset] || 0
    }

    case Elasticsearch.post(Cluster, "/articles/_search", body) do
      {:ok, response} ->
        hits = get_in(response, ["hits", "hits"])
        Enum.map(hits, fn hit ->
          %{
            id: hit["_id"],
            score: hit["_score"],
            highlight: hit["highlight"],
            source: hit["_source"]
          }
        end)

      {:error, reason} ->
        require Logger
        Logger.error("Elasticsearch error: #{inspect(reason)}")
        []
    end
  end
end
```

---

## สรุป

```
Search Options:
├── PostgreSQL tsvector: built-in, no extra service
├── Elasticsearch: advanced full-text, fuzzy, facets
└── Algolia: managed search-as-a-service

PostgreSQL Search:
├── tsvector + GIN index
├── ts_rank for relevance
├── ts_headline for snippets
└── Trigger for auto-update

Performance:
├── GIN index: fast @@ operator
├── Partial index for published only
└── Caching popular searches
```

---

*ก่อนหน้า: [Part 66](part_66.md) | ต่อไป: [Part 68 - Internationalization](part_68.md)*
