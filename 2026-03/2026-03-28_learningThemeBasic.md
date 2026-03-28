# [卒制](https://github.com/QynToKey/HowLongWillItLast) (MVP 後)：`LearningTheme` 再実装（基盤構築 と 既存データの移行）

## 0️⃣ 実装方針の確認

以前に一度生成した `LearningTheme` と同名モデルを再実装するため、差分を整理しておく。

| | 古いファイル | 今回の設計 |
| --- | --- | --- |
| `name` | NOT NULL | *※ NULL許可* |
| `description` | あり | なし |
| UNIQUE制約 | `(user_id, name)` | *※ なし* |
| `on_delete: :cascade` | あり | あり |

📝 `name` のUNIQUE制約について：
ユーザーが２つ以上の「学習テーマ」を登録する場合は、無題のテーマを許可してしまうと混乱を招く可能性があるので NOT NULL としたい。

👉 *DB ではなく、アプリケーション側のバリデーションで管理することとする*

```ruby
# モデル側で制御
validates :name, presence: true, if: -> { user.learning_themes.count >= 1 }
```

---

## 1️⃣ `LearningTheme` モデルの生成

### g コマンドでモデルを生成

👉 *既存のマイグレーションファイルに上書き*

```bash
$ docker compose exec web rails g model LearningTheme user:references name:string --force
      invoke  active_record
      remove    db/migrate/20260304080053_create_learning_themes.rb
      create    db/migrate/20260328045747_create_learning_themes.rb
      create    app/models/learning_theme.rb
```

### 生成されたマイグレーションファイルを修正

```ruby
# db/migrate/20260328045747_create_learning_themes.rb
class CreateLearningThemes < ActiveRecord::Migration[7.2]
  def change
    create_table :learning_themes do |t|
      t.references :user, null: false, foreign_key: { on_delete: :cascade } # ⬅️ 親が消えたら子も消える
      t.string :name

      t.timestamps
    end

    # 同じユーザーが同じ名前のテーマを2つ作れない、ただしname未設定はいくつあってもOK
    add_index :learning_themes, [:user_id, :name], unique: true, where: "name IS NOT NULL"
  end
end
```

📝 `on_delete: :cascade` と `dependent: :destroy` の違い

| | `dependent: :destroy` | `on_delete: :cascade` |
| --- | --- | --- |
| 動く場所 | Railsアプリ（Ruby） | DB（PostgreSQL） |
| コール | Railsのコールバック経由 | SQLレベルで直接削除 |
| コスト | 重い（Rubyを経由） | 軽い（DBが直接処理） |

- `dependent: :destroy` はRailsのコールバックが発火するので、`before_destroy` などが効く
- `on_delete: :cascade` はDBが直接削除するのでRailsを素通りする

📝 `add_index`

```ruby
add_index :learning_themes, [:user_id, :name], unique: true, where: "name IS NOT NULL"
```

- `[:user_id, :name]` → この2カラムの組み合わせにインデックスを貼る
- `unique: true` → その組み合わせで重複を禁止する
- `where: "name IS NOT NULL"` → **NULLの行はこの制約の対象外にする**（部分インデックス）

👉 *結果として「同じユーザーが同じ名前のテーマを2つ作れない、ただしname未設定はいくつあってもOK」という制約になる*

### `db:migrate`

```bash
$ docker compose exec web rails db:migrate
== 20260328045747 CreateLearningThemes: migrating =============================
-- create_table(:learning_themes)
   -> 0.0366s
-- add_index(:learning_themes, [:user_id, :name], {:unique=>true, :where=>"name IS NOT NULL"})
   -> 0.0154s
== 20260328045747 CreateLearningThemes: migrated (0.0521s) ====================
```

### `User` との紐付け

👉 *`LearningTheme` 側は `rails g model` で自動生成済みなので、`User` 側に設定を追加*

```ruby
# app/models/user.rb
  has_many :learning_themes, dependent: :destroy
```

---

## 2️⃣ 既存データの移行

### マイグレーションファイルを生成

```bash
$ docker compose exec web rails g migration MigrateLearningThemeToLearningThemes
      invoke  active_record
      create    db/migrate/20260328060103_migrate_learning_theme_to_learning_themes.rb
```

### `users.learning_theme` カラムを削除

```ruby
# 2026-03/2026-03-28_learningThemeBasic.md
class MigrateLearningThemeToLearningThemes < ActiveRecord::Migration[8.1]
  def up
    # 全ユーザーに learning_themes レコードを１件作成
    User.find_each do |user|
      LearningTheme.create!(
        user_id: user.id,
        name: user.learning_theme #nil でもそのまま learning_themes.name へコピー
      )
    end

    # users.learning_theme カラムを削除
    remove_column :users, :learning_theme
  end

  # db:rollback が必要になったときは up メソッドを逆再生する
  def down
    add_column :users, :learning_theme, :string

    LearningTheme.find_each do |theme|
      User.find(theme.user_id).update!(learning_theme: theme.name)
    end
  end
end
```

  ⬇️ `db:migrate`

```bash
$ docker compose exec web rails db:migrate
== 20260328060103 MigrateLearningThemeToLearningThemes: migrating =============
-- remove_column(:users, :learning_theme)
   -> 0.0061s
== 20260328060103 MigrateLearningThemeToLearningThemes: migrated (0.0925s) ====
```

### `UsersController` から `:learning_theme` を削除

```ruby
# app/controllers/users_controller.rb
  def user_params
    params.require(:user).permit(:name, :email, :password, :password_confirmation)
  end
```

---

## 3️⃣ `LearningTheme` カラム削除に伴うビューの表示エラーに対応

### マイページ / TOP ページ

👉 *`current_user.learning_theme` カラムがもう存在しないため、`users.learning_theme`（文字列）ではなく `learning_themes` テーブルから取得するように変更する*

```ruby
# app/views/users/show.html.erb
      学習テーマ： <%= current_user.learning_themes.first&.name %>
```

```ruby
# app/views/home/index.html.erb
      <% if current_user.learning_themes.first&.name.present? %>
        <p class='small mt-3'>'<%= current_user.learning_themes.first&.name %>' の総学習時間</p>
      <% else %>
```

### マイページ `new` / `edit`

| 機能 | 実装方針 |
| --- | --- |
| 1つ目のテーマを登録・編集する | ✅ 実装・公開する |
| 2つ目以降のテーマを追加する導線 | 🔒 実装済み・公開保留 |
