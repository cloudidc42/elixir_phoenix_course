# Part 66: Search System (ระบบค้นหา)

## เป้าหมายการเรียนรู้

- สร้าง full-text search ด้วย PostgreSQL ในโปรเจกต์จริง
- เชื่อมต่อ Elasticsearch ด้วย Elastix library
- ใช้ Algolia สำหรับ managed search service
- เข้าใจการจัดลำดับผลการค้นหา (search result ranking)
- สร้าง auto-suggestions และ typeahead
- ทำ faceted search พร้อม aggregations
- เก็บ search analytics เพื่อปรับปรุงระบบ

---

## 1. PostgreSQL Full-Text Search ในระดับ Production

### 1.1 Advanced tsvector Setup

```elixir
# migration สำหรับ multi-language full-text search
defmodule MyApp.Repo.Migrations.SetupFullTextSearch do
  use Ecto.Migration

  def up do
    alter table(:products) do
      add :search_en, :tsvector
      add :search_th, :tsvector
    end

    execute "CREATE INDEX products_search_en_idx ON products USING GIN(search_en)"
    execute "CREATE INDEX products_search_th_idx ON products USING GIN(search_th)"

    execute """
    CREATE OR REPLACE FUNCTION update_products_search()
    RETURNS TRIGGER AS $$
    DECLARE
      category_name TEXT := '';
      brand_name TEXT := '';
    BEGIN
      SELECT c.name INTO category_name
      FROM categories c WHERE c.id = NEW.category_id;

      SELECT b.name INTO brand_name
      FROM brands b WHERE b.id = NEW.brand_id;

      NEW.search_en :=
        setweight(to_tsvector('english', coalesce(NEW.name, '')), 'A') ||
        setweight(to_tsvector('english', coalesce(NEW.description, '')), 'B') ||
        setweight(to_tsvector('english', coalesce(category_name, '')), 'C') ||
        setweight(to_tsvector('english', coalesce(brand_name, '')), 'C') ||
        setweight(to_tsvector('english', coalesce(array_to_string(NEW.tags, ' '), '')), 'D');

      RETURN NEW;
    END;
    $$ LANGUAGE plpgsql;
    """

    execute """
    CREATE TRIGGER products_search_update
    BEFORE INSERT OR UPDATE ON products
    FOR EACH ROW EXECUTE FUNCTION update_products_search();
    """
  end

  def down do
    execute "DROP TRIGGER IF EXISTS products_search_update ON products"
    execute "DROP FUNCTION IF EXISTS update_products_search()"
    execute "DROP INDEX IF EXISTS products_search_en_idx"
    execute "DROP INDEX IF EXISTS products_search_th_idx"
    alter table(:products) do
      remove :search_en
      remove :search_th
    end
  end
end
```

### 1.2 Search Module พร้อม Pagination และ Filters

```elixir
defmodule MyApp.Search.ProductSearch do
  import Ecto.Query
  alias MyApp.Repo
  alias MyApp.Catalog.Product

  defstruct [:query, :page, :per_page, :filters, :sort, :lang]

  def search(%__MODULE__{} = params) do
    base_query = build_base_query(params)

    %{
      results: fetch_results(base_query, params),
      total: count_results(base_query),
      facets: build_facets(params),
      query: params.query,
      page: params.page,
      per_page: params.per_page
    }
  end

  defp build_base_query(%{query: query, filters: filters, lang: lang}) do
    search_column = if lang == "th", do: :search_th, else: :search_en
    tsquery = build_tsquery(query)
    language = if lang == "th", do: "simple", else: "english"

    base =
      from(p in Product,
        join: c in assoc(p, :category),
        join: b in assoc(p, :brand),
        where: p.active == true
      )

    base =
      if query && String.length(query) > 0 do
        from([p, c, b] in base,
          where: fragment(
            "? @@ to_tsquery(?, ?)",
            field(p, ^search_column),
            ^language,
            ^tsquery
          )
        )
      else
        base
      end

    base
    |> apply_category_filter(filters[:category_ids])
    |> apply_brand_filter(filters[:brand_ids])
    |> apply_price_filter(filters[:min_price], filters[:max_price])
    |> apply_rating_filter(filters[:min_rating])
  end

  defp fetch_results(query, %{query: q, lang: lang, page: page, per_page: per_page, sort: sort}) do
    search_column = if lang == "th", do: :search_th, else: :search_en
    tsquery = build_tsquery(q)
    language = if lang == "th", do: "simple", else: "english"
    offset = (page - 1) * per_page

    query
    |> select_with_ranking(search_column, tsquery, language, q)
    |> apply_sort(sort, search_column, tsquery, language)
    |> limit(^per_page)
    |> offset(^offset)
    |> Repo.all()
  end

  defp select_with_ranking(query, search_column, tsquery, language, raw_query) do
    if raw_query && String.length(raw_query) > 0 do
      from([p, c, b] in query,
        select: %{
          id: p.id,
          name: p.name,
          price: p.price,
          image_url: p.image_url,
          category_name: c.name,
          brand_name: b.name,
          rank: fragment(
            "ts_rank_cd(?, to_tsquery(?, ?), 32)",
            field(p, ^search_column), ^language, ^tsquery
          ),
          headline: fragment(
            "ts_headline(?, to_tsquery(?, ?), 'StartSel=<mark>,StopSel=</mark>,MaxWords=20')",
            p.name, ^language, ^tsquery
          )
        }
      )
    else
      from([p, c, b] in query,
        select: %{
          id: p.id,
          name: p.name,
          price: p.price,
          image_url: p.image_url,
          category_name: c.name,
          brand_name: b.name,
          rank: 0.0,
          headline: p.name
        }
      )
    end
  end

  defp apply_sort(query, "relevance", sc, tsquery, language) do
    from(p in query, order_by: [desc: fragment(
      "ts_rank_cd(?, to_tsquery(?, ?))",
      field(p, ^sc), ^language, ^tsquery
    )])
  end
  defp apply_sort(query, "price_asc", _, _, _), do: from(p in query, order_by: [asc: p.price])
  defp apply_sort(query, "price_desc", _, _, _), do: from(p in query, order_by: [desc: p.price])
  defp apply_sort(query, "newest", _, _, _), do: from(p in query, order_by: [desc: p.inserted_at])
  defp apply_sort(query, "popular", _, _, _), do: from(p in query, order_by: [desc: p.sales_count])
  defp apply_sort(query, _, sc, tsq, lang), do: apply_sort(query, "relevance", sc, tsq, lang)

  defp apply_category_filter(query, nil), do: query
  defp apply_category_filter(query, []), do: query
  defp apply_category_filter(query, ids), do: from([p, c] in query, where: c.id in ^ids)

  defp apply_brand_filter(query, nil), do: query
  defp apply_brand_filter(query, []), do: query
  defp apply_brand_filter(query, ids), do: from([p, c, b] in query, where: b.id in ^ids)

  defp apply_price_filter(query, nil, nil), do: query
  defp apply_price_filter(query, min, nil), do: from(p in query, where: p.price >= ^min)
  defp apply_price_filter(query, nil, max), do: from(p in query, where: p.price <= ^max)
  defp apply_price_filter(query, min, max), do: from(p in query, where: p.price between ^min and ^max)

  defp apply_rating_filter(query, nil), do: query
  defp apply_rating_filter(query, min), do: from(p in query, where: p.avg_rating >= ^min)

  defp count_results(query) do
    from(p in query, select: count(p.id)) |> Repo.one()
  end

  defp build_tsquery(nil), do: ""
  defp build_tsquery(""), do: ""
  defp build_tsquery(query) do
    query
    |> String.split(~r/\s+/)
    |> Enum.filter(&(String.length(&1) >= 2))
    |> Enum.map(&"#{&1}:*")
    |> Enum.join(" & ")
  end

  defp build_facets(%{query: query, filters: filters}) do
    %{
      categories: facet_categories(query, filters),
      brands: facet_brands(query, filters),
      price_ranges: facet_price_ranges()
    }
  end

  defp facet_categories(_query, _filters) do
    from(p in Product,
      join: c in assoc(p, :category),
      where: p.active == true,
      group_by: [c.id, c.name],
      select: %{id: c.id, name: c.name, count: count(p.id)},
      order_by: [desc: count(p.id)],
      limit: 20
    )
    |> Repo.all()
  end

  defp facet_brands(_query, _filters) do
    from(p in Product,
      join: b in assoc(p, :brand),
      where: p.active == true,
      group_by: [b.id, b.name],
      select: %{id: b.id, name: b.name, count: count(p.id)},
      order_by: [desc: count(p.id)],
      limit: 20
    )
    |> Repo.all()
  end

  defp facet_price_ranges do
    [
      %{label: "ต่ำกว่า 500 บาท", min: nil, max: 500},
      %{label: "500 - 1,000 บาท", min: 500, max: 1_000},
      %{label: "1,000 - 5,000 บาท", min: 1_000, max: 5_000},
      %{label: "มากกว่า 5,000 บาท", min: 5_000, max: nil}
    ]
  end
end
```

---

## 2. Elasticsearch ด้วย Elastix

### 2.1 ติดตั้งและตั้งค่า

```elixir
# mix.exs
defp deps do
  [
    {:elastix, "~> 0.10"},
  ]
end
```

```elixir
# config/runtime.exs
config :my_app, :elasticsearch,
  url: System.get_env("ELASTICSEARCH_URL", "http://localhost:9200"),
  index_prefix: System.get_env("ES_INDEX_PREFIX", "myapp")
```

### 2.2 Index Management

```elixir
defmodule MyApp.Search.ElasticsearchIndex do
  @base_url Application.compile_env(:my_app, [:elasticsearch, :url])
  @prefix Application.compile_env(:my_app, [:elasticsearch, :index_prefix])

  def index_name(type), do: "#{@prefix}_#{type}"

  def create_products_index do
    mapping = %{
      mappings: %{
        properties: %{
          id: %{type: "integer"},
          name: %{
            type: "text",
            analyzer: "standard",
            fields: %{keyword: %{type: "keyword"}}
          },
          description: %{type: "text"},
          price: %{type: "float"},
          category: %{
            type: "nested",
            properties: %{
              id: %{type: "integer"},
              name: %{type: "keyword"}
            }
          },
          tags: %{type: "keyword"},
          avg_rating: %{type: "float"},
          sales_count: %{type: "integer"},
          created_at: %{type: "date"}
        }
      },
      settings: %{
        number_of_shards: 1,
        number_of_replicas: 1
      }
    }

    Elastix.Index.create(@base_url, index_name("products"), mapping)
  end

  def index_product(product) do
    doc = %{
      id: product.id,
      name: product.name,
      description: product.description,
      price: Decimal.to_float(product.price),
      category: %{id: product.category.id, name: product.category.name},
      brand: %{id: product.brand.id, name: product.brand.name},
      tags: product.tags,
      avg_rating: product.avg_rating || 0,
      sales_count: product.sales_count || 0,
      active: product.active,
      created_at: DateTime.to_iso8601(product.inserted_at)
    }

    Elastix.Document.index(
      @base_url,
      index_name("products"),
      "_doc",
      product.id,
      doc
    )
  end

  def bulk_index_products(products) do
    lines =
      Enum.flat_map(products, fn product ->
        action = %{index: %{_index: index_name("products"), _id: product.id}}
        doc = %{
          id: product.id,
          name: product.name,
          price: product.price,
          active: product.active
        }
        [action, doc]
      end)

    Elastix.Bulk.post(@base_url, lines, index: index_name("products"))
  end

  def search_products(query, filters \\ %{}, page \\ 1, per_page \\ 20) do
    es_query = %{
      from: (page - 1) * per_page,
      size: per_page,
      query: %{
        bool: %{
          must: build_must_clauses(query),
          filter: build_filter_clauses(filters)
        }
      },
      aggs: %{
        categories: %{terms: %{field: "category.name"}},
        brands: %{terms: %{field: "brand.name"}},
        price_stats: %{stats: %{field: "price"}}
      },
      highlight: %{
        fields: %{
          name: %{},
          description: %{fragment_size: 150}
        }
      }
    }

    Elastix.Search.search(@base_url, index_name("products"), ["_doc"], es_query)
  end

  defp build_must_clauses(nil), do: [%{match_all: %{}}]
  defp build_must_clauses(""), do: [%{match_all: %{}}]
  defp build_must_clauses(query) do
    [%{
      multi_match: %{
        query: query,
        fields: ["name^3", "description^1", "tags^2"],
        type: "best_fields",
        fuzziness: "AUTO"
      }
    }]
  end

  defp build_filter_clauses(filters) when map_size(filters) == 0, do: []
  defp build_filter_clauses(filters) do
    []
    |> maybe_add_filter(:category_ids, filters, fn ids ->
      %{terms: %{"category.id" => ids}}
    end)
    |> maybe_add_filter(:min_rating, filters, fn rating ->
      %{range: %{avg_rating: %{gte: rating}}}
    end)
  end

  defp maybe_add_filter(list, key, filters, build_fn) do
    case Map.get(filters, key) do
      nil -> list
      value -> [build_fn.(value) | list]
    end
  end
end
```

---

## 3. Algolia Integration

### 3.1 ตั้งค่า Algolia

```elixir
# mix.exs
defp deps do
  [
    {:algoliax, "~> 0.7"},
  ]
end
```

```elixir
# config/runtime.exs
config :algoliax,
  application_id: System.get_env("ALGOLIA_APP_ID"),
  api_key: System.get_env("ALGOLIA_ADMIN_API_KEY")
```

### 3.2 Algolia Search Module

```elixir
defmodule MyApp.Search.AlgoliaSearch do
  def search_products(query, opts \\ []) do
    page = Keyword.get(opts, :page, 0)
    per_page = Keyword.get(opts, :per_page, 20)
    filters = Keyword.get(opts, :filters, [])

    params = %{
      hitsPerPage: per_page,
      page: page,
      filters: build_algolia_filters(filters),
      highlightPreTag: "<mark>",
      highlightPostTag: "</mark>"
    }

    case Algoliax.search("products", query, params) do
      {:ok, %{hits: hits, nbHits: total, facets: facets}} ->
        {:ok, %{
          results: format_hits(hits),
          total: total,
          facets: facets
        }}

      {:error, reason} ->
        {:error, reason}
    end
  end

  defp format_hits(hits) do
    Enum.map(hits, fn hit ->
      %{
        id: hit["objectID"],
        name: get_highlighted(hit, "name"),
        price: hit["price"],
        image_url: hit["imageUrl"],
        url: hit["url"],
        category: hit["categoryName"],
        rating: hit["avgRating"]
      }
    end)
  end

  defp get_highlighted(hit, field) do
    get_in(hit, ["_highlightResult", field, "value"]) || hit[field]
  end

  defp build_algolia_filters([]), do: ""
  defp build_algolia_filters(filters) do
    filters
    |> Enum.map(fn
      {:category, name} -> "categoryName:\"#{name}\""
      {:min_price, price} -> "price >= #{price}"
      {:max_price, price} -> "price <= #{price}"
    end)
    |> Enum.join(" AND ")
  end

  # Configure index settings
  def configure_index do
    Algoliax.Index.set_settings("products", %{
      searchableAttributes: ["name", "unordered(description)", "categoryName", "tags"],
      attributesForFaceting: ["filterOnly(categoryName)", "filterOnly(brandName)"],
      customRanking: ["desc(salesCount)", "desc(avgRating)"],
      highlightPreTag: "<mark>",
      highlightPostTag: "</mark>"
    })
  end
end
```

---

## 4. Auto-suggestions (Typeahead)

### 4.1 Suggestion API

```elixir
defmodule MyAppWeb.SearchSuggestionController do
  use MyAppWeb, :controller
  alias MyApp.Search.Suggestions

  def index(conn, %{"q" => query}) when byte_size(query) >= 2 do
    suggestions = Suggestions.get_suggestions(query, limit: 8)
    json(conn, %{suggestions: suggestions})
  end

  def index(conn, _params), do: json(conn, %{suggestions: []})
end

defmodule MyApp.Search.Suggestions do
  import Ecto.Query
  alias MyApp.Repo
  alias MyApp.Catalog.Product

  def get_suggestions(query, opts \\ []) do
    limit = Keyword.get(opts, :limit, 8)

    product_names = suggest_from_products(query, div(limit, 2))
    search_history = suggest_from_history(query, div(limit, 2))

    (product_names ++ search_history)
    |> Enum.uniq_by(& &1.text)
    |> Enum.take(limit)
  end

  defp suggest_from_products(query, limit) do
    from(p in Product,
      where: ilike(p.name, ^"#{query}%") and p.active == true,
      select: %{text: p.name, type: "product", count: p.sales_count},
      order_by: [desc: p.sales_count],
      limit: ^limit
    )
    |> Repo.all()
  end

  defp suggest_from_history(query, limit) do
    from(s in "search_queries",
      where: ilike(s.query, ^"#{query}%"),
      group_by: s.query,
      select: %{text: s.query, type: "history", count: count(s.id)},
      order_by: [desc: count(s.id)],
      limit: ^limit
    )
    |> Repo.all()
  end
end
```

### 4.2 Frontend Typeahead Hook

```javascript
// assets/js/hooks/typeahead.js
export const Typeahead = {
  mounted() {
    this.input = this.el.querySelector('input[type="search"]');
    this.dropdown = this.el.querySelector('[data-dropdown]');
    this.debounceTimer = null;
    this.selectedIndex = -1;

    this.input.addEventListener('input', (e) => {
      clearTimeout(this.debounceTimer);
      const query = e.target.value.trim();

      if (query.length < 2) {
        this.hideDropdown();
        return;
      }

      this.debounceTimer = setTimeout(() => {
        this.fetchSuggestions(query);
      }, 200);
    });

    this.input.addEventListener('keydown', (e) => {
      this.handleKeyNav(e);
    });

    document.addEventListener('click', (e) => {
      if (!this.el.contains(e.target)) this.hideDropdown();
    });
  },

  async fetchSuggestions(query) {
    try {
      const res = await fetch(`/api/search/suggestions?q=${encodeURIComponent(query)}`);
      const { suggestions } = await res.json();
      this.showSuggestions(suggestions, query);
    } catch (err) {
      console.error('Suggestion error:', err);
    }
  },

  showSuggestions(suggestions, query) {
    if (!suggestions.length) { this.hideDropdown(); return; }

    this.dropdown.innerHTML = suggestions.map((s, i) =>
      `<div class="suggestion-item px-4 py-2 hover:bg-gray-100 cursor-pointer"
            data-index="${i}" data-value="${s.text}">
        <span>${this.highlight(s.text, query)}</span>
        <span class="text-xs text-gray-400 float-right">${s.type}</span>
      </div>`
    ).join('');

    this.dropdown.querySelectorAll('.suggestion-item').forEach(item => {
      item.addEventListener('click', () => {
        this.input.value = item.dataset.value;
        this.hideDropdown();
        this.el.closest('form').submit();
      });
    });

    this.dropdown.classList.remove('hidden');
  },

  highlight(text, query) {
    return text.replace(new RegExp(`(${query})`, 'gi'), '<strong>$1</strong>');
  },

  handleKeyNav(e) {
    const items = this.dropdown.querySelectorAll('.suggestion-item');
    if (!items.length) return;

    if (e.key === 'ArrowDown') {
      e.preventDefault();
      this.selectedIndex = Math.min(this.selectedIndex + 1, items.length - 1);
    } else if (e.key === 'ArrowUp') {
      e.preventDefault();
      this.selectedIndex = Math.max(this.selectedIndex - 1, -1);
    } else if (e.key === 'Enter' && this.selectedIndex >= 0) {
      e.preventDefault();
      this.input.value = items[this.selectedIndex].dataset.value;
      this.hideDropdown();
      return;
    } else if (e.key === 'Escape') {
      this.hideDropdown();
    }

    items.forEach((item, i) =>
      item.classList.toggle('bg-gray-100', i === this.selectedIndex)
    );
  },

  hideDropdown() {
    this.dropdown.classList.add('hidden');
    this.selectedIndex = -1;
  },

  destroyed() { clearTimeout(this.debounceTimer); }
};
```

---

## 5. Faceted Search LiveView

```elixir
defmodule MyAppWeb.SearchLive do
  use MyAppWeb, :live_view
  alias MyApp.Search.ProductSearch

  def mount(params, _session, socket) do
    results = perform_search(params)
    {:ok, assign(socket,
      query: params["q"] || "",
      results: results.results,
      total: results.total,
      facets: results.facets,
      filters: %{},
      sort: params["sort"] || "relevance",
      page: 1,
      per_page: 20
    )}
  end

  def handle_event("filter_toggle", %{"type" => type, "id" => id}, socket) do
    id = String.to_integer(id)
    key = String.to_atom(type <> "_ids")

    updated_filters =
      Map.update(socket.assigns.filters, key, [id], fn ids ->
        if id in ids, do: List.delete(ids, id), else: [id | ids]
      end)

    results = search_with(socket, filters: updated_filters, page: 1)

    {:noreply, socket
      |> assign(:filters, updated_filters)
      |> assign(:results, results.results)
      |> assign(:total, results.total)
      |> assign(:page, 1)}
  end

  def handle_event("sort_change", %{"sort" => sort}, socket) do
    results = search_with(socket, sort: sort, page: 1)
    {:noreply, assign(socket, sort: sort, results: results.results, page: 1)}
  end

  def handle_event("load_more", _params, socket) do
    next_page = socket.assigns.page + 1
    results = search_with(socket, page: next_page)
    {:noreply, socket
      |> update(:results, &(&1 ++ results.results))
      |> assign(:page, next_page)}
  end

  defp perform_search(params) do
    %ProductSearch{
      query: params["q"],
      page: String.to_integer(params["page"] || "1"),
      per_page: 20,
      filters: %{},
      sort: params["sort"] || "relevance",
      lang: params["lang"] || "th"
    }
    |> ProductSearch.search()
  end

  defp search_with(socket, overrides) do
    %ProductSearch{
      query: socket.assigns.query,
      page: Keyword.get(overrides, :page, socket.assigns.page),
      per_page: socket.assigns.per_page,
      filters: Keyword.get(overrides, :filters, socket.assigns.filters),
      sort: Keyword.get(overrides, :sort, socket.assigns.sort),
      lang: "th"
    }
    |> ProductSearch.search()
  end

  def render(assigns) do
    ~H"""
    <div class="max-w-7xl mx-auto px-4 py-6">
      <div class="flex gap-6">
        <!-- Facet sidebar -->
        <aside class="w-64 flex-shrink-0">
          <div class="bg-white rounded-lg shadow p-4">
            <h3 class="font-semibold mb-3">หมวดหมู่</h3>
            <%= for cat <- @facets.categories do %>
              <label class="flex items-center gap-2 py-1 cursor-pointer">
                <input
                  type="checkbox"
                  checked={cat.id in (@filters[:category_ids] || [])}
                  phx-click="filter_toggle"
                  phx-value-type="category"
                  phx-value-id={cat.id}
                />
                <span class="text-sm"><%= cat.name %></span>
                <span class="text-xs text-gray-400 ml-auto">(<%= cat.count %>)</span>
              </label>
            <% end %>
          </div>
        </aside>

        <!-- Results -->
        <main class="flex-1">
          <div class="flex justify-between items-center mb-4">
            <p class="text-gray-600">พบ <%= @total %> รายการ</p>
            <select phx-change="sort_change" name="sort" class="select select-sm select-bordered">
              <option value="relevance" selected={@sort == "relevance"}>เกี่ยวข้องมากสุด</option>
              <option value="price_asc" selected={@sort == "price_asc"}>ราคาต่ำ-สูง</option>
              <option value="price_desc" selected={@sort == "price_desc"}>ราคาสูง-ต่ำ</option>
              <option value="popular" selected={@sort == "popular"}>ขายดีสุด</option>
            </select>
          </div>

          <div class="grid grid-cols-3 gap-4">
            <%= for product <- @results do %>
              <div class="bg-white rounded-lg shadow p-3">
                <p class="font-medium text-sm" innerHTML={product.headline}></p>
                <p class="text-blue-600 font-bold mt-1">฿<%= product.price %></p>
              </div>
            <% end %>
          </div>

          <%= if length(@results) < @total do %>
            <div class="text-center mt-6">
              <button phx-click="load_more" class="btn btn-outline">โหลดเพิ่ม</button>
            </div>
          <% end %>
        </main>
      </div>
    </div>
    """
  end
end
```

---

## 6. Search Analytics

### 6.1 บันทึก Search Events

```elixir
defmodule MyApp.Search.Analytics do
  alias MyApp.Repo
  import Ecto.Query

  def track_search(user_id, query, result_count, filters \\ %{}) do
    Repo.insert_all("search_queries", [%{
      user_id: user_id,
      query: query,
      result_count: result_count,
      filters: Jason.encode!(filters),
      inserted_at: DateTime.utc_now() |> DateTime.truncate(:second),
      updated_at: DateTime.utc_now() |> DateTime.truncate(:second)
    }])
  end

  def track_click(user_id, query, product_id, position) do
    Repo.insert_all("search_clicks", [%{
      user_id: user_id,
      query: query,
      product_id: product_id,
      position: position,
      inserted_at: DateTime.utc_now() |> DateTime.truncate(:second)
    }])
  end

  def popular_searches(limit \\ 10) do
    since = DateTime.add(DateTime.utc_now(), -7 * 86400)
    from(s in "search_queries",
      where: s.inserted_at >= ^since and s.result_count > 0,
      group_by: s.query,
      select: %{query: s.query, count: count(s.id)},
      order_by: [desc: count(s.id)],
      limit: ^limit
    )
    |> Repo.all()
  end

  def zero_result_searches(limit \\ 20) do
    since = DateTime.add(DateTime.utc_now(), -7 * 86400)
    from(s in "search_queries",
      where: s.result_count == 0 and s.inserted_at >= ^since,
      group_by: s.query,
      select: %{query: s.query, count: count(s.id)},
      order_by: [desc: count(s.id)],
      limit: ^limit
    )
    |> Repo.all()
  end

  def click_through_rate(query) do
    searches = from(s in "search_queries",
      where: s.query == ^query, select: count(s.id)) |> Repo.one()

    clicks = from(c in "search_clicks",
      where: c.query == ^query, select: count(c.id)) |> Repo.one()

    if searches > 0, do: Float.round(clicks / searches * 100, 1), else: 0.0
  end

  def avg_position_for_query(query) do
    from(c in "search_clicks",
      where: c.query == ^query,
      select: avg(c.position)
    )
    |> Repo.one()
  end
end
```

### 6.2 Analytics Dashboard

```elixir
defmodule MyAppWeb.Admin.SearchAnalyticsLive do
  use MyAppWeb, :live_view
  alias MyApp.Search.Analytics

  def mount(_params, _session, socket) do
    {:ok, assign(socket,
      popular: Analytics.popular_searches(10),
      zero_results: Analytics.zero_result_searches(10)
    )}
  end

  def render(assigns) do
    ~H"""
    <div class="grid grid-cols-2 gap-6 p-6">
      <div class="bg-white rounded-lg shadow p-6">
        <h2 class="text-lg font-semibold mb-4">คำค้นหายอดนิยม (7 วันล่าสุด)</h2>
        <ol class="space-y-2">
          <%= for {item, i} <- Enum.with_index(@popular, 1) do %>
            <li class="flex justify-between items-center">
              <span class="flex items-center gap-2">
                <span class="text-gray-400 text-sm w-6"><%= i %>.</span>
                <span><%= item.query %></span>
              </span>
              <span class="text-sm text-gray-500 font-medium"><%= item.count %> ครั้ง</span>
            </li>
          <% end %>
        </ol>
      </div>

      <div class="bg-white rounded-lg shadow p-6">
        <h2 class="text-lg font-semibold mb-4 text-red-600">คำค้นที่ไม่มีผลลัพธ์</h2>
        <ol class="space-y-2">
          <%= for item <- @zero_results do %>
            <li class="flex justify-between items-center">
              <span class="text-red-500"><%= item.query %></span>
              <span class="text-sm text-gray-500"><%= item.count %> ครั้ง</span>
            </li>
          <% end %>
        </ol>
        <p class="text-xs text-gray-400 mt-4">
          ใช้รายการนี้เพิ่ม synonyms หรือ content ที่ขาดหายไป
        </p>
      </div>
    </div>
    """
  end
end
```

---

## สรุป

```
Search System Options Comparison
═════════════════════════════════════════════════════════
Option 1: PostgreSQL Full-Text Search
  ✓ ไม่ต้องใช้ external service
  ✓ ACID compliant (real-time consistency)
  ✓ เหมาะกับข้อมูลไม่เกิน 10M rows
  ✗ Thai tokenizer ต้อง config เพิ่ม
  ✗ ขาด fuzzy search, synonyms

Option 2: Elasticsearch
  ✓ Powerful aggregations
  ✓ Horizontal scaling
  ✓ Rich query DSL, fuzzy matching
  ✗ ต้อง manage cluster
  ✗ Eventual consistency

Option 3: Algolia
  ✓ Managed service, ตั้งง่าย
  ✓ Best relevance out-of-the-box
  ✓ Built-in analytics dashboard
  ✗ Cost per operation
  ✗ Data stored externally

Search Pipeline
  User Input
    → Typeahead suggestions (debounce 200ms)
    → Search query → Full-text match
    → Ranking (ts_rank / BM25 / Algolia)
    → Facet aggregations
    → Result highlighting
    → Analytics tracking (async)

Performance Tips
  - GIN index บน tsvector
  - Paginate: 20 items per page
  - Cache popular queries (Redis / ETS)
  - Track zero-results → improve content
═════════════════════════════════════════════════════════
```

---

*ก่อนหน้า: [Part 65 - Real-time Notifications System](part_65.md) | ต่อไป: [Part 67 - Internationalization (i18n)](part_67.md)*
