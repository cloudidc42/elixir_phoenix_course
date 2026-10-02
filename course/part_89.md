# Part 89: GraphQL Subscriptions และ Real-time (GraphQL แบบ Real-time)

## เป้าหมายการเรียนรู้

หลังจากจบ Part นี้ คุณจะสามารถ:
- GraphQL Subscriptions ด้วย Absinthe
- Real-time updates ผ่าน WebSocket
- Authentication ใน subscriptions
- Connection management

---

## 1. Absinthe Subscription Setup

```elixir
# mix.exs
{:absinthe, "~> 1.7"},
{:absinthe_phoenix, "~> 2.0"},
{:absinthe_plug, "~> 1.5"}

# endpoint.ex
socket "/graphql/ws", Absinthe.Phoenix.Socket,
  websocket: true,
  init: {MyAppWeb.GraphQL.Socket, :init}

# application.ex
children = [
  {Absinthe.Subscription, MyAppWeb.Endpoint},
]
```

---

## 2. Schema with Subscriptions

```elixir
defmodule MyApp.Schema do
  use Absinthe.Schema

  subscription do
    field :message_sent, :message do
      arg :room_id, non_null(:id)

      config fn args, %{context: %{current_user: user}} ->
        {:ok, topic: "room:#{args.room_id}:messages"}
      end

      # Filter: only deliver if user has access
      resolve fn message, %{context: %{current_user: user}} ->
        if MyApp.Rooms.member?(user.id, message.room_id) do
          {:ok, message}
        else
          {:ok, nil}  # filtered out
        end
      end
    end

    field :notification_received, :notification do
      config fn _args, %{context: %{current_user: user}} ->
        {:ok, topic: "user:#{user.id}:notifications"}
      end

      resolve fn notification, _ -> {:ok, notification} end
    end

    field :order_updated, :order do
      arg :order_id, non_null(:id)

      config fn args, %{context: %{current_user: user}} ->
        # Verify user owns the order
        order = MyApp.Orders.get_order!(args.order_id)
        if order.user_id == user.id do
          {:ok, topic: "order:#{args.order_id}"}
        else
          {:error, "Unauthorized"}
        end
      end
    end
  end
end
```

---

## 3. Publishing Subscription Events

```elixir
defmodule MyApp.Chat do
  alias MyAppWeb.Endpoint

  def create_message(attrs) do
    with {:ok, message} <- Repo.insert(Message.changeset(%Message{}, attrs)) do
      # Publish to subscription
      Absinthe.Subscription.publish(
        Endpoint,
        message,
        message_sent: "room:#{message.room_id}:messages"
      )

      {:ok, message}
    end
  end
end

defmodule MyApp.Notifications do
  alias MyAppWeb.Endpoint

  def send_notification(user_id, notification_attrs) do
    with {:ok, notification} <- create_notification_in_db(user_id, notification_attrs) do
      Absinthe.Subscription.publish(
        Endpoint,
        notification,
        notification_received: "user:#{user_id}:notifications"
      )

      {:ok, notification}
    end
  end
end
```

---

## 4. WebSocket Authentication

```elixir
defmodule MyAppWeb.GraphQL.Socket do
  def init(%{params: params}) do
    token = params["token"] || params["Authorization"] |> extract_bearer()

    case Phoenix.Token.verify(MyAppWeb.Endpoint, "graphql socket", token, max_age: 86400) do
      {:ok, user_id} ->
        user = MyApp.Accounts.get_user!(user_id)
        {:ok, %{context: %{current_user: user}}}

      {:error, _reason} ->
        {:error, :unauthorized}
    end
  end

  defp extract_bearer(nil), do: nil
  defp extract_bearer("Bearer " <> token), do: token
  defp extract_bearer(_), do: nil
end
```

---

## 5. Client-side Subscription (JavaScript)

```javascript
import { ApolloClient, InMemoryCache, split, HttpLink } from '@apollo/client'
import { GraphQLWsLink } from '@apollo/client/link/subscriptions'
import { createClient } from 'graphql-ws'
import { getMainDefinition } from '@apollo/client/utilities'
import { gql } from '@apollo/client'

const httpLink = new HttpLink({ uri: '/api/graphql' })

const wsLink = new GraphQLWsLink(createClient({
  url: 'wss://myapp.com/graphql/ws',
  connectionParams: {
    token: localStorage.getItem('auth_token')
  }
}))

// Route to HTTP or WS based on operation type
const splitLink = split(
  ({ query }) => {
    const def = getMainDefinition(query)
    return def.kind === 'OperationDefinition' && def.operation === 'subscription'
  },
  wsLink,
  httpLink
)

const client = new ApolloClient({ link: splitLink, cache: new InMemoryCache() })

// Subscribe to new messages
const SUBSCRIBE_MESSAGES = gql`
  subscription OnMessageSent($roomId: ID!) {
    messageSent(roomId: $roomId) {
      id
      body
      author { name avatarUrl }
      insertedAt
    }
  }
`

// In React component:
const { data } = useSubscription(SUBSCRIBE_MESSAGES, {
  variables: { roomId: '123' },
  onData: ({ data: { data } }) => {
    if (data?.messageSent) {
      setMessages(prev => [...prev, data.messageSent])
    }
  }
})
```

---

## สรุป

```
GraphQL Subscriptions:
├── Setup: Absinthe.Subscription + socket
├── Schema: subscription block, config fn
├── Publish: Absinthe.Subscription.publish/3
└── Auth: init/1 returns {:ok, context}

Event Flow:
├── Mutation creates data
├── publish/3 delivers to topic
├── Absinthe filters subscribers
└── WebSocket delivers to clients

Config function:
├── Receives args + context
├── Returns {:ok, topic: "..."} or {:error, _}
└── Topic is subscription channel
```

---

*ก่อนหน้า: [Part 88](part_88.md) | ต่อไป: [Part 90 - Elixir Patterns ขั้นสูง](part_90.md)*
