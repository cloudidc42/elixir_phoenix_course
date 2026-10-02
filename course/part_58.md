# Part 58: CQRS Pattern (Command Query Responsibility Segregation)

## เป้าหมายการเรียนรู้
- เข้าใจแนวคิด CQRS และการแยก Command กับ Query
- สร้าง Command Handlers และ Query Handlers ใน Elixir
- ใช้ ETS สำหรับ Read Model Projections
- จัดการ Eventual Consistency
- ผสาน CQRS กับ Phoenix
- เขียน Tests สำหรับระบบ CQRS

---

## 1. แนวคิด CQRS คืออะไร?

CQRS (Command Query Responsibility Segregation) เป็น pattern ที่แยกการ **เขียนข้อมูล (Commands)** ออกจากการ **อ่านข้อมูล (Queries)** โดยชัดเจน ทำให้ระบบ scale ได้ดีขึ้น และ optimize แต่ละส่วนได้อย่างอิสระ

```
Write Side (Commands)          Read Side (Queries)
┌─────────────────┐            ┌─────────────────┐
│   Command Bus   │            │   Query Bus     │
│   ┌─────────┐  │            │   ┌─────────┐  │
│   │Command  │  │  Events    │   │  Read   │  │
│   │Handler  │──┼──────────► │   │ Model   │  │
│   └─────────┘  │            │   └─────────┘  │
│   Write DB     │            │   Read Store    │
└─────────────────┘            └─────────────────┘
```

ข้อดีของ CQRS:
- **Performance**: Read side ปรับแต่งสำหรับ query โดยเฉพาะ
- **Scalability**: Scale Read และ Write แยกกันได้
- **Simplicity**: แต่ละส่วนมีความรับผิดชอบชัดเจน
- **Audit Trail**: บันทึก Commands ไว้ได้ทั้งหมด

---

## 2. โครงสร้างโปรเจกต์ CQRS

```elixir
# mix.exs
defmodule CqrsExample.MixProject do
  use Mix.Project

  def project do
    [
      app: :cqrs_example,
      version: "0.1.0",
      elixir: "~> 1.15",
      deps: deps()
    ]
  end

  defp deps do
    [
      {:phoenix, "~> 1.7"},
      {:ecto_sql, "~> 3.10"},
      {:postgrex, ">= 0.0.0"},
      {:jason, "~> 1.4"}
    ]
  end
end
```

โครงสร้างไฟล์:
```
lib/
  cqrs_example/
    commands/           # Command definitions
      create_order.ex
      update_order.ex
    command_handlers/   # Command handlers
      order_handler.ex
    queries/            # Query definitions
      get_order.ex
      list_orders.ex
    query_handlers/     # Query handlers
      order_query_handler.ex
    projections/        # Read model projections
      order_projection.ex
    events/             # Domain events
      order_events.ex
```

---

## 3. Commands และ Command Handlers

Command คือ intent ที่ต้องการทำบางอย่างกับระบบ ชื่อ Command ควรเป็น imperative verb เช่น `CreateOrder`, `UpdateOrder`

```elixir
# lib/cqrs_example/commands/create_order.ex
defmodule CqrsExample.Commands.CreateOrder do
  @moduledoc """
  Command สำหรับสร้าง Order ใหม่
  """

  @enforce_keys [:order_id, :customer_id, :items]
  defstruct [:order_id, :customer_id, :items, :metadata]

  @type t :: %__MODULE__{
    order_id: String.t(),
    customer_id: String.t(),
    items: list(map()),
    metadata: map() | nil
  }

  def new(params) do
    %__MODULE__{
      order_id: params[:order_id] || UUID.uuid4(),
      customer_id: params.customer_id,
      items: params.items,
      metadata: params[:metadata]
    }
  end

  def validate(%__MODULE__{} = cmd) do
    errors = []
    errors = if is_nil(cmd.customer_id), do: ["customer_id is required" | errors], else: errors
    errors = if Enum.empty?(cmd.items), do: ["items cannot be empty" | errors], else: errors

    case errors do
      [] -> {:ok, cmd}
      errs -> {:error, errs}
    end
  end
end
```

```elixir
# lib/cqrs_example/commands/update_order_status.ex
defmodule CqrsExample.Commands.UpdateOrderStatus do
  @enforce_keys [:order_id, :status]
  defstruct [:order_id, :status, :reason, :updated_by]

  @valid_statuses ~w(pending confirmed shipped delivered cancelled)a

  def validate(%__MODULE__{} = cmd) do
    if cmd.status in @valid_statuses do
      {:ok, cmd}
    else
      {:error, ["Invalid status: #{cmd.status}"]}
    end
  end
end
```

```elixir
# lib/cqrs_example/command_handlers/order_handler.ex
defmodule CqrsExample.CommandHandlers.OrderHandler do
  @moduledoc """
  Handler สำหรับจัดการ Order Commands
  """

  alias CqrsExample.Commands.{CreateOrder, UpdateOrderStatus}
  alias CqrsExample.Events.{OrderCreated, OrderStatusUpdated}
  alias CqrsExample.EventStore
  alias CqrsExample.Repo

  # Handle CreateOrder command
  def handle(%CreateOrder{} = cmd) do
    with {:ok, validated_cmd} <- CreateOrder.validate(cmd),
         {:ok, order} <- persist_order(validated_cmd),
         event = build_order_created_event(order),
         {:ok, _} <- EventStore.append(event) do
      # Publish event for projections to update
      Phoenix.PubSub.broadcast(
        CqrsExample.PubSub,
        "order_events",
        {:order_created, event}
      )
      {:ok, order}
    end
  end

  # Handle UpdateOrderStatus command
  def handle(%UpdateOrderStatus{} = cmd) do
    with {:ok, validated_cmd} <- UpdateOrderStatus.validate(cmd),
         {:ok, order} <- find_order(validated_cmd.order_id),
         {:ok, updated_order} <- update_order_status(order, validated_cmd),
         event = build_status_updated_event(updated_order, validated_cmd),
         {:ok, _} <- EventStore.append(event) do
      Phoenix.PubSub.broadcast(
        CqrsExample.PubSub,
        "order_events",
        {:order_status_updated, event}
      )
      {:ok, updated_order}
    end
  end

  # Private functions

  defp persist_order(cmd) do
    %CqrsExample.Order{}
    |> CqrsExample.Order.changeset(%{
      id: cmd.order_id,
      customer_id: cmd.customer_id,
      items: cmd.items,
      status: :pending
    })
    |> Repo.insert()
  end

  defp find_order(order_id) do
    case Repo.get(CqrsExample.Order, order_id) do
      nil -> {:error, :not_found}
      order -> {:ok, order}
    end
  end

  defp update_order_status(order, cmd) do
    order
    |> CqrsExample.Order.changeset(%{status: cmd.status})
    |> Repo.update()
  end

  defp build_order_created_event(order) do
    %OrderCreated{
      event_id: UUID.uuid4(),
      order_id: order.id,
      customer_id: order.customer_id,
      items: order.items,
      occurred_at: DateTime.utc_now()
    }
  end

  defp build_status_updated_event(order, cmd) do
    %OrderStatusUpdated{
      event_id: UUID.uuid4(),
      order_id: order.id,
      previous_status: order.status,
      new_status: cmd.status,
      reason: cmd.reason,
      occurred_at: DateTime.utc_now()
    }
  end
end
```

---

## 4. Command Bus

Command Bus ทำหน้าที่เป็น dispatcher ที่รับ Command แล้วส่งไปยัง Handler ที่เหมาะสม

```elixir
# lib/cqrs_example/command_bus.ex
defmodule CqrsExample.CommandBus do
  @moduledoc """
  Central dispatcher สำหรับ Commands
  """

  alias CqrsExample.Commands.{CreateOrder, UpdateOrderStatus}
  alias CqrsExample.CommandHandlers.OrderHandler

  # Map ระหว่าง Command type กับ Handler
  @handlers %{
    CreateOrder => OrderHandler,
    UpdateOrderStatus => OrderHandler
  }

  def dispatch(command) do
    case Map.get(@handlers, command.__struct__) do
      nil ->
        {:error, {:no_handler, "No handler found for #{inspect(command.__struct__)}"}}

      handler ->
        execute_with_middleware(handler, command)
    end
  end

  # Middleware pipeline สำหรับ logging, validation, etc.
  defp execute_with_middleware(handler, command) do
    command
    |> log_command()
    |> handler.handle()
    |> log_result()
  end

  defp log_command(command) do
    require Logger
    Logger.info("Dispatching command: #{inspect(command.__struct__)}")
    command
  end

  defp log_result({:ok, _} = result) do
    require Logger
    Logger.info("Command executed successfully")
    result
  end

  defp log_result({:error, reason} = result) do
    require Logger
    Logger.error("Command failed: #{inspect(reason)}")
    result
  end
end
```

---

## 5. Query Handlers

Query Handler ทำหน้าที่อ่านข้อมูลจาก Read Model โดยเฉพาะ ไม่มี side effects

```elixir
# lib/cqrs_example/queries/get_order.ex
defmodule CqrsExample.Queries.GetOrder do
  defstruct [:order_id, :include_items]

  def new(order_id, opts \\ []) do
    %__MODULE__{
      order_id: order_id,
      include_items: Keyword.get(opts, :include_items, true)
    }
  end
end
```

```elixir
# lib/cqrs_example/query_handlers/order_query_handler.ex
defmodule CqrsExample.QueryHandlers.OrderQueryHandler do
  @moduledoc """
  Handler สำหรับ Order Queries - อ่านจาก Read Model (ETS)
  """

  alias CqrsExample.Projections.OrderProjection
  alias CqrsExample.Queries.{GetOrder, ListOrders, GetOrderSummary}

  # Query จาก ETS Read Model (เร็วมาก, ไม่ต้องไปที่ DB)
  def handle(%GetOrder{} = query) do
    case OrderProjection.find(query.order_id) do
      nil -> {:error, :not_found}
      order -> {:ok, maybe_include_items(order, query.include_items)}
    end
  end

  def handle(%ListOrders{} = query) do
    orders = OrderProjection.list(
      customer_id: query.customer_id,
      status: query.status,
      page: query.page,
      per_page: query.per_page
    )
    {:ok, orders}
  end

  def handle(%GetOrderSummary{} = query) do
    summary = OrderProjection.summary(query.customer_id)
    {:ok, summary}
  end

  defp maybe_include_items(order, true), do: order
  defp maybe_include_items(order, false), do: Map.delete(order, :items)
end
```

---

## 6. Read Model Projections ด้วย ETS

ETS (Erlang Term Storage) เหมาะมากสำหรับ Read Model เพราะเร็วและอยู่ใน memory

```elixir
# lib/cqrs_example/projections/order_projection.ex
defmodule CqrsExample.Projections.OrderProjection do
  @moduledoc """
  Read Model สำหรับ Orders โดยใช้ ETS เป็น storage
  
  Projection จะ subscribe ไปยัง order_events และ update ETS table
  เมื่อมี event เข้ามา
  """

  use GenServer
  require Logger

  @table_name :order_read_model
  @customer_index :order_by_customer
  @status_index :order_by_status

  # Client API

  def start_link(opts \\ []) do
    GenServer.start_link(__MODULE__, opts, name: __MODULE__)
  end

  def find(order_id) do
    case :ets.lookup(@table_name, order_id) do
      [{^order_id, order}] -> order
      [] -> nil
    end
  end

  def list(filters) do
    all_orders = :ets.tab2list(@table_name)
    |> Enum.map(fn {_id, order} -> order end)

    all_orders
    |> maybe_filter_by_customer(filters[:customer_id])
    |> maybe_filter_by_status(filters[:status])
    |> paginate(filters[:page], filters[:per_page])
  end

  def summary(customer_id) do
    orders = list(customer_id: customer_id)

    %{
      total_orders: length(orders),
      total_amount: Enum.sum(Enum.map(orders, & &1.total_amount)),
      by_status: Enum.group_by(orders, & &1.status) |> Map.new(fn {k, v} -> {k, length(v)} end)
    }
  end

  # Server callbacks

  @impl true
  def init(_opts) do
    # สร้าง ETS tables
    :ets.new(@table_name, [:named_table, :set, :public, read_concurrency: true])
    :ets.new(@customer_index, [:named_table, :bag, :public, read_concurrency: true])
    :ets.new(@status_index, [:named_table, :bag, :public, read_concurrency: true])

    # Subscribe to events
    Phoenix.PubSub.subscribe(CqrsExample.PubSub, "order_events")

    # Rebuild projection from event store (replay)
    rebuild_from_events()

    Logger.info("OrderProjection started and rebuilt from events")
    {:ok, %{}}
  end

  @impl true
  def handle_info({:order_created, event}, state) do
    order = %{
      id: event.order_id,
      customer_id: event.customer_id,
      items: event.items,
      status: :pending,
      total_amount: calculate_total(event.items),
      created_at: event.occurred_at,
      updated_at: event.occurred_at
    }

    # Insert ลง ETS table หลัก
    :ets.insert(@table_name, {order.id, order})

    # Update indexes
    :ets.insert(@customer_index, {order.customer_id, order.id})
    :ets.insert(@status_index, {order.status, order.id})

    {:noreply, state}
  end

  @impl true
  def handle_info({:order_status_updated, event}, state) do
    case find(event.order_id) do
      nil ->
        Logger.warning("Cannot update projection for unknown order: #{event.order_id}")

      order ->
        # ลบ old status index
        :ets.delete_object(@status_index, {order.status, order.id})

        # Update order ใน ETS
        updated_order = %{order |
          status: event.new_status,
          updated_at: event.occurred_at
        }
        :ets.insert(@table_name, {updated_order.id, updated_order})

        # เพิ่ม new status index
        :ets.insert(@status_index, {event.new_status, updated_order.id})
    end

    {:noreply, state}
  end

  # Private helpers

  defp rebuild_from_events do
    CqrsExample.EventStore.all_events()
    |> Enum.each(fn event ->
      send(self(), {event.type, event})
    end)
  end

  defp calculate_total(items) do
    Enum.sum(Enum.map(items, fn item -> item.price * item.quantity end))
  end

  defp maybe_filter_by_customer(orders, nil), do: orders
  defp maybe_filter_by_customer(orders, customer_id) do
    Enum.filter(orders, & &1.customer_id == customer_id)
  end

  defp maybe_filter_by_status(orders, nil), do: orders
  defp maybe_filter_by_status(orders, status) do
    Enum.filter(orders, & &1.status == status)
  end

  defp paginate(orders, nil, _), do: orders
  defp paginate(orders, page, per_page) do
    per_page = per_page || 20
    offset = (page - 1) * per_page
    Enum.slice(orders, offset, per_page)
  end
end
```

---

## 7. Event Store

```elixir
# lib/cqrs_example/event_store.ex
defmodule CqrsExample.EventStore do
  @moduledoc """
  Simple Event Store สำหรับบันทึก Domain Events
  ใน production ควรใช้ EventStoreDB หรือ library เช่น Commanded
  """

  use GenServer

  def start_link(opts \\ []) do
    GenServer.start_link(__MODULE__, opts, name: __MODULE__)
  end

  def append(event) do
    GenServer.call(__MODULE__, {:append, event})
  end

  def all_events do
    GenServer.call(__MODULE__, :all_events)
  end

  def events_for(aggregate_id) do
    GenServer.call(__MODULE__, {:events_for, aggregate_id})
  end

  @impl true
  def init(_opts) do
    {:ok, %{events: [], sequence: 0}}
  end

  @impl true
  def handle_call({:append, event}, _from, state) do
    sequence = state.sequence + 1
    stored_event = Map.put(event, :sequence, sequence)

    new_state = %{state |
      events: state.events ++ [stored_event],
      sequence: sequence
    }

    {:reply, {:ok, stored_event}, new_state}
  end

  @impl true
  def handle_call(:all_events, _from, state) do
    {:reply, state.events, state}
  end

  @impl true
  def handle_call({:events_for, aggregate_id}, _from, state) do
    events = Enum.filter(state.events, fn event ->
      Map.get(event, :order_id) == aggregate_id
    end)
    {:reply, events, state}
  end
end
```

---

## 8. Eventual Consistency

ใน CQRS, Write Model และ Read Model อาจไม่ sync กันทันที (eventual consistency) ซึ่งเป็นเรื่องปกติ

```elixir
# lib/cqrs_example/consistency_checker.ex
defmodule CqrsExample.ConsistencyChecker do
  @moduledoc """
  ตรวจสอบและ reconcile ความแตกต่างระหว่าง Write Model และ Read Model
  """

  alias CqrsExample.{Repo, Projections.OrderProjection}
  import Ecto.Query

  def check_consistency do
    db_orders = Repo.all(from o in CqrsExample.Order, select: {o.id, o.status})
    read_model_ids = OrderProjection.list([]) |> Enum.map(& &1.id)

    # หา orders ที่อยู่ใน DB แต่ไม่อยู่ใน Read Model
    db_ids = Enum.map(db_orders, fn {id, _} -> id end)
    missing_ids = db_ids -- read_model_ids

    if Enum.empty?(missing_ids) do
      {:ok, :consistent}
    else
      {:inconsistent, missing_ids}
    end
  end

  # Rebuild projection เมื่อพบความไม่สอดคล้อง
  def rebuild_projection do
    GenServer.cast(OrderProjection, :rebuild)
  end
end
```

---

## 9. Integration กับ Phoenix Controller

```elixir
# lib/cqrs_example_web/controllers/order_controller.ex
defmodule CqrsExampleWeb.OrderController do
  use CqrsExampleWeb, :controller

  alias CqrsExample.{CommandBus, QueryBus}
  alias CqrsExample.Commands.{CreateOrder, UpdateOrderStatus}
  alias CqrsExample.Queries.{GetOrder, ListOrders}

  def index(conn, params) do
    query = %ListOrders{
      customer_id: params["customer_id"],
      status: params["status"],
      page: String.to_integer(params["page"] || "1"),
      per_page: String.to_integer(params["per_page"] || "20")
    }

    # Query ไปที่ Read Model (ETS) - เร็วมาก
    case QueryBus.dispatch(query) do
      {:ok, orders} ->
        render(conn, :index, orders: orders)

      {:error, reason} ->
        conn
        |> put_status(:internal_server_error)
        |> json(%{error: inspect(reason)})
    end
  end

  def show(conn, %{"id" => order_id}) do
    query = GetOrder.new(order_id)

    case QueryBus.dispatch(query) do
      {:ok, order} ->
        render(conn, :show, order: order)

      {:error, :not_found} ->
        conn |> put_status(:not_found) |> json(%{error: "Order not found"})
    end
  end

  def create(conn, params) do
    command = CreateOrder.new(%{
      customer_id: params["customer_id"],
      items: params["items"]
    })

    # Command ไปที่ Write Model (DB)
    case CommandBus.dispatch(command) do
      {:ok, order} ->
        conn
        |> put_status(:created)
        |> json(%{order_id: order.id, status: "pending"})

      {:error, errors} ->
        conn
        |> put_status(:unprocessable_entity)
        |> json(%{errors: errors})
    end
  end

  def update_status(conn, %{"id" => order_id} = params) do
    command = %UpdateOrderStatus{
      order_id: order_id,
      status: String.to_existing_atom(params["status"]),
      reason: params["reason"]
    }

    case CommandBus.dispatch(command) do
      {:ok, _order} ->
        json(conn, %{message: "Status updated successfully"})

      {:error, reason} ->
        conn
        |> put_status(:unprocessable_entity)
        |> json(%{error: inspect(reason)})
    end
  end
end
```

---

## 10. Testing CQRS Systems

```elixir
# test/cqrs_example/command_handlers/order_handler_test.exs
defmodule CqrsExample.CommandHandlers.OrderHandlerTest do
  use CqrsExample.DataCase, async: false

  alias CqrsExample.CommandHandlers.OrderHandler
  alias CqrsExample.Commands.{CreateOrder, UpdateOrderStatus}
  alias CqrsExample.Projections.OrderProjection

  setup do
    # Start projection for tests
    start_supervised!(OrderProjection)
    :ok
  end

  describe "handle/1 with CreateOrder" do
    test "สร้าง order สำเร็จ" do
      command = CreateOrder.new(%{
        customer_id: "cust-123",
        items: [%{product_id: "prod-1", quantity: 2, price: 100.0}]
      })

      assert {:ok, order} = OrderHandler.handle(command)
      assert order.customer_id == "cust-123"
      assert order.status == :pending
    end

    test "ล้มเหลวเมื่อไม่มี customer_id" do
      command = %CreateOrder{
        order_id: "order-1",
        customer_id: nil,
        items: [%{product_id: "prod-1", quantity: 1, price: 50.0}]
      }

      assert {:error, errors} = OrderHandler.handle(command)
      assert "customer_id is required" in errors
    end

    test "ล้มเหลวเมื่อ items ว่างเปล่า" do
      command = %CreateOrder{
        order_id: "order-1",
        customer_id: "cust-123",
        items: []
      }

      assert {:error, errors} = OrderHandler.handle(command)
      assert "items cannot be empty" in errors
    end
  end

  describe "handle/1 with UpdateOrderStatus" do
    setup do
      # สร้าง order ก่อนทดสอบ
      create_cmd = CreateOrder.new(%{
        customer_id: "cust-123",
        items: [%{product_id: "prod-1", quantity: 1, price: 100.0}]
      })
      {:ok, order} = OrderHandler.handle(create_cmd)
      {:ok, order: order}
    end

    test "อัพเดต status สำเร็จ", %{order: order} do
      command = %UpdateOrderStatus{
        order_id: order.id,
        status: :confirmed
      }

      assert {:ok, updated_order} = OrderHandler.handle(command)
      assert updated_order.status == :confirmed
    end

    test "ล้มเหลวเมื่อ status ไม่ถูกต้อง", %{order: order} do
      command = %UpdateOrderStatus{
        order_id: order.id,
        status: :invalid_status
      }

      assert {:error, errors} = OrderHandler.handle(command)
      assert Enum.any?(errors, &String.contains?(&1, "Invalid status"))
    end
  end
end
```

```elixir
# test/cqrs_example/projections/order_projection_test.exs
defmodule CqrsExample.Projections.OrderProjectionTest do
  use ExUnit.Case, async: false

  alias CqrsExample.Projections.OrderProjection
  alias CqrsExample.Events.{OrderCreated, OrderStatusUpdated}

  setup do
    start_supervised!(CqrsExample.PubSub)
    start_supervised!(OrderProjection)
    :ok
  end

  test "projection อัพเดตเมื่อมี OrderCreated event" do
    event = %OrderCreated{
      event_id: "evt-1",
      order_id: "order-1",
      customer_id: "cust-1",
      items: [%{product_id: "prod-1", quantity: 2, price: 50.0}],
      occurred_at: DateTime.utc_now()
    }

    Phoenix.PubSub.broadcast(CqrsExample.PubSub, "order_events", {:order_created, event})

    # รอให้ projection update (eventual consistency)
    Process.sleep(50)

    order = OrderProjection.find("order-1")
    assert order != nil
    assert order.customer_id == "cust-1"
    assert order.status == :pending
    assert order.total_amount == 100.0
  end

  test "projection อัพเดต status เมื่อมี OrderStatusUpdated event" do
    # ก่อนอื่น สร้าง order ใน projection
    create_event = %OrderCreated{
      event_id: "evt-1",
      order_id: "order-2",
      customer_id: "cust-1",
      items: [],
      occurred_at: DateTime.utc_now()
    }
    Phoenix.PubSub.broadcast(CqrsExample.PubSub, "order_events", {:order_created, create_event})
    Process.sleep(50)

    # จากนั้น update status
    status_event = %OrderStatusUpdated{
      event_id: "evt-2",
      order_id: "order-2",
      previous_status: :pending,
      new_status: :confirmed,
      occurred_at: DateTime.utc_now()
    }
    Phoenix.PubSub.broadcast(CqrsExample.PubSub, "order_events", {:order_status_updated, status_event})
    Process.sleep(50)

    order = OrderProjection.find("order-2")
    assert order.status == :confirmed
  end
end
```

---

## สรุป

```
CQRS Flow Summary:
                        
  User Request
      │
      ▼
  ┌─────────┐    Write     ┌─────────────────┐
  │Phoenix  │─────────────►│  Command Bus    │
  │         │              │  CommandHandler │
  │Controller│              │  Write DB       │
  │         │              └────────┬────────┘
  │         │                       │ Events
  │         │                       ▼
  │         │              ┌────────────────┐
  │         │    Read      │  Projections   │
  │         │◄─────────────│  (ETS Tables)  │
  └─────────┘              └────────────────┘

Key Concepts:
- Commands: เปลี่ยน state (write)
- Queries: อ่าน state (read, no side effects)  
- Events: บันทึกสิ่งที่เกิดขึ้น
- Projections: สร้าง Read Model จาก Events
- ETS: fast in-memory read store
- Eventual Consistency: Read Model sync หลัง event
```

CQRS เหมาะกับระบบที่ Read/Write มี requirement ต่างกัน เช่น dashboard ที่ query ซับซ้อนแต่ write ง่าย หรือระบบที่ต้องการ audit trail ครบถ้วน

---

*ก่อนหน้า: [Part 57](part_57.md) | ต่อไป: [Part 59 - Stream Processing](part_59.md)*
