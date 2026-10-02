# Part 32: Phoenix Channels และ WebSockets

## เป้าหมายการเรียนรู้

หลังจากจบ Part นี้ คุณจะสามารถ:
- เข้าใจ Phoenix Channels architecture
- สร้าง real-time features ด้วย Channels
- จัดการ presence tracking
- สร้าง chat application

---

## 1. Phoenix Channels คืออะไร?

Phoenix Channels ให้ communication แบบ bidirectional ระหว่าง server และ client ผ่าน WebSocket

```
┌──────────────────────────────────────────────────┐
│            Phoenix Channel Architecture          │
│                                                  │
│  Client A ──WS──┐                               │
│  Client B ──WS──┤──> Channel Process ──> Topic  │
│  Client C ──WS──┘         ↑                     │
│                            │ PubSub              │
│                    Server Logic                  │
└──────────────────────────────────────────────────┘
```

### Topics

Topics คือ unique identifiers สำหรับ channel เช่น:
- `"chat:general"` - general chat room
- `"chat:room:123"` - specific room
- `"user:123"` - user-specific channel

---

## 2. สร้าง Channel

```elixir
# lib/my_app_web/channels/chat_channel.ex
defmodule MyAppWeb.ChatChannel do
  use MyAppWeb, :channel

  alias MyApp.Chat
  alias MyAppWeb.Presence

  @impl true
  def join("chat:" <> room_id, payload, socket) do
    if authorized?(payload, room_id) do
      send(self(), :after_join)
      {:ok, assign(socket, room_id: room_id)}
    else
      {:error, %{reason: "unauthorized"}}
    end
  end

  @impl true
  def handle_info(:after_join, socket) do
    # Track presence
    {:ok, _} = Presence.track(socket, socket.assigns.user_id, %{
      online_at: :os.system_time(:millisecond),
      username: socket.assigns.username
    })

    # Send recent messages
    messages = Chat.get_recent_messages(socket.assigns.room_id)
    push(socket, "history", %{messages: messages})

    {:noreply, socket}
  end

  # Handle incoming message
  @impl true
  def handle_in("new_message", %{"body" => body}, socket) do
    user = socket.assigns.current_user

    case Chat.create_message(socket.assigns.room_id, user, body) do
      {:ok, message} ->
        broadcast!(socket, "new_message", %{
          id: message.id,
          body: message.body,
          user: %{id: user.id, username: user.username},
          timestamp: message.inserted_at
        })
        {:reply, :ok, socket}

      {:error, reason} ->
        {:reply, {:error, %{reason: reason}}, socket}
    end
  end

  # Handle typing indicator
  @impl true
  def handle_in("typing", _params, socket) do
    broadcast_from!(socket, "user_typing", %{
      user_id: socket.assigns.user_id,
      username: socket.assigns.username
    })
    {:noreply, socket}
  end

  # Private
  defp authorized?(_payload, _room_id), do: true
end
```

### User Socket

```elixir
# lib/my_app_web/channels/user_socket.ex
defmodule MyAppWeb.UserSocket do
  use Phoenix.Socket

  channel "chat:*", MyAppWeb.ChatChannel
  channel "notification:*", MyAppWeb.NotificationChannel

  @impl true
  def connect(%{"token" => token}, socket, _connect_info) do
    case verify_token(token) do
      {:ok, user_id} ->
        socket = assign(socket, :user_id, user_id)
        {:ok, socket}

      {:error, _reason} ->
        :error
    end
  end

  @impl true
  def id(socket), do: "user_socket:#{socket.assigns.user_id}"

  defp verify_token(token) do
    Phoenix.Token.verify(MyAppWeb.Endpoint, "user socket", token, max_age: 86400)
  end
end
```

---

## 3. JavaScript Client

```javascript
// assets/js/chat.js
import {Socket} from "phoenix"

let socket = new Socket("/socket", {
  params: {token: userToken}
})

socket.connect()

// Join channel
let channel = socket.channel("chat:general", {})

channel.join()
  .receive("ok", resp => {
    console.log("Joined successfully", resp)
  })
  .receive("error", resp => {
    console.log("Unable to join", resp)
  })

// Receive history
channel.on("history", ({messages}) => {
  messages.forEach(displayMessage)
})

// Receive new message
channel.on("new_message", message => {
  displayMessage(message)
})

// Typing indicator
channel.on("user_typing", ({username}) => {
  showTypingIndicator(username)
})

// Send message
function sendMessage(body) {
  channel.push("new_message", {body})
    .receive("ok", () => console.log("Message sent"))
    .receive("error", ({reason}) => console.error("Error:", reason))
}

// Typing
let typingTimeout
document.getElementById("message-input").addEventListener("keypress", () => {
  channel.push("typing", {})

  clearTimeout(typingTimeout)
  typingTimeout = setTimeout(stopTyping, 3000)
})

function displayMessage(msg) {
  const div = document.createElement("div")
  div.innerHTML = `<strong>${msg.user.username}</strong>: ${msg.body}`
  document.getElementById("messages").appendChild(div)
}

function showTypingIndicator(username) {
  document.getElementById("typing").textContent = `${username} is typing...`
}

function stopTyping() {
  document.getElementById("typing").textContent = ""
}
```

---

## 4. Presence

Phoenix Presence ให้ track ว่า users ไหน online อยู่

```elixir
# lib/my_app_web/channels/presence.ex
defmodule MyAppWeb.Presence do
  use Phoenix.Presence,
    otp_app: :my_app,
    pubsub_server: MyApp.PubSub
end
```

```elixir
# lib/my_app/application.ex
def start(_type, _args) do
  children = [
    # ...
    MyAppWeb.Presence,
    # ...
  ]
  # ...
end
```

### Using Presence

```elixir
defmodule MyAppWeb.RoomChannel do
  use MyAppWeb, :channel
  alias MyAppWeb.Presence

  def join("room:" <> room_id, _params, socket) do
    send(self(), :after_join)
    {:ok, socket}
  end

  def handle_info(:after_join, socket) do
    # Track user presence
    {:ok, _} = Presence.track(socket, socket.assigns.user_id, %{
      online_at: System.system_time(:second),
      user_id: socket.assigns.user_id,
      username: socket.assigns.username,
      status: :online
    })

    # Push current presence state
    push(socket, "presence_state", Presence.list(socket))

    {:noreply, socket}
  end

  def handle_in("update_status", %{"status" => status}, socket) do
    Presence.update(socket, socket.assigns.user_id, fn meta ->
      %{meta | status: status}
    end)
    {:noreply, socket}
  end
end
```

---

## 5. ตัวอย่างจริง: Real-time Collaboration

```elixir
defmodule MyAppWeb.DocumentChannel do
  use MyAppWeb, :channel
  alias MyApp.Documents
  alias MyAppWeb.Presence

  def join("document:" <> doc_id, _params, socket) do
    case Documents.get_document(doc_id) do
      nil ->
        {:error, %{reason: "Document not found"}}

      document ->
        send(self(), :after_join)

        socket =
          socket
          |> assign(:document_id, doc_id)
          |> assign(:document, document)

        {:ok, %{content: document.content}, socket}
    end
  end

  def handle_info(:after_join, socket) do
    {:ok, _} = Presence.track(socket, socket.assigns.user_id, %{
      username: socket.assigns.username,
      cursor_position: 0
    })

    push(socket, "presence_state", Presence.list(socket))
    {:noreply, socket}
  end

  # Handle text operations (Operational Transform)
  def handle_in("operation", %{"op" => op, "version" => version}, socket) do
    doc_id = socket.assigns.document_id

    case Documents.apply_operation(doc_id, op, version) do
      {:ok, new_version, transformed_op} ->
        broadcast!(socket, "operation", %{
          op: transformed_op,
          version: new_version,
          user_id: socket.assigns.user_id
        })
        {:reply, {:ok, %{version: new_version}}, socket}

      {:error, :conflict} ->
        {:reply, {:error, %{reason: "conflict"}}, socket}
    end
  end

  # Cursor position
  def handle_in("cursor", %{"position" => position}, socket) do
    Presence.update(socket, socket.assigns.user_id, fn meta ->
      %{meta | cursor_position: position}
    end)
    {:noreply, socket}
  end
end
```

---

## 6. Channel Testing

```elixir
defmodule MyAppWeb.ChatChannelTest do
  use MyAppWeb.ChannelCase

  setup do
    {:ok, _, socket} =
      MyAppWeb.UserSocket
      |> socket("user_id", %{user_id: 1, username: "Alice"})
      |> subscribe_and_join(MyAppWeb.ChatChannel, "chat:general")

    %{socket: socket}
  end

  test "receives messages", %{socket: socket} do
    push(socket, "new_message", %{body: "Hello!"})
    assert_broadcast "new_message", %{body: "Hello!"}
  end

  test "sends history on join" do
    assert_push "history", %{messages: _messages}
  end
end
```

---

## แบบฝึกหัด

### Exercise 1: Notification Channel
สร้าง channel สำหรับ user notifications:
- User subscribe ด้วย `"notification:{user_id}"`
- Server สามารถ push notifications
- Client สามารถ mark as read

### Exercise 2: Live Scoreboard
สร้าง real-time scoreboard:
- Players join game room
- Score updates broadcast ไปทุกคน
- Presence tracking ว่าใครออนไลน์

---

## สรุป

```
Phoenix Channels:
├── WebSocket-based communication
├── Topics: "namespace:id"
├── Join/Leave/Handle events
└── Broadcast, push, broadcast_from!

Presence:
├── Track who's online
├── Real-time user status
└── Sync across clients

JavaScript Client:
├── Socket connection
├── Channel join
├── push/receive events
└── Presence hooks
```

---

*ก่อนหน้า: [Part 31](part_31.md) | ต่อไป: [Part 33 - Authentication](part_33.md)*
