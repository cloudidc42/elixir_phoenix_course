# Part 68: Content Management System (CMS)

## เป้าหมายการเรียนรู้

- ออกแบบ dynamic content types สำหรับ pages, blog, products
- รวม WYSIWYG editor (Trix/ProseMirror) เข้ากับ Phoenix
- จัดการ media library พร้อมจัดระเบียบไฟล์
- สร้าง URL slugs และ redirects อัตโนมัติ
- ตั้งค่า SEO สำหรับแต่ละหน้า
- วาง draft/publish/schedule workflow
- จัดการ content versioning

---

## 1. ออกแบบ Content Type System

### 1.1 Schema พื้นฐาน

```elixir
defmodule MyApp.Repo.Migrations.CreateCmsSystem do
  use Ecto.Migration

  def change do
    # Content types definition
    create table(:content_types) do
      add :name, :string, null: false           # "page", "blog_post", "product"
      add :label, :string, null: false          # "หน้าเว็บ", "บทความ"
      add :schema_definition, :map, default: %{} # field definitions
      add :active, :boolean, default: true
      timestamps()
    end
    create unique_index(:content_types, [:name])

    # Content entries (the actual content)
    create table(:content_entries) do
      add :content_type_id, references(:content_types), null: false
      add :author_id, references(:users), null: false
      add :title, :string, null: false
      add :slug, :string, null: false
      add :body, :text
      add :excerpt, :text
      add :fields, :map, default: %{}           # custom fields ตาม content type
      add :status, :string, default: "draft"    # draft, published, scheduled, archived
      add :published_at, :utc_datetime
      add :scheduled_at, :utc_datetime
      add :featured_image_id, references(:media_files)
      add :parent_id, references(:content_entries)  # สำหรับ hierarchy
      add :sort_order, :integer, default: 0
      add :metadata, :map, default: %{}         # SEO metadata
      timestamps()
    end

    create unique_index(:content_entries, [:slug, :content_type_id])
    create index(:content_entries, [:content_type_id, :status])
    create index(:content_entries, [:author_id])
    create index(:content_entries, [:scheduled_at])

    # Content versions (versioning)
    create table(:content_versions) do
      add :entry_id, references(:content_entries, on_delete: :delete_all), null: false
      add :version_number, :integer, null: false
      add :title, :string
      add :body, :text
      add :fields, :map, default: %{}
      add :author_id, references(:users)
      add :change_note, :string
      timestamps(updated_at: false)
    end

    create index(:content_versions, [:entry_id, :version_number])

    # URL redirects
    create table(:url_redirects) do
      add :from_path, :string, null: false
      add :to_path, :string, null: false
      add :redirect_type, :integer, default: 301  # 301 หรือ 302
      add :active, :boolean, default: true
      timestamps()
    end

    create unique_index(:url_redirects, [:from_path])

    # Media files
    create table(:media_files) do
      add :uploader_id, references(:users)
      add :filename, :string, null: false
      add :original_filename, :string
      add :file_type, :string               # "image", "video", "document"
      add :content_type, :string
      add :file_size, :integer
      add :storage_key, :string, null: false
      add :url, :string
      add :alt_text, :string
      add :caption, :string
      add :folder, :string, default: "/"
      add :metadata, :map, default: %{}     # width, height, duration, etc.
      timestamps()
    end

    create index(:media_files, [:folder])
    create index(:media_files, [:file_type])
  end
end
```

### 1.2 Content Entry Schema

```elixir
defmodule MyApp.CMS.ContentEntry do
  use Ecto.Schema
  import Ecto.Changeset

  schema "content_entries" do
    belongs_to :content_type, MyApp.CMS.ContentType
    belongs_to :author, MyApp.Accounts.User
    belongs_to :featured_image, MyApp.CMS.MediaFile
    belongs_to :parent, MyApp.CMS.ContentEntry
    has_many :versions, MyApp.CMS.ContentVersion
    has_many :children, MyApp.CMS.ContentEntry, foreign_key: :parent_id

    field :title, :string
    field :slug, :string
    field :body, :string
    field :excerpt, :string
    field :fields, :map, default: %{}
    field :status, :string, default: "draft"
    field :published_at, :utc_datetime
    field :scheduled_at, :utc_datetime
    field :sort_order, :integer, default: 0
    field :metadata, :map, default: %{}
    timestamps()
  end

  @statuses ~w(draft published scheduled archived)

  def changeset(entry, attrs) do
    entry
    |> cast(attrs, [:title, :slug, :body, :excerpt, :fields, :status,
                    :published_at, :scheduled_at, :sort_order, :metadata,
                    :content_type_id, :author_id, :featured_image_id, :parent_id])
    |> validate_required([:title, :content_type_id, :author_id])
    |> validate_inclusion(:status, @statuses)
    |> maybe_generate_slug()
    |> unique_constraint(:slug, name: :content_entries_slug_content_type_id_index)
    |> validate_scheduled_at()
  end

  def publish_changeset(entry) do
    entry
    |> change(status: "published", published_at: DateTime.utc_now() |> DateTime.truncate(:second))
  end

  def schedule_changeset(entry, scheduled_at) do
    entry
    |> change(status: "scheduled", scheduled_at: scheduled_at)
  end

  defp maybe_generate_slug(changeset) do
    case get_change(changeset, :slug) do
      nil ->
        case get_change(changeset, :title) do
          nil -> changeset
          title -> put_change(changeset, :slug, Slugify.slugify(title))
        end
      _ -> changeset
    end
  end

  defp validate_scheduled_at(changeset) do
    case {get_field(changeset, :status), get_field(changeset, :scheduled_at)} do
      {"scheduled", nil} ->
        add_error(changeset, :scheduled_at, "จำเป็นต้องกำหนดเวลาเผยแพร่")
      {"scheduled", dt} when not is_nil(dt) ->
        if DateTime.compare(dt, DateTime.utc_now()) == :gt,
          do: changeset,
          else: add_error(changeset, :scheduled_at, "ต้องเป็นเวลาในอนาคต")
      _ -> changeset
    end
  end
end
```

---

## 2. WYSIWYG Editor Integration

### 2.1 Trix Editor (ง่ายกว่า)

```heex
<%# lib/my_app_web/components/trix_editor.html.heex %>
<div class="trix-wrapper" phx-update="ignore" id={"trix-#{@id}"}>
  <input
    type="hidden"
    id={"trix-input-#{@id}"}
    name={@field_name}
    value={@value || ""}
  />
  <trix-editor
    input={"trix-input-#{@id}"}
    class="trix-content min-h-64 border rounded-md p-3 focus:outline-none"
    placeholder={@placeholder || "เริ่มพิมพ์เนื้อหา..."}
  ></trix-editor>
</div>
```

```javascript
// assets/js/hooks/trix_editor.js
export const TrixEditor = {
  mounted() {
    const editor = this.el.querySelector('trix-editor');

    editor.addEventListener('trix-change', (event) => {
      const content = event.target.editor.getDocument().toString();
      this.pushEvent('content_changed', {
        field: this.el.dataset.field,
        value: event.target.value
      });
    });

    // Custom toolbar
    this.setupCustomToolbar();
  },

  setupCustomToolbar() {
    // เพิ่ม custom buttons เช่น insert image
    const toolbar = this.el.querySelector('trix-toolbar');
    if (!toolbar) return;

    const fileInput = document.createElement('input');
    fileInput.type = 'file';
    fileInput.accept = 'image/*';
    fileInput.style.display = 'none';
    fileInput.addEventListener('change', (e) => {
      this.handleImageUpload(e.target.files[0]);
    });
    toolbar.appendChild(fileInput);
  },

  async handleImageUpload(file) {
    const formData = new FormData();
    formData.append('file', file);

    const response = await fetch('/api/media/upload', {
      method: 'POST',
      body: formData
    });

    if (response.ok) {
      const { url } = await response.json();
      const editor = this.el.querySelector('trix-editor').editor;
      editor.insertFile(file);
    }
  }
};
```

### 2.2 ProseMirror Editor (advanced)

```javascript
// assets/js/hooks/prosemirror_editor.js
import { EditorState } from 'prosemirror-state';
import { EditorView } from 'prosemirror-view';
import { Schema, DOMParser, DOMSerializer } from 'prosemirror-model';
import { schema } from 'prosemirror-schema-basic';
import { addListNodes } from 'prosemirror-schema-list';
import { exampleSetup } from 'prosemirror-example-setup';

const mySchema = new Schema({
  nodes: addListNodes(schema.spec.nodes, 'paragraph block*', 'block'),
  marks: schema.spec.marks
});

export const ProseMirrorEditor = {
  mounted() {
    const hiddenInput = document.getElementById(`pm-input-${this.el.dataset.id}`);
    const container = this.el.querySelector('.pm-editor-container');

    // Parse existing content
    const doc = hiddenInput.value
      ? DOMParser.fromSchema(mySchema).parse(
          new DOMParser().parseFromString(hiddenInput.value, 'text/html').body
        )
      : mySchema.node('doc', null, [mySchema.node('paragraph')]);

    this.view = new EditorView(container, {
      state: EditorState.create({
        doc,
        plugins: exampleSetup({ schema: mySchema })
      }),
      dispatchTransaction: (transaction) => {
        const newState = this.view.state.apply(transaction);
        this.view.updateState(newState);

        if (transaction.docChanged) {
          const serializer = DOMSerializer.fromSchema(mySchema);
          const fragment = serializer.serializeFragment(newState.doc.content);
          const div = document.createElement('div');
          div.appendChild(fragment);
          hiddenInput.value = div.innerHTML;

          // Notify LiveView
          this.pushEvent('content_updated', { value: div.innerHTML });
        }
      }
    });
  },

  destroyed() {
    if (this.view) {
      this.view.destroy();
    }
  }
};
```

---

## 3. Media Library

### 3.1 Upload Handler

```elixir
defmodule MyApp.CMS.MediaLibrary do
  alias MyApp.Repo
  alias MyApp.CMS.MediaFile
  import Ecto.Query

  def upload(user, file, opts \\ []) do
    folder = Keyword.get(opts, :folder, "/uploads")
    alt_text = Keyword.get(opts, :alt_text, "")

    with {:ok, {storage_key, url}} <- store_file(file),
         {:ok, metadata} <- extract_metadata(file),
         {:ok, media} <- create_record(user, file, storage_key, url, folder, alt_text, metadata) do
      {:ok, media}
    end
  end

  defp store_file(file) do
    storage_key = generate_storage_key(file.filename)

    case MyApp.Storage.upload(file.path, storage_key) do
      {:ok, url} -> {:ok, {storage_key, url}}
      error -> error
    end
  end

  defp extract_metadata(file) do
    metadata = %{
      file_size: File.stat!(file.path).size,
      content_type: MIME.from_path(file.filename)
    }

    # ดึง image dimensions ถ้าเป็นรูป
    metadata =
      if String.starts_with?(metadata.content_type, "image/") do
        case Image.open(file.path) do
          {:ok, image} ->
            {width, height, _} = Image.shape(image)
            Map.merge(metadata, %{width: width, height: height})
          _ -> metadata
        end
      else
        metadata
      end

    {:ok, metadata}
  end

  defp create_record(user, file, storage_key, url, folder, alt_text, metadata) do
    attrs = %{
      uploader_id: user.id,
      filename: storage_key |> Path.basename(),
      original_filename: file.filename,
      file_type: classify_file_type(metadata.content_type),
      content_type: metadata.content_type,
      file_size: metadata.file_size,
      storage_key: storage_key,
      url: url,
      alt_text: alt_text,
      folder: folder,
      metadata: Map.drop(metadata, [:file_size, :content_type])
    }

    %MediaFile{}
    |> MediaFile.changeset(attrs)
    |> Repo.insert()
  end

  defp classify_file_type(content_type) do
    cond do
      String.starts_with?(content_type, "image/") -> "image"
      String.starts_with?(content_type, "video/") -> "video"
      String.starts_with?(content_type, "audio/") -> "audio"
      true -> "document"
    end
  end

  defp generate_storage_key(filename) do
    ext = Path.extname(filename)
    date = Date.utc_today()
    "#{date.year}/#{String.pad_leading("#{date.month}", 2, "0")}/#{Ecto.UUID.generate()}#{ext}"
  end

  # Query functions
  def list_files(folder \\ "/", opts \\ []) do
    page = Keyword.get(opts, :page, 1)
    per_page = Keyword.get(opts, :per_page, 30)
    file_type = Keyword.get(opts, :file_type)
    search = Keyword.get(opts, :search)

    from(m in MediaFile,
      where: m.folder == ^folder
    )
    |> maybe_filter_by_type(file_type)
    |> maybe_search(search)
    |> order_by([m], desc: m.inserted_at)
    |> limit(^per_page)
    |> offset(^((page - 1) * per_page))
    |> Repo.all()
  end

  defp maybe_filter_by_type(query, nil), do: query
  defp maybe_filter_by_type(query, type), do: from(m in query, where: m.file_type == ^type)

  defp maybe_search(query, nil), do: query
  defp maybe_search(query, search) do
    from(m in query, where: ilike(m.original_filename, ^"%#{search}%"))
  end

  def move_file(media_id, new_folder) do
    from(m in MediaFile, where: m.id == ^media_id)
    |> Repo.update_all(set: [folder: new_folder])
  end

  def delete_file(media_id) do
    case Repo.get(MediaFile, media_id) do
      nil -> {:error, :not_found}
      media ->
        MyApp.Storage.delete(media.storage_key)
        Repo.delete(media)
    end
  end
end
```

### 3.2 Media Library LiveView

```elixir
defmodule MyAppWeb.CMS.MediaLibraryLive do
  use MyAppWeb, :live_view

  alias MyApp.CMS.MediaLibrary

  def mount(_params, _session, socket) do
    {:ok, assign(socket,
      files: MediaLibrary.list_files("/"),
      current_folder: "/",
      selected_files: [],
      uploading: false,
      search: "",
      filter_type: nil
    ), temporary_assigns: [files: []]}
  end

  def handle_event("upload", params, socket) do
    # ใช้ Phoenix LiveView uploads
    {:noreply, socket}
  end

  def handle_event("select_file", %{"id" => id}, socket) do
    id = String.to_integer(id)
    selected =
      if id in socket.assigns.selected_files,
        do: List.delete(socket.assigns.selected_files, id),
        else: [id | socket.assigns.selected_files]

    {:noreply, assign(socket, :selected_files, selected)}
  end

  def handle_event("filter_type", %{"type" => type}, socket) do
    type = if type == "", do: nil, else: type
    files = MediaLibrary.list_files(socket.assigns.current_folder, file_type: type)
    {:noreply, assign(socket, filter_type: type, files: files)}
  end

  def handle_event("search", %{"value" => search}, socket) do
    files = MediaLibrary.list_files(socket.assigns.current_folder, search: search)
    {:noreply, assign(socket, search: search, files: files)}
  end
end
```

---

## 4. URL Slugs และ Redirects

### 4.1 Slug Generation

```elixir
defmodule MyApp.CMS.SlugHelper do
  alias MyApp.Repo
  import Ecto.Query

  def generate_unique_slug(title, content_type_id, exclude_id \\ nil) do
    base_slug = Slugify.slugify(title)
    find_unique_slug(base_slug, content_type_id, exclude_id, 0)
  end

  defp find_unique_slug(base, content_type_id, exclude_id, 0) do
    if slug_available?(base, content_type_id, exclude_id),
      do: base,
      else: find_unique_slug(base, content_type_id, exclude_id, 1)
  end

  defp find_unique_slug(base, content_type_id, exclude_id, n) do
    candidate = "#{base}-#{n}"
    if slug_available?(candidate, content_type_id, exclude_id),
      do: candidate,
      else: find_unique_slug(base, content_type_id, exclude_id, n + 1)
  end

  defp slug_available?(slug, content_type_id, exclude_id) do
    query = from(e in "content_entries",
      where: e.slug == ^slug and e.content_type_id == ^content_type_id
    )
    query = if exclude_id, do: from(e in query, where: e.id != ^exclude_id), else: query
    Repo.one(from(e in query, select: count(e.id))) == 0
  end

  # จัดการ slug เปลี่ยน - สร้าง redirect อัตโนมัติ
  def handle_slug_change(old_slug, new_slug, content_type_name) do
    old_path = build_path(content_type_name, old_slug)
    new_path = build_path(content_type_name, new_slug)

    Repo.insert_all("url_redirects", [%{
      from_path: old_path,
      to_path: new_path,
      redirect_type: 301,
      active: true,
      inserted_at: DateTime.utc_now() |> DateTime.truncate(:second),
      updated_at: DateTime.utc_now() |> DateTime.truncate(:second)
    }],
    on_conflict: {:replace, [:to_path, :updated_at]},
    conflict_target: :from_path)
  end

  defp build_path("page", slug), do: "/#{slug}"
  defp build_path("blog_post", slug), do: "/blog/#{slug}"
  defp build_path("product", slug), do: "/products/#{slug}"
  defp build_path(_, slug), do: "/#{slug}"
end
```

### 4.2 Redirect Plug

```elixir
defmodule MyAppWeb.Plugs.HandleRedirects do
  import Plug.Conn
  alias MyApp.Repo
  import Ecto.Query

  def init(opts), do: opts

  def call(conn, _opts) do
    path = conn.request_path

    case find_redirect(path) do
      nil -> conn
      redirect ->
        conn
        |> put_status(redirect.redirect_type)
        |> Phoenix.Controller.redirect(to: redirect.to_path)
        |> halt()
    end
  end

  defp find_redirect(path) do
    from(r in "url_redirects",
      where: r.from_path == ^path and r.active == true,
      select: %{to_path: r.to_path, redirect_type: r.redirect_type}
    )
    |> Repo.one()
  end
end
```

---

## 5. SEO Settings

```elixir
defmodule MyApp.CMS.SEO do
  # Default SEO metadata structure
  def default_metadata do
    %{
      meta_title: nil,
      meta_description: nil,
      meta_keywords: [],
      og_title: nil,
      og_description: nil,
      og_image: nil,
      canonical_url: nil,
      robots: "index,follow",
      schema_org: nil
    }
  end

  def build_page_meta(entry, base_url) do
    meta = Map.merge(default_metadata(), entry.metadata || %{})

    %{
      title: meta["meta_title"] || entry.title,
      description: meta["meta_description"] || entry.excerpt || truncate_body(entry.body),
      keywords: meta["meta_keywords"] || [],
      og: %{
        title: meta["og_title"] || meta["meta_title"] || entry.title,
        description: meta["og_description"] || meta["meta_description"],
        image: meta["og_image"] || get_featured_image_url(entry),
        url: meta["canonical_url"] || build_canonical_url(entry, base_url),
        type: og_type_for(entry.content_type.name)
      },
      canonical: meta["canonical_url"] || build_canonical_url(entry, base_url),
      robots: meta["robots"] || "index,follow"
    }
  end

  defp truncate_body(nil), do: ""
  defp truncate_body(body) do
    body
    |> HtmlSanitizeEx.strip_tags()
    |> String.slice(0, 160)
  end

  defp get_featured_image_url(%{featured_image: %{url: url}}), do: url
  defp get_featured_image_url(_), do: nil

  defp build_canonical_url(entry, base_url) do
    path =
      case entry.content_type.name do
        "page" -> "/#{entry.slug}"
        "blog_post" -> "/blog/#{entry.slug}"
        _ -> "/#{entry.slug}"
      end
    "#{base_url}#{path}"
  end

  defp og_type_for("blog_post"), do: "article"
  defp og_type_for("product"), do: "product"
  defp og_type_for(_), do: "website"
end
```

```heex
<%# lib/my_app_web/components/layouts/seo_tags.html.heex %>
<meta name="description" content={@seo.description} />
<meta name="keywords" content={Enum.join(@seo.keywords, ", ")} />
<link rel="canonical" href={@seo.canonical} />
<meta name="robots" content={@seo.robots} />

<!-- Open Graph -->
<meta property="og:title" content={@seo.og.title} />
<meta property="og:description" content={@seo.og.description} />
<meta property="og:type" content={@seo.og.type} />
<meta property="og:url" content={@seo.og.url} />
<%= if @seo.og.image do %>
  <meta property="og:image" content={@seo.og.image} />
<% end %>

<!-- Twitter Card -->
<meta name="twitter:card" content="summary_large_image" />
<meta name="twitter:title" content={@seo.og.title} />
<meta name="twitter:description" content={@seo.og.description} />
```

---

## 6. Draft/Publish/Schedule Workflow

```elixir
defmodule MyApp.CMS.WorkflowManager do
  alias MyApp.{Repo, CMS}
  alias MyApp.CMS.{ContentEntry, ContentVersion}
  import Ecto.Query

  # บันทึก draft
  def save_draft(entry_id, attrs, user) do
    entry = Repo.get!(ContentEntry, entry_id)

    with {:ok, updated} <- update_entry(entry, Map.put(attrs, :status, "draft")),
         {:ok, _version} <- create_version(updated, user, attrs[:change_note] || "Draft saved") do
      {:ok, updated}
    end
  end

  # เผยแพร่ทันที
  def publish(entry_id, user) do
    entry = Repo.get!(ContentEntry, entry_id)

    if can_publish?(user, entry) do
      with {:ok, updated} <- update_entry(entry, %{
             status: "published",
             published_at: DateTime.utc_now() |> DateTime.truncate(:second)
           }),
           {:ok, _version} <- create_version(updated, user, "Published") do
        broadcast_published(updated)
        {:ok, updated}
      end
    else
      {:error, :unauthorized}
    end
  end

  # กำหนดเวลาเผยแพร่
  def schedule(entry_id, scheduled_at, user) do
    entry = Repo.get!(ContentEntry, entry_id)

    if DateTime.compare(scheduled_at, DateTime.utc_now()) != :gt do
      {:error, :invalid_schedule_time}
    else
      update_entry(entry, %{status: "scheduled", scheduled_at: scheduled_at})
    end
  end

  # ยกเลิกการเผยแพร่
  def unpublish(entry_id) do
    entry = Repo.get!(ContentEntry, entry_id)
    update_entry(entry, %{status: "draft", published_at: nil})
  end

  # archive
  def archive(entry_id) do
    entry = Repo.get!(ContentEntry, entry_id)
    update_entry(entry, %{status: "archived"})
  end

  defp update_entry(entry, attrs) do
    entry
    |> ContentEntry.changeset(attrs)
    |> Repo.update()
  end

  defp can_publish?(user, _entry) do
    "cms:publish" in user.permissions
  end

  defp broadcast_published(entry) do
    Phoenix.PubSub.broadcast(
      MyApp.PubSub,
      "cms:published",
      {:content_published, entry}
    )
  end

  defp create_version(entry, user, note) do
    attrs = %{
      entry_id: entry.id,
      version_number: next_version_number(entry.id),
      title: entry.title,
      body: entry.body,
      fields: entry.fields,
      author_id: user.id,
      change_note: note,
      inserted_at: DateTime.utc_now() |> DateTime.truncate(:second)
    }

    Repo.insert_all(ContentVersion, [attrs])
    {:ok, attrs}
  end

  defp next_version_number(entry_id) do
    current = from(v in ContentVersion,
      where: v.entry_id == ^entry_id,
      select: max(v.version_number)
    )
    |> Repo.one() || 0
    current + 1
  end
end
```

### 6.1 Oban Worker สำหรับ Scheduled Publishing

```elixir
defmodule MyApp.Workers.PublishScheduledContent do
  use Oban.Worker, queue: :cms, max_attempts: 3

  @impl Oban.Worker
  def perform(_job) do
    now = DateTime.utc_now()

    import Ecto.Query
    entries_to_publish =
      from(e in MyApp.CMS.ContentEntry,
        where: e.status == "scheduled"
          and not is_nil(e.scheduled_at)
          and e.scheduled_at <= ^now
      )
      |> MyApp.Repo.all()

    Enum.each(entries_to_publish, fn entry ->
      MyApp.CMS.WorkflowManager.publish(entry.id, %{permissions: ["cms:publish"]})
    end)

    :ok
  end
end
```

```elixir
# config/config.exs - ตั้ง cron ทุกนาที
config :my_app, Oban,
  plugins: [
    {Oban.Plugins.Cron,
     crontab: [
       {"* * * * *", MyApp.Workers.PublishScheduledContent}
     ]}
  ]
```

---

## 7. Content Versioning

```elixir
defmodule MyApp.CMS.VersionManager do
  alias MyApp.{Repo, CMS}
  alias MyApp.CMS.{ContentEntry, ContentVersion}
  import Ecto.Query

  def list_versions(entry_id) do
    from(v in ContentVersion,
      where: v.entry_id == ^entry_id,
      join: u in assoc(v, :author),
      preload: [author: u],
      order_by: [desc: v.version_number]
    )
    |> Repo.all()
  end

  def get_version(entry_id, version_number) do
    from(v in ContentVersion,
      where: v.entry_id == ^entry_id and v.version_number == ^version_number
    )
    |> Repo.one()
  end

  # Restore เนื้อหาจาก version เก่า
  def restore_version(entry_id, version_number, user) do
    with version when not is_nil(version) <- get_version(entry_id, version_number),
         entry when not is_nil(entry) <- Repo.get(ContentEntry, entry_id) do
      attrs = %{
        title: version.title,
        body: version.body,
        fields: version.fields
      }

      entry
      |> ContentEntry.changeset(attrs)
      |> Repo.update()
    else
      nil -> {:error, :not_found}
    end
  end

  # เปรียบเทียบ 2 versions
  def diff_versions(entry_id, version_a, version_b) do
    v_a = get_version(entry_id, version_a)
    v_b = get_version(entry_id, version_b)

    if v_a && v_b do
      %{
        title_changed: v_a.title != v_b.title,
        title_diff: diff_text(v_a.title, v_b.title),
        body_diff: diff_text(v_a.body || "", v_b.body || ""),
        fields_changed: v_a.fields != v_b.fields
      }
    else
      {:error, :version_not_found}
    end
  end

  defp diff_text(text_a, text_b) do
    lines_a = String.split(text_a, "\n")
    lines_b = String.split(text_b, "\n")

    # Simple line-by-line diff
    added = length(lines_b) - length(Enum.filter(lines_b, &(&1 in lines_a)))
    removed = length(lines_a) - length(Enum.filter(lines_a, &(&1 in lines_b)))

    %{added_lines: added, removed_lines: removed}
  end

  # ลบ versions เก่า (เก็บแค่ N versions)
  def cleanup_old_versions(entry_id, keep_count \\ 20) do
    versions_to_delete =
      from(v in ContentVersion,
        where: v.entry_id == ^entry_id,
        order_by: [desc: v.version_number],
        offset: ^keep_count
      )
      |> Repo.all()
      |> Enum.map(& &1.id)

    from(v in ContentVersion, where: v.id in ^versions_to_delete)
    |> Repo.delete_all()
  end
end
```

---

## สรุป

```
CMS Architecture
═════════════════════════════════════════════════════════
Content Layer
  ┌─────────────────────────────────────────────────┐
  │  ContentTypes (schema definitions)               │
  │    ├── Page                                      │
  │    ├── BlogPost                                  │
  │    ├── Product                                   │
  │    └── [custom types]                            │
  │                                                  │
  │  ContentEntries (actual content)                 │
  │    ├── title, slug, body                        │
  │    ├── fields (flexible JSONB)                   │
  │    ├── status: draft→scheduled→published        │
  │    └── SEO metadata                             │
  └─────────────────────────────────────────────────┘

Supporting Systems
  ┌──────────────┬──────────────┬──────────────────┐
  │ Media Library│  Versioning  │   URL Management  │
  │ - Upload     │ - Auto-save  │ - Slug generation │
  │ - Organize   │ - Restore    │ - Redirects (301) │
  │ - S3/Local   │ - Diff view  │ - Canonical URLs  │
  └──────────────┴──────────────┴──────────────────┘

Workflow
  Author → Draft → Review → Publish
                          ↓
                       Schedule (Oban Cron)
                          ↓
                       Auto-publish at scheduled time

Editor Options
  - Trix: ง่าย, built-in Phoenix ActionText
  - ProseMirror: customizable, plugin-based
  - Quill: feature-rich, easy to extend
═════════════════════════════════════════════════════════
```

---

*ก่อนหน้า: [Part 67 - Internationalization (i18n)](part_67.md) | ต่อไป: [Part 69 - Microservices Communication](part_69.md)*
