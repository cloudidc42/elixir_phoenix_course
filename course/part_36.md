# Part 36: Background Jobs ด้วย Oban

## เป้าหมายการเรียนรู้

หลังจากจบ Part นี้ คุณจะสามารถ:
- ใช้ Oban สำหรับ background jobs
- สร้าง workers ต่างๆ
- จัดการ queues และ priorities
- Monitor และ retry jobs

---

## 1. ติดตั้ง Oban

```elixir
# mix.exs
{:oban, "~> 2.17"}

# config/config.exs
config :my_app, Oban,
  repo: MyApp.Repo,
  queues: [
    default: 10,
    emails: 5,
    uploads: 3,
    critical: 20
  ],
  plugins: [
    Oban.Plugins.Pruner,
    {Oban.Plugins.Cron,
     crontab: [
       {"0 * * * *", MyApp.Workers.HourlyCleanup},
       {"0 0 * * *", MyApp.Workers.DailyReport}
     ]}
  ]

# config/test.exs
config :my_app, Oban, testing: :inline  # รัน jobs ทันที

# lib/my_app/application.ex
def start(_type, _args) do
  children = [
    MyApp.Repo,
    {Oban, Application.fetch_env!(:my_app, Oban)},
    # ...
  ]
end
```

---

## 2. สร้าง Migration

```bash
mix ecto.gen.migration add_oban_jobs_table
```

```elixir
defmodule MyApp.Repo.Migrations.AddObanJobsTable do
  use Ecto.Migration

  def up, do: Oban.Migrations.up()
  def down, do: Oban.Migrations.down()
end
```

---

## 3. Worker พื้นฐาน

```elixir
defmodule MyApp.Workers.WelcomeEmail do
  use Oban.Worker,
    queue: :emails,
    max_attempts: 3,
    tags: ["email", "onboarding"]

  @impl Oban.Worker
  def perform(%Oban.Job{args: %{"user_id" => user_id}}) do
    user = MyApp.Accounts.get_user!(user_id)

    case MyApp.Mailer.send_welcome(user) do
      {:ok, _} -> :ok
      {:error, reason} -> {:error, reason}
    end
  end
end

# สร้าง Job
%{user_id: user.id}
|> MyApp.Workers.WelcomeEmail.new()
|> Oban.insert()

# หรือ
MyApp.Workers.WelcomeEmail.new(%{user_id: user.id})
|> Oban.insert!()
```

---

## 4. Worker Options

```elixir
defmodule MyApp.Workers.SendReport do
  use Oban.Worker,
    queue: :default,
    max_attempts: 5,
    priority: 0,          # 0 = highest, 3 = lowest
    unique: [
      period: :infinity,
      fields: [:args, :queue, :worker]  # avoid duplicates
    ]

  @impl Oban.Worker
  def perform(%Oban.Job{args: args, attempt: attempt}) do
    IO.puts("Attempt #{attempt}")

    # Backoff ตาม attempt
    backoff = :timer.seconds(:math.pow(2, attempt) |> round())

    case generate_and_send_report(args) do
      :ok ->
        :ok

      {:error, :temporary} ->
        # Schedule retry with custom backoff
        {:snooze, backoff}

      {:error, :permanent} ->
        # ไม่ retry
        {:cancel, "Permanent failure"}
    end
  end

  @impl Oban.Worker
  def timeout(%Oban.Job{}), do: :timer.minutes(10)

  @impl Oban.Worker
  def backoff(%Oban.Job{attempt: attempt}) do
    # Exponential backoff
    :timer.seconds(:math.pow(2, attempt) |> round())
  end

  defp generate_and_send_report(_args) do
    :ok
  end
end
```

---

## 5. Scheduled Jobs

```elixir
defmodule MyApp.Workers.DailyReport do
  use Oban.Worker, queue: :default

  @impl Oban.Worker
  def perform(%Oban.Job{}) do
    # รันทุกวันตี 1
    MyApp.Reports.generate_daily_report()
    :ok
  end
end

# config ใน Cron plugin
{Oban.Plugins.Cron,
 crontab: [
   {"0 1 * * *", MyApp.Workers.DailyReport},                    # ทุกวันตี 1
   {"0 */4 * * *", MyApp.Workers.HourlySync},                   # ทุก 4 ชั่วโมง
   {"*/30 * * * *", MyApp.Workers.CacheWarmer},                  # ทุก 30 นาที
   {"0 9 * * 1", MyApp.Workers.WeeklyNewsletter},               # จันทร์ 9โมง
   {"0 0 1 * *", MyApp.Workers.MonthlyBilling},                  # ต้นเดือน
 ]}
```

---

## 6. Oban Pro Features (สำหรับ production)

```elixir
# Batching
defmodule MyApp.Workers.BatchEmail do
  use Oban.Pro.Worker, queue: :emails

  def process_batch(jobs) do
    user_ids = Enum.map(jobs, fn job -> job.args["user_id"] end)
    users = MyApp.Accounts.get_users(user_ids)

    results = Enum.map(users, fn user ->
      case MyApp.Mailer.send_promotional(user) do
        {:ok, _} -> {user.id, :ok}
        {:error, reason} -> {user.id, {:error, reason}}
      end
    end)

    failed = Enum.filter(results, fn {_, result} -> result != :ok end)

    if length(failed) > 0 do
      {:error, "#{length(failed)} emails failed"}
    else
      :ok
    end
  end
end
```

---

## 7. ตัวอย่างจริง: Image Processing Pipeline

```elixir
defmodule MyApp.Workers.ProcessImage do
  use Oban.Worker,
    queue: :uploads,
    max_attempts: 3

  @impl Oban.Worker
  def perform(%Oban.Job{args: %{"image_id" => image_id}}) do
    with {:ok, image} <- MyApp.Gallery.get_image(image_id),
         {:ok, original_path} <- download_original(image),
         {:ok, thumbnail_url} <- create_thumbnail(original_path),
         {:ok, medium_url} <- create_medium(original_path),
         {:ok, _} <- MyApp.Gallery.update_image(image, %{
           thumbnail_url: thumbnail_url,
           medium_url: medium_url,
           processed: true
         }) do
      :ok
    else
      {:error, reason} -> {:error, "Processing failed: #{inspect(reason)}"}
    end
  end

  defp download_original(image) do
    tmp_path = Path.join(System.tmp_dir!(), "#{Ecto.UUID.generate()}.jpg")
    # Download from S3
    {:ok, tmp_path}
  end

  defp create_thumbnail(path) do
    thumb_path = String.replace(path, ".jpg", "_thumb.jpg")
    MyApp.ImageProcessor.create_thumbnail(path, thumb_path, 200)
    MyApp.Storage.upload(thumb_path, "thumbnails/#{Path.basename(thumb_path)}")
  end

  defp create_medium(path) do
    medium_path = String.replace(path, ".jpg", "_medium.jpg")
    MyApp.ImageProcessor.resize(path, medium_path, 800, 600)
    MyApp.Storage.upload(medium_path, "medium/#{Path.basename(medium_path)}")
  end
end

# เมื่อ upload รูปภาพ
{:ok, image} = MyApp.Gallery.create_image(attrs)
%{image_id: image.id}
|> MyApp.Workers.ProcessImage.new()
|> Oban.insert!()
```

---

## 8. Testing Oban Jobs

```elixir
defmodule MyApp.Workers.WelcomeEmailTest do
  use MyApp.DataCase

  import Oban.Testing, repo: MyApp.Repo

  alias MyApp.Workers.WelcomeEmail

  test "sends welcome email" do
    user = insert(:user)

    assert :ok = perform_job(WelcomeEmail, %{user_id: user.id})
  end

  test "enqueues job after registration" do
    {:ok, user} = MyApp.Accounts.register_user(%{
      name: "Alice",
      email: "alice@example.com",
      password: "Password123!"
    })

    assert_enqueued worker: WelcomeEmail, args: %{user_id: user.id}
  end

  test "retries on temporary failure" do
    user = insert(:user, email: "bad@example.com")

    assert {:error, _reason} = perform_job(WelcomeEmail, %{user_id: user.id},
      attempt: 1)

    # ตรวจสอบว่า job จะ retry
  end
end
```

---

## 9. Monitoring Oban

```elixir
# Phoenix LiveDashboard integration
# mix.exs
{:oban_web, "~> 2.9"}

# router.ex
scope "/" do
  pipe_through :browser

  live_dashboard "/dashboard",
    metrics: MyAppWeb.Telemetry,
    additional_pages: [
      oban: {ObanWeb.DashboardPage, oban: Oban}
    ]
end

# ดู jobs ด้วย Oban API
Oban.drain_queue(queue: :emails)           # รัน pending jobs ทันที
Oban.pause_queue(queue: :emails)           # หยุด queue
Oban.resume_queue(queue: :emails)          # เริ่มใหม่
Oban.scale_queue(queue: :emails, limit: 20)  # เปลี่ยน concurrency
```

---

## สรุป

```
Oban:
├── Persistent job queue (ใช้ PostgreSQL)
├── Automatic retry ด้วย backoff
├── Scheduled jobs (cron)
└── Unique jobs (deduplication)

Workers:
├── use Oban.Worker
├── perform/1 callback
├── max_attempts, priority, queue
└── timeout, backoff

Queues:
├── แยก queue ตาม priority
├── Scale ตาม load
└── Pause/resume

Testing:
├── config testing: :inline
├── perform_job/2 สำหรับ unit test
└── assert_enqueued/1 สำหรับ integration
```

---

*ก่อนหน้า: [Part 35](part_35.md) | ต่อไป: [Part 37 - Email](part_37.md)*
