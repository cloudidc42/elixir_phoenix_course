# Part 35: File Uploads

## เป้าหมายการเรียนรู้

หลังจากจบ Part นี้ คุณจะสามารถ:
- Upload files ด้วย Phoenix
- ใช้ LiveView uploads
- เก็บไฟล์ใน local หรือ cloud storage
- Validate และ process files

---

## 1. Controller-based Upload

```elixir
# router.ex
post "/users/:id/avatar", UserController, :upload_avatar

# user_controller.ex
def upload_avatar(conn, %{"id" => id, "avatar" => upload}) do
  with {:ok, user} <- Accounts.get_user(id),
       :ok <- validate_upload(upload),
       {:ok, path} <- save_upload(upload),
       {:ok, user} <- Accounts.update_avatar(user, path) do
    conn
    |> put_flash(:info, "Avatar updated!")
    |> redirect(to: ~p"/users/#{user.id}")
  end
end

defp validate_upload(%Plug.Upload{content_type: type, filename: name}) do
  allowed_types = ~w(image/jpeg image/png image/gif image/webp)
  max_size = 5 * 1024 * 1024  # 5MB

  cond do
    type not in allowed_types ->
      {:error, "Only JPEG, PNG, GIF, and WebP images are allowed"}

    byte_size(File.read!(name)) > max_size ->
      {:error, "File size must be under 5MB"}

    true ->
      :ok
  end
end

defp save_upload(%Plug.Upload{filename: filename, path: tmp_path}) do
  ext = Path.extname(filename)
  new_name = "#{Ecto.UUID.generate()}#{ext}"
  dest = Path.join(["priv", "static", "uploads", new_name])

  File.cp!(tmp_path, dest)
  {:ok, "/uploads/#{new_name}"}
end
```

---

## 2. LiveView File Uploads

```elixir
defmodule MyAppWeb.UserSettingsLive do
  use MyAppWeb, :live_view

  @impl true
  def mount(_params, _session, socket) do
    socket =
      socket
      |> allow_upload(:avatar,
        accept: ~w(.jpg .jpeg .png .gif .webp),
        max_entries: 1,
        max_file_size: 5_000_000,  # 5MB
        auto_upload: false
      )

    {:ok, socket}
  end

  @impl true
  def render(assigns) do
    ~H"""
    <div>
      <h1>Profile Settings</h1>

      <form phx-submit="save" phx-change="validate">
        <div class="upload-zone" phx-drop-target={@uploads.avatar.ref}>
          <.live_file_input upload={@uploads.avatar} />

          <%= for entry <- @uploads.avatar.entries do %>
            <div class="preview">
              <.live_img_preview entry={entry} width="100" />
              <span><%= entry.client_name %></span>
              <span><%= round(entry.progress) %>%</span>
              <button type="button" phx-click="cancel-upload"
                phx-value-ref={entry.ref}>✕</button>
            </div>

            <%= for err <- upload_errors(@uploads.avatar, entry) do %>
              <p class="error"><%= error_to_string(err) %></p>
            <% end %>
          <% end %>

          <%= for err <- upload_errors(@uploads.avatar) do %>
            <p class="error"><%= error_to_string(err) %></p>
          <% end %>
        </div>

        <button type="submit">Save</button>
      </form>
    </div>
    """
  end

  @impl true
  def handle_event("validate", _params, socket) do
    {:noreply, socket}
  end

  @impl true
  def handle_event("cancel-upload", %{"ref" => ref}, socket) do
    {:noreply, cancel_upload(socket, :avatar, ref)}
  end

  @impl true
  def handle_event("save", _params, socket) do
    uploaded_files =
      consume_uploaded_entries(socket, :avatar, fn %{path: path}, entry ->
        ext = Path.extname(entry.client_name)
        new_name = "#{Ecto.UUID.generate()}#{ext}"
        dest = Path.join([:code.priv_dir(:my_app), "static", "uploads", new_name])

        File.cp!(path, dest)
        {:ok, ~p"/uploads/#{new_name}"}
      end)

    case uploaded_files do
      [] ->
        {:noreply, put_flash(socket, :error, "Please select a file")}

      [avatar_url | _] ->
        case Accounts.update_avatar(socket.assigns.current_user, avatar_url) do
          {:ok, _user} ->
            {:noreply,
             socket
             |> put_flash(:info, "Avatar updated!")
             |> push_navigate(to: ~p"/users/settings")}

          {:error, _changeset} ->
            {:noreply, put_flash(socket, :error, "Failed to save")}
        end
    end
  end

  defp error_to_string(:too_large), do: "Too large (max 5MB)"
  defp error_to_string(:not_accepted), do: "Not an accepted file type"
  defp error_to_string(:too_many_files), do: "Too many files"
end
```

---

## 3. Multiple File Uploads

```elixir
defmodule MyAppWeb.GalleryLive do
  use MyAppWeb, :live_view

  def mount(_params, _session, socket) do
    {:ok,
     allow_upload(socket, :photos,
       accept: ~w(.jpg .jpeg .png .gif .webp),
       max_entries: 10,
       max_file_size: 10_000_000,
       chunk_size: 64_000
     )}
  end

  def handle_event("save", _params, socket) do
    photo_urls =
      consume_uploaded_entries(socket, :photos, fn %{path: path}, entry ->
        save_and_process(path, entry)
      end)

    # สร้าง gallery ใน database
    Enum.each(photo_urls, fn url ->
      MyApp.Gallery.create_photo(%{url: url, user_id: socket.assigns.current_user.id})
    end)

    {:noreply, socket}
  end

  defp save_and_process(tmp_path, entry) do
    ext = Path.extname(entry.client_name)
    filename = "#{Ecto.UUID.generate()}#{ext}"
    dest = Path.join([:code.priv_dir(:my_app), "static", "uploads", filename])

    File.cp!(tmp_path, dest)

    # Process image (resize, thumbnail, etc)
    # Mogrify.open(dest) |> Mogrify.resize("800x600") |> Mogrify.save()

    {:ok, ~p"/uploads/#{filename}"}
  end
end
```

---

## 4. Cloud Storage (S3)

```elixir
# mix.exs
{:ex_aws, "~> 2.0"},
{:ex_aws_s3, "~> 2.0"},
{:hackney, "~> 1.9"},
{:sweet_xml, "~> 0.6"}

# config/config.exs
config :ex_aws,
  access_key_id: System.get_env("AWS_ACCESS_KEY_ID"),
  secret_access_key: System.get_env("AWS_SECRET_ACCESS_KEY"),
  region: System.get_env("AWS_REGION", "ap-southeast-1")

# lib/my_app/storage.ex
defmodule MyApp.Storage do
  @bucket System.get_env("S3_BUCKET", "my-app-uploads")

  def upload(local_path, remote_path, opts \\ []) do
    content_type = Keyword.get(opts, :content_type, "application/octet-stream")
    acl = Keyword.get(opts, :acl, :public_read)

    local_path
    |> ExAws.S3.Upload.stream_file()
    |> ExAws.S3.upload(@bucket, remote_path,
      content_type: content_type,
      acl: acl
    )
    |> ExAws.request!()

    {:ok, s3_url(remote_path)}
  end

  def delete(remote_path) do
    @bucket
    |> ExAws.S3.delete_object(remote_path)
    |> ExAws.request()
  end

  def url(remote_path) do
    s3_url(remote_path)
  end

  def presigned_url(remote_path, opts \\ []) do
    expires_in = Keyword.get(opts, :expires_in, 3600)

    ExAws.S3.presigned_url(
      ExAws.Config.new(:s3),
      :get,
      @bucket,
      remote_path,
      expires_in: expires_in
    )
  end

  defp s3_url(path) do
    "https://#{@bucket}.s3.amazonaws.com/#{path}"
  end
end

# ใช้ใน LiveView
consume_uploaded_entries(socket, :avatar, fn %{path: tmp_path}, entry ->
  ext = Path.extname(entry.client_name)
  remote_path = "avatars/#{Ecto.UUID.generate()}#{ext}"

  MyApp.Storage.upload(tmp_path, remote_path,
    content_type: entry.client_type
  )
end)
```

---

## 5. Image Processing

```elixir
# mix.exs
{:image, "~> 0.37"}  # ใช้ libvips

defmodule MyApp.ImageProcessor do
  alias Vix.Vips.Image
  alias Vix.Vips.Operation

  def process_avatar(input_path, output_path) do
    with {:ok, image} <- Image.new_from_file(input_path),
         {:ok, resized} <- Operation.thumbnail_image(image, 300,
           height: 300, crop: :VIPS_INTERESTING_ATTENTION),
         :ok <- Image.write_to_file(resized, output_path) do
      {:ok, output_path}
    end
  end

  def create_thumbnail(input_path, output_path, size \\ 200) do
    with {:ok, image} <- Image.new_from_file(input_path),
         {:ok, thumb} <- Operation.thumbnail_image(image, size,
           height: size, crop: :VIPS_INTERESTING_CENTRE),
         :ok <- Image.write_to_file(thumb, output_path) do
      {:ok, output_path}
    end
  end

  def get_dimensions(path) do
    with {:ok, image} <- Image.new_from_file(path) do
      {:ok, %{
        width: Image.width(image),
        height: Image.height(image)
      }}
    end
  end
end
```

---

## 6. File Validation

```elixir
defmodule MyApp.FileValidator do
  @image_types ~w(image/jpeg image/png image/gif image/webp)
  @doc_types ~w(application/pdf application/msword
    application/vnd.openxmlformats-officedocument.wordprocessingml.document)

  def validate_image(path, content_type) do
    with :ok <- validate_content_type(content_type, @image_types),
         :ok <- validate_magic_bytes(path, :image),
         {:ok, dims} <- MyApp.ImageProcessor.get_dimensions(path),
         :ok <- validate_dimensions(dims) do
      :ok
    end
  end

  defp validate_content_type(type, allowed) do
    if type in allowed do
      :ok
    else
      {:error, "File type #{type} not allowed"}
    end
  end

  defp validate_magic_bytes(path, :image) do
    # ตรวจสอบ magic bytes ของไฟล์
    case File.open(path, [:read, :binary]) do
      {:ok, file} ->
        bytes = IO.binread(file, 12)
        File.close(file)
        check_image_magic_bytes(bytes)

      {:error, reason} ->
        {:error, "Cannot read file: #{reason}"}
    end
  end

  defp check_image_magic_bytes(<<0xFF, 0xD8, _::binary>>), do: :ok  # JPEG
  defp check_image_magic_bytes(<<0x89, "PNG", _::binary>>), do: :ok  # PNG
  defp check_image_magic_bytes(<<"GIF87a", _::binary>>), do: :ok     # GIF
  defp check_image_magic_bytes(<<"GIF89a", _::binary>>), do: :ok     # GIF
  defp check_image_magic_bytes(<<"RIFF", _, _, _, _, "WEBP", _::binary>>), do: :ok  # WebP
  defp check_image_magic_bytes(_), do: {:error, "Not a valid image file"}

  defp validate_dimensions(%{width: w, height: h}) do
    max_dimension = 4096
    if w <= max_dimension and h <= max_dimension do
      :ok
    else
      {:error, "Image dimensions too large (max #{max_dimension}px)"}
    end
  end
end
```

---

## สรุป

```
File Uploads:
├── Controller: Plug.Upload
├── LiveView: allow_upload/3
├── consume_uploaded_entries/3
└── Validate: type, size, content

Storage:
├── Local: priv/static/uploads
├── S3/Cloud: ex_aws_s3
└── CDN integration

Image Processing:
├── Image library (libvips)
├── Resize, crop, thumbnail
└── Format conversion

Security:
├── Validate content type
├── Check magic bytes
├── Scan for malware (optional)
└── Limit file sizes
```

---

*ก่อนหน้า: [Part 34](part_34.md) | ต่อไป: [Part 36 - Background Jobs](part_36.md)*
