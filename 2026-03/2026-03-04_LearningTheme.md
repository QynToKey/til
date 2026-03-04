# [卒制](https://github.com/QynToKey/HowLongWillItLast) (day 7)：LearningTheme機能の実装

---

## 0️⃣ 設計方針の確認

### `LearningTheme` には `user_id`を持たせる

- `LearningTheme` は「Userに従属する存在」とする。
  - 同じ `LearningTheme` であっても他のユーザーのそれとは区別される *(SNS 的な「交流機能」よりも「内省」の機会を提供したい)
  - `LearningTheme` は当該ユーザーによってのみ CRUD できる

  ---

### migration の要件

#### 🔸 `user_id` → NOT NULL

  `validates :presence` は Rails 側のバリデーションなので、DB 側にも制約をかけるために `null: false` を併記する必要がある。

  📝 `null: false` と `validates :presence` の違い

  | 種類 | どこでチェックされる？ |
  | --- | --- |
  | `null: false` | DB（データベース） |
  | `validates :presence` | Rails（アプリ） |

#### 🔸 `user_id` /  `name` → UNIQUE

  同じユーザー内で同じ `name` （テーマ名）を重複させない

#### 🔸 `foreign_key: true`

  `user_id` に「実在するユーザーID」だけを許可する制約

#### 🔸 `dependent: :destroy` と `ON DELETE CASCADE`

- `dependent: :destroy` ： Rails経由でUserを削除したとき、関連テーマも削除

  ```ruby
  has_many :learning_themes, dependent: :destroy
  ```

- `ON DELETE CASCADE` ： DBレベルで、親が消えたら子も消す

  ```ruby
  foreign_key: { on_delete: :cascade }
  ```

---

## 実装

### 1️⃣ `LearningTheme` を生成

```bash
docker compose exec web rails g model LearningTheme name:string description:text user:references
```

👉 *カラムの記述は順不同で構わないが、実務では可読性の観点で以下のようにすることが多い*

  1. 主要カラム（`name`）
  2. 補足カラム（`description`）
  3. 外部キー（`user`）

### 2️⃣ migrate ファイルを修正

```ruby

class CreateLearningThemes < ActiveRecord::Migration[7.0]
  def change
    create_table :learning_themes do |t|
      t.string :name, null: false
      t.text :description
      t.references :user, null: false,
                          foreign_key: { on_delete: :cascade }

      t.timestamps
    end

    add_index :learning_themes, [:user_id, :name], unique: true
  end
end
```

👉 *以下を実装*

- user_id → NOT NULL
- 外部キー制約
- ON DELETE CASCADE
- (user_id, name) UNIQUE
- description → text

### 3️⃣ モデル側にバリデーション および アソシエーション を追記

- `app/models/learning_theme.rb`

```ruby
class LearningTheme < ApplicationRecord
  belongs_to :user

  validates :name, presence: true, uniqueness: { scope: :user_id }
end
```

- `user.rb'

```ruby
  has_many :learning_themes, dependent: :destroy
```

📝 モデルファイルの構造は、可読性の観点から以下が一般的

```ruby
class User < ApplicationRecord
  # 認証
  authenticates_with_sorcery!

  # Association
  has_many :learning_themes, dependent: :destroy

  # Callback
  before_validation ...

  # Validation
  validates ...
end
```

---

### 4️⃣ `db:migrate` を実行

```bash
docker compose exec rails db:migrate
```

⬇️

```bash
== 20260304080053 CreateLearningThemes: migrating =============================
-- create_table(:learning_themes)
   -> 0.0436s
-- add_index(:learning_themes, [:user_id, :name], {:unique=>true})
   -> 0.0021s
== 20260304080053 CreateLearningThemes: migrated (0.0458s) ====================
```

---
