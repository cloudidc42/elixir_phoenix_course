# Part 61: WebAssembly and Nerves IoT

## เป้าหมายการเรียนรู้
- เข้าใจ Nerves framework สำหรับ IoT ด้วย Elixir
- ตั้งค่า Nerves project และ build สำหรับ embedded devices
- ควบคุม GPIO และเซ็นเซอร์ต่างๆ
- ส่งข้อมูลเซ็นเซอร์ไปยัง Phoenix server
- ใช้ WasmEx สำหรับ WebAssembly ใน Elixir
- Pattern สำหรับ embedded systems

---

## 1. Nerves คืออะไร?

Nerves เป็น framework ที่ทำให้เขียน firmware สำหรับ IoT devices เช่น Raspberry Pi ด้วย Elixir ได้อย่างง่ายดาย

ข้อดีของ Nerves:
- **OTP**: ใช้ supervision trees และ fault tolerance ของ BEAM
- **Small footprint**: image ขนาดเล็ก (ประมาณ 30-40MB)
- **Immutable firmware**: update แบบ atomic
- **Remote access**: SSH เข้า device ได้โดยตรง
- **Hot code reload**: update code ได้โดยไม่ต้องรีสตาร์ท

```
Nerves Stack:
┌────────────────────────────────────┐
│        Your Elixir Application     │
├────────────────────────────────────┤
│          Nerves Runtime            │
├────────────────────────────────────┤
│      Erlang/OTP + BEAM VM          │
├────────────────────────────────────┤
│         Linux Kernel               │
├────────────────────────────────────┤
│      Hardware (Raspberry Pi)       │
└────────────────────────────────────┘
```

---

## 2. ตั้งค่า Nerves Project

```bash
# ติดตั้ง Nerves archive
mix archive.install hex nerves_bootstrap

# สร้าง Nerves project ใหม่
mix nerves.new my_iot_device

# หรือเพิ่มเข้าใน existing project
# ดู https://hexdocs.pm/nerves/
```

```elixir
# mix.exs สำหรับ Nerves project
defmodule MyIoTDevice.MixProject do
  use Mix.Project

  @app :my_iot_device
  @version "0.1.0"
  @all_targets [:rpi0, :rpi3, :rpi4, :bbb, :x86_64]

  def project do
    [
      app: @app,
      version: @version,
      elixir: "~> 1.15",
      archives: [nerves_bootstrap: "~> 1.11"],
      start_permanent: Mix.env() == :prod,
      deps: deps(),
      releases: [{@app, release()}],
      preferred_cli_target: [run: :host, test: :host]
    ]
  end

  def application do
    [
      mod: {MyIoTDevice.Application, []},
      extra_applications: [:logger, :runtime_tools]
    ]
  end

  defp deps do
    [
      # Nerves core
      {:nerves, "~> 1.10", runtime: false},
      {:shoehorn, "~> 0.9"},
      {:ring_logger, "~> 0.10"},
      {:toolshed, "~> 0.3"},

      # Target-specific dependencies
      {:nerves_runtime, "~> 0.13"},
      {:nerves_pack, "~> 0.7"},

      # GPIO and hardware
      {:circuits_gpio, "~> 1.0"},
      {:circuits_i2c, "~> 1.0"},
      {:circuits_spi, "~> 1.0"},
      {:circuits_uart, "~> 1.4"},

      # Networking
      {:nerves_network_interface, "~> 0.4"},

      # Target system images
      {:nerves_system_rpi4, "~> 1.22", runtime: false, targets: :rpi4},
      {:nerves_system_rpi3, "~> 1.22", runtime: false, targets: :rpi3},

      # HTTP client สำหรับ send data to Phoenix
      {:finch, "~> 0.16"},
      {:jason, "~> 1.4"},

      # Sensors
      {:bmp280, "~> 0.3"},  # Temperature/Pressure sensor
    ] ++ local_only_deps()
  end

  defp local_only_deps do
    [
      {:ex_doc, "~> 0.30", only: :dev, runtime: false}
    ]
  end

  defp release do
    [
      overwrite: true,
      cookie: "#{@app}_cookie",
      include_erts: &Nerves.Release.erts/0,
      steps: [&Nerves.Release.init/1, :assemble],
      strip_beams: Mix.env() == :prod
    ]
  end
end
```

```bash
# Build firmware สำหรับ Raspberry Pi 4
export MIX_TARGET=rpi4
mix deps.get
mix firmware

# Burn firmware ลง SD card
mix firmware.burn

# Upload firmware ผ่าน SSH (สำหรับ update)
mix firmware.gen.script
./upload.sh
```

---

## 3. GPIO - ควบคุม LED และ Button

```elixir
# lib/my_iot_device/gpio_controller.ex
defmodule MyIoTDevice.GpioController do
  @moduledoc """
  ควบคุม GPIO สำหรับ LED และ Button
  """

  use GenServer
  require Logger

  alias Circuits.GPIO

  @led_pin 18      # GPIO pin สำหรับ LED
  @button_pin 23   # GPIO pin สำหรับ Button

  def start_link(opts \\ []) do
    GenServer.start_link(__MODULE__, opts, name: __MODULE__)
  end

  # Client API
  def led_on, do: GenServer.call(__MODULE__, :led_on)
  def led_off, do: GenServer.call(__MODULE__, :led_off)
  def blink(times \\ 3), do: GenServer.cast(__MODULE__, {:blink, times})
  def button_state, do: GenServer.call(__MODULE__, :button_state)

  # Server callbacks
  @impl true
  def init(_opts) do
    # เปิด GPIO pins
    {:ok, led_gpio} = GPIO.open(@led_pin, :output)
    {:ok, button_gpio} = GPIO.open(@button_pin, :input, pull_mode: :pullup)

    # Subscribe ไปยัง button interrupts
    :ok = GPIO.set_interrupts(button_gpio, :both)

    Logger.info("GPIO Controller started - LED: pin #{@led_pin}, Button: pin #{@button_pin}")

    {:ok, %{
      led_gpio: led_gpio,
      button_gpio: button_gpio,
      led_state: :off,
      button_pressed: false
    }}
  end

  @impl true
  def handle_call(:led_on, _from, state) do
    :ok = GPIO.write(state.led_gpio, 1)
    {:reply, :ok, %{state | led_state: :on}}
  end

  @impl true
  def handle_call(:led_off, _from, state) do
    :ok = GPIO.write(state.led_gpio, 0)
    {:reply, :ok, %{state | led_state: :off}}
  end

  @impl true
  def handle_call(:button_state, _from, state) do
    {:reply, state.button_pressed, state}
  end

  @impl true
  def handle_cast({:blink, times}, state) do
    Enum.each(1..times, fn _ ->
      GPIO.write(state.led_gpio, 1)
      Process.sleep(200)
      GPIO.write(state.led_gpio, 0)
      Process.sleep(200)
    end)
    {:noreply, %{state | led_state: :off}}
  end

  # รับ GPIO interrupts จาก button
  @impl true
  def handle_info({:circuits_gpio, @button_pin, _timestamp, 0}, state) do
    # Button pressed (active low เพราะ pullup)
    Logger.info("Button pressed!")
    MyIoTDevice.EventBus.publish(:button_pressed)
    {:noreply, %{state | button_pressed: true}}
  end

  @impl true
  def handle_info({:circuits_gpio, @button_pin, _timestamp, 1}, state) do
    # Button released
    Logger.info("Button released!")
    MyIoTDevice.EventBus.publish(:button_released)
    {:noreply, %{state | button_pressed: false}}
  end

  @impl true
  def terminate(_reason, state) do
    GPIO.close(state.led_gpio)
    GPIO.close(state.button_gpio)
    :ok
  end
end
```

---

## 4. I2C Sensor - อ่านค่า Temperature และ Humidity

```elixir
# lib/my_iot_device/sensor_reader.ex
defmodule MyIoTDevice.SensorReader do
  @moduledoc """
  อ่านค่าจาก DHT22 temperature/humidity sensor ผ่าน I2C
  หรือใช้ BMP280 สำหรับ temperature/pressure
  """

  use GenServer
  require Logger

  alias Circuits.I2C

  @i2c_bus "i2c-1"
  @bmp280_address 0x76
  @read_interval 5_000  # อ่านทุก 5 วินาที

  def start_link(opts \\ []) do
    GenServer.start_link(__MODULE__, opts, name: __MODULE__)
  end

  def get_reading do
    GenServer.call(__MODULE__, :get_reading)
  end

  @impl true
  def init(_opts) do
    {:ok, i2c_ref} = I2C.open(@i2c_bus)

    # Initialize BMP280
    :ok = init_bmp280(i2c_ref)

    # Schedule first reading
    Process.send_after(self(), :read_sensor, 1_000)

    {:ok, %{
      i2c_ref: i2c_ref,
      last_reading: nil,
      read_count: 0
    }}
  end

  @impl true
  def handle_call(:get_reading, _from, state) do
    {:reply, state.last_reading, state}
  end

  @impl true
  def handle_info(:read_sensor, state) do
    reading = case read_bmp280(state.i2c_ref) do
      {:ok, data} ->
        %{
          temperature_c: data.temperature,
          pressure_hpa: data.pressure,
          altitude_m: calculate_altitude(data.pressure),
          timestamp: DateTime.utc_now()
        }

      {:error, reason} ->
        Logger.error("Failed to read sensor: #{inspect(reason)}")
        nil
    end

    # Publish reading เพื่อให้ส่วนอื่นๆ ได้รับข้อมูล
    if reading do
      MyIoTDevice.DataReporter.report_sensor_data(reading)
    end

    # Schedule next reading
    Process.send_after(self(), :read_sensor, @read_interval)

    {:noreply, %{state |
      last_reading: reading || state.last_reading,
      read_count: state.read_count + 1
    }}
  end

  defp init_bmp280(i2c_ref) do
    # Reset device
    I2C.write(i2c_ref, @bmp280_address, <<0xE0, 0xB6>>)
    Process.sleep(10)

    # Configure for normal mode, oversampling x2
    I2C.write(i2c_ref, @bmp280_address, <<0xF4, 0x57>>)
    I2C.write(i2c_ref, @bmp280_address, <<0xF5, 0xA0>>)
    :ok
  end

  defp read_bmp280(i2c_ref) do
    # อ่านค่าดิบจาก BMP280 registers
    case I2C.write_read(i2c_ref, @bmp280_address, <<0xF7>>, 6) do
      {:ok, <<p_msb, p_lsb, p_xlsb, t_msb, t_lsb, t_xlsb>>} ->
        raw_pressure = (p_msb <<< 12) ||| (p_lsb <<< 4) ||| (p_xlsb >>> 4)
        raw_temp = (t_msb <<< 12) ||| (t_lsb <<< 4) ||| (t_xlsb >>> 4)

        # ใน production ต้องใช้ calibration data จาก BMP280 registers
        temperature = compensate_temperature(raw_temp)
        pressure = compensate_pressure(raw_pressure)

        {:ok, %{temperature: temperature, pressure: pressure}}

      {:error, _} = error ->
        error
    end
  end

  defp compensate_temperature(raw) do
    # Simplified; ใน production ใช้ calibration coefficients จาก chip
    raw / 5120.0
  end

  defp compensate_pressure(raw) do
    # hPa
    raw / 25600.0
  end

  defp calculate_altitude(pressure_hpa) do
    # Barometric formula
    sea_level_pressure = 1013.25
    44330.0 * (1.0 - :math.pow(pressure_hpa / sea_level_pressure, 0.1903))
  end
end
```

---

## 5. ส่งข้อมูลเซ็นเซอร์ไปยัง Phoenix

```elixir
# lib/my_iot_device/data_reporter.ex
defmodule MyIoTDevice.DataReporter do
  @moduledoc """
  ส่งข้อมูลเซ็นเซอร์ไปยัง Phoenix server
  รองรับ HTTP POST และ WebSocket (Phoenix Channels)
  """

  use GenServer
  require Logger

  @phoenix_url Application.compile_env(:my_iot_device, :phoenix_url, "http://192.168.1.100:4000")
  @device_id Application.compile_env(:my_iot_device, :device_id, "device-001")
  @api_key Application.compile_env(:my_iot_device, :api_key, "secret")

  # Client API
  def start_link(opts \\ []) do
    GenServer.start_link(__MODULE__, opts, name: __MODULE__)
  end

  def report_sensor_data(data) do
    GenServer.cast(__MODULE__, {:report, data})
  end

  # Server callbacks
  @impl true
  def init(_opts) do
    # Start Finch HTTP client
    {:ok, _} = Finch.start_link(name: MyFinch)

    # Queue สำหรับ offline buffering
    {:ok, %{
      queue: :queue.new(),
      connected: false,
      retry_count: 0
    }}
  end

  @impl true
  def handle_cast({:report, data}, state) do
    payload = build_payload(data)

    case send_to_phoenix(payload) do
      :ok ->
        Logger.debug("Sensor data sent successfully")
        # Flush queued data ถ้ามี
        new_state = flush_queue(state)
        {:noreply, %{new_state | connected: true, retry_count: 0}}

      {:error, reason} ->
        Logger.warning("Failed to send data: #{inspect(reason)}. Queuing...")
        # Queue data เพื่อส่งใหม่ทีหลัง
        new_queue = :queue.in(payload, state.queue)
        schedule_retry(state.retry_count)
        {:noreply, %{state | queue: new_queue, connected: false}}
    end
  end

  @impl true
  def handle_info(:retry, state) do
    case :queue.out(state.queue) do
      {:empty, _} ->
        {:noreply, state}

      {{:value, payload}, remaining_queue} ->
        case send_to_phoenix(payload) do
          :ok ->
            new_state = flush_queue(%{state | queue: remaining_queue})
            {:noreply, %{new_state | connected: true, retry_count: 0}}

          {:error, _} ->
            schedule_retry(state.retry_count + 1)
            {:noreply, %{state | retry_count: state.retry_count + 1}}
        end
    end
  end

  defp send_to_phoenix(payload) do
    url = "#{@phoenix_url}/api/sensor_data"
    body = Jason.encode!(payload)

    request = Finch.build(
      :post,
      url,
      [
        {"Content-Type", "application/json"},
        {"X-Device-ID", @device_id},
        {"X-API-Key", @api_key}
      ],
      body
    )

    case Finch.request(request, MyFinch, receive_timeout: 5_000) do
      {:ok, %Finch.Response{status: status}} when status in 200..299 ->
        :ok

      {:ok, %Finch.Response{status: status}} ->
        {:error, {:http_error, status}}

      {:error, _} = error ->
        error
    end
  end

  defp build_payload(sensor_data) do
    %{
      device_id: @device_id,
      data: sensor_data,
      sent_at: DateTime.utc_now() |> DateTime.to_iso8601()
    }
  end

  defp flush_queue(state) do
    case :queue.out(state.queue) do
      {:empty, _} ->
        state

      {{:value, payload}, remaining} ->
        case send_to_phoenix(payload) do
          :ok -> flush_queue(%{state | queue: remaining})
          {:error, _} -> state
        end
    end
  end

  defp schedule_retry(retry_count) do
    # Exponential backoff: 1s, 2s, 4s, 8s, max 60s
    delay = min(:math.pow(2, retry_count) * 1000, 60_000) |> round()
    Process.send_after(self(), :retry, delay)
  end
end
```

---

## 6. Phoenix Side: รับข้อมูลจาก IoT Devices

```elixir
# lib/my_app_web/controllers/sensor_data_controller.ex
defmodule MyAppWeb.SensorDataController do
  use MyAppWeb, :controller

  alias MyApp.{Devices, SensorReadings}

  def create(conn, params) do
    device_id = get_req_header(conn, "x-device-id") |> List.first()
    api_key = get_req_header(conn, "x-api-key") |> List.first()

    with {:ok, device} <- Devices.authenticate(device_id, api_key),
         {:ok, reading} <- SensorReadings.create(device, params["data"]) do
      # Broadcast ไปยัง LiveView dashboards
      Phoenix.PubSub.broadcast(
        MyApp.PubSub,
        "device:#{device_id}",
        {:new_reading, reading}
      )

      conn |> put_status(:created) |> json(%{status: "ok", id: reading.id})
    else
      {:error, :unauthorized} ->
        conn |> put_status(:unauthorized) |> json(%{error: "Invalid credentials"})

      {:error, changeset} ->
        conn |> put_status(:unprocessable_entity) |> json(%{errors: format_errors(changeset)})
    end
  end
end
```

---

## 7. WasmEx - WebAssembly ใน Elixir

WasmEx ทำให้รัน WebAssembly modules ใน Elixir ได้ เหมาะสำหรับรัน code ที่ compiled จาก Rust, C, Go

```elixir
# mix.exs
{:wasmex, "~> 0.9"}
```

```elixir
# lib/wasm_example.ex
defmodule WasmExample do
  @moduledoc """
  ตัวอย่างการรัน WebAssembly modules ใน Elixir
  """

  # โหลดและรัน WASM module
  def run_wasm_function do
    # โหลด WASM binary
    wasm_bytes = File.read!("priv/wasm/calculator.wasm")

    # สร้าง WASM store และ instance
    {:ok, store} = Wasmex.Store.new()
    {:ok, module} = Wasmex.Module.compile(store, wasm_bytes)
    {:ok, instance} = Wasmex.Instance.new(store, module, %{})

    # เรียกฟังก์ชันใน WASM
    {:ok, [result]} = Wasmex.Instance.call_exported_function(
      store,
      instance,
      "add",
      [5, 3],
      :infinity
    )

    IO.puts("5 + 3 = #{result}")  # 8
  end

  # ตัวอย่าง: รัน Rust code ที่ compiled เป็น WASM
  def run_rust_algorithm do
    wasm_bytes = File.read!("priv/wasm/algorithms.wasm")

    {:ok, store} = Wasmex.Store.new()
    {:ok, module} = Wasmex.Module.compile(store, wasm_bytes)
    {:ok, instance} = Wasmex.Instance.new(store, module, %{})

    # เรียก fibonacci function
    {:ok, [fib_result]} = Wasmex.Instance.call_exported_function(
      store, instance, "fibonacci", [40], :infinity
    )

    IO.puts("fibonacci(40) = #{fib_result}")
  end

  # ส่งข้อมูลผ่าน WASM memory
  def process_data_in_wasm(data) when is_binary(data) do
    wasm_bytes = File.read!("priv/wasm/data_processor.wasm")

    {:ok, store} = Wasmex.Store.new()
    {:ok, module} = Wasmex.Module.compile(store, wasm_bytes)
    {:ok, instance} = Wasmex.Instance.new(store, module, %{})

    # Allocate memory ใน WASM
    data_len = byte_size(data)
    {:ok, [ptr]} = Wasmex.Instance.call_exported_function(
      store, instance, "allocate", [data_len], :infinity
    )

    # Write data ลงใน WASM memory
    {:ok, memory} = Wasmex.Instance.export_memory(store, instance, "memory")
    Wasmex.Memory.write_binary(store, memory, ptr, data)

    # Process data ใน WASM
    {:ok, [result_ptr, result_len]} = Wasmex.Instance.call_exported_function(
      store, instance, "process", [ptr, data_len], :infinity
    )

    # อ่าน result จาก WASM memory
    result = Wasmex.Memory.read_binary(store, memory, result_ptr, result_len)

    # Free allocated memory
    Wasmex.Instance.call_exported_function(store, instance, "deallocate", [ptr, data_len], :infinity)

    result
  end
end
```

```rust
// algorithms.rs - คอมไพล์เป็น WASM
// cargo build --target wasm32-unknown-unknown --release

#[no_mangle]
pub extern "C" fn fibonacci(n: i32) -> i64 {
    if n <= 1 {
        return n as i64;
    }
    let mut a: i64 = 0;
    let mut b: i64 = 1;
    for _ in 2..=n {
        let temp = a + b;
        a = b;
        b = temp;
    }
    b
}

#[no_mangle]
pub extern "C" fn add(a: i32, b: i32) -> i32 {
    a + b
}
```

---

## 8. Embedded Systems Patterns

```elixir
# lib/my_iot_device/watchdog.ex
defmodule MyIoTDevice.Watchdog do
  @moduledoc """
  Watchdog pattern - รีสตาร์ท system เมื่อ detect ปัญหา
  """

  use GenServer
  require Logger

  @check_interval 30_000  # ตรวจสอบทุก 30 วินาที
  @max_failures 3

  def start_link(opts \\ []) do
    GenServer.start_link(__MODULE__, opts, name: __MODULE__)
  end

  @impl true
  def init(_opts) do
    Process.send_after(self(), :check_health, @check_interval)
    {:ok, %{failures: 0, last_check: DateTime.utc_now()}}
  end

  @impl true
  def handle_info(:check_health, state) do
    health_status = check_all_components()

    new_state = case health_status do
      :healthy ->
        Logger.debug("Health check: OK")
        %{state | failures: 0, last_check: DateTime.utc_now()}

      {:unhealthy, reasons} ->
        Logger.error("Health check failed: #{inspect(reasons)}")
        failures = state.failures + 1

        if failures >= @max_failures do
          Logger.error("Too many failures, initiating system reboot...")
          Nerves.Runtime.reboot()
        end

        %{state | failures: failures}
    end

    Process.send_after(self(), :check_health, @check_interval)
    {:noreply, new_state}
  end

  defp check_all_components do
    checks = [
      check_sensor_reader(),
      check_data_reporter(),
      check_network_connectivity()
    ]

    failures = Enum.reject(checks, &(&1 == :ok))

    case failures do
      [] -> :healthy
      reasons -> {:unhealthy, reasons}
    end
  end

  defp check_sensor_reader do
    case Process.whereis(MyIoTDevice.SensorReader) do
      nil -> {:error, :sensor_reader_dead}
      _pid ->
        case MyIoTDevice.SensorReader.get_reading() do
          nil -> {:error, :no_sensor_reading}
          _reading -> :ok
        end
    end
  end

  defp check_data_reporter do
    case Process.whereis(MyIoTDevice.DataReporter) do
      nil -> {:error, :data_reporter_dead}
      _pid -> :ok
    end
  end

  defp check_network_connectivity do
    case :gen_tcp.connect(~c"8.8.8.8", 53, [], 3_000) do
      {:ok, socket} ->
        :gen_tcp.close(socket)
        :ok
      {:error, _} ->
        {:error, :no_network}
    end
  end
end
```

```elixir
# lib/my_iot_device/application.ex
defmodule MyIoTDevice.Application do
  use Application

  @impl true
  def start(_type, _args) do
    children = [
      # Core OTP infrastructure
      {Registry, keys: :unique, name: MyIoTDevice.Registry},

      # Hardware controllers
      MyIoTDevice.GpioController,
      MyIoTDevice.SensorReader,

      # Network and reporting
      {Finch, name: MyFinch},
      MyIoTDevice.DataReporter,

      # Monitoring
      MyIoTDevice.Watchdog,
    ]

    opts = [strategy: :one_for_one, name: MyIoTDevice.Supervisor]
    Supervisor.start_link(children, opts)
  end
end
```

---

## 9. Over-the-Air (OTA) Updates

```elixir
# lib/my_iot_device/ota_updater.ex
defmodule MyIoTDevice.OtaUpdater do
  @moduledoc """
  ตรวจสอบและติดตั้ง firmware updates ผ่าน network
  """

  use GenServer
  require Logger

  @check_interval 60 * 60 * 1_000  # ตรวจสอบทุกชั่วโมง
  @update_server Application.compile_env(:my_iot_device, :update_server)

  def start_link(opts \\ []) do
    GenServer.start_link(__MODULE__, opts, name: __MODULE__)
  end

  @impl true
  def init(_opts) do
    Process.send_after(self(), :check_for_update, 60_000)
    {:ok, %{current_version: current_firmware_version()}}
  end

  @impl true
  def handle_info(:check_for_update, state) do
    case check_for_update(state.current_version) do
      {:update_available, url, version} ->
        Logger.info("New firmware available: #{version}")
        download_and_apply_update(url, version)

      :up_to_date ->
        Logger.debug("Firmware is up to date: #{state.current_version}")
    end

    Process.send_after(self(), :check_for_update, @check_interval)
    {:noreply, state}
  end

  defp check_for_update(current_version) do
    device_id = Application.get_env(:my_iot_device, :device_id)
    url = "#{@update_server}/api/firmware/check"

    request = Finch.build(:get, url, [
      {"X-Device-ID", device_id},
      {"X-Current-Version", current_version}
    ])

    case Finch.request(request, MyFinch) do
      {:ok, %{status: 200, body: body}} ->
        info = Jason.decode!(body, keys: :atoms)
        if info.has_update do
          {:update_available, info.download_url, info.version}
        else
          :up_to_date
        end

      _ ->
        :up_to_date
    end
  end

  defp download_and_apply_update(url, version) do
    Logger.info("Downloading firmware #{version}...")

    fw_path = "/tmp/update.fw"

    case download_file(url, fw_path) do
      :ok ->
        Logger.info("Applying firmware update...")
        case Nerves.Runtime.KV.put("nerves_fw_active", fw_path) do
          :ok ->
            Logger.info("Update applied! Rebooting...")
            Nerves.Runtime.reboot()
          {:error, reason} ->
            Logger.error("Failed to apply update: #{inspect(reason)}")
        end

      {:error, reason} ->
        Logger.error("Failed to download update: #{inspect(reason)}")
    end
  end

  defp download_file(url, destination) do
    request = Finch.build(:get, url)

    case Finch.request(request, MyFinch, receive_timeout: 120_000) do
      {:ok, %{status: 200, body: body}} ->
        File.write(destination, body)
      _ ->
        {:error, :download_failed}
    end
  end

  defp current_firmware_version do
    case Nerves.Runtime.KV.get("nerves_fw_version") do
      {:ok, version} -> version
      _ -> "unknown"
    end
  end
end
```

---

## 10. Testing Nerves Applications

```elixir
# test/gpio_controller_test.exs
defmodule MyIoTDevice.GpioControllerTest do
  use ExUnit.Case

  # ใน Host mode, mock Circuits.GPIO
  setup do
    # Stub hardware สำหรับ testing บน host machine
    Mox.defmock(MockGPIO, for: Circuits.GPIO.Behaviour)

    MockGPIO
    |> Mox.expect(:open, 2, fn _pin, _direction, _opts -> {:ok, :mock_gpio_ref} end)
    |> Mox.expect(:write, fn _ref, _value -> :ok end)
    |> Mox.expect(:set_interrupts, fn _ref, _mode -> :ok end)

    :ok
  end

  test "led_on/0 ส่ง 1 ไปยัง GPIO" do
    # ทดสอบ logic โดยไม่ต้องมี hardware จริง
    assert :ok = MyIoTDevice.GpioController.led_on()
    Mox.verify!()
  end
end
```

---

## สรุป

```
Nerves IoT Architecture:
                     
┌────────────────────────────────────────────┐
│          Cloud (Phoenix Server)            │
│  ┌────────────┐    ┌───────────────────┐   │
│  │  REST API  │    │  LiveView Dashboard│  │
│  │  /sensor   │    │  (Real-time data) │   │
│  └────────────┘    └───────────────────┘   │
└──────────────┬─────────────────────────────┘
               │ HTTPS
┌──────────────┴─────────────────────────────┐
│          Nerves Device (Raspberry Pi)      │
│  ┌──────────────┐    ┌──────────────────┐  │
│  │SensorReader  │    │ DataReporter     │  │
│  │ (I2C/SPI)   │───►│ (HTTP/WebSocket) │  │
│  └──────────────┘    └──────────────────┘  │
│  ┌──────────────┐    ┌──────────────────┐  │
│  │GpioController│    │   Watchdog       │  │
│  │ (LED/Button) │    │  (Health check)  │  │
│  └──────────────┘    └──────────────────┘  │
└────────────────────────────────────────────┘

Key Concepts:
- Circuits.GPIO/I2C/SPI: hardware interfaces
- Finch: HTTP client สำหรับ report data
- Watchdog: detect และ recover จากปัญหา
- OTA Updates: update firmware ผ่าน network
- WasmEx: รัน WASM modules ใน Elixir
```

---

*ก่อนหน้า: [Part 60 - Machine Learning Integration](part_60.md) | ต่อไป: [Part 62 - Phoenix Livebook](part_62.md)*
