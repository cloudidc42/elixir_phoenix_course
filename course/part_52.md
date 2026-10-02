# Part 52: Real-time Chat Application (แอปพลิเคชันแชทแบบ Real-time)

## เป้าหมายการเรียนรู้

หลังจากจบ Part นี้ คุณจะสามารถ:
- สร้าง chat application ด้วย Phoenix Channels
- จัดการ rooms และ direct messages
- Online presence tracking
- Message history

---

## 1. โครงสร้างระบบ

```
Chat System:
├── Rooms (public channels)
├── Direct Messages (private)
├── Message history (database)
├── Online users (Presence)
└── Read receipts
```

---

## 2. Database Schema

```elixir
# migrations
create table(:chat_rooms) do
  add :name, :string, null: false
  add :description, :string
  add :public, :boolean, default: true
  add :creator_id, references(:users), null: false
  timestamps()
end

create table(:chat_messages) do
  add :content, :text, null: false
  add :message_type, :string, default: "text"  # text, image, file
  add :room_id, references(:chat_rooms)
  add :sender_id, references(:users), null: false
  add :read_by, {:array, :integer}, default: []
  timestamps()
end

create table(:room_memberships) do
  add :user_id, references(:users), null: false
  add :room_id, references(:chat_rooms), null: false
  add :role, :string, default: "member"  # member, admin
  add :last_read_at, :utc_datetime
  timestamps()
end

create index(:chat_messages, [:room_id, :inserted_at])
create unique_index(:room_memberships, [:user_id, :room_id])
```

---

## 3. Phoenix Channel

```elixir
defmodule ChatWeb.RoomChannel do
  use Phoenix.Channel
  alias ChatWeb.Presence
  alias Chat.{Messages, Rooms}

  def join("room:" <> room_id, params, socket) do
    if authorized?(socket, room_id) do
      send(self(), :after_join)
      {:ok, assign(socket, room_id: room_id, page: 1)}
    else
      {:error, %{reason: "unauthorized"}}
    end
  end

  def handle_info(:after_join, socket) do
    # Track presence
    {:ok, _} = Presence.track(socket, socket.assigns.current_user_id, %{
      online_at: System.system_time(:second),
      name: socket.assigns.username
    })

    # Send recent messages
    messages = Messages.recent_messages(socket.assigns.room_id, 50)
    push(socket, "history", %{messages: serialize_messages(messages)})

    # Broadcast current users
    push(socket, "presence_state", Presence.list(socket))

    {:noreply, socket}
  end

  def handle_in("new_message", %{"content" => content}, socket) do
    room_id = socket.assigns.room_id
    user_id = socket.assigns.current_user_id

    case Messages.create_message(%{
      content: content,
      room_id: room_id,
      sender_id: user_id
    }) do
      {:ok, message} ->
        message = Chat.Repo.preload(message, :sender)
        broadcast!(socket, "new_message", serialize_message(message))
        {:reply, :ok, socket}

      {:error, changeset} ->
        {:reply, {:error, %{errors: format_errors(changeset)}}, socket}
    end
  end

  def handle_in("typing", _payload, socket) do
    broadcast_from!(socket, "user_typing", %{
      user_id: socket.assigns.current_user_id,
      username: socket.assigns.username
    })
    {:noreply, socket}
  end

  def handle_in("load_more", %{"before" => before_id}, socket) do
    messages = Messages.messages_before(socket.assigns.room_id, before_id, 20)
    push(socket, "history", %{messages: serialize_messages(messages), type: "older"})
    {:noreply, socket}
  end

  def handle_in("mark_read", %{"message_id" => msg_id}, socket) do
    Messages.mark_as_read(msg_id, socket.assigns.current_user_id)
    {:noreply, socket}
  end

  def terminate(_reason, socket) do
    Presence.untrack(socket, socket.assigns.current_user_id)
    :ok
  end

  defp authorized?(socket, room_id) do
    Rooms.member?(room_id, socket.assigns.current_user_id)
  end

  defp serialize_message(message) do
    %{
      id: message.id,
      content: message.content,
      sender_id: message.sender_id,
      sender_name: message.sender.name,
      inserted_at: message.inserted_at
    }
  end

  defp serialize_messages(messages), do: Enum.map(messages, &serialize_message/1)

  defp format_errors(changeset) do
    Ecto.Changeset.traverse_errors(changeset, fn {msg, opts} ->
      Regex.replace(~r"%{(\w+)}", msg, fn _, key ->
        opts |> Keyword.get(String.to_existing_atom(key), key) |> to_string()
      end)
    end)
  end
end
```

---

## 4. Presence Tracking

```elixir
defmodule ChatWeb.Presence do
  use Phoenix.Presence,
    otp_app: :chat,
    pubsub_server: Chat.PubSub

  def fetch(_topic, presences) do
    users = presences
    |> Map.keys()
    |> Enum.map(&String.to_integer/1)
    |> Chat.Accounts.get_users_by_ids()
    |> Map.new(&{to_string(&1.id), &1})

    for {key, %{metas: metas}} <- presences, into: %{} do
      user = Map.get(users, key, %{name: "Unknown"})
      {key, %{metas: metas, user: %{name: user.name}}}
    end
  end
end
```

---

## 5. LiveView Chat Interface

```elixir
defmodule ChatWeb.ChatLive do
  use ChatWeb, :live_view

  alias Chat.{Messages, Rooms}
  alias ChatWeb.Presence

  def mount(%{"room_id" => room_id}, session, socket) do
    user = session["current_user"]

    if connected?(socket) do
      # Subscribe to room messages
      Chat.PubSub.subscribe("room:#{room_id}")
      # Track presence
      Presence.track(self(), "presence:room:#{room_id}", user.id, %{
        name: user.name,
        online_at: System.system_time(:second)
      })
    end

    messages = Messages.recent_messages(room_id, 50)
    online_users = Presence.list("presence:room:#{room_id}")

    {:ok, assign(socket,
      room_id: room_id,
      messages: messages,
      online_users: online_users,
      message_input: "",
      typing_users: []
    )}
  end

  def handle_event("send_message", %{"content" => content}, socket) do
    when content != "" do
    Messages.create_message(%{
      content: content,
      room_id: socket.assigns.room_id,
      sender_id: socket.assigns.current_user.id
    })

    {:noreply, assign(socket, message_input: "")}
  end

  def handle_event("typing", _params, socket) do
    Phoenix.PubSub.broadcast(
      Chat.PubSub,
      "room:#{socket.assigns.room_id}",
      {:typing, socket.assigns.current_user.id, socket.assigns.current_user.name}
    )
    {:noreply, socket}
  end

  # รับ broadcast จาก Channel
  def handle_info({:new_message, message}, socket) do
    {:noreply, update(socket, :messages, &(&1 ++ [message]))}
  end

  def handle_info({:typing, user_id, username}, socket) do
    typing = [{user_id, username} | socket.assigns.typing_users]
    |> Enum.uniq_by(fn {id, _} -> id end)

    Process.send_after(self(), {:stop_typing, user_id}, 3000)
    {:noreply, assign(socket, typing_users: typing)}
  end

  def handle_info({:stop_typing, user_id}, socket) do
    typing = Enum.reject(socket.assigns.typing_users, fn {id, _} -> id == user_id end)
    {:noreply, assign(socket, typing_users: typing)}
  end

  def handle_info(%{event: "presence_diff"}, socket) do
    online_users = Presence.list("presence:room:#{socket.assigns.room_id}")
    {:noreply, assign(socket, online_users: online_users)}
  end

  def render(assigns) do
    ~H"""
    <div class="flex h-screen">
      <!-- Sidebar: Online users -->
      <div class="w-64 bg-gray-800 text-white p-4">
        <h3 class="font-bold mb-4">Online (<%= map_size(@online_users) %>)</h3>
        <%= for {_id, %{metas: [meta | _]}} <- @online_users do %>
          <div class="flex items-center gap-2 mb-2">
            <div class="w-2 h-2 bg-green-400 rounded-full"></div>
            <span><%= meta.name %></span>
          </div>
        <% end %>
      </div>

      <!-- Main chat area -->
      <div class="flex-1 flex flex-col">
        <!-- Messages -->
        <div id="messages" class="flex-1 overflow-y-auto p-4 space-y-4"
          phx-hook="ScrollBottom">
          <%= for msg <- @messages do %>
            <div class="flex gap-3">
              <div class="w-8 h-8 bg-blue-500 rounded-full flex items-center justify-center text-white">
                <%= String.first(msg.sender.name) %>
              </div>
              <div>
                <div class="flex items-baseline gap-2">
                  <span class="font-semibold"><%= msg.sender.name %></span>
                  <span class="text-xs text-gray-500">
                    <%= Calendar.strftime(msg.inserted_at, "%H:%M") %>
                  </span>
                </div>
                <p class="text-gray-800"><%= msg.content %></p>
              </div>
            </div>
          <% end %>
        </div>

        <!-- Typing indicator -->
        <%= if @typing_users != [] do %>
          <div class="px-4 py-1 text-sm text-gray-500 italic">
            <%= Enum.map_join(@typing_users, ", ", fn {_, name} -> name end) %>
            กำลังพิมพ์...
          </div>
        <% end %>

        <!-- Input -->
        <form phx-submit="send_message" class="p-4 border-t">
          <div class="flex gap-2">
            <input
              type="text"
              name="content"
              value={@message_input}
              placeholder="พิมพ์ข้อความ..."
              class="flex-1 border rounded-lg px-4 py-2"
              phx-keyup="typing"
              phx-debounce="500"
            />
            <button type="submit"
              class="bg-blue-600 text-white px-6 py-2 rounded-lg">
              ส่ง
            </button>
          </div>
        </form>
      </div>
    </div>
    """
  end
end
```

---

## 6. Direct Messages

```elixir
defmodule Chat.DirectMessages do
  alias Chat.Repo
  alias Chat.Messages.DirectMessage
  import Ecto.Query

  def get_or_create_conversation(user1_id, user2_id) do
    [a, b] = Enum.sort([user1_id, user2_id])
    conversation_key = "dm:#{a}:#{b}"

    case Repo.get_by(DirectConversation, key: conversation_key) do
      nil ->
        %DirectConversation{}
        |> DirectConversation.changeset(%{key: conversation_key, user_ids: [a, b]})
        |> Repo.insert()

      conv ->
        {:ok, conv}
    end
  end

  def send_message(conversation_id, sender_id, content) do
    %DirectMessage{}
    |> DirectMessage.changeset(%{
      conversation_id: conversation_id,
      sender_id: sender_id,
      content: content
    })
    |> Repo.insert()
  end

  def mark_as_read(conversation_id, reader_id) do
    from(m in DirectMessage,
      where: m.conversation_id == ^conversation_id
        and m.sender_id != ^reader_id
        and ^reader_id not in m.read_by
    )
    |> Repo.update_all(push: [read_by: reader_id])
  end

  def unread_count(user_id) do
    from(m in DirectMessage,
      join: c in assoc(m, :conversation),
      where: ^user_id in c.user_ids
        and m.sender_id != ^user_id
        and ^user_id not in m.read_by,
      select: count(m.id)
    )
    |> Repo.one()
  end
end
```

---

## สรุป

```
Chat System:
├── Phoenix Channels: WebSocket connections
├── Presence: track online users
├── PubSub: broadcast messages
└── LiveView: real-time UI updates

Features:
├── Group chat rooms
├── Direct messages
├── Typing indicators
├── Online status
└── Read receipts

Data Flow:
User sends message
  → Channel receives
  → Save to database
  → Broadcast to all room members
  → LiveView updates UI
```

---

*ก่อนหน้า: [Part 51](part_51.md) | ต่อไป: [Part 53 - Task Management App](part_53.md)*
