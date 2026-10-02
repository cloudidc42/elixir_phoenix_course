# Part 57: Event Sourcing (การจัดการ Events เป็นแหล่งข้อมูล)

## เป้าหมายการเรียนรู้

หลังจากจบ Part นี้ คุณจะสามารถ:
- เข้าใจ Event Sourcing pattern
- สร้าง event store ด้วย PostgreSQL
- Rebuild state จาก events
- Event projections

---

## 1. Event Sourcing คืออะไร

```
Traditional: เก็บ state ปัจจุบัน
  Users table: {id: 1, name: "Alice", balance: 1000}

Event Sourcing: เก็บ sequence of events
  Events:
  1. UserRegistered {name: "Alice", balance: 0}
  2. MoneyDeposited {amount: 500}
  3. MoneyDeposited {amount: 500}
  4. MoneyWithdrawn {amount: 200} (ถ้า balance 1000 ลบ 200 = 800?)
  
  ไม่ถูก เพราะ 500 + 500 - 200 = 800 ไม่ใช่ 1000
  
  คำตอบ: Rebuild จาก events เสมอ
```

---

## 2. Event Store Schema

```elixir
defmodule EventStore.Events.Migration do
  use Ecto.Migration

  def change do
    create table(:event_store_events, primary_key: false) do
      add :id, :uuid, primary_key: true, default: fragment("gen_random_uuid()")
      add :stream_id, :string, null: false       # e.g., "account-123"
      add :stream_version, :integer, null: false  # sequence per stream
      add :event_type, :string, null: false       # "MoneyDeposited"
      add :data, :map, null: false                # event payload
      add :metadata, :map, default: %{}           # correlation_id, causation_id, user_id
      add :occurred_at, :utc_datetime_usec, null: false
      timestamps(updated_at: false)
    end

    create unique_index(:event_store_events, [:stream_id, :stream_version])
    create index(:event_store_events, [:stream_id, :occurred_at])
    create index(:event_store_events, [:event_type])
  end
end
```

---

## 3. Event Store Implementation

```elixir
defmodule EventStore.Store do
  alias EventStore.Repo
  alias EventStore.Events.Event
  import Ecto.Query

  def append_events(stream_id, events, expected_version \\ :any) do
    Repo.transaction(fn ->
      current_version = get_stream_version(stream_id)

      case expected_version do
        :any -> :ok
        :no_stream when current_version == 0 -> :ok
        v when v == current_version -> :ok
        _ ->
          Repo.rollback({:wrong_expected_version, current_version, expected_version})
      end

      events_to_insert = events
      |> Enum.with_index(current_version + 1)
      |> Enum.map(fn {event, version} ->
        %{
          stream_id: stream_id,
          stream_version: version,
          event_type: event.__struct__ |> Module.split() |> List.last(),
          data: Map.from_struct(event),
          metadata: event.__metadata__ || %{},
          occurred_at: DateTime.utc_now()
        }
      end)

      {count, inserted} = Repo.insert_all(Event, events_to_insert, returning: true)
      inserted
    end)
  end

  def read_stream(stream_id, opts \\ []) do
    from_version = Keyword.get(opts, :from_version, 0)
    limit = Keyword.get(opts, :limit, nil)

    query = from(e in Event,
      where: e.stream_id == ^stream_id
        and e.stream_version > ^from_version,
      order_by: [asc: e.stream_version]
    )

    query = if limit, do: limit(query, ^limit), else: query
    Repo.all(query)
  end

  def read_all_events(event_types \\ [], opts \\ []) do
    from_position = Keyword.get(opts, :from_position, 0)

    query = from(e in Event,
      where: e.id > ^from_position,
      order_by: [asc: e.inserted_at]
    )

    query = if event_types != [] do
      where(query, [e], e.event_type in ^event_types)
    else
      query
    end

    Repo.all(query)
  end

  defp get_stream_version(stream_id) do
    Repo.one(
      from(e in Event,
        where: e.stream_id == ^stream_id,
        select: max(e.stream_version)
      )
    ) || 0
  end
end
```

---

## 4. Aggregate Pattern

```elixir
defmodule BankAccount do
  defstruct [:id, :owner, :balance, :status, version: 0]

  # Commands
  def open_account(%{owner: owner, initial_deposit: amount}) do
    if amount >= 0 do
      {:ok, [
        %BankAccount.Events.AccountOpened{
          owner: owner,
          initial_balance: amount
        }
      ]}
    else
      {:error, "Initial deposit cannot be negative"}
    end
  end

  def deposit(%BankAccount{status: "closed"}, _amount) do
    {:error, "Account is closed"}
  end

  def deposit(%BankAccount{} = account, amount) when amount > 0 do
    {:ok, [%BankAccount.Events.MoneyDeposited{
      amount: amount,
      balance_after: account.balance + amount
    }]}
  end

  def withdraw(%BankAccount{balance: balance}, amount) when amount > balance do
    {:error, "Insufficient funds"}
  end

  def withdraw(%BankAccount{}, amount) when amount > 0 do
    {:ok, [%BankAccount.Events.MoneyWithdrawn{amount: amount}]}
  end

  # Apply events to rebuild state
  def apply(%BankAccount{} = state, %BankAccount.Events.AccountOpened{} = event) do
    %{state |
      owner: event.owner,
      balance: event.initial_balance,
      status: "open",
      version: state.version + 1
    }
  end

  def apply(%BankAccount{} = state, %BankAccount.Events.MoneyDeposited{} = event) do
    %{state | balance: state.balance + event.amount, version: state.version + 1}
  end

  def apply(%BankAccount{} = state, %BankAccount.Events.MoneyWithdrawn{} = event) do
    %{state | balance: state.balance - event.amount, version: state.version + 1}
  end

  # Load aggregate from event store
  def load(account_id) do
    events = EventStore.Store.read_stream("account-#{account_id}")

    initial = %BankAccount{id: account_id}
    Enum.reduce(events, initial, fn event, state ->
      domain_event = deserialize_event(event)
      apply(state, domain_event)
    end)
  end

  defp deserialize_event(%{event_type: type, data: data}) do
    module = Module.concat(BankAccount.Events, type)
    struct(module, data)
  end
end
```

---

## 5. Projections (Read Models)

```elixir
defmodule BankAccount.Projections.AccountSummary do
  use GenServer
  alias EventStore.Store

  # Read model: precomputed view
  defmodule State do
    defstruct [:account_id, :owner, :balance, :transaction_count, :last_transaction_at]
  end

  def start_link(account_id) do
    GenServer.start_link(__MODULE__, account_id, name: via_name(account_id))
  end

  def init(account_id) do
    state = rebuild_projection(account_id)
    # Subscribe to new events
    Phoenix.PubSub.subscribe(MyApp.PubSub, "account-#{account_id}")
    {:ok, state}
  end

  def get_summary(account_id) do
    GenServer.call(via_name(account_id), :get_summary)
  end

  def handle_call(:get_summary, _from, state) do
    {:reply, state, state}
  end

  def handle_info({:event, event}, state) do
    new_state = apply_event(state, event)
    {:noreply, new_state}
  end

  defp rebuild_projection(account_id) do
    events = Store.read_stream("account-#{account_id}")
    Enum.reduce(events, %State{account_id: account_id}, &apply_event(&2, &1))
  end

  defp apply_event(state, %{event_type: "AccountOpened", data: data}) do
    %{state | owner: data["owner"], balance: data["initial_balance"], transaction_count: 0}
  end

  defp apply_event(state, %{event_type: "MoneyDeposited", data: data}) do
    %{state |
      balance: state.balance + data["amount"],
      transaction_count: (state.transaction_count || 0) + 1,
      last_transaction_at: DateTime.utc_now()
    }
  end

  defp apply_event(state, %{event_type: "MoneyWithdrawn", data: data}) do
    %{state |
      balance: state.balance - data["amount"],
      transaction_count: (state.transaction_count || 0) + 1,
      last_transaction_at: DateTime.utc_now()
    }
  end

  defp apply_event(state, _event), do: state

  defp via_name(account_id), do: {:via, Registry, {MyApp.Registry, {:account_summary, account_id}}}
end
```

---

## 6. Command Handler

```elixir
defmodule BankAccount.CommandHandler do
  alias EventStore.Store
  alias BankAccount

  def handle({:open_account, attrs}) do
    account_id = Ecto.UUID.generate()

    with {:ok, events} <- BankAccount.open_account(attrs),
         {:ok, stored_events} <- Store.append_events("account-#{account_id}", events, :no_stream) do
      broadcast_events("account-#{account_id}", stored_events)
      {:ok, account_id}
    end
  end

  def handle({:deposit_money, account_id, amount}) do
    account = BankAccount.load(account_id)

    with {:ok, events} <- BankAccount.deposit(account, amount),
         {:ok, stored_events} <- Store.append_events(
           "account-#{account_id}",
           events,
           account.version
         ) do
      broadcast_events("account-#{account_id}", stored_events)
      {:ok, account_id}
    end
  end

  def handle({:withdraw_money, account_id, amount}) do
    account = BankAccount.load(account_id)

    with {:ok, events} <- BankAccount.withdraw(account, amount),
         {:ok, stored_events} <- Store.append_events(
           "account-#{account_id}",
           events,
           account.version
         ) do
      broadcast_events("account-#{account_id}", stored_events)
      {:ok, account_id}
    end
  end

  defp broadcast_events(stream_id, events) do
    Enum.each(events, fn event ->
      Phoenix.PubSub.broadcast(MyApp.PubSub, stream_id, {:event, event})
    end)
  end
end
```

---

## สรุป

```
Event Sourcing:
├── เก็บ events แทน state
├── Rebuild state โดย replay events
└── Audit log ฟรี (every change is recorded)

Components:
├── Event Store: append-only log
├── Aggregate: business logic + apply
├── Command Handler: validates and persists
└── Projections: read models from events

Trade-offs:
├── + Perfect audit trail
├── + Time travel (replay to any point)
├── + Event-driven naturally
└── - More complex than CRUD
└── - Eventual consistency for projections
```

---

*ก่อนหน้า: [Part 56](part_56.md) | ต่อไป: [Part 58 - CQRS Pattern](part_58.md)*
