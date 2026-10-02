# Part 20: ExUnit: Testing Framework

## เป้าหมายการเรียนรู้

หลังจากจบ Part นี้ คุณจะสามารถ:
- เขียน tests ด้วย `test`, `describe`, `setup`, `setup_all`
- ใช้ assertions: `assert`, `refute`, `assert_raise`, `assert_receive`
- Mock dependencies ด้วย Mox
- เขียน property-based tests ด้วย StreamData
- สร้าง test suite ครบถ้วนสำหรับ application จริง

---

## 1. ExUnit พื้นฐาน

```elixir
# test/my_app_test.exs
defmodule MyAppTest do
  use ExUnit.Case

  test "the truth" do
    assert 1 + 1 == 2
  end

  test "string operations" do
    assert String.length("hello") == 5
    assert String.upcase("hello") == "HELLO"
    assert String.contains?("hello world", "world")
  end

  test "list operations" do
    list = [1, 2, 3, 4, 5]
    assert length(list) == 5
    assert hd(list) == 1
    assert Enum.sum(list) == 15
  end
end
```

---

## 2. assert และ refute

```elixir
defmodule AssertionsTest do
  use ExUnit.Case

  # assert - ต้องเป็น true
  test "assert examples" do
    assert true
    assert 2 + 2 == 4
    assert "hello" =~ "hell"     # regex/substring match
    assert [1, 2, 3] == [1, 2, 3]
    assert %{a: 1} = %{a: 1, b: 2}  # pattern match (left is subset)
  end

  # refute - ต้องเป็น false
  test "refute examples" do
    refute false
    refute 1 > 2
    refute String.starts_with?("hello", "world")
  end

  # assert_raise
  test "exception assertions" do
    assert_raise ArithmeticError, fn ->
      1 / 0
    end

    assert_raise ArgumentError, "invalid argument", fn ->
      raise ArgumentError, "invalid argument"
    end

    # ตรวจสอบ exception message
    error = assert_raise RuntimeError, fn ->
      raise "something went wrong"
    end
    assert error.message == "something went wrong"
  end

  # assert_receive / refute_receive
  test "message assertions" do
    send(self(), {:hello, "world"})

    assert_receive {:hello, name}
    assert name == "world"

    # ด้วย timeout
    assert_receive :some_message, 500

    refute_receive :unexpected_message
  end

  # เปรียบเทียบ float
  test "float comparison" do
    assert_in_delta 3.14, :math.pi(), 0.01
    assert_in_delta 1.0, 1.0000001, 0.0001
  end
end
```

---

## 3. describe และ context organization

```elixir
defmodule UserTest do
  use ExUnit.Case

  describe "User.create/1" do
    test "creates user with valid params" do
      params = %{name: "Alice", email: "alice@example.com", age: 25}
      assert {:ok, user} = User.create(params)
      assert user.name == "Alice"
      assert user.email == "alice@example.com"
    end

    test "returns error for missing name" do
      params = %{email: "alice@example.com"}
      assert {:error, changeset} = User.create(params)
      assert "can't be blank" in errors_on(changeset).name
    end

    test "returns error for invalid email" do
      params = %{name: "Alice", email: "not-an-email"}
      assert {:error, changeset} = User.create(params)
      assert "has invalid format" in errors_on(changeset).email
    end

    test "returns error for duplicate email" do
      params = %{name: "Alice", email: "alice@example.com"}
      {:ok, _} = User.create(params)
      assert {:error, changeset} = User.create(params)
      assert "has already been taken" in errors_on(changeset).email
    end
  end

  describe "User.authenticate/2" do
    test "authenticates with valid credentials" do
      {:ok, user} = User.create(%{
        name: "Alice",
        email: "alice@example.com",
        password: "secret123"
      })
      assert {:ok, authenticated} = User.authenticate("alice@example.com", "secret123")
      assert authenticated.id == user.id
    end

    test "returns error for wrong password" do
      User.create(%{name: "Alice", email: "alice@example.com", password: "correct"})
      assert {:error, :invalid_credentials} = User.authenticate("alice@example.com", "wrong")
    end

    test "returns error for unknown email" do
      assert {:error, :invalid_credentials} = User.authenticate("unknown@example.com", "any")
    end
  end

  defp errors_on(changeset) do
    Ecto.Changeset.traverse_errors(changeset, fn {msg, opts} ->
      Regex.replace(~r"%{(\w+)}", msg, fn _, key ->
        opts |> Keyword.get(String.to_existing_atom(key), key) |> to_string()
      end)
    end)
  end
end
```

---

## 4. setup และ setup_all

```elixir
defmodule ProductTest do
  use ExUnit.Case

  # setup_all รันครั้งเดียวสำหรับทุก tests ใน module
  setup_all do
    IO.puts("Setting up test module...")
    categories = [:electronics, :clothing, :food]
    %{categories: categories}
  end

  # setup รันก่อนแต่ละ test
  setup do
    # Cleanup before each test
    :ok = Application.ensure_started(:my_app)

    # Return context map
    product = %{
      id: 1,
      name: "Test Product",
      price: 99.99,
      stock: 10
    }

    %{product: product}
  end

  # รับ context ผ่าน parameter
  test "product has correct attributes", %{product: product} do
    assert product.name == "Test Product"
    assert product.price == 99.99
  end

  test "product stock is positive", %{product: product} do
    assert product.stock > 0
  end

  # รับทั้ง setup และ setup_all context
  test "product has valid category", %{product: product, categories: categories} do
    # ตัวอย่าง: ตรวจสอบ product category ต้องอยู่ใน list
    assert product[:category] in categories or product[:category] == nil
  end
end
```

### setup ด้วย Ecto Sandbox

```elixir
defmodule MyApp.AccountsTest do
  use ExUnit.Case
  use MyApp.DataCase  # customized case

  # DataCase ที่กำหนดเอง:
  # test/support/data_case.ex
  defmodule MyApp.DataCase do
    use ExUnit.CaseTemplate

    using do
      quote do
        import Ecto
        import Ecto.Query
        import MyApp.DataCase

        alias MyApp.Repo
      end
    end

    setup tags do
      :ok = Ecto.Adapters.SQL.Sandbox.checkout(MyApp.Repo)

      unless tags[:async] do
        Ecto.Adapters.SQL.Sandbox.mode(MyApp.Repo, {:shared, self()})
      end

      :ok
    end

    def errors_on(changeset) do
      Ecto.Changeset.traverse_errors(changeset, fn {msg, opts} ->
        Enum.reduce(opts, msg, fn {key, value}, acc ->
          String.replace(acc, "%{#{key}}", to_string(value))
        end)
      end)
    end
  end
end
```

---

## 5. Tags

```elixir
defmodule SlowTest do
  use ExUnit.Case

  # Tag ระดับ test
  @tag :slow
  test "this test takes a while" do
    :timer.sleep(5000)
    assert true
  end

  @tag :integration
  @tag :external
  test "calls external service" do
    # ...
  end

  @tag timeout: 30_000
  test "test with custom timeout" do
    # ...
  end
end

# รัน tests เฉพาะ tag
# mix test --only slow
# mix test --exclude integration
# mix test --only "tag1 or tag2"
```

### ExUnit.Case options

```elixir
defmodule AsyncTest do
  # async: true - รัน tests พร้อมกัน (ระวัง shared state!)
  use ExUnit.Case, async: true

  test "this runs in parallel" do
    assert true
  end
end
```

---

## 6. Mocking ด้วย Mox

```elixir
# test/support/mocks.ex
Mox.defmock(MyApp.PaymentGatewayMock, for: Payment.Gateway)
Mox.defmock(MyApp.EmailServiceMock, for: MyApp.EmailService)

# config/test.exs
config :my_app,
  payment_gateway: MyApp.PaymentGatewayMock,
  email_service: MyApp.EmailServiceMock
```

### ใช้ Mock ใน Tests

```elixir
defmodule OrderProcessingTest do
  use ExUnit.Case

  import Mox

  # ต้อง verify ว่า mocks ถูกเรียกตามที่ expect
  setup :verify_on_exit!

  describe "process_order/1" do
    test "successfully processes order" do
      order = %{id: 1, total: 99.99, user_id: 1}

      # Setup mock expectations
      MyApp.PaymentGatewayMock
      |> expect(:charge, fn amount, _currency, _card_info, _opts ->
        assert amount == 99.99
        {:ok, %{transaction_id: "txn_123", status: :success}}
      end)

      MyApp.EmailServiceMock
      |> expect(:send_receipt, fn user_id, order_id ->
        assert user_id == 1
        assert order_id == 1
        :ok
      end)

      assert {:ok, processed_order} = OrderProcessing.process(order)
      assert processed_order.status == :paid
      assert processed_order.transaction_id == "txn_123"
    end

    test "handles payment failure" do
      order = %{id: 2, total: 50.00, user_id: 2}

      MyApp.PaymentGatewayMock
      |> expect(:charge, fn _amount, _currency, _card_info, _opts ->
        {:error, %{code: "card_declined", message: "Your card was declined"}}
      end)

      # Email ไม่ควรถูกส่งถ้า payment ล้มเหลว
      MyApp.EmailServiceMock
      |> expect(:send_receipt, 0, fn _, _ -> :ok end)

      assert {:error, "Payment failed: Your card was declined"} =
        OrderProcessing.process(order)
    end

    test "retries payment on timeout" do
      order = %{id: 3, total: 75.00, user_id: 3}

      MyApp.PaymentGatewayMock
      |> expect(:charge, 2, fn _amount, _currency, _card_info, _opts ->
        {:error, %{code: "timeout", message: "Request timed out"}}
      end)
      |> expect(:charge, fn _amount, _currency, _card_info, _opts ->
        {:ok, %{transaction_id: "txn_retry", status: :success}}
      end)

      assert {:ok, _} = OrderProcessing.process(order, retries: 2)
    end
  end
end
```

### Stub (ไม่ verify call count)

```elixir
test "uses mock without strict verification" do
  stub(MyApp.EmailServiceMock, :send_receipt, fn _user_id, _order_id ->
    :ok
  end)

  # Mock จะ return :ok เสมอ ไม่ว่าจะ call กี่ครั้ง
  assert :ok == MyApp.EmailServiceMock.send_receipt(1, 1)
  assert :ok == MyApp.EmailServiceMock.send_receipt(2, 2)
end
```

---

## 7. Property-based Testing ด้วย StreamData

```elixir
# mix.exs
{:stream_data, "~> 0.5", only: [:dev, :test]}

defmodule PropertyTest do
  use ExUnit.Case
  use ExUnitProperties

  # ทดสอบว่า String.reverse(String.reverse(s)) == s
  property "reversing a string twice returns the original" do
    check all str <- string(:printable) do
      assert String.reverse(String.reverse(str)) == str
    end
  end

  # ทดสอบ integer properties
  property "sum of positive integers is positive" do
    check all a <- positive_integer(),
              b <- positive_integer() do
      assert a + b > 0
      assert a + b >= a
      assert a + b >= b
    end
  end

  # ทดสอบ list properties
  property "Enum.sort is idempotent" do
    check all list <- list_of(integer()) do
      sorted = Enum.sort(list)
      assert Enum.sort(sorted) == sorted
    end
  end

  # Custom generator
  property "valid emails pass validation" do
    check all email <- email_generator() do
      assert MyApp.Validators.valid_email?(email)
    end
  end

  defp email_generator do
    gen all local <- string(:alphanumeric, min_length: 1),
            domain <- string(:alphanumeric, min_length: 2),
            tld <- string(:alphanumeric, min_length: 2, max_length: 4) do
      "#{local}@#{domain}.#{tld}"
    end
  end
end
```

### StreamData Generators

```elixir
defmodule GeneratorsTest do
  use ExUnit.Case
  use ExUnitProperties

  # Built-in generators
  property "integer generators" do
    check all n <- integer() do
      assert is_integer(n)
    end

    check all n <- positive_integer() do
      assert n > 0
    end

    check all n <- integer(1..100) do
      assert n >= 1 and n <= 100
    end
  end

  property "string generators" do
    check all s <- string(:alphanumeric) do
      assert String.match?(s, ~r/^[a-zA-Z0-9]*$/)
    end

    check all s <- string(:printable, min_length: 1, max_length: 50) do
      assert String.length(s) >= 1
      assert String.length(s) <= 50
    end
  end

  property "list generators" do
    check all list <- list_of(integer(), min_length: 1, max_length: 100) do
      assert length(list) >= 1
      assert length(list) <= 100
      assert Enum.all?(list, &is_integer/1)
    end
  end

  property "map generators" do
    check all map <- map_of(atom(:alphanumeric), integer()) do
      assert is_map(map)
      assert Enum.all?(Map.keys(map), &is_atom/1)
    end
  end

  property "one_of generator" do
    check all status <- StreamData.one_of([
                StreamData.constant(:active),
                StreamData.constant(:inactive),
                StreamData.constant(:pending)
              ]) do
      assert status in [:active, :inactive, :pending]
    end
  end
end
```

---

## 8. ตัวอย่างจริง: Test Suite ครบถ้วน

```elixir
# Test สำหรับ Shopping Cart system

defmodule ShoppingCartTest do
  use ExUnit.Case, async: true

  alias MyApp.{Cart, Product, User}

  setup do
    user = %User{id: 1, name: "Alice", email: "alice@example.com"}
    products = [
      %Product{id: 1, name: "Laptop", price: 999.99, stock: 10},
      %Product{id: 2, name: "Mouse", price: 29.99, stock: 50},
      %Product{id: 3, name: "Keyboard", price: 79.99, stock: 0}  # out of stock
    ]
    cart = Cart.new(user.id)

    %{user: user, products: products, cart: cart}
  end

  describe "Cart.add_item/3" do
    test "adds item to empty cart", %{cart: cart, products: [laptop | _]} do
      assert {:ok, updated_cart} = Cart.add_item(cart, laptop, 1)
      assert length(Cart.items(updated_cart)) == 1
      assert Cart.item_count(updated_cart) == 1
    end

    test "increases quantity for existing item", %{cart: cart, products: [laptop | _]} do
      {:ok, cart1} = Cart.add_item(cart, laptop, 1)
      {:ok, cart2} = Cart.add_item(cart1, laptop, 2)

      items = Cart.items(cart2)
      assert length(items) == 1
      assert hd(items).quantity == 3
    end

    test "returns error when product is out of stock", %{cart: cart, products: products} do
      keyboard = Enum.find(products, fn p -> p.id == 3 end)
      assert {:error, :out_of_stock} = Cart.add_item(cart, keyboard, 1)
    end

    test "returns error when quantity exceeds stock", %{cart: cart, products: [laptop | _]} do
      assert {:error, {:insufficient_stock, 10}} = Cart.add_item(cart, laptop, 100)
    end

    test "returns error for invalid quantity", %{cart: cart, products: [laptop | _]} do
      assert {:error, :invalid_quantity} = Cart.add_item(cart, laptop, 0)
      assert {:error, :invalid_quantity} = Cart.add_item(cart, laptop, -1)
    end
  end

  describe "Cart.remove_item/2" do
    setup %{cart: cart, products: [laptop, mouse | _]} do
      {:ok, cart_with_items} = Cart.add_item(cart, laptop, 2)
      {:ok, full_cart} = Cart.add_item(cart_with_items, mouse, 1)
      %{full_cart: full_cart}
    end

    test "removes item from cart", %{full_cart: cart} do
      assert {:ok, updated} = Cart.remove_item(cart, 1)  # laptop id
      assert length(Cart.items(updated)) == 1
      refute Enum.any?(Cart.items(updated), fn i -> i.product_id == 1 end)
    end

    test "returns error for non-existent item", %{full_cart: cart} do
      assert {:error, :item_not_found} = Cart.remove_item(cart, 999)
    end
  end

  describe "Cart.total/1" do
    test "calculates correct total", %{cart: cart, products: [laptop, mouse | _]} do
      {:ok, cart1} = Cart.add_item(cart, laptop, 1)
      {:ok, cart2} = Cart.add_item(cart1, mouse, 3)

      expected = 999.99 + (3 * 29.99)
      assert_in_delta Cart.total(cart2), expected, 0.01
    end

    test "empty cart has zero total", %{cart: cart} do
      assert Cart.total(cart) == 0.0
    end
  end

  describe "Cart.apply_coupon/2" do
    setup %{cart: cart, products: [laptop | _]} do
      {:ok, cart_with_laptop} = Cart.add_item(cart, laptop, 1)
      %{loaded_cart: cart_with_laptop}
    end

    test "applies valid percentage coupon", %{loaded_cart: cart} do
      assert {:ok, discounted} = Cart.apply_coupon(cart, "SAVE10")
      expected_total = 999.99 * 0.90
      assert_in_delta Cart.total(discounted), expected_total, 0.01
    end

    test "returns error for invalid coupon", %{loaded_cart: cart} do
      assert {:error, :invalid_coupon} = Cart.apply_coupon(cart, "NOTREAL")
    end

    test "cannot apply coupon to empty cart" do
      empty = Cart.new(99)
      assert {:error, :empty_cart} = Cart.apply_coupon(empty, "SAVE10")
    end
  end

  describe "Cart.checkout/2" do
    setup %{cart: cart, products: [laptop | _]} do
      {:ok, ready_cart} = Cart.add_item(cart, laptop, 1)
      payment_info = %{
        card_number: "4242424242424242",
        exp_month: 12,
        exp_year: 2025,
        cvv: "123"
      }
      %{ready_cart: ready_cart, payment_info: payment_info}
    end

    test "successfully checks out", %{ready_cart: cart, payment_info: payment} do
      assert {:ok, order} = Cart.checkout(cart, payment)
      assert order.status == :completed
      assert order.total > 0
      assert order.user_id == cart.user_id
    end

    test "returns error for empty cart", %{cart: cart, payment_info: payment} do
      assert {:error, :empty_cart} = Cart.checkout(cart, payment)
    end

    test "returns error for invalid payment", %{ready_cart: cart} do
      bad_payment = %{card_number: "invalid"}
      assert {:error, :payment_failed} = Cart.checkout(cart, bad_payment)
    end
  end
end
```

---

## 9. Testing GenServer

```elixir
defmodule CounterServerTest do
  use ExUnit.Case, async: true

  alias MyApp.CounterServer

  setup do
    {:ok, pid} = CounterServer.start_link(initial: 0)
    %{counter: pid}
  end

  test "starts with initial value", %{counter: pid} do
    assert CounterServer.value(pid) == 0
  end

  test "increments correctly", %{counter: pid} do
    CounterServer.increment(pid)
    CounterServer.increment(pid)
    assert CounterServer.value(pid) == 2
  end

  test "decrements correctly", %{counter: pid} do
    CounterServer.increment(pid, 10)
    CounterServer.decrement(pid, 3)
    assert CounterServer.value(pid) == 7
  end

  test "resets to zero", %{counter: pid} do
    CounterServer.increment(pid, 100)
    CounterServer.reset(pid)
    assert CounterServer.value(pid) == 0
  end

  test "handles concurrent increments" do
    {:ok, pid} = CounterServer.start_link(initial: 0)
    tasks = Enum.map(1..100, fn _ ->
      Task.async(fn -> CounterServer.increment(pid) end)
    end)
    Enum.each(tasks, &Task.await/1)
    assert CounterServer.value(pid) == 100
  end

  test "stops gracefully", %{counter: pid} do
    ref = Process.monitor(pid)
    CounterServer.stop(pid)
    assert_receive {:DOWN, ^ref, :process, ^pid, :normal}
  end
end
```

---

## 10. Testing Processes

```elixir
defmodule ProcessTest do
  use ExUnit.Case

  test "process sends message back" do
    parent = self()

    spawn(fn ->
      receive do
        {:ping, from} -> send(from, :pong)
      end
    end)
    |> send({:ping, parent})

    assert_receive :pong, 1000
  end

  test "process crashes and linked process gets exit" do
    Process.flag(:trap_exit, true)

    pid = spawn_link(fn ->
      raise "intentional crash"
    end)

    assert_receive {:EXIT, ^pid, {%RuntimeError{message: "intentional crash"}, _}}, 1000
  end

  test "monitored process sends DOWN message" do
    {pid, ref} = spawn_monitor(fn ->
      :timer.sleep(50)
    end)

    assert_receive {:DOWN, ^ref, :process, ^pid, :normal}, 1000
  end
end
```

---

## 11. ExUnit Configuration

```elixir
# test/test_helper.exs
ExUnit.start(
  timeout: 60_000,
  max_failures: 10,
  capture_log: true,
  formatters: [ExUnit.CLIFormatter],
  seed: 0  # deterministic order
)

Mox.defmock(MyApp.PaymentMock, for: Payment.Gateway)
Mox.defmock(MyApp.EmailMock, for: MyApp.Mailer)
```

```bash
# Mix test options
mix test                          # รัน ทุก tests
mix test test/user_test.exs       # รัน specific file
mix test test/user_test.exs:42    # รัน test ที่ line 42
mix test --only unit              # รัน tests ที่ tag :unit
mix test --exclude integration    # exclude tag :integration
mix test --seed 0                 # deterministic order
mix test --max-failures 5         # หยุดหลัง 5 failures
mix test --cover                  # รัน with coverage
mix test --trace                  # verbose output
mix test --stale                  # รัน tests ที่ affected โดยการเปลี่ยนแปลง
```

---

## 12. Code Coverage

```elixir
# mix.exs
def project do
  [
    ...
    test_coverage: [tool: ExCoveralls],
    preferred_cli_env: [
      coveralls: :test,
      "coveralls.detail": :test,
      "coveralls.post": :test,
      "coveralls.html": :test
    ]
  ]
end

defp deps do
  [
    {:excoveralls, "~> 0.16", only: :test},
    ...
  ]
end
```

```bash
mix coveralls          # coverage summary
mix coveralls.html     # HTML report ใน cover/excoveralls.html
mix coveralls.detail   # line-by-line detail
```

---

## 13. Exercises

### Exercise 1: Calculator Test Suite

```elixir
# สร้าง test suite ครบถ้วนสำหรับ Calculator module
defmodule CalculatorTest do
  use ExUnit.Case, async: true
  use ExUnitProperties

  describe "calculate/1" do
    # TODO: test all operations
    # TODO: test edge cases (division by zero, overflow)
    # TODO: test invalid input
  end

  # Property-based tests
  describe "properties" do
    # TODO: สมบัติของ addition (commutative, associative)
    # TODO: สมบัติของ multiplication
    # TODO: inverse operations (add then subtract = original)
  end
end
```

### Exercise 2: HTTP Client Mock Test

```elixir
# สร้าง tests สำหรับ service ที่ call HTTP API
# โดยใช้ Mox เพื่อ mock HTTP calls

Mox.defmock(HTTPClientMock, for: HTTPClient.Behaviour)

defmodule WeatherServiceTest do
  use ExUnit.Case
  import Mox

  setup :verify_on_exit!

  # TODO: test get_weather/1
  # TODO: test error handling
  # TODO: test caching behavior
end
```

### Exercise 3: Full Integration Test

```elixir
# สร้าง integration test สำหรับ User Registration flow:
# 1. User submits registration form
# 2. System validates input
# 3. System creates user in DB
# 4. System sends verification email
# 5. System creates session

defmodule RegistrationIntegrationTest do
  use ExUnit.Case
  import Mox

  @tag :integration
  # TODO: test full registration flow
  # TODO: test duplicate email
  # TODO: test invalid data
end
```

---

## เฉลย Exercises

### เฉลย Exercise 1: Calculator Test Suite

```elixir
defmodule CalculatorTest do
  use ExUnit.Case, async: true
  use ExUnitProperties

  describe "calculate/1 - addition" do
    test "adds two positive numbers" do
      assert {:ok, 15} = Calculator.calculate("10 + 5")
    end

    test "adds negative numbers" do
      assert {:ok, -3} = Calculator.calculate("-5 + 2")
    end

    test "adds floats" do
      {:ok, result} = Calculator.calculate("1.5 + 2.5")
      assert_in_delta result, 4.0, 0.001
    end
  end

  describe "calculate/1 - division" do
    test "divides evenly" do
      assert {:ok, 5.0} = Calculator.calculate("10 / 2")
    end

    test "returns error for division by zero" do
      assert {:error, "Division by zero"} = Calculator.calculate("10 / 0")
    end

    test "returns error for division by zero float" do
      assert {:error, "Division by zero"} = Calculator.calculate("10 / 0.0")
    end
  end

  describe "calculate/1 - error cases" do
    test "returns error for non-numeric input" do
      assert {:error, _} = Calculator.calculate("abc + 2")
    end

    test "returns error for missing operand" do
      assert {:error, _} = Calculator.calculate("10 +")
    end

    test "returns error for unknown operator" do
      assert {:error, _} = Calculator.calculate("10 ^ 2")
    end

    test "returns error for non-string input" do
      assert {:error, _} = Calculator.calculate(123)
    end
  end

  describe "properties" do
    property "addition is commutative" do
      check all a <- integer(), b <- integer() do
        {:ok, r1} = Calculator.calculate("#{a} + #{b}")
        {:ok, r2} = Calculator.calculate("#{b} + #{a}")
        assert r1 == r2
      end
    end

    property "subtracting then adding returns original" do
      check all a <- integer(), b <- integer() do
        {:ok, subtracted} = Calculator.calculate("#{a} - #{b}")
        {:ok, result} = Calculator.calculate("#{subtracted} + #{b}")
        assert_in_delta result, a, 0.0001
      end
    end

    property "multiply by 1 is identity" do
      check all n <- integer() do
        {:ok, result} = Calculator.calculate("#{n} * 1")
        assert result == n
      end
    end
  end
end
```

### เฉลย Exercise 2: Mock Test

```elixir
defmodule WeatherServiceTest do
  use ExUnit.Case
  import Mox

  setup :verify_on_exit!

  describe "get_weather/1" do
    test "returns weather data for valid city" do
      HTTPClientMock
      |> expect(:get, fn "https://api.weather.com/v1/current?city=Bangkok" ->
        {:ok, %{
          status: 200,
          body: ~s({"temp": 30, "humidity": 80, "description": "Partly cloudy"})
        }}
      end)

      assert {:ok, weather} = WeatherService.get_weather("Bangkok")
      assert weather.temp == 30
      assert weather.humidity == 80
    end

    test "returns error for unknown city" do
      HTTPClientMock
      |> expect(:get, fn _url ->
        {:ok, %{status: 404, body: ~s({"error": "City not found"})}}
      end)

      assert {:error, "City not found"} = WeatherService.get_weather("UnknownCity")
    end

    test "returns error on network failure" do
      HTTPClientMock
      |> expect(:get, fn _url ->
        {:error, :timeout}
      end)

      assert {:error, :network_error} = WeatherService.get_weather("Bangkok")
    end

    test "returns cached data on second call" do
      HTTPClientMock
      |> expect(:get, 1, fn _url ->
        # เรียกแค่ครั้งเดียว - ครั้งที่สองควร return จาก cache
        {:ok, %{status: 200, body: ~s({"temp": 32})}}
      end)

      WeatherService.get_weather("Bangkok")
      WeatherService.get_weather("Bangkok")  # ควร return จาก cache
    end
  end
end
```

---

## สรุป

```
ExUnit Structure:
├── use ExUnit.Case
├── setup / setup_all - test preparation
├── test "name", context do ... end
├── describe "context" do ... end
└── @tag :name - tagging

Assertions:
├── assert expr
├── refute expr
├── assert_raise ExceptionType, fn
├── assert_receive message, timeout
├── assert_in_delta float1, float2, delta
└── assert pattern = expr (pattern match)

Mox:
├── Mox.defmock(MockName, for: BehaviourModule)
├── expect(mock, :function, fn -> ... end)
├── expect(mock, :function, N, fn -> ... end)
├── stub(mock, :function, fn -> ... end)
└── setup :verify_on_exit!

StreamData (Property Testing):
├── use ExUnitProperties
├── property "name" do check all gen <- generator do ... end end
├── Generators: integer(), string(), list_of(), map_of(), one_of()
└── Custom: gen all field <- generator do ... end
```

---

*ก่อนหน้า: [Part 19](part_19.md) | ต่อไป: Phoenix Framework (Part 21 - coming soon)*
