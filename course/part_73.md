# Part 73: Data Import/Export (นำเข้าและส่งออกข้อมูล)

## เป้าหมายการเรียนรู้

หลังจากจบ Part นี้ คุณจะสามารถ:
- Import ข้อมูลจาก CSV/Excel
- Export ข้อมูลเป็น CSV/Excel/PDF
- Background import jobs
- Data validation ระหว่าง import

---

## 1. CSV Import

```elixir
# mix.exs
{:nimble_csv, "~> 1.2"},
{:xlsx_parser, "~> 0.5"}

defmodule MyApp.Import.CsvImporter do
  alias NimbleCSV.RFC4180, as: CSV
  alias MyApp.Repo

  def import_users(file_path) do
    stream = File.stream!(file_path)

    results = CSV.parse_stream(stream)
    |> Stream.with_index()
    |> Stream.map(fn {row, index} ->
      line = index + 2  # +1 for header, +1 for 1-indexed
      parse_user_row(row, line)
    end)
    |> Enum.to_list()

    valid = Enum.filter(results, &match?({:ok, _}, &1))
    errors = Enum.filter(results, &match?({:error, _, _}, &1))

    # Bulk insert valid records
    users = Enum.map(valid, fn {:ok, attrs} -> attrs end)
    now = DateTime.utc_now() |> DateTime.truncate(:second)
    users_with_timestamps = Enum.map(users, &Map.merge(&1, %{inserted_at: now, updated_at: now}))

    {inserted_count, _} = Repo.insert_all(MyApp.User, users_with_timestamps,
      on_conflict: :nothing,
      conflict_target: [:email]
    )

    %{
      total: length(results),
      imported: inserted_count,
      errors: Enum.map(errors, fn {:error, line, reason} ->
        %{line: line, reason: reason}
      end)
    }
  end

  defp parse_user_row([name, email, role], line) do
    with :ok <- validate_email(email),
         :ok <- validate_role(role) do
      {:ok, %{name: String.trim(name), email: String.trim(email), role: String.trim(role)}}
    else
      {:error, reason} -> {:error, line, reason}
    end
  end

  defp parse_user_row(row, line) do
    {:error, line, "Invalid column count: expected 3, got #{length(row)}"}
  end

  defp validate_email(email) do
    if String.match?(String.trim(email), ~r/^[^\s@]+@[^\s@]+\.[^\s@]+$/) do
      :ok
    else
      {:error, "Invalid email: #{email}"}
    end
  end

  defp validate_role(role) do
    if String.trim(role) in ["admin", "user", "viewer"] do
      :ok
    else
      {:error, "Invalid role: #{role}"}
    end
  end
end
```

---

## 2. LiveView File Upload สำหรับ Import

```elixir
defmodule MyAppWeb.Admin.ImportLive do
  use MyAppWeb, :live_view

  @max_file_size 10 * 1024 * 1024  # 10MB

  def mount(_params, _session, socket) do
    {:ok,
     socket
     |> assign(upload_status: nil, import_results: nil)
     |> allow_upload(:csv_file,
       accept: ~w(.csv .xlsx),
       max_entries: 1,
       max_file_size: @max_file_size
     )
    }
  end

  def handle_event("validate", _params, socket) do
    {:noreply, socket}
  end

  def handle_event("import", _params, socket) do
    [{:ok, results}] =
      consume_uploaded_entries(socket, :csv_file, fn %{path: path}, entry ->
        extension = Path.extname(entry.client_name)
        result = case extension do
          ".csv" -> MyApp.Import.CsvImporter.import_users(path)
          ".xlsx" -> MyApp.Import.ExcelImporter.import_users(path)
          _ -> {:error, "Unsupported file format"}
        end
        {:ok, result}
      end)

    {:noreply, assign(socket, import_results: results, upload_status: :done)}
  end

  def render(assigns) do
    ~H"""
    <div class="p-6 max-w-2xl">
      <h1 class="text-2xl font-bold mb-6">นำเข้าผู้ใช้จาก CSV</h1>

      <div class="mb-4 bg-blue-50 p-4 rounded">
        <p class="text-sm font-semibold">รูปแบบ CSV ที่รองรับ:</p>
        <code class="text-xs">name,email,role</code><br>
        <code class="text-xs">John Doe,john@example.com,user</code>
      </div>

      <form phx-submit="import" phx-change="validate">
        <div class="border-2 border-dashed rounded-lg p-8 text-center mb-4"
          phx-drop-target={@uploads.csv_file.ref}>
          <.live_file_input upload={@uploads.csv_file} class="hidden" />
          <p class="text-gray-500">ลากไฟล์มาวางที่นี่ หรือ</p>
          <label class="cursor-pointer text-blue-600 hover:underline"
            phx-click={JS.dispatch("click", to: "##{@uploads.csv_file.ref}")}>
            เลือกไฟล์
          </label>
        </div>

        <%= for entry <- @uploads.csv_file.entries do %>
          <div class="flex items-center gap-3 mb-2 p-3 bg-gray-50 rounded">
            <span class="text-sm"><%= entry.client_name %></span>
            <span class="text-xs text-gray-500">
              (<%= Float.round(entry.client_size / 1024, 1) %> KB)
            </span>
            <progress value={entry.progress} max="100" class="flex-1"></progress>
            <button type="button" phx-click="cancel-upload"
              phx-value-ref={entry.ref} class="text-red-500">
              ✕
            </button>
          </div>
        <% end %>

        <button type="submit"
          disabled={@uploads.csv_file.entries == []}
          class="bg-blue-600 text-white px-6 py-2 rounded disabled:opacity-50">
          นำเข้าข้อมูล
        </button>
      </form>

      <%= if @import_results do %>
        <div class="mt-6 p-4 bg-green-50 rounded">
          <p class="font-semibold text-green-800">
            นำเข้าสำเร็จ: <%= @import_results.imported %>/<%= @import_results.total %> รายการ
          </p>
          <%= if @import_results.errors != [] do %>
            <div class="mt-3">
              <p class="font-semibold text-red-700">ข้อผิดพลาด:</p>
              <ul class="text-sm text-red-600">
                <%= for error <- @import_results.errors do %>
                  <li>แถว <%= error.line %>: <%= error.reason %></li>
                <% end %>
              </ul>
            </div>
          <% end %>
        </div>
      <% end %>
    </div>
    """
  end
end
```

---

## 3. CSV Export

```elixir
defmodule MyApp.Export.CsvExporter do
  alias NimbleCSV.RFC4180, as: CSV

  def export_users(filters \\ []) do
    headers = ["ID", "Name", "Email", "Role", "Created At"]

    users = MyApp.Accounts.list_users(filters)

    rows = Enum.map(users, fn user ->
      [
        user.id,
        user.name,
        user.email,
        user.role,
        Calendar.strftime(user.inserted_at, "%Y-%m-%d %H:%M:%S")
      ]
    end)

    [headers | rows]
    |> CSV.dump_to_iodata()
    |> IO.iodata_to_binary()
  end

  def stream_large_export(filters \\ []) do
    headers = ["ID", "Name", "Email", "Role", "Created At"]

    Stream.resource(
      fn -> {0, 100} end,  # {offset, limit}
      fn {offset, limit} ->
        users = MyApp.Accounts.list_users_paginated(filters, offset, limit)
        if users == [] do
          {:halt, nil}
        else
          rows = Enum.map(users, fn user ->
            [user.id, user.name, user.email, user.role,
             Calendar.strftime(user.inserted_at, "%Y-%m-%d")]
          end)
          {rows, {offset + limit, limit}}
        end
      end,
      fn _ -> :ok end
    )
    |> (fn rows ->
      Stream.concat([[headers]], rows)
    end).()
    |> CSV.dump_to_stream()
  end
end

# Controller
defmodule MyAppWeb.Admin.ExportController do
  use MyAppWeb, :controller

  def export_users(conn, params) do
    filename = "users_#{Date.utc_today()}.csv"

    conn
    |> put_resp_content_type("text/csv")
    |> put_resp_header("content-disposition", ~s(attachment; filename="#{filename}"))
    |> send_chunked(200)
    |> stream_csv(MyApp.Export.CsvExporter.stream_large_export(params))
  end

  defp stream_csv(conn, stream) do
    Enum.reduce_while(stream, conn, fn chunk, conn ->
      case chunk(conn, chunk) do
        {:ok, conn} -> {:cont, conn}
        {:error, _} -> {:halt, conn}
      end
    end)
  end
end
```

---

## 4. Excel Export

```elixir
# mix.exs: {:elixlsx, "~> 0.5"}

defmodule MyApp.Export.ExcelExporter do
  alias Elixlsx.{Workbook, Sheet}

  def export_orders(orders) do
    headers = ["Order #", "Customer", "Total", "Status", "Date"]

    rows = Enum.map(orders, fn order ->
      [
        order.id,
        order.user.name,
        Decimal.to_string(order.total),
        order.status,
        Calendar.strftime(order.inserted_at, "%Y-%m-%d")
      ]
    end)

    sheet = %Sheet{
      name: "Orders",
      rows: [headers | rows],
      col_widths: %{1 => 10, 2 => 30, 3 => 15, 4 => 15, 5 => 15}
    }

    workbook = %Workbook{sheets: [sheet]}

    {:ok, {_filename, binary}} = Elixlsx.write_to_memory(workbook, "orders.xlsx")
    binary
  end
end
```

---

## 5. Background Import Job

```elixir
defmodule MyApp.Workers.ImportJob do
  use Oban.Worker, queue: :imports, max_attempts: 3

  @impl Oban.Worker
  def perform(%Oban.Job{args: %{"file_path" => path, "type" => type, "user_id" => user_id}}) do
    user = MyApp.Accounts.get_user!(user_id)

    result = case type do
      "users" -> MyApp.Import.CsvImporter.import_users(path)
      "products" -> MyApp.Import.CsvImporter.import_products(path)
      _ -> {:error, "Unknown import type: #{type}"}
    end

    # Notify user when done
    MyApp.Notifications.notify_import_complete(user, result)

    # Clean up temp file
    File.rm(path)

    :ok
  end
end

# Start import
def start_import(user, file_path, type) do
  %{file_path: file_path, type: type, user_id: user.id}
  |> MyApp.Workers.ImportJob.new()
  |> Oban.insert!()
end
```

---

## สรุป

```
Import/Export:
├── CSV: NimbleCSV สำหรับ parse/generate
├── Excel: Elixlsx สำหรับ .xlsx
├── PDF: Chromic PDF สำหรับ reports
└── Background: Oban สำหรับ large imports

Patterns:
├── Stream สำหรับ large file export
├── Chunked response สำหรับ downloads
├── Validation ก่อน bulk insert
└── Error reporting per row
```

---

*ก่อนหน้า: [Part 72](part_72.md) | ต่อไป: [Part 74 - Scheduler and Cron Jobs](part_74.md)*
