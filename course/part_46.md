# Part 46: LiveView Advanced (LiveView ขั้นสูง)

## เป้าหมายการเรียนรู้

- เข้าใจ Live Components ทั้งแบบ stateful และ stateless
- ใช้ JS hooks สำหรับการผสาน JavaScript ฝั่ง client
- จัดการรายการขนาดใหญ่ด้วย LiveView streams
- ทำ Optimistic UI updates เพื่อประสบการณ์ผู้ใช้ที่ดี
- ตรวจสอบ form ด้วย `phx-change`
- อัปโหลดไฟล์ใน LiveView
- สร้าง multi-step forms
- นำทางด้วย live navigation และ patching

---

## 1. Live Components — Stateless vs Stateful

### Stateless Component

**Stateless component** ไม่มี state เป็นของตัวเอง ทำงานเหมือน function component ใน React ข้อมูลไหลลงมาจาก parent LiveView เท่านั้น

```elixir
defmodule MyAppWeb.AlertComponent do
  use Phoenix.Component

  # stateless: ใช้ attr เพื่อประกาศ props
  attr :type, :string, default: "info", values: ["info", "warning", "error"]
  attr :message, :string, required: true

  def alert(assigns) do
    ~H"""
    <div class={"alert alert-#{@type}"}>
      <p><%= @message %></p>
    </div>
    """
  end
end
```

ใช้งานใน template:

```heex
<.alert type="warning" message="กรุณาบันทึกข้อมูลก่อนออก" />
```

### Stateful Component

**Stateful component** มี state เป็นของตัวเอง มี lifecycle (`mount/3`, `update/2`, `handle_event/3`) และสื่อสารกับ parent ผ่าน `send_update/2` หรือ message

```elixir
defmodule MyAppWeb.CounterComponent do
  use Phoenix.LiveComponent

  def mount(socket) do
    {:ok, assign(socket, count: 0)}
  end

  def update(%{initial: initial}, socket) do
    {:ok, assign(socket, count: initial)}
  end

  def render(assigns) do
    ~H"""
    <div id={@id} class="counter">
      <button phx-click="decrement" phx-target={@myself}>-</button>
      <span class="count"><%= @count %></span>
      <button phx-click="increment" phx-target={@myself}>+</button>
    </div>
    """
  end

  def handle_event("increment", _, socket) do
    {:noreply, update(socket, :count, &(&1 + 1))}
  end

  def handle_event("decrement", _, socket) do
    {:noreply, update(socket, :count, &(&1 - 1))}
  end
end
```

เรียกใช้จาก LiveView หลัก:

```heex
<.live_component module={MyAppWeb.CounterComponent} id="counter-1" initial={5} />
```

> **หมายเหตุ:** `phx-target={@myself}` จำเป็นมากเพื่อให้ event ส่งไปยัง component ตัวนี้ ไม่ใช่ LiveView หลัก

---

## 2. JS Hooks — ผสาน JavaScript ฝั่ง Client

เมื่อต้องการใช้ library JavaScript หรือจัดการ DOM โดยตรง ใช้ **JS Hooks**

### สร้าง Hook

```javascript
// assets/js/hooks/chart_hook.js
const ChartHook = {
  mounted() {
    // เรียกใช้ Chart.js หรือ library ใดก็ได้
    const ctx = this.el.querySelector("canvas");
    this.chart = new Chart(ctx, {
      type: "line",
      data: JSON.parse(this.el.dataset.chartData),
    });

    // รับข้อมูลจาก server ผ่าน pushEvent
    this.handleEvent("update_chart", ({ data }) => {
      this.chart.data.datasets[0].data = data;
      this.chart.update();
    });
  },

  updated() {
    // เรียกเมื่อ element ถูก update
    const newData = JSON.parse(this.el.dataset.chartData);
    this.chart.data = newData;
    this.chart.update();
  },

  destroyed() {
    // cleanup เมื่อ element ถูกลบ
    this.chart.destroy();
  },
};

export default ChartHook;
```

### ลงทะเบียน Hook ใน app.js

```javascript
// assets/js/app.js
import ChartHook from "./hooks/chart_hook";

const liveSocket = new LiveSocket("/live", Socket, {
  hooks: { ChartHook },
  params: { _csrf_token: csrfToken },
});
```

### ใช้งานใน template

```heex
<div
  id="sales-chart"
  phx-hook="ChartHook"
  data-chart-data={Jason.encode!(@chart_data)}
>
  <canvas width="400" height="200"></canvas>
</div>
```

### ส่ง event จาก server ไป client

```elixir
def handle_info({:update_chart, data}, socket) do
  {:noreply, push_event(socket, "update_chart", %{data: data})}
end
```

---

## 3. LiveView Streams — รายการขนาดใหญ่

**Streams** แก้ปัญหา memory ของการเก็บรายการใหญ่ใน socket assigns โดย server เก็บเฉพาะ ID ของแต่ละ item ไม่ใช่ข้อมูลทั้งหมด

```elixir
defmodule MyAppWeb.PostsLive do
  use Phoenix.LiveView

  def mount(_params, _session, socket) do
    # stream_configure กำหนด key สำหรับ DOM id
    socket =
      socket
      |> stream_configure(:posts, dom_id: &"post-#{&1.id}")
      |> stream(:posts, Blog.list_posts())

    {:ok, socket}
  end

  def handle_event("delete", %{"id" => id}, socket) do
    post = Blog.get_post!(id)
    {:ok, _} = Blog.delete_post(post)

    # ลบออกจาก stream โดยไม่ต้อง re-render ทั้งหมด
    {:noreply, stream_delete(socket, :posts, post)}
  end

  def handle_event("load_more", _, socket) do
    more_posts = Blog.list_posts(after: socket.assigns.cursor)
    {:noreply, stream_insert(socket, :posts, more_posts, at: -1)}
  end

  def render(assigns) do
    ~H"""
    <ul id="posts-list" phx-update="stream">
      <li :for={{dom_id, post} <- @streams.posts} id={dom_id}>
        <h3><%= post.title %></h3>
        <button phx-click="delete" phx-value-id={post.id}>ลบ</button>
      </li>
    </ul>
    <button phx-click="load_more">โหลดเพิ่มเติม</button>
    """
  end
end
```

> **สำคัญ:** ต้องใช้ `phx-update="stream"` บน container element และแต่ละ item ต้องมี `id` ตรงกับ `dom_id`

---

## 4. Optimistic UI Updates

Optimistic UI คือการอัปเดต UI ทันทีก่อนที่ server จะตอบกลับ เพื่อให้ UI รู้สึกเร็วขึ้น

```elixir
defmodule MyAppWeb.TodoLive do
  use Phoenix.LiveView

  def mount(_params, _session, socket) do
    {:ok, stream(socket, :todos, Todos.list_todos())}
  end

  def handle_event("toggle", %{"id" => id}, socket) do
    todo = Todos.get_todo!(String.to_integer(id))

    # 1. อัปเดต UI ทันที (optimistic)
    optimistic_todo = %{todo | completed: !todo.completed}
    socket = stream_insert(socket, :todos, optimistic_todo)

    # 2. บันทึกจริงใน background
    case Todos.toggle_todo(todo) do
      {:ok, updated_todo} ->
        # ยืนยันด้วยข้อมูลจริงจาก DB
        {:noreply, stream_insert(socket, :todos, updated_todo)}

      {:error, _reason} ->
        # rollback: คืนค่าเดิม
        socket =
          socket
          |> stream_insert(:todos, todo)
          |> put_flash(:error, "เกิดข้อผิดพลาด กรุณาลองใหม่")

        {:noreply, socket}
    end
  end
end
```

---

## 5. Form Validation ด้วย phx-change

`phx-change` ส่ง event ทุกครั้งที่ผู้ใช้เปลี่ยนค่าใน form ทำให้ validate แบบ real-time ได้

```elixir
defmodule MyAppWeb.UserRegistrationLive do
  use Phoenix.LiveView

  alias MyApp.Accounts
  alias MyApp.Accounts.User

  def mount(_params, _session, socket) do
    changeset = Accounts.change_user_registration(%User{})
    {:ok, assign(socket, form: to_form(changeset, as: "user"))}
  end

  # phx-change: validate เมื่อผู้ใช้พิมพ์
  def handle_event("validate", %{"user" => user_params}, socket) do
    changeset =
      %User{}
      |> Accounts.change_user_registration(user_params)
      |> Map.put(:action, :validate)

    {:noreply, assign(socket, form: to_form(changeset, as: "user"))}
  end

  # phx-submit: บันทึกเมื่อ submit form
  def handle_event("save", %{"user" => user_params}, socket) do
    case Accounts.register_user(user_params) do
      {:ok, user} ->
        {:noreply,
         socket
         |> put_flash(:info, "สมัครสมาชิกสำเร็จ!")
         |> redirect(to: "/users/#{user.id}")}

      {:error, %Ecto.Changeset{} = changeset} ->
        {:noreply, assign(socket, form: to_form(changeset, as: "user"))}
    end
  end

  def render(assigns) do
    ~H"""
    <.form for={@form} phx-change="validate" phx-submit="save">
      <.input field={@form[:email]} type="email" label="อีเมล" />
      <.input field={@form[:password]} type="password" label="รหัสผ่าน" />
      <.input field={@form[:password_confirmation]} type="password" label="ยืนยันรหัสผ่าน" />
      <.button type="submit">สมัครสมาชิก</.button>
    </.form>
    """
  end
end
```

---

## 6. File Uploads ใน LiveView

Phoenix LiveView รองรับ file upload โดยตรงโดยไม่ต้องใช้ JavaScript พิเศษ

```elixir
defmodule MyAppWeb.ProfilePhotoLive do
  use Phoenix.LiveView

  @max_file_size 5_000_000  # 5MB

  def mount(_params, _session, socket) do
    {:ok,
     socket
     |> assign(:uploaded_files, [])
     |> allow_upload(:photo,
       accept: ~w(.jpg .jpeg .png .webp),
       max_entries: 1,
       max_file_size: @max_file_size,
       auto_upload: true
     )}
  end

  def handle_event("validate", _params, socket) do
    {:noreply, socket}
  end

  def handle_event("save", _params, socket) do
    uploaded_files =
      consume_uploaded_entries(socket, :photo, fn %{path: path}, entry ->
        # path คือ temporary file path
        dest = Path.join([:code.priv_dir(:my_app), "static", "uploads", entry.client_name])
        File.cp!(path, dest)
        {:ok, "/uploads/#{entry.client_name}"}
      end)

    {:noreply,
     socket
     |> update(:uploaded_files, &(&1 ++ uploaded_files))
     |> put_flash(:info, "อัปโหลดรูปสำเร็จ!")}
  end

  def render(assigns) do
    ~H"""
    <form phx-submit="save" phx-change="validate">
      <.live_file_input upload={@uploads.photo} />

      <%= for entry <- @uploads.photo.entries do %>
        <div class="upload-preview">
          <.live_img_preview entry={entry} width="200" />
          <p><%= entry.client_name %> — <%= entry.progress %>%</p>

          <%= for err <- upload_errors(@uploads.photo, entry) do %>
            <p class="error"><%= error_to_string(err) %></p>
          <% end %>
        </div>
      <% end %>

      <button type="submit">บันทึก</button>
    </form>
    """
  end

  defp error_to_string(:too_large), do: "ไฟล์ใหญ่เกินไป (สูงสุด 5MB)"
  defp error_to_string(:not_accepted), do: "ประเภทไฟล์ไม่รองรับ"
  defp error_to_string(:too_many_files), do: "เลือกได้สูงสุด 1 ไฟล์"
end
```

---

## 7. Multi-Step Forms

การสร้าง form หลายขั้นตอนด้วย LiveView โดยใช้ state เพื่อติดตาม step ปัจจุบัน

```elixir
defmodule MyAppWeb.CheckoutLive do
  use Phoenix.LiveView

  @steps [:cart, :shipping, :payment, :confirmation]

  def mount(_params, _session, socket) do
    {:ok,
     assign(socket,
       step: :cart,
       order: %{items: [], shipping: %{}, payment: %{}}
     )}
  end

  def handle_event("next_step", %{"step_data" => data}, socket) do
    current_step = socket.assigns.step
    order = merge_step_data(socket.assigns.order, current_step, data)
    next = next_step(current_step)

    socket =
      socket
      |> assign(order: order)
      |> assign(step: next)

    {:noreply, socket}
  end

  def handle_event("prev_step", _, socket) do
    prev = prev_step(socket.assigns.step)
    {:noreply, assign(socket, step: prev)}
  end

  def handle_event("submit_order", _, socket) do
    case Orders.create_order(socket.assigns.order) do
      {:ok, order} ->
        {:noreply,
         socket
         |> assign(step: :confirmation)
         |> assign(order_id: order.id)}

      {:error, reason} ->
        {:noreply, put_flash(socket, :error, "เกิดข้อผิดพลาด: #{reason}")}
    end
  end

  def render(assigns) do
    ~H"""
    <div class="checkout">
      <!-- Progress indicator -->
      <div class="steps">
        <span :for={step <- @steps} class={step_class(@step, step)}>
          <%= step_label(step) %>
        </span>
      </div>

      <!-- Step content -->
      <div class="step-content">
        <%= render_step(assigns) %>
      </div>
    </div>
    """
  end

  defp render_step(%{step: :cart} = assigns) do
    ~H"""
    <h2>ตะกร้าสินค้า</h2>
    <form phx-submit="next_step">
      <!-- cart items -->
      <button type="submit">ถัดไป: ที่อยู่จัดส่ง</button>
    </form>
    """
  end

  defp render_step(%{step: :shipping} = assigns) do
    ~H"""
    <h2>ที่อยู่จัดส่ง</h2>
    <form phx-submit="next_step">
      <input name="step_data[name]" placeholder="ชื่อ-นามสกุล" />
      <input name="step_data[address]" placeholder="ที่อยู่" />
      <button phx-click="prev_step" type="button">ย้อนกลับ</button>
      <button type="submit">ถัดไป: ชำระเงิน</button>
    </form>
    """
  end

  defp render_step(%{step: :payment} = assigns) do
    ~H"""
    <h2>ชำระเงิน</h2>
    <!-- payment form -->
    """
  end

  defp render_step(%{step: :confirmation} = assigns) do
    ~H"""
    <h2>สั่งซื้อสำเร็จ!</h2>
    <p>หมายเลขคำสั่งซื้อ: <%= @order_id %></p>
    """
  end

  defp next_step(:cart), do: :shipping
  defp next_step(:shipping), do: :payment
  defp next_step(:payment), do: :confirmation

  defp prev_step(:shipping), do: :cart
  defp prev_step(:payment), do: :shipping

  defp step_class(current, step) when current == step, do: "step active"
  defp step_class(_current, _step), do: "step"

  defp step_label(:cart), do: "ตะกร้า"
  defp step_label(:shipping), do: "จัดส่ง"
  defp step_label(:payment), do: "ชำระเงิน"
  defp step_label(:confirmation), do: "ยืนยัน"

  defp merge_step_data(order, :shipping, data), do: %{order | shipping: data}
  defp merge_step_data(order, :payment, data), do: %{order | payment: data}
  defp merge_step_data(order, _, _), do: order
end
```

---

## 8. Live Navigation และ Patching

LiveView มีสองวิธีในการนำทางโดยไม่ต้อง reload หน้า:

### `push_patch/2` — อัปเดต URL โดยไม่ re-mount

```elixir
defmodule MyAppWeb.PostsLive do
  use Phoenix.LiveView

  def mount(_params, _session, socket) do
    {:ok, assign(socket, posts: Blog.list_posts(), filter: "all")}
  end

  # handle_params เรียกเมื่อ URL เปลี่ยน (ทั้ง mount และ patch)
  def handle_params(%{"filter" => filter}, _url, socket) do
    posts = Blog.list_posts(filter: filter)
    {:noreply, assign(socket, posts: posts, filter: filter)}
  end

  def handle_params(_params, _url, socket) do
    {:noreply, assign(socket, posts: Blog.list_posts(), filter: "all")}
  end

  def handle_event("filter", %{"type" => type}, socket) do
    # patch เปลี่ยน URL และเรียก handle_params โดยไม่ re-mount
    {:noreply, push_patch(socket, to: ~p"/posts?filter=#{type}")}
  end

  def render(assigns) do
    ~H"""
    <div>
      <.link patch={~p"/posts?filter=all"}>ทั้งหมด</.link>
      <.link patch={~p"/posts?filter=published"}>เผยแพร่แล้ว</.link>
      <.link patch={~p"/posts?filter=draft"}>ฉบับร่าง</.link>

      <ul>
        <li :for={post <- @posts}>
          <.link navigate={~p"/posts/#{post.id}"}><%= post.title %></.link>
        </li>
      </ul>
    </div>
    """
  end
end
```

### `push_navigate/2` — นำทางไป LiveView ใหม่ (re-mount)

```elixir
def handle_event("go_to_post", %{"id" => id}, socket) do
  # navigate จะ mount LiveView ใหม่
  {:noreply, push_navigate(socket, to: ~p"/posts/#{id}")}
end
```

### ความแตกต่าง `patch` vs `navigate`

| Feature | `push_patch` | `push_navigate` |
|---------|-------------|-----------------|
| Re-mount LiveView | ไม่ | ใช่ |
| เรียก `handle_params` | ใช่ | ใช่ |
| ใช้เมื่อ | filter/sort/pagination | เปลี่ยนหน้าจริง |

---

## สรุป

```
LiveView Advanced Components
├── Stateless Components  → ใช้ Phoenix.Component, ไม่มี state
├── Stateful Components   → ใช้ Phoenix.LiveComponent, มี lifecycle
├── JS Hooks              → mounted/updated/destroyed callbacks
├── Streams               → DOM-only list management, ประหยัด memory
├── Optimistic UI         → อัปเดต UI ก่อน server ตอบ
├── phx-change            → real-time form validation
├── File Uploads          → allow_upload + consume_uploaded_entries
├── Multi-step Forms      → state machine ด้วย assigns
└── Navigation
    ├── push_patch        → เปลี่ยน URL, ไม่ re-mount
    └── push_navigate     → เปลี่ยน URL, re-mount LiveView ใหม่
```

---

*ก่อนหน้า: [Part 45 - LiveView Basics](part_45.md) | ต่อไป: [Part 47 - GenStage and Flow](part_47.md)*
