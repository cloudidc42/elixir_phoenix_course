# Part 31: Phoenix LiveView พื้นฐาน

## เป้าหมายการเรียนรู้

หลังจากจบ Part นี้ คุณจะสามารถ:
- เข้าใจว่า LiveView คืออะไรและทำงานอย่างไร
- สร้าง LiveView components ได้
- จัดการ state และ events ใน LiveView
- สร้าง real-time UI โดยไม่ต้องเขียน JavaScript

---

## 1. LiveView คืออะไร?

Phoenix LiveView ทำให้สามารถสร้าง real-time, interactive web applications โดยใช้ Elixir เพียงอย่างเดียว ไม่ต้องเขียน JavaScript

```
┌─────────────────────────────────────────────┐
│              Traditional Web                │
│                                             │
│  Browser ──HTTP──> Server ──HTTP──> Browser │
│  (Full page reload on every action)         │
└─────────────────────────────────────────────┘

┌─────────────────────────────────────────────┐
│               LiveView                      │
│                                             │
│  Browser ──WS──> LiveView Process           │
│         <──Diff─── (Only changed HTML)      │
│  (Real-time, no full page reload)           │
└─────────────────────────────────────────────┘
```

### คุณสมบัติ LiveView

1. **Real-time updates** - UI อัปเดตทันทีเมื่อ state เปลี่ยน
2. **No JavaScript needed** - สำหรับ basic interactions
3. **Server-side state** - State อยู่บน server
4. **Efficient updates** - ส่งเฉพาะ HTML ที่เปลี่ยน (diff)
5. **WebSocket** - การสื่อสารผ่าน persistent connection

---

## 2. การตั้งค่า LiveView

### เพิ่ม dependency

```elixir
# mix.exs
defp deps do
  [
    {:phoenix_live_view, "~> 0.20"},
    # ...
  ]
end
```

### Configure

```elixir
# config/config.exs
config :my_app, MyAppWeb.Endpoint,
  live_view: [signing_salt: "random_signing_salt"]
```

```javascript
// assets/js/app.js
import {Socket} from "phoenix"
import {LiveSocket} from "phoenix_live_view"

let csrfToken = document.querySelector("meta[name='csrf-token']").getAttribute("content")
let liveSocket = new LiveSocket("/live", Socket, {params: {_csrf_token: csrfToken}})
liveSocket.connect()
```

---

## 3. LiveView พื้นฐาน

### สร้าง Counter

```elixir
# lib/my_app_web/live/counter_live.ex
defmodule MyAppWeb.CounterLive do
  use MyAppWeb, :live_view

  def mount(_params, _session, socket) do
    {:ok, assign(socket, count: 0)}
  end

  def render(assigns) do
    ~H"""
    <div class="counter">
      <h1>Counter: <%= @count %></h1>
      <button phx-click="increment">+</button>
      <button phx-click="decrement">-</button>
      <button phx-click="reset">Reset</button>
    </div>
    """
  end

  def handle_event("increment", _params, socket) do
    {:noreply, assign(socket, count: socket.assigns.count + 1)}
  end

  def handle_event("decrement", _params, socket) do
    {:noreply, assign(socket, count: socket.assigns.count - 1)}
  end

  def handle_event("reset", _params, socket) do
    {:noreply, assign(socket, count: 0)}
  end
end
```

### Router

```elixir
# lib/my_app_web/router.ex
live "/counter", CounterLive
```

---

## 4. assigns และ State Management

```elixir
defmodule MyAppWeb.ShoppingCartLive do
  use MyAppWeb, :live_view

  def mount(_params, _session, socket) do
    {:ok,
     assign(socket,
       items: [],
       total: 0,
       loading: false
     )}
  end

  # Pattern: เปลี่ยน state ด้วย assign
  def handle_event("add_item", %{"product_id" => id}, socket) do
    product = get_product(id)
    new_items = [product | socket.assigns.items]
    new_total = Enum.sum(Enum.map(new_items, & &1.price))

    {:noreply,
     socket
     |> assign(items: new_items)
     |> assign(total: new_total)}
  end

  def handle_event("remove_item", %{"index" => index_str}, socket) do
    index = String.to_integer(index_str)
    new_items = List.delete_at(socket.assigns.items, index)
    new_total = Enum.sum(Enum.map(new_items, & &1.price))

    {:noreply, assign(socket, items: new_items, total: new_total)}
  end

  def render(assigns) do
    ~H"""
    <div class="cart">
      <h1>Shopping Cart</h1>

      <div :if={@loading} class="spinner">Loading...</div>

      <ul>
        <li :for={{item, index} <- Enum.with_index(@items)}>
          <%= item.name %> - ฿<%= item.price %>
          <button phx-click="remove_item" phx-value-index={index}>Remove</button>
        </li>
      </ul>

      <p>Total: ฿<%= @total %></p>
    </div>
    """
  end

  defp get_product(id) do
    %{id: id, name: "Product #{id}", price: :rand.uniform(100)}
  end
end
```

---

## 5. LiveView Forms

```elixir
defmodule MyAppWeb.UserFormLive do
  use MyAppWeb, :live_view
  alias MyApp.Accounts
  alias MyApp.Accounts.User

  def mount(_params, _session, socket) do
    changeset = Accounts.change_user(%User{})
    {:ok, assign(socket, form: to_form(changeset))}
  end

  def handle_event("validate", %{"user" => user_params}, socket) do
    changeset =
      %User{}
      |> Accounts.change_user(user_params)
      |> Map.put(:action, :validate)

    {:noreply, assign(socket, form: to_form(changeset))}
  end

  def handle_event("save", %{"user" => user_params}, socket) do
    case Accounts.create_user(user_params) do
      {:ok, user} ->
        {:noreply,
         socket
         |> put_flash(:info, "User #{user.name} created!")
         |> push_navigate(to: "/users/#{user.id}")}

      {:error, changeset} ->
        {:noreply, assign(socket, form: to_form(changeset))}
    end
  end

  def render(assigns) do
    ~H"""
    <div class="user-form">
      <h1>Create User</h1>

      <.form for={@form} phx-change="validate" phx-submit="save">
        <.input field={@form[:name]} type="text" label="Name" />
        <.input field={@form[:email]} type="email" label="Email" />
        <.input field={@form[:password]} type="password" label="Password" />

        <.button type="submit">Create User</.button>
      </.form>
    </div>
    """
  end
end
```

---

## 6. Real-time Updates

```elixir
defmodule MyAppWeb.DashboardLive do
  use MyAppWeb, :live_view

  def mount(_params, _session, socket) do
    if connected?(socket) do
      :timer.send_interval(1000, self(), :update_stats)
      Phoenix.PubSub.subscribe(MyApp.PubSub, "dashboard:updates")
    end

    {:ok,
     assign(socket,
       stats: get_stats(),
       last_updated: DateTime.utc_now()
     )}
  end

  def handle_info(:update_stats, socket) do
    {:noreply,
     assign(socket,
       stats: get_stats(),
       last_updated: DateTime.utc_now()
     )}
  end

  def handle_info({:new_order, order}, socket) do
    {:noreply,
     socket
     |> update(:stats, fn stats -> update_stats_with_order(stats, order) end)
     |> put_flash(:info, "New order received!")}
  end

  defp get_stats do
    %{
      total_users: :rand.uniform(1000),
      active_sessions: :rand.uniform(100),
      revenue_today: :rand.uniform(50000)
    }
  end

  defp update_stats_with_order(stats, _order) do
    %{stats | revenue_today: stats.revenue_today + :rand.uniform(1000)}
  end

  def render(assigns) do
    ~H"""
    <div class="dashboard">
      <h1>Dashboard</h1>
      <p>Last updated: <%= @last_updated %></p>

      <div class="stats-grid">
        <div class="stat">
          <h3>Total Users</h3>
          <p class="value"><%= @stats.total_users %></p>
        </div>
        <div class="stat">
          <h3>Active Sessions</h3>
          <p class="value"><%= @stats.active_sessions %></p>
        </div>
        <div class="stat">
          <h3>Revenue Today</h3>
          <p class="value">฿<%= :erlang.float_to_binary(@stats.revenue_today * 1.0, decimals: 2) %></p>
        </div>
      </div>
    </div>
    """
  end
end
```

---

## 7. LiveComponents

```elixir
defmodule MyAppWeb.Components.SearchComponent do
  use MyAppWeb, :live_component

  def update(%{placeholder: placeholder}, socket) do
    {:ok,
     assign(socket,
       placeholder: placeholder,
       query: "",
       results: []
     )}
  end

  def handle_event("search", %{"query" => query}, socket) do
    results = if String.length(query) >= 2 do
      perform_search(query)
    else
      []
    end

    {:noreply, assign(socket, query: query, results: results)}
  end

  def render(assigns) do
    ~H"""
    <div class="search-component">
      <input
        type="text"
        placeholder={@placeholder}
        value={@query}
        phx-change="search"
        phx-target={@myself}
      />

      <ul :if={length(@results) > 0} class="results">
        <li :for={result <- @results}>
          <a href={result.url}><%= result.title %></a>
        </li>
      </ul>
    </div>
    """
  end

  defp perform_search(query) do
    # ค้นหาจริงๆ
    [%{title: "Result for #{query}", url: "/search?q=#{query}"}]
  end
end

# ใช้ใน parent LiveView
# <.live_component module={SearchComponent} id="search" placeholder="Search..." />
```

---

## 8. Phoenix.PubSub Integration

```elixir
defmodule MyAppWeb.ChatLive do
  use MyAppWeb, :live_view
  alias Phoenix.PubSub

  @topic "chat:room"

  def mount(_params, session, socket) do
    username = session["username"] || "Anonymous"

    if connected?(socket) do
      PubSub.subscribe(MyApp.PubSub, @topic)
    end

    {:ok,
     assign(socket,
       username: username,
       messages: [],
       message: ""
     )}
  end

  def handle_event("send_message", %{"message" => msg}, socket) do
    unless String.trim(msg) == "" do
      message = %{
        id: System.unique_integer([:positive]),
        user: socket.assigns.username,
        content: msg,
        timestamp: DateTime.utc_now()
      }

      PubSub.broadcast(MyApp.PubSub, @topic, {:new_message, message})
    end

    {:noreply, assign(socket, message: "")}
  end

  def handle_info({:new_message, message}, socket) do
    {:noreply,
     update(socket, :messages, fn msgs ->
       [message | msgs] |> Enum.take(50)  # Keep last 50 messages
     end)}
  end

  def render(assigns) do
    ~H"""
    <div class="chat-room">
      <h1>Chat Room</h1>
      <p>Logged in as: <%= @username %></p>

      <div class="messages" id="messages" phx-update="prepend">
        <div :for={msg <- @messages} id={"msg-#{msg.id}"} class="message">
          <strong><%= msg.user %></strong>: <%= msg.content %>
          <small><%= Calendar.strftime(msg.timestamp, "%H:%M") %></small>
        </div>
      </div>

      <form phx-submit="send_message">
        <input
          type="text"
          name="message"
          value={@message}
          placeholder="Type a message..."
          autocomplete="off"
        />
        <button type="submit">Send</button>
      </form>
    </div>
    """
  end
end
```

---

## 9. LiveView Hooks (JavaScript)

```javascript
// assets/js/hooks.js
let Hooks = {}

// Scroll to bottom on new message
Hooks.ScrollToBottom = {
  mounted() {
    this.el.scrollTop = this.el.scrollHeight
  },
  updated() {
    this.el.scrollTop = this.el.scrollHeight
  }
}

// Auto-focus input
Hooks.AutoFocus = {
  mounted() {
    this.el.focus()
  }
}

// Clipboard copy
Hooks.CopyToClipboard = {
  mounted() {
    this.el.addEventListener("click", () => {
      const text = this.el.dataset.text
      navigator.clipboard.writeText(text)
      this.el.textContent = "Copied!"
      setTimeout(() => { this.el.textContent = "Copy" }, 2000)
    })
  }
}

export default Hooks
```

```javascript
// assets/js/app.js
import Hooks from "./hooks"

let liveSocket = new LiveSocket("/live", Socket, {
  hooks: Hooks,
  params: {_csrf_token: csrfToken}
})
```

```elixir
# ใน template
def render(assigns) do
  ~H"""
  <div id="messages" phx-hook="ScrollToBottom">
    <!-- messages -->
  </div>

  <button phx-hook="CopyToClipboard" data-text="Hello World!" id="copy-btn">
    Copy
  </button>
  """
end
```

---

## 10. ตัวอย่างจริง: Todo App

```elixir
defmodule MyAppWeb.TodoLive do
  use MyAppWeb, :live_view

  defmodule Todo do
    defstruct [:id, :title, :completed, :created_at]

    def new(title) do
      %__MODULE__{
        id: System.unique_integer([:positive]),
        title: title,
        completed: false,
        created_at: DateTime.utc_now()
      }
    end
  end

  def mount(_params, _session, socket) do
    {:ok,
     assign(socket,
       todos: sample_todos(),
       new_todo: "",
       filter: :all
     )}
  end

  def handle_event("add_todo", %{"title" => title}, socket) do
    unless String.trim(title) == "" do
      todo = Todo.new(String.trim(title))
      {:noreply,
       socket
       |> update(:todos, &[todo | &1])
       |> assign(new_todo: "")}
    else
      {:noreply, socket}
    end
  end

  def handle_event("toggle_todo", %{"id" => id_str}, socket) do
    id = String.to_integer(id_str)
    todos = Enum.map(socket.assigns.todos, fn todo ->
      if todo.id == id, do: %{todo | completed: !todo.completed}, else: todo
    end)
    {:noreply, assign(socket, todos: todos)}
  end

  def handle_event("delete_todo", %{"id" => id_str}, socket) do
    id = String.to_integer(id_str)
    todos = Enum.reject(socket.assigns.todos, &(&1.id == id))
    {:noreply, assign(socket, todos: todos)}
  end

  def handle_event("clear_completed", _params, socket) do
    todos = Enum.reject(socket.assigns.todos, & &1.completed)
    {:noreply, assign(socket, todos: todos)}
  end

  def handle_event("set_filter", %{"filter" => filter}, socket) do
    {:noreply, assign(socket, filter: String.to_atom(filter))}
  end

  def handle_event("update_new_todo", %{"value" => value}, socket) do
    {:noreply, assign(socket, new_todo: value)}
  end

  def filtered_todos(todos, :all), do: todos
  def filtered_todos(todos, :active), do: Enum.filter(todos, &(!&1.completed))
  def filtered_todos(todos, :completed), do: Enum.filter(todos, & &1.completed)

  defp sample_todos do
    [
      Todo.new("เรียน Elixir"),
      Todo.new("สร้าง Phoenix app"),
      %{Todo.new("อ่าน documentation") | completed: true},
    ]
  end

  def render(assigns) do
    ~H"""
    <div class="todo-app">
      <h1>Todo App</h1>

      <form phx-submit="add_todo">
        <input
          type="text"
          name="title"
          value={@new_todo}
          phx-change="update_new_todo"
          placeholder="What needs to be done?"
        />
        <button type="submit">Add</button>
      </form>

      <div class="filters">
        <button phx-click="set_filter" phx-value-filter="all"
                class={if @filter == :all, do: "active"}>
          All
        </button>
        <button phx-click="set_filter" phx-value-filter="active"
                class={if @filter == :active, do: "active"}>
          Active
        </button>
        <button phx-click="set_filter" phx-value-filter="completed"
                class={if @filter == :completed, do: "active"}>
          Completed
        </button>
      </div>

      <ul class="todo-list">
        <li :for={todo <- filtered_todos(@todos, @filter)}
            class={if todo.completed, do: "completed"}>
          <input
            type="checkbox"
            checked={todo.completed}
            phx-click="toggle_todo"
            phx-value-id={todo.id}
          />
          <span><%= todo.title %></span>
          <button phx-click="delete_todo" phx-value-id={todo.id}>×</button>
        </li>
      </ul>

      <div class="footer">
        <span><%= Enum.count(@todos, &(!&1.completed)) %> items left</span>
        <button phx-click="clear_completed">Clear Completed</button>
      </div>
    </div>
    """
  end
end
```

---

## แบบฝึกหัด

### Exercise 1: Counter with Step
แก้ไข Counter LiveView ให้มี:
- Input สำหรับ step size
- increment/decrement ตาม step
- Reset กลับไป 0

### Exercise 2: Real-time Clock
สร้าง LiveView ที่แสดงเวลาปัจจุบัน และอัปเดตทุกวินาที

### Exercise 3: Live Search
สร้าง search interface ที่:
- ค้นหา real-time เมื่อ user พิมพ์
- แสดง loading state
- Debounce 300ms ก่อน search

---

## เฉลย Exercise 2

```elixir
defmodule MyAppWeb.ClockLive do
  use MyAppWeb, :live_view

  def mount(_params, _session, socket) do
    if connected?(socket) do
      :timer.send_interval(1000, self(), :tick)
    end

    {:ok, assign(socket, time: DateTime.utc_now())}
  end

  def handle_info(:tick, socket) do
    {:noreply, assign(socket, time: DateTime.utc_now())}
  end

  def render(assigns) do
    ~H"""
    <div class="clock">
      <h1>🕐 Current Time</h1>
      <p class="time">
        <%= Calendar.strftime(@time, "%H:%M:%S") %>
      </p>
      <p class="date">
        <%= Calendar.strftime(@time, "%A, %B %d, %Y") %>
      </p>
    </div>
    """
  end
end
```

---

## สรุป

```
Phoenix LiveView:
├── Real-time UI ด้วย WebSocket
├── Server-side state management
├── No JavaScript สำหรับ basic interactions

Callbacks:
├── mount/3 - initialize state
├── render/1 - render HTML
├── handle_event/3 - handle user actions
└── handle_info/2 - handle messages

Features:
├── assigns - state management
├── phx-click, phx-change, phx-submit
├── Live components
├── PubSub integration
└── JavaScript hooks
```

---

*ก่อนหน้า: [Part 30](part_30.md) | ต่อไป: [Part 32 - Phoenix Channels](part_32.md)*
