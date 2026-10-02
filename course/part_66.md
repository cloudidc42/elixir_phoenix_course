# Part 66: Real-time Notifications (การแจ้งเตือนแบบ Real-time)

## เป้าหมายการเรียนรู้

หลังจากจบ Part นี้ คุณจะสามารถ:
- สร้าง notification system ที่ครอบคลุม
- Push notifications ผ่าน Web Push API
- In-app notifications ด้วย LiveView
- Email digest notifications

---

## 1. Notification Schema

```elixir
defmodule MyApp.Notifications.Notification do
  use Ecto.Schema
  import Ecto.Changeset

  schema "notifications" do
    field :type, :string          # comment, mention, follow, like
    field :title, :string
    field :body, :string
    field :data, :map             # extra payload (post_id, comment_id, etc)
    field :read_at, :utc_datetime
    field :sent_at, :utc_datetime

    belongs_to :user, MyApp.User
    belongs_to :actor, MyApp.User  # who triggered the notification

    timestamps()
  end

  def changeset(notification, attrs) do
    notification
    |> cast(attrs, [:type, :title, :body, :data, :user_id, :actor_id])
    |> validate_required([:type, :title, :user_id])
  end
end
```

---

## 2. Notification Context

```elixir
defmodule MyApp.Notifications do
  alias MyApp.Repo
  alias MyApp.Notifications.Notification
  import Ecto.Query

  def create_notification(attrs) do
    result =
      %Notification{}
      |> Notification.changeset(attrs)
      |> Repo.insert()

    case result do
      {:ok, notification} ->
        # Broadcast real-time to user
        Phoenix.PubSub.broadcast(
          MyApp.PubSub,
          "user:#{attrs.user_id}:notifications",
          {:new_notification, notification}
        )
        {:ok, notification}

      error -> error
    end
  end

  def list_for_user(user_id, opts \\ []) do
    limit = Keyword.get(opts, :limit, 20)
    only_unread = Keyword.get(opts, :unread, false)

    query = from(n in Notification,
      where: n.user_id == ^user_id,
      order_by: [desc: n.inserted_at],
      limit: ^limit,
      preload: [:actor]
    )

    query = if only_unread, do: where(query, [n], is_nil(n.read_at)), else: query
    Repo.all(query)
  end

  def unread_count(user_id) do
    Repo.one(
      from n in Notification,
      where: n.user_id == ^user_id and is_nil(n.read_at),
      select: count(n.id)
    )
  end

  def mark_read(user_id, notification_ids) do
    from(n in Notification,
      where: n.user_id == ^user_id and n.id in ^notification_ids
    )
    |> Repo.update_all(set: [read_at: DateTime.utc_now()])
  end

  def mark_all_read(user_id) do
    from(n in Notification,
      where: n.user_id == ^user_id and is_nil(n.read_at)
    )
    |> Repo.update_all(set: [read_at: DateTime.utc_now()])
  end
end
```

---

## 3. LiveView Notification Bell

```elixir
defmodule MyAppWeb.Components.NotificationBell do
  use MyAppWeb, :live_component

  alias MyApp.Notifications

  def update(%{user: user} = assigns, socket) do
    if connected?(socket) do
      Phoenix.PubSub.subscribe(MyApp.PubSub, "user:#{user.id}:notifications")
    end

    {:ok, assign(socket,
      user: user,
      notifications: Notifications.list_for_user(user.id, limit: 10),
      unread_count: Notifications.unread_count(user.id),
      open: false
    )}
  end

  def handle_event("toggle", _, socket) do
    {:noreply, update(socket, :open, &(!&1))}
  end

  def handle_event("mark_read", %{"id" => id}, socket) do
    Notifications.mark_read(socket.assigns.user.id, [String.to_integer(id)])
    notifications = Notifications.list_for_user(socket.assigns.user.id, limit: 10)
    {:noreply, assign(socket,
      notifications: notifications,
      unread_count: Notifications.unread_count(socket.assigns.user.id)
    )}
  end

  def handle_event("mark_all_read", _, socket) do
    Notifications.mark_all_read(socket.assigns.user.id)
    {:noreply, assign(socket,
      notifications: Enum.map(socket.assigns.notifications, &Map.put(&1, :read_at, DateTime.utc_now())),
      unread_count: 0
    )}
  end

  def handle_info({:new_notification, notification}, socket) do
    notifications = [notification | Enum.take(socket.assigns.notifications, 9)]
    {:noreply, assign(socket,
      notifications: notifications,
      unread_count: socket.assigns.unread_count + 1
    )}
  end

  def render(assigns) do
    ~H"""
    <div class="relative">
      <!-- Bell icon with badge -->
      <button phx-click="toggle" phx-target={@myself} class="relative p-2">
        <.icon name="hero-bell" class="w-6 h-6" />
        <%= if @unread_count > 0 do %>
          <span class="absolute -top-1 -right-1 bg-red-500 text-white
            text-xs rounded-full w-5 h-5 flex items-center justify-center">
            <%= min(@unread_count, 99) %>
          </span>
        <% end %>
      </button>

      <!-- Dropdown -->
      <%= if @open do %>
        <div class="absolute right-0 top-full mt-2 w-80 bg-white rounded-lg shadow-lg
          border z-50 max-h-96 overflow-y-auto">
          <div class="flex justify-between items-center p-3 border-b">
            <h3 class="font-semibold">การแจ้งเตือน</h3>
            <button phx-click="mark_all_read" phx-target={@myself}
              class="text-sm text-blue-600">
              อ่านทั้งหมด
            </button>
          </div>

          <%= for n <- @notifications do %>
            <div class={"p-3 hover:bg-gray-50 border-b last:border-0 #{if is_nil(n.read_at), do: "bg-blue-50"}"}>
              <div class="flex items-start gap-3">
                <%= if n.actor do %>
                  <img src={n.actor.avatar_url || "/images/default_avatar.png"}
                    class="w-8 h-8 rounded-full" />
                <% end %>
                <div class="flex-1 min-w-0">
                  <p class="text-sm font-medium"><%= n.title %></p>
                  <p class="text-xs text-gray-500 truncate"><%= n.body %></p>
                  <p class="text-xs text-gray-400 mt-1">
                    <%= Timex.from_now(n.inserted_at) %>
                  </p>
                </div>
                <%= if is_nil(n.read_at) do %>
                  <button phx-click="mark_read" phx-value-id={n.id}
                    phx-target={@myself} class="w-2 h-2 bg-blue-500 rounded-full mt-1 flex-shrink-0">
                  </button>
                <% end %>
              </div>
            </div>
          <% end %>

          <%= if @notifications == [] do %>
            <p class="p-6 text-center text-gray-500">ไม่มีการแจ้งเตือน</p>
          <% end %>
        </div>
      <% end %>
    </div>
    """
  end
end
```

---

## 4. Web Push Notifications

```elixir
# mix.exs: {:web_push_encryption, "~> 0.3"}

defmodule MyApp.WebPush do
  @vapid_public_key Application.compile_env(:my_app, :vapid_public_key)
  @vapid_private_key Application.compile_env(:my_app, :vapid_private_key)

  def send(subscription, payload) do
    WebPushEncryption.send_web_push(
      Jason.encode!(payload),
      subscription,
      @vapid_private_key,
      @vapid_public_key
    )
  end

  def notify_user(user_id, title, body, opts \\ []) do
    subscriptions = MyApp.Repo.all(
      from(s in MyApp.PushSubscription, where: s.user_id == ^user_id)
    )

    payload = %{
      title: title,
      body: body,
      icon: opts[:icon] || "/icons/icon-192.png",
      url: opts[:url] || "/"
    }

    Enum.each(subscriptions, fn subscription ->
      case send(subscription.data, payload) do
        {:ok, _} -> :ok
        {:error, :gone} ->
          # Remove expired subscription
          MyApp.Repo.delete(subscription)
        {:error, reason} ->
          require Logger
          Logger.warning("Push notification failed: #{inspect(reason)}")
      end
    end)
  end
end
```

---

## สรุป

```
Notification System:
├── Notification schema: type, title, body, read_at
├── PubSub: real-time delivery via broadcast
├── LiveView bell: unread badge, dropdown
└── Web Push: background push to browser/mobile

Patterns:
├── Async delivery (don't block on notification)
├── Cleanup expired push subscriptions
├── Batch mark-as-read
└── Notification type enum for routing
```

---

*ก่อนหน้า: [Part 65](part_65.md) | ต่อไป: [Part 67 - Search System](part_67.md)*
