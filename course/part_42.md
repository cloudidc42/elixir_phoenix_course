# Part 42: Testing Advanced (การทดสอบขั้นสูง)

## เป้าหมายการเรียนรู้

- เขียน integration tests ด้วย ConnCase อย่างมีประสิทธิภาพ
- ทดสอบ LiveView components ด้วย LiveViewTest
- ทดสอบ Phoenix Channels
- ใช้ property-based testing ด้วย StreamData
- เข้าใจ contract testing
- สร้าง test factories ด้วย ExMachina
- วัด test coverage ด้วย mix coveralls
- เข้าใจ mutation testing concepts

---

## 1. Integration Tests ด้วย ConnCase

ConnCase ช่วยให้เราทดสอบ HTTP requests และ responses ได้อย่างสมบูรณ์

```elixir
# test/my_app_web/controllers/user_controller_test.exs
defmodule MyAppWeb.UserControllerTest do
  use MyAppWeb.ConnCase

  import MyApp.AccountsFixtures

  describe "GET /users" do
    test "renders list of users for admin", %{conn: conn} do
      admin = user_fixture(%{is_admin: true})
      user1 = user_fixture(%{name: "Alice"})
      user2 = user_fixture(%{name: "Bob"})

      conn =
        conn
        |> log_in_user(admin)
        |> get(~p"/admin/users")

      assert html_response(conn, 200) =~ "Alice"
      assert html_response(conn, 200) =~ "Bob"
    end

    test "redirects when not logged in", %{conn: conn} do
      conn = get(conn, ~p"/admin/users")

      assert redirected_to(conn) == ~p"/login"
    end

    test "returns 403 for non-admin users", %{conn: conn} do
      user = user_fixture(%{is_admin: false})

      conn =
        conn
        |> log_in_user(user)
        |> get(~p"/admin/users")

      assert response(conn, 403)
    end
  end

  describe "POST /users" do
    test "creates user with valid data", %{conn: conn} do
      valid_attrs = %{
        email: "test@example.com",
        password: "SecurePass123!",
        name: "Test User"
      }

      conn = post(conn, ~p"/users", user: valid_attrs)

      assert %{id: id} = redirected_params(conn)
      assert redirected_to(conn) == ~p"/users/#{id}"

      # ตรวจสอบว่า user ถูกสร้างในฐานข้อมูล
      assert MyApp.Accounts.get_user!(id).email == "test@example.com"
    end

    test "renders errors for invalid data", %{conn: conn} do
      invalid_attrs = %{email: "not-an-email", password: "short"}

      conn = post(conn, ~p"/users", user: invalid_attrs)

      assert html_response(conn, 422) =~ "must have the @ sign"
    end
  end

  describe "PUT /users/:id" do
    test "updates user when authorized", %{conn: conn} do
      user = user_fixture()

      conn =
        conn
        |> log_in_user(user)
        |> put(~p"/users/#{user}", user: %{name: "Updated Name"})

      assert redirected_to(conn) == ~p"/users/#{user}"
      assert MyApp.Accounts.get_user!(user.id).name == "Updated Name"
    end

    test "returns 403 when updating another user", %{conn: conn} do
      user = user_fixture()
      other_user = user_fixture()

      conn =
        conn
        |> log_in_user(user)
        |> put(~p"/users/#{other_user}", user: %{name: "Hack"})

      assert response(conn, 403)
    end
  end
end
```

```elixir
# test/support/conn_case.ex
defmodule MyAppWeb.ConnCase do
  use ExUnit.CaseTemplate

  using do
    quote do
      use Phoenix.ConnTest
      import MyAppWeb.Router.Helpers
      alias MyAppWeb.Router.Helpers, as: Routes
      import MyApp.AccountsFixtures

      # The default endpoint for testing
      @endpoint MyAppWeb.Endpoint

      # helper สำหรับ login ใน tests
      def log_in_user(conn, user) do
        token = MyApp.Accounts.generate_user_session_token(user)
        conn |> Phoenix.ConnTest.init_test_session(%{user_token: token})
      end
    end
  end

  setup tags do
    MyApp.DataCase.setup_sandbox(tags)
    {:ok, conn: Phoenix.ConnTest.build_conn()}
  end
end
```

```elixir
# JSON API testing
defmodule MyAppWeb.Api.PostControllerTest do
  use MyAppWeb.ConnCase

  import MyApp.PostsFixtures

  setup %{conn: conn} do
    user = user_fixture()
    token = MyApp.Accounts.generate_api_token(user)

    conn =
      conn
      |> put_req_header("accept", "application/json")
      |> put_req_header("authorization", "Bearer #{token}")

    {:ok, conn: conn, user: user}
  end

  describe "index" do
    test "lists all posts", %{conn: conn, user: user} do
      post1 = post_fixture(%{user_id: user.id, title: "First Post"})
      post2 = post_fixture(%{user_id: user.id, title: "Second Post"})

      conn = get(conn, ~p"/api/posts")

      assert %{"data" => posts} = json_response(conn, 200)
      assert length(posts) == 2

      titles = Enum.map(posts, & &1["title"])
      assert "First Post" in titles
      assert "Second Post" in titles
    end

    test "filters posts by tag", %{conn: conn, user: user} do
      _post1 = post_fixture(%{user_id: user.id, tags: ["elixir"]})
      _post2 = post_fixture(%{user_id: user.id, tags: ["python"]})

      conn = get(conn, ~p"/api/posts?tag=elixir")

      assert %{"data" => [post]} = json_response(conn, 200)
      assert "elixir" in post["tags"]
    end
  end

  describe "create" do
    test "creates post with valid attrs", %{conn: conn} do
      attrs = %{title: "New Post", body: "Content here", tags: ["elixir"]}

      conn = post(conn, ~p"/api/posts", post: attrs)

      assert %{"data" => %{"id" => id, "title" => "New Post"}} =
               json_response(conn, 201)

      assert MyApp.Posts.get_post!(id).title == "New Post"
    end

    test "returns 422 with invalid attrs", %{conn: conn} do
      conn = post(conn, ~p"/api/posts", post: %{title: ""})

      assert %{"errors" => errors} = json_response(conn, 422)
      assert errors["title"]
    end
  end
end
```

---

## 2. LiveView Testing ด้วย LiveViewTest

```elixir
# test/my_app_web/live/todo_live_test.exs
defmodule MyAppWeb.TodoLiveTest do
  use MyAppWeb.ConnCase
  import Phoenix.LiveViewTest

  describe "Index page" do
    test "renders todo list", %{conn: conn} do
      user = user_fixture()
      todo1 = todo_fixture(%{user_id: user.id, title: "Buy groceries"})
      todo2 = todo_fixture(%{user_id: user.id, title: "Write tests"})

      {:ok, view, html} =
        conn
        |> log_in_user(user)
        |> live(~p"/todos")

      assert html =~ "Buy groceries"
      assert html =~ "Write tests"
    end

    test "creates new todo", %{conn: conn} do
      user = user_fixture()

      {:ok, view, _html} =
        conn
        |> log_in_user(user)
        |> live(~p"/todos")

      # กรอก form และ submit
      view
      |> form("#todo-form", todo: %{title: "New Task"})
      |> render_submit()

      # ตรวจสอบว่า todo ปรากฏใน DOM
      assert render(view) =~ "New Task"
    end

    test "marks todo as complete", %{conn: conn} do
      user = user_fixture()
      todo = todo_fixture(%{user_id: user.id, title: "Task to complete"})

      {:ok, view, _html} =
        conn
        |> log_in_user(user)
        |> live(~p"/todos")

      # click toggle button
      view
      |> element("#todo-#{todo.id} [data-role='toggle']")
      |> render_click()

      # ตรวจสอบ class หรือ state
      assert render(view) =~ "completed"
    end

    test "filters todos by status", %{conn: conn} do
      user = user_fixture()
      _active = todo_fixture(%{user_id: user.id, title: "Active", completed: false})
      _done = todo_fixture(%{user_id: user.id, title: "Done", completed: true})

      {:ok, view, _html} =
        conn
        |> log_in_user(user)
        |> live(~p"/todos")

      # คลิก filter
      view
      |> element("[data-filter='active']")
      |> render_click()

      assert render(view) =~ "Active"
      refute render(view) =~ "Done"
    end
  end

  describe "Real-time updates" do
    test "updates when new todo is added by another user", %{conn: conn} do
      user = user_fixture()

      {:ok, view, _html} =
        conn
        |> log_in_user(user)
        |> live(~p"/todos")

      # simulate message from another process (PubSub broadcast)
      send(view.pid, {:new_todo, %{id: 999, title: "Broadcasted Todo"}})

      assert render(view) =~ "Broadcasted Todo"
    end
  end
end
```

```elixir
# ทดสอบ LiveView components แบบแยก
defmodule MyAppWeb.Components.ModalTest do
  use MyAppWeb.ConnCase
  import Phoenix.LiveViewTest

  test "renders modal content", %{conn: conn} do
    {:ok, view, _html} = live_isolated(conn, MyAppWeb.Components.Modal,
      session: %{"title" => "Test Modal", "content" => "Modal body"}
    )

    assert render(view) =~ "Test Modal"
    assert render(view) =~ "Modal body"
  end

  test "closes modal on close button click", %{conn: conn} do
    {:ok, view, _html} = live_isolated(conn, MyAppWeb.Components.Modal,
      session: %{"show" => true}
    )

    assert render(view) =~ "modal-open"

    view |> element("[data-action='close']") |> render_click()

    refute render(view) =~ "modal-open"
  end
end
```

---

## 3. Channel Testing

```elixir
# test/my_app_web/channels/room_channel_test.exs
defmodule MyAppWeb.RoomChannelTest do
  use MyAppWeb.ChannelCase

  alias MyAppWeb.RoomChannel
  alias MyAppWeb.UserSocket

  setup do
    user = user_fixture()
    {:ok, token} = MyApp.Accounts.create_socket_token(user)

    {:ok, socket} = connect(UserSocket, %{"token" => token})
    {:ok, socket: socket, user: user}
  end

  describe "join" do
    test "joins room successfully", %{socket: socket} do
      {:ok, _reply, _socket} = subscribe_and_join(socket, RoomChannel, "room:general")
    end

    test "joins private room with authorization", %{socket: socket, user: user} do
      room = room_fixture(%{private: true, members: [user.id]})

      {:ok, reply, _socket} =
        subscribe_and_join(socket, RoomChannel, "room:#{room.id}")

      assert reply.status == "ok"
    end

    test "rejects join for unauthorized private room", %{socket: socket} do
      room = room_fixture(%{private: true, members: []})

      assert {:error, %{reason: "unauthorized"}} =
               subscribe_and_join(socket, RoomChannel, "room:#{room.id}")
    end
  end

  describe "handle_in" do
    setup %{socket: socket} do
      {:ok, _, socket} = subscribe_and_join(socket, RoomChannel, "room:general")
      {:ok, socket: socket}
    end

    test "new:msg broadcasts message to room", %{socket: socket, user: user} do
      push(socket, "new:msg", %{"body" => "Hello World"})

      # ตรวจสอบว่า message ถูก broadcast
      assert_broadcast "new:msg", %{body: "Hello World", user_id: _}
    end

    test "typing:start broadcasts typing indicator", %{socket: socket, user: user} do
      push(socket, "typing:start", %{})

      assert_broadcast "typing:start", %{user_id: user_id}
      assert user_id == user.id
    end

    test "handles message with reply", %{socket: socket} do
      ref = push(socket, "ping", %{})

      assert_reply ref, :ok, %{pong: true}
    end
  end

  describe "handle_out" do
    test "intercepts and transforms messages", %{socket: socket, user: user} do
      {:ok, _, socket} = subscribe_and_join(socket, RoomChannel, "room:general")

      # Simulate outgoing message
      broadcast_from!(socket, "new:msg", %{
        body: "Test message",
        user_id: user.id
      })

      # ตรวจสอบว่า socket ได้รับ message
      assert_push "new:msg", %{body: "Test message"}
    end
  end
end
```

```elixir
# test/support/channel_case.ex
defmodule MyAppWeb.ChannelCase do
  use ExUnit.CaseTemplate

  using do
    quote do
      use Phoenix.ChannelTest
      import MyApp.AccountsFixtures

      @endpoint MyAppWeb.Endpoint
    end
  end

  setup tags do
    MyApp.DataCase.setup_sandbox(tags)
    :ok
  end
end
```

---

## 4. Property-Based Testing ด้วย StreamData

```elixir
# mix.exs
defp deps do
  [
    {:stream_data, "~> 1.1", only: [:test, :dev]}
  ]
end
```

```elixir
# test/my_app/accounts_property_test.exs
defmodule MyApp.AccountsPropertyTest do
  use ExUnit.Case
  use ExUnitProperties

  alias MyApp.Accounts.User

  describe "User.registration_changeset/2" do
    property "valid email formats are accepted" do
      check all email <- valid_email_generator() do
        attrs = %{email: email, password: "SecurePass123!"}
        changeset = User.registration_changeset(%User{}, attrs)

        # email ที่ valid ต้องไม่มี error ใน email field
        assert changeset.errors[:email] == nil
      end
    end

    property "invalid email formats are rejected" do
      check all email <- invalid_email_generator() do
        attrs = %{email: email, password: "SecurePass123!"}
        changeset = User.registration_changeset(%User{}, attrs)

        assert changeset.errors[:email] != nil
      end
    end

    property "password must be at least 12 characters" do
      check all password <- string(:ascii, min_length: 1, max_length: 11) do
        attrs = %{email: "test@example.com", password: password}
        changeset = User.registration_changeset(%User{}, attrs)

        assert changeset.errors[:password] != nil
      end
    end
  end

  # Custom generators
  defp valid_email_generator do
    gen all local <- string(:alphanumeric, min_length: 1),
            domain <- string(:alphanumeric, min_length: 1),
            tld <- member_of(["com", "org", "net", "io"]) do
      "#{local}@#{domain}.#{tld}"
    end
  end

  defp invalid_email_generator do
    one_of([
      constant(""),
      constant("notanemail"),
      constant("missing@tld"),
      constant("@nodomain.com"),
      string(:alphanumeric)  # random strings without @
    ])
  end
end
```

```elixir
# ทดสอบ pure functions ด้วย property-based testing
defmodule MyApp.MathPropertyTest do
  use ExUnit.Case
  use ExUnitProperties

  alias MyApp.Calculator

  property "addition is commutative" do
    check all a <- integer(), b <- integer() do
      assert Calculator.add(a, b) == Calculator.add(b, a)
    end
  end

  property "addition is associative" do
    check all a <- integer(), b <- integer(), c <- integer() do
      assert Calculator.add(Calculator.add(a, b), c) ==
             Calculator.add(a, Calculator.add(b, c))
    end
  end

  property "sorting preserves elements" do
    check all list <- list_of(integer()) do
      sorted = Enum.sort(list)
      assert length(sorted) == length(list)
      assert Enum.sort(sorted) == sorted
      assert MapSet.new(sorted) == MapSet.new(list)
    end
  end

  property "encoding/decoding is identity" do
    check all data <- binary() do
      assert data
             |> Base.encode64()
             |> Base.decode64!() == data
    end
  end
end
```

---

## 5. Test Factories ด้วย ExMachina

```elixir
# mix.exs
defp deps do
  [
    {:ex_machina, "~> 2.7", only: :test}
  ]
end
```

```elixir
# test/support/factory.ex
defmodule MyApp.Factory do
  use ExMachina.Ecto, repo: MyApp.Repo

  alias MyApp.Accounts.User
  alias MyApp.Posts.Post
  alias MyApp.Comments.Comment

  def user_factory do
    %User{
      name: sequence(:name, &"User #{&1}"),
      email: sequence(:email, &"user#{&1}@example.com"),
      hashed_password: Argon2.hash_pwd_salt("password123!"),
      is_admin: false,
      verified: true
    }
  end

  def admin_user_factory do
    struct!(
      user_factory(),
      %{
        is_admin: true,
        email: sequence(:admin_email, &"admin#{&1}@example.com")
      }
    )
  end

  def post_factory do
    %Post{
      title: sequence(:title, &"Post Title #{&1}"),
      body: "This is the post body with some content.",
      published: true,
      tags: ["elixir", "phoenix"],
      user: build(:user)
    }
  end

  def draft_post_factory do
    struct!(post_factory(), %{published: false})
  end

  def comment_factory do
    %Comment{
      body: "This is a comment.",
      user: build(:user),
      post: build(:post)
    }
  end

  # Factory ที่มี associations
  def post_with_comments_factory do
    %Post{
      post_factory()
      | comments: build_list(3, :comment)
    }
  end
end
```

```elixir
# ใช้ factory ใน tests
defmodule MyApp.PostsTest do
  use MyApp.DataCase
  import MyApp.Factory

  describe "create_post/1" do
    test "creates post successfully" do
      user = insert(:user)
      attrs = params_for(:post, user_id: user.id)

      assert {:ok, post} = MyApp.Posts.create_post(attrs)
      assert post.title == attrs.title
    end
  end

  describe "list_posts/0" do
    test "returns published posts only" do
      insert(:post, published: true, title: "Published")
      insert(:draft_post, title: "Draft")

      posts = MyApp.Posts.list_published_posts()

      titles = Enum.map(posts, & &1.title)
      assert "Published" in titles
      refute "Draft" in titles
    end

    test "returns posts with comments count" do
      post = insert(:post_with_comments)

      posts = MyApp.Posts.list_posts_with_stats()
      found = Enum.find(posts, & &1.id == post.id)

      assert found.comments_count == 3
    end
  end
end
```

```elixir
# build vs insert
# build - สร้าง struct ใน memory (ไม่บันทึก DB)
user = build(:user)

# insert - บันทึกลง DB
user = insert(:user)

# insert_list - สร้างหลายรายการ
users = insert_list(5, :user)

# params_for - คืน map ของ attributes (ใช้กับ controller tests)
attrs = params_for(:post)

# build_list - สร้าง list โดยไม่บันทึก
posts = build_list(3, :post, user: user)
```

---

## 6. Test Coverage ด้วย mix coveralls

```elixir
# mix.exs
defp deps do
  [
    {:excoveralls, "~> 0.18", only: :test}
  ]
end

def project do
  [
    # ...
    test_coverage: [tool: ExCoveralls],
    preferred_cli_env: [
      coveralls: :test,
      "coveralls.detail": :test,
      "coveralls.post": :test,
      "coveralls.html": :test,
      "coveralls.cobertura": :test
    ]
  ]
end
```

```bash
# รัน tests พร้อม coverage
mix coveralls

# แสดง detail ว่า lines ไหนไม่ได้ test
mix coveralls.detail

# สร้าง HTML report
mix coveralls.html

# กำหนด minimum coverage threshold
mix coveralls --minimum-coverage 80
```

```elixir
# coveralls.json - กำหนด configuration
{
  "coverage_options": {
    "minimum_coverage": 80,
    "treat_no_relevant_lines_as_covered": true
  },
  "skip_files": [
    "lib/my_app_web/telemetry.ex",
    "lib/my_app/release.ex",
    "test/"
  ]
}
```

```elixir
# ไฟล์ที่ควร exclude จาก coverage
# - Migration files
# - Release tasks
# - Generated code
# - Configuration modules

# ตัวอย่าง ignore specific lines
defmodule MyApp.SomeModule do
  def some_function do
    # coveralls-ignore-start
    # Legacy code ที่ยังไม่ได้ test
    :legacy_behavior
    # coveralls-ignore-stop
  end
end
```

---

## 7. Fixtures และ Test Helpers

```elixir
# test/support/fixtures/accounts_fixtures.ex
defmodule MyApp.AccountsFixtures do
  @moduledoc """
  Test fixtures สำหรับ Accounts context
  """

  def unique_user_email, do: "user#{System.unique_integer()}@example.com"
  def valid_user_password, do: "SecurePassword123!"

  def valid_user_attributes(attrs \\ %{}) do
    Enum.into(attrs, %{
      email: unique_user_email(),
      password: valid_user_password(),
      name: "Test User"
    })
  end

  def user_fixture(attrs \\ %{}) do
    {:ok, user} =
      attrs
      |> valid_user_attributes()
      |> MyApp.Accounts.register_user()

    user
  end

  def confirmed_user_fixture(attrs \\ %{}) do
    user = user_fixture(attrs)

    token =
      extract_user_token(fn url ->
        MyApp.Accounts.deliver_user_confirmation_instructions(user, url)
      end)

    {:ok, user} = MyApp.Accounts.confirm_user(token)
    user
  end

  def extract_user_token(fun) do
    {:ok, captured_email} = fun.(&"[TOKEN]#{&1}[TOKEN]")
    [_, token | _] = String.split(captured_email.text_body, "[TOKEN]")
    token
  end
end
```

---

## 8. Mutation Testing Concepts

Mutation testing คือการทดสอบคุณภาพของ tests โดยการเปลี่ยน (mutate) code และดูว่า tests จับได้

```elixir
# ตัวอย่าง mutation ที่ tests ควรจับได้

# Original code
def is_adult?(age), do: age >= 18

# Mutation 1: เปลี่ยน >= เป็น >
def is_adult?(age), do: age > 18

# Tests ที่ดีต้องจับ mutation นี้ได้
test "17 year old is not adult" do
  refute MyApp.Users.is_adult?(17)
end

test "18 year old is adult" do
  assert MyApp.Users.is_adult?(18)   # Test นี้จะ fail ถ้า mutation 1 ถูกใช้
end

test "19 year old is adult" do
  assert MyApp.Users.is_adult?(19)
end
```

```elixir
# Mutation testing ด้วย muzak (Elixir mutation testing tool)
# mix.exs
defp deps do
  [
    {:muzak, "~> 1.1", only: :dev}
  ]
end

# รัน mutation tests
# mix muzak

# ตัวอย่าง mutations ที่ muzak ทำได้:
# - เปลี่ยน boolean conditions (== เป็น !=, > เป็น >=)
# - เปลี่ยน arithmetic operators (+ เป็น -, * เป็น /)
# - ลบ function calls
# - เปลี่ยน return values
# - เปลี่ยน string literals
```

```elixir
# Best practices เพื่อ mutation score สูง

# 1. ทดสอบ boundary conditions
test "exactly at boundary" do
  assert MyApp.Pricing.discount_rate(100) == 0.1  # ≥100 gets 10% discount
  assert MyApp.Pricing.discount_rate(99) == 0.0   # <100 gets no discount
end

# 2. ทดสอบ ทั้ง true และ false branches
test "active user can post" do
  assert MyApp.Permissions.can_post?(%{active: true, banned: false})
end

test "banned user cannot post" do
  refute MyApp.Permissions.can_post?(%{active: true, banned: true})
end

test "inactive user cannot post" do
  refute MyApp.Permissions.can_post?(%{active: false, banned: false})
end

# 3. Assert specific values ไม่ใช่แค่ truthy/falsy
test "calculates correct total" do
  assert MyApp.Cart.total([%{price: 100, qty: 2}, %{price: 50, qty: 1}]) == 250
  # ไม่ใช่แค่: assert MyApp.Cart.total(...) > 0
end
```

---

## สรุป

```
Testing Pyramid สำหรับ Phoenix Application:
                    ┌─────────┐
                    │  E2E    │  (น้อยที่สุด)
                   /│Wallaby  │\
                  / └─────────┘ \
                 /               \
            ┌───────────────────┐
            │   Integration     │
            │  ConnCase/LiveView│
            └───────────────────┘
           /                     \
          /                       \
     ┌───────────────────────────┐
     │         Unit Tests         │
     │  DataCase / ExUnit        │
     └───────────────────────────┘

Test Types Summary:
┌────────────────┬──────────────────────────────┐
│ Tool           │ Use Case                     │
├────────────────┼──────────────────────────────┤
│ ConnCase       │ HTTP request/response tests  │
│ LiveViewTest   │ LiveView component tests     │
│ ChannelCase    │ WebSocket channel tests      │
│ StreamData     │ Property-based testing       │
│ ExMachina      │ Test data factories          │
│ ExCoveralls    │ Code coverage reporting      │
│ Muzak          │ Mutation testing             │
└────────────────┴──────────────────────────────┘

Commands:
mix test                    # รัน all tests
mix test --only integration # รัน integration tests
mix coveralls.html          # Coverage HTML report
mix muzak                   # Mutation testing
```

---

*ก่อนหน้า: [Part 41 - Security](part_41.md) | ต่อไป: [Part 43 - Phoenix PubSub and Real-time Patterns](part_43.md)*
