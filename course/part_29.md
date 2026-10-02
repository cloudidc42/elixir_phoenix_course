# Part 29: Ecto Schema และ Migrations

## เป้าหมายการเรียนรู้

หลังจากจบ Part นี้ คุณจะสามารถ:
- กำหนด Schema ด้วย Ecto ได้
- สร้างและรัน Migrations ได้
- กำหนด Associations (belongs_to, has_many, has_one, many_to_many) ได้
- ใช้ Embedded Schemas ได้
- ออกแบบ Blog schema ที่สมบูรณ์ได้

---

## 1. Ecto คืออะไร?

Ecto เป็น database library สำหรับ Elixir ที่ประกอบด้วย:
- **Schema**: define data structure และ types
- **Changeset**: validate และ cast data
- **Query**: type-safe database queries
- **Repo**: database access (insert, update, delete, query)
- **Migrations**: database schema versioning

---

## 2. Schema พื้นฐาน

```elixir
defmodule Blog.Post do
  use Ecto.Schema

  schema "posts" do
    # Field types
    field :title,        :string
    field :body,         :text
    field :excerpt,      :string
    field :slug,         :string
    field :is_published, :boolean, default: false
    field :view_count,   :integer, default: 0
    field :published_at, :utc_datetime
    field :cover_image,  :string

    # Virtual field (ไม่ได้ save ลง DB)
    field :reading_time, :integer, virtual: true

    timestamps()  # inserted_at, updated_at
  end
end
```

### Field Types

```elixir
schema "examples" do
  # Text types
  field :name,        :string       # VARCHAR
  field :description, :text         # TEXT
  field :data,        :binary       # BYTEA

  # Number types
  field :count,       :integer      # INTEGER
  field :score,       :float        # FLOAT
  field :price,       :decimal      # DECIMAL
  field :big_id,      :id           # BIGINT (for IDs)

  # Boolean
  field :active,      :boolean

  # Date/Time
  field :birthday,    :date
  field :started_at,  :time
  field :created_at,  :naive_datetime      # ไม่มี timezone
  field :published_at, :utc_datetime       # UTC timezone
  field :event_at,    :utc_datetime_usec   # microsecond precision

  # Misc
  field :metadata,    :map          # JSON object
  field :tags,        {:array, :string}  # Array
  field :status,      Ecto.Enum, values: [:draft, :published, :archived]

  # UUID primary key
  # field :id, :binary_id, primary_key: true

  timestamps()
end
```

### Primary Key และ Custom Schema

```elixir
defmodule Blog.Tag do
  use Ecto.Schema

  # ใช้ UUID เป็น primary key
  @primary_key {:id, :binary_id, autogenerate: true}
  @foreign_key_type :binary_id

  schema "tags" do
    field :name, :string
    field :slug, :string
    field :color, :string, default: "#3B82F6"

    many_to_many :posts, Blog.Post, join_through: "post_tags"

    timestamps()
  end
end

# Schema ที่ไม่มี timestamps
defmodule Blog.PostView do
  use Ecto.Schema

  @timestamps_opts false

  schema "post_views" do
    field :post_id, :integer
    field :user_ip, :string
    field :viewed_at, :utc_datetime
  end
end
```

---

## 3. Migrations

Migration เป็นไฟล์ที่กำหนดการเปลี่ยนแปลง database schema

### สร้าง Migration

```bash
# สร้าง migration
mix ecto.gen.migration create_posts
# สร้างไฟล์: priv/repo/migrations/20240101000000_create_posts.exs
```

### Migration พื้นฐาน

```elixir
# priv/repo/migrations/20240101000000_create_posts.exs
defmodule Blog.Repo.Migrations.CreatePosts do
  use Ecto.Migration

  def change do
    create table(:posts) do
      add :title,        :string,   null: false, size: 255
      add :body,         :text
      add :excerpt,      :string,   size: 500
      add :slug,         :string,   null: false, size: 100
      add :is_published, :boolean,  default: false, null: false
      add :view_count,   :integer,  default: 0, null: false
      add :published_at, :utc_datetime
      add :cover_image,  :string

      timestamps(type: :utc_datetime)
    end

    # สร้าง index
    create unique_index(:posts, [:slug])
    create index(:posts, [:is_published])
    create index(:posts, [:published_at])
  end
end
```

### Migration Operations

```elixir
defmodule Blog.Repo.Migrations.AlterPosts do
  use Ecto.Migration

  def change do
    # เพิ่ม column
    alter table(:posts) do
      add :author_id, references(:users, on_delete: :nilify_all)
      add :tags, {:array, :string}, default: []
      add :metadata, :map, default: %{}
    end

    # แก้ไข column
    alter table(:posts) do
      modify :excerpt, :string, size: 1000  # เปลี่ยน size
      modify :view_count, :bigint           # เปลี่ยน type
    end

    # ลบ column
    alter table(:posts) do
      remove :old_column
      remove :another_column, :string, null: false  # ระบุ type สำหรับ rollback
    end

    # เพิ่ม index
    create index(:posts, [:author_id])
    create unique_index(:posts, [:title, :author_id])

    # Full-text search index (PostgreSQL)
    execute(
      "CREATE INDEX posts_search_idx ON posts USING GIN (to_tsvector('english', title || ' ' || COALESCE(body, '')))",
      "DROP INDEX IF EXISTS posts_search_idx"
    )
  end
end
```

### Up/Down Migrations

```elixir
defmodule Blog.Repo.Migrations.AddStatusToUsers do
  use Ecto.Migration

  # change = up + down อัตโนมัติ
  # ใช้ up/down เมื่อต้องการ control rollback

  def up do
    execute "ALTER TABLE users ADD COLUMN status VARCHAR(50) DEFAULT 'active'"
    execute "UPDATE users SET status = 'active'"
    execute "ALTER TABLE users ALTER COLUMN status SET NOT NULL"
  end

  def down do
    execute "ALTER TABLE users DROP COLUMN status"
  end
end
```

---

## 4. Associations

### belongs_to

```elixir
defmodule Blog.Post do
  use Ecto.Schema

  schema "posts" do
    field :title, :string
    field :body, :text

    # posts.author_id -> users.id
    belongs_to :author, Blog.User

    # Custom foreign key
    belongs_to :category, Blog.Category,
      foreign_key: :category_id,
      type: :integer

    timestamps()
  end
end

# Migration สำหรับ belongs_to
defmodule Blog.Repo.Migrations.AddAuthorToPosts do
  use Ecto.Migration

  def change do
    alter table(:posts) do
      add :author_id, references(:users, on_delete: :nilify_all)
      add :category_id, references(:categories, on_delete: :nilify_all)
    end

    create index(:posts, [:author_id])
    create index(:posts, [:category_id])
  end
end
```

### has_many

```elixir
defmodule Blog.User do
  use Ecto.Schema

  schema "users" do
    field :name, :string
    field :email, :string

    # user has many posts
    has_many :posts, Blog.Post, foreign_key: :author_id

    # user has many comments
    has_many :comments, Blog.Comment

    # through association
    has_many :commented_posts, through: [:comments, :post]

    timestamps()
  end
end
```

### has_one

```elixir
defmodule Blog.User do
  use Ecto.Schema

  schema "users" do
    field :name, :string

    # user has one profile
    has_one :profile, Blog.UserProfile

    timestamps()
  end
end

defmodule Blog.UserProfile do
  use Ecto.Schema

  schema "user_profiles" do
    field :bio, :text
    field :website, :string
    field :twitter, :string
    field :avatar, :string
    field :location, :string

    belongs_to :user, Blog.User

    timestamps()
  end
end

# Migration
defmodule Blog.Repo.Migrations.CreateUserProfiles do
  use Ecto.Migration

  def change do
    create table(:user_profiles) do
      add :bio, :text
      add :website, :string
      add :twitter, :string
      add :avatar, :string
      add :location, :string
      add :user_id, references(:users, on_delete: :delete_all), null: false

      timestamps(type: :utc_datetime)
    end

    create unique_index(:user_profiles, [:user_id])
  end
end
```

### many_to_many

```elixir
defmodule Blog.Post do
  use Ecto.Schema

  schema "posts" do
    field :title, :string

    # Post มีหลาย Tags, Tag มีหลาย Posts
    many_to_many :tags, Blog.Tag,
      join_through: "post_tags",  # join table
      on_replace: :delete         # ลบ join records เมื่อแทนที่

    timestamps()
  end
end

defmodule Blog.Tag do
  use Ecto.Schema

  schema "tags" do
    field :name, :string
    field :slug, :string

    many_to_many :posts, Blog.Post,
      join_through: "post_tags"

    timestamps()
  end
end

# Migration สำหรับ join table
defmodule Blog.Repo.Migrations.CreatePostTags do
  use Ecto.Migration

  def change do
    create table(:post_tags, primary_key: false) do
      add :post_id, references(:posts, on_delete: :delete_all), null: false
      add :tag_id, references(:tags, on_delete: :delete_all), null: false
      add :inserted_at, :utc_datetime, null: false
    end

    create unique_index(:post_tags, [:post_id, :tag_id])
    create index(:post_tags, [:tag_id])
  end
end
```

### on_delete Options

```elixir
# on_delete: :nothing         - ไม่ทำอะไร (default) - อาจ error
# on_delete: :delete_all      - ลบ children ด้วย (CASCADE)
# on_delete: :nilify_all      - set foreign key = null
# on_delete: :restrict        - block deletion ถ้ามี children

references(:users, on_delete: :delete_all)
references(:users, on_delete: :nilify_all)
references(:users, on_delete: :restrict)
```

---

## 5. Embedded Schemas

Embedded schemas เก็บ nested data ใน column เดียว (JSON)

```elixir
defmodule Blog.Post do
  use Ecto.Schema
  import Ecto.Changeset

  schema "posts" do
    field :title, :string
    field :body, :text

    # Embed single struct
    embeds_one :seo, Blog.Post.SEO, on_replace: :update

    # Embed list of structs
    embeds_many :sections, Blog.Post.Section, on_replace: :delete

    timestamps()
  end

  def changeset(post, attrs) do
    post
    |> cast(attrs, [:title, :body])
    |> cast_embed(:seo)
    |> cast_embed(:sections)
    |> validate_required([:title])
  end
end

defmodule Blog.Post.SEO do
  use Ecto.Schema
  import Ecto.Changeset

  @primary_key false

  embedded_schema do
    field :meta_title, :string
    field :meta_description, :string
    field :og_image, :string
    field :keywords, {:array, :string}, default: []
    field :canonical_url, :string
    field :no_index, :boolean, default: false
  end

  def changeset(seo, attrs) do
    seo
    |> cast(attrs, [:meta_title, :meta_description, :og_image, :keywords, :canonical_url, :no_index])
    |> validate_length(:meta_title, max: 60)
    |> validate_length(:meta_description, max: 160)
  end
end

defmodule Blog.Post.Section do
  use Ecto.Schema
  import Ecto.Changeset

  @primary_key {:id, :binary_id, autogenerate: true}

  embedded_schema do
    field :type, Ecto.Enum, values: [:text, :image, :code, :quote, :divider]
    field :content, :string
    field :caption, :string
    field :language, :string  # สำหรับ code block
    field :order, :integer, default: 0
  end

  def changeset(section, attrs) do
    section
    |> cast(attrs, [:type, :content, :caption, :language, :order])
    |> validate_required([:type])
    |> validate_inclusion(:type, [:text, :image, :code, :quote, :divider])
  end
end
```

```elixir
# การใช้งาน embedded schema
post = %Blog.Post{}

changeset = Blog.Post.changeset(post, %{
  title: "My Post",
  body: "Content here",
  seo: %{
    meta_title: "My Post - Blog",
    meta_description: "Read about my post",
    keywords: ["elixir", "phoenix"]
  },
  sections: [
    %{type: :text, content: "Introduction paragraph", order: 1},
    %{type: :code, content: "IO.puts(\"Hello\")", language: "elixir", order: 2},
    %{type: :image, content: "/images/demo.png", caption: "Demo image", order: 3}
  ]
})

{:ok, saved_post} = Blog.Repo.insert(changeset)

# Access embedded data
IO.puts(saved_post.seo.meta_title)    # "My Post - Blog"
IO.inspect(saved_post.sections)        # List of Section structs
```

---

## 6. ตัวอย่างจริง: Blog Schema ครบชุด

### Schema Files

```elixir
# lib/blog/accounts/user.ex
defmodule Blog.Accounts.User do
  use Ecto.Schema
  import Ecto.Changeset

  @valid_roles [:reader, :author, :admin]

  schema "users" do
    field :name,             :string
    field :email,            :string
    field :password_hash,    :string
    field :password,         :string, virtual: true
    field :role,             Ecto.Enum, values: @valid_roles, default: :reader
    field :confirmed_at,     :utc_datetime
    field :last_sign_in_at,  :utc_datetime
    field :avatar,           :string
    field :bio,              :text
    field :website,          :string

    has_many :posts, Blog.Posts.Post, foreign_key: :author_id
    has_many :comments, Blog.Comments.Comment
    has_one :profile, Blog.Accounts.UserProfile

    timestamps(type: :utc_datetime)
  end

  def registration_changeset(user, attrs) do
    user
    |> cast(attrs, [:name, :email, :password])
    |> validate_required([:name, :email, :password])
    |> validate_email()
    |> validate_password()
    |> put_password_hash()
  end

  def profile_changeset(user, attrs) do
    user
    |> cast(attrs, [:name, :bio, :website, :avatar])
    |> validate_required([:name])
    |> validate_length(:name, min: 2, max: 100)
    |> validate_length(:bio, max: 500)
    |> validate_url(:website)
  end

  defp validate_email(changeset) do
    changeset
    |> validate_required([:email])
    |> validate_format(:email, ~r/^[^\s@]+@[^\s@]+\.[^\s@]+$/, message: "must be a valid email")
    |> validate_length(:email, max: 160)
    |> unique_constraint(:email)
  end

  defp validate_password(changeset) do
    changeset
    |> validate_length(:password, min: 8, max: 72)
    |> validate_format(:password, ~r/[a-z]/, message: "must contain lowercase letter")
    |> validate_format(:password, ~r/[A-Z]/, message: "must contain uppercase letter")
    |> validate_format(:password, ~r/[0-9]/, message: "must contain a number")
  end

  defp put_password_hash(%Ecto.Changeset{valid?: true, changes: %{password: pwd}} = changeset) do
    change(changeset, password_hash: Bcrypt.hash_pwd_salt(pwd))
  end
  defp put_password_hash(changeset), do: changeset

  defp validate_url(changeset, field) do
    validate_change(changeset, field, fn _, url ->
      case URI.parse(url) do
        %URI{scheme: scheme} when scheme in ["http", "https"] -> []
        _ -> [{field, "must be a valid URL"}]
      end
    end)
  end
end
```

```elixir
# lib/blog/posts/post.ex
defmodule Blog.Posts.Post do
  use Ecto.Schema
  import Ecto.Changeset

  schema "posts" do
    field :title,        :string
    field :slug,         :string
    field :body,         :text
    field :excerpt,      :string
    field :cover_image,  :string
    field :is_published, :boolean, default: false
    field :view_count,   :integer, default: 0
    field :published_at, :utc_datetime

    # Embedded SEO data
    embeds_one :seo, Blog.Posts.Post.SEO, on_replace: :update

    # Associations
    belongs_to :author, Blog.Accounts.User
    belongs_to :category, Blog.Categories.Category

    has_many :comments, Blog.Comments.Comment, on_delete: :delete_all
    many_to_many :tags, Blog.Tags.Tag, join_through: "post_tags", on_replace: :delete

    timestamps(type: :utc_datetime)
  end

  def changeset(post, attrs) do
    post
    |> cast(attrs, [:title, :slug, :body, :excerpt, :cover_image,
                    :is_published, :published_at, :author_id, :category_id])
    |> validate_required([:title, :body, :author_id])
    |> validate_length(:title, min: 5, max: 255)
    |> validate_length(:excerpt, max: 500)
    |> generate_slug()
    |> unique_constraint(:slug)
    |> cast_embed(:seo)
    |> set_published_at()
  end

  def update_changeset(post, attrs) do
    post
    |> changeset(attrs)
    |> maybe_put_tags(attrs)
  end

  defp generate_slug(changeset) do
    case get_change(changeset, :title) do
      nil -> changeset
      title ->
        slug = title
        |> String.downcase()
        |> String.replace(~r/[^a-z0-9\s-]/, "")
        |> String.replace(~r/\s+/, "-")
        |> String.trim("-")

        case get_field(changeset, :slug) do
          nil -> put_change(changeset, :slug, slug)
          _   -> changeset  # ไม่ overwrite ถ้ามีอยู่แล้ว
        end
    end
  end

  defp set_published_at(changeset) do
    case get_change(changeset, :is_published) do
      true ->
        if get_field(changeset, :published_at) == nil do
          put_change(changeset, :published_at, DateTime.utc_now())
        else
          changeset
        end
      _ ->
        changeset
    end
  end

  defp maybe_put_tags(changeset, %{"tags" => tag_ids}) do
    tags = Blog.Tags.get_tags(tag_ids)
    put_assoc(changeset, :tags, tags)
  end
  defp maybe_put_tags(changeset, _), do: changeset
end

defmodule Blog.Posts.Post.SEO do
  use Ecto.Schema
  import Ecto.Changeset

  @primary_key false

  embedded_schema do
    field :meta_title,       :string
    field :meta_description, :string
    field :og_image,         :string
    field :keywords,         {:array, :string}, default: []
    field :no_index,         :boolean, default: false
  end

  def changeset(seo, attrs) do
    seo
    |> cast(attrs, [:meta_title, :meta_description, :og_image, :keywords, :no_index])
    |> validate_length(:meta_title, max: 60)
    |> validate_length(:meta_description, max: 160)
  end
end
```

```elixir
# lib/blog/categories/category.ex
defmodule Blog.Categories.Category do
  use Ecto.Schema
  import Ecto.Changeset

  schema "categories" do
    field :name,        :string
    field :slug,        :string
    field :description, :text
    field :color,       :string, default: "#3B82F6"
    field :icon,        :string
    field :position,    :integer, default: 0

    has_many :posts, Blog.Posts.Post

    timestamps(type: :utc_datetime)
  end

  def changeset(category, attrs) do
    category
    |> cast(attrs, [:name, :slug, :description, :color, :icon, :position])
    |> validate_required([:name])
    |> validate_length(:name, min: 2, max: 50)
    |> generate_slug()
    |> unique_constraint(:slug)
    |> validate_format(:color, ~r/^#[0-9A-Fa-f]{6}$/, message: "must be a valid hex color")
  end

  defp generate_slug(changeset) do
    case get_change(changeset, :name) do
      nil -> changeset
      name ->
        slug = name
        |> String.downcase()
        |> String.replace(~r/[^a-z0-9\s-]/, "")
        |> String.replace(~r/\s+/, "-")

        put_change(changeset, :slug, slug)
    end
  end
end
```

```elixir
# lib/blog/tags/tag.ex
defmodule Blog.Tags.Tag do
  use Ecto.Schema
  import Ecto.Changeset

  schema "tags" do
    field :name,  :string
    field :slug,  :string
    field :color, :string, default: "#6B7280"

    many_to_many :posts, Blog.Posts.Post,
      join_through: "post_tags",
      on_replace: :delete

    timestamps(type: :utc_datetime)
  end

  def changeset(tag, attrs) do
    tag
    |> cast(attrs, [:name, :slug, :color])
    |> validate_required([:name])
    |> validate_length(:name, min: 1, max: 50)
    |> generate_slug()
    |> unique_constraint(:name)
    |> unique_constraint(:slug)
  end

  defp generate_slug(changeset) do
    case get_change(changeset, :name) do
      nil -> changeset
      name ->
        slug = String.downcase(name) |> String.replace(~r/\s+/, "-")
        put_change(changeset, :slug, slug)
    end
  end
end
```

```elixir
# lib/blog/comments/comment.ex
defmodule Blog.Comments.Comment do
  use Ecto.Schema
  import Ecto.Changeset

  schema "comments" do
    field :body,          :text
    field :author_name,   :string
    field :author_email,  :string
    field :is_approved,   :boolean, default: false
    field :ip_address,    :string

    belongs_to :post, Blog.Posts.Post
    belongs_to :user, Blog.Accounts.User

    # Self-referential: reply ถึง comment อื่น
    belongs_to :parent, Blog.Comments.Comment
    has_many :replies, Blog.Comments.Comment, foreign_key: :parent_id

    timestamps(type: :utc_datetime)
  end

  def changeset(comment, attrs) do
    comment
    |> cast(attrs, [:body, :author_name, :author_email,
                    :post_id, :user_id, :parent_id, :ip_address])
    |> validate_required([:body, :post_id])
    |> validate_length(:body, min: 5, max: 2000)
    |> validate_author_info()
    |> foreign_key_constraint(:post_id)
    |> foreign_key_constraint(:parent_id)
  end

  defp validate_author_info(changeset) do
    # ถ้าไม่มี user_id ต้องมี author_name
    case get_field(changeset, :user_id) do
      nil ->
        changeset
        |> validate_required([:author_name])
        |> validate_length(:author_name, min: 2, max: 100)
        |> validate_format(:author_email, ~r/@/, message: "must include @")
      _ ->
        changeset
    end
  end
end
```

---

## 7. Migrations ครบชุดสำหรับ Blog

```elixir
# priv/repo/migrations/20240101000001_create_users.exs
defmodule Blog.Repo.Migrations.CreateUsers do
  use Ecto.Migration

  def change do
    create table(:users) do
      add :name,            :string,   null: false, size: 100
      add :email,           :string,   null: false, size: 160
      add :password_hash,   :string,   null: false
      add :role,            :string,   null: false, default: "reader"
      add :confirmed_at,    :utc_datetime
      add :last_sign_in_at, :utc_datetime
      add :avatar,          :string
      add :bio,             :text
      add :website,         :string

      timestamps(type: :utc_datetime)
    end

    create unique_index(:users, [:email])
    create index(:users, [:role])
  end
end

# priv/repo/migrations/20240101000002_create_categories.exs
defmodule Blog.Repo.Migrations.CreateCategories do
  use Ecto.Migration

  def change do
    create table(:categories) do
      add :name,        :string,  null: false, size: 50
      add :slug,        :string,  null: false, size: 60
      add :description, :text
      add :color,       :string,  default: "#3B82F6", size: 7
      add :icon,        :string
      add :position,    :integer, default: 0

      timestamps(type: :utc_datetime)
    end

    create unique_index(:categories, [:slug])
    create unique_index(:categories, [:name])
    create index(:categories, [:position])
  end
end

# priv/repo/migrations/20240101000003_create_posts.exs
defmodule Blog.Repo.Migrations.CreatePosts do
  use Ecto.Migration

  def change do
    create table(:posts) do
      add :title,        :string,   null: false, size: 255
      add :slug,         :string,   null: false, size: 100
      add :body,         :text
      add :excerpt,      :string,   size: 500
      add :cover_image,  :string
      add :is_published, :boolean,  default: false, null: false
      add :view_count,   :integer,  default: 0, null: false
      add :published_at, :utc_datetime
      add :seo,          :map,      default: %{}

      add :author_id,    references(:users, on_delete: :nilify_all)
      add :category_id,  references(:categories, on_delete: :nilify_all)

      timestamps(type: :utc_datetime)
    end

    create unique_index(:posts, [:slug])
    create index(:posts, [:author_id])
    create index(:posts, [:category_id])
    create index(:posts, [:is_published])
    create index(:posts, [:published_at])
  end
end

# priv/repo/migrations/20240101000004_create_tags.exs
defmodule Blog.Repo.Migrations.CreateTags do
  use Ecto.Migration

  def change do
    create table(:tags) do
      add :name,  :string, null: false, size: 50
      add :slug,  :string, null: false, size: 60
      add :color, :string, default: "#6B7280", size: 7

      timestamps(type: :utc_datetime)
    end

    create unique_index(:tags, [:name])
    create unique_index(:tags, [:slug])

    # Join table สำหรับ many_to_many
    create table(:post_tags, primary_key: false) do
      add :post_id, references(:posts, on_delete: :delete_all), null: false
      add :tag_id,  references(:tags, on_delete: :delete_all), null: false
      add :inserted_at, :utc_datetime, null: false
    end

    create unique_index(:post_tags, [:post_id, :tag_id])
    create index(:post_tags, [:tag_id])
  end
end

# priv/repo/migrations/20240101000005_create_comments.exs
defmodule Blog.Repo.Migrations.CreateComments do
  use Ecto.Migration

  def change do
    create table(:comments) do
      add :body,         :text,    null: false
      add :author_name,  :string,  size: 100
      add :author_email, :string,  size: 160
      add :is_approved,  :boolean, default: false, null: false
      add :ip_address,   :string,  size: 45

      add :post_id,   references(:posts, on_delete: :delete_all), null: false
      add :user_id,   references(:users, on_delete: :nilify_all)
      add :parent_id, references(:comments, on_delete: :delete_all)

      timestamps(type: :utc_datetime)
    end

    create index(:comments, [:post_id])
    create index(:comments, [:user_id])
    create index(:comments, [:parent_id])
    create index(:comments, [:is_approved])
  end
end
```

---

## 8. Exercises

### Exercise 1: Schema สำหรับ User Profile

สร้าง Schema สำหรับ extended user profile พร้อม embedded social links

**เฉลย:**

```elixir
defmodule Blog.Accounts.UserProfile do
  use Ecto.Schema
  import Ecto.Changeset

  schema "user_profiles" do
    field :display_name,  :string
    field :bio,           :text
    field :location,      :string
    field :website,       :string
    field :birthday,      :date
    field :is_public,     :boolean, default: true
    field :followers_count, :integer, default: 0, virtual: true

    embeds_one :social_links, SocialLinks, on_replace: :update do
      @primary_key false
      field :twitter,   :string
      field :github,    :string
      field :linkedin,  :string
      field :instagram, :string
    end

    belongs_to :user, Blog.Accounts.User

    timestamps(type: :utc_datetime)
  end

  def changeset(profile, attrs) do
    profile
    |> cast(attrs, [:display_name, :bio, :location, :website, :birthday, :is_public])
    |> validate_length(:display_name, max: 100)
    |> validate_length(:bio, max: 500)
    |> cast_embed(:social_links, with: &social_links_changeset/2)
    |> unique_constraint(:user_id)
  end

  defp social_links_changeset(social, attrs) do
    social
    |> cast(attrs, [:twitter, :github, :linkedin, :instagram])
    |> validate_format(:twitter, ~r/^@?[A-Za-z0-9_]{1,15}$/, message: "invalid Twitter handle")
  end
end
```

### Exercise 2: Polymorphic Association

สร้าง Notification schema ที่ associate กับหลาย model types

**เฉลย:**

```elixir
defmodule Blog.Notifications.Notification do
  use Ecto.Schema
  import Ecto.Changeset

  schema "notifications" do
    field :type,          :string  # "post_liked", "comment_added", etc.
    field :read_at,       :utc_datetime
    field :data,          :map, default: %{}  # Extra data

    # Polymorphic: ใช้ notifiable_type + notifiable_id
    field :notifiable_type, :string
    field :notifiable_id,   :integer

    belongs_to :user, Blog.Accounts.User  # recipient

    timestamps(type: :utc_datetime)
  end

  def changeset(notification, attrs) do
    notification
    |> cast(attrs, [:type, :user_id, :notifiable_type, :notifiable_id, :data])
    |> validate_required([:type, :user_id, :notifiable_type, :notifiable_id])
    |> validate_inclusion(:notifiable_type, ["Post", "Comment", "User"])
    |> foreign_key_constraint(:user_id)
  end

  def mark_read(notification) do
    change(notification, read_at: DateTime.utc_now())
  end

  def unread?(notification), do: is_nil(notification.read_at)
end
```

---

## สรุป

```
Ecto Schema:
├── schema "table_name" do ... end
├── Field types: :string, :text, :integer, :boolean,
│   :float, :decimal, :date, :utc_datetime, :map,
│   {:array, :string}, Ecto.Enum
├── Virtual fields: field :x, :string, virtual: true
└── timestamps() - inserted_at, updated_at

Migrations:
├── create table(:name) do ... end
├── alter table(:name) do add/modify/remove
├── create index/unique_index
├── references(:table, on_delete: ...)
└── change() vs up/down()

Associations:
├── belongs_to :name, Module  (has FK column)
├── has_many :name, Module, foreign_key: :col
├── has_one :name, Module
├── many_to_many :name, Module, join_through: "table"
└── on_delete: :delete_all | :nilify_all | :restrict

Embedded:
├── embeds_one :name, Module
├── embeds_many :name, Module
└── cast_embed(:name) ใน changeset
```

---

*ก่อนหน้า: [Part 28 - Phoenix Templates (HEEx)](part_28.md) | ต่อไป: [Part 30 - Ecto Queries และ Changesets](part_30.md)*
