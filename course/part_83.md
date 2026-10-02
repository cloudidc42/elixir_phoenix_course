# Part 83: File Storage and CDN (ระบบจัดเก็บไฟล์และ CDN)

## เป้าหมายการเรียนรู้

หลังจากจบ Part นี้ คุณจะสามารถ:
- อัปโหลดไฟล์ไปยัง S3/R2/GCS
- CDN integration ด้วย CloudFront
- Image processing (resize, crop)
- Secure file access (pre-signed URLs)

---

## 1. ExAws S3 Integration

```elixir
# mix.exs
{:ex_aws, "~> 2.5"},
{:ex_aws_s3, "~> 2.5"},
{:hackney, "~> 1.18"},
{:sweet_xml, "~> 0.7"}

# config/runtime.exs
config :ex_aws,
  access_key_id: System.fetch_env!("AWS_ACCESS_KEY_ID"),
  secret_access_key: System.fetch_env!("AWS_SECRET_ACCESS_KEY"),
  region: System.get_env("AWS_REGION", "ap-southeast-1")

config :my_app,
  s3_bucket: System.fetch_env!("S3_BUCKET"),
  cdn_url: System.get_env("CDN_URL")
```

---

## 2. Upload Service

```elixir
defmodule MyApp.Storage do
  alias ExAws.S3

  @bucket Application.compile_env(:my_app, :s3_bucket)
  @cdn_url Application.compile_env(:my_app, :cdn_url)

  def upload(file_path, destination_key, opts \\ []) do
    content_type = Keyword.get(opts, :content_type) || detect_content_type(file_path)
    acl = Keyword.get(opts, :acl, :private)

    file_path
    |> S3.Upload.stream_file()
    |> S3.upload(@bucket, destination_key,
      content_type: content_type,
      acl: acl
    )
    |> ExAws.request()
    |> case do
      {:ok, _} -> {:ok, public_url(destination_key)}
      {:error, reason} -> {:error, reason}
    end
  end

  def upload_binary(binary, destination_key, opts \\ []) do
    content_type = Keyword.get(opts, :content_type, "application/octet-stream")
    acl = Keyword.get(opts, :acl, :private)

    S3.put_object(@bucket, destination_key, binary,
      content_type: content_type,
      acl: acl
    )
    |> ExAws.request()
    |> case do
      {:ok, _} -> {:ok, public_url(destination_key)}
      {:error, reason} -> {:error, reason}
    end
  end

  def delete(key) do
    S3.delete_object(@bucket, key)
    |> ExAws.request()
  end

  def presigned_url(key, opts \\ []) do
    expires_in = Keyword.get(opts, :expires_in, 3600)  # 1 hour
    {:ok, url} = S3.presigned_url(
      ExAws.Config.new(:s3),
      :get,
      @bucket,
      key,
      expires_in: expires_in
    )
    url
  end

  def public_url(key) do
    if @cdn_url do
      "#{@cdn_url}/#{key}"
    else
      "https://#{@bucket}.s3.amazonaws.com/#{key}"
    end
  end

  defp detect_content_type(file_path) do
    MIME.from_path(file_path)
  end
end
```

---

## 3. LiveView File Upload to S3

```elixir
defmodule MyAppWeb.ProfileLive do
  use MyAppWeb, :live_view
  alias MyApp.Storage

  def mount(_params, _session, socket) do
    {:ok,
     socket
     |> assign(:uploaded_url, socket.assigns.current_user.avatar_url)
     |> allow_upload(:avatar,
       accept: ~w(.jpg .jpeg .png .webp),
       max_entries: 1,
       max_file_size: 5 * 1024 * 1024  # 5MB
     )
    }
  end

  def handle_event("save_avatar", _params, socket) do
    uploads = consume_uploaded_entries(socket, :avatar, fn %{path: path}, entry ->
      dest_key = "avatars/#{socket.assigns.current_user.id}/#{entry.uuid}#{Path.extname(entry.client_name)}"

      case Storage.upload(path, dest_key, content_type: entry.client_type, acl: :public_read) do
        {:ok, url} ->
          # Update user avatar
          MyApp.Accounts.update_user(socket.assigns.current_user, %{avatar_url: url})
          {:ok, url}

        {:error, reason} ->
          {:postpone, reason}
      end
    end)

    case uploads do
      [{:ok, url}] ->
        {:noreply,
         socket
         |> assign(:uploaded_url, url)
         |> put_flash(:info, "Avatar updated!")}

      _ ->
        {:noreply, put_flash(socket, :error, "Upload failed")}
    end
  end

  def render(assigns) do
    ~H"""
    <div class="max-w-md mx-auto p-6">
      <h2 class="text-xl font-bold mb-4">Profile Photo</h2>

      <%= if @uploaded_url do %>
        <img src={@uploaded_url} class="w-32 h-32 rounded-full mb-4 object-cover" />
      <% end %>

      <form phx-submit="save_avatar" phx-change="validate">
        <div class="border-2 border-dashed rounded-lg p-6 text-center mb-4"
          phx-drop-target={@uploads.avatar.ref}>
          <.live_file_input upload={@uploads.avatar} />
          <p class="text-gray-500 text-sm mt-2">JPG, PNG, WebP up to 5MB</p>
        </div>

        <%= for entry <- @uploads.avatar.entries do %>
          <div class="flex items-center gap-3 mb-3">
            <.live_img_preview entry={entry} class="w-16 h-16 rounded object-cover" />
            <div class="flex-1">
              <p class="text-sm"><%= entry.client_name %></p>
              <progress value={entry.progress} max="100" class="w-full"></progress>
            </div>
          </div>
        <% end %>

        <.button type="submit" disabled={@uploads.avatar.entries == []}>
          Upload
        </.button>
      </form>
    </div>
    """
  end
end
```

---

## 4. Image Processing with Image

```elixir
# mix.exs: {:image, "~> 0.54"}

defmodule MyApp.ImageProcessor do
  alias MyApp.Storage

  def process_and_upload(source_path, user_id, filename) do
    base_key = "images/#{user_id}/#{Path.basename(filename, Path.extname(filename))}"

    with {:ok, image} <- Image.open(source_path),
         {:ok, _} <- upload_variant(image, "#{base_key}_original.jpg", nil),
         {:ok, _} <- upload_variant(image, "#{base_key}_large.jpg", {1200, 1200}),
         {:ok, _} <- upload_variant(image, "#{base_key}_medium.jpg", {600, 600}),
         {:ok, _} <- upload_variant(image, "#{base_key}_thumb.jpg", {200, 200}) do
      {:ok, %{
        original: Storage.public_url("#{base_key}_original.jpg"),
        large: Storage.public_url("#{base_key}_large.jpg"),
        medium: Storage.public_url("#{base_key}_medium.jpg"),
        thumb: Storage.public_url("#{base_key}_thumb.jpg")
      }}
    end
  end

  defp upload_variant(image, key, nil) do
    with {:ok, binary} <- Image.stream!(image, suffix: ".jpg") |> Enum.join() |> then(&{:ok, &1}) do
      Storage.upload_binary(binary, key, content_type: "image/jpeg", acl: :public_read)
    end
  end

  defp upload_variant(image, key, {max_w, max_h}) do
    with {:ok, resized} <- Image.thumbnail(image, max_w, height: max_h),
         {:ok, binary} <- Image.stream!(resized, suffix: ".jpg") |> Enum.join() |> then(&{:ok, &1}) do
      Storage.upload_binary(binary, key, content_type: "image/jpeg", acl: :public_read)
    end
  end
end
```

---

## 5. Cloudflare R2 (S3-compatible)

```elixir
# config for Cloudflare R2 (works with ex_aws_s3)
config :ex_aws,
  access_key_id: System.fetch_env!("R2_ACCESS_KEY_ID"),
  secret_access_key: System.fetch_env!("R2_SECRET_ACCESS_KEY")

config :ex_aws, :s3,
  scheme: "https://",
  host: "#{System.fetch_env!("R2_ACCOUNT_ID")}.r2.cloudflarestorage.com",
  region: "auto"
```

---

## สรุป

```
Storage Options:
├── AWS S3: industry standard
├── Cloudflare R2: S3-compatible, cheaper egress
├── Google Cloud Storage: ex_aws_s3 compatible
└── MinIO: self-hosted S3

File Operations:
├── upload: stream file to S3
├── delete: remove object
├── presigned_url: temporary access
└── public_url: CDN or S3 URL

Image Processing:
├── Image.thumbnail: resize keeping aspect ratio
├── Image.crop: exact dimensions
└── Multiple variants: original, large, medium, thumb

CDN:
├── CloudFront + S3: AWS stack
├── Cloudflare CDN + R2: performance + cost
└── Cache-Control headers for browser caching
```

---

*ก่อนหน้า: [Part 82](part_82.md) | ต่อไป: [Part 84 - Email System Advanced](part_84.md)*
