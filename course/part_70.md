# Part 70: Microservices with Elixir (สถาปัตยกรรม Microservices)

## เป้าหมายการเรียนรู้

หลังจากจบ Part นี้ คุณจะสามารถ:
- สื่อสารระหว่าง services ด้วย HTTP/gRPC
- Service discovery
- Circuit breaker pattern
- API Gateway

---

## 1. HTTP Client สำหรับ Service Communication

```elixir
# mix.exs: {:req, "~> 0.5"}

defmodule MyApp.Services.UserService do
  @base_url Application.compile_env(:my_app, :user_service_url, "http://user-service:4001")

  def get_user(user_id) do
    case Req.get("#{@base_url}/api/users/#{user_id}") do
      {:ok, %{status: 200, body: body}} ->
        {:ok, body}
      {:ok, %{status: 404}} ->
        {:error, :not_found}
      {:ok, %{status: status}} ->
        {:error, {:http_error, status}}
      {:error, exception} ->
        {:error, {:network_error, exception}}
    end
  end

  def create_user(attrs) do
    case Req.post("#{@base_url}/api/users", json: attrs) do
      {:ok, %{status: 201, body: body}} -> {:ok, body}
      {:ok, %{status: 422, body: body}} -> {:error, {:validation, body}}
      {:error, exception} -> {:error, {:network_error, exception}}
    end
  end
end
```

---

## 2. Circuit Breaker

```elixir
# mix.exs: {:fuse, "~> 2.4"}

defmodule MyApp.Services.CircuitBreaker do
  @max_failures 5
  @reset_timeout :timer.seconds(30)

  def call(service_name, fun) do
    case :fuse.ask(service_name, :sync) do
      :ok ->
        try do
          result = fun.()
          case result do
            {:error, {:network_error, _}} ->
              :fuse.melt(service_name)
            {:error, {:http_error, status}} when status >= 500 ->
              :fuse.melt(service_name)
            _ -> :ok
          end
          result
        rescue
          e ->
            :fuse.melt(service_name)
            {:error, {:exception, e}}
        end

      :blown ->
        {:error, :circuit_open}
    end
  end

  def setup(service_name) do
    :fuse.install(service_name, {
      {:standard, @max_failures, :timer.seconds(10)},
      {:reset, @reset_timeout}
    })
  end
end

# ใช้งาน
defmodule MyApp.Services.PaymentService do
  @service_name :payment_service

  def charge(customer_id, amount) do
    MyApp.Services.CircuitBreaker.call(@service_name, fn ->
      Req.post("#{base_url()}/charge", json: %{customer_id: customer_id, amount: amount})
    end)
  end
end
```

---

## 3. Event-driven Communication via RabbitMQ

```elixir
# mix.exs: {:amqp, "~> 3.3"}

defmodule MyApp.MessageBus do
  use GenServer
  require Logger

  def start_link(opts \\ []) do
    GenServer.start_link(__MODULE__, opts, name: __MODULE__)
  end

  def publish(exchange, routing_key, payload) do
    GenServer.call(__MODULE__, {:publish, exchange, routing_key, payload})
  end

  def subscribe(queue, handler) do
    GenServer.cast(__MODULE__, {:subscribe, queue, handler})
  end

  def init(_opts) do
    {:ok, conn} = AMQP.Connection.open(
      host: Application.get_env(:my_app, :rabbitmq_host, "localhost"),
      username: "guest",
      password: "guest"
    )
    {:ok, channel} = AMQP.Channel.open(conn)

    # Setup exchanges
    AMQP.Exchange.declare(channel, "events", :topic, durable: true)

    {:ok, %{conn: conn, channel: channel}}
  end

  def handle_call({:publish, exchange, routing_key, payload}, _from, state) do
    message = Jason.encode!(payload)
    result = AMQP.Basic.publish(
      state.channel,
      exchange,
      routing_key,
      message,
      content_type: "application/json",
      persistent: true
    )
    {:reply, result, state}
  end

  def handle_cast({:subscribe, queue, handler}, state) do
    AMQP.Queue.declare(state.channel, queue, durable: true)
    AMQP.Queue.bind(state.channel, queue, "events", routing_key: "#")
    AMQP.Basic.consume(state.channel, queue)

    {:noreply, Map.put(state, :handler, handler)}
  end

  def handle_info({:basic_deliver, payload, meta}, state) do
    case Jason.decode(payload) do
      {:ok, event} ->
        Task.start(fn -> state.handler.(event, meta.routing_key) end)
      {:error, _} ->
        Logger.error("Failed to decode message: #{payload}")
    end

    AMQP.Basic.ack(state.channel, meta.delivery_tag)
    {:noreply, state}
  end
end

# Publishing events
defmodule MyApp.Orders do
  alias MyApp.MessageBus

  def create_order(attrs) do
    with {:ok, order} <- create_order_in_db(attrs) do
      MessageBus.publish("events", "orders.created", %{
        order_id: order.id,
        user_id: order.user_id,
        total: order.total,
        items: order.items
      })
      {:ok, order}
    end
  end
end

# Consuming events in another service
MessageBus.subscribe("notification-queue", fn event, routing_key ->
  case routing_key do
    "orders.created" -> NotificationService.send_order_confirmation(event)
    "users.registered" -> EmailService.send_welcome_email(event)
    _ -> :ok
  end
end)
```

---

## 4. API Gateway

```elixir
defmodule MyAppWeb.Gateway do
  use Phoenix.Router

  # Route to appropriate microservice
  forward "/api/users", MyApp.Proxies.UserServiceProxy
  forward "/api/payments", MyApp.Proxies.PaymentServiceProxy
  forward "/api/orders", MyApp.Proxies.OrderServiceProxy
end

defmodule MyApp.Proxies.UserServiceProxy do
  use Plug.Router

  plug :match
  plug :dispatch

  get "/" do
    case MyApp.Services.UserService.list_users(conn.query_params) do
      {:ok, users} -> send_resp(conn, 200, Jason.encode!(users))
      {:error, reason} -> send_resp(conn, 500, Jason.encode!(%{error: reason}))
    end
  end

  get "/:id" do
    case MyApp.Services.UserService.get_user(id) do
      {:ok, user} -> send_resp(conn, 200, Jason.encode!(user))
      {:error, :not_found} -> send_resp(conn, 404, Jason.encode!(%{error: "Not found"}))
      {:error, :circuit_open} ->
        send_resp(conn, 503, Jason.encode!(%{error: "Service unavailable"}))
    end
  end
end
```

---

## 5. Health Check Aggregation

```elixir
defmodule MyApp.HealthCheck do
  alias MyApp.Services

  def check_all do
    services = [
      {"user-service", &Services.UserService.health_check/0},
      {"payment-service", &Services.PaymentService.health_check/0},
      {"notification-service", &Services.NotificationService.health_check/0},
    ]

    results = Task.async_stream(
      services,
      fn {name, check_fn} ->
        {name, check_service(check_fn)}
      end,
      max_concurrency: 5,
      timeout: 5_000
    )
    |> Enum.map(fn {:ok, result} -> result end)
    |> Map.new()

    all_healthy = Enum.all?(results, fn {_, status} -> status == :healthy end)

    %{
      status: if(all_healthy, do: :healthy, else: :degraded),
      services: results,
      checked_at: DateTime.utc_now()
    }
  end

  defp check_service(fun) do
    try do
      case fun.() do
        :ok -> :healthy
        {:ok, _} -> :healthy
        _ -> :unhealthy
      end
    catch
      _, _ -> :unhealthy
    end
  end
end
```

---

## สรุป

```
Microservices Patterns:
├── HTTP client: Req library
├── Circuit breaker: Fuse
├── Event-driven: RabbitMQ/AMQP
└── API Gateway: Plug.Router proxy

Communication:
├── Sync: HTTP REST, gRPC
├── Async: RabbitMQ, Kafka events
└── Service mesh: Consul, Istio

Resilience:
├── Circuit breaker: prevent cascading failures
├── Retry with backoff
├── Timeout: all external calls
└── Health checks: readiness + liveness
```

---

*ก่อนหน้า: [Part 69](part_69.md) | ต่อไป: [Part 71 - SaaS Platform](part_71.md)*
