# Part 30: Ecto Queries และ Changesets

## เป้าหมายการเรียนรู้

หลังจากจบ Part นี้ คุณจะสามารถ:
- ใช้ Changeset สำหรับ cast และ validate data ได้
- เขียน Ecto queries ด้วย `from`, `where`, `select`, `join`, `preload` ได้
- ใช้ Repo operations ได้
- สร้าง User registration พร้อม validations เป็นตัวอย่างจริง

---

## 1. Changeset

Changeset คือ data structure ที่ใช้สำหรับ:
- **Cast**: แปลง external data เป็น internal types
- **Validate**: ตรวจสอบ data
- **Track changes**: ติดตามว่า field ไหนเปลี่ยน
- **Collect errors**: รวบรวม validation errors

```elixir
defmodule Blog.User do
  use Ecto.Schema
  import Ecto.Changeset

  schema "users" do
    field :name, :string
    field :email, :string
    field :age, :integer
    field :role, :string, default: "user"
    field :password, :string, virtual: true
    field :password_hash, :string

    timestamps()
  end

  def changeset(user, attrs) do
    user
    |> cast(attrs, [:name, :email, :age, :role])
    |> validate_required([:name, :email])
    |> validate_length(:name, min: 2, max: 100)
    |> validate_format(:email, ~r/^[^\s@]+@[^\s@]+\.[^\s@]+$/)
    |> validate_number(:age, greater_than: 0, less_than: 150)
    |> validate_inclusion(:role, ["user", "admin", "moderator"])
    |> unique_constraint(:email)
  end
end
```

---

## 2. cast/3 และ validate_*

### cast/3

```elixir
# cast(changeset_or_data, attrs, allowed_fields)
# เฉพาะ fields ใน allowed_fields เท่านั้นที่จะถูก cast

def changeset(user, attrs) do
  user
  |> cast(attrs, [:name, :email, :age])
  # :password_hash จะไม่ถูก cast แม้ attrs มีมันก็ตาม
end

# สร้าง changeset จาก struct หรือ existing changeset
user = %Blog.User{}
changeset = Blog.User.changeset(user, %{name: "Alice", email: "alice@example.com"})

IO.inspect(changeset.valid?)   # true หรือ false
IO.inspect(changeset.changes)  # %{name: "Alice", email: "alice@example.com"}
IO.inspect(changeset.errors)   # [] หรือ [field: {"message", opts}]
```

### Validation Functions

```elixir
def changeset(schema, attrs) do
  schema
  |> cast(attrs, [...])

  # Required
  |> validate_required([:name, :email])

  # Length
  |> validate_length(:name, min: 2, max: 100)
  |> validate_length(:password, min: 8)
  |> validate_length(:bio, max: 500, count: :graphemes)

  # Format (regex)
  |> validate_format(:email, ~r/^[^\s@]+@[^\s@]+\.[^\s@]+$/)
  |> validate_format(:phone, ~r/^\+?[0-9\s-]{10,15}$/, message: "invalid phone format")

  # Inclusion/Exclusion
  |> validate_inclusion(:role, ["admin", "user"])
  |> validate_exclusion(:username, ["admin", "root", "system"])

  # Number
  |> validate_number(:age, greater_than: 0)
  |> validate_number(:price, greater_than_or_equal_to: 0)
  |> validate_number(:score, less_than: 100)
  |> validate_number(:rating, greater_than: 0, less_than_or_equal_to: 5)

  # Acceptance (checkbox terms)
  |> validate_acceptance(:terms_of_service)

  # Confirmation (password confirm)
  |> validate_confirmation(:password, message: "passwords do not match")

  # Custom validation
  |> validate_change(:slug, fn field, value ->
    if String.contains?(value, " ") do
      [{field, "cannot contain spaces"}]
    else
      []
    end
  end)
end
```

### Constraints

Constraints เป็น database-level validations

```elixir
def changeset(schema, attrs) do
  schema
  |> cast(attrs, [...])

  # Unique constraint (ต้องมี unique index ใน DB)
  |> unique_constraint(:email)
  |> unique_constraint([:title, :author_id])  # composite unique

  # Foreign key constraint
  |> foreign_key_constraint(:user_id)
  |> foreign_key_constraint(:post_id, message: "post does not exist")

  # Check constraint
  |> check_constraint(:age, name: :age_must_be_positive,
      message: "age must be positive")

  # No assoc constraint (ป้องกัน delete parent ถ้ามี children)
  |> no_assoc_constraint(:posts, message: "must delete posts first")
end
```

### Custom Validation

```elixir
defmodule Blog.User do
  use Ecto.Schema
  import Ecto.Changeset

  def changeset(user, attrs) do
    user
    |> cast(attrs, [:username, :birthdate])
    |> validate_username()
    |> validate_adult()
  end

  defp validate_username(changeset) do
    validate_change(changeset, :username, fn _field, username ->
      cond do
        String.length(username) < 3 ->
          [{:username, "must be at least 3 characters"}]
        not String.match?(username, ~r/^[a-zA-Z0-9_]+$/) ->
          [{:username, "can only contain letters, numbers, and underscores"}]
        String.starts_with?(username, "_") ->
          [{:username, "cannot start with underscore"}]
        true ->
          []
      end
    end)
  end

  defp validate_adult(changeset) do
    validate_change(changeset, :birthdate, fn _field, birthdate ->
      today = Date.utc_today()
      age = Date.diff(today, birthdate) / 365

      if age < 18 do
        [{:birthdate, "must be at least 18 years old"}]
      else
        []
      end
    end)
  end
end
```

---

## 3. Ecto Queries

### Query DSL

```elixir
import Ecto.Query

# Basic query
query = from p in Blog.Post, select: p

# หรือใช้ string as source
query = from p in "posts", select: p
```

### where

```elixir
import Ecto.Query

# Simple where
from p in Post,
  where: p.is_published == true

# Multiple conditions (AND)
from p in Post,
  where: p.is_published == true and p.view_count > 100

# OR condition
from p in Post,
  where: p.is_published == true or p.author_id == ^user_id

# Dynamic values ใช้ ^ (pin operator)
user_id = 1
from p in Post,
  where: p.author_id == ^user_id

# IN clause
ids = [1, 2, 3, 4]
from p in Post,
  where: p.id in ^ids

# LIKE
from p in Post,
  where: like(p.title, ^"%elixir%")

# ILIKE (case-insensitive, PostgreSQL)
from p in Post,
  where: ilike(p.title, ^"%elixir%")

# IS NULL
from p in Post,
  where: is_nil(p.published_at)

# IS NOT NULL
from p in Post,
  where: not is_nil(p.published_at)
```

### select

```elixir
# Select all fields
from p in Post, select: p

# Select specific fields
from p in Post, select: {p.id, p.title}

# Select as map
from p in Post, select: %{id: p.id, title: p.title}

# Select with computation
from p in Post,
  select: %{
    id: p.id,
    title: p.title,
    word_count: fragment("array_length(string_to_array(?, ' '), 1)", p.body)
  }

# Select with struct_fields
from p in Post,
  select: struct(p, [:id, :title, :published_at])
```

### order_by, limit, offset

```elixir
# Order by
from p in Post,
  order_by: [desc: p.published_at]

from p in Post,
  order_by: [asc: p.title, desc: p.view_count]

# Dynamic order
direction = :desc
field = :published_at
from p in Post,
  order_by: [{^direction, ^field}]

# Limit and Offset
from p in Post,
  limit: 10,
  offset: 20

# Pagination helper
page = 3
per_page = 10
from p in Post,
  limit: ^per_page,
  offset: ^((page - 1) * per_page)
```

### join

```elixir
# INNER JOIN
from p in Post,
  join: u in User, on: p.author_id == u.id,
  select: {p, u}

# JOIN with association
from p in Post,
  join: u in assoc(p, :author),
  select: %{post: p, author_name: u.name}

# LEFT JOIN
from p in Post,
  left_join: c in Comment, on: c.post_id == p.id,
  group_by: p.id,
  select: {p, count(c.id)}

# Multiple JOINs
from p in Post,
  join: u in assoc(p, :author),
  left_join: cat in assoc(p, :category),
  where: p.is_published == true,
  select: %{
    title: p.title,
    author: u.name,
    category: cat.name
  }

# JOIN กับ conditions
from p in Post,
  join: c in Comment,
    on: c.post_id == p.id and c.is_approved == true,
  group_by: p.id,
  select: {p, count(c.id)}
```

### preload

```elixir
# Preload associations

# ใน query
from p in Post,
  preload: [:author, :tags, :category]

# Preload nested
from p in Post,
  preload: [author: [:profile], comments: [:user]]

# Preload กับ query
comments_query = from c in Comment,
  where: c.is_approved == true,
  order_by: [asc: c.inserted_at]

from p in Post,
  preload: [comments: ^comments_query]

# Preload หลัง query
posts = Repo.all(from p in Post, where: p.is_published == true)
posts = Repo.preload(posts, [:author, :tags])

# Preload เดียว
post = Repo.get!(Post, 1)
post = Repo.preload(post, :comments)
```

### group_by และ aggregate

```elixir
# Count
from p in Post,
  group_by: p.category_id,
  select: {p.category_id, count(p.id)}

# Sum
from p in Post,
  select: sum(p.view_count)

# Average
from p in Post,
  select: avg(p.view_count)

# Max/Min
from p in Post,
  select: {max(p.view_count), min(p.view_count)}

# Having
from p in Post,
  group_by: p.author_id,
  having: count(p.id) > 5,
  select: {p.author_id, count(p.id)}
```

---

## 4. Repo Operations

```elixir
alias Blog.Repo
alias Blog.Post

# ============= Read =============

# Get by primary key
post = Repo.get(Post, 1)        # returns nil if not found
post = Repo.get!(Post, 1)       # raises if not found

# Get by field
post = Repo.get_by(Post, slug: "my-post")
post = Repo.get_by!(Post, slug: "my-post")

# Get all
posts = Repo.all(Post)
posts = Repo.all(from p in Post, where: p.is_published == true)

# Get first/one
post = Repo.one(from p in Post, limit: 1)
post = Repo.one!(from p in Post, limit: 1)  # raises if not exactly 1

# Aggregate
count = Repo.aggregate(Post, :count)
total_views = Repo.aggregate(Post, :sum, :view_count)
avg_views = Repo.aggregate(Post, :avg, :view_count)

# Exists
Repo.exists?(from p in Post, where: p.slug == ^slug)

# Stream (สำหรับ large datasets)
Repo.stream(from p in Post, where: p.is_published == true)
|> Stream.each(fn post -> process(post) end)
|> Stream.run()

# ============= Write =============

# Insert
{:ok, post} = Repo.insert(changeset)
{:error, changeset} = Repo.insert(invalid_changeset)

post = Repo.insert!(changeset)  # raises on error

# Insert with options
{:ok, post} = Repo.insert(changeset,
  returning: [:id, :inserted_at],  # return specific fields
  on_conflict: :nothing,           # ignore if conflict
  conflict_target: [:slug]         # conflict on slug
)

# Update
{:ok, post} = Repo.update(changeset)
post = Repo.update!(changeset)

# Delete
{:ok, post} = Repo.delete(post)
post = Repo.delete!(post)

# Delete with query
Repo.delete_all(from p in Post, where: p.is_published == false)

# Update all
Repo.update_all(
  from(p in Post, where: p.is_published == false),
  set: [view_count: 0]
)

# Insert or update (upsert)
Repo.insert(changeset,
  on_conflict: {:replace, [:title, :body, :updated_at]},
  conflict_target: [:slug]
)
```

---

## 5. Dynamic Queries

```elixir
defmodule Blog.Posts do
  import Ecto.Query

  def list_posts(params \\ %{}) do
    Post
    |> base_query()
    |> maybe_filter_category(params["category"])
    |> maybe_filter_tag(params["tag"])
    |> maybe_search(params["q"])
    |> maybe_filter_author(params["author_id"])
    |> apply_sorting(params["sort"])
    |> paginate(params["page"], params["per_page"])
    |> Repo.all()
  end

  defp base_query(query) do
    from p in query,
      where: p.is_published == true,
      preload: [:author, :category, :tags]
  end

  defp maybe_filter_category(query, nil), do: query
  defp maybe_filter_category(query, category_slug) do
    from p in query,
      join: c in assoc(p, :category),
      where: c.slug == ^category_slug
  end

  defp maybe_filter_tag(query, nil), do: query
  defp maybe_filter_tag(query, tag_slug) do
    from p in query,
      join: t in assoc(p, :tags),
      where: t.slug == ^tag_slug
  end

  defp maybe_search(query, nil), do: query
  defp maybe_search(query, ""), do: query
  defp maybe_search(query, search) do
    pattern = "%#{search}%"
    from p in query,
      where: ilike(p.title, ^pattern) or ilike(p.excerpt, ^pattern)
  end

  defp maybe_filter_author(query, nil), do: query
  defp maybe_filter_author(query, author_id) do
    from p in query,
      where: p.author_id == ^author_id
  end

  defp apply_sorting(query, "oldest") do
    from p in query, order_by: [asc: p.published_at]
  end
  defp apply_sorting(query, "views") do
    from p in query, order_by: [desc: p.view_count]
  end
  defp apply_sorting(query, _) do
    from p in query, order_by: [desc: p.published_at]
  end

  defp paginate(query, page, per_page) do
    page = (page || "1") |> String.to_integer() |> max(1)
    per_page = (per_page || "10") |> String.to_integer() |> min(100)

    from p in query,
      limit: ^per_page,
      offset: ^((page - 1) * per_page)
  end
end
```

---

## 6. Transactions

```elixir
defmodule Blog.Posts do
  alias Blog.Repo

  def create_post_with_tags(post_attrs, tag_names) do
    Repo.transaction(fn ->
      # สร้าง post
      post_changeset = Post.changeset(%Post{}, post_attrs)

      post = case Repo.insert(post_changeset) do
        {:ok, post} -> post
        {:error, changeset} -> Repo.rollback(changeset)
      end

      # สร้างหรือหา tags
      tags = Enum.map(tag_names, fn name ->
        case Repo.get_by(Tag, name: name) do
          nil ->
            case Repo.insert(Tag.changeset(%Tag{}, %{name: name, slug: slugify(name)})) do
              {:ok, tag} -> tag
              {:error, _} -> Repo.rollback("Failed to create tag: #{name}")
            end
          tag ->
            tag
        end
      end)

      # Associate tags กับ post
      post = post
      |> Repo.preload(:tags)
      |> Ecto.Changeset.change()
      |> Ecto.Changeset.put_assoc(:tags, tags)
      |> Repo.update!()

      post
    end)
  end

  # Multi.new สำหรับ complex transactions
  def transfer_post_ownership(post_id, new_author_id) do
    Ecto.Multi.new()
    |> Ecto.Multi.run(:post, fn repo, _ ->
      case repo.get(Post, post_id) do
        nil -> {:error, :post_not_found}
        post -> {:ok, post}
      end
    end)
    |> Ecto.Multi.run(:new_author, fn repo, _ ->
      case repo.get(User, new_author_id) do
        nil -> {:error, :user_not_found}
        user -> {:ok, user}
      end
    end)
    |> Ecto.Multi.update(:update_post, fn %{post: post} ->
      Post.changeset(post, %{author_id: new_author_id})
    end)
    |> Ecto.Multi.insert(:notification, fn %{post: post, new_author: author} ->
      Notification.changeset(%Notification{}, %{
        type: "post_transferred",
        user_id: author.id,
        data: %{post_id: post.id, post_title: post.title}
      })
    end)
    |> Repo.transaction()
  end
end
```

---

## 7. ตัวอย่างจริง: User Registration พร้อม Validations

```elixir
# lib/blog/accounts.ex
defmodule Blog.Accounts do
  import Ecto.Query
  alias Blog.Repo
  alias Blog.Accounts.{User, UserToken}

  # ============= Registration =============

  def register_user(attrs) do
    %User{}
    |> User.registration_changeset(attrs)
    |> Repo.insert()
  end

  # ============= Authentication =============

  def authenticate_user(email, password) do
    user = Repo.get_by(User, email: email)

    cond do
      user == nil ->
        # Timing attack prevention: still check password
        Bcrypt.no_user_verify()
        {:error, :invalid_credentials}

      not Bcrypt.verify_pass(password, user.password_hash) ->
        {:error, :invalid_credentials}

      not user.is_active ->
        {:error, :account_disabled}

      not user.confirmed ->
        {:error, :email_not_confirmed}

      true ->
        # Update last sign in
        Repo.update!(User.sign_in_changeset(user))
        {:ok, user}
    end
  end

  # ============= Queries =============

  def get_user(id), do: Repo.get(User, id)

  def get_user!(id), do: Repo.get!(User, id)

  def get_user_by_email(email) when is_binary(email) do
    Repo.get_by(User, email: String.downcase(email))
  end

  def list_users(opts \\ []) do
    page = Keyword.get(opts, :page, 1)
    per_page = Keyword.get(opts, :per_page, 20)
    role = Keyword.get(opts, :role)
    search = Keyword.get(opts, :search)

    User
    |> maybe_filter_role(role)
    |> maybe_search_users(search)
    |> order_by([u], desc: u.inserted_at)
    |> paginate_query(page, per_page)
    |> Repo.all()
  end

  defp maybe_filter_role(query, nil), do: query
  defp maybe_filter_role(query, role) do
    where(query, [u], u.role == ^role)
  end

  defp maybe_search_users(query, nil), do: query
  defp maybe_search_users(query, search) do
    pattern = "%#{search}%"
    where(query, [u], ilike(u.name, ^pattern) or ilike(u.email, ^pattern))
  end

  defp paginate_query(query, page, per_page) do
    offset = (page - 1) * per_page
    query |> limit(^per_page) |> offset(^offset)
  end
end

# lib/blog/accounts/user.ex - Full Schema
defmodule Blog.Accounts.User do
  use Ecto.Schema
  import Ecto.Changeset

  @roles [:reader, :author, :admin]

  schema "users" do
    field :name,          :string
    field :email,         :string
    field :password_hash, :string
    field :password,      :string, virtual: true
    field :role,          Ecto.Enum, values: @roles, default: :reader
    field :is_active,     :boolean, default: true
    field :confirmed,     :boolean, default: false
    field :confirmed_at,  :utc_datetime
    field :last_sign_in_at, :utc_datetime

    has_many :posts, Blog.Posts.Post, foreign_key: :author_id
    has_many :comments, Blog.Comments.Comment

    timestamps(type: :utc_datetime)
  end

  # ===== Changesets =====

  def registration_changeset(user, attrs) do
    user
    |> cast(attrs, [:name, :email, :password])
    |> validate_name()
    |> validate_email()
    |> validate_password()
    |> hash_password()
  end

  def profile_changeset(user, attrs) do
    user
    |> cast(attrs, [:name])
    |> validate_name()
  end

  def confirm_email_changeset(user) do
    now = DateTime.utc_now() |> DateTime.truncate(:second)
    change(user, confirmed: true, confirmed_at: now)
  end

  def sign_in_changeset(user) do
    now = DateTime.utc_now() |> DateTime.truncate(:second)
    change(user, last_sign_in_at: now)
  end

  def change_role_changeset(user, role) do
    user
    |> cast(%{role: role}, [:role])
    |> validate_inclusion(:role, @roles)
  end

  def deactivate_changeset(user) do
    change(user, is_active: false)
  end

  # ===== Private Validations =====

  defp validate_name(changeset) do
    changeset
    |> validate_required([:name])
    |> validate_length(:name, min: 2, max: 100)
    |> validate_format(:name, ~r/^[[:alnum:]\s\-\.]+$/u,
        message: "can only contain letters, numbers, spaces, hyphens, and periods")
  end

  defp validate_email(changeset) do
    changeset
    |> validate_required([:email])
    |> validate_format(:email, ~r/^[^\s@]+@[^\s@]+\.[^\s@]+$/,
        message: "must be a valid email address")
    |> validate_length(:email, max: 160)
    |> update_change(:email, &String.downcase/1)
    |> unique_constraint(:email,
        message: "This email is already registered. Try logging in instead.")
  end

  defp validate_password(changeset) do
    changeset
    |> validate_required([:password])
    |> validate_length(:password, min: 8, max: 72,
        message: "must be between 8 and 72 characters")
    |> validate_format(:password, ~r/[a-z]/,
        message: "must contain at least one lowercase letter")
    |> validate_format(:password, ~r/[A-Z]/,
        message: "must contain at least one uppercase letter")
    |> validate_format(:password, ~r/[0-9]/,
        message: "must contain at least one number")
    |> validate_password_not_common()
  end

  defp validate_password_not_common(changeset) do
    common_passwords = ["password", "12345678", "qwerty123", "password1"]

    validate_change(changeset, :password, fn _field, password ->
      if String.downcase(password) in common_passwords do
        [{:password, "is too common, please choose a stronger password"}]
      else
        []
      end
    end)
  end

  defp hash_password(%Ecto.Changeset{valid?: true, changes: %{password: password}} = changeset) do
    changeset
    |> put_change(:password_hash, Bcrypt.hash_pwd_salt(password))
    |> delete_change(:password)  # ลบ plain password หลัง hash
  end
  defp hash_password(changeset), do: changeset
end
```

### Context สำหรับ User Registration

```elixir
# ใน Controller
defmodule BlogWeb.UserController do
  use BlogWeb, :controller

  alias Blog.Accounts

  def new(conn, _params) do
    changeset = Accounts.change_user_registration()
    render(conn, :new, changeset: changeset)
  end

  def create(conn, %{"user" => user_params}) do
    case Accounts.register_user(user_params) do
      {:ok, user} ->
        # ส่ง confirmation email
        Blog.Mailer.send_confirmation_email(user)

        conn
        |> put_flash(:info, "Account created! Please check your email to confirm.")
        |> redirect(to: ~p"/login")

      {:error, %Ecto.Changeset{} = changeset} ->
        render(conn, :new, changeset: changeset)
    end
  end
end
```

---

## 8. Error Handling และ Display

```elixir
# Format changeset errors สำหรับ JSON API
defmodule BlogWeb.ErrorHelpers do
  def translate_error({msg, opts}) do
    Regex.replace(~r"%{(\w+)}", msg, fn _, key ->
      opts
      |> Keyword.get(String.to_existing_atom(key), key)
      |> to_string()
    end)
  end

  def format_changeset_errors(changeset) do
    Ecto.Changeset.traverse_errors(changeset, &translate_error/1)
  end
end

# ใน template .heex อ่าน errors:
# @changeset.errors => [{:email, {"has already been taken", []}}]

# ใช้ core_components .error:
# <.input field={@form[:email]} type="email" label="Email" />
# จะ show error อัตโนมัติ
```

---

## 9. Exercises

### Exercise 1: สร้าง Post Query ที่สมบูรณ์

สร้าง query function ที่รองรับ filter, search, sort, และ pagination

**เฉลย:**

```elixir
defmodule Blog.Posts do
  import Ecto.Query
  alias Blog.Repo
  alias Blog.Posts.Post

  def list_posts(opts \\ []) do
    {posts, total} = build_query(opts)
    {posts, total}
  end

  def build_query(opts) do
    base = from p in Post,
      where: p.is_published == true,
      preload: [:author, :category, :tags]

    query = base
    |> apply_search(Keyword.get(opts, :search))
    |> apply_category(Keyword.get(opts, :category_slug))
    |> apply_tag(Keyword.get(opts, :tag_slug))
    |> apply_author(Keyword.get(opts, :author_id))
    |> apply_date_range(Keyword.get(opts, :from_date), Keyword.get(opts, :to_date))
    |> apply_sort(Keyword.get(opts, :sort, "newest"))

    total = Repo.aggregate(query, :count)

    paginated = query
    |> apply_pagination(
      Keyword.get(opts, :page, 1),
      Keyword.get(opts, :per_page, 10)
    )

    {Repo.all(paginated), total}
  end

  defp apply_search(query, nil), do: query
  defp apply_search(query, ""), do: query
  defp apply_search(query, search) do
    pattern = "%#{search}%"
    where(query, [p], ilike(p.title, ^pattern) or ilike(p.excerpt, ^pattern) or ilike(p.body, ^pattern))
  end

  defp apply_category(query, nil), do: query
  defp apply_category(query, slug) do
    from p in query,
      join: c in assoc(p, :category),
      where: c.slug == ^slug
  end

  defp apply_tag(query, nil), do: query
  defp apply_tag(query, slug) do
    from p in query,
      join: t in assoc(p, :tags),
      where: t.slug == ^slug,
      distinct: true
  end

  defp apply_author(query, nil), do: query
  defp apply_author(query, author_id) do
    where(query, [p], p.author_id == ^author_id)
  end

  defp apply_date_range(query, nil, nil), do: query
  defp apply_date_range(query, from_date, nil) do
    where(query, [p], p.published_at >= ^from_date)
  end
  defp apply_date_range(query, nil, to_date) do
    where(query, [p], p.published_at <= ^to_date)
  end
  defp apply_date_range(query, from_date, to_date) do
    where(query, [p],
      p.published_at >= ^from_date and p.published_at <= ^to_date
    )
  end

  defp apply_sort(query, "newest"), do: order_by(query, [p], desc: p.published_at)
  defp apply_sort(query, "oldest"), do: order_by(query, [p], asc: p.published_at)
  defp apply_sort(query, "popular"), do: order_by(query, [p], desc: p.view_count)
  defp apply_sort(query, "title"), do: order_by(query, [p], asc: p.title)
  defp apply_sort(query, _), do: order_by(query, [p], desc: p.published_at)

  defp apply_pagination(query, page, per_page) do
    page = max(page, 1)
    per_page = min(per_page, 100)
    offset = (page - 1) * per_page

    query |> limit(^per_page) |> offset(^offset)
  end
end

# ใช้งาน
{posts, total} = Blog.Posts.list_posts(
  search: "elixir",
  category_slug: "programming",
  sort: "popular",
  page: 2,
  per_page: 5
)

IO.puts("Found #{total} posts, showing page 2")
IO.inspect(Enum.map(posts, & &1.title))
```

### Exercise 2: Changeset สำหรับ Post

สร้าง changeset ที่มี validation ครบถ้วนและ unique slug generation

**เฉลย:**

```elixir
defmodule Blog.Posts.Post do
  use Ecto.Schema
  import Ecto.Changeset
  alias Blog.Repo

  schema "posts" do
    field :title, :string
    field :slug, :string
    field :body, :text
    field :excerpt, :string
    field :is_published, :boolean, default: false
    field :published_at, :utc_datetime
    field :view_count, :integer, default: 0

    belongs_to :author, Blog.Accounts.User
    belongs_to :category, Blog.Categories.Category
    many_to_many :tags, Blog.Tags.Tag, join_through: "post_tags", on_replace: :delete

    timestamps(type: :utc_datetime)
  end

  def changeset(post, attrs) do
    post
    |> cast(attrs, [:title, :slug, :body, :excerpt, :is_published,
                    :published_at, :author_id, :category_id])
    |> validate_required([:title, :body, :author_id])
    |> validate_length(:title, min: 5, max: 255)
    |> validate_length(:excerpt, max: 500)
    |> validate_length(:body, min: 50)
    |> generate_unique_slug()
    |> validate_slug_format()
    |> unique_constraint(:slug)
    |> foreign_key_constraint(:author_id)
    |> foreign_key_constraint(:category_id)
    |> set_published_at()
    |> maybe_put_tags(attrs)
  end

  defp generate_unique_slug(changeset) do
    if get_change(changeset, :slug) do
      changeset
    else
      case get_change(changeset, :title) do
        nil -> changeset
        title ->
          base_slug = title
          |> String.downcase()
          |> String.replace(~r/[^a-z0-9\s-]/, "")
          |> String.replace(~r/\s+/, "-")
          |> String.trim("-")

          unique_slug = ensure_unique_slug(base_slug, get_field(changeset, :id))
          put_change(changeset, :slug, unique_slug)
      end
    end
  end

  defp ensure_unique_slug(slug, post_id, suffix \\ 0) do
    candidate = if suffix == 0, do: slug, else: "#{slug}-#{suffix}"

    query = from p in __MODULE__,
      where: p.slug == ^candidate and (is_nil(^post_id) or p.id != ^post_id)

    if Repo.exists?(query) do
      ensure_unique_slug(slug, post_id, suffix + 1)
    else
      candidate
    end
  end

  defp validate_slug_format(changeset) do
    validate_format(changeset, :slug, ~r/^[a-z0-9-]+$/,
      message: "can only contain lowercase letters, numbers, and hyphens")
  end

  defp set_published_at(changeset) do
    case {get_change(changeset, :is_published), get_field(changeset, :published_at)} do
      {true, nil} -> put_change(changeset, :published_at, DateTime.utc_now() |> DateTime.truncate(:second))
      _ -> changeset
    end
  end

  defp maybe_put_tags(changeset, %{"tag_ids" => tag_ids}) when is_list(tag_ids) do
    tags = Blog.Tags.get_tags_by_ids(tag_ids)
    put_assoc(changeset, :tags, tags)
  end
  defp maybe_put_tags(changeset, _attrs), do: changeset
end
```

---

## สรุป

```
Changeset:
├── cast(struct, attrs, allowed_fields)
├── Validations:
│   ├── validate_required/2
│   ├── validate_length/3
│   ├── validate_format/3
│   ├── validate_inclusion/3
│   ├── validate_number/3
│   ├── validate_confirmation/2
│   └── validate_change/3 (custom)
├── Constraints (DB-level):
│   ├── unique_constraint/2
│   ├── foreign_key_constraint/2
│   └── check_constraint/3
└── changeset.valid?, .errors, .changes

Ecto Query:
├── from p in Post, ...
├── where: p.field == ^value
├── select: struct_or_map
├── order_by: [desc: field]
├── limit: n, offset: n
├── join: assoc(p, :relation)
├── preload: [:relation, nested: [:sub]]
└── group_by, having, aggregate

Repo Operations:
├── Repo.get/3, Repo.get!/3
├── Repo.get_by/3
├── Repo.all/2
├── Repo.one/2, Repo.one!/2
├── Repo.insert/2, Repo.insert!/2
├── Repo.update/2, Repo.update!/2
├── Repo.delete/2, Repo.delete!/2
├── Repo.update_all/3
├── Repo.delete_all/2
└── Repo.transaction/2
```

---

*ก่อนหน้า: [Part 29 - Ecto Schema และ Migrations](part_29.md) | ต่อไป: Part 31*
