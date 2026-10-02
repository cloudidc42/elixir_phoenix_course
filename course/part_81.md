# Part 81: WebSockets Advanced (WebSockets ขั้นสูง)

## เป้าหมายการเรียนรู้

หลังจากจบ Part นี้ คุณจะสามารถ:
- Custom Channel implementations
- Rate limiting ใน Channels
- Multiplexing หลาย topics
- Message buffering สำหรับ offline clients

---

## 1. Custom Channel with Rate Limiting

```elixir
defmodule MyAppWeb.ChatChannel do
  use Phoenix.Channel
  alias MyApp.RateLimiter

  @message_limit 10  # 10 messages per 10 seconds
  @window_ms 10_000

  def join("room:" <> room_id, _params, socket) do
    if MyApp.Rooms.member?(socket.assigns.user_id, room_id) do
      send(self(), {:after_join, room_id})
      {:ok, assign(socket, room_id: room_id, message_count: 0, window_start: System.monotonic_time(:millisecond))}
    else
      {:error, %{reason: "unauthorized"}}
    end
  end

  def handle_info({:after_join, room_id}, socket) do
    # Send recent messages
    messages = MyApp.Chat.recent_messages(room_id, 50)
    push(socket, "history", %{messages: messages})

    # Track online presence
    {:ok, _} = MyApp.Presence.track(socket, socket.assigns.user_id, %{
      name: socket.assigns.user_name,
      online_at: System.os_time(:second)
    })

    presences = MyApp.Presence.list(socket)
    push(socket, "presence_state", presences)

    {:noreply, socket}
  end

  def handle_in("message", %{"body" => body}, socket) do
    case check_rate_limit(socket) do
      {:ok, socket} ->
        message = %{
          id: Ecto.UUID.generate(),
          body: body,
          user_id: socket.assigns.user_id,
          user_name: socket.assigns.user_name,
          inserted_at: DateTime.utc_now()
        }

        # Persist message
        Task.start(fn -> MyApp.Chat.store_message(socket.assigns.room_id, message) end)

        # Broadcast to all in room
        broadcast!(socket, "new_message", message)

        {:reply, :ok, socket}

      {:error, :rate_limited} ->
        {:reply, {:error, %{reason: "Rate limit exceeded"}}, socket}
    end
  end

  defp check_rate_limit(socket) do
    now = System.monotonic_time(:millisecond)
    window_start = socket.assigns.window_start

    if now - window_start > @window_ms do
      # Reset window
      {:ok, assign(socket, message_count: 1, window_start: now)}
    else
      count = socket.assigns.message_count + 1
      if count <= @message_limit do
        {:ok, assign(socket, message_count: count)}
      else
        {:error, :rate_limited}
      end
    end
  end
end
```

---

## 2. Message Buffering for Reconnection

```elixir
defmodule MyApp.MessageBuffer do
  use GenServer

  @buffer_ttl :timer.minutes(5)
  @max_buffer 100

  def start_link(_) do
    GenServer.start_link(__MODULE__, %{}, name: __MODULE__)
  end

  def push(user_id, message) do
    GenServer.cast(__MODULE__, {:push, user_id, message})
  end

  def flush(user_id) do
    GenServer.call(__MODULE__, {:flush, user_id})
  end

  def init(_), do: {:ok, %{}}

  def handle_cast({:push, user_id, message}, state) do
    messages = Map.get(state, user_id, [])
    limited = Enum.take([message | messages], @max_buffer)

    # Schedule cleanup
    Process.send_after(self(), {:cleanup, user_id}, @buffer_ttl)

    {:noreply, Map.put(state, user_id, limited)}
  end

  def handle_call({:flush, user_id}, _from, state) do
    {messages, new_state} = Map.pop(state, user_id, [])
    {:reply, Enum.reverse(messages), new_state}
  end

  def handle_info({:cleanup, user_id}, state) do
    {:noreply, Map.delete(state, user_id)}
  end
end

# In channel: deliver buffered messages on reconnect
def handle_info({:after_join, _room_id}, socket) do
  buffered = MyApp.MessageBuffer.flush(socket.assigns.user_id)
  Enum.each(buffered, fn msg ->
    push(socket, "buffered_message", msg)
  end)

  {:noreply, socket}
end
```

---

## 3. Channel Broadcasting with Filters

```elixir
defmodule MyAppWeb.NotificationChannel do
  use Phoenix.Channel

  def join("notifications:" <> user_id, _params, socket) do
    if socket.assigns.user_id == String.to_integer(user_id) do
      {:ok, socket}
    else
      {:error, %{reason: "unauthorized"}}
    end
  end

  def handle_in("subscribe", %{"topics" => topics}, socket) do
    # Subscribe to specific notification types
    socket = assign(socket, :subscribed_topics, topics)
    {:reply, :ok, socket}
  end

  # Server-side: targeted broadcast
  def notify_user(user_id, notification) do
    topic = notification.type

    MyAppWeb.Endpoint.broadcast(
      "notifications:#{user_id}",
      "notification",
      notification
    )
  end
end
```

---

## 4. Channel Authentication Token

```elixir
defmodule MyAppWeb.UserSocket do
  use Phoenix.Socket

  channel "room:*", MyAppWeb.ChatChannel
  channel "notifications:*", MyAppWeb.NotificationChannel
  channel "presence", MyAppWeb.PresenceChannel

  @max_age 24 * 60 * 60  # 24 hours

  def connect(%{"token" => token}, socket, _connect_info) do
    case Phoenix.Token.verify(MyAppWeb.Endpoint, "user socket", token, max_age: @max_age) do
      {:ok, user_id} ->
        user = MyApp.Accounts.get_user!(user_id)
        {:ok, assign(socket, :current_user, user) |> assign(:user_id, user_id) |> assign(:user_name, user.name)}

      {:error, reason} ->
        {:error, reason}
    end
  end

  def connect(%{"api_key" => api_key}, socket, _connect_info) do
    case MyApp.APIKeys.authenticate(api_key) do
      {:ok, user} ->
        {:ok, assign(socket, current_user: user, user_id: user.id, user_name: user.name)}
      {:error, _} ->
        {:error, :unauthorized}
    end
  end

  def connect(_params, _socket, _connect_info) do
    {:error, :missing_token}
  end

  def id(socket), do: "user_socket:#{socket.assigns.user_id}"
end
```

---

## 5. Disconnect and Cleanup

```elixir
defmodule MyAppWeb.ChatChannel do
  use Phoenix.Channel

  def terminate(reason, socket) do
    room_id = socket.assigns.room_id
    user_id = socket.assigns.user_id

    MyApp.Presence.untrack(socket, user_id)

    broadcast!(socket, "user_left", %{user_id: user_id, reason: to_string(reason)})

    :ok
  end

  # Handle disconnect grace period
  def handle_info(:disconnect_check, socket) do
    # Check if user reconnected (presence key still exists elsewhere)
    if MyApp.Presence.get_by_key(socket, socket.assigns.user_id) == %{} do
      # User truly disconnected
      broadcast!(socket, "user_offline", %{user_id: socket.assigns.user_id})
    end
    {:noreply, socket}
  end
end
```

---

## สรุป

```
WebSocket Patterns:
├── Rate limiting: count messages per window
├── Message buffer: store for offline clients
├── Topic filtering: subscribe selectively
└── Auth: Phoenix.Token or API key

Channel Lifecycle:
├── join/3: authorize and setup
├── handle_in/3: receive client message
├── handle_info/2: receive server message
└── terminate/2: cleanup on disconnect

Presence:
├── track: register user online
├── untrack: remove on leave
└── list: get all online users
```

---

*ก่อนหน้า: [Part 80](part_80.md) | ต่อไป: [Part 82 - Plug and Middleware](part_82.md)*
