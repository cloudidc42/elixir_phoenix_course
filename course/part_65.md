# Part 65: Real-time Notifications System

## เป้าหมายการเรียนรู้

- ส่ง browser push notifications ด้วย Web Push API
- รวม Service Worker เข้ากับ Phoenix application
- สร้าง notification bell ด้วย LiveView แบบ real-time
- ส่ง email digests ด้วย Swoosh
- จัดการ notification preferences ของผู้ใช้
- ส่ง notifications หลายช่องทาง (email, push, in-app)
- ประมวลผล notifications แบบ batch ด้วย Oban

---

## 1. ออกแบบ Notification System

ระบบ notifications ที่ดีต้องรองรับหลาย delivery channels และจัดการได้ในที่เดียว

### 1.1 Database Schema

```elixir
# migration
defmodule MyApp.Repo.Migrations.CreateNotificationsSystem do
  use Ecto.Migration

  def change do
    # ตาราง notifications หลัก
    create table(:notifications) do
      add :user_id, references(:users, on_delete: :delete_all), null: false
      add :type, :string, null: false  # "comment", "mention", "system", etc.
      add :title, :string, null: false
      add :body, :string
      add :data, :map, default: %{}   # extra metadata
      add :action_url, :string
      add :read_at, :utc_datetime
      add :delivered_at, :utc_datetime
      timestamps()
    end

    create index(:notifications, [:user_id, :read_at])
    create index(:notifications, [:user_id, :inserted_at])

    # ตาราง push subscriptions
    create table(:push_subscriptions) do
      add :user_id, references(:users, on_delete: :delete_all), null: false
      add :endpoint, :string, null: false
      add :p256dh_key, :string, null: false
      add :auth_key, :string, null: false
      add :user_agent, :string
      add :active, :boolean, default: true
      timestamps()
    end

    create unique_index(:push_subscriptions, [:endpoint])
    create index(:push_subscriptions, [:user_id, :active])

    # ตาราง notification preferences
    create table(:notification_preferences) do
      add :user_id, references(:users, on_delete: :delete_all), null: false
      add :channel, :string, null: false   # "email", "push", "in_app"
      add :notification_type, :string, null: false
      add :enabled, :boolean, default: true
      timestamps()
    end

    create unique_index(:notification_preferences, [:user_id, :channel, :notification_type])
  end
end
```

### 1.2 Schemas

```elixir
defmodule MyApp.Notifications.Notification do
  use Ecto.Schema
  import Ecto.Changeset

  schema "notifications" do
    belongs_to :user, MyApp.Accounts.User
    field :type, :string
    field :title, :string
    field :body, :string
    field :data, :map, default: %{}
    field :action_url, :string
    field :read_at, :utc_datetime
    field :delivered_at, :utc_datetime
    timestamps()
  end

  def changeset(notification, attrs) do
    notification
    |> cast(attrs, [:user_id, :type, :title, :body, :data, :action_url])
    |> validate_required([:user_id, :type, :title])
    |> validate_inclusion(:type, valid_types())
  end

  def valid_types do
    ~w(comment mention follow like system security digest)
  end
end
```

---

## 2. Web Push Notifications

Web Push ช่วยส่ง notifications ไปยัง browser แม้ผู้ใช้ไม่ได้เปิดเว็บไซต์

### 2.1 ติดตั้ง Web Push Library

```elixir
# mix.exs
defp deps do
  [
    {:web_push_encryption, "~> 0.3"},
    # ...
  ]
end
```

```elixir
# config/runtime.exs - สร้าง VAPID keys ก่อน:
# $ mix run -e "IO.inspect(:web_push_encryption.generate_vapid_keys())"
config :web_push_encryption,
  subject: "mailto:admin@example.com",
  public_key: System.get_env("VAPID_PUBLIC_KEY"),
  private_key: System.get_env("VAPID_PRIVATE_KEY")
```

### 2.2 Service Worker

```javascript
// assets/js/service-worker.js
const CACHE_NAME = 'myapp-v1';
const ASSETS_TO_CACHE = ['/offline.html'];

// Install event
self.addEventListener('install', (event) => {
  event.waitUntil(
    caches.open(CACHE_NAME).then((cache) => cache.addAll(ASSETS_TO_CACHE))
  );
  self.skipWaiting();
});

// Activate event
self.addEventListener('activate', (event) => {
  event.waitUntil(
    caches.keys().then((cacheNames) => {
      return Promise.all(
        cacheNames
          .filter((name) => name !== CACHE_NAME)
          .map((name) => caches.delete(name))
      );
    })
  );
  self.clients.claim();
});

// Push event - รับ push notification
self.addEventListener('push', (event) => {
  if (!event.data) return;

  const data = event.data.json();

  const options = {
    body: data.body,
    icon: '/images/notification-icon.png',
    badge: '/images/badge.png',
    data: {
      url: data.action_url || '/',
      notificationId: data.id
    },
    actions: [
      { action: 'open', title: 'เปิดดู' },
      { action: 'dismiss', title: 'ปิด' }
    ],
    requireInteraction: false,
    tag: data.type || 'default'
  };

  event.waitUntil(
    self.registration.showNotification(data.title, options)
  );
});

// Notification click event
self.addEventListener('notificationclick', (event) => {
  event.notification.close();

  if (event.action === 'dismiss') return;

  const url = event.notification.data.url;

  event.waitUntil(
    clients.matchAll({ type: 'window', includeUncontrolled: true })
      .then((clientList) => {
        const existingClient = clientList.find(c => c.url === url);
        if (existingClient) {
          return existingClient.focus();
        }
        return clients.openWindow(url);
      })
  );
});
```

### 2.3 JavaScript Client

```javascript
// assets/js/push-notifications.js
export const PushNotifications = {
  async subscribe(vapidPublicKey) {
    if (!('serviceWorker' in navigator) || !('PushManager' in window)) {
      console.log('Push notifications not supported');
      return null;
    }

    try {
      const registration = await navigator.serviceWorker.register('/sw.js');
      await navigator.serviceWorker.ready;

      // ขอ permission
      const permission = await Notification.requestPermission();
      if (permission !== 'granted') {
        console.log('Push permission denied');
        return null;
      }

      // Subscribe
      const subscription = await registration.pushManager.subscribe({
        userVisibleOnly: true,
        applicationServerKey: this.urlBase64ToUint8Array(vapidPublicKey)
      });

      return subscription.toJSON();
    } catch (error) {
      console.error('Push subscription failed:', error);
      return null;
    }
  },

  urlBase64ToUint8Array(base64String) {
    const padding = '='.repeat((4 - (base64String.length % 4)) % 4);
    const base64 = (base64String + padding)
      .replace(/-/g, '+')
      .replace(/_/g, '/');
    const rawData = window.atob(base64);
    return Uint8Array.from([...rawData].map(c => c.charCodeAt(0)));
  }
};
```

### 2.4 Phoenix Controller สำหรับ Subscribe

```elixir
defmodule MyAppWeb.PushSubscriptionController do
  use MyAppWeb, :controller

  alias MyApp.Notifications

  def create(conn, %{"subscription" => subscription_params}) do
    user = conn.assigns.current_user

    attrs = %{
      user_id: user.id,
      endpoint: subscription_params["endpoint"],
      p256dh_key: get_in(subscription_params, ["keys", "p256dh"]),
      auth_key: get_in(subscription_params, ["keys", "auth"]),
      user_agent: get_req_header(conn, "user-agent") |> List.first()
    }

    case Notifications.create_push_subscription(attrs) do
      {:ok, _subscription} ->
        json(conn, %{success: true})
      {:error, changeset} ->
        conn
        |> put_status(:unprocessable_entity)
        |> json(%{errors: format_errors(changeset)})
    end
  end

  def delete(conn, %{"endpoint" => endpoint}) do
    user = conn.assigns.current_user
    Notifications.deactivate_push_subscription(user.id, endpoint)
    json(conn, %{success: true})
  end
end
```

### 2.5 ส่ง Push Notification

```elixir
defmodule MyApp.Notifications.PushDelivery do
  require Logger

  def send_push(subscription, notification) do
    payload = Jason.encode!(%{
      id: notification.id,
      title: notification.title,
      body: notification.body,
      action_url: notification.action_url,
      type: notification.type
    })

    case :web_push_encryption.send_web_push(
      payload,
      %{
        keys: %{
          p256dh: subscription.p256dh_key,
          auth: subscription.auth_key
        },
        endpoint: subscription.endpoint
      }
    ) do
      {:ok, %{status_code: status}} when status in [200, 201] ->
        Logger.info("Push sent successfully to #{subscription.endpoint}")
        {:ok, :delivered}

      {:ok, %{status_code: 410}} ->
        # Subscription expired - deactivate
        Logger.info("Push subscription expired, deactivating: #{subscription.id}")
        MyApp.Notifications.deactivate_push_subscription_by_id(subscription.id)
        {:error, :expired}

      {:ok, %{status_code: status}} ->
        Logger.warning("Push delivery failed with status #{status}")
        {:error, {:http_error, status}}

      {:error, reason} ->
        Logger.error("Push delivery error: #{inspect(reason)}")
        {:error, reason}
    end
  end
end
```

---

## 3. In-App Notification Bell ด้วย LiveView

```elixir
defmodule MyAppWeb.NotificationBellLive do
  use MyAppWeb, :live_component

  alias MyApp.Notifications

  def mount(socket) do
    {:ok, assign(socket, :open, false)}
  end

  def update(%{user_id: user_id} = assigns, socket) do
    # Subscribe ไปยัง PubSub channel ของ user
    if connected?(socket) do
      Phoenix.PubSub.subscribe(
        MyApp.PubSub,
        "notifications:#{user_id}"
      )
    end

    notifications = Notifications.list_recent_notifications(user_id, limit: 10)
    unread_count = Notifications.count_unread(user_id)

    {:ok,
     socket
     |> assign(assigns)
     |> assign(:notifications, notifications)
     |> assign(:unread_count, unread_count)}
  end

  def handle_event("toggle", _params, socket) do
    {:noreply, assign(socket, :open, !socket.assigns.open)}
  end

  def handle_event("mark_all_read", _params, socket) do
    Notifications.mark_all_read(socket.assigns.user_id)
    {:noreply,
     socket
     |> assign(:unread_count, 0)
     |> update(:notifications, fn notifications ->
       Enum.map(notifications, &%{&1 | read_at: DateTime.utc_now()})
     end)}
  end

  def handle_event("mark_read", %{"id" => id}, socket) do
    Notifications.mark_read(id)
    {:noreply,
     socket
     |> update(:notifications, fn notifications ->
       Enum.map(notifications, fn n ->
         if n.id == String.to_integer(id),
           do: %{n | read_at: DateTime.utc_now()},
           else: n
       end)
     end)
     |> update(:unread_count, &max(0, &1 - 1))}
  end

  # รับ notification ใหม่จาก PubSub
  def handle_info({:new_notification, notification}, socket) do
    {:noreply,
     socket
     |> update(:notifications, &[notification | Enum.take(&1, 9)])
     |> update(:unread_count, &(&1 + 1))}
  end

  def render(assigns) do
    ~H"""
    <div class="relative" id={"notification-bell-#{@id}"}>
      <button
        phx-click="toggle"
        phx-target={@myself}
        class="relative p-2 text-gray-500 hover:text-gray-700"
      >
        <svg class="w-6 h-6" fill="none" stroke="currentColor" viewBox="0 0 24 24">
          <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2"
            d="M15 17h5l-1.405-1.405A2.032 2.032 0 0118 14.158V11a6.002 6.002 0 00-4-5.659V5a2 2 0 10-4 0v.341C7.67 6.165 6 8.388 6 11v3.159c0 .538-.214 1.055-.595 1.436L4 17h5m6 0v1a3 3 0 11-6 0v-1m6 0H9" />
        </svg>
        <%= if @unread_count > 0 do %>
          <span class="absolute top-0 right-0 inline-flex items-center justify-center px-2 py-1 text-xs font-bold leading-none text-red-100 bg-red-600 rounded-full">
            <%= if @unread_count > 99, do: "99+", else: @unread_count %>
          </span>
        <% end %>
      </button>

      <%= if @open do %>
        <div class="absolute right-0 mt-2 w-80 bg-white rounded-lg shadow-xl z-50 border border-gray-200">
          <div class="flex items-center justify-between px-4 py-3 border-b">
            <h3 class="text-sm font-semibold text-gray-900">การแจ้งเตือน</h3>
            <button
              phx-click="mark_all_read"
              phx-target={@myself}
              class="text-xs text-blue-600 hover:text-blue-800"
            >
              อ่านทั้งหมด
            </button>
          </div>
          <div class="max-h-96 overflow-y-auto">
            <%= if Enum.empty?(@notifications) do %>
              <div class="px-4 py-8 text-center text-gray-500 text-sm">
                ไม่มีการแจ้งเตือน
              </div>
            <% else %>
              <%= for notification <- @notifications do %>
                <div
                  class={"px-4 py-3 hover:bg-gray-50 cursor-pointer border-b #{if is_nil(notification.read_at), do: "bg-blue-50"}"}
                  phx-click="mark_read"
                  phx-value-id={notification.id}
                  phx-target={@myself}
                >
                  <div class="flex items-start gap-3">
                    <div class="flex-1">
                      <p class="text-sm font-medium text-gray-900"><%= notification.title %></p>
                      <%= if notification.body do %>
                        <p class="text-xs text-gray-500 mt-1"><%= notification.body %></p>
                      <% end %>
                      <p class="text-xs text-gray-400 mt-1">
                        <%= format_time_ago(notification.inserted_at) %>
                      </p>
                    </div>
                    <%= if is_nil(notification.read_at) do %>
                      <div class="w-2 h-2 rounded-full bg-blue-500 mt-1 flex-shrink-0"></div>
                    <% end %>
                  </div>
                </div>
              <% end %>
            <% end %>
          </div>
          <div class="px-4 py-3 border-t">
            <.link navigate="/notifications" class="text-xs text-blue-600 hover:text-blue-800">
              ดูการแจ้งเตือนทั้งหมด
            </.link>
          </div>
        </div>
      <% end %>
    </div>
    """
  end

  defp format_time_ago(datetime) do
    diff = DateTime.diff(DateTime.utc_now(), datetime, :second)
    cond do
      diff < 60 -> "เมื่อกี้"
      diff < 3600 -> "#{div(diff, 60)} นาทีที่แล้ว"
      diff < 86400 -> "#{div(diff, 3600)} ชั่วโมงที่แล้ว"
      true -> "#{div(diff, 86400)} วันที่แล้ว"
    end
  end
end
```

---

## 4. Notification Preferences

```elixir
defmodule MyApp.Notifications.PreferenceManager do
  alias MyApp.Repo
  alias MyApp.Notifications.NotificationPreference
  import Ecto.Query

  @default_preferences %{
    "comment" => %{"email" => true, "push" => true, "in_app" => true},
    "mention" => %{"email" => true, "push" => true, "in_app" => true},
    "follow" => %{"email" => false, "push" => true, "in_app" => true},
    "like" => %{"email" => false, "push" => false, "in_app" => true},
    "system" => %{"email" => true, "push" => true, "in_app" => true},
    "digest" => %{"email" => true, "push" => false, "in_app" => false}
  }

  def get_user_preferences(user_id) do
    user_prefs =
      from(p in NotificationPreference,
        where: p.user_id == ^user_id,
        select: {p.notification_type, p.channel, p.enabled}
      )
      |> Repo.all()
      |> Enum.group_by(&elem(&1, 0), fn {_, channel, enabled} -> {channel, enabled} end)
      |> Map.new(fn {type, channels} -> {type, Map.new(channels)} end)

    # Merge กับ defaults
    Map.merge(@default_preferences, user_prefs, fn _key, default, user ->
      Map.merge(default, user)
    end)
  end

  def enabled?(user_id, notification_type, channel) do
    prefs = get_user_preferences(user_id)
    get_in(prefs, [notification_type, channel]) != false
  end

  def update_preference(user_id, notification_type, channel, enabled) do
    attrs = %{
      user_id: user_id,
      notification_type: notification_type,
      channel: channel,
      enabled: enabled
    }

    %NotificationPreference{}
    |> NotificationPreference.changeset(attrs)
    |> Repo.insert(
      on_conflict: {:replace, [:enabled, :updated_at]},
      conflict_target: [:user_id, :notification_type, :channel]
    )
  end
end
```

---

## 5. Multi-Channel Delivery

```elixir
defmodule MyApp.Notifications.Dispatcher do
  alias MyApp.Notifications
  alias MyApp.Notifications.{PushDelivery, EmailDelivery}
  alias MyApp.Notifications.PreferenceManager
  require Logger

  # ส่ง notification ผ่านทุก channel ที่เปิดใช้งาน
  def dispatch(user, notification_type, payload) do
    # บันทึก in-app notification เสมอ
    {:ok, notification} = Notifications.create_notification(%{
      user_id: user.id,
      type: notification_type,
      title: payload.title,
      body: payload[:body],
      data: payload[:data] || %{},
      action_url: payload[:action_url]
    })

    # แจ้ง LiveView แบบ real-time
    Phoenix.PubSub.broadcast(
      MyApp.PubSub,
      "notifications:#{user.id}",
      {:new_notification, notification}
    )

    # ส่ง push notification
    if PreferenceManager.enabled?(user.id, notification_type, "push") do
      deliver_push(user, notification)
    end

    # ส่ง email
    if PreferenceManager.enabled?(user.id, notification_type, "email") do
      deliver_email(user, notification, payload)
    end

    {:ok, notification}
  end

  defp deliver_push(user, notification) do
    subscriptions = Notifications.list_active_push_subscriptions(user.id)

    Enum.each(subscriptions, fn subscription ->
      Task.Supervisor.start_child(
        MyApp.TaskSupervisor,
        fn -> PushDelivery.send_push(subscription, notification) end
      )
    end)
  end

  defp deliver_email(user, notification, payload) do
    %{
      user_id: user.id,
      notification_id: notification.id,
      email: user.email,
      template: payload[:email_template] || notification.type
    }
    |> MyApp.Workers.SendNotificationEmail.new()
    |> Oban.insert()
  end
end
```

---

## 6. Email Digests ด้วย Swoosh

```elixir
defmodule MyApp.Notifications.DigestMailer do
  use Swoosh.Mailer, otp_app: :my_app
  import Swoosh.Email

  alias MyApp.Notifications

  def send_daily_digest(user) do
    notifications = Notifications.list_unread_for_digest(user.id, hours: 24)

    if length(notifications) > 0 do
      email = build_digest_email(user, notifications)
      deliver(email)
    else
      {:ok, :skipped}
    end
  end

  defp build_digest_email(user, notifications) do
    grouped = Enum.group_by(notifications, & &1.type)

    new()
    |> to({user.name, user.email})
    |> from({"MyApp", "noreply@myapp.com"})
    |> subject("สรุปการแจ้งเตือนของคุณ - #{format_date(Date.utc_today())}")
    |> html_body(render_digest_html(user, grouped, notifications))
    |> text_body(render_digest_text(user, notifications))
  end

  defp render_digest_html(user, grouped, notifications) do
    """
    <!DOCTYPE html>
    <html>
    <head>
      <meta charset="utf-8">
      <style>
        body { font-family: sans-serif; color: #333; max-width: 600px; margin: 0 auto; }
        .header { background: #4F46E5; color: white; padding: 20px; }
        .notification { padding: 12px; border-bottom: 1px solid #eee; }
        .unread { background: #EEF2FF; }
        .type-badge { display: inline-block; padding: 2px 8px; border-radius: 10px; font-size: 11px; }
        .footer { padding: 20px; text-align: center; color: #999; font-size: 12px; }
      </style>
    </head>
    <body>
      <div class="header">
        <h1>สวัสดี #{user.name}</h1>
        <p>คุณมีการแจ้งเตือน #{length(notifications)} รายการที่ยังไม่ได้อ่าน</p>
      </div>
      <div class="content">
        #{render_grouped_notifications(grouped)}
      </div>
      <div class="footer">
        <a href="#{MyAppWeb.Endpoint.url()}/notifications/preferences">
          จัดการการตั้งค่าการแจ้งเตือน
        </a>
      </div>
    </body>
    </html>
    """
  end

  defp render_grouped_notifications(grouped) do
    Enum.map_join(grouped, "\n", fn {type, notifications} ->
      type_label = type_display_name(type)
      items = Enum.map_join(notifications, "\n", fn n ->
        """
        <div class="notification">
          <strong>#{n.title}</strong>
          #{if n.body, do: "<p>#{n.body}</p>", else: ""}
          <small style="color: #999;">#{format_time(n.inserted_at)}</small>
        </div>
        """
      end)
      "<h2>#{type_label} (#{length(notifications)})</h2>\n#{items}"
    end)
  end

  defp render_digest_text(user, notifications) do
    items = Enum.map_join(notifications, "\n\n", fn n ->
      "- #{n.title}#{if n.body, do: "\n  #{n.body}", else: ""}"
    end)
    """
    สวัสดี #{user.name},

    คุณมีการแจ้งเตือน #{length(notifications)} รายการ:

    #{items}

    ดูทั้งหมดได้ที่: #{MyAppWeb.Endpoint.url()}/notifications
    """
  end

  defp type_display_name("comment"), do: "ความคิดเห็น"
  defp type_display_name("mention"), do: "การกล่าวถึง"
  defp type_display_name("follow"), do: "ผู้ติดตาม"
  defp type_display_name("like"), do: "การถูกใจ"
  defp type_display_name(type), do: type

  defp format_date(date), do: "#{date.day}/#{date.month}/#{date.year}"
  defp format_time(datetime) do
    "#{datetime.hour}:#{String.pad_leading("#{datetime.minute}", 2, "0")}"
  end
end
```

---

## 7. Batch Notification Processing ด้วย Oban

```elixir
# workers/notification_workers.ex
defmodule MyApp.Workers.SendNotificationEmail do
  use Oban.Worker,
    queue: :notifications,
    max_attempts: 3,
    priority: 2

  @impl Oban.Worker
  def perform(%Oban.Job{args: %{"notification_id" => id, "email" => email}}) do
    notification = MyApp.Notifications.get_notification!(id)
    user = MyApp.Accounts.get_user_by_email!(email)

    case MyApp.Notifications.NotificationMailer.send_notification(user, notification) do
      {:ok, _} ->
        MyApp.Notifications.mark_email_delivered(id)
        :ok
      {:error, reason} ->
        {:error, "Email delivery failed: #{inspect(reason)}"}
    end
  end
end

defmodule MyApp.Workers.SendDailyDigest do
  use Oban.Worker,
    queue: :digests,
    max_attempts: 2,
    unique: [period: 86_400, fields: [:args]]

  @impl Oban.Worker
  def perform(%Oban.Job{args: %{"user_id" => user_id}}) do
    user = MyApp.Accounts.get_user!(user_id)

    case MyApp.Notifications.DigestMailer.send_daily_digest(user) do
      {:ok, _} -> :ok
      {:ok, :skipped} -> :ok
      {:error, reason} -> {:error, reason}
    end
  end
end

defmodule MyApp.Workers.ScheduleDailyDigests do
  use Oban.Worker, queue: :scheduled, max_attempts: 1

  @impl Oban.Worker
  def perform(_job) do
    # หา users ที่เปิดใช้ digest
    users = MyApp.Accounts.list_users_with_digest_enabled()

    jobs =
      Enum.map(users, fn user ->
        %{user_id: user.id}
        |> MyApp.Workers.SendDailyDigest.new()
      end)

    Oban.insert_all(jobs)
    :ok
  end
end
```

### 7.1 ตั้งเวลา Digest

```elixir
# config/config.exs
config :my_app, Oban,
  queues: [
    notifications: 10,
    digests: 5,
    scheduled: 2
  ],
  plugins: [
    {Oban.Plugins.Cron,
     crontab: [
       # ส่ง digest ทุกวัน เวลา 8:00 น.
       {"0 8 * * *", MyApp.Workers.ScheduleDailyDigests}
     ]}
  ]
```

---

## 8. Notification Context Module

```elixir
defmodule MyApp.Notifications do
  alias MyApp.Repo
  alias MyApp.Notifications.{Notification, PushSubscription}
  import Ecto.Query

  def create_notification(attrs) do
    %Notification{}
    |> Notification.changeset(attrs)
    |> Repo.insert()
  end

  def list_recent_notifications(user_id, opts \\ []) do
    limit = Keyword.get(opts, :limit, 20)

    from(n in Notification,
      where: n.user_id == ^user_id,
      order_by: [desc: n.inserted_at],
      limit: ^limit
    )
    |> Repo.all()
  end

  def count_unread(user_id) do
    from(n in Notification,
      where: n.user_id == ^user_id and is_nil(n.read_at),
      select: count(n.id)
    )
    |> Repo.one()
  end

  def mark_read(notification_id) do
    from(n in Notification, where: n.id == ^notification_id)
    |> Repo.update_all(set: [read_at: DateTime.utc_now()])
  end

  def mark_all_read(user_id) do
    from(n in Notification,
      where: n.user_id == ^user_id and is_nil(n.read_at)
    )
    |> Repo.update_all(set: [read_at: DateTime.utc_now()])
  end

  def list_unread_for_digest(user_id, opts \\ []) do
    hours = Keyword.get(opts, :hours, 24)
    since = DateTime.add(DateTime.utc_now(), -hours * 3600, :second)

    from(n in Notification,
      where: n.user_id == ^user_id
        and is_nil(n.read_at)
        and n.inserted_at >= ^since,
      order_by: [desc: n.inserted_at]
    )
    |> Repo.all()
  end

  def list_active_push_subscriptions(user_id) do
    from(s in PushSubscription,
      where: s.user_id == ^user_id and s.active == true
    )
    |> Repo.all()
  end

  def create_push_subscription(attrs) do
    %PushSubscription{}
    |> PushSubscription.changeset(attrs)
    |> Repo.insert(
      on_conflict: {:replace_all_except, [:id, :inserted_at]},
      conflict_target: [:endpoint]
    )
  end

  def deactivate_push_subscription(user_id, endpoint) do
    from(s in PushSubscription,
      where: s.user_id == ^user_id and s.endpoint == ^endpoint
    )
    |> Repo.update_all(set: [active: false])
  end
end
```

---

## สรุป

```
Notification System Architecture
═════════════════════════════════════════════════════════
Trigger (Event occurs)
  │
  ▼
Dispatcher
  ├── Create in-app notification (DB)
  ├── Broadcast via PubSub → LiveView Bell (real-time)
  ├── Enqueue push delivery (Oban, if pref enabled)
  └── Enqueue email delivery (Oban, if pref enabled)

Delivery Channels
  ┌─────────────────────────────────────────────────┐
  │  In-App (LiveView)                              │
  │    PubSub → Bell Component → Instant update     │
  │                                                  │
  │  Push Notifications (Web Push API)               │
  │    VAPID keys → Subscription → Service Worker   │
  │                                                  │
  │  Email (Swoosh + Oban)                           │
  │    Individual: notify email                      │
  │    Daily digest: batch (Cron 8:00 AM)           │
  └─────────────────────────────────────────────────┘

Preferences
  user_id + notification_type + channel → enabled/disabled

Database Tables
  notifications       (in-app store)
  push_subscriptions  (Web Push endpoints)
  notification_preferences (per-user settings)
═════════════════════════════════════════════════════════
```

---

*ก่อนหน้า: [Part 64 - Database Optimization Advanced](part_64.md) | ต่อไป: [Part 66 - Search System](part_66.md)*
