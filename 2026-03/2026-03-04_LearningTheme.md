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
