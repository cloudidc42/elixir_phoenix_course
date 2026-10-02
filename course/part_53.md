# Part 53: Task Management Application (แอปจัดการงาน)

## เป้าหมายการเรียนรู้

หลังจากจบ Part นี้ คุณจะสามารถ:
- สร้าง task management ด้วย LiveView
- Drag-and-drop Kanban board
- Team collaboration features
- Notifications และ activity log

---

## 1. Schema Design

```elixir
# migrations
create table(:projects) do
  add :name, :string, null: false
  add :description, :text
  add :status, :string, default: "active"
  add :owner_id, references(:users), null: false
  add :due_date, :date
  timestamps()
end

create table(:project_members) do
  add :project_id, references(:projects), null: false
  add :user_id, references(:users), null: false
  add :role, :string, default: "member"  # member, admin, viewer
  timestamps()
end

create table(:columns) do
  add :name, :string, null: false
  add :position, :integer, null: false
  add :project_id, references(:projects), null: false
  timestamps()
end

create table(:tasks) do
  add :title, :string, null: false
  add :description, :text
  add :status, :string, default: "todo"
  add :priority, :string, default: "medium"  # low, medium, high, urgent
  add :position, :float, null: false
  add :due_date, :date
  add :estimated_hours, :decimal
  add :column_id, references(:columns), null: false
  add :project_id, references(:projects), null: false
  add :assignee_id, references(:users)
  add :creator_id, references(:users), null: false
  add :labels, {:array, :string}, default: []
  timestamps()
end

create table(:task_comments) do
  add :content, :text, null: false
  add :task_id, references(:tasks), null: false
  add :author_id, references(:users), null: false
  timestamps()
end

create table(:activity_logs) do
  add :action, :string, null: false
  add :details, :map, default: %{}
  add :project_id, references(:projects)
  add :task_id, references(:tasks)
  add :user_id, references(:users), null: false
  timestamps()
end
```

---

## 2. Task Context

```elixir
defmodule TaskApp.Tasks do
  alias TaskApp.Repo
  alias TaskApp.Tasks.{Task, Column, ActivityLog}
  import Ecto.Query

  def list_project_tasks(project_id) do
    from(t in Task,
      where: t.project_id == ^project_id,
      order_by: [asc: t.column_id, asc: t.position],
      preload: [:assignee, :column]
    )
    |> Repo.all()
    |> Enum.group_by(& &1.column_id)
  end

  def create_task(attrs, creator_id) do
    position = next_position(attrs["column_id"])

    %Task{}
    |> Task.changeset(Map.merge(attrs, %{
      "creator_id" => creator_id,
      "position" => position
    }))
    |> Repo.insert()
    |> case do
      {:ok, task} ->
        log_activity(task.project_id, task.id, creator_id, "task_created", %{title: task.title})
        {:ok, task}
      error -> error
    end
  end

  def move_task(task_id, column_id, position) do
    task = Repo.get!(Task, task_id)

    task
    |> Task.changeset(%{column_id: column_id, position: position})
    |> Repo.update()
  end

  def assign_task(task_id, assignee_id, assigner_id) do
    task = Repo.get!(Task, task_id)

    task
    |> Task.changeset(%{assignee_id: assignee_id})
    |> Repo.update()
    |> case do
      {:ok, task} ->
        log_activity(task.project_id, task.id, assigner_id, "task_assigned", %{
          assignee_id: assignee_id
        })
        # Send notification to assignee
        TaskApp.Notifications.notify_assignment(task, assignee_id)
        {:ok, task}
      error -> error
    end
  end

  def complete_task(task_id, user_id) do
    task = Repo.get!(Task, task_id)

    Ecto.Multi.new()
    |> Ecto.Multi.update(:task, Task.changeset(task, %{status: "done"}))
    |> Ecto.Multi.run(:log, fn _repo, %{task: task} ->
      log_activity(task.project_id, task.id, user_id, "task_completed", %{})
      {:ok, :logged}
    end)
    |> Repo.transaction()
  end

  defp next_position(column_id) do
    case Repo.one(from t in Task,
      where: t.column_id == ^column_id,
      select: max(t.position)
    ) do
      nil -> 1.0
      max -> max + 1.0
    end
  end

  def reorder_tasks(task_id, before_id, after_id) do
    before_pos = if before_id, do: Repo.get!(Task, before_id).position, else: 0.0
    after_pos = if after_id, do: Repo.get!(Task, after_id).position, else: before_pos + 2.0

    new_position = (before_pos + after_pos) / 2

    Repo.get!(Task, task_id)
    |> Task.changeset(%{position: new_position})
    |> Repo.update()
  end

  defp log_activity(project_id, task_id, user_id, action, details) do
    %ActivityLog{}
    |> ActivityLog.changeset(%{
      project_id: project_id,
      task_id: task_id,
      user_id: user_id,
      action: action,
      details: details
    })
    |> Repo.insert()
  end
end
```

---

## 3. Kanban Board LiveView

```elixir
defmodule TaskAppWeb.KanbanLive do
  use TaskAppWeb, :live_view

  alias TaskApp.{Tasks, Projects}

  def mount(%{"project_id" => project_id}, session, socket) do
    if connected?(socket) do
      TaskApp.PubSub.subscribe("project:#{project_id}")
    end

    project = Projects.get_project!(project_id)
    columns = Tasks.list_columns(project_id)
    tasks_by_column = Tasks.list_project_tasks(project_id)

    {:ok, assign(socket,
      project: project,
      columns: columns,
      tasks_by_column: tasks_by_column,
      dragging_task: nil
    )}
  end

  def handle_event("drag_start", %{"task_id" => task_id}, socket) do
    {:noreply, assign(socket, dragging_task: task_id)}
  end

  def handle_event("drop_task", %{
    "task_id" => task_id,
    "column_id" => column_id,
    "before_id" => before_id,
    "after_id" => after_id
  }, socket) do
    {:ok, _task} = Tasks.move_task_between_columns(
      task_id,
      column_id,
      before_id && String.to_integer(before_id),
      after_id && String.to_integer(after_id)
    )

    # Broadcast to all collaborators
    Phoenix.PubSub.broadcast(
      TaskApp.PubSub,
      "project:#{socket.assigns.project.id}",
      {:task_moved, task_id, column_id}
    )

    tasks_by_column = Tasks.list_project_tasks(socket.assigns.project.id)
    {:noreply, assign(socket, tasks_by_column: tasks_by_column, dragging_task: nil)}
  end

  def handle_event("create_task", %{"title" => title, "column_id" => column_id}, socket) do
    {:ok, task} = Tasks.create_task(%{
      "title" => title,
      "column_id" => column_id,
      "project_id" => socket.assigns.project.id
    }, socket.assigns.current_user.id)

    Phoenix.PubSub.broadcast(
      TaskApp.PubSub,
      "project:#{socket.assigns.project.id}",
      {:task_created, task}
    )

    tasks_by_column = Tasks.list_project_tasks(socket.assigns.project.id)
    {:noreply, assign(socket, tasks_by_column: tasks_by_column)}
  end

  def handle_info({:task_moved, _task_id, _column_id}, socket) do
    tasks_by_column = Tasks.list_project_tasks(socket.assigns.project.id)
    {:noreply, assign(socket, tasks_by_column: tasks_by_column)}
  end

  def handle_info({:task_created, _task}, socket) do
    tasks_by_column = Tasks.list_project_tasks(socket.assigns.project.id)
    {:noreply, assign(socket, tasks_by_column: tasks_by_column)}
  end

  def render(assigns) do
    ~H"""
    <div class="p-4">
      <h1 class="text-2xl font-bold mb-6"><%= @project.name %></h1>

      <div class="flex gap-4 overflow-x-auto" id="kanban-board" phx-hook="KanbanDrag">
        <%= for column <- @columns do %>
          <div
            class="w-72 bg-gray-100 rounded-lg p-3 flex-shrink-0"
            data-column-id={column.id}
          >
            <div class="flex justify-between items-center mb-3">
              <h3 class="font-semibold"><%= column.name %></h3>
              <span class="text-sm text-gray-500">
                <%= length(Map.get(@tasks_by_column, column.id, [])) %>
              </span>
            </div>

            <div class="space-y-2 min-h-16" id={"column-#{column.id}"}>
              <%= for task <- Map.get(@tasks_by_column, column.id, []) do %>
                <div
                  class={"p-3 bg-white rounded shadow-sm cursor-grab #{priority_color(task.priority)}"}
                  data-task-id={task.id}
                  draggable="true"
                >
                  <p class="font-medium text-sm"><%= task.title %></p>
                  <%= if task.assignee do %>
                    <div class="flex items-center gap-1 mt-2">
                      <div class="w-5 h-5 bg-blue-500 rounded-full text-white text-xs flex items-center justify-center">
                        <%= String.first(task.assignee.name) %>
                      </div>
                      <span class="text-xs text-gray-500"><%= task.assignee.name %></span>
                    </div>
                  <% end %>
                  <%= if task.due_date do %>
                    <p class={"text-xs mt-1 #{if Date.compare(task.due_date, Date.utc_today()) == :lt, do: "text-red-500", else: "text-gray-500"}"}>
                      <%= task.due_date %>
                    </p>
                  <% end %>
                </div>
              <% end %>
            </div>

            <button
              phx-click={JS.show(to: "#new-task-#{column.id}")}
              class="mt-2 w-full text-left text-gray-500 hover:text-gray-700 text-sm py-1"
            >
              + เพิ่มงาน
            </button>

            <div id={"new-task-#{column.id}"} class="hidden mt-2">
              <form phx-submit="create_task">
                <input type="hidden" name="column_id" value={column.id} />
                <input
                  type="text"
                  name="title"
                  placeholder="ชื่องาน..."
                  class="w-full border rounded p-2 text-sm"
                  autofocus
                />
                <div class="flex gap-2 mt-2">
                  <button type="submit" class="bg-blue-600 text-white px-3 py-1 rounded text-sm">
                    เพิ่ม
                  </button>
                  <button
                    type="button"
                    phx-click={JS.hide(to: "#new-task-#{column.id}")}
                    class="text-gray-500 text-sm"
                  >
                    ยกเลิก
                  </button>
                </div>
              </form>
            </div>
          </div>
        <% end %>
      </div>
    </div>
    """
  end

  defp priority_color("urgent"), do: "border-l-4 border-red-500"
  defp priority_color("high"), do: "border-l-4 border-orange-500"
  defp priority_color("medium"), do: "border-l-4 border-yellow-500"
  defp priority_color(_), do: "border-l-4 border-gray-300"
end
```

---

## 4. Drag and Drop JavaScript Hook

```javascript
// assets/js/hooks/kanban_drag.js
const KanbanDrag = {
  mounted() {
    this.setupDragAndDrop()
  },

  setupDragAndDrop() {
    const board = this.el

    board.addEventListener("dragstart", (e) => {
      const taskEl = e.target.closest("[data-task-id]")
      if (!taskEl) return

      e.dataTransfer.setData("task_id", taskEl.dataset.taskId)
      taskEl.classList.add("opacity-50")

      this.pushEvent("drag_start", { task_id: taskEl.dataset.taskId })
    })

    board.addEventListener("dragend", (e) => {
      e.target.classList.remove("opacity-50")
    })

    board.addEventListener("dragover", (e) => {
      e.preventDefault()
      const column = e.target.closest("[data-column-id]")
      if (column) column.classList.add("bg-blue-50")
    })

    board.addEventListener("dragleave", (e) => {
      const column = e.target.closest("[data-column-id]")
      if (column) column.classList.remove("bg-blue-50")
    })

    board.addEventListener("drop", (e) => {
      e.preventDefault()
      const taskId = e.dataTransfer.getData("task_id")
      const column = e.target.closest("[data-column-id]")
      if (!column) return

      column.classList.remove("bg-blue-50")

      const targetTask = e.target.closest("[data-task-id]")
      const beforeId = targetTask ? targetTask.dataset.taskId : null
      const afterId = targetTask?.nextElementSibling?.dataset?.taskId || null

      this.pushEvent("drop_task", {
        task_id: taskId,
        column_id: column.dataset.columnId,
        before_id: beforeId,
        after_id: afterId
      })
    })
  }
}

export default KanbanDrag
```

---

## 5. Notifications System

```elixir
defmodule TaskApp.Notifications do
  alias TaskApp.Repo
  alias TaskApp.Notifications.Notification

  def notify_assignment(task, assignee_id) do
    %Notification{}
    |> Notification.changeset(%{
      user_id: assignee_id,
      type: "task_assigned",
      message: "คุณได้รับมอบหมายงาน: #{task.title}",
      data: %{task_id: task.id, project_id: task.project_id},
      read: false
    })
    |> Repo.insert()

    # Push via PubSub to user's LiveView
    Phoenix.PubSub.broadcast(
      TaskApp.PubSub,
      "user:#{assignee_id}:notifications",
      {:new_notification, task.title}
    )
  end

  def get_unread(user_id) do
    import Ecto.Query
    from(n in Notification,
      where: n.user_id == ^user_id and n.read == false,
      order_by: [desc: n.inserted_at],
      limit: 20
    )
    |> Repo.all()
  end

  def mark_all_read(user_id) do
    import Ecto.Query
    from(n in Notification,
      where: n.user_id == ^user_id and n.read == false
    )
    |> Repo.update_all(set: [read: true])
  end
end
```

---

## สรุป

```
Task Management:
├── Projects + Columns + Tasks hierarchy
├── Kanban board with drag-and-drop
├── Real-time collaboration via PubSub
└── Activity logs + Notifications

Key Patterns:
├── Float positions for ordering (1.5, 2.5, etc.)
├── Reorder by averaging positions
├── PubSub broadcasts for collaboration
└── JS hooks for drag-and-drop
```

---

*ก่อนหน้า: [Part 52](part_52.md) | ต่อไป: [Part 54 - API Platform](part_54.md)*
