# Part 92: Distributed Elixir (Elixir แบบกระจาย)

## เป้าหมายการเรียนรู้

หลังจากจบ Part นี้ คุณจะสามารถ:
- เชื่อมต่อ Elixir nodes
- Global process registration
- Distributed PubSub
- Horde สำหรับ distributed supervisor

---

## 1. Node Connection

```elixir
# เริ่ม nodes สองตัว
# Terminal 1:
iex --name node1@127.0.0.1 --cookie secret_cookie -S mix

# Terminal 2:
iex --name node2@127.0.0.1 --cookie secret_cookie -S mix

# เชื่อมต่อ nodes
Node.connect(:"node1@127.0.0.1")
Node.list()  # => [:"node1@127.0.0.1"]

# รัน function บน node อื่น
Node.spawn(:"node1@127.0.0.1", fn ->
  IO.puts("Hello from #{Node.self()}")
end)

# ส่ง message ไปยัง process บน node อื่น
send({:my_process, :"node1@127.0.0.1"}, :hello)
```

---

## 2. libcluster - Auto Node Discovery

```elixir
# mix.exs
{:libcluster, "~> 3.3"}

# config/config.exs
config :libcluster,
  topologies: [
    k8s: [
      strategy: Cluster.Strategy.Kubernetes,
      config: [
        mode: :ip,
        kubernetes_node_basename: "my_app",
        kubernetes_selector: "app=my-app",
        polling_interval: 10_000
      ]
    ],
    # สำหรับ development - Gossip protocol
    gossip: [
      strategy: Cluster.Strategy.Gossip,
      config: [
        port: 45892,
        if_addr: "0.0.0.0",
        multicast_addr: "230.1.1.251",
        multicast_ttl: 1
      ]
    ]
  ]

# application.ex
children = [
  {Cluster.Supervisor, [Application.get_env(:libcluster, :topologies), [name: MyApp.ClusterSupervisor]]}
]
```

---

## 3. Horde - Distributed Supervisor

```elixir
# mix.exs: {:horde, "~> 0.8"}

defmodule MyApp.HordeRegistry do
  use Horde.Registry

  def start_link(_) do
    Horde.Registry.start_link(__MODULE__, [keys: :unique], name: __MODULE__)
  end

  def init(init_arg) do
    [members: members()]
    |> Keyword.merge(init_arg)
    |> Horde.Registry.init()
  end

  defp members do
    [Node.self() | Node.list()]
    |> Enum.map(fn node -> {__MODULE__, node} end)
  end
end

defmodule MyApp.HordeSupervisor do
  use Horde.DynamicSupervisor

  def start_link(_) do
    Horde.DynamicSupervisor.start_link(__MODULE__, [strategy: :one_for_one], name: __MODULE__)
  end

  def init(init_arg) do
    [members: members()]
    |> Keyword.merge(init_arg)
    |> Horde.DynamicSupervisor.init()
  end

  defp members do
    [Node.self() | Node.list()]
    |> Enum.map(fn node -> {__MODULE__, node} end)
  end

  def start_worker(id, opts) do
    child_spec = {MyApp.Worker, Keyword.put(opts, :id, id)}
    Horde.DynamicSupervisor.start_child(__MODULE__, child_spec)
  end
end

# Worker registers itself globally via Horde
defmodule MyApp.Worker do
  use GenServer

  def start_link(opts) do
    id = Keyword.fetch!(opts, :id)
    GenServer.start_link(__MODULE__, opts,
      name: {:via, Horde.Registry, {MyApp.HordeRegistry, id}}
    )
  end

  # Only one instance runs across the cluster
  # If a node fails, Horde restarts it on another node
end
```

---

## 4. Distributed PubSub dengan Phoenix

```elixir
# Phoenix PubSub already works across nodes when configured
# config/config.exs
config :my_app, MyAppWeb.Endpoint,
  pubsub_server: MyApp.PubSub

# application.ex
children = [
  {Phoenix.PubSub, name: MyApp.PubSub}  # auto-distributed to all nodes
]

# Broadcast from any node - all nodes' subscribers receive it
Phoenix.PubSub.broadcast(MyApp.PubSub, "room:123", {:new_message, message})

# Subscribe on any node
Phoenix.PubSub.subscribe(MyApp.PubSub, "room:123")
```

---

## 5. Global Process Registration

```elixir
# :global module - built-in distributed registry
defmodule MyApp.GlobalWorker do
  use GenServer

  def start_link(name) do
    GenServer.start_link(__MODULE__, name,
      name: {:global, {:worker, name}}
    )
  end

  # Find anywhere in cluster
  def find(name) do
    :global.whereis_name({:worker, name})
  end

  # Call from any node
  def get_state(name) do
    GenServer.call({:global, {:worker, name}}, :get_state)
  end
end

# :global.register_name/2 for manual registration
:global.register_name(:my_unique_process, self())
:global.whereis_name(:my_unique_process)  # works from any node
```

---

## 6. Node Monitoring

```elixir
defmodule MyApp.NodeMonitor do
  use GenServer
  require Logger

  def start_link(_) do
    GenServer.start_link(__MODULE__, [], name: __MODULE__)
  end

  def init([]) do
    :net_kernel.monitor_nodes(true, [node_type: :all])
    {:ok, %{nodes: Node.list()}}
  end

  def handle_info({:nodeup, node, _info}, state) do
    Logger.info("Node joined cluster: #{node}")
    # Update Horde members
    Horde.DynamicSupervisor.set_members(
      MyApp.HordeSupervisor,
      Enum.map([Node.self() | Node.list()], &{MyApp.HordeSupervisor, &1})
    )
    {:noreply, %{state | nodes: [node | state.nodes]}}
  end

  def handle_info({:nodedown, node, _info}, state) do
    Logger.warning("Node left cluster: #{node}")
    {:noreply, %{state | nodes: List.delete(state.nodes, node)}}
  end

  def connected_nodes, do: GenServer.call(__MODULE__, :nodes)
  def handle_call(:nodes, _from, state), do: {:reply, state.nodes, state}
end
```

---

## สรุป

```
Distributed Elixir Components:
├── libcluster: auto node discovery
├── Horde: distributed supervisor + registry
├── Phoenix.PubSub: cluster-wide pub/sub
└── :global: built-in process registry

Node Strategies:
├── Kubernetes: label selector
├── Gossip: UDP multicast (LAN)
├── DNS: hostname polling
└── Epmd: manual list

CAP Theorem in Elixir:
├── Partition tolerance: BEAM handles it
├── Choose AP: Horde default (availability)
└── Consider CP: :global (consistency)
```

---

*ก่อนหน้า: [Part 91](part_91.md) | ต่อไป: [Part 93 - LiveView Advanced](part_93.md)*
