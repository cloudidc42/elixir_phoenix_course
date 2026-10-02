# Part 93: LiveView Advanced Patterns (LiveView ขั้นสูง)

## เป้าหมายการเรียนรู้

หลังจากจบ Part นี้ คุณจะสามารถ:
- LiveView Streams สำหรับ large lists
- LiveView Components แบบ advanced
- Form handling ขั้นสูง
- Optimistic UI updates

---

## 1. LiveView Streams

```elixir
defmodule MyAppWeb.PostsLive do
  use MyAppWeb, :live_view

  def mount(_params, _session, socket) do
    posts = MyApp.Blog.list_posts()

    socket = stream(socket, :posts, posts)
    {:ok, socket}
  end

  def handle_event("delete", %{"id" => id}, socket) do
    post = MyApp.Blog.get_post!(id)
    {:ok, _} = MyApp.Blog.delete_post(post)

    # Remove from stream without re-fetching all
    {:noreply, stream_delete(socket, :posts, post)}
  end

  def handle_event("create", %{"post" => params}, socket) do
    case MyApp.Blog.create_post(params) do
      {:ok, post} ->
        # Prepend to stream
        {:noreply, stream_insert(socket, :posts, post, at: 0)}

      {:error, changeset} ->
        {:noreply, assign(socket, :changeset, changeset)}
    end
  end

  def render(assigns) do
    ~H"""
    <ul id="posts" phx-update="stream">
      <li :for={{dom_id, post} <- @streams.posts} id={dom_id}>
        <span><%= post.title %></span>
        <button phx-click="delete" phx-value-id={post.id}>Delete</button>
      </li>
    </ul>
    """
  end
end
```

---

## 2. Advanced LiveComponents

```elixir
defmodule MyAppWeb.Components.DataTable do
  use MyAppWeb, :live_component

  def update(assigns, socket) do
    socket =
      socket
      |> assign(assigns)
      |> assign_new(:sort_col, fn -> nil end)
      |> assign_new(:sort_dir, fn -> :asc end)
      |> assign_new(:page, fn -> 1 end)
      |> load_data()

    {:ok, socket}
  end

  defp load_data(socket) do
    %{query: query, page: page, sort_col: col, sort_dir: dir} = socket.assigns

    rows =
      query
      |> maybe_sort(col, dir)
      |> paginate(page, 20)
      |> MyApp.Repo.all()

    assign(socket, :rows, rows)
  end

  def handle_event("sort", %{"col" => col}, socket) do
    col = String.to_atom(col)
    dir = if socket.assigns.sort_col == col and socket.assigns.sort_dir == :asc, do: :desc, else: :asc
    socket = socket |> assign(sort_col: col, sort_dir: dir) |> load_data()
    {:noreply, socket}
  end

  def handle_event("page", %{"page" => page}, socket) do
    socket = socket |> assign(:page, String.to_integer(page)) |> load_data()
    {:noreply, socket}
  end

  def render(assigns) do
    ~H"""
    <div>
      <table class="w-full">
        <thead>
          <tr>
            <th :for={col <- @columns}
                phx-click="sort"
                phx-value-col={col.key}
                phx-target={@myself}
                class="cursor-pointer">
              <%= col.label %>
              <span :if={@sort_col == col.key}>
                <%= if @sort_dir == :asc, do: "↑", else: "↓" %>
              </span>
            </th>
          </tr>
        </thead>
        <tbody>
          <tr :for={row <- @rows}>
            <td :for={col <- @columns}><%= Map.get(row, col.key) %></td>
          </tr>
        </tbody>
      </table>
    </div>
    """
  end

  defp maybe_sort(query, nil, _), do: query
  defp maybe_sort(query, col, dir) do
    import Ecto.Query
    order_by(query, [{^dir, ^col}])
  end

  defp paginate(query, page, per_page) do
    import Ecto.Query
    offset = (page - 1) * per_page
    from(q in query, limit: ^per_page, offset: ^offset)
  end
end

# Usage in parent LiveView:
# <.live_component module={MyAppWeb.Components.DataTable}
#   id="users-table"
#   query={from u in User}
#   columns={[%{key: :name, label: "Name"}, %{key: :email, label: "Email"}]} />
```

---

## 3. Multi-Step Forms

```elixir
defmodule MyAppWeb.OnboardingLive do
  use MyAppWeb, :live_view

  @steps [:account, :profile, :preferences, :complete]

  def mount(_params, _session, socket) do
    {:ok, assign(socket,
      step: :account,
      data: %{},
      changesets: %{}
    )}
  end

  def handle_event("next", %{"step" => step_str, step_str => params}, socket) do
    step = String.to_atom(step_str)

    case validate_step(step, params) do
      {:ok, validated} ->
        data = Map.merge(socket.assigns.data, validated)
        next_step = next_step(step)

        if next_step == :complete do
          complete_onboarding(data, socket)
        else
          {:noreply, assign(socket, step: next_step, data: data)}
        end

      {:error, changeset} ->
        {:noreply, update(socket, :changesets, &Map.put(&1, step, changeset))}
    end
  end

  def handle_event("back", _, socket) do
    prev = prev_step(socket.assigns.step)
    {:noreply, assign(socket, :step, prev)}
  end

  defp validate_step(:account, params) do
    changeset = MyApp.Accounts.User.registration_changeset(%User{}, params)
    if changeset.valid?, do: {:ok, apply_changes(changeset) |> Map.from_struct()}, else: {:error, changeset}
  end

  defp validate_step(:profile, params) do
    # Validate profile fields
    required = ["name", "bio"]
    if Enum.all?(required, &(Map.has_key?(params, &1) and params[&1] != "")) do
      {:ok, Map.take(params, required)}
    else
      {:error, :invalid}
    end
  end

  defp complete_onboarding(data, socket) do
    case MyApp.Accounts.complete_onboarding(data) do
      {:ok, user} ->
        {:noreply,
          socket
          |> assign(step: :complete)
          |> put_flash(:info, "Welcome!")
          |> push_navigate(to: ~p"/dashboard")}

      {:error, _} ->
        {:noreply, put_flash(socket, :error, "Something went wrong")}
    end
  end

  defp next_step(step) do
    idx = Enum.find_index(@steps, &(&1 == step))
    Enum.at(@steps, idx + 1)
  end

  defp prev_step(step) do
    idx = Enum.find_index(@steps, &(&1 == step))
    Enum.at(@steps, max(0, idx - 1))
  end

  def render(assigns) do
    ~H"""
    <div class="max-w-lg mx-auto">
      <.step_indicator steps={@steps} current={@step} />

      <%= case @step do %>
        <% :account -> %>
          <.account_form changeset={@changesets[:account]} />
        <% :profile -> %>
          <.profile_form data={@data} />
        <% :preferences -> %>
          <.preferences_form />
        <% :complete -> %>
          <p>Setup complete!</p>
      <% end %>
    </div>
    """
  end
end
```

---

## 4. Optimistic UI Updates

```elixir
defmodule MyAppWeb.TodosLive do
  use MyAppWeb, :live_view

  def handle_event("toggle", %{"id" => id}, socket) do
    id = String.to_integer(id)

    # Optimistic update - update UI immediately
    socket = update(socket, :todos, fn todos ->
      Enum.map(todos, fn todo ->
        if todo.id == id, do: %{todo | done: !todo.done}, else: todo
      end)
    end)

    # Async update to DB
    Task.start(fn ->
      todo = MyApp.Todos.get_todo!(id)
      case MyApp.Todos.toggle(todo) do
        {:ok, updated} ->
          # Confirm update
          send(self(), {:todo_updated, updated})

        {:error, _} ->
          # Rollback on failure
          send(self(), {:todo_rollback, id})
      end
    end)

    {:noreply, socket}
  end

  def handle_info({:todo_updated, _updated}, socket) do
    # Already up to date from optimistic update
    {:noreply, socket}
  end

  def handle_info({:todo_rollback, id}, socket) do
    # Revert the optimistic update
    original = MyApp.Todos.get_todo!(id)
    socket = update(socket, :todos, fn todos ->
      Enum.map(todos, fn todo ->
        if todo.id == id, do: original, else: todo
      end)
    end)

    {:noreply, put_flash(socket, :error, "Failed to update. Reverted.")}
  end
end
```

---

## สรุป

```
LiveView Advanced:
├── Streams: efficient large list management
│   ├── stream/3: initialize
│   ├── stream_insert/4: add/update item
│   └── stream_delete/3: remove item
├── LiveComponents: encapsulated stateful components
├── Multi-step forms: step state + validation
└── Optimistic UI: immediate + async reconcile

Stream vs Assign:
├── Streams: >100 items, frequent updates
└── Assign: <100 items, less frequent

LiveComponent lifecycle:
├── update/2: called on parent render
├── handle_event/3: local events
└── send_update/3: parent updates child
```

---

*ก่อนหน้า: [Part 92](part_92.md) | ต่อไป: [Part 94 - Performance Optimization](part_94.md)*
