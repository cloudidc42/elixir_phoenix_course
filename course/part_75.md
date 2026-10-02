# Part 75: Real-time Analytics (การวิเคราะห์ข้อมูลแบบ Real-time)

## เป้าหมายการเรียนรู้

หลังจากจบ Part นี้ คุณจะสามารถ:
- สร้าง event tracking system
- Real-time dashboard ด้วย LiveView
- Time-series data ด้วย PostgreSQL
- Charts ด้วย VegaLite/Chart.js

---

## 1. Event Tracking

```elixir
defmodule MyApp.Analytics do
  alias MyApp.Repo
  alias MyApp.Analytics.Event
  import Ecto.Query

  def track(event_name, properties \\ %{}, opts \\ []) do
    %Event{}
    |> Event.changeset(%{
      name: event_name,
      properties: properties,
      session_id: opts[:session_id],
      user_id: opts[:user_id],
      page_url: opts[:page_url],
      referrer: opts[:referrer],
      user_agent: opts[:user_agent],
      ip_address: opts[:ip_address]
    })
    |> Repo.insert(returning: false)  # no return for performance
  end

  # Batch insert for high-volume tracking
  def track_batch(events) do
    now = DateTime.utc_now() |> DateTime.truncate(:second)
    entries = Enum.map(events, fn event ->
      Map.merge(event, %{inserted_at: now})
    end)

    Repo.insert_all(Event, entries)
  end

  def page_views(opts \\ []) do
    start_date = Keyword.get(opts, :start_date, Date.add(Date.utc_today(), -30))
    end_date = Keyword.get(opts, :end_date, Date.utc_today())

    from(e in Event,
      where: e.name == "page_view"
        and fragment("?::date", e.inserted_at) >= ^start_date
        and fragment("?::date", e.inserted_at) <= ^end_date,
      group_by: fragment("?::date", e.inserted_at),
      order_by: fragment("?::date", e.inserted_at),
      select: {
        fragment("?::date", e.inserted_at),
        count(e.id),
        count(e.session_id, :distinct)
      }
    )
    |> Repo.all()
    |> Enum.map(fn {date, views, sessions} ->
      %{date: date, views: views, sessions: sessions}
    end)
  end

  def top_pages(limit \\ 10) do
    from(e in Event,
      where: e.name == "page_view",
      group_by: e.properties["url"],
      order_by: [desc: count(e.id)],
      limit: ^limit,
      select: {e.properties["url"], count(e.id)}
    )
    |> Repo.all()
    |> Enum.map(fn {url, count} -> %{url: url, views: count} end)
  end

  def conversion_funnel(steps) do
    # สร้าง funnel analysis
    Enum.map(steps, fn {step_name, event_name} ->
      count = Repo.one(
        from(e in Event,
          where: e.name == ^event_name,
          select: count(e.session_id, :distinct)
        )
      )
      {step_name, count}
    end)
  end
end
```

---

## 2. Real-time Dashboard

```elixir
defmodule MyAppWeb.AnalyticsDashboardLive do
  use MyAppWeb, :live_view

  alias MyApp.Analytics

  @refresh_interval 5_000  # 5 seconds

  def mount(_params, _session, socket) do
    if connected?(socket) do
      :timer.send_interval(@refresh_interval, self(), :refresh)
      # Subscribe to real-time events
      Phoenix.PubSub.subscribe(MyApp.PubSub, "analytics:events")
    end

    {:ok, assign(socket,
      page_views_today: Analytics.count_today(:page_view),
      active_users: Analytics.count_active_users(5),  # last 5 minutes
      top_pages: Analytics.top_pages(5),
      hourly_data: Analytics.hourly_breakdown(Date.utc_today()),
      events_per_minute: 0
    )}
  end

  def handle_info(:refresh, socket) do
    {:noreply, assign(socket,
      page_views_today: Analytics.count_today(:page_view),
      active_users: Analytics.count_active_users(5),
      top_pages: Analytics.top_pages(5)
    )}
  end

  def handle_info({:new_event, event}, socket) do
    # Increment counter from real-time event
    {:noreply, update(socket, :events_per_minute, &(&1 + 1))}
  end

  def render(assigns) do
    ~H"""
    <div class="p-6">
      <h1 class="text-2xl font-bold mb-6">Analytics Dashboard</h1>

      <!-- Real-time counters -->
      <div class="grid grid-cols-3 gap-4 mb-8">
        <div class="bg-white p-4 rounded-lg shadow text-center">
          <div class="text-4xl font-bold text-blue-600"><%= @page_views_today %></div>
          <div class="text-gray-500 mt-1">Page Views Today</div>
        </div>
        <div class="bg-white p-4 rounded-lg shadow text-center">
          <div class="text-4xl font-bold text-green-600">
            <span class="animate-pulse">●</span> <%= @active_users %>
          </div>
          <div class="text-gray-500 mt-1">Active Users (5min)</div>
        </div>
        <div class="bg-white p-4 rounded-lg shadow text-center">
          <div class="text-4xl font-bold text-purple-600"><%= @events_per_minute %></div>
          <div class="text-gray-500 mt-1">Events/Min</div>
        </div>
      </div>

      <!-- Chart -->
      <div class="bg-white p-4 rounded-lg shadow mb-6">
        <h3 class="font-semibold mb-4">Hourly Traffic</h3>
        <div id="hourly-chart"
          phx-hook="BarChart"
          data-labels={Jason.encode!(Enum.map(@hourly_data, & &1.hour))}
          data-values={Jason.encode!(Enum.map(@hourly_data, & &1.views))}>
        </div>
      </div>

      <!-- Top Pages -->
      <div class="bg-white p-4 rounded-lg shadow">
        <h3 class="font-semibold mb-4">Top Pages</h3>
        <table class="w-full">
          <%= for {page, index} <- Enum.with_index(@top_pages, 1) do %>
            <tr class="border-b last:border-0">
              <td class="py-2 text-gray-500 w-8"><%= index %></td>
              <td class="py-2 truncate"><%= page.url %></td>
              <td class="py-2 text-right font-mono"><%= page.views %></td>
            </tr>
          <% end %>
        </table>
      </div>
    </div>
    """
  end
end
```

---

## 3. Chart.js Integration

```javascript
// assets/js/hooks/bar_chart.js
import Chart from 'chart.js/auto'

const BarChart = {
  mounted() {
    this.chart = new Chart(this.el, {
      type: 'bar',
      data: {
        labels: JSON.parse(this.el.dataset.labels),
        datasets: [{
          label: 'Page Views',
          data: JSON.parse(this.el.dataset.values),
          backgroundColor: 'rgba(59, 130, 246, 0.5)',
          borderColor: 'rgb(59, 130, 246)',
          borderWidth: 1
        }]
      },
      options: {
        responsive: true,
        scales: {
          y: { beginAtZero: true }
        }
      }
    })
  },

  updated() {
    this.chart.data.labels = JSON.parse(this.el.dataset.labels)
    this.chart.data.datasets[0].data = JSON.parse(this.el.dataset.values)
    this.chart.update()
  },

  destroyed() {
    this.chart.destroy()
  }
}

export default BarChart
```

---

## 4. Cohort Analysis

```elixir
defmodule MyApp.Analytics.Cohort do
  alias MyApp.Repo
  import Ecto.Query

  def retention_by_week(cohort_weeks \\ 12) do
    # สร้าง cohort analysis: ผู้ใช้ที่สมัครสัปดาห์ไหน ยังใช้งานอยู่ไหม
    today = Date.utc_today()

    Enum.map(0..(cohort_weeks - 1), fn week_offset ->
      cohort_start = Date.add(today, -(week_offset + 1) * 7)
      cohort_end = Date.add(today, -week_offset * 7)

      cohort_users = get_cohort_users(cohort_start, cohort_end)

      retention = Enum.map(0..week_offset, fn retention_week ->
        retention_start = Date.add(cohort_start, retention_week * 7)
        retention_end = Date.add(retention_start, 7)

        active = count_active_in_period(cohort_users, retention_start, retention_end)
        total = length(cohort_users)

        if total > 0, do: Float.round(active / total * 100, 1), else: 0
      end)

      %{
        cohort_date: cohort_start,
        cohort_size: length(cohort_users),
        retention: retention
      }
    end)
  end

  defp get_cohort_users(start_date, end_date) do
    from(u in MyApp.User,
      where: u.inserted_at >= ^DateTime.new!(start_date, ~T[00:00:00], "Etc/UTC")
        and u.inserted_at < ^DateTime.new!(end_date, ~T[00:00:00], "Etc/UTC"),
      select: u.id
    )
    |> Repo.all()
  end

  defp count_active_in_period(user_ids, start_date, end_date) do
    from(e in MyApp.Analytics.Event,
      where: e.user_id in ^user_ids
        and e.inserted_at >= ^DateTime.new!(start_date, ~T[00:00:00], "Etc/UTC")
        and e.inserted_at < ^DateTime.new!(end_date, ~T[00:00:00], "Etc/UTC"),
      select: count(e.user_id, :distinct)
    )
    |> Repo.one()
  end
end
```

---

## 5. Analytics Tracking Plug

```elixir
defmodule MyAppWeb.Plugs.TrackPageView do
  import Plug.Conn
  alias MyApp.Analytics

  def init(opts), do: opts

  def call(conn, _opts) do
    Plug.Conn.register_before_send(conn, fn conn ->
      if conn.status in [200, 301, 302] do
        Task.start(fn ->
          Analytics.track("page_view", %{
            url: conn.request_path,
            status: conn.status
          }, [
            session_id: get_session_id(conn),
            user_id: conn.assigns[:current_user]?.id,
            referrer: get_req_header(conn, "referer") |> List.first(),
            user_agent: get_req_header(conn, "user-agent") |> List.first(),
            ip_address: to_string(:inet.ntoa(conn.remote_ip))
          ])
        end)
      end
      conn
    end)
  end

  defp get_session_id(conn) do
    case get_session(conn, :session_id) do
      nil ->
        id = Ecto.UUID.generate()
        put_session(conn, :session_id, id)
        id
      id -> id
    end
  end
end
```

---

## สรุป

```
Analytics System:
├── Event tracking: page_view, click, conversion
├── Real-time dashboard: LiveView + PubSub
├── Charts: Chart.js via JS hooks
└── Cohort analysis: retention metrics

Key Metrics:
├── Page views, unique visitors
├── Active users (last N minutes)
├── Conversion funnel
└── Retention cohort

Performance:
├── Async tracking (Task.start)
├── Batch inserts for high volume
└── Aggregate queries for reports
```

---

*ก่อนหน้า: [Part 74](part_74.md) | ต่อไป: [Part 76 - Testing Strategies](part_76.md)*
