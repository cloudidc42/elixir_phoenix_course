# Part 80: Refactoring Patterns (การ Refactor Code)

## เป้าหมายการเรียนรู้

หลังจากจบ Part นี้ คุณจะสามารถ:
- Refactor ด้วย pattern matching
- Extract และ compose functions
- Replace conditionals with polymorphism
- Improve code readability ใน Elixir

---

## 1. Replace Nested Conditionals with Pattern Matching

```elixir
# Before: nested if/case
def process_payment(order) do
  if order.status == "pending" do
    if order.user.payment_method do
      if order.total > 0 do
        charge_card(order)
      else
        {:error, :zero_total}
      end
    else
      {:error, :no_payment_method}
    end
  else
    {:error, :invalid_status}
  end
end

# After: pattern matching + with
def process_payment(%{status: "pending", total: total}) when total <= 0 do
  {:error, :zero_total}
end

def process_payment(%{status: "pending", user: %{payment_method: nil}}) do
  {:error, :no_payment_method}
end

def process_payment(%{status: "pending"} = order) do
  with {:ok, charge} <- charge_card(order),
       {:ok, _} <- record_payment(order, charge) do
    {:ok, charge}
  end
end

def process_payment(_order) do
  {:error, :invalid_status}
end
```

---

## 2. Extract Function Pipeline

```elixir
# Before: long function with mutations
def create_user(attrs) do
  email = String.downcase(attrs.email)
  name = String.trim(attrs.name)
  hashed_pw = Bcrypt.hash_pwd_salt(attrs.password)

  if String.match?(email, ~r/@/) do
    if String.length(name) > 0 do
      Repo.insert(%User{email: email, name: name, hashed_password: hashed_pw})
    else
      {:error, "Name required"}
    end
  else
    {:error, "Invalid email"}
  end
end

# After: pipeline with small focused functions
def create_user(attrs) do
  attrs
  |> normalize_attrs()
  |> validate_user_attrs()
  |> insert_user()
end

defp normalize_attrs(attrs) do
  %{attrs |
    email: String.downcase(attrs[:email] || ""),
    name: String.trim(attrs[:name] || ""),
    hashed_password: Bcrypt.hash_pwd_salt(attrs[:password] || "")
  }
end

defp validate_user_attrs(attrs) do
  cond do
    not String.match?(attrs.email, ~r/^[^\s@]+@[^\s@]+\.[^\s@]+$/) ->
      {:error, "Invalid email"}
    String.length(attrs.name) == 0 ->
      {:error, "Name required"}
    true ->
      {:ok, attrs}
  end
end

defp insert_user({:error, _} = error), do: error
defp insert_user({:ok, attrs}), do: Repo.insert(struct(User, attrs))
```

---

## 3. Replace Flags with Separate Functions

```elixir
# Before: boolean flag changes behaviour
def list_articles(include_drafts \\ false) do
  query = from(a in Article, preload: [:author])
  query = if include_drafts do
    query
  else
    where(query, [a], a.status == "published")
  end
  Repo.all(query)
end

# After: separate well-named functions
def list_published_articles do
  from(a in Article,
    where: a.status == "published",
    preload: [:author]
  )
  |> Repo.all()
end

def list_all_articles do
  from(a in Article, preload: [:author])
  |> Repo.all()
end

def list_articles_for_author(author_id) do
  from(a in Article,
    where: a.author_id == ^author_id,
    preload: [:author]
  )
  |> Repo.all()
end
```

---

## 4. Protocol-based Polymorphism

```elixir
# Before: type-checking with is_map/is_struct
def to_notification(resource) do
  if is_struct(resource, Comment) do
    %{type: "comment", message: "New comment: #{resource.body}"}
  else
    if is_struct(resource, Like) do
      %{type: "like", message: "#{resource.user.name} liked your post"}
    else
      {:error, :unknown_type}
    end
  end
end

# After: Protocol
defprotocol MyApp.Notifiable do
  def to_notification(resource)
end

defimpl MyApp.Notifiable, for: MyApp.Comment do
  def to_notification(%{body: body, author: author}) do
    %{type: "comment", message: "#{author.name} commented: #{String.slice(body, 0, 50)}"}
  end
end

defimpl MyApp.Notifiable, for: MyApp.Like do
  def to_notification(%{user: user}) do
    %{type: "like", message: "#{user.name} liked your post"}
  end
end

defimpl MyApp.Notifiable, for: MyApp.Follow do
  def to_notification(%{follower: follower}) do
    %{type: "follow", message: "#{follower.name} started following you"}
  end
end

# Usage: polymorphic, no type checking
MyApp.Notifiable.to_notification(comment)
MyApp.Notifiable.to_notification(like)
```

---

## 5. Reducer Pattern for Complex State

```elixir
# Before: state mutation in multiple places
def handle_event("checkout", _params, socket) do
  cart = socket.assigns.cart
  user = socket.assigns.current_user
  items = cart.items

  if length(items) == 0 do
    {:noreply, put_flash(socket, :error, "Cart is empty")}
  else
    subtotal = Enum.sum(Enum.map(items, & &1.price * &1.quantity))
    tax = subtotal * 0.07
    total = subtotal + tax

    case MyApp.Orders.create(user, items, total) do
      {:ok, order} ->
        socket = assign(socket, :cart, %Cart{items: []})
        socket = assign(socket, :order, order)
        socket = put_flash(socket, :info, "Order created!")
        {:noreply, socket}
      {:error, _} ->
        {:noreply, put_flash(socket, :error, "Failed")}
    end
  end
end

# After: clear pipeline with error handling
def handle_event("checkout", _params, socket) do
  result = socket.assigns
  |> build_checkout_data()
  |> validate_checkout()
  |> process_order()

  case result do
    {:ok, order} ->
      {:noreply,
       socket
       |> assign(:cart, %Cart{items: []})
       |> assign(:order, order)
       |> put_flash(:info, "Order #{order.id} created!")}

    {:error, message} ->
      {:noreply, put_flash(socket, :error, message)}
  end
end

defp build_checkout_data(%{cart: cart, current_user: user}) do
  subtotal = Enum.sum(Enum.map(cart.items, & &1.price * &1.quantity))
  %{user: user, items: cart.items, subtotal: subtotal, tax: subtotal * 0.07}
end

defp validate_checkout(%{items: []}), do: {:error, "Cart is empty"}
defp validate_checkout(%{subtotal: s}) when s <= 0, do: {:error, "Invalid total"}
defp validate_checkout(data), do: {:ok, data}

defp process_order({:error, _} = err), do: err
defp process_order({:ok, %{user: user, items: items, subtotal: sub, tax: tax}}) do
  MyApp.Orders.create(user, items, sub + tax)
end
```

---

## สรุป

```
Refactoring Principles:
├── Pattern match instead of if/else chains
├── Small focused functions (do one thing)
├── Pipeline: clear data transformation
└── Protocols: polymorphism without type checks

Code Smells:
├── Boolean flag parameter → separate functions
├── Deep nesting → with / pattern match
├── Long function → extract + pipeline
└── is_struct checks → defprotocol

Benefits:
├── Easier to test (small functions)
├── Easier to extend (protocols/behaviours)
├── Easier to read (declarative pipelines)
└── Compiler catches missing cases
```

---

*ก่อนหน้า: [Part 79](part_79.md) | ต่อไป: [Part 81 - WebSockets Advanced](part_81.md)*
