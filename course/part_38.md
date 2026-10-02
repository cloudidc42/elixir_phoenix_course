# Part 38: GraphQL ด้วย Absinthe

## เป้าหมายการเรียนรู้

หลังจากจบ Part นี้ คุณจะสามารถ:
- สร้าง GraphQL API ด้วย Absinthe
- สร้าง Queries, Mutations, Subscriptions
- จัดการ Authorization
- Optimize ด้วย Dataloader

---

## 1. ติดตั้ง Absinthe

```elixir
# mix.exs
{:absinthe, "~> 1.7"},
{:absinthe_plug, "~> 1.5"},
{:absinthe_phoenix, "~> 2.0"},
{:dataloader, "~> 2.0"}

# router.ex
forward "/api/graphql", Absinthe.Plug,
  schema: MyAppWeb.Schema,
  json_codec: Jason

if Mix.env() == :dev do
  forward "/api/graphiql", Absinthe.Plug.GraphiQL,
    schema: MyAppWeb.Schema,
    interface: :playground
end
```

---

## 2. Schema พื้นฐาน

```elixir
# lib/my_app_web/schema.ex
defmodule MyAppWeb.Schema do
  use Absinthe.Schema
  import AbsintheErrorPayload.Payload

  import_types Absinthe.Type.Custom
  import_types MyAppWeb.Schema.Types.UserTypes
  import_types MyAppWeb.Schema.Types.PostTypes

  query do
    import_fields :user_queries
    import_fields :post_queries
  end

  mutation do
    import_fields :user_mutations
    import_fields :post_mutations
  end

  subscription do
    import_fields :post_subscriptions
  end
end
```

---

## 3. Types

```elixir
# lib/my_app_web/schema/types/user_types.ex
defmodule MyAppWeb.Schema.Types.UserTypes do
  use Absinthe.Schema.Notation
  import Absinthe.Resolution.Helpers, only: [dataloader: 1]

  object :user do
    field :id, non_null(:id)
    field :name, non_null(:string)
    field :email, non_null(:string)
    field :role, non_null(:string)
    field :active, non_null(:boolean)
    field :inserted_at, :naive_datetime
    field :updated_at, :naive_datetime

    # Relations
    field :posts, list_of(:post) do
      resolve dataloader(MyApp.Blog)
    end

    field :post_count, :integer do
      resolve fn user, _args, _ctx ->
        count = MyApp.Blog.count_posts(user.id)
        {:ok, count}
      end
    end
  end

  input_object :create_user_input do
    field :name, non_null(:string)
    field :email, non_null(:string)
    field :password, non_null(:string)
  end

  input_object :update_user_input do
    field :name, :string
    field :email, :string
  end

  object :user_queries do
    field :me, :user do
      resolve fn _parent, _args, %{context: %{current_user: user}} ->
        {:ok, user}
      end
    end

    field :user, :user do
      arg :id, non_null(:id)
      resolve fn _parent, %{id: id}, _ctx ->
        MyApp.Accounts.get_user(id)
      end
    end

    field :users, list_of(:user) do
      arg :limit, :integer, default_value: 20
      arg :offset, :integer, default_value: 0
      arg :search, :string

      resolve fn _parent, args, _ctx ->
        users = MyApp.Accounts.list_users(args)
        {:ok, users}
      end
    end
  end

  object :user_mutations do
    field :create_user, :user do
      arg :input, non_null(:create_user_input)

      resolve fn _parent, %{input: attrs}, _ctx ->
        MyApp.Accounts.register_user(attrs)
      end
    end

    field :update_user, :user do
      arg :id, non_null(:id)
      arg :input, non_null(:update_user_input)

      resolve fn _parent, %{id: id, input: attrs}, %{context: %{current_user: current}} ->
        with {:ok, user} <- MyApp.Accounts.get_user(id),
             :ok <- authorize_user_update(current, user),
             {:ok, updated} <- MyApp.Accounts.update_user(user, attrs) do
          {:ok, updated}
        end
      end
    end

    field :delete_user, :boolean do
      arg :id, non_null(:id)

      resolve fn _parent, %{id: id}, %{context: %{current_user: current}} ->
        with {:ok, user} <- MyApp.Accounts.get_user(id),
             :ok <- authorize_admin(current),
             {:ok, _} <- MyApp.Accounts.delete_user(user) do
          {:ok, true}
        end
      end
    end
  end

  defp authorize_user_update(current, user) do
    if current.id == user.id or current.role == "admin" do
      :ok
    else
      {:error, "Unauthorized"}
    end
  end

  defp authorize_admin(user) do
    if user.role == "admin", do: :ok, else: {:error, "Admin only"}
  end
end
```

---

## 4. Subscriptions (Real-time)

```elixir
defmodule MyAppWeb.Schema.Types.PostTypes do
  use Absinthe.Schema.Notation

  object :post do
    field :id, non_null(:id)
    field :title, non_null(:string)
    field :body, :string
    field :published, :boolean
    field :user, :user do
      resolve fn post, _args, %{context: %{loader: loader}} ->
        loader
        |> Dataloader.load(MyApp.Accounts, :user, post)
        |> on_load(fn loader ->
          {:ok, Dataloader.get(loader, MyApp.Accounts, :user, post)}
        end)
      end
    end
  end

  object :post_subscriptions do
    field :post_created, :post do
      config fn _args, %{context: %{current_user: user}} ->
        {:ok, topic: "user:#{user.id}:posts"}
      end

      trigger :create_post,
        topic: fn post ->
          ["user:#{post.user_id}:posts", "global:posts"]
        end
    end

    field :post_updated, :post do
      config fn %{post_id: post_id}, _ctx ->
        {:ok, topic: "post:#{post_id}"}
      end

      trigger :update_post,
        topic: fn post -> "post:#{post.id}" end
    end
  end
end
```

---

## 5. Dataloader (N+1 Solution)

```elixir
defmodule MyApp.Blog.Loader do
  def data do
    Dataloader.Ecto.new(MyApp.Repo)
  end
end

defmodule MyAppWeb.Schema do
  use Absinthe.Schema

  def context(ctx) do
    loader =
      Dataloader.new()
      |> Dataloader.add_source(MyApp.Blog, MyApp.Blog.Loader.data())
      |> Dataloader.add_source(MyApp.Accounts, Dataloader.Ecto.new(MyApp.Repo))

    Map.put(ctx, :loader, loader)
  end

  def plugins do
    [Absinthe.Middleware.Dataloader | Absinthe.Plugin.defaults()]
  end
end
```

---

## 6. Authentication Middleware

```elixir
defmodule MyAppWeb.Schema.Middleware.Auth do
  @behaviour Absinthe.Middleware

  def call(resolution, _config) do
    case resolution.context do
      %{current_user: nil} ->
        resolution
        |> Absinthe.Resolution.put_result({:error, "Not authenticated"})

      %{current_user: _user} ->
        resolution
    end
  end
end

defmodule MyAppWeb.Schema.Middleware.RequireAdmin do
  @behaviour Absinthe.Middleware

  def call(resolution, _config) do
    case resolution.context do
      %{current_user: %{role: "admin"}} ->
        resolution

      _ ->
        resolution
        |> Absinthe.Resolution.put_result({:error, "Admin access required"})
    end
  end
end

# ใช้ใน schema
object :user_mutations do
  field :delete_user, :boolean do
    middleware MyAppWeb.Schema.Middleware.Auth
    middleware MyAppWeb.Schema.Middleware.RequireAdmin
    arg :id, non_null(:id)
    resolve &MyAppWeb.Resolvers.Users.delete/3
  end
end
```

---

## 7. Context ด้วย Plug

```elixir
defmodule MyAppWeb.Plugs.AbsintheContext do
  @behaviour Plug

  import Plug.Conn

  def init(opts), do: opts

  def call(conn, _) do
    context = %{
      current_user: conn.assigns[:current_user],
      remote_ip: conn.remote_ip
    }

    Absinthe.Plug.put_options(conn, context: context)
  end
end

# router.ex
pipeline :graphql do
  plug :accepts, ["json"]
  plug MyAppWeb.Plugs.ApiAuth
  plug MyAppWeb.Plugs.AbsintheContext
end

scope "/api" do
  pipe_through :graphql
  forward "/graphql", Absinthe.Plug, schema: MyAppWeb.Schema
end
```

---

## 8. Testing GraphQL

```elixir
defmodule MyAppWeb.Schema.UserQueryTest do
  use MyAppWeb.ConnCase

  @me_query """
  query {
    me {
      id
      name
      email
    }
  }
  """

  @create_user_mutation """
  mutation CreateUser($input: CreateUserInput!) {
    createUser(input: $input) {
      id
      name
      email
    }
  }
  """

  test "returns current user", %{conn: conn} do
    user = insert(:user)
    conn = authenticate(conn, user)

    response =
      conn
      |> post("/api/graphql", %{query: @me_query})
      |> json_response(200)

    assert response["data"]["me"]["id"] == to_string(user.id)
    assert response["data"]["me"]["email"] == user.email
  end

  test "creates user", %{conn: conn} do
    variables = %{
      input: %{
        name: "Alice",
        email: "alice@example.com",
        password: "Password123!"
      }
    }

    response =
      conn
      |> post("/api/graphql", %{query: @create_user_mutation, variables: variables})
      |> json_response(200)

    assert response["data"]["createUser"]["name"] == "Alice"
    refute response["errors"]
  end

  defp authenticate(conn, user) do
    token = MyApp.Auth.Token.generate!(user)
    put_req_header(conn, "authorization", "Bearer #{token}")
  end
end
```

---

## สรุป

```
Absinthe GraphQL:
├── Schema: types, queries, mutations, subscriptions
├── Types: object, input_object, enum, scalar
├── Resolvers: functions ที่ return {:ok, result}
└── Middleware: auth, validation, logging

Performance:
├── Dataloader: batch N+1 queries
├── Complexity analysis
└── Query depth limiting

Real-time:
├── Subscriptions ด้วย WebSocket
├── Trigger on mutations
└── Topic-based delivery
```

---

*ก่อนหน้า: [Part 37](part_37.md) | ต่อไป: [Part 39 - Deployment](part_39.md)*
