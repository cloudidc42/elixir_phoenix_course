# Part 99: Machine Learning in Production (ML ใน Production)

## เป้าหมายการเรียนรู้

หลังจากจบ Part นี้ คุณจะสามารถ:
- Deploy ML models ด้วย Bumblebee
- Async inference pipeline
- Batch processing สำหรับ ML
- A/B testing สำหรับ models

---

## 1. Serving ML Models ด้วย Nx.Serving

```elixir
# mix.exs
{:bumblebee, "~> 0.5"},
{:nx, "~> 0.7"},
{:exla, "~> 0.7"}  # or {:torchx, "~> 0.7"}

# application.ex
defmodule MyApp.Application do
  def start(_type, _args) do
    {:ok, model_info} = Bumblebee.load_model({:hf, "distilbert-base-uncased-finetuned-sst-2-english"})
    {:ok, tokenizer} = Bumblebee.load_tokenizer({:hf, "distilbert-base-uncased"})
    {:ok, generation_config} = Bumblebee.load_generation_config({:hf, "distilbert-base-uncased-finetuned-sst-2-english"})

    # Nx.Serving handles batching and concurrency automatically
    sentiment_serving = Bumblebee.Text.text_classification(
      model_info,
      tokenizer,
      compile: [batch_size: 8, sequence_length: 64],
      defn_options: [compiler: EXLA]
    )

    children = [
      {Nx.Serving, serving: sentiment_serving, name: MyApp.SentimentAnalysis, batch_timeout: 100},
      # Other children...
    ]

    Supervisor.start_link(children, strategy: :one_for_one)
  end
end
```

---

## 2. Inference Service

```elixir
defmodule MyApp.ML.SentimentService do
  @serving MyApp.SentimentAnalysis

  def analyze(text) when is_binary(text) do
    case Nx.Serving.run(@serving, text) do
      %{predictions: [%{label: label, score: score} | _]} ->
        {:ok, %{sentiment: label, confidence: Float.round(score, 4)}}
    end
  end

  def analyze_batch(texts) when is_list(texts) do
    tasks = Enum.map(texts, fn text ->
      Task.async(fn ->
        {text, analyze(text)}
      end)
    end)

    Task.await_many(tasks, 30_000)
    |> Enum.map(fn {text, result} ->
      %{text: text, result: result}
    end)
  end
end

defmodule MyApp.ML.EmbeddingService do
  @serving MyApp.TextEmbedding

  def embed(text) do
    %{embedding: embedding} = Nx.Serving.run(@serving, text)
    {:ok, Nx.to_list(embedding)}
  end

  # Semantic similarity
  def similarity(text1, text2) do
    {:ok, emb1} = embed(text1)
    {:ok, emb2} = embed(text2)

    emb1_t = Nx.tensor(emb1)
    emb2_t = Nx.tensor(emb2)

    # Cosine similarity
    dot = Nx.dot(emb1_t, emb2_t)
    norm1 = Nx.LinAlg.norm(emb1_t)
    norm2 = Nx.LinAlg.norm(emb2_t)

    score = Nx.divide(dot, Nx.multiply(norm1, norm2)) |> Nx.to_number()
    {:ok, Float.round(score, 4)}
  end
end
```

---

## 3. Async Inference Pipeline

```elixir
defmodule MyApp.ML.InferencePipeline do
  use GenStage

  def start_link(opts) do
    GenStage.start_link(__MODULE__, opts, name: __MODULE__)
  end

  def init(_opts) do
    {:producer_consumer, %{pending: %{}},
      subscribe_to: [{MyApp.ML.InferenceQueue, max_demand: 10}]}
  end

  def handle_events(events, _from, state) do
    results = Enum.map(events, fn %{id: id, text: text} ->
      case MyApp.ML.SentimentService.analyze(text) do
        {:ok, result} -> %{id: id, result: result, status: :ok}
        {:error, reason} -> %{id: id, error: reason, status: :error}
      end
    end)

    {:noreply, results, state}
  end
end

# Queue-based async inference
defmodule MyApp.ML.InferenceWorker do
  use Oban.Worker, queue: :ml_inference, max_attempts: 3

  @impl Oban.Worker
  def perform(%Oban.Job{args: %{"text" => text, "record_id" => record_id, "type" => type}}) do
    result = case type do
      "sentiment" -> MyApp.ML.SentimentService.analyze(text)
      "embedding" -> MyApp.ML.EmbeddingService.embed(text)
    end

    case result do
      {:ok, data} ->
        MyApp.ML.Results.store(record_id, type, data)
        # Broadcast result via PubSub
        Phoenix.PubSub.broadcast(
          MyApp.PubSub,
          "ml:#{record_id}",
          {:ml_result, type, data}
        )
        :ok

      {:error, reason} ->
        {:error, reason}
    end
  end
end
```

---

## 4. A/B Testing Models

```elixir
defmodule MyApp.ML.ModelRouter do
  @models %{
    "sentiment_v1" => MyApp.SentimentV1,
    "sentiment_v2" => MyApp.SentimentV2
  }

  def analyze(text, user_id) do
    model = select_model(user_id)

    start = System.monotonic_time(:millisecond)
    result = Nx.Serving.run(@models[model], text)
    duration = System.monotonic_time(:millisecond) - start

    # Track for A/B comparison
    :telemetry.execute(
      [:my_app, :ml, :inference],
      %{duration: duration},
      %{model: model, user_id: user_id}
    )

    {:ok, Map.put(result, :model, model)}
  end

  # Hash-based model selection (deterministic per user)
  defp select_model(user_id) do
    bucket = :erlang.phash2(user_id, 100)  # 0-99
    cond do
      bucket < 20 -> "sentiment_v2"  # 20% traffic to new model
      true -> "sentiment_v1"
    end
  end
end
```

---

## 5. Model Health Monitoring

```elixir
defmodule MyApp.ML.HealthCheck do
  use GenServer
  require Logger

  @check_interval 60_000

  def start_link(_) do
    GenServer.start_link(__MODULE__, %{}, name: __MODULE__)
  end

  def init(state) do
    schedule_check()
    {:ok, state}
  end

  def handle_info(:check, state) do
    test_cases = [
      {"I love this product! Amazing!", "POSITIVE"},
      {"This is terrible, very disappointed.", "NEGATIVE"}
    ]

    results = Enum.map(test_cases, fn {text, expected} ->
      case MyApp.ML.SentimentService.analyze(text) do
        {:ok, %{sentiment: label}} when label == expected -> :pass
        result ->
          Logger.error("Model health check failed: expected #{expected}, got #{inspect(result)}")
          :fail
      end
    end)

    failures = Enum.count(results, &(&1 == :fail))

    if failures > 0 do
      :telemetry.execute([:my_app, :ml, :health], %{failures: failures}, %{})
      Logger.error("ML model health check: #{failures} failures")
    end

    schedule_check()
    {:noreply, state}
  end

  defp schedule_check, do: Process.send_after(self(), :check, @check_interval)
end
```

---

## สรุป

```
ML in Production:
├── Nx.Serving: batching + concurrency
├── EXLA/TorchX: hardware acceleration
├── Async: Oban worker for background
└── Batch timeout: 100ms default

Model Management:
├── Load at startup: warm models ready
├── A/B testing: phash2 for routing
├── Health check: GenServer periodic test
└── Monitoring: telemetry + Prometheus

Performance:
├── Batch inference: more efficient than 1-by-1
├── GPU: EXLA with CUDA (production)
├── Compile: JIT compilation first run slow
└── Cache embeddings: store in DB

Bumblebee Models:
├── text_classification: sentiment, intent
├── fill_mask: masked language model
├── text_generation: GPT-style completion
├── image_classification: vision models
└── zero_shot_classification: no-fine-tune
```

---

*ก่อนหน้า: [Part 98](part_98.md) | ต่อไป: [Part 100 - Final Project: World-class SaaS Platform](part_100.md)*
