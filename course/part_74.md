# Part 74: Scheduler and Cron Jobs (งานตามกำหนดเวลา)

## เป้าหมายการเรียนรู้

หลังจากจบ Part นี้ คุณจะสามารถ:
- ตั้ง cron jobs ด้วย Oban
- สร้าง custom scheduler GenServer
- Timezone-aware scheduling
- Job monitoring และ failure handling

---

## 1. Oban Cron Jobs

```elixir
# mix.exs
{:oban, "~> 2.18"}
{:oban_web, "~> 2.10"}  # optional UI

# config/config.exs
config :my_app, Oban,
  repo: MyApp.Repo,
  plugins: [
    {Oban.Plugins.Pruner, max_age: 60 * 60 * 24 * 7},  # keep 7 days
    {Oban.Plugins.Cron,
     crontab: [
       {"0 0 * * *",   MyApp.Workers.DailyReport},      # ทุกวันเที่ยงคืน
       {"0 9 * * 1-5", MyApp.Workers.WeekdayDigest},    # จ-ศ 9am
       {"*/30 * * * *", MyApp.Workers.SyncInventory},   # ทุก 30 นาที
       {"0 1 * * 0",   MyApp.Workers.WeeklyCleanup},    # วันอาทิตย์ 1am
     ]
    }
  ],
  queues: [
    default: 10,
    emails: 5,
    imports: 2,
    reports: 3
  ]

# application.ex
children = [
  {Oban, Application.fetch_env!(:my_app, Oban)},
]
```

---

## 2. Cron Job Workers

```elixir
defmodule MyApp.Workers.DailyReport do
  use Oban.Worker, queue: :reports, max_attempts: 2

  @impl Oban.Worker
  def perform(%Oban.Job{}) do
    report_date = Date.utc_today() |> Date.add(-1)  # yesterday

    stats = MyApp.Reports.generate_daily_stats(report_date)
    admins = MyApp.Accounts.list_admins()

    Enum.each(admins, fn admin ->
      MyApp.Emails.DailyReport.deliver(admin, stats, report_date)
      |> MyApp.Mailer.deliver()
    end)

    :ok
  end
end

defmodule MyApp.Workers.WeeklyCleanup do
  use Oban.Worker, queue: :default, max_attempts: 3

  @impl Oban.Worker
  def perform(%Oban.Job{}) do
    # Clean up old data
    MyApp.Cleanup.run([
      :expired_sessions,
      :soft_deleted_older_than_30_days,
      :temporary_files,
      :old_audit_logs
    ])

    :ok
  end
end

defmodule MyApp.Workers.SyncInventory do
  use Oban.Worker, queue: :default

  @impl Oban.Worker
  def perform(%Oban.Job{}) do
    # Sync with external inventory system
    case MyApp.Inventory.sync_from_warehouse() do
      {:ok, synced_count} ->
        MyApp.Telemetry.emit(:inventory_synced, %{count: synced_count})
        :ok

      {:error, reason} ->
        {:error, "Sync failed: #{inspect(reason)}"}
    end
  end
end
```

---

## 3. Custom Scheduler GenServer

```elixir
defmodule MyApp.Scheduler do
  use GenServer
  require Logger

  defstruct jobs: []

  # Public API
  def start_link(jobs \\ []) do
    GenServer.start_link(__MODULE__, jobs, name: __MODULE__)
  end

  def add_job(name, interval_ms, fun) do
    GenServer.call(__MODULE__, {:add_job, name, interval_ms, fun})
  end

  def remove_job(name) do
    GenServer.call(__MODULE__, {:remove_job, name})
  end

  def list_jobs do
    GenServer.call(__MODULE__, :list_jobs)
  end

  # Callbacks
  def init(job_configs) do
    jobs = Enum.map(job_configs, fn {name, interval_ms, fun} ->
      timer_ref = schedule(interval_ms)
      {name, %{interval: interval_ms, fun: fun, timer_ref: timer_ref, last_run: nil}}
    end)

    {:ok, %{jobs: Map.new(jobs)}}
  end

  def handle_call({:add_job, name, interval_ms, fun}, _from, state) do
    timer_ref = schedule(interval_ms)
    job = %{interval: interval_ms, fun: fun, timer_ref: timer_ref, last_run: nil}
    {:reply, :ok, put_in(state.jobs[name], job)}
  end

  def handle_call({:remove_job, name}, _from, state) do
    case Map.pop(state.jobs, name) do
      {nil, _} ->
        {:reply, {:error, :not_found}, state}
      {job, jobs} ->
        Process.cancel_timer(job.timer_ref)
        {:reply, :ok, %{state | jobs: jobs}}
    end
  end

  def handle_call(:list_jobs, _from, state) do
    jobs = Enum.map(state.jobs, fn {name, job} ->
      %{name: name, interval: job.interval, last_run: job.last_run}
    end)
    {:reply, jobs, state}
  end

  def handle_info({:run_job, name}, state) do
    case Map.get(state.jobs, name) do
      nil ->
        {:noreply, state}

      job ->
        Task.start(fn ->
          try do
            job.fun.()
          rescue
            e ->
              Logger.error("Job #{name} failed: #{Exception.message(e)}")
          end
        end)

        timer_ref = schedule(job.interval, name)
        updated_job = %{job | timer_ref: timer_ref, last_run: DateTime.utc_now()}
        {:noreply, put_in(state.jobs[name], updated_job)}
    end
  end

  defp schedule(interval_ms, name \\ nil) do
    Process.send_after(self(), {:run_job, name}, interval_ms)
  end
end

# application.ex
children = [
  {MyApp.Scheduler, [
    {:clear_expired_tokens, :timer.hours(1), &MyApp.Auth.clear_expired_tokens/0},
    {:update_trending, :timer.minutes(5), &MyApp.Blog.update_trending_posts/0},
  ]}
]
```

---

## 4. Timezone-aware Scheduling

```elixir
defmodule MyApp.TimeZoneScheduler do
  # Send daily email at 9am in each user's timezone

  def schedule_user_emails do
    now = DateTime.utc_now()

    MyApp.Accounts.users_with_timezones()
    |> Enum.each(fn {user, timezone} ->
      local_time = DateTime.shift_zone!(now, timezone)

      # ถ้าเวลาท้องถิ่นเป็น 9am ของวันนี้ ให้ส่ง email
      if local_time.hour == 9 and local_time.minute < 5 do
        schedule_email(user)
      end
    end)
  end

  defp schedule_email(user) do
    %{user_id: user.id}
    |> MyApp.Workers.MorningDigest.new()
    |> Oban.insert!()
  end
end
```

---

## 5. Job Monitoring

```elixir
defmodule MyApp.JobMonitor do
  import Ecto.Query
  alias MyApp.Repo
  alias Oban.Job

  def check_stuck_jobs do
    # Jobs stuck in executing state for > 10 minutes
    cutoff = DateTime.add(DateTime.utc_now(), -600)  # 10 minutes ago

    stuck_jobs = Repo.all(
      from j in Job,
      where: j.state == "executing"
        and j.attempted_at < ^cutoff
    )

    if length(stuck_jobs) > 0 do
      Enum.each(stuck_jobs, fn job ->
        MyApp.Alerts.notify_admin("Stuck job detected",
          %{job_id: job.id, worker: job.worker, queue: job.queue}
        )
      end)
    end

    stuck_jobs
  end

  def get_queue_stats do
    Repo.all(
      from j in Job,
      where: j.state in ["available", "executing", "retryable"],
      group_by: [j.queue, j.state],
      select: {j.queue, j.state, count(j.id)}
    )
    |> Enum.group_by(fn {queue, _, _} -> queue end)
    |> Enum.map(fn {queue, entries} ->
      stats = Map.new(entries, fn {_, state, count} -> {state, count} end)
      {queue, stats}
    end)
  end

  def failed_jobs_today do
    today = DateTime.new!(Date.utc_today(), ~T[00:00:00], "Etc/UTC")

    Repo.all(
      from j in Job,
      where: j.state == "discarded"
        and j.discarded_at >= ^today,
      order_by: [desc: j.discarded_at],
      limit: 50
    )
  end
end
```

---

## 6. Retry Strategy

```elixir
defmodule MyApp.Workers.ResilientWorker do
  use Oban.Worker,
    queue: :default,
    max_attempts: 5

  @impl Oban.Worker
  def perform(%Oban.Job{args: args, attempt: attempt}) do
    case do_work(args) do
      :ok ->
        :ok

      {:error, :temporary, reason} ->
        # Retry with exponential backoff
        {:snooze, backoff(attempt)}

      {:error, :permanent, reason} ->
        # Don't retry
        {:discard, reason}
    end
  end

  # Exponential backoff: 5s, 25s, 125s, 625s, 3125s
  defp backoff(attempt), do: :math.pow(5, attempt) |> round()

  @impl Oban.Worker
  def timeout(%Oban.Job{}), do: :timer.minutes(5)
end
```

---

## สรุป

```
Scheduled Jobs:
├── Oban Cron: crontab expressions
├── GenServer Scheduler: interval-based
└── Timezone-aware: per-user time zones

Oban Crontab Examples:
├── "0 0 * * *"    - ทุกวันเที่ยงคืน
├── "0 9 * * 1-5"  - จ-ศ 9am
├── "*/30 * * * *" - ทุก 30 นาที
└── "0 0 1 * *"    - วันที่ 1 ของทุกเดือน

Error Handling:
├── max_attempts: จำกัดการ retry
├── {:snooze, seconds}: ชะลอ retry
├── {:discard, reason}: หยุด retry
└── Monitoring: ตรวจ stuck jobs
```

---

*ก่อนหน้า: [Part 73](part_73.md) | ต่อไป: [Part 75 - Real-time Analytics](part_75.md)*
