# Part 72: Admin Panel (แผงควบคุมสำหรับผู้ดูแลระบบ)

## เป้าหมายการเรียนรู้

หลังจากจบ Part นี้ คุณจะสามารถ:
- สร้าง admin panel ด้วย LiveView
- Dashboard metrics
- User management
- Content moderation

---

## 1. Admin Authentication

```elixir
defmodule MyAppWeb.Plugs.AdminAuth do
  import Plug.Conn
  import Phoenix.Controller

  def require_admin(conn, _opts) do
    case conn.assigns[:current_user] do
      %{role: "admin"} -> conn
      %{role: "super_admin"} -> conn
      nil ->
        conn
        |> redirect(to: "/admin/login")
        |> halt()
      _ ->
        conn
        |> put_status(:forbidden)
        |> put_view(MyAppWeb.ErrorHTML)
        |> render("403.html")
        |> halt()
    end
  end
end

# router.ex
scope "/admin", MyAppWeb.Admin do
  pipe_through [:browser, :require_admin]

  live "/", DashboardLive, :index
  live "/users", UsersLive, :index
  live "/users/:id", UsersLive, :show
  live "/posts", PostsLive, :index
  live "/settings", SettingsLive, :index
end
```

---

## 2. Dashboard Metrics

```elixir
defmodule MyAppWeb.Admin.DashboardLive do
  use MyAppWeb, :live_view

  alias MyApp.Admin.Stats

  def mount(_params, _session, socket) do
    if connected?(socket) do
      :timer.send_interval(30_000, self(), :refresh_stats)
    end

    {:ok, assign(socket, stats: Stats.get_dashboard_stats())}
  end

  def handle_info(:refresh_stats, socket) do
    {:noreply, assign(socket, stats: Stats.get_dashboard_stats())}
  end

  def render(assigns) do
    ~H"""
    <div class="p-6">
      <h1 class="text-2xl font-bold mb-6">Dashboard</h1>

      <!-- Stat Cards -->
      <div class="grid grid-cols-4 gap-4 mb-8">
        <div class="bg-white p-4 rounded-lg shadow">
          <p class="text-gray-500 text-sm">ผู้ใช้ทั้งหมด</p>
          <p class="text-3xl font-bold"><%= @stats.total_users %></p>
          <p class={"text-sm #{if @stats.users_growth >= 0, do: "text-green-600", else: "text-red-600"}"}>
            <%= if @stats.users_growth >= 0, do: "+", else: "" %><%= @stats.users_growth %>% จากเดือนก่อน
          </p>
        </div>
        <div class="bg-white p-4 rounded-lg shadow">
          <p class="text-gray-500 text-sm">Active Users (7d)</p>
          <p class="text-3xl font-bold"><%= @stats.active_users_7d %></p>
        </div>
        <div class="bg-white p-4 rounded-lg shadow">
          <p class="text-gray-500 text-sm">รายได้เดือนนี้</p>
          <p class="text-3xl font-bold">฿<%= format_number(@stats.mrr) %></p>
        </div>
        <div class="bg-white p-4 rounded-lg shadow">
          <p class="text-gray-500 text-sm">โพสต์ทั้งหมด</p>
          <p class="text-3xl font-bold"><%= @stats.total_posts %></p>
        </div>
      </div>

      <!-- Recent Activity -->
      <div class="bg-white rounded-lg shadow p-4">
        <h3 class="font-semibold mb-4">กิจกรรมล่าสุด</h3>
        <!-- activity list -->
      </div>
    </div>
    """
  end

  defp format_number(n), do: Number.Delimit.number_to_delimited(n)
end

defmodule MyApp.Admin.Stats do
  alias MyApp.Repo
  import Ecto.Query

  def get_dashboard_stats do
    %{
      total_users: count_users(),
      active_users_7d: count_active_users(7),
      users_growth: calculate_growth(:users),
      total_posts: count_posts(),
      mrr: calculate_mrr()
    }
  end

  defp count_users do
    Repo.one(from u in MyApp.User, select: count(u.id))
  end

  defp count_active_users(days) do
    cutoff = DateTime.add(DateTime.utc_now(), -days * 86_400)
    Repo.one(
      from u in MyApp.User,
      where: u.last_seen_at >= ^cutoff,
      select: count(u.id)
    )
  end

  defp calculate_growth(:users) do
    this_month = count_new_users_this_month()
    last_month = count_new_users_last_month()

    if last_month == 0 do
      100
    else
      round((this_month - last_month) / last_month * 100)
    end
  end

  defp count_new_users_this_month do
    today = Date.utc_today()
    start = Date.new!(today.year, today.month, 1)
    |> DateTime.new!(~T[00:00:00], "Etc/UTC")

    Repo.one(from u in MyApp.User, where: u.inserted_at >= ^start, select: count(u.id))
  end

  defp count_new_users_last_month do
    today = Date.utc_today()
    last_month = Date.add(Date.new!(today.year, today.month, 1), -1)
    start = Date.new!(last_month.year, last_month.month, 1)
    |> DateTime.new!(~T[00:00:00], "Etc/UTC")
    end_date = Date.new!(today.year, today.month, 1)
    |> DateTime.new!(~T[00:00:00], "Etc/UTC")

    Repo.one(
      from u in MyApp.User,
      where: u.inserted_at >= ^start and u.inserted_at < ^end_date,
      select: count(u.id)
    )
  end

  defp count_posts, do: Repo.one(from p in MyApp.Post, select: count(p.id))

  defp calculate_mrr do
    # Query active subscriptions
    Repo.one(
      from s in MyApp.Subscription,
      join: p in assoc(s, :plan),
      where: s.status == "active",
      select: sum(p.price_monthly)
    ) || Decimal.new(0)
  end
end
```

---

## 3. User Management

```elixir
defmodule MyAppWeb.Admin.UsersLive do
  use MyAppWeb, :live_view

  alias MyApp.{Accounts, Admin}

  def mount(params, _session, socket) do
    users = Accounts.list_users(search: params["search"], page: 1)
    {:ok, assign(socket, users: users, search: params["search"] || "")}
  end

  def handle_event("search", %{"query" => query}, socket) do
    users = Accounts.list_users(search: query)
    {:noreply, assign(socket, users: users, search: query)}
  end

  def handle_event("suspend_user", %{"id" => id}, socket) do
    user = Accounts.get_user!(id)
    {:ok, _} = Admin.suspend_user(user, socket.assigns.current_user)
    users = Accounts.list_users(search: socket.assigns.search)
    {:noreply, put_flash(assign(socket, users: users), :info, "ระงับบัญชีผู้ใช้แล้ว")}
  end

  def handle_event("restore_user", %{"id" => id}, socket) do
    user = Accounts.get_user!(id)
    {:ok, _} = Admin.restore_user(user, socket.assigns.current_user)
    users = Accounts.list_users(search: socket.assigns.search)
    {:noreply, put_flash(assign(socket, users: users), :info, "กู้คืนบัญชีผู้ใช้แล้ว")}
  end

  def handle_event("impersonate", %{"id" => id}, socket) do
    user = Accounts.get_user!(id)
    # Log impersonation for audit
    Admin.log_action(socket.assigns.current_user, "impersonate", %{target_user_id: user.id})
    token = MyAppWeb.UserAuth.generate_user_session_token(user)
    {:noreply, push_navigate(socket, to: "/users/log_in?token=#{token}&impersonating=true")}
  end

  def render(assigns) do
    ~H"""
    <div class="p-6">
      <h1 class="text-2xl font-bold mb-6">จัดการผู้ใช้</h1>

      <form phx-change="search" class="mb-4">
        <input
          type="text"
          name="query"
          value={@search}
          placeholder="ค้นหาด้วยชื่อหรืออีเมล..."
          class="border rounded-lg px-4 py-2 w-64"
          phx-debounce="300"
        />
      </form>

      <table class="w-full bg-white shadow rounded-lg">
        <thead class="bg-gray-50">
          <tr>
            <th class="px-4 py-3 text-left">ผู้ใช้</th>
            <th class="px-4 py-3 text-left">สมัครเมื่อ</th>
            <th class="px-4 py-3 text-left">สถานะ</th>
            <th class="px-4 py-3 text-left">แผน</th>
            <th class="px-4 py-3"></th>
          </tr>
        </thead>
        <tbody>
          <%= for user <- @users do %>
            <tr class="border-t">
              <td class="px-4 py-3">
                <div class="font-medium"><%= user.name %></div>
                <div class="text-sm text-gray-500"><%= user.email %></div>
              </td>
              <td class="px-4 py-3 text-sm text-gray-500">
                <%= Calendar.strftime(user.inserted_at, "%d/%m/%Y") %>
              </td>
              <td class="px-4 py-3">
                <span class={"px-2 py-1 rounded text-xs font-medium #{status_color(user.status)}"}>
                  <%= user.status %>
                </span>
              </td>
              <td class="px-4 py-3 text-sm">
                <%= user.subscription_plan || "Free" %>
              </td>
              <td class="px-4 py-3">
                <div class="flex gap-2">
                  <.link navigate={~p"/admin/users/#{user.id}"} class="text-blue-600 text-sm">
                    ดู
                  </.link>
                  <%= if user.status == "active" do %>
                    <button phx-click="suspend_user" phx-value-id={user.id}
                      class="text-orange-600 text-sm">
                      ระงับ
                    </button>
                  <% else %>
                    <button phx-click="restore_user" phx-value-id={user.id}
                      class="text-green-600 text-sm">
                      กู้คืน
                    </button>
                  <% end %>
                  <button phx-click="impersonate" phx-value-id={user.id}
                    class="text-purple-600 text-sm">
                    แอบดู
                  </button>
                </div>
              </td>
            </tr>
          <% end %>
        </tbody>
      </table>
    </div>
    """
  end

  defp status_color("active"), do: "bg-green-100 text-green-800"
  defp status_color("suspended"), do: "bg-red-100 text-red-800"
  defp status_color(_), do: "bg-gray-100 text-gray-800"
end
```

---

## 4. Audit Log

```elixir
defmodule MyApp.Admin.AuditLog do
  alias MyApp.Repo
  alias MyApp.Admin.AuditEntry
  import Ecto.Query

  def log(admin_user, action, metadata \\ %{}) do
    %AuditEntry{}
    |> AuditEntry.changeset(%{
      admin_id: admin_user.id,
      action: action,
      metadata: metadata,
      ip_address: metadata[:ip_address],
      user_agent: metadata[:user_agent]
    })
    |> Repo.insert()
  end

  def list_recent(limit \\ 50) do
    from(a in AuditEntry,
      order_by: [desc: a.inserted_at],
      limit: ^limit,
      preload: [:admin]
    )
    |> Repo.all()
  end

  def for_user(user_id) do
    from(a in AuditEntry,
      where: a.metadata["target_user_id"] == ^to_string(user_id),
      order_by: [desc: a.inserted_at],
      preload: [:admin]
    )
    |> Repo.all()
  end
end
```

---

## สรุป

```
Admin Panel:
├── Dashboard: metrics, KPIs
├── User Management: list, suspend, restore, impersonate
├── Audit Log: track admin actions
└── Role-based access

Features:
├── Real-time metrics refresh
├── User impersonation (with audit trail)
├── Search and filtering
└── Action confirmation dialogs
```

---

*ก่อนหน้า: [Part 71](part_71.md) | ต่อไป: [Part 73 - Data Import/Export](part_73.md)*
