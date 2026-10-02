# Part 76: Advanced Testing Patterns (การทดสอบขั้นสูง)

## เป้าหมายการเรียนรู้

หลังจากจบ Part นี้ คุณจะสามารถ:
- Contract testing ระหว่าง services
- Mutation testing ด้วย Muzak
- Load testing ด้วย k6/Tsung
- Test data management patterns

---

## 1. Contract Testing

```elixir
# ตรวจสอบว่า API ตรงตาม contract ที่กำหนด
defmodule MyApp.ContractTest do
  use ExUnit.Case

  @user_contract %{
    "id" => {:required, :integer},
    "email" => {:required, :string},
    "name" => {:required, :string},
    "created_at" => {:required, :string}
  }

  test "GET /api/users/:id returns correct contract" do
    {:ok, user} = MyApp.Accounts.create_user(%{
      email: "test@example.com",
      name: "Test User",
      password: "password123"
    })

    conn = build_conn()
    |> put_req_header("authorization", "Bearer #{generate_token(user)}")
    |> get("/api/users/#{user.id}")

    assert conn.status == 200
    body = Jason.decode!(conn.resp_body)

    assert_contract(body, @user_contract)
  end

  defp assert_contract(data, contract) do
    Enum.each(contract, fn
      {field, {:required, type}} ->
        assert Map.has_key?(data, field), "Missing required field: #{field}"
        assert_type(data[field], type, field)

      {field, {:optional, type}} ->
        if Map.has_key?(data, field) do
          assert_type(data[field], type, field)
        end
    end)
  end

  defp assert_type(value, :integer, field) do
    assert is_integer(value), "Field #{field} should be integer, got #{inspect(value)}"
  end
  defp assert_type(value, :string, field) do
    assert is_binary(value), "Field #{field} should be string, got #{inspect(value)}"
  end
end
```

---

## 2. Behaviour-driven Testing

```elixir
defmodule MyApp.UserRegistrationTest do
  use ExUnit.Case
  use MyApp.DataCase

  describe "given a valid registration form" do
    setup do
      valid_params = %{
        name: "John Doe",
        email: "john@example.com",
        password: "SecurePass123!",
        password_confirmation: "SecurePass123!"
      }
      %{params: valid_params}
    end

    test "it creates a new user account", %{params: params} do
      assert {:ok, user} = MyApp.Accounts.register_user(params)
      assert user.email == params.email
      assert user.name == params.name
    end

    test "it hashes the password", %{params: params} do
      {:ok, user} = MyApp.Accounts.register_user(params)
      refute user.hashed_password == params.password
      assert Bcrypt.verify_pass(params.password, user.hashed_password)
    end

    test "it sends a confirmation email", %{params: params} do
      {:ok, _user} = MyApp.Accounts.register_user(params)
      assert_email_delivered_with(subject: "Confirm your email")
    end
  end

  describe "given an email that already exists" do
    setup [:create_existing_user]

    test "it returns a duplicate email error", %{existing_user: existing} do
      params = %{email: existing.email, name: "Someone Else", password: "Pass123!"}
      assert {:error, changeset} = MyApp.Accounts.register_user(params)
      assert "has already been taken" in errors_on(changeset).email
    end
  end
end
```

---

## 3. Property-based Testing

```elixir
defmodule MyApp.SlugTest do
  use ExUnit.Case
  use ExUnitProperties

  property "slugs never contain spaces or special chars" do
    check all title <- string(:printable, min_length: 1, max_length: 200) do
      slug = MyApp.Slugifier.slugify(title)
      refute String.contains?(slug, " ")
      assert slug =~ ~r/^[a-z0-9-]*$/
    end
  end

  property "slug length is always <= 250 chars" do
    check all title <- string(:printable, min_length: 1, max_length: 1000) do
      slug = MyApp.Slugifier.slugify(title)
      assert String.length(slug) <= 250
    end
  end

  property "slug is idempotent" do
    check all title <- string(:alphanumeric, min_length: 1) do
      slug = MyApp.Slugifier.slugify(title)
      assert MyApp.Slugifier.slugify(slug) == slug
    end
  end
end
```

---

## 4. Load Testing Script (k6)

```javascript
// load_test.js
import http from 'k6/http'
import { check, sleep } from 'k6'
import { Rate } from 'k6/metrics'

const errorRate = new Rate('errors')

export const options = {
  stages: [
    { duration: '30s', target: 10 },   // ramp up
    { duration: '1m', target: 50 },    // stay at 50
    { duration: '30s', target: 100 },  // ramp to 100
    { duration: '1m', target: 100 },   // stay at 100
    { duration: '30s', target: 0 },    // ramp down
  ],
  thresholds: {
    errors: ['rate<0.1'],
    http_req_duration: ['p(95)<500'],
  },
}

export default function() {
  const res = http.get('https://myapp.example.com/api/articles')

  const ok = check(res, {
    'status is 200': (r) => r.status === 200,
    'response time < 500ms': (r) => r.timings.duration < 500,
    'body has articles': (r) => JSON.parse(r.body).articles !== undefined
  })

  errorRate.add(!ok)
  sleep(1)
}
```

---

## 5. Test Data Factories

```elixir
defmodule MyApp.Factory do
  use ExMachina.Ecto, repo: MyApp.Repo

  def user_factory do
    %MyApp.User{
      name: sequence("User Name"),
      email: sequence(:email, &"user#{&1}@example.com"),
      hashed_password: Bcrypt.hash_pwd_salt("password"),
      role: "user",
      status: "active"
    }
  end

  def admin_user_factory do
    struct!(user_factory(), role: "admin")
  end

  def article_factory do
    %MyApp.Article{
      title: sequence("Article Title"),
      slug: sequence("article-slug"),
      body: "This is the body content of the article.",
      status: "published",
      author: build(:user)
    }
  end

  def draft_article_factory do
    struct!(article_factory(), status: "draft")
  end

  def order_factory do
    %MyApp.Order{
      user: build(:user),
      status: "pending",
      total: Decimal.new("99.99"),
      items: build_list(2, :order_item)
    }
  end

  def order_item_factory do
    %MyApp.OrderItem{
      product: build(:product),
      quantity: 1,
      unit_price: Decimal.new("49.99")
    }
  end
end

# Usage in tests
defmodule MyApp.ArticleTest do
  use MyApp.DataCase
  import MyApp.Factory

  test "published articles are visible" do
    _draft = insert(:draft_article)
    published = insert(:article)

    articles = MyApp.Blog.list_published_articles()
    assert length(articles) == 1
    assert hd(articles).id == published.id
  end
end
```

---

## 6. Mocking External Services

```elixir
defmodule MyApp.Mocks do
  # Define mock behaviours
  defmock(MyApp.MockStripe, for: MyApp.Stripe.Behaviour)
  defmock(MyApp.MockEmail, for: MyApp.Mailer.Behaviour)
  defmock(MyApp.MockS3, for: MyApp.Storage.Behaviour)
end

defmodule MyApp.OrdersTest do
  use ExUnit.Case, async: true
  import Mox

  setup :verify_on_exit!

  test "creates order and charges customer" do
    user = %{id: 1, stripe_customer_id: "cus_123"}

    expect(MyApp.MockStripe, :charge, fn customer_id, amount ->
      assert customer_id == "cus_123"
      assert amount == 9999
      {:ok, %{charge_id: "ch_test123"}}
    end)

    expect(MyApp.MockEmail, :deliver, fn email ->
      assert email.subject =~ "Order Confirmation"
      :ok
    end)

    assert {:ok, order} = MyApp.Orders.create(user, %{items: [...], total: 9999})
    assert order.stripe_charge_id == "ch_test123"
  end
end
```

---

## สรุป

```
Testing Pyramid:
├── Unit tests: pure functions, contexts
├── Integration tests: database, external calls
├── Contract tests: API shape verification
└── Load tests: performance under stress

Key Libraries:
├── ExMachina: test data factories
├── Mox: mock behaviours
├── StreamData: property-based testing
└── k6/Tsung: load testing

Patterns:
├── Behaviour-driven: given/when/then
├── Factory pattern: test data
├── Contract testing: API shape
└── Property testing: edge cases
```

---

*ก่อนหน้า: [Part 75](part_75.md) | ต่อไป: [Part 77 - Phoenix Deployment Patterns](part_77.md)*
