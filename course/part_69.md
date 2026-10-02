# Part 69: Content Management System (ระบบจัดการเนื้อหา)

## เป้าหมายการเรียนรู้

หลังจากจบ Part นี้ คุณจะสามารถ:
- สร้าง CMS ด้วย Phoenix LiveView
- Rich text editor integration
- Media management
- Content versioning

---

## 1. Content Schema with Rich Text

```elixir
# mix.exs: {:ex_doc, "~> 0.31"}

defmodule MyApp.CMS.Page do
  use Ecto.Schema
  import Ecto.Changeset

  schema "cms_pages" do
    field :title, :string
    field :slug, :string
    field :content, :string       # HTML content from rich text editor
    field :content_json, :map     # Tiptap/ProseMirror JSON
    field :meta_title, :string
    field :meta_description, :string
    field :meta_image, :string
    field :status, :string, default: "draft"  # draft, published, archived
    field :published_at, :utc_datetime
    field :template, :string, default: "default"  # which template to use

    belongs_to :author, MyApp.User
    belongs_to :category, MyApp.CMS.Category
    has_many :page_versions, MyApp.CMS.PageVersion

    timestamps()
  end

  def changeset(page, attrs) do
    page
    |> cast(attrs, [:title, :slug, :content, :content_json, :meta_title,
                     :meta_description, :status, :template, :author_id, :category_id])
    |> validate_required([:title, :status, :author_id])
    |> maybe_generate_slug()
    |> unique_constraint(:slug)
    |> validate_inclusion(:status, ["draft", "published", "archived"])
  end

  defp maybe_generate_slug(changeset) do
    case get_field(changeset, :slug) do
      nil ->
        title = get_field(changeset, :title) || ""
        put_change(changeset, :slug, Slug.slugify(title))
      _ -> changeset
    end
  end
end
```

---

## 2. CMS Context

```elixir
defmodule MyApp.CMS do
  alias MyApp.Repo
  alias MyApp.CMS.{Page, PageVersion}
  import Ecto.Query

  def list_pages(opts \\ []) do
    status = Keyword.get(opts, :status, "published")

    from(p in Page,
      where: p.status == ^status,
      order_by: [desc: p.inserted_at],
      preload: [:author, :category]
    )
    |> Repo.all()
  end

  def get_page_by_slug!(slug) do
    Repo.one!(
      from p in Page,
      where: p.slug == ^slug and p.status == "published",
      preload: [:author, :category]
    )
  end

  def create_page(attrs, author) do
    Ecto.Multi.new()
    |> Ecto.Multi.insert(:page, Page.changeset(%Page{}, Map.put(attrs, :author_id, author.id)))
    |> Ecto.Multi.run(:version, fn repo, %{page: page} ->
      create_version(repo, page)
    end)
    |> Repo.transaction()
    |> case do
      {:ok, %{page: page}} -> {:ok, page}
      {:error, :page, changeset, _} -> {:error, changeset}
    end
  end

  def update_page(page, attrs) do
    Ecto.Multi.new()
    |> Ecto.Multi.update(:page, Page.changeset(page, attrs))
    |> Ecto.Multi.run(:version, fn repo, %{page: updated_page} ->
      create_version(repo, updated_page)
    end)
    |> Repo.transaction()
    |> case do
      {:ok, %{page: page}} -> {:ok, page}
      {:error, :page, changeset, _} -> {:error, changeset}
    end
  end

  def publish_page(page) do
    update_page(page, %{status: "published", published_at: DateTime.utc_now()})
  end

  defp create_version(repo, page) do
    repo.insert(%PageVersion{
      page_id: page.id,
      title: page.title,
      content: page.content,
      content_json: page.content_json,
      version_number: next_version_number(repo, page.id)
    })
  end

  defp next_version_number(repo, page_id) do
    repo.one(
      from v in PageVersion,
      where: v.page_id == ^page_id,
      select: count(v.id)
    ) + 1
  end
end
```

---

## 3. LiveView Editor

```elixir
defmodule MyAppWeb.Admin.PageEditorLive do
  use MyAppWeb, :live_view
  alias MyApp.CMS

  def mount(%{"id" => id}, _session, socket) do
    page = CMS.get_page!(id)
    changeset = CMS.change_page(page)

    {:ok, assign(socket, page: page, form: to_form(changeset), preview: false)}
  end

  def mount(_params, _session, socket) do
    changeset = CMS.change_page(%CMS.Page{})
    {:ok, assign(socket, page: nil, form: to_form(changeset), preview: false)}
  end

  def handle_event("validate", %{"page" => params}, socket) do
    changeset = CMS.change_page(socket.assigns.page || %CMS.Page{}, params)
    {:noreply, assign(socket, form: to_form(changeset, action: :validate))}
  end

  def handle_event("save", %{"page" => params}, socket) do
    result = if socket.assigns.page do
      CMS.update_page(socket.assigns.page, params)
    else
      CMS.create_page(params, socket.assigns.current_user)
    end

    case result do
      {:ok, page} ->
        {:noreply,
         socket
         |> put_flash(:info, "บันทึกสำเร็จ")
         |> push_navigate(to: ~p"/admin/pages/#{page.id}/edit")}

      {:error, changeset} ->
        {:noreply, assign(socket, form: to_form(changeset))}
    end
  end

  def handle_event("publish", _, socket) do
    case CMS.publish_page(socket.assigns.page) do
      {:ok, page} ->
        {:noreply, put_flash(assign(socket, page: page), :info, "เผยแพร่สำเร็จ")}
      {:error, _} ->
        {:noreply, put_flash(socket, :error, "เกิดข้อผิดพลาด")}
    end
  end

  def handle_event("content_updated", %{"content" => html, "json" => json}, socket) do
    # Receive from JS TipTap editor hook
    changeset = CMS.change_page(
      socket.assigns.page || %CMS.Page{},
      %{content: html, content_json: json}
    )
    {:noreply, assign(socket, form: to_form(changeset))}
  end

  def render(assigns) do
    ~H"""
    <div class="flex h-screen">
      <!-- Editor Sidebar -->
      <div class="w-80 border-r p-4 overflow-y-auto">
        <h2 class="font-semibold mb-4">Page Settings</h2>
        <.form for={@form} phx-change="validate" phx-submit="save">
          <div class="space-y-4">
            <.input field={@form[:title]} label="Title" />
            <.input field={@form[:slug]} label="Slug" />
            <.input field={@form[:meta_description]} label="Meta Description"
              type="textarea" rows="3" />
            <.input field={@form[:template]} label="Template" type="select"
              options={[{"Default", "default"}, {"Landing", "landing"}, {"Blog", "blog"}]} />
          </div>

          <div class="flex gap-2 mt-6">
            <.button type="submit">บันทึก</.button>
            <%= if @page && @page.status != "published" do %>
              <.button type="button" phx-click="publish"
                class="bg-green-600 hover:bg-green-700">
                เผยแพร่
              </.button>
            <% end %>
          </div>
        </.form>
      </div>

      <!-- Rich Text Editor Area -->
      <div class="flex-1 flex flex-col">
        <div class="border-b p-3 flex justify-between items-center">
          <span class={"px-2 py-1 rounded text-sm #{status_badge(@page?.status)}"}>
            <%= @page?.status || "new" %>
          </span>
        </div>

        <div class="flex-1 p-6 overflow-y-auto">
          <div
            id="editor"
            phx-hook="TiptapEditor"
            data-content={Jason.encode!(@page?.content_json || %{})}
            class="prose max-w-none min-h-96 outline-none"
          ></div>
        </div>
      </div>
    </div>
    """
  end

  defp status_badge("published"), do: "bg-green-100 text-green-800"
  defp status_badge("draft"), do: "bg-yellow-100 text-yellow-800"
  defp status_badge(_), do: "bg-gray-100 text-gray-800"
end
```

---

## 4. TipTap Editor Hook

```javascript
// assets/js/hooks/tiptap_editor.js
import { Editor } from '@tiptap/core'
import StarterKit from '@tiptap/starter-kit'
import Image from '@tiptap/extension-image'
import Link from '@tiptap/extension-link'

const TiptapEditor = {
  mounted() {
    const content = JSON.parse(this.el.dataset.content || '{}')

    this.editor = new Editor({
      element: this.el,
      extensions: [StarterKit, Image, Link.configure({ openOnClick: false })],
      content,
      onUpdate: ({ editor }) => {
        this.pushEvent('content_updated', {
          content: editor.getHTML(),
          json: editor.getJSON()
        })
      }
    })
  },

  destroyed() {
    this.editor.destroy()
  }
}

export default TiptapEditor
```

---

## สรุป

```
CMS Features:
├── Page: title, slug, content, status
├── Rich text: TipTap + Phoenix hooks
├── Versioning: auto-save versions
└── Publishing workflow: draft → published

Content Storage:
├── content: HTML string (for rendering)
├── content_json: Tiptap JSON (for editing)
└── Versions: track all changes

Workflow:
├── Create draft
├── Edit with rich text
├── Preview before publish
└── Publish (set status + published_at)
```

---

*ก่อนหน้า: [Part 68](part_68.md) | ต่อไป: [Part 70 - Microservices](part_70.md)*
