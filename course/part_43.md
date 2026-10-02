# Part 43: Phoenix PubSub and Real-time Patterns

## เป้าหมายการเรียนรู้

- เข้าใจ Phoenix.PubSub และการใช้งาน subscribe/broadcast
- กำหนด custom PubSub topics อย่างมีระบบ
- ติดตาม user presence ด้วย Phoenix.Presence
- สร้างระบบ user online/offline status
- สร้าง collaborative editing pattern
- สร้าง notifications system
- สร้าง real-time dashboard

---

## 1. Phoenix.PubSub พื้นฐาน

PubSub (Publish-Subscribe) เป็น pattern สำหรับการสื่อสารแบบ real-time ระหว่าง processes

```elixir
# config/config.exs - PubSub configuration
config :my_app, MyApp.PubSub,
  name: MyApp.PubSub

# application.ex
defmodule MyApp.Application do
  use Application

  def start(_type, _args) do
    children = [
      MyApp.Repo,
      MyAppWeb.Endpoint,
      # PubSub ต้องเริ่มก่อน Endpoint
      {Phoenix.PubSub, name: MyApp.PubSub},
      MyAppWeb.Presence
    ]

    opts = [strategy: :one_for_one, name: MyApp.Supervisor]
    Supervisor.start_link(children, opts)
  end
end
```

```elixir
# การ subscribe และ broadcast พื้นฐาน
defmodule MyApp.Notifications do
  alias Phoenix.PubSub

  @pubsub MyApp.PubSub

  # Subscribe to a topic
  def subscribe(topic) do
    PubSub.subscribe(@pubsub, topic)
  end

  # Broadcast ไปยังทุก subscriber
  def broadcast(topic, message) do
    PubSub.broadcast(@pubsub, topic, message)
  end

  # Broadcast ยกเว้น process ที่ส่ง (ใช้ใน LiveView)
  def broadcast_from(topic, message) do
    PubSub.broadcast_from(@pubsub, self(), topic, message)
  end

  # Local broadcast - เฉพาะ node เดียว (ไม่ข้าม cluster)
  def local_broadcast(topic, message) do
    PubSub.local_broadcast(@pubsub, topic, message)
  end
end
```

```elixir
# ตัวอย่างการใช้ใน LiveView
defmodule MyAppWeb.ChatLive do
  use MyAppWeb, :live_view
  alias MyApp.Notifications

  @topic "chat:general"

  def mount(_params, _session, socket) do
    if connected?(socket) do
      Notifications.subscribe(@topic)
    end

    {:ok, assign(socket, messages: [])}
  end

  def handle_event("send_message", %{"body" => body}, socket) do
    message = %{
      id: System.unique_integer([:positive]),
      body: body,
      user: socket.assigns.current_user,
      sent_at: DateTime.utc_now()
    }

    # Broadcast ไปยังทุก subscriber
    Notifications.broadcast(@topic, {:new_message, message})

    {:noreply, socket}
  end

  # รับ message จาก PubSub
  def handle_info({:new_message, message}, socket) do
    {:noreply, update(socket, :messages, fn msgs -> [message | msgs] end)}
  end
end
```

---

## 2. Custom PubSub Topics

การออกแบบ topic naming convention อย่างมีระบบ

```elixir
# lib/my_app/topics.ex - Centralized topic management
defmodule MyApp.Topics do
  @doc "Topic สำหรับ room chat"
  def room(room_id), do: "room:#{room_id}"

  @doc "Topic สำหรับ user notifications"
  def user_notifications(user_id), do: "user:#{user_id}:notifications"

  @doc "Topic สำหรับ user presence"
  def user_presence(user_id), do: "user:#{user_id}:presence"

  @doc "Topic สำหรับ document editing"
  def document(doc_id), do: "document:#{doc_id}"

  @doc "Topic สำหรับ dashboard metrics"
  def dashboard_metrics(org_id), do: "org:#{org_id}:metrics"

  @doc "Topic สำหรับ system-wide announcements"
  def system_announcements(), do: "system:announcements"

  @doc "Topic สำหรับ order status"
  def order(order_id), do: "order:#{order_id}"
end
```

```elixir
# lib/my_app/chat.ex - ใช้ Topics module
defmodule MyApp.Chat do
  alias MyApp.{Topics, Repo}
  alias MyApp.Chat.Message
  alias Phoenix.PubSub

  @pubsub MyApp.PubSub

  def subscribe_room(room_id) do
    PubSub.subscribe(@pubsub, Topics.room(room_id))
  end

  def send_message(room_id, user, body) do
    with {:ok, message} <- create_message(room_id, user, body) do
      PubSub.broadcast(@pubsub, Topics.room(room_id), {
        :new_message,
        %{
          id: message.id,
          body: message.body,
          user_id: user.id,
          user_name: user.name,
          sent_at: message.inserted_at
        }
      })

      {:ok, message}
    end
  end

  def delete_message(room_id, message_id) do
    with {:ok, _} <- do_delete_message(message_id) do
      PubSub.broadcast(@pubsub, Topics.room(room_id), {
        :message_deleted,
        %{message_id: message_id}
      })
    end
  end

  defp create_message(room_id, user, body) do
    %Message{}
    |> Message.changeset(%{room_id: room_id, user_id: user.id, body: body})
    |> Repo.insert()
  end

  defp do_delete_message(message_id) do
    case Repo.get(Message, message_id) do
      nil -> {:error, :not_found}
      message -> Repo.delete(message)
    end
  end
end
```

---

## 3. Phoenix.Presence

Presence ช่วย track ว่า user คนไหนกำลัง online บน topic ใด

```elixir
# lib/my_app_web/presence.ex
defmodule MyAppWeb.Presence do
  use Phoenix.Presence,
    otp_app: :my_app,
    pubsub_server: MyApp.PubSub
end
```

```elixir
# ใช้ Presence ใน LiveView
defmodule MyAppWeb.RoomLive do
  use MyAppWeb, :live_view
  alias MyAppWeb.Presence
  alias MyApp.Topics

  def mount(%{"room_id" => room_id}, _session, socket) do
    user = socket.assigns.current_user
    topic = Topics.room(room_id)

    if connected?(socket) do
      # Subscribe to PubSub
      Phoenix.PubSub.subscribe(MyApp.PubSub, topic)

      # Track user presence ใน room
      {:ok, _} = Presence.track(self(), topic, user.id, %{
        name: user.name,
        avatar: user.avatar_url,
        joined_at: System.system_time(:second)
      })
    end

    # ดู users ที่ online ทั้งหมดในตอนแรก
    online_users = Presence.list(topic)

    socket =
      socket
      |> assign(:room_id, room_id)
      |> assign(:online_users, online_users)

    {:ok, socket}
  end

  # รับ presence update
  def handle_info(%Phoenix.Socket.Broadcast{
        event: "presence_diff",
        payload: diff
      }, socket) do
    online_users =
      socket.assigns.online_users
      |> Presence.List.update(diff)

    {:noreply, assign(socket, :online_users, online_users)}
  end

  def render(assigns) do
    ~H"""
    <div class="room-container">
      <div class="online-users">
        <h3>Online (<%= map_size(@online_users) %>)</h3>
        <%= for {_id, %{metas: [meta | _]}} <- @online_users do %>
          <div class="user">
            <img src={meta.avatar} alt={meta.name} />
            <span><%= meta.name %></span>
          </div>
        <% end %>
      </div>
    </div>
    """
  end
end
```

---

## 4. User Online/Offline Status

```elixir
# lib/my_app_web/presence_tracker.ex
defmodule MyApp.PresenceTracker do
  @moduledoc """
  จัดการ user online/offline status ทั่วทั้ง application
  """

  alias MyAppWeb.Presence
  alias Phoenix.PubSub
  alias MyApp.Topics

  @global_topic "presence:global"

  def track_user(user) do
    # Track ใน global presence topic
    {:ok, _} = Presence.track(self(), @global_topic, user.id, %{
      name: user.name,
      avatar: user.avatar_url,
      status: "online",
      last_seen: DateTime.utc_now()
    })

    # Notify friends ว่า user online
    notify_user_online(user)
  end

  def update_status(user, status) when status in ["online", "away", "busy"] do
    Presence.update(self(), @global_topic, user.id, fn meta ->
      Map.put(meta, :status, status)
    end)
  end

  def get_online_users do
    Presence.list(@global_topic)
    |> Enum.map(fn {user_id, %{metas: [meta | _]}} ->
      {user_id, meta}
    end)
    |> Map.new()
  end

  def user_online?(user_id) do
    case Presence.get_by_key(@global_topic, user_id) do
      [] -> false
      _ -> true
    end
  end

  defp notify_user_online(user) do
    PubSub.broadcast(MyApp.PubSub, Topics.user_notifications(user.id), {
      :user_came_online,
      %{user_id: user.id, name: user.name}
    })
  end
end
```

```elixir
# LiveView ที่ track presence ของ user
defmodule MyAppWeb.UserDashboardLive do
  use MyAppWeb, :live_view
  alias MyApp.PresenceTracker

  def mount(_params, _session, socket) do
    user = socket.assigns.current_user

    if connected?(socket) do
      # Track user เมื่อ connect
      PresenceTracker.track_user(user)

      # Subscribe เพื่อรับ notifications
      Phoenix.PubSub.subscribe(MyApp.PubSub, "user:#{user.id}:notifications")
    end

    {:ok, socket}
  end

  def terminate(_reason, socket) do
    # Presence จะ untrack อัตโนมัติเมื่อ process terminate
    # แต่เราสามารถ log ได้
    user = socket.assigns.current_user
    MyApp.Accounts.update_last_seen(user)

    :ok
  end

  def handle_event("set_status", %{"status" => status}, socket) do
    user = socket.assigns.current_user
    PresenceTracker.update_status(user, status)

    {:noreply, assign(socket, :status, status)}
  end
end
```

---

## 5. Collaborative Editing Pattern

```elixir
# lib/my_app/documents/collab.ex
defmodule MyApp.Documents.Collab do
  @moduledoc """
  Collaborative editing สำหรับ documents
  ใช้ Operational Transform แบบง่ายๆ
  """

  alias Phoenix.PubSub
  alias MyApp.{Topics, Repo}
  alias MyApp.Documents.Document
  alias MyAppWeb.Presence

  def subscribe(doc_id) do
    PubSub.subscribe(MyApp.PubSub, Topics.document(doc_id))
  end

  def join_document(doc_id, user) do
    topic = Topics.document(doc_id)

    # Track user ที่กำลัง edit document
    {:ok, _} = Presence.track(self(), topic, user.id, %{
      name: user.name,
      avatar: user.avatar_url,
      cursor_position: 0,
      joined_at: System.system_time(:second)
    })

    # Broadcast ว่า user เข้ามา edit
    PubSub.broadcast(MyApp.PubSub, topic, {
      :user_joined,
      %{user_id: user.id, name: user.name}
    })
  end

  def apply_operation(doc_id, user_id, operation) do
    topic = Topics.document(doc_id)

    # Apply operation ลง database
    with {:ok, doc} <- do_apply_operation(doc_id, operation) do
      # Broadcast operation ไปยังทุก editor
      PubSub.broadcast_from(MyApp.PubSub, self(), topic, {
        :operation_applied,
        %{
          doc_id: doc_id,
          user_id: user_id,
          operation: operation,
          version: doc.version
        }
      })

      {:ok, doc}
    end
  end

  def update_cursor(doc_id, user_id, position) do
    topic = Topics.document(doc_id)

    # Update cursor position ใน presence
    Presence.update(self(), topic, user_id, fn meta ->
      Map.put(meta, :cursor_position, position)
    end)
  end

  defp do_apply_operation(doc_id, operation) do
    Repo.transaction(fn ->
      doc = Repo.get!(Document, doc_id)
      new_content = apply_op(doc.content, operation)

      doc
      |> Document.changeset(%{content: new_content, version: doc.version + 1})
      |> Repo.update!()
    end)
  end

  # ตัวอย่าง Operational Transform แบบง่าย
  defp apply_op(content, %{type: :insert, pos: pos, text: text}) do
    {before, after_pos} = String.split_at(content, pos)
    before <> text <> after_pos
  end

  defp apply_op(content, %{type: :delete, pos: pos, length: len}) do
    {before, rest} = String.split_at(content, pos)
    {_deleted, after_pos} = String.split_at(rest, len)
    before <> after_pos
  end
end
```

```elixir
# LiveView สำหรับ collaborative editor
defmodule MyAppWeb.DocumentEditorLive do
  use MyAppWeb, :live_view
  alias MyApp.Documents.{Collab, Document}
  alias MyAppWeb.Presence
  alias MyApp.Topics

  def mount(%{"id" => doc_id}, _session, socket) do
    user = socket.assigns.current_user
    doc = Document.get!(doc_id)
    topic = Topics.document(doc_id)

    if connected?(socket) do
      Collab.subscribe(doc_id)
      Collab.join_document(doc_id, user)
    end

    editors = Presence.list(topic)

    {:ok,
     socket
     |> assign(:doc, doc)
     |> assign(:doc_id, doc_id)
     |> assign(:editors, editors)}
  end

  def handle_event("apply_operation", %{"operation" => op}, socket) do
    doc_id = socket.assigns.doc_id
    user = socket.assigns.current_user
    operation = parse_operation(op)

    case Collab.apply_operation(doc_id, user.id, operation) do
      {:ok, doc} ->
        {:noreply, assign(socket, :doc, doc)}

      {:error, reason} ->
        {:noreply, put_flash(socket, :error, "Failed to apply: #{reason}")}
    end
  end

  def handle_event("cursor_move", %{"position" => pos}, socket) do
    Collab.update_cursor(socket.assigns.doc_id, socket.assigns.current_user.id, pos)
    {:noreply, socket}
  end

  # รับ operations จาก other editors
  def handle_info({:operation_applied, %{operation: op, version: ver}}, socket) do
    doc = %{socket.assigns.doc | content: apply_local_op(socket.assigns.doc.content, op),
                                  version: ver}
    {:noreply, assign(socket, :doc, doc)}
  end

  def handle_info({:user_joined, %{name: name}}, socket) do
    {:noreply, put_flash(socket, :info, "#{name} joined the document")}
  end

  def handle_info(%Phoenix.Socket.Broadcast{event: "presence_diff", payload: diff}, socket) do
    editors = Presence.List.update(socket.assigns.editors, diff)
    {:noreply, assign(socket, :editors, editors)}
  end

  defp parse_operation(%{"type" => "insert", "pos" => pos, "text" => text}) do
    %{type: :insert, pos: pos, text: text}
  end

  defp parse_operation(%{"type" => "delete", "pos" => pos, "length" => len}) do
    %{type: :delete, pos: pos, length: len}
  end

  defp apply_local_op(content, op) do
    Collab.apply_op(content, op)
  end
end
```

---

## 6. Notifications System

```elixir
# lib/my_app/notifications.ex
defmodule MyApp.Notifications do
  @moduledoc """
  ระบบ notifications แบบ real-time
  """

  alias Phoenix.PubSub
  alias MyApp.{Repo, Topics}
  alias MyApp.Notifications.Notification

  @pubsub MyApp.PubSub

  @doc "Subscribe เพื่อรับ notifications ของ user"
  def subscribe(user_id) do
    PubSub.subscribe(@pubsub, Topics.user_notifications(user_id))
  end

  @doc "ส่ง notification ไปยัง user"
  def notify(user_id, type, data) do
    # บันทึก notification ลง DB
    {:ok, notification} = create_notification(user_id, type, data)

    # Broadcast ผ่าน PubSub
    PubSub.broadcast(@pubsub, Topics.user_notifications(user_id), {
      :new_notification,
      format_notification(notification)
    })

    {:ok, notification}
  end

  @doc "Notify หลาย users พร้อมกัน"
  def notify_many(user_ids, type, data) do
    Enum.each(user_ids, fn user_id ->
      notify(user_id, type, data)
    end)
  end

  @doc "Mark notification as read"
  def mark_as_read(notification_id, user_id) do
    with {:ok, notification} <- get_notification(notification_id, user_id) do
      notification
      |> Notification.changeset(%{read: true, read_at: DateTime.utc_now()})
      |> Repo.update()
    end
  end

  @doc "Mark ทั้งหมดเป็น read"
  def mark_all_as_read(user_id) do
    import Ecto.Query

    from(n in Notification,
      where: n.user_id == ^user_id and n.read == false
    )
    |> Repo.update_all(set: [read: true, read_at: DateTime.utc_now()])
  end

  @doc "นับ unread notifications"
  def unread_count(user_id) do
    import Ecto.Query

    from(n in Notification,
      where: n.user_id == ^user_id and n.read == false,
      select: count(n.id)
    )
    |> Repo.one()
  end

  defp create_notification(user_id, type, data) do
    %Notification{}
    |> Notification.changeset(%{
      user_id: user_id,
      type: type,
      data: data,
      read: false
    })
    |> Repo.insert()
  end

  defp get_notification(id, user_id) do
    case Repo.get_by(Notification, id: id, user_id: user_id) do
      nil -> {:error, :not_found}
      n -> {:ok, n}
    end
  end

  defp format_notification(notification) do
    %{
      id: notification.id,
      type: notification.type,
      data: notification.data,
      read: notification.read,
      inserted_at: notification.inserted_at
    }
  end
end
```

```elixir
# LiveView notification bell component
defmodule MyAppWeb.NotificationBellLive do
  use MyAppWeb, :live_view
  alias MyApp.Notifications

  def mount(_params, _session, socket) do
    user = socket.assigns.current_user

    if connected?(socket) do
      Notifications.subscribe(user.id)
    end

    unread = Notifications.unread_count(user.id)

    {:ok, assign(socket, unread_count: unread, notifications: [])}
  end

  def handle_event("mark_all_read", _, socket) do
    user = socket.assigns.current_user
    Notifications.mark_all_as_read(user.id)

    {:noreply, assign(socket, unread_count: 0)}
  end

  def handle_info({:new_notification, notification}, socket) do
    socket =
      socket
      |> update(:unread_count, &(&1 + 1))
      |> update(:notifications, fn notifs ->
        [notification | Enum.take(notifs, 19)]  # keep latest 20
      end)

    {:noreply, socket}
  end

  def render(assigns) do
    ~H"""
    <div class="notification-bell" phx-click="toggle_dropdown">
      <.icon name="hero-bell" />
      <%= if @unread_count > 0 do %>
        <span class="badge"><%= @unread_count %></span>
      <% end %>
    </div>
    """
  end
end
```

---

## 7. Real-time Dashboard

```elixir
# lib/my_app/metrics.ex
defmodule MyApp.Metrics do
  alias Phoenix.PubSub
  alias MyApp.Topics

  @pubsub MyApp.PubSub

  def subscribe_dashboard(org_id) do
    PubSub.subscribe(@pubsub, Topics.dashboard_metrics(org_id))
  end

  def broadcast_metric_update(org_id, metric_type, value) do
    PubSub.broadcast(@pubsub, Topics.dashboard_metrics(org_id), {
      :metric_updated,
      %{type: metric_type, value: value, timestamp: DateTime.utc_now()}
    })
  end

  def get_current_metrics(org_id) do
    %{
      active_users: count_active_users(org_id),
      orders_today: count_orders_today(org_id),
      revenue_today: calculate_revenue_today(org_id),
      conversion_rate: calculate_conversion_rate(org_id)
    }
  end

  defp count_active_users(org_id) do
    # Query จาก DB หรือ cache
    MyApp.Analytics.count_active_users(org_id)
  end

  defp count_orders_today(org_id) do
    MyApp.Orders.count_today(org_id)
  end

  defp calculate_revenue_today(org_id) do
    MyApp.Orders.revenue_today(org_id)
  end

  defp calculate_conversion_rate(org_id) do
    MyApp.Analytics.conversion_rate(org_id)
  end
end
```

```elixir
# LiveView Dashboard
defmodule MyAppWeb.DashboardLive do
  use MyAppWeb, :live_view
  alias MyApp.Metrics

  @refresh_interval 5_000  # refresh ทุก 5 วินาที

  def mount(_params, _session, socket) do
    user = socket.assigns.current_user
    org_id = user.org_id

    if connected?(socket) do
      Metrics.subscribe_dashboard(org_id)
      # ตั้ง timer เพื่อ refresh ทุก 5 วินาที
      :timer.send_interval(@refresh_interval, self(), :refresh_metrics)
    end

    metrics = Metrics.get_current_metrics(org_id)

    {:ok,
     socket
     |> assign(:org_id, org_id)
     |> assign(:metrics, metrics)
     |> assign(:chart_data, build_chart_data(metrics))}
  end

  def handle_info(:refresh_metrics, socket) do
    metrics = Metrics.get_current_metrics(socket.assigns.org_id)

    {:noreply,
     socket
     |> assign(:metrics, metrics)
     |> assign(:chart_data, build_chart_data(metrics))}
  end

  def handle_info({:metric_updated, %{type: type, value: value}}, socket) do
    # อัปเดต specific metric ที่เปลี่ยนแปลง
    metrics = Map.put(socket.assigns.metrics, type, value)

    {:noreply, assign(socket, :metrics, metrics)}
  end

  defp build_chart_data(metrics) do
    # แปลง metrics เป็น format ที่ใช้กับ chart library
    %{
      labels: ["Active Users", "Orders", "Revenue"],
      data: [metrics.active_users, metrics.orders_today, metrics.revenue_today]
    }
  end

  def render(assigns) do
    ~H"""
    <div class="dashboard">
      <div class="metric-cards">
        <.metric_card title="Active Users" value={@metrics.active_users} icon="users" />
        <.metric_card title="Orders Today" value={@metrics.orders_today} icon="shopping-cart" />
        <.metric_card title="Revenue" value={"$#{@metrics.revenue_today}"} icon="currency-dollar" />
        <.metric_card
          title="Conversion Rate"
          value={"#{@metrics.conversion_rate}%"}
          icon="chart-bar"
        />
      </div>

      <div id="revenue-chart" phx-update="ignore">
        <%# Chart จะถูก render ด้วย JavaScript %>
      </div>
    </div>
    """
  end

  defp metric_card(assigns) do
    ~H"""
    <div class="card">
      <.icon name={"hero-#{@icon}"} />
      <div class="metric-value"><%= @value %></div>
      <div class="metric-title"><%= @title %></div>
    </div>
    """
  end
end
```

```elixir
# Background job ที่ broadcast metrics update
defmodule MyApp.Workers.MetricsBroadcaster do
  use GenServer
  alias MyApp.Metrics

  @interval 10_000  # 10 seconds

  def start_link(_opts) do
    GenServer.start_link(__MODULE__, %{}, name: __MODULE__)
  end

  def init(state) do
    schedule_broadcast()
    {:ok, state}
  end

  def handle_info(:broadcast, state) do
    broadcast_all_org_metrics()
    schedule_broadcast()
    {:noreply, state}
  end

  defp broadcast_all_org_metrics do
    # ดึง orgs ทั้งหมดที่ active
    MyApp.Orgs.list_active_orgs()
    |> Enum.each(fn org ->
      metrics = Metrics.get_current_metrics(org.id)

      Enum.each(metrics, fn {type, value} ->
        Metrics.broadcast_metric_update(org.id, type, value)
      end)
    end)
  end

  defp schedule_broadcast do
    Process.send_after(self(), :broadcast, @interval)
  end
end
```

---

## สรุป

```
Phoenix PubSub Architecture:
┌─────────────────────────────────────────────────┐
│                 Phoenix.PubSub                  │
│           (Distributed Message Bus)             │
│                                                 │
│  Topics:                                        │
│  • "room:{id}"          - Chat rooms            │
│  • "user:{id}:notifs"   - User notifications    │
│  • "document:{id}"      - Collaborative editing │
│  • "org:{id}:metrics"   - Dashboard updates     │
│  • "presence:global"    - Online status         │
└───────────────────┬─────────────────────────────┘
                    │
        ┌───────────┴───────────┐
        │                       │
┌───────▼───────┐       ┌───────▼───────┐
│  LiveView     │       │  Channel      │
│  Processes    │       │  Processes    │
│  (subscribe)  │       │  (subscribe)  │
└───────────────┘       └───────────────┘

Phoenix.Presence:
• Track users per topic
• Auto-cleanup เมื่อ process terminate
• Distributed state ข้าม nodes
• presence_diff events สำหรับ real-time updates

Patterns สรุป:
┌─────────────────────┬──────────────────────────────┐
│ Pattern             │ Use Case                     │
├─────────────────────┼──────────────────────────────┤
│ PubSub.broadcast    │ Send to all subscribers      │
│ PubSub.broadcast_from│ Skip sender (self)          │
│ Presence.track      │ Mark user as online          │
│ Presence.list       │ Get all online users         │
│ :timer.send_interval│ Periodic refresh             │
└─────────────────────┴──────────────────────────────┘
```

---

*ก่อนหน้า: [Part 42 - Testing Advanced](part_42.md) | ต่อไป: [Part 44 - Ecto Advanced](part_44.md)*
