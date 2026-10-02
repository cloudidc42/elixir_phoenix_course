# Part 82: Plug and Custom Middleware (Plug และ Middleware)

## เป้าหมายการเรียนรู้

หลังจากจบ Part นี้ คุณจะสามารถ:
- สร้าง Custom Plugs ที่ซับซอน
- Plug pipeline composition
- Request/Response transformation
- Plug สำหรับ API versioning

---

## 1. Request ID Tracking

```elixir
defmodule MyAppWeb.Plugs.RequestId do
  import Plug.Conn

  def init(opts), do: opts

  def call(conn, _opts) do
    request_id =
      get_req_header(conn, "x-request-id")
      |> List.first()
      |> case do
        nil -> Ecto.UUID.generate()
        id -> id
      end

    conn
    |> put_resp_header("x-request-id", request_id)
    |> assign(:request_id, request_id)
    |> put_private(:logger_metadata, request_id: request_id)
  end
end
```

---

## 2. API Versioning Plug

```elixir
defmodule MyAppWeb.Plugs.APIVersion do
  import Plug.Conn
  import Phoenix.Controller

  @supported_versions ["v1", "v2", "v3"]
  @default_version "v1"

  def init(opts), do: opts

  def call(conn, _opts) do
    version = extract_version(conn)

    if version in @supported_versions do
      assign(conn, :api_version, version)
    else
      conn
      |> put_status(400)
      |> json(%{
        error: "Unsupported API version",
        supported_versions: @supported_versions
      })
      |> halt()
    end
  end

  defp extract_version(conn) do
    # From URL: /api/v2/users
    path_version = Regex.run(~r|/api/(v\d+)/|, conn.request_path)
    |> case do
      [_, version] -> version
      _ -> nil
    end

    # From header: Accept: application/vnd.myapp.v2+json
    header_version = get_req_header(conn, "accept")
    |> List.first()
    |> case do
      nil -> nil
      accept -> Regex.run(~r/vnd\.myapp\.(v\d+)/, accept) |> then(fn
        [_, v] -> v
        _ -> nil
      end)
    end

    path_version || header_version || @default_version
  end
end

# Router usage
pipeline :api_v1 do
  plug MyAppWeb.Plugs.APIVersion
  plug :put_version_view, "v1"
end
```

---

## 3. Response Transformation Plug

```elixir
defmodule MyAppWeb.Plugs.JSONWrapper do
  import Plug.Conn

  def init(opts), do: opts

  def call(conn, _opts) do
    register_before_send(conn, fn conn ->
      if json_response?(conn) do
        transform_response(conn)
      else
        conn
      end
    end)
  end

  defp json_response?(conn) do
    conn.resp_headers
    |> Enum.any?(fn {k, v} ->
      k == "content-type" and String.contains?(v, "application/json")
    end)
  end

  defp transform_response(conn) do
    case Jason.decode(conn.resp_body) do
      {:ok, body} ->
        wrapped = %{
          data: body,
          meta: %{
            request_id: conn.assigns[:request_id],
            timestamp: DateTime.utc_now(),
            version: conn.assigns[:api_version] || "v1"
          }
        }
        %{conn | resp_body: Jason.encode!(wrapped)}

      {:error, _} ->
        conn
    end
  end
end
```

---

## 4. Telemetry Plug

```elixir
defmodule MyAppWeb.Plugs.Telemetry do
  import Plug.Conn
  require Logger

  def init(opts), do: opts

  def call(conn, _opts) do
    start_time = System.monotonic_time()

    register_before_send(conn, fn conn ->
      duration = System.monotonic_time() - start_time

      :telemetry.execute(
        [:my_app, :request, :complete],
        %{duration: duration},
        %{
          method: conn.method,
          path: conn.request_path,
          status: conn.status,
          request_id: conn.assigns[:request_id]
        }
      )

      if duration > System.convert_time_unit(500, :millisecond, :native) do
        Logger.warning("Slow request: #{conn.method} #{conn.request_path} #{div(duration, 1_000_000)}ms")
      end

      conn
    end)
  end
end
```

---

## 5. Content Negotiation Plug

```elixir
defmodule MyAppWeb.Plugs.ContentNegotiation do
  import Plug.Conn
  import Phoenix.Controller

  @accepted_formats ~w(json html csv)
  @format_content_types %{
    "json" => "application/json",
    "html" => "text/html",
    "csv" => "text/csv"
  }

  def init(opts), do: opts

  def call(conn, _opts) do
    format = determine_format(conn)

    conn
    |> assign(:response_format, format)
    |> put_resp_content_type(@format_content_types[format])
  end

  defp determine_format(conn) do
    # 1. Check query param
    case conn.query_params["format"] do
      format when format in @accepted_formats -> format
      _ ->
        # 2. Check Accept header
        accept = get_req_header(conn, "accept") |> List.first() || ""
        cond do
          String.contains?(accept, "application/json") -> "json"
          String.contains?(accept, "text/csv") -> "csv"
          true -> "json"  # default
        end
    end
  end
end
```

---

## 6. Plug Router for API

```elixir
defmodule MyApp.HealthAPI do
  use Plug.Router

  plug :match
  plug Plug.Parsers, parsers: [:json], json_decoder: Jason
  plug :dispatch

  get "/health" do
    send_resp(conn, 200, Jason.encode!(%{status: "ok"}))
  end

  get "/health/db" do
    case Ecto.Adapters.SQL.query(MyApp.Repo, "SELECT 1", []) do
      {:ok, _} -> send_resp(conn, 200, Jason.encode!(%{database: "ok"}))
      _ -> send_resp(conn, 503, Jason.encode!(%{database: "error"}))
    end
  end

  match _ do
    send_resp(conn, 404, Jason.encode!(%{error: "not found"}))
  end
end

# Embed in Phoenix router
scope "/" do
  forward "/probe", MyApp.HealthAPI
end
```

---

## สรุป

```
Plug Types:
├── Module plug: %{init, call}
├── Function plug: fn conn, opts -> conn
└── Plug.Router: mini-app

Common Plugs:
├── Request ID: track across services
├── API Version: URL or header based
├── Response wrapper: add meta to JSON
└── Telemetry: measure request duration

Plug Pipeline (order matters):
plug :req_id      # assigns request_id
plug :auth        # assigns current_user
plug :rate_limit  # checks limits
plug :telemetry   # measures duration
plug :body        # parse request body
```

---

*ก่อนหน้า: [Part 81](part_81.md) | ต่อไป: [Part 83 - File Storage and CDN](part_83.md)*
