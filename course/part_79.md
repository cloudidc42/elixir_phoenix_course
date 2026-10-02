# Part 79: Advanced Ecto Queries (Query ขั้นสูงด้วย Ecto)

## เป้าหมายการเรียนรู้

หลังจากจบ Part นี้ คุณจะสามารถ:
- Window functions ใน Ecto
- Recursive CTEs
- Dynamic query building
- Ecto.Repo ที่กำหนดเอง

---

## 1. Window Functions

```elixir
defmodule MyApp.Analytics.Queries do
  import Ecto.Query

  # Running total ด้วย window function
  def running_totals(user_id) do
    from(o in MyApp.Order,
      where: o.user_id == ^user_id,
      order_by: [asc: o.inserted_at],
      select: %{
        id: o.id,
        total: o.total,
        date: o.inserted_at,
        running_total: fragment(
          "SUM(?) OVER (ORDER BY ? ROWS UNBOUNDED PRECEDING)",
          o.total, o.inserted_at
        )
      }
    )
  end

  # Rank products by revenue per category
  def rank_by_category do
    from(p in MyApp.Product,
      join: o in assoc(p, :order_items),
      group_by: [p.id, p.category_id, p.name],
      select: %{
        id: p.id,
        name: p.name,
        category_id: p.category_id,
        revenue: sum(o.subtotal),
        rank_in_category: fragment(
          "RANK() OVER (PARTITION BY ? ORDER BY SUM(?) DESC)",
          p.category_id, o.subtotal
        )
      }
    )
  end

  # Lag/Lead: compare to previous record
  def month_over_month_growth do
    from(r in MyApp.Revenue,
      order_by: [asc: r.month],
      select: %{
        month: r.month,
        revenue: r.amount,
        prev_month: fragment(
          "LAG(?, 1) OVER (ORDER BY ?)",
          r.amount, r.month
        ),
        growth_pct: fragment(
          "ROUND(100.0 * (? - LAG(?, 1) OVER (ORDER BY ?)) / NULLIF(LAG(?, 1) OVER (ORDER BY ?), 0), 2)",
          r.amount, r.amount, r.month, r.amount, r.month
        )
      }
    )
  end
end
```

---

## 2. Common Table Expressions (CTE)

```elixir
defmodule MyApp.Blog.Queries do
  import Ecto.Query

  # Category tree ด้วย Recursive CTE
  def get_category_tree(root_id) do
    from(c in MyApp.Category,
      where: c.id == ^root_id or fragment(
        "? IN (WITH RECURSIVE tree AS (
          SELECT id FROM categories WHERE id = ?
          UNION ALL
          SELECT c.id FROM categories c
          JOIN tree t ON c.parent_id = t.id
        ) SELECT id FROM tree)",
        c.id, ^root_id
      )
    )
    |> MyApp.Repo.all()
  end

  # CTE สำหรับ complex query
  def articles_with_stats do
    comment_counts =
      from(c in MyApp.Comment,
        group_by: c.article_id,
        select: %{article_id: c.article_id, count: count(c.id)}
      )

    view_counts =
      from(v in MyApp.PageView,
        where: v.resource_type == "article",
        group_by: v.resource_id,
        select: %{resource_id: v.resource_id, views: count(v.id)}
      )

    from(a in MyApp.Article,
      left_join: cc in subquery(comment_counts), on: a.id == cc.article_id,
      left_join: vc in subquery(view_counts), on: a.id == vc.resource_id,
      where: a.status == "published",
      order_by: [desc: coalesce(vc.views, 0)],
      select: %{
        id: a.id,
        title: a.title,
        comments: coalesce(cc.count, 0),
        views: coalesce(vc.views, 0)
      }
    )
  end
end
```

---

## 3. Dynamic Query Building

```elixir
defmodule MyApp.Search.DynamicQuery do
  import Ecto.Query

  def build_search(base_query, filters) do
    Enum.reduce(filters, base_query, fn
      {:keyword, nil}, q -> q
      {:keyword, ""}, q -> q
      {:keyword, keyword}, q ->
        search_term = "%#{keyword}%"
        where(q, [r], ilike(r.title, ^search_term) or ilike(r.body, ^search_term))

      {:status, nil}, q -> q
      {:status, statuses}, q when is_list(statuses) ->
        where(q, [r], r.status in ^statuses)
      {:status, status}, q ->
        where(q, [r], r.status == ^status)

      {:author_id, nil}, q -> q
      {:author_id, id}, q -> where(q, [r], r.author_id == ^id)

      {:date_from, nil}, q -> q
      {:date_from, date}, q ->
        where(q, [r], fragment("?::date", r.inserted_at) >= ^date)

      {:date_to, nil}, q -> q
      {:date_to, date}, q ->
        where(q, [r], fragment("?::date", r.inserted_at) <= ^date)

      {:sort_by, {field, direction}}, q ->
        order_by(q, [r], [{^direction, field(r, ^field)}])

      _, q -> q
    end)
  end
end

# Usage:
def search_articles(filters) do
  from(a in MyApp.Article, preload: [:author])
  |> MyApp.Search.DynamicQuery.build_search(filters)
  |> MyApp.Repo.all()
end
```

---

## 4. Custom Ecto Repo Functions

```elixir
defmodule MyApp.Repo do
  use Ecto.Repo, otp_app: :my_app, adapter: Ecto.Adapters.Postgres

  # Paginate query
  def paginate(query, page, per_page \\ 20) do
    offset = (page - 1) * per_page

    total_count = one(from q in subquery(query), select: count(q.id))
    items = all(from q in query, limit: ^per_page, offset: ^offset)

    total_pages = ceil(total_count / per_page)

    %{
      items: items,
      page: page,
      per_page: per_page,
      total_count: total_count,
      total_pages: total_pages,
      has_next: page < total_pages,
      has_prev: page > 1
    }
  end

  # Find or create
  def find_or_create(queryable, attrs, unique_fields) do
    changeset = queryable.__struct__
    |> queryable.changeset(attrs)

    case get_by(queryable, Map.take(attrs, unique_fields)) do
      nil -> insert(changeset)
      existing -> {:ok, existing}
    end
  end

  # Exists?
  def exists?(query) do
    one(from q in query, select: fragment("1"), limit: 1) != nil
  end
end
```

---

## 5. Raw SQL with Typed Results

```elixir
defmodule MyApp.Analytics.RawQueries do
  alias MyApp.Repo

  def get_cohort_retention do
    sql = """
    WITH cohorts AS (
      SELECT
        DATE_TRUNC('month', created_at) AS cohort_month,
        id AS user_id
      FROM users
    ),
    activities AS (
      SELECT DISTINCT
        user_id,
        DATE_TRUNC('month', inserted_at) AS activity_month
      FROM events
    )
    SELECT
      c.cohort_month,
      COUNT(DISTINCT c.user_id) AS cohort_size,
      a.activity_month,
      COUNT(DISTINCT a.user_id) AS retained_users,
      ROUND(
        100.0 * COUNT(DISTINCT a.user_id) / COUNT(DISTINCT c.user_id),
        2
      ) AS retention_rate
    FROM cohorts c
    LEFT JOIN activities a ON c.user_id = a.user_id
      AND a.activity_month >= c.cohort_month
    GROUP BY c.cohort_month, a.activity_month
    ORDER BY c.cohort_month, a.activity_month
    """

    {:ok, %{rows: rows, columns: columns}} = Ecto.Adapters.SQL.query(Repo, sql, [])

    columns = Enum.map(columns, &String.to_atom/1)
    Enum.map(rows, fn row ->
      Enum.zip(columns, row) |> Map.new()
    end)
  end
end
```

---

## สรุป

```
Advanced Ecto:
├── Window functions: SUM/RANK/LAG OVER (...)
├── CTEs: WITH clause for complex queries
├── Dynamic queries: Enum.reduce over filters
└── Custom Repo: paginate, exists?, find_or_create

Window Functions:
├── SUM OVER: running totals
├── RANK/ROW_NUMBER OVER (PARTITION BY)
├── LAG/LEAD: access adjacent rows
└── FIRST_VALUE/LAST_VALUE

Dynamic Filtering:
├── Enum.reduce accumulates conditions
├── Pattern match on nil to skip
└── Dynamic.field/1 for dynamic field names
```

---

*ก่อนหน้า: [Part 78](part_78.md) | ต่อไป: [Part 80 - Refactoring Patterns](part_80.md)*
