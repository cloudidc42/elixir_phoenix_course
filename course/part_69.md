# Part 69: Microservices Communication

## เป้าหมายการเรียนรู้

- เรียก REST API ภายนอกด้วย Req library
- สร้าง circuit breaker pattern ป้องกัน cascading failures
- เข้าใจ service discovery patterns
- ส่งข้อความผ่าน RabbitMQ ด้วย AMQP
- ใช้ Phoenix PubSub สำหรับ event-driven communication
- เชื่อมต่อ gRPC ด้วย protobuf
- ออกแบบ API gateway pattern

---

## 1. REST API Client ด้วย Req

Req เป็น HTTP client ที่ทันสมัยสำหรับ Elixir มี middleware system ที่ยืดหยุ่น

### 1.1 ติดตั้งและตั้งค่า

```elixir
# mix.exs
defp deps do
  [
    {:req, "~> 0.5"},
    # ...
  ]
end
```

```elixir
# lib/my_app/api_client.ex
defmodule MyApp.APIClient do
  @moduledoc """
  Base HTTP client พร้อม retry, logging, และ error handling
  """

  def new(base_url, opts \\ []) do
    default_opts = [
      base_url: base_url,
      retry: :transient,
      retry_delay: fn attempt -> :timer.seconds(attempt) end,
      max_retries: 3,
      receive_timeout: 30_000,
      connect_options: [timeout: 5_000],
      headers: [
        {"content-type", "application/json"},
        {"accept", "application/json"},
        {"user-agent", "MyApp/1.0"}
      ]
    ]

    Req.new(Keyword.merge(default_opts, opts))
  end

  # ตั้งค่า request ด้วย authentication
  def with_bearer_token(client, token) do
    Req.merge(client, auth: {:bearer, token})
  end

  def with_api_key(client, api_key, header_name \\ "x-api-key") do
    Req.merge(client, headers: [{header_name, api_key}])
  end
end
```

### 1.2 Service Client ตัวอย่าง

```elixir
defmodule MyApp.Services.PaymentService do
  alias MyApp.APIClient
  require Logger

  @base_url Application.compile_env(:my_app, [:payment_service, :url])
  @api_key Application.compile_env(:my_app, [:payment_service, :api_key])

  defp client do
    APIClient.new(@base_url)
    |> APIClient.with_api_key(@api_key)
  end

  def charge(customer_id, amount, currency \\ "THB") do
    params = %{
      customer_id: customer_id,
      amount: amount,
      currency: currency
    }

    case Req.post(client(), url: "/charges", json: params) do
      {:ok, %{status: 200, body: body}} ->
        {:ok, body}

      {:ok, %{status: 402, body: body}} ->
        {:error, {:payment_declined, body["message"]}}

      {:ok, %{status: status, body: body}} ->
        Logger.warning("Payment API error: #{status}, #{inspect(body)}")
        {:error, {:api_error, status, body}}

      {:error, %Req.TransportError{reason: reason}} ->
        Logger.error("Payment service connection failed: #{inspect(reason)}")
        {:error, {:connection_failed, reason}}
    end
  end

  def get_customer(customer_id) do
    case Req.get(client(), url: "/customers/#{customer_id}") do
      {:ok, %{status: 200, body: body}} -> {:ok, body}
      {:ok, %{status: 404}} -> {:error, :not_found}
      {:ok, %{status: status}} -> {:error, {:unexpected_status, status}}
      {:error, error} -> {:error, {:request_failed, error}}
    end
  end

  def refund(charge_id, amount \\ nil) do
    params = if amount, do: %{amount: amount}, else: %{}

    case Req.post(client(), url: "/charges/#{charge_id}/refund", json: params) do
      {:ok, %{status: 200, body: body}} -> {:ok, body}
      {:ok, %{status: status, body: body}} -> {:error, {status, body}}
      {:error, error} -> {:error, error}
    end
  end
end
```

---

## 2. Circuit Breaker Pattern

Circuit breaker ป้องกันการเรียก service ที่ล้มเหลวซ้ำๆ ลด load และรอให้ recover

### 2.1 ติดตั้ง Fuse

```elixir
# mix.exs
defp deps do
  [
    {:fuse, "~> 2.4"},
    # ...
  ]
end
```

### 2.2 Circuit Breaker Module

```elixir
defmodule MyApp.CircuitBreaker do
  require Logger

  @fuse_options {{:standard, 5, 10_000}, {:reset, 30_000}}
  # 5 failures within 10 seconds → open circuit
  # Try to reset after 30 seconds

  def setup(service_name) do
    :fuse.install(service_name, @fuse_options)
  end

  def call(service_name, fun, opts \\ []) do
    timeout = Keyword.get(opts, :timeout, 10_000)
    fallback = Keyword.get(opts, :fallback, nil)

    case :fuse.ask(service_name, :sync) do
      :ok ->
        # Circuit closed - ลองเรียก
        execute_with_timeout(service_name, fun, timeout, fallback)

      :blown ->
        # Circuit open - reject immediately
        Logger.warning("Circuit breaker open for #{service_name}")
        if fallback, do: {:ok, fallback.()}, else: {:error, :circuit_open}

      {:error, :not_found} ->
        # Circuit ยังไม่ได้ setup
        setup(service_name)
        call(service_name, fun, opts)
    end
  end

  defp execute_with_timeout(service_name, fun, timeout, fallback) do
    task = Task.async(fun)

    case Task.yield(task, timeout) || Task.shutdown(task) do
      {:ok, {:ok, result}} ->
        {:ok, result}

      {:ok, {:error, reason}} ->
        :fuse.melt(service_name)  # บันทึก failure
        Logger.warning("#{service_name} failed: #{inspect(reason)}")
        if fallback, do: {:ok, fallback.()}, else: {:error, reason}

      nil ->
        # Timeout
        :fuse.melt(service_name)
        Logger.error("#{service_name} timed out after #{timeout}ms")
        if fallback, do: {:ok, fallback.()}, else: {:error, :timeout}

      {:exit, reason} ->
        :fuse.melt(service_name)
        Logger.error("#{service_name} crashed: #{inspect(reason)}")
        if fallback, do: {:ok, fallback.()}, else: {:error, {:crashed, reason}}
    end
  end

  def status(service_name) do
    case :fuse.ask(service_name, :sync) do
      :ok -> :closed
      :blown -> :open
      _ -> :unknown
    end
  end

  def reset(service_name) do
    :fuse.reset(service_name)
  end
end
```

### 2.3 ใช้งาน Circuit Breaker

```elixir
defmodule MyApp.Services.RecommendationService do
  alias MyApp.CircuitBreaker
  alias MyApp.APIClient

  @service_name :recommendation_service

  def init do
    CircuitBreaker.setup(@service_name)
  end

  def get_recommendations(user_id) do
    CircuitBreaker.call(
      @service_name,
      fn ->
        client = APIClient.new("https://recommendations.internal")
        case Req.get(client, url: "/users/#{user_id}/recommendations") do
          {:ok, %{status: 200, body: body}} -> {:ok, body["items"]}
          {:ok, %{status: status}} -> {:error, {:http_error, status}}
          {:error, error} -> {:error, error}
        end
      end,
      # Fallback: return popular items instead
      fallback: fn ->
        MyApp.Products.get_popular_products(limit: 10)
      end,
      timeout: 5_000
    )
  end
end
```

---

## 3. Message Queues ด้วย RabbitMQ

### 3.1 ติดตั้ง AMQP

```elixir
# mix.exs
defp deps do
  [
    {:amqp, "~> 3.3"},
    # ...
  ]
end
```

### 3.2 Connection Pool

```elixir
defmodule MyApp.RabbitMQ do
  use Application

  def start(_type, _args) do
    children = [
      {MyApp.RabbitMQ.ConnectionSupervisor, []},
    ]
    Supervisor.start_link(children, strategy: :one_for_one)
  end
end

defmodule MyApp.RabbitMQ.Connection do
  use GenServer
  require Logger

  @reconnect_interval 5_000

  def start_link(opts) do
    GenServer.start_link(__MODULE__, opts, name: __MODULE__)
  end

  def get_channel do
    GenServer.call(__MODULE__, :get_channel)
  end

  def init(_opts) do
    {:ok, %{conn: nil, channel: nil}, {:continue, :connect}}
  end

  def handle_continue(:connect, state) do
    case connect() do
      {:ok, conn, channel} ->
        Process.monitor(conn.pid)
        {:noreply, %{state | conn: conn, channel: channel}}
      {:error, reason} ->
        Logger.error("RabbitMQ connection failed: #{inspect(reason)}")
        schedule_reconnect()
        {:noreply, state}
    end
  end

  def handle_call(:get_channel, _from, %{channel: nil} = state) do
    {:reply, {:error, :not_connected}, state}
  end
  def handle_call(:get_channel, _from, %{channel: channel} = state) do
    {:reply, {:ok, channel}, state}
  end

  def handle_info({:DOWN, _, :process, _, reason}, state) do
    Logger.warning("RabbitMQ connection lost: #{inspect(reason)}")
    schedule_reconnect()
    {:noreply, %{state | conn: nil, channel: nil}}
  end

  def handle_info(:reconnect, state) do
    {:noreply, state, {:continue, :connect}}
  end

  defp connect do
    url = Application.get_env(:my_app, :rabbitmq_url, "amqp://guest:guest@localhost")

    with {:ok, conn} <- AMQP.Connection.open(url),
         {:ok, channel} <- AMQP.Channel.open(conn) do
      {:ok, conn, channel}
    end
  end

  defp schedule_reconnect do
    Process.send_after(self(), :reconnect, @reconnect_interval)
  end
end
```

### 3.3 Publisher

```elixir
defmodule MyApp.RabbitMQ.Publisher do
  alias MyApp.RabbitMQ.Connection
  require Logger

  def publish(exchange, routing_key, payload, opts \\ []) do
    with {:ok, channel} <- Connection.get_channel() do
      message = Jason.encode!(payload)

      AMQP.Basic.publish(
        channel,
        exchange,
        routing_key,
        message,
        [
          content_type: "application/json",
          persistent: Keyword.get(opts, :persistent, true),
          timestamp: System.system_time(:second),
          headers: Keyword.get(opts, :headers, [])
        ]
      )
    else
      {:error, :not_connected} ->
        Logger.error("Cannot publish: RabbitMQ not connected")
        {:error, :not_connected}
    end
  end

  # Publish ด้วย confirmation (ชัวร์ว่าถึง broker)
  def publish_confirmed(exchange, routing_key, payload, timeout \\ 5_000) do
    with {:ok, channel} <- Connection.get_channel() do
      AMQP.Confirm.select(channel)
      message = Jason.encode!(payload)

      AMQP.Basic.publish(channel, exchange, routing_key, message,
        content_type: "application/json",
        persistent: true
      )

      case AMQP.Confirm.wait_for_confirms(channel, timeout) do
        :ok -> :ok
        :timeout -> {:error, :confirmation_timeout}
        {:error, reason} -> {:error, reason}
      end
    end
  end
end
```

### 3.4 Consumer

```elixir
defmodule MyApp.RabbitMQ.Consumer do
  use GenServer
  use AMQP
  require Logger

  @queue "order_events"
  @exchange "orders"

  def start_link(opts) do
    GenServer.start_link(__MODULE__, opts, name: __MODULE__)
  end

  def init(_opts) do
    {:ok, %{channel: nil}, {:continue, :setup}}
  end

  def handle_continue(:setup, state) do
    case setup_consumer() do
      {:ok, channel} ->
        {:noreply, %{state | channel: channel}}
      {:error, reason} ->
        Logger.error("Consumer setup failed: #{inspect(reason)}")
        Process.send_after(self(), :retry_setup, 5_000)
        {:noreply, state}
    end
  end

  def handle_info({:basic_deliver, payload, meta}, %{channel: channel} = state) do
    spawn(fn -> process_message(payload, meta, channel) end)
    {:noreply, state}
  end

  def handle_info({:basic_cancel, _}, state) do
    Logger.warning("Consumer cancelled by broker")
    {:stop, :cancelled, state}
  end

  def handle_info(:retry_setup, state) do
    {:noreply, state, {:continue, :setup}}
  end

  defp setup_consumer do
    with {:ok, channel} <- MyApp.RabbitMQ.Connection.get_channel() do
      # Declare exchange
      AMQP.Exchange.declare(channel, @exchange, :topic, durable: true)

      # Declare queue
      AMQP.Queue.declare(channel, @queue,
        durable: true,
        arguments: [
          {"x-dead-letter-exchange", :longstr, "orders.dlx"},
          {"x-message-ttl", :long, 86_400_000}
        ]
      )

      # Bind queue to exchange
      AMQP.Queue.bind(channel, @queue, @exchange, routing_key: "order.#")

      # Set prefetch count
      AMQP.Basic.qos(channel, prefetch_count: 10)

      # Start consuming
      {:ok, _tag} = AMQP.Basic.consume(channel, @queue, nil, no_ack: false)

      {:ok, channel}
    end
  end

  defp process_message(payload, meta, channel) do
    try do
      data = Jason.decode!(payload)
      event_type = meta.routing_key

      case handle_event(event_type, data) do
        :ok ->
          AMQP.Basic.ack(channel, meta.delivery_tag)
        {:error, reason} ->
          Logger.error("Failed to process #{event_type}: #{inspect(reason)}")
          # Requeue ถ้า attempt น้อย
          requeue = meta.redelivered == false
          AMQP.Basic.nack(channel, meta.delivery_tag, requeue: requeue)
      end
    rescue
      error ->
        Logger.error("Message processing crashed: #{inspect(error)}")
        AMQP.Basic.nack(channel, meta.delivery_tag, requeue: false)
    end
  end

  defp handle_event("order.created", data) do
    MyApp.Orders.process_new_order(data)
  end

  defp handle_event("order.paid", data) do
    MyApp.Orders.mark_as_paid(data["order_id"])
  end

  defp handle_event("order.cancelled", data) do
    MyApp.Orders.process_cancellation(data["order_id"])
  end

  defp handle_event(event_type, _data) do
    Logger.warning("Unhandled event type: #{event_type}")
    :ok
  end
end
```

---

## 4. Event-Driven Communication ผ่าน PubSub

```elixir
defmodule MyApp.EventBus do
  @moduledoc """
  Internal event bus สำหรับ communication ระหว่าง modules
  """

  def publish(event_type, payload) do
    event = %{
      id: Ecto.UUID.generate(),
      type: event_type,
      payload: payload,
      timestamp: DateTime.utc_now()
    }

    Phoenix.PubSub.broadcast(MyApp.PubSub, "events:#{event_type}", event)
    Phoenix.PubSub.broadcast(MyApp.PubSub, "events:*", event)

    :ok
  end

  def subscribe(event_type) do
    Phoenix.PubSub.subscribe(MyApp.PubSub, "events:#{event_type}")
  end

  def subscribe_all do
    Phoenix.PubSub.subscribe(MyApp.PubSub, "events:*")
  end
end

# Event handlers
defmodule MyApp.EventHandlers.OrderHandler do
  use GenServer
  alias MyApp.EventBus

  def start_link(_opts) do
    GenServer.start_link(__MODULE__, [], name: __MODULE__)
  end

  def init(_opts) do
    EventBus.subscribe("order.created")
    EventBus.subscribe("order.paid")
    {:ok, %{}}
  end

  def handle_info(%{type: "order.created", payload: payload}, state) do
    # ส่ง notification ให้ admin
    MyApp.Notifications.Dispatcher.dispatch(
      %{id: payload["admin_user_id"], email: "admin@example.com"},
      "system",
      %{title: "คำสั่งซื้อใหม่ ##{payload["order_id"]}"}
    )
    {:noreply, state}
  end

  def handle_info(%{type: "order.paid", payload: payload}, state) do
    # อัพเดท inventory
    Enum.each(payload["items"], fn item ->
      MyApp.Inventory.decrease_stock(item["product_id"], item["quantity"])
    end)
    {:noreply, state}
  end

  def handle_info(_event, state), do: {:noreply, state}
end
```

---

## 5. gRPC ด้วย Protobuf

### 5.1 ติดตั้ง gRPC

```elixir
# mix.exs
defp deps do
  [
    {:grpc, "~> 0.7"},
    {:protobuf, "~> 0.12"},
    # ...
  ]
end
```

### 5.2 Protobuf Definition

```protobuf
// priv/protos/user_service.proto
syntax = "proto3";

package myapp;

service UserService {
  rpc GetUser (GetUserRequest) returns (UserResponse);
  rpc CreateUser (CreateUserRequest) returns (UserResponse);
  rpc ListUsers (ListUsersRequest) returns (stream UserResponse);
  rpc BatchCreateUsers (stream CreateUserRequest) returns (BatchCreateResponse);
}

message GetUserRequest {
  int64 id = 1;
}

message CreateUserRequest {
  string email = 1;
  string name = 2;
  string role = 3;
}

message UserResponse {
  int64 id = 1;
  string email = 2;
  string name = 3;
  string role = 4;
  string created_at = 5;
}

message ListUsersRequest {
  int32 page = 1;
  int32 per_page = 2;
  string filter = 3;
}

message BatchCreateResponse {
  int32 created_count = 1;
  repeated string errors = 2;
}
```

### 5.3 Generate Elixir Code จาก Proto

```bash
# ติดตั้ง protoc และ protoc-gen-elixir
# แล้วรัน:
protoc --elixir_out=plugins=grpc:./lib priv/protos/user_service.proto
```

### 5.4 gRPC Server

```elixir
defmodule MyApp.GRPC.UserServiceServer do
  use GRPC.Server, service: MyApp.UserService.Service

  alias MyApp.{Accounts, Repo}
  import Ecto.Query

  def get_user(request, _stream) do
    case Accounts.get_user(request.id) do
      nil ->
        raise GRPC.RPCError, status: :not_found, message: "User not found"
      user ->
        user_to_proto(user)
    end
  end

  def create_user(request, _stream) do
    case Accounts.create_user(%{
      email: request.email,
      name: request.name,
      role: request.role
    }) do
      {:ok, user} -> user_to_proto(user)
      {:error, changeset} ->
        errors = Ecto.Changeset.traverse_errors(changeset, fn {msg, _opts} -> msg end)
        raise GRPC.RPCError,
          status: :invalid_argument,
          message: "Validation failed: #{inspect(errors)}"
    end
  end

  # Server-side streaming
  def list_users(request, stream) do
    page = max(request.page, 1)
    per_page = max(request.per_page, 10) |> min(100)

    from(u in MyApp.Accounts.User,
      limit: ^per_page,
      offset: ^((page - 1) * per_page),
      order_by: u.id
    )
    |> Repo.all()
    |> Enum.each(fn user ->
      GRPC.Server.send_reply(stream, user_to_proto(user))
    end)
  end

  # Client-side streaming
  def batch_create_users(stream, _reply_stream) do
    results = Enum.reduce(GRPC.Server.recv(stream), {0, []}, fn request, {count, errors} ->
      case Accounts.create_user(%{email: request.email, name: request.name}) do
        {:ok, _} -> {count + 1, errors}
        {:error, cs} -> {count, ["#{request.email}: #{format_errors(cs)}" | errors]}
      end
    end)

    {created, errors} = results
    MyApp.BatchCreateResponse.new(created_count: created, errors: errors)
  end

  defp user_to_proto(user) do
    MyApp.UserResponse.new(
      id: user.id,
      email: user.email,
      name: user.name,
      role: user.role,
      created_at: DateTime.to_iso8601(user.inserted_at)
    )
  end

  defp format_errors(changeset) do
    changeset
    |> Ecto.Changeset.traverse_errors(fn {msg, _} -> msg end)
    |> inspect()
  end
end
```

### 5.5 gRPC Client

```elixir
defmodule MyApp.Services.UserServiceClient do
  alias GRPC.Channel

  @host Application.compile_env(:my_app, [:user_service_grpc, :host], "localhost")
  @port Application.compile_env(:my_app, [:user_service_grpc, :port], 50051)

  defp channel do
    {:ok, channel} = GRPC.Stub.connect("#{@host}:#{@port}")
    channel
  end

  def get_user(user_id) do
    request = MyApp.GetUserRequest.new(id: user_id)

    case MyApp.UserService.Stub.get_user(channel(), request) do
      {:ok, response} -> {:ok, response}
      {:error, %GRPC.RPCError{status: :not_found}} -> {:error, :not_found}
      {:error, error} -> {:error, error}
    end
  end

  def list_users_stream(page \\ 1, per_page \\ 20) do
    request = MyApp.ListUsersRequest.new(page: page, per_page: per_page)
    {:ok, stream} = MyApp.UserService.Stub.list_users(channel(), request)

    Enum.map(stream, fn {:ok, user} -> user end)
  end
end
```

---

## 6. API Gateway Pattern

```elixir
defmodule MyAppWeb.APIGateway do
  use Plug.Router

  plug :match
  plug :dispatch

  # Rate limiting middleware
  plug MyAppWeb.Plugs.RateLimit, max_requests: 100, window: 60_000

  # Authentication
  plug MyAppWeb.Plugs.APIAuthentication

  # Route ไปยัง internal services
  forward "/users", to: MyAppWeb.Proxies.UserServiceProxy
  forward "/payments", to: MyAppWeb.Proxies.PaymentServiceProxy
  forward "/inventory", to: MyAppWeb.Proxies.InventoryServiceProxy

  match _ do
    send_resp(conn, 404, Jason.encode!(%{error: "Route not found"}))
  end
end

defmodule MyAppWeb.Proxies.UserServiceProxy do
  use Plug.Router
  alias MyApp.Services.UserServiceClient

  plug :match
  plug :dispatch

  get "/:id" do
    case UserServiceClient.get_user(String.to_integer(id)) do
      {:ok, user} ->
        conn
        |> put_resp_content_type("application/json")
        |> send_resp(200, Jason.encode!(proto_to_map(user)))
      {:error, :not_found} ->
        send_resp(conn, 404, Jason.encode!(%{error: "User not found"}))
      {:error, _} ->
        send_resp(conn, 503, Jason.encode!(%{error: "Service unavailable"}))
    end
  end

  defp proto_to_map(user) do
    %{
      id: user.id,
      email: user.email,
      name: user.name,
      role: user.role
    }
  end
end
```

### 6.1 Rate Limiting Plug

```elixir
defmodule MyAppWeb.Plugs.RateLimit do
  import Plug.Conn
  require Logger

  def init(opts), do: opts

  def call(conn, opts) do
    max_requests = Keyword.get(opts, :max_requests, 100)
    window_ms = Keyword.get(opts, :window, 60_000)

    client_id = get_client_id(conn)
    key = "rate_limit:#{client_id}"

    case check_rate_limit(key, max_requests, window_ms) do
      {:ok, remaining} ->
        conn
        |> put_resp_header("x-ratelimit-limit", "#{max_requests}")
        |> put_resp_header("x-ratelimit-remaining", "#{remaining}")

      {:error, :exceeded} ->
        conn
        |> put_status(429)
        |> put_resp_content_type("application/json")
        |> send_resp(429, Jason.encode!(%{
          error: "Rate limit exceeded",
          retry_after: div(window_ms, 1000)
        }))
        |> halt()
    end
  end

  defp check_rate_limit(key, max_requests, window_ms) do
    # ใช้ ETS สำหรับ in-memory rate limiting
    now = System.monotonic_time(:millisecond)

    case :ets.lookup(:rate_limit_table, key) do
      [] ->
        :ets.insert(:rate_limit_table, {key, 1, now})
        {:ok, max_requests - 1}

      [{^key, count, timestamp}] ->
        if now - timestamp > window_ms do
          # Window expired - reset
          :ets.insert(:rate_limit_table, {key, 1, now})
          {:ok, max_requests - 1}
        else
          if count >= max_requests do
            {:error, :exceeded}
          else
            :ets.update_counter(:rate_limit_table, key, {2, 1})
            {:ok, max_requests - count - 1}
          end
        end
    end
  end

  defp get_client_id(conn) do
    case get_req_header(conn, "x-api-key") do
      [key | _] -> "api_key:#{key}"
      [] ->
        # Fallback to IP
        conn.remote_ip |> :inet.ntoa() |> to_string()
    end
  end
end
```

---

## 7. Service Discovery

```elixir
defmodule MyApp.ServiceRegistry do
  @moduledoc """
  Simple service registry ด้วย Elixir process registry
  สำหรับ production จริงควรใช้ Consul, etcd, หรือ Kubernetes DNS
  """
  use GenServer

  def start_link(_opts) do
    GenServer.start_link(__MODULE__, %{}, name: __MODULE__)
  end

  def register(service_name, host, port, metadata \\ %{}) do
    GenServer.call(__MODULE__, {:register, service_name, {host, port, metadata}})
  end

  def lookup(service_name) do
    GenServer.call(__MODULE__, {:lookup, service_name})
  end

  def deregister(service_name, host, port) do
    GenServer.call(__MODULE__, {:deregister, service_name, {host, port}})
  end

  # Server callbacks

  def init(_), do: {:ok, %{}}

  def handle_call({:register, name, instance}, _from, state) do
    instances = Map.get(state, name, [])
    new_state = Map.put(state, name, [instance | instances])
    {:reply, :ok, new_state}
  end

  def handle_call({:lookup, name}, _from, state) do
    case Map.get(state, name, []) do
      [] -> {:reply, {:error, :not_found}, state}
      instances ->
        # Round-robin load balancing
        instance = Enum.random(instances)
        {:reply, {:ok, instance}, state}
    end
  end

  def handle_call({:deregister, name, instance}, _from, state) do
    instances = Map.get(state, name, [])
    new_instances = Enum.reject(instances, &match?(^instance, &1))
    {:reply, :ok, Map.put(state, name, new_instances)}
  end
end
```

---

## สรุป

```
Microservices Communication Patterns
═════════════════════════════════════════════════════════
Synchronous (request/response)
  REST (Req)           → External APIs, simple integration
  gRPC (Protobuf)     → High-performance, typed contracts
  GraphQL             → Flexible queries

Asynchronous (fire and forget / event-driven)
  Phoenix PubSub       → In-process events (single node)
  RabbitMQ (AMQP)     → Cross-service events (durable)
  Kafka               → High-throughput event streaming

Resilience Patterns
  Circuit Breaker     → Prevent cascading failures
  Retry with backoff  → Handle transient errors
  Timeout             → Prevent resource exhaustion
  Fallback            → Graceful degradation

API Gateway
  Single entry point
  ├── Authentication/Authorization
  ├── Rate limiting
  ├── Request routing
  ├── Response transformation
  └── Logging & metrics

Choosing Communication
  Same service       → Function call
  Real-time update   → Phoenix PubSub
  Background task    → Oban (PostgreSQL)
  Cross-service sync → gRPC or REST
  Cross-service async → RabbitMQ
  External API       → Req + Circuit Breaker
═════════════════════════════════════════════════════════
```

---

*ก่อนหน้า: [Part 68 - Content Management System](part_68.md) | ต่อไป: [Part 70 - Advanced Authentication Patterns](part_70.md)*
