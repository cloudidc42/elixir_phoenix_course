# Part 55: Monitoring and Observability (การติดตามและสังเกตระบบ)

## เป้าหมายการเรียนรู้

หลังจากจบ Part นี้ คุณจะสามารถ:
- ตั้งค่า Telemetry metrics
- Integrate กับ Prometheus/Grafana
- Structured logging
- Distributed tracing
- Health checks และ readiness probes

---

## 1. Telemetry Events

```elixir
# lib/my_app/telemetry.ex
defmodule MyApp.Telemetry do
  require Logger

  def attach_handlers do
    :telemetry.attach_many(
      "my-app-handlers",
      [
        [:my_app, :repo, :query],
        [:my_app, :cache, :hit],
        [:my_app, :cache, :miss],
        [:my_app, :job, :start],
        [:my_app, :job, :stop],
        [:my_app, :api, :request],
      ],
      &handle_event/4,
      nil
    )
  end

  def handle_event([:my_app, :repo, :query], measurements, metadata, _config) do
    duration_ms = System.convert_time_unit(measurements.total_time, :native, :millisecond)
    Logger.debug("DB Query: #{metadata.query} (#{duration_ms}ms)")

    if duration_ms > 100 do
      Logger.warning("Slow query (#{duration_ms}ms): #{metadata.query}")
    end
  end

  def handle_event([:my_app, :cache, event], _measurements, metadata, _config) do
    Logger.debug("Cache #{event}: #{metadata.key}")
  end

  def handle_event([:my_app, :job, :stop], measurements, metadata, _config) do
    duration_ms = System.convert_time_unit(measurements.duration, :native, :millisecond)

    if metadata.status == :error do
      Logger.error("Job failed: #{metadata.job} - #{metadata.error} (#{duration_ms}ms)")
    end
  end

  def handle_event(event, measurements, metadata, _config) do
    Logger.debug("Telemetry: #{inspect(event)} #{inspect(measurements)}")
  end
end

# lib/my_app/application.ex
def start(_type, _args) do
  MyApp.Telemetry.attach_handlers()
  # ...
end
```

---

## 2. Custom Telemetry Metrics

```elixir
defmodule MyApp.Telemetry.Metrics do
  import Telemetry.Metrics

  def metrics do
    [
      # HTTP metrics
      counter("phoenix.endpoint.stop.duration",
        event_name: [:phoenix, :endpoint, :stop],
        measurement: :duration,
        tags: [:method, :route, :status]
      ),
      summary("phoenix.endpoint.stop.duration",
        unit: {:native, :millisecond}
      ),

      # Database metrics
      summary("my_app.repo.query.total_time",
        unit: {:native, :millisecond},
        tags: [:source]
      ),
      counter("my_app.repo.query.count"),

      # Cache metrics
      counter("my_app.cache.hit.count"),
      counter("my_app.cache.miss.count"),

      # Business metrics
      counter("my_app.orders.created.count"),
      counter("my_app.users.registered.count"),

      # VM metrics
      last_value("vm.memory.total", unit: {:byte, :megabyte}),
      last_value("vm.total_run_queue_lengths.total"),
      last_value("vm.system_counts.process_count"),
    ]
  end
end
```

---

## 3. Prometheus Integration

```elixir
# mix.exs
{:prom_ex, "~> 1.9"},
{:prometheus_ex, "~> 3.0"}

# lib/my_app/prom_ex.ex
defmodule MyApp.PromEx do
  use PromEx, otp_app: :my_app

  alias PromEx.Plugins

  @impl true
  def plugins do
    [
      Plugins.Application,
      Plugins.Beam,
      Plugins.PhoenixLiveView,
      {Plugins.Phoenix, router: MyAppWeb.Router},
      {Plugins.Ecto, repos: [MyApp.Repo]},
      {Plugins.Oban, queue_poll_interval: 5_000},
    ]
  end

  @impl true
  def dashboard_assigns do
    [
      datasource_id: "Prometheus",
      default_selected_interval: "30s"
    ]
  end

  @impl true
  def dashboards do
    [
      {:prom_ex, "application.json"},
      {:prom_ex, "beam.json"},
      {:prom_ex, "phoenix.json"},
      {:prom_ex, "ecto.json"},
    ]
  end
end

# router.ex
scope "/" do
  get "/metrics", PromEx.Plug, prom_ex: MyApp.PromEx
end
```

---

## 4. Structured Logging

```elixir
# config/prod.exs
config :logger,
  backends: [:console],
  level: :info

config :logger, :console,
  format: {MyApp.LogFormatter, :format},
  metadata: [:request_id, :user_id, :trace_id]

# lib/my_app/log_formatter.ex
defmodule MyApp.LogFormatter do
  def format(level, message, timestamp, metadata) do
    log_entry = %{
      "@timestamp" => format_timestamp(timestamp),
      "level" => level,
      "message" => IO.iodata_to_binary(message),
      "request_id" => metadata[:request_id],
      "user_id" => metadata[:user_id],
      "trace_id" => metadata[:trace_id],
      "app" => "my_app",
      "env" => Application.get_env(:my_app, :env, "production")
    }
    |> Map.reject(fn {_, v} -> is_nil(v) end)

    [Jason.encode!(log_entry), "\n"]
  rescue
    _ -> "#{level}: #{message}\n"
  end

  defp format_timestamp({{year, month, day}, {hour, min, sec, ms}}) do
    "#{year}-#{pad(month)}-#{pad(day)}T#{pad(hour)}:#{pad(min)}:#{pad(sec)}.#{ms}Z"
  end

  defp pad(n), do: String.pad_leading(to_string(n), 2, "0")
end

# Plug for adding request context
defmodule MyAppWeb.Plugs.RequestLogger do
  import Plug.Conn
  require Logger

  def init(opts), do: opts

  def call(conn, _opts) do
    start_time = System.monotonic_time()
    request_id = Ecto.UUID.generate()

    conn = put_private(conn, :request_id, request_id)
    Logger.metadata(request_id: request_id)

    Plug.Conn.register_before_send(conn, fn conn ->
      duration_ms = System.convert_time_unit(
        System.monotonic_time() - start_time,
        :native,
        :millisecond
      )

      Logger.info("HTTP Request",
        method: conn.method,
        path: conn.request_path,
        status: conn.status,
        duration_ms: duration_ms,
        request_id: request_id
      )

      conn
    end)
  end
end
```

---

## 5. Health Checks

```elixir
defmodule MyAppWeb.HealthController do
  use MyAppWeb, :controller

  def check(conn, _params) do
    checks = %{
      database: check_database(),
      cache: check_cache(),
      oban: check_oban()
    }

    overall = if Enum.all?(Map.values(checks), & &1[:status] == "ok"),
      do: "ok",
      else: "degraded"

    status = if overall == "ok", do: 200, else: 503

    conn
    |> put_status(status)
    |> json(%{
      status: overall,
      checks: checks,
      version: Application.spec(:my_app, :vsn),
      timestamp: DateTime.utc_now()
    })
  end

  def ready(conn, _params) do
    if MyApp.Repo.connected?() do
      json(conn, %{status: "ready"})
    else
      conn |> put_status(503) |> json(%{status: "not ready"})
    end
  end

  def live(conn, _params) do
    json(conn, %{status: "alive"})
  end

  defp check_database do
    case MyApp.Repo.query("SELECT 1", [], timeout: 1000) do
      {:ok, _} -> %{status: "ok"}
      {:error, e} -> %{status: "error", message: Exception.message(e)}
    end
  rescue
    e -> %{status: "error", message: Exception.message(e)}
  end

  defp check_cache do
    key = "health_check_#{System.unique_integer()}"
    :ok = Cachex.put!(:app_cache, key, "test", ttl: :timer.seconds(5))
    case Cachex.get(:app_cache, key) do
      {:ok, "test"} -> %{status: "ok"}
      _ -> %{status: "error"}
    end
  rescue
    _ -> %{status: "error"}
  end

  defp check_oban do
    case Oban.check_queue(:default) do
      %{} -> %{status: "ok"}
      _ -> %{status: "error"}
    end
  rescue
    _ -> %{status: "error"}
  end
end
```

---

## 6. Distributed Tracing (OpenTelemetry)

```elixir
# mix.exs
{:opentelemetry, "~> 1.4"},
{:opentelemetry_api, "~> 1.3"},
{:opentelemetry_exporter, "~> 1.6"},
{:opentelemetry_phoenix, "~> 2.0"},
{:opentelemetry_ecto, "~> 1.1"}

# config/config.exs
config :opentelemetry_exporter,
  otlp_protocol: :grpc,
  otlp_endpoint: "http://jaeger:4317"

# lib/my_app/application.ex
def start(_type, _args) do
  OpentelemetryPhoenix.setup()
  OpentelemetryEcto.setup([:my_app, :repo])
  # ...
end

# Custom spans
defmodule MyApp.Service do
  require OpenTelemetry.Tracer, as: Tracer

  def process(user_id, data) do
    Tracer.with_span "process_user_data" do
      Tracer.set_attributes([
        {"user.id", user_id},
        {"data.size", byte_size(inspect(data))}
      ])

      result = do_processing(data)

      Tracer.set_status(:ok, "")
      result
    end
  end
end
```

---

## 7. Phoenix LiveDashboard

```elixir
# mix.exs
{:phoenix_live_dashboard, "~> 0.8"}

# router.ex
import Phoenix.LiveDashboard.Router

scope "/" do
  pipe_through [:browser, :require_admin]
  live_dashboard "/dashboard",
    metrics: MyAppWeb.Telemetry,
    additional_pages: [
      jobs: {Oban.LiveDashboard, oban: MyApp.Oban}
    ]
end
```

---

## สรุป

```
Observability:
├── Telemetry: custom events and metrics
├── Prometheus: metrics scraping
├── Structured logs: JSON format for ELK
└── OpenTelemetry: distributed tracing

Health Checks:
├── /health - overall system status
├── /health/ready - readiness probe (Kubernetes)
└── /health/live - liveness probe

Monitoring Stack:
├── App → Prometheus → Grafana (metrics)
├── App → Elasticsearch (logs)
└── App → Jaeger (traces)
```

---

*ก่อนหน้า: [Part 54](part_54.md) | ต่อไป: [Part 56 - Multi-tenancy](part_56.md)*
