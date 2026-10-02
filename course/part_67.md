# Part 67: Internationalization (i18n) and Localization (l10n)

## เป้าหมายการเรียนรู้

- ใช้ Gettext สำหรับแปลภาษาใน Phoenix application
- ตรวจจับ locale จาก Accept-Language header อัตโนมัติ
- จัดรูปแบบวันที่/เวลาตาม locale
- แสดงราคาและสกุลเงินอย่างถูกต้องในแต่ละภูมิภาค
- รองรับภาษา RTL (Right-to-Left) เช่น Arabic, Hebrew
- เก็บและดึงการแปลจาก database แบบ dynamic
- วาง workflow สำหรับทีมแปลภาษา

---

## 1. ตั้งค่า Gettext

Gettext เป็น standard สากลสำหรับ internationalization ที่ Phoenix ใช้ในตัว

### 1.1 โครงสร้างไฟล์

```
priv/
  gettext/
    en/
      LC_MESSAGES/
        default.po
        errors.po
    th/
      LC_MESSAGES/
        default.po
        errors.po
    ja/
      LC_MESSAGES/
        default.po
    ar/
      LC_MESSAGES/
        default.po
```

### 1.2 Gettext Backend Module

```elixir
# lib/my_app_web/gettext.ex
defmodule MyAppWeb.Gettext do
  use Gettext.Backend, otp_app: :my_app

  @locales ~w(en th ja zh ar)

  def supported_locales, do: @locales

  def locale_name("en"), do: "English"
  def locale_name("th"), do: "ภาษาไทย"
  def locale_name("ja"), do: "日本語"
  def locale_name("zh"), do: "中文"
  def locale_name("ar"), do: "العربية"
  def locale_name(_), do: "Unknown"

  def rtl_locales, do: ~w(ar he fa ur)
  def rtl?(locale), do: locale in rtl_locales()
end
```

### 1.3 ไฟล์ .po (Translation Catalog)

```po
# priv/gettext/th/LC_MESSAGES/default.po
msgid ""
msgstr ""
"Language: th\n"
"Content-Type: text/plain; charset=UTF-8\n"

msgid "Welcome, %{name}!"
msgstr "ยินดีต้อนรับ, %{name}!"

msgid "You have %{count} item in cart"
msgid_plural "You have %{count} items in cart"
msgstr[0] "คุณมี %{count} รายการในตะกร้า"

msgid "Sign in"
msgstr "เข้าสู่ระบบ"

msgid "Sign out"
msgstr "ออกจากระบบ"

msgid "Create account"
msgstr "สร้างบัญชีใหม่"

msgid "Search products..."
msgstr "ค้นหาสินค้า..."

msgid "Add to cart"
msgstr "เพิ่มลงตะกร้า"

msgid "Product not found"
msgstr "ไม่พบสินค้า"

msgid "Order confirmed"
msgstr "ยืนยันคำสั่งซื้อแล้ว"

msgid "Payment failed"
msgstr "การชำระเงินล้มเหลว"
```

```po
# priv/gettext/th/LC_MESSAGES/errors.po
msgid ""
msgstr ""
"Language: th\n"

msgid "can't be blank"
msgstr "ไม่สามารถเว้นว่างได้"

msgid "has already been taken"
msgstr "ถูกใช้ไปแล้ว"

msgid "is invalid"
msgstr "ไม่ถูกต้อง"

msgid "should be at most %{count} character(s)"
msgid_plural "should be at most %{count} character(s)"
msgstr[0] "ต้องมีความยาวไม่เกิน %{count} ตัวอักษร"

msgid "should be at least %{count} character(s)"
msgid_plural "should be at least %{count} character(s)"
msgstr[0] "ต้องมีความยาวอย่างน้อย %{count} ตัวอักษร"
```

### 1.4 ใช้งาน Gettext ใน Code

```heex
<%# lib/my_app_web/components/layouts/app.html.heex %>
<nav>
  <span><%= gettext("Welcome, %{name}!", name: @current_user.name) %></span>
  <a href="/cart">
    <%= ngettext(
      "You have %{count} item in cart",
      "You have %{count} items in cart",
      @cart_count,
      count: @cart_count
    ) %>
  </a>
</nav>
```

```elixir
# ใน LiveView
defmodule MyAppWeb.ProductLive.Show do
  use MyAppWeb, :live_view
  import MyAppWeb.Gettext

  def handle_event("add_to_cart", %{"id" => id}, socket) do
    case Cart.add_item(socket.assigns.cart, id) do
      {:ok, cart} ->
        {:noreply,
         socket
         |> put_flash(:info, gettext("Added to cart"))
         |> assign(:cart, cart)}
      {:error, :out_of_stock} ->
        {:noreply, put_flash(socket, :error, gettext("Product not found"))}
    end
  end
end
```

---

## 2. Locale Detection Plug

### 2.1 SetLocale Plug

```elixir
defmodule MyAppWeb.Plugs.SetLocale do
  import Plug.Conn

  @supported_locales MyAppWeb.Gettext.supported_locales()
  @default_locale "th"

  def init(opts), do: opts

  def call(conn, _opts) do
    locale = determine_locale(conn)

    Gettext.put_locale(MyAppWeb.Gettext, locale)

    conn
    |> assign(:locale, locale)
    |> assign(:rtl, MyAppWeb.Gettext.rtl?(locale))
    |> put_resp_header("content-language", locale)
  end

  defp determine_locale(conn) do
    locale =
      get_from_params(conn) ||
      get_from_cookie(conn) ||
      get_from_accept_language(conn) ||
      @default_locale

    if locale in @supported_locales, do: locale, else: @default_locale
  end

  defp get_from_params(conn), do: conn.params["locale"]

  defp get_from_cookie(conn), do: conn.cookies["locale"]

  defp get_from_accept_language(conn) do
    conn
    |> get_req_header("accept-language")
    |> List.first()
    |> parse_accept_language()
  end

  # Parse "th-TH,th;q=0.9,en-US;q=0.8,en;q=0.7"
  defp parse_accept_language(nil), do: nil
  defp parse_accept_language(header) do
    header
    |> String.split(",")
    |> Enum.map(&parse_locale_quality/1)
    |> Enum.sort_by(fn {_, q} -> -q end)
    |> Enum.find_value(fn {lang, _} ->
      normalized = lang |> String.downcase() |> String.slice(0, 2)
      if normalized in @supported_locales, do: normalized
    end)
  end

  defp parse_locale_quality(part) do
    case String.split(String.trim(part), ";q=") do
      [lang] -> {String.trim(lang), 1.0}
      [lang, q] -> {String.trim(lang), String.to_float(q)}
    end
  end
end
```

### 2.2 Language Switcher Controller

```elixir
defmodule MyAppWeb.LocaleController do
  use MyAppWeb, :controller

  @supported_locales MyAppWeb.Gettext.supported_locales()

  def update(conn, %{"locale" => locale, "return" => return_to}) do
    if locale in @supported_locales do
      conn
      |> put_resp_cookie("locale", locale, max_age: 365 * 24 * 3600, same_site: "Lax")
      |> redirect(to: sanitize_redirect(return_to))
    else
      conn |> put_status(400) |> json(%{error: "Invalid locale"})
    end
  end

  defp sanitize_redirect(path) do
    if String.starts_with?(path, "/"), do: path, else: "/"
  end
end
```

### 2.3 Language Switcher Component

```elixir
defmodule MyAppWeb.Components.LanguageSwitcher do
  use MyAppWeb, :html

  def language_switcher(assigns) do
    ~H"""
    <div class="relative" x-data="{ open: false }">
      <button @click="open = !open"
        class="flex items-center gap-2 px-3 py-2 rounded-md hover:bg-gray-100">
        <span><%= locale_flag(@current_locale) %></span>
        <span class="text-sm"><%= MyAppWeb.Gettext.locale_name(@current_locale) %></span>
        <svg class="w-4 h-4" fill="none" stroke="currentColor" viewBox="0 0 24 24">
          <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M19 9l-7 7-7-7"/>
        </svg>
      </button>
      <div x-show="open" x-transition
        class="absolute right-0 mt-1 w-40 bg-white rounded-lg shadow-lg border z-50">
        <%= for locale <- MyAppWeb.Gettext.supported_locales() do %>
          <a href={~p"/settings/locale?locale=#{locale}&return=#{@current_path}"}
            class={"flex items-center gap-2 px-4 py-2 hover:bg-gray-50 text-sm #{if locale == @current_locale, do: "bg-blue-50 text-blue-700"}"}>
            <span><%= locale_flag(locale) %></span>
            <span><%= MyAppWeb.Gettext.locale_name(locale) %></span>
          </a>
        <% end %>
      </div>
    </div>
    """
  end

  defp locale_flag("en"), do: "🇺🇸"
  defp locale_flag("th"), do: "🇹🇭"
  defp locale_flag("ja"), do: "🇯🇵"
  defp locale_flag("zh"), do: "🇨🇳"
  defp locale_flag("ar"), do: "🇸🇦"
  defp locale_flag(_), do: "🌐"
end
```

---

## 3. Date/Time Localization

### 3.1 ตั้งค่า Timezone

```elixir
# mix.exs
defp deps do
  [
    {:tzdata, "~> 1.1"},
    {:timex, "~> 3.7"},
  ]
end
```

```elixir
# config/config.exs
config :elixir, :time_zone_database, Tzdata.TimeZoneDatabase
```

### 3.2 DateTime Formatter

```elixir
defmodule MyApp.Localization.DateTimeFormatter do
  @locale_configs %{
    "th" => %{
      timezone: "Asia/Bangkok",
      date_format: "{D}/{M}/{YYYY}",
      time_format: "{h24}:{m}"
    },
    "en" => %{
      timezone: "America/New_York",
      date_format: "{M}/{D}/{YYYY}",
      time_format: "{h12}:{m} {AM}"
    },
    "ja" => %{
      timezone: "Asia/Tokyo",
      date_format: "{YYYY}年{M}月{D}日",
      time_format: "{h24}:{m}"
    },
    "ar" => %{
      timezone: "Asia/Riyadh",
      date_format: "{D}/{M}/{YYYY}",
      time_format: "{h24}:{m}"
    }
  }

  def format_date(datetime, locale, user_timezone \\ nil) do
    config = Map.get(@locale_configs, locale, @locale_configs["en"])
    tz = user_timezone || config.timezone

    datetime
    |> DateTime.shift_zone!(tz)
    |> Timex.format!(config.date_format)
  end

  def format_time(datetime, locale, user_timezone \\ nil) do
    config = Map.get(@locale_configs, locale, @locale_configs["en"])
    tz = user_timezone || config.timezone

    datetime
    |> DateTime.shift_zone!(tz)
    |> Timex.format!(config.time_format)
  end

  def format_relative(datetime, locale) do
    diff = DateTime.diff(DateTime.utc_now(), datetime, :second)

    case locale do
      "th" -> format_relative_th(diff)
      "en" -> format_relative_en(diff)
      "ja" -> format_relative_ja(diff)
      _ -> format_relative_en(diff)
    end
  end

  defp format_relative_th(diff) do
    cond do
      diff < 60 -> "เมื่อสักครู่"
      diff < 3600 -> "#{div(diff, 60)} นาทีที่แล้ว"
      diff < 86400 -> "#{div(diff, 3600)} ชั่วโมงที่แล้ว"
      diff < 604800 -> "#{div(diff, 86400)} วันที่แล้ว"
      diff < 2592000 -> "#{div(diff, 604800)} สัปดาห์ที่แล้ว"
      diff < 31536000 -> "#{div(diff, 2592000)} เดือนที่แล้ว"
      true -> "#{div(diff, 31536000)} ปีที่แล้ว"
    end
  end

  defp format_relative_en(diff) do
    cond do
      diff < 60 -> "just now"
      diff < 3600 -> "#{div(diff, 60)} minutes ago"
      diff < 86400 -> "#{div(diff, 3600)} hours ago"
      diff < 604800 -> "#{div(diff, 86400)} days ago"
      true -> "#{div(diff, 604800)} weeks ago"
    end
  end

  defp format_relative_ja(diff) do
    cond do
      diff < 60 -> "たった今"
      diff < 3600 -> "#{div(diff, 60)}分前"
      diff < 86400 -> "#{div(diff, 3600)}時間前"
      diff < 604800 -> "#{div(diff, 86400)}日前"
      true -> "#{div(diff, 604800)}週間前"
    end
  end
end
```

---

## 4. Currency Formatting

```elixir
defmodule MyApp.Localization.Currency do
  @configs %{
    "th" => %{code: "THB", symbol: "฿", position: :before, separator: ",", decimal: ".", decimals: 2},
    "en" => %{code: "USD", symbol: "$", position: :before, separator: ",", decimal: ".", decimals: 2},
    "ja" => %{code: "JPY", symbol: "¥", position: :before, separator: ",", decimal: ".", decimals: 0},
    "zh" => %{code: "CNY", symbol: "¥", position: :before, separator: ",", decimal: ".", decimals: 2},
    "ar" => %{code: "SAR", symbol: "﷼", position: :after, separator: ",", decimal: ".", decimals: 2}
  }

  def format(amount, locale, override_code \\ nil) do
    config = Map.get(@configs, locale, @configs["en"])
    symbol = if override_code, do: override_code, else: config.symbol

    decimal_amount =
      case amount do
        %Decimal{} -> amount
        v -> Decimal.new(to_string(v))
      end

    formatted = format_number(decimal_amount, config)

    case config.position do
      :before -> "#{symbol}#{formatted}"
      :after -> "#{formatted} #{symbol}"
    end
  end

  defp format_number(decimal, config) do
    rounded = Decimal.round(decimal, config.decimals)
    str = Decimal.to_string(rounded, :normal)
    [int_part | dec_parts] = String.split(str, ".")

    formatted_int =
      int_part
      |> String.graphemes()
      |> Enum.reverse()
      |> Enum.chunk_every(3)
      |> Enum.join(config.separator)
      |> String.graphemes()
      |> Enum.reverse()
      |> Enum.join()

    if config.decimals > 0 do
      dec_str = List.first(dec_parts) || "0"
      padded = String.pad_trailing(dec_str, config.decimals, "0")
      "#{formatted_int}#{config.decimal}#{padded}"
    else
      formatted_int
    end
  end
end
```

---

## 5. RTL Language Support

### 5.1 Layout สำหรับ RTL

```heex
<%# lib/my_app_web/components/layouts/root.html.heex %>
<!DOCTYPE html>
<html
  lang={@locale}
  dir={if @rtl, do: "rtl", else: "ltr"}
  class={[@rtl && "rtl"]}
>
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1" />
  <link phx-track-static rel="stylesheet" href={~p"/assets/app.css"} />
  <%= if @rtl do %>
    <link rel="stylesheet" href={~p"/assets/rtl.css"} />
  <% end %>
</head>
<body class={[@rtl && "font-arabic"]}>
  <%= @inner_content %>
</body>
</html>
```

### 5.2 CSS สำหรับ RTL

```css
/* assets/css/rtl.css */
.rtl .ml-auto { margin-left: 0; margin-right: auto; }
.rtl .mr-auto { margin-right: 0; margin-left: auto; }
.rtl .pl-4 { padding-left: 0; padding-right: 1rem; }
.rtl .pr-4 { padding-right: 0; padding-left: 1rem; }
.rtl .text-left { text-align: right; }
.rtl .text-right { text-align: left; }
.rtl input, .rtl textarea { text-align: right; direction: rtl; }
.rtl .arrow-icon { transform: scaleX(-1); }
.font-arabic { font-family: 'Noto Naskh Arabic', 'Arial', sans-serif; }

/* Flex direction adjustments */
.rtl .flex-row { flex-direction: row-reverse; }
.rtl nav .flex { flex-direction: row-reverse; }
```

---

## 6. Dynamic Translations จาก Database

### 6.1 Database Schema

```elixir
defmodule MyApp.Repo.Migrations.CreateTranslations do
  use Ecto.Migration

  def change do
    create table(:translations) do
      add :locale, :string, null: false
      add :namespace, :string, null: false, default: "default"
      add :key, :string, null: false
      add :value, :text, null: false
      timestamps()
    end

    create unique_index(:translations, [:locale, :namespace, :key])
    create index(:translations, [:locale])
  end
end
```

### 6.2 Translation Cache (GenServer + ETS)

```elixir
defmodule MyApp.Translations do
  use GenServer
  alias MyApp.Repo
  import Ecto.Query
  require Logger

  @table :translations_cache
  @refresh_interval :timer.minutes(15)

  def start_link(_opts) do
    GenServer.start_link(__MODULE__, [], name: __MODULE__)
  end

  def translate(key, locale, namespace \\ "default", default \\ nil) do
    case :ets.lookup(@table, {locale, namespace, key}) do
      [{_, value}] ->
        value
      [] ->
        # Fallback: English
        case :ets.lookup(@table, {"en", namespace, key}) do
          [{_, value}] -> value
          [] -> default || key
        end
    end
  end

  def upsert(locale, namespace, key, value) do
    now = DateTime.utc_now() |> DateTime.truncate(:second)
    Repo.insert_all("translations",
      [%{locale: locale, namespace: namespace, key: key, value: value,
         inserted_at: now, updated_at: now}],
      on_conflict: {:replace, [:value, :updated_at]},
      conflict_target: [:locale, :namespace, :key]
    )
    :ets.insert(@table, {{locale, namespace, key}, value})
    :ok
  end

  def refresh_cache, do: GenServer.cast(__MODULE__, :refresh)

  @impl GenServer
  def init(_opts) do
    :ets.new(@table, [:named_table, :public, read_concurrency: true])
    load_all()
    schedule_refresh()
    {:ok, %{}}
  end

  @impl GenServer
  def handle_cast(:refresh, state) do
    load_all()
    {:noreply, state}
  end

  @impl GenServer
  def handle_info(:refresh, state) do
    load_all()
    schedule_refresh()
    {:noreply, state}
  end

  defp load_all do
    from(t in "translations", select: {t.locale, t.namespace, t.key, t.value})
    |> Repo.all()
    |> Enum.each(fn {locale, ns, key, value} ->
      :ets.insert(@table, {{locale, ns, key}, value})
    end)
  end

  defp schedule_refresh, do: Process.send_after(self(), :refresh, @refresh_interval)
end
```

---

## 7. Translation Admin Panel

```elixir
defmodule MyAppWeb.Admin.TranslationLive do
  use MyAppWeb, :live_view
  alias MyApp.Translations

  def mount(params, _session, socket) do
    locale = params["locale"] || "th"
    namespace = params["namespace"] || "default"

    {:ok, assign(socket,
      locale: locale,
      namespace: namespace,
      translations: list_translations(locale, namespace),
      filter: "",
      editing_key: nil
    )}
  end

  def handle_event("save", %{"key" => key, "value" => value}, socket) do
    :ok = Translations.upsert(socket.assigns.locale, socket.assigns.namespace, key, value)
    translations = list_translations(socket.assigns.locale, socket.assigns.namespace)
    {:noreply, assign(socket, translations: translations, editing_key: nil)}
  end

  def handle_event("start_edit", %{"key" => key}, socket) do
    {:noreply, assign(socket, editing_key: key)}
  end

  def handle_event("cancel_edit", _params, socket) do
    {:noreply, assign(socket, editing_key: nil)}
  end

  def handle_event("filter", %{"value" => filter}, socket) do
    {:noreply, assign(socket, filter: filter)}
  end

  def render(assigns) do
    ~H"""
    <div class="p-6 max-w-5xl mx-auto">
      <div class="flex items-center justify-between mb-6">
        <h1 class="text-2xl font-bold">จัดการการแปลภาษา</h1>
        <select phx-change="change_locale" name="locale" class="select select-bordered">
          <%= for locale <- MyAppWeb.Gettext.supported_locales() do %>
            <option value={locale} selected={locale == @locale}>
              <%= MyAppWeb.Gettext.locale_name(locale) %>
            </option>
          <% end %>
        </select>
      </div>

      <input type="search" placeholder="ค้นหา key..."
        phx-keyup="filter" phx-value-value=""
        class="input input-bordered w-full max-w-sm mb-4" />

      <div class="overflow-x-auto">
        <table class="table w-full bg-white">
          <thead>
            <tr>
              <th class="w-1/3">Key</th>
              <th class="w-1/2">Translation</th>
              <th>Updated</th>
              <th></th>
            </tr>
          </thead>
          <tbody>
            <%= for t <- filter_translations(@translations, @filter) do %>
              <tr>
                <td class="font-mono text-xs text-gray-600"><%= t.key %></td>
                <td>
                  <%= if @editing_key == t.key do %>
                    <form phx-submit="save">
                      <input type="hidden" name="key" value={t.key} />
                      <textarea name="value" class="textarea textarea-bordered w-full text-sm"><%= t.value %></textarea>
                      <div class="flex gap-2 mt-1">
                        <button type="submit" class="btn btn-xs btn-primary">บันทึก</button>
                        <button type="button" phx-click="cancel_edit" class="btn btn-xs">ยกเลิก</button>
                      </div>
                    </form>
                  <% else %>
                    <span class="text-sm"><%= t.value %></span>
                  <% end %>
                </td>
                <td class="text-xs text-gray-400">
                  <%= Calendar.strftime(t.updated_at, "%d/%m/%Y") %>
                </td>
                <td>
                  <button phx-click="start_edit" phx-value-key={t.key}
                    class="btn btn-xs btn-outline">แก้ไข</button>
                </td>
              </tr>
            <% end %>
          </tbody>
        </table>
      </div>
    </div>
    """
  end

  defp filter_translations(translations, ""), do: translations
  defp filter_translations(translations, filter) do
    Enum.filter(translations, fn t ->
      String.contains?(String.downcase(t.key), String.downcase(filter)) ||
        String.contains?(String.downcase(t.value || ""), String.downcase(filter))
    end)
  end

  defp list_translations(locale, namespace) do
    import Ecto.Query
    MyApp.Repo.all(
      from t in "translations",
      where: t.locale == ^locale and t.namespace == ^namespace,
      order_by: t.key,
      select: %{key: t.key, value: t.value, updated_at: t.updated_at}
    )
  end
end
```

---

## 8. Translatable Content ใน Database (Ecto)

```elixir
# สำหรับ content ที่ต้องแปล เช่น product name, description
defmodule MyApp.Repo.Migrations.CreateProductTranslations do
  use Ecto.Migration

  def change do
    create table(:product_translations) do
      add :product_id, references(:products, on_delete: :delete_all), null: false
      add :locale, :string, null: false
      add :name, :string
      add :description, :text
      add :slug, :string
      timestamps()
    end

    create unique_index(:product_translations, [:product_id, :locale])
    create index(:product_translations, [:locale])
  end
end
```

```elixir
defmodule MyApp.Catalog.Product do
  use Ecto.Schema

  schema "products" do
    field :base_name, :string
    field :base_description, :text
    has_many :translations, MyApp.Catalog.ProductTranslation
    timestamps()
  end

  def translated(product, locale) do
    case Enum.find(product.translations, &(&1.locale == locale)) do
      nil -> %{name: product.base_name, description: product.base_description}
      t -> %{name: t.name || product.base_name, description: t.description || product.base_description}
    end
  end
end
```

---

## สรุป

```
i18n / l10n Architecture
═════════════════════════════════════════════════════════
Request Flow
  Browser Accept-Language header
    → URL param (?locale=th)
    → Cookie (user preference)
    → SetLocale Plug → Gettext.put_locale()

Translation Sources (priority order)
  1. Database translations (dynamic, admin-editable)
  2. .po files (static, version-controlled)
  3. Key itself (fallback)

Formatting by Locale
  ┌─────────────────────────────────────────┐
  │  Date/Time   Timex + timezone-aware     │
  │  Currency    Decimal + symbol/position  │
  │  Numbers     Custom formatter (,/.)     │
  │  RTL         dir="rtl" + CSS overrides  │
  └─────────────────────────────────────────┘

Content Translation Strategies
  Static UI text     → Gettext (.po files, version controlled)
  Dynamic content    → translations table (ETS cache)
  Product/CMS data   → entity_translations table

Translation Workflow
  Developer:   mix gettext.extract --merge (sync .pot)
  Translator:  Edit .po files in Poedit/POEditor
  Admin:       Update via translation admin panel
  Cache:       Auto-refresh every 15 min (GenServer)

Commands
  # Extract all strings from code
  mix gettext.extract --merge

  # Compile translations
  mix compile

  # Check missing translations
  mix gettext.extract
═════════════════════════════════════════════════════════
```

---

*ก่อนหน้า: [Part 66 - Search System](part_66.md) | ต่อไป: [Part 68 - Content Management System](part_68.md)*
