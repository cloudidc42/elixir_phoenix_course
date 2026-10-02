# Part 87: Production Monitoring (การ Monitor Production)

## เป้าหมายการเรียนรู้

หลังจากจบ Part นี้ คุณจะสามารถ:
- Error tracking ด้วย Sentry
- Application Performance Monitoring (APM)
- Log aggregation ด้วย structured logging
- Alerting และ on-call workflows

---

## 1. Sentry Integration

```elixir
# mix.exs: {:sentry, "~> 10.7"}, {:jason, "~> 1.4"}

# config/config.exs
config :sentry,
  dsn: System.get_env("SENTRY_DSN"),
  environment_name: Mix.env(),
  enable_source_code_context: true,
  root_source_code_paths: [File.cwd!()]

# endpoint.ex
defmodule MyAppWeb.Endpoint do
  use Phoenix.Endpoint, otp_app: :my_app
  use Sentry.PlugCapture  # capture unhandled errors

  plug Sentry.PlugContext  # add request context
  # ...
end

# Capture errors manually
def create_order(attrs) do
  case MyApp.Orders.create(attrs) do
    {:ok, order} -> {:ok, order}
    {:error, reason} = error ->
      Sentry.capture_message("Order creation failed",
        extra: %{attrs: attrs, reason: inspect(reason)},
        level: :error
      )
      error
  end
rescue
  exception ->
    Sentry.capture_exception(exception,
      stacktrace: __STACKTRACE__,
      extra: %{attrs: attrs}
    )
    reraise exception, __STACKTRACE__
end
```

---

## 2. Structured Logging

```elixir
# config/config.exs
config :logger,
  backends: [:console],
  level: :info

config :logger, :console,
  format: {MyApp.Logger.Formatter, :format},
  metadata: [:request_id, :user_id, :trace_id]

defmodule MyApp.Logger.Formatter do
  def format(level, message, timestamp, metadata) do
    log = %{
      level: level,
      message: to_string(message),
      timestamp: format_timestamp(timestamp),
      request_id: metadata[:request_id],
      user_id: metadata[:user_id],
      trace_id: metadata[:trace_id]
    }
    |> Map.reject(fn {_, v} -> is_nil(v) end)

    Jason.encode!(log) <> "\n"
  rescue
    _ -> "#{level}: #{message}\n"
  end

  defp format_timestamp({date, time}) do
    "#{:calendar.date_to_gregorian_days(date)}T#{format_time(time)}Z"
  end

  defp format_time({h, m, s, _ms}) do
    "#{pad(h)}:#{pad(m)}:#{pad(s)}"
  end

  defp pad(n), do: String.pad_leading(to_string(n), 2, "0")
end

# Plug to add metadata
defmodule MyAppWeb.Plugs.LoggerMetadata do
  import Plug.Conn

  def init(opts), do: opts

  def call(conn, _opts) do
    request_id = conn.assigns[:request_id] || Ecto.UUID.generate()

    Logger.metadata(
      request_id: request_id,
      user_id: conn.assigns[:current_user]?.id,
      method: conn.method,
      path: conn.request_path
    )

    conn
  end
end
```

---

## 3. PromEx Metrics

```elixir
# mix.exs: {:prom_ex, "~> 1.10"}

defmodule MyApp.PromEx do
  use PromEx, otp_app: :my_app

  @impl true
  def plugins do
    [
      PromEx.Plugins.Application,
      PromEx.Plugins.Beam,
      PromEx.Plugins.Phoenix,
      PromEx.Plugins.Ecto,
      PromEx.Plugins.Oban,
      {PromEx.Plugins.Phoenix, endpoint: MyAppWeb.Endpoint}
    ]
  end

  @impl true
  def dashboard_assigns do
    [
      datasource_id: "Prometheus",
      default_selected_interval: "30s"
    ]
  end

  # Custom metrics
  @impl true
  def init_opts do
    PromEx.plugin_init_opts(drop_metrics_groups: [])
  end
end

# Custom business metrics
defmodule MyApp.Metrics do
  def track_order_created(amount) do
    :telemetry.execute(
      [:my_app, :orders, :created],
      %{total: amount},
      %{}
    )
  end

  def track_payment_processed(status, duration_ms) do
    :telemetry.execute(
      [:my_app, :payments, :processed],
      %{duration: duration_ms},
      %{status: status}
    )
  end
end

# In application.ex
:telemetry.attach_many(
  "my-app-metrics",
  [
    [:my_app, :orders, :created],
    [:my_app, :payments, :processed],
  ],
  &MyApp.Metrics.handle_event/4,
  nil
)
```

---

## 4. Alerting Rules (Prometheus/Grafana)

```yaml
# prometheus/rules/my_app.yml
groups:
  - name: my_app
    rules:
      # High error rate
      - alert: HighErrorRate
        expr: |
          rate(phoenix_http_request_duration_milliseconds_count{status=~"5.."}[5m]) /
          rate(phoenix_http_request_duration_milliseconds_count[5m]) > 0.05
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "High error rate: {{ $value | humanizePercentage }}"

      # Slow response time
      - alert: SlowResponseTime
        expr: |
          histogram_quantile(0.95, rate(phoenix_http_request_duration_milliseconds_bucket[5m])) > 1000
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "P95 response time > 1s: {{ $value }}ms"

      # High memory usage
      - alert: HighMemoryUsage
        expr: beam_memory_processes_bytes > 1073741824  # 1GB
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Elixir process memory > 1GB"

      # Database connection pool exhausted
      - alert: DBPoolExhausted
        expr: ecto_repo_pool_checked_out_connections / ecto_repo_pool_size > 0.9
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "DB connection pool > 90% utilized"
```

---

## 5. Health Dashboard LiveView

```elixir
defmodule MyAppWeb.Admin.SystemHealthLive do
  use MyAppWeb, :live_view

  def mount(_params, _session, socket) do
    if connected?(socket) do
      :timer.send_interval(5_000, self(), :refresh)
    end

    {:ok, assign(socket, system_info: get_system_info())}
  end

  def handle_info(:refresh, socket) do
    {:noreply, assign(socket, system_info: get_system_info())}
  end

  defp get_system_info do
    %{
      memory: :erlang.memory(),
      process_count: length(Process.list()),
      schedulers: :erlang.system_info(:schedulers_online),
      db_pool_size: pool_size(),
      db_pool_checked_out: pool_checked_out(),
      uptime_seconds: uptime()
    }
  end

  defp pool_size do
    DBConnection.get_connection(MyApp.Repo.get_dynamic_repo(), [], []).pool_size
  rescue
    _ -> 0
  end

  defp pool_checked_out, do: 0  # query Telemetry metrics

  defp uptime do
    {total, _} = :erlang.statistics(:wall_clock)
    div(total, 1000)
  end

  def render(assigns) do
    ~H"""
    <div class="p-6">
      <h1 class="text-2xl font-bold mb-6">System Health</h1>
      <div class="grid grid-cols-3 gap-4">
        <div class="bg-white p-4 rounded shadow">
          <p class="text-gray-500 text-sm">Memory (MB)</p>
          <p class="text-2xl font-bold">
            <%= Float.round(@system_info.memory[:total] / 1_048_576, 1) %>
          </p>
        </div>
        <div class="bg-white p-4 rounded shadow">
          <p class="text-gray-500 text-sm">Processes</p>
          <p class="text-2xl font-bold"><%= @system_info.process_count %></p>
        </div>
        <div class="bg-white p-4 rounded shadow">
          <p class="text-gray-500 text-sm">Uptime</p>
          <p class="text-2xl font-bold">
            <%= format_uptime(@system_info.uptime_seconds) %>
          </p>
        </div>
      </div>
    </div>
    """
  end

  defp format_uptime(seconds) do
    days = div(seconds, 86400)
    hours = div(rem(seconds, 86400), 3600)
    "#{days}d #{hours}h"
  end
end
```

---

## สรุป

```
Monitoring Stack:
├── Sentry: error tracking + alerts
├── PromEx: Prometheus metrics
├── Grafana: dashboards + alerting
└── Structured logs: JSON to Loki/CloudWatch

Key Metrics:
├── HTTP: request rate, error rate, latency p95
├── Database: pool utilization, query time
├── BEAM: memory, processes, GC
└── Business: orders, payments, signups

Log Levels:
├── debug: development only
├── info: normal operations
├── warning: non-critical issues
└── error: failures requiring attention
```

---

*ก่อนหน้า: [Part 86](part_86.md) | ต่อไป: [Part 88 - API Rate Limiting](part_88.md)*
