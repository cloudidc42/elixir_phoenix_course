# Part 60: Machine Learning Integration

## เป้าหมายการเรียนรู้
- ใช้ Nx (Numerical Elixir) สำหรับ tensor operations
- สร้าง neural networks ด้วย Axon
- โหลด pre-trained models ด้วย Bumblebee
- Serve ML models ใน Phoenix
- Real-time inference ด้วย LiveView
- สร้าง Sentiment Analysis และ Image Classification

---

## 1. Nx (Numerical Elixir) - พื้นฐาน

Nx เป็น library สำหรับ numerical computing ใน Elixir รองรับ CPU และ GPU (via XLA/EXLA)

```elixir
# mix.exs
defp deps do
  [
    {:nx, "~> 0.7"},
    {:exla, "~> 0.7"},   # XLA backend สำหรับ GPU acceleration
    {:axon, "~> 0.6"},   # Neural Networks
    {:bumblebee, "~> 0.5"},  # Pre-trained models
    {:phoenix, "~> 1.7"},
    {:phoenix_live_view, "~> 0.20"}
  ]
end
```

```elixir
# กำหนด backend (EXLA สำหรับ production performance)
# config/config.exs
config :nx, default_backend: EXLA.Backend
```

```elixir
# iex: สร้างและจัดการ Tensors
# Tensor เป็น multi-dimensional array
t1 = Nx.tensor([1, 2, 3, 4])
# #Nx.Tensor<
#   s64[4]
#   [1, 2, 3, 4]
# >

# Matrix (2D tensor)
matrix = Nx.tensor([[1, 2, 3], [4, 5, 6], [7, 8, 9]])
# shape: {3, 3}

# Float tensor
floats = Nx.tensor([1.0, 2.5, 3.7], type: :f32)

# Tensor operations
a = Nx.tensor([1, 2, 3])
b = Nx.tensor([4, 5, 6])

Nx.add(a, b)      # [5, 7, 9]
Nx.multiply(a, b) # [4, 10, 18]
Nx.dot(a, b)      # 32 (dot product)

# Matrix multiplication
m1 = Nx.tensor([[1, 2], [3, 4]])
m2 = Nx.tensor([[5, 6], [7, 8]])
Nx.dot(m1, m2)
# [[19, 22], [43, 50]]
```

```elixir
# lib/ml_example/tensor_ops.ex
defmodule MLExample.TensorOps do
  import Nx.Defn  # เปิดใช้ defn สำหรับ compile และ JIT

  # defn สำหรับ numerical functions ที่ compile ได้
  defn sigmoid(x) do
    1.0 / (1.0 + Nx.exp(-x))
  end

  defn relu(x) do
    Nx.max(x, 0)
  end

  defn normalize(tensor) do
    mean = Nx.mean(tensor)
    std = Nx.standard_deviation(tensor)
    (tensor - mean) / std
  end

  defn softmax(x) do
    exp_x = Nx.exp(x)
    exp_x / Nx.sum(exp_x)
  end

  # Linear regression prediction
  defn linear_predict(x, weights, bias) do
    Nx.dot(x, weights) + bias
  end

  # Mean Squared Error loss
  defn mse_loss(predictions, targets) do
    diff = predictions - targets
    Nx.mean(Nx.multiply(diff, diff))
  end
end

# ใช้งาน
tensor = Nx.tensor([-2.0, -1.0, 0.0, 1.0, 2.0])
MLExample.TensorOps.sigmoid(tensor)
# Nx.Tensor: [0.1192, 0.2689, 0.5, 0.7311, 0.8808]
```

---

## 2. ตัวอย่าง: Simple Linear Regression ด้วย Nx

```elixir
# lib/ml_example/linear_regression.ex
defmodule MLExample.LinearRegression do
  import Nx.Defn

  defn predict(x, params) do
    %{weight: w, bias: b} = params
    x * w + b
  end

  defn loss(x, y, params) do
    predictions = predict(x, params)
    diff = predictions - y
    Nx.mean(diff * diff)
  end

  defn update_params(x, y, params, learning_rate) do
    # คำนวณ gradients
    {_loss, gradients} = Nx.Defn.value_and_grad(
      params,
      fn p -> loss(x, y, p) end
    )

    # Update parameters
    %{
      weight: params.weight - learning_rate * gradients.weight,
      bias: params.bias - learning_rate * gradients.bias
    }
  end

  def train(x_data, y_data, epochs \\ 1000, lr \\ 0.01) do
    x = Nx.tensor(x_data, type: :f32)
    y = Nx.tensor(y_data, type: :f32)

    # Initial parameters
    params = %{
      weight: Nx.tensor(0.0, type: :f32),
      bias: Nx.tensor(0.0, type: :f32)
    }

    # Training loop
    Enum.reduce(1..epochs, params, fn epoch, params ->
      params = update_params(x, y, params, lr)

      if rem(epoch, 100) == 0 do
        current_loss = loss(x, y, params)
        IO.puts("Epoch #{epoch}: loss = #{Nx.to_number(current_loss)}")
      end

      params
    end)
  end
end

# ทดสอบ: y = 2x + 1
x_train = [1.0, 2.0, 3.0, 4.0, 5.0]
y_train = [3.0, 5.0, 7.0, 9.0, 11.0]

params = MLExample.LinearRegression.train(x_train, y_train)
IO.puts("Weight: #{Nx.to_number(params.weight)}")  # ~2.0
IO.puts("Bias: #{Nx.to_number(params.bias)}")      # ~1.0
```

---

## 3. Axon สำหรับ Neural Networks

```elixir
# lib/ml_example/neural_net.ex
defmodule MLExample.NeuralNet do
  @moduledoc """
  Neural Network สำหรับ classification
  """

  # สร้าง model architecture
  def build_model(input_size, hidden_size, output_size) do
    Axon.input("input", shape: {nil, input_size})
    |> Axon.dense(hidden_size, activation: :relu)
    |> Axon.dropout(rate: 0.2)
    |> Axon.dense(hidden_size, activation: :relu)
    |> Axon.dropout(rate: 0.2)
    |> Axon.dense(output_size, activation: :softmax)
  end

  # Train model
  def train_model(model, train_data, opts \\ []) do
    epochs = Keyword.get(opts, :epochs, 50)
    batch_size = Keyword.get(opts, :batch_size, 32)
    learning_rate = Keyword.get(opts, :learning_rate, 0.001)

    model
    |> Axon.Loop.trainer(
      :categorical_cross_entropy,
      Axon.Optimizers.adam(learning_rate)
    )
    |> Axon.Loop.metric(:accuracy)
    |> Axon.Loop.run(
      train_data,
      %{},  # initial params
      epochs: epochs,
      compiler: EXLA
    )
  end

  # Predict
  def predict(model, params, input) do
    input_tensor = Nx.tensor(input, type: :f32)
    Axon.predict(model, params, %{"input" => input_tensor})
  end
end
```

```elixir
# ตัวอย่าง: XOR classification
defmodule MLExample.XORExample do
  def run do
    # Training data: XOR problem
    x = Nx.tensor([[0, 0], [0, 1], [1, 0], [1, 1]], type: :f32)
    y = Nx.tensor([[1, 0], [0, 1], [0, 1], [1, 0]], type: :f32)

    train_data = [{%{"input" => x}, y}]
    |> Stream.cycle()
    |> Stream.take(1000)

    model = MLExample.NeuralNet.build_model(2, 8, 2)
    params = MLExample.NeuralNet.train_model(model, train_data, epochs: 100)

    # Test predictions
    test_inputs = [[0, 0], [0, 1], [1, 0], [1, 1]]
    Enum.each(test_inputs, fn input ->
      result = MLExample.NeuralNet.predict(model, params, [input])
      predicted_class = result |> Nx.argmax() |> Nx.to_number()
      IO.puts("#{inspect(input)} -> class #{predicted_class}")
    end)

    {:ok, params}
  end
end
```

---

## 4. Bumblebee สำหรับ Pre-trained Models

Bumblebee ช่วยโหลดและใช้ pre-trained models เช่น BERT, RoBERTa, GPT-2 ได้ง่ายมาก

```elixir
# lib/ml_example/sentiment_analyzer.ex
defmodule MLExample.SentimentAnalyzer do
  @moduledoc """
  Sentiment Analysis โดยใช้ BERT ที่ fine-tuned สำหรับ sentiment
  """

  def start_link(_opts) do
    # โหลด model จาก HuggingFace Hub
    {:ok, model_info} = Bumblebee.load_model(
      {:hf, "finiteautomata/bertweet-base-sentiment-analysis"}
    )

    {:ok, tokenizer} = Bumblebee.load_tokenizer(
      {:hf, "finiteautomata/bertweet-base-sentiment-analysis"}
    )

    # สร้าง serving (pipeline สำหรับ inference)
    serving = Bumblebee.Text.text_classification(
      model_info,
      tokenizer,
      top_k: 3,
      compile: [batch_size: 8, sequence_length: 512],
      defn_options: [compiler: EXLA]
    )

    Nx.Serving.start_link(serving: serving, name: SentimentServing)
  end

  def analyze(text) when is_binary(text) do
    Nx.Serving.run(SentimentServing, text)
  end

  def analyze_batch(texts) when is_list(texts) do
    Nx.Serving.run(SentimentServing, texts)
  end
end
```

```elixir
# application.ex - เพิ่มใน supervision tree
children = [
  # ... other children ...
  {MLExample.SentimentAnalyzer, []}
]
```

---

## 5. Image Classification ด้วย ResNet

```elixir
# lib/ml_example/image_classifier.ex
defmodule MLExample.ImageClassifier do
  @moduledoc """
  Image Classification ด้วย ResNet50
  """

  def start_serving do
    {:ok, model_info} = Bumblebee.load_model({:hf, "microsoft/resnet-50"})
    {:ok, featurizer} = Bumblebee.load_featurizer({:hf, "microsoft/resnet-50"})

    serving = Bumblebee.Vision.image_classification(
      model_info,
      featurizer,
      top_k: 5,
      compile: [batch_size: 4],
      defn_options: [compiler: EXLA]
    )

    Nx.Serving.start_link(
      serving: serving,
      name: ImageClassifierServing,
      batch_size: 4,
      batch_timeout: 100
    )
  end

  # รับ image path และคืนค่า top predictions
  def classify_image(image_path) do
    image = image_path |> File.read!() |> decode_image()
    Nx.Serving.run(ImageClassifierServing, image)
  end

  # รับ base64 encoded image
  def classify_base64(base64_string) do
    image = base64_string |> Base.decode64!() |> decode_image()
    Nx.Serving.run(ImageClassifierServing, image)
  end

  defp decode_image(binary) do
    # ใช้ Image library หรือ StbImage
    {:ok, image} = Image.from_binary(binary)
    image
  end
end
```

---

## 6. Text Generation ด้วย GPT-2

```elixir
# lib/ml_example/text_generator.ex
defmodule MLExample.TextGenerator do
  @moduledoc """
  Text generation ด้วย GPT-2
  """

  def start_serving do
    {:ok, model_info} = Bumblebee.load_model({:hf, "gpt2"})
    {:ok, tokenizer} = Bumblebee.load_tokenizer({:hf, "gpt2"})
    {:ok, generation_config} = Bumblebee.load_generation_config({:hf, "gpt2"})

    generation_config = Bumblebee.configure(generation_config,
      max_new_tokens: 100,
      strategy: %{type: :multinomial_sampling, top_p: 0.9}
    )

    serving = Bumblebee.Text.generation(
      model_info,
      tokenizer,
      generation_config,
      compile: [batch_size: 1, sequence_length: 1024],
      defn_options: [compiler: EXLA],
      stream: true  # Enable streaming output
    )

    Nx.Serving.start_link(serving: serving, name: TextGeneratorServing)
  end

  def generate(prompt, opts \\ []) do
    Nx.Serving.run(TextGeneratorServing, prompt)
  end

  # Streaming generation (for real-time output)
  def generate_stream(prompt, callback) do
    serving_output = Nx.Serving.run(TextGeneratorServing, prompt)

    Enum.each(serving_output, fn chunk ->
      callback.(chunk.text_diff)
    end)
  end
end
```

---

## 7. Serving ML Models ใน Phoenix

```elixir
# lib/ml_web/controllers/ml_controller.ex
defmodule MLWeb.MLController do
  use MLWeb, :controller

  alias MLExample.{SentimentAnalyzer, ImageClassifier}

  def analyze_sentiment(conn, %{"text" => text}) do
    result = SentimentAnalyzer.analyze(text)

    top_result = hd(result.predictions)

    json(conn, %{
      text: text,
      sentiment: top_result.label,
      confidence: Float.round(top_result.score * 100, 2),
      all_predictions: Enum.map(result.predictions, fn p ->
        %{label: p.label, score: Float.round(p.score * 100, 2)}
      end)
    })
  end

  def classify_image(conn, %{"image" => %Plug.Upload{} = upload}) do
    result = ImageClassifier.classify_image(upload.path)

    json(conn, %{
      predictions: Enum.map(result.predictions, fn p ->
        %{label: p.label, confidence: Float.round(p.score * 100, 2)}
      end)
    })
  end

  def batch_sentiment(conn, %{"texts" => texts}) when is_list(texts) do
    results = SentimentAnalyzer.analyze_batch(texts)

    json(conn, %{
      results: Enum.zip(texts, results) |> Enum.map(fn {text, result} ->
        %{
          text: text,
          sentiment: hd(result.predictions).label,
          confidence: hd(result.predictions).score
        }
      end)
    })
  end
end
```

---

## 8. Real-time ML Inference กับ LiveView

```elixir
# lib/ml_web/live/sentiment_live.ex
defmodule MLWeb.SentimentLive do
  use MLWeb, :live_view

  alias MLExample.SentimentAnalyzer

  @impl true
  def mount(_params, _session, socket) do
    {:ok, assign(socket,
      text: "",
      analyzing: false,
      result: nil,
      history: []
    )}
  end

  @impl true
  def handle_event("analyze", %{"text" => text}, socket) when byte_size(text) > 0 do
    # ส่ง task ไปทำงาน async เพื่อไม่ block LiveView process
    task = Task.async(fn -> SentimentAnalyzer.analyze(text) end)

    {:noreply, socket
      |> assign(:analyzing, true)
      |> assign(:text, text)
      |> assign(:task_ref, task.ref)}
  end

  @impl true
  def handle_event("analyze", _params, socket) do
    {:noreply, socket}
  end

  # รับผลลัพธ์จาก async task
  @impl true
  def handle_info({ref, result}, socket) when socket.assigns.task_ref == ref do
    Process.demonitor(ref, [:flush])

    top = hd(result.predictions)
    history_entry = %{
      text: socket.assigns.text,
      sentiment: top.label,
      confidence: Float.round(top.score * 100, 1),
      analyzed_at: DateTime.utc_now()
    }

    {:noreply, socket
      |> assign(:analyzing, false)
      |> assign(:result, result)
      |> update(:history, fn h -> [history_entry | h] |> Enum.take(10) end)}
  end

  @impl true
  def render(assigns) do
    ~H"""
    <div class="max-w-2xl mx-auto p-6">
      <h1 class="text-2xl font-bold mb-6">Sentiment Analyzer</h1>

      <form phx-submit="analyze" class="mb-6">
        <textarea
          name="text"
          value={@text}
          placeholder="พิมพ์ข้อความที่ต้องการวิเคราะห์..."
          class="w-full p-3 border rounded-lg h-32"
        ></textarea>
        <button
          type="submit"
          disabled={@analyzing}
          class="mt-2 px-6 py-2 bg-blue-600 text-white rounded-lg disabled:opacity-50"
        >
          <%= if @analyzing, do: "กำลังวิเคราะห์...", else: "วิเคราะห์" %>
        </button>
      </form>

      <%= if @result do %>
        <div class="p-4 bg-gray-100 rounded-lg mb-6">
          <h2 class="font-bold text-lg mb-2">ผลลัพธ์:</h2>
          <%= for prediction <- @result.predictions do %>
            <div class="flex items-center gap-2 mb-2">
              <span class={sentiment_color(prediction.label)}>
                <%= prediction.label %>
              </span>
              <div class="flex-1 bg-gray-200 rounded-full h-4">
                <div
                  class="bg-blue-500 h-4 rounded-full"
                  style={"width: #{prediction.score * 100}%"}
                ></div>
              </div>
              <span><%= Float.round(prediction.score * 100, 1) %>%</span>
            </div>
          <% end %>
        </div>
      <% end %>

      <%= if length(@history) > 0 do %>
        <div>
          <h2 class="font-bold text-lg mb-2">ประวัติการวิเคราะห์:</h2>
          <%= for entry <- @history do %>
            <div class="p-3 border rounded mb-2 flex justify-between items-center">
              <span class="truncate flex-1"><%= entry.text %></span>
              <span class={["ml-4 font-bold", sentiment_color(entry.sentiment)]}>
                <%= entry.sentiment %> (<%= entry.confidence %>%)
              </span>
            </div>
          <% end %>
        </div>
      <% end %>
    </div>
    """
  end

  defp sentiment_color("POS"), do: "text-green-600"
  defp sentiment_color("NEG"), do: "text-red-600"
  defp sentiment_color("NEU"), do: "text-gray-600"
  defp sentiment_color(_), do: "text-gray-600"
end
```

---

## 9. Image Classification LiveView

```elixir
# lib/ml_web/live/image_classifier_live.ex
defmodule MLWeb.ImageClassifierLive do
  use MLWeb, :live_view

  alias MLExample.ImageClassifier

  @impl true
  def mount(_params, _session, socket) do
    {:ok, assign(socket,
      classifying: false,
      predictions: nil,
      preview_url: nil,
      error: nil
    )}
  end

  @impl true
  def handle_event("classify", _params, socket) do
    {:noreply, socket}
  end

  @impl true
  def handle_event("upload_image", %{"image_data" => base64_data}, socket) do
    socket = assign(socket,
      classifying: true,
      preview_url: "data:image/jpeg;base64,#{base64_data}",
      error: nil
    )

    # Async classification
    task = Task.async(fn ->
      ImageClassifier.classify_base64(base64_data)
    end)

    {:noreply, assign(socket, :task_ref, task.ref)}
  end

  @impl true
  def handle_info({ref, result}, socket) when socket.assigns[:task_ref] == ref do
    Process.demonitor(ref, [:flush])

    {:noreply, socket
      |> assign(:classifying, false)
      |> assign(:predictions, result.predictions)}
  end

  @impl true
  def handle_info({ref, {:error, reason}}, socket) when socket.assigns[:task_ref] == ref do
    Process.demonitor(ref, [:flush])

    {:noreply, socket
      |> assign(:classifying, false)
      |> assign(:error, inspect(reason))}
  end

  @impl true
  def render(assigns) do
    ~H"""
    <div class="max-w-2xl mx-auto p-6">
      <h1 class="text-2xl font-bold mb-6">Image Classifier</h1>

      <div
        class="border-2 border-dashed border-gray-300 rounded-lg p-8 text-center cursor-pointer"
        phx-hook="ImageUploader"
        id="image-uploader"
      >
        <%= if @preview_url do %>
          <img src={@preview_url} class="max-h-64 mx-auto mb-4" />
        <% else %>
          <p class="text-gray-500">คลิกหรือลากไฟล์ภาพมาที่นี่</p>
        <% end %>
      </div>

      <%= if @classifying do %>
        <div class="mt-4 text-center text-blue-600">กำลังวิเคราะห์ภาพ...</div>
      <% end %>

      <%= if @predictions do %>
        <div class="mt-4">
          <h2 class="font-bold text-lg mb-3">ผลลัพธ์:</h2>
          <%= for {pred, i} <- Enum.with_index(@predictions) do %>
            <div class="flex items-center gap-3 mb-2">
              <span class="w-6 text-gray-500 text-sm"><%= i + 1 %>.</span>
              <span class="flex-1"><%= pred.label %></span>
              <div class="w-32 bg-gray-200 rounded-full h-3">
                <div
                  class="bg-indigo-500 h-3 rounded-full"
                  style={"width: #{pred.score * 100}%"}
                ></div>
              </div>
              <span class="text-sm text-gray-600 w-16 text-right">
                <%= Float.round(pred.score * 100, 1) %>%
              </span>
            </div>
          <% end %>
        </div>
      <% end %>

      <%= if @error do %>
        <div class="mt-4 p-3 bg-red-100 text-red-700 rounded">
          Error: <%= @error %>
        </div>
      <% end %>
    </div>
    """
  end
end
```

---

## 10. Batching สำหรับ Performance

```elixir
# lib/ml_example/batch_serving.ex
defmodule MLExample.BatchServing do
  @moduledoc """
  ปรับแต่ง serving สำหรับ high throughput
  """

  def start_sentiment_serving do
    {:ok, model_info} = Bumblebee.load_model(
      {:hf, "finiteautomata/bertweet-base-sentiment-analysis"}
    )
    {:ok, tokenizer} = Bumblebee.load_tokenizer(
      {:hf, "finiteautomata/bertweet-base-sentiment-analysis"}
    )

    serving = Bumblebee.Text.text_classification(model_info, tokenizer,
      top_k: 1,
      # Compile สำหรับ batch sizes ต่างๆ
      compile: [batch_size: 32, sequence_length: 256],
      defn_options: [compiler: EXLA]
    )

    # Configure batching
    Nx.Serving.start_link(
      serving: serving,
      name: BatchSentimentServing,
      batch_size: 32,       # รวม requests ใน batch 32 ชุด
      batch_timeout: 50     # หรือ flush ทุก 50ms
    )
  end

  # ทดสอบ throughput
  def benchmark do
    texts = Enum.map(1..1000, fn i ->
      "This is test sentence number #{i}. It could be positive or negative."
    end)

    start_time = System.monotonic_time(:millisecond)

    # ส่ง requests พร้อมกัน 100 tasks
    tasks = Enum.map(texts, fn text ->
      Task.async(fn ->
        Nx.Serving.run(BatchSentimentServing, text)
      end)
    end)

    results = Enum.map(tasks, &Task.await(&1, 30_000))
    end_time = System.monotonic_time(:millisecond)

    duration_ms = end_time - start_time
    throughput = length(results) / (duration_ms / 1000)

    IO.puts("Processed #{length(results)} texts in #{duration_ms}ms")
    IO.puts("Throughput: #{Float.round(throughput, 1)} texts/second")

    {:ok, results}
  end
end
```

---

## สรุป

```
ML Stack in Elixir:

┌─────────────────────────────────────────────┐
│              Phoenix Application             │
│  ┌──────────────┐    ┌─────────────────┐    │
│  │  LiveView    │    │   REST API      │    │
│  │  (real-time) │    │   Controller    │    │
│  └──────┬───────┘    └────────┬────────┘    │
│         └───────────┬─────────┘            │
│               ┌─────▼─────┐                │
│               │Nx.Serving │                │
│               │ (batching) │                │
│               └─────┬─────┘                │
└─────────────────────┼────────────────────────┘
                       │
         ┌─────────────┼─────────────┐
         ▼             ▼             ▼
    Bumblebee        Axon          Nx
  (Pre-trained)  (Custom NN)   (Tensors)
    BERT/GPT2     Training      Math ops
    ResNet50       Axon API      EXLA GPU

Key Libraries:
- Nx: tensor operations, numerical computing
- Axon: neural network training
- Bumblebee: pre-trained HuggingFace models
- EXLA: XLA compiler (GPU acceleration)
- Nx.Serving: production inference serving

Tips:
1. ใช้ EXLA backend สำหรับ production
2. Nx.Serving รองรับ automatic batching
3. Task.async เพื่อ non-blocking inference ใน LiveView
4. Load models ครั้งเดียวตอน startup
5. compile: [batch_size: N] เพื่อ optimize สำหรับ batch
```

---

*ก่อนหน้า: [Part 59 - Stream Processing](part_59.md) | ต่อไป: [Part 61 - WebAssembly and Nerves IoT](part_61.md)*
