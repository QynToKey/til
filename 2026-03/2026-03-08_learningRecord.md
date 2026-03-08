# [卒制](https://github.com/QynToKey/HowLongWillItLast) (day 11)：LearningRecord 機能

---

## 0️⃣ 現在地の確認

- LearningRecord モデルは生成済み

```ruby
class LearningRecord < ApplicationRecord
  belongs_to :user

  has_many :record_tags, dependent: :destroy
  has_many :tags, through: :record_tags
end
```

- 必要なテーブルも全て揃っている

```ruby
# db/schema.rb
  create_table "learning_records", force: :cascade do |t|
    t.bigint "user_id", null: false
    t.date "study_date", null: false
    t.integer "duration_minutes"
    t.text "content", null: false
    t.datetime "started_at"
    t.datetime "ended_at"
    t.datetime "created_at", null: false
    t.datetime "updated_at", null: false
    t.index ["user_id"], name: "index_learning_records_on_user_id"
  end

  create_table "record_tags", force: :cascade do |t|
    t.bigint "learning_record_id", null: false
    t.bigint "tag_id", null: false
    t.datetime "created_at", null: false
    t.datetime "updated_at", null: false
    t.index ["learning_record_id", "tag_id"], name: "index_record_tags_on_learning_record_id_and_tag_id", unique: true
    t.index ["learning_record_id"], name: "index_record_tags_on_learning_record_id"
    t.index ["tag_id"], name: "index_record_tags_on_tag_id"
  end

  create_table "tags", force: :cascade do |t|
    t.bigint "user_id", null: false
    t.string "name", null: false
    t.datetime "created_at", null: false
    t.datetime "updated_at", null: false
    t.index ["user_id", "name"], name: "index_tags_on_user_id_and_name", unique: true
    t.index ["user_id"], name: "index_tags_on_user_id"
  end

  create_table "todo_tags", force: :cascade do |t|
    t.bigint "todo_id", null: false
    t.bigint "tag_id", null: false
    t.datetime "created_at", null: false
    t.datetime "updated_at", null: false
    t.index ["tag_id"], name: "index_todo_tags_on_tag_id"
    t.index ["todo_id", "tag_id"], name: "index_todo_tags_on_todo_id_and_tag_id", unique: true
    t.index ["todo_id"], name: "index_todo_tags_on_todo_id"
  end

  create_table "todos", force: :cascade do |t|
    t.bigint "user_id", null: false
    t.string "title"
    t.text "description"
    t.boolean "is_completed", default: false, null: false
    t.datetime "created_at", null: false
    t.datetime "updated_at", null: false
    t.index ["user_id"], name: "index_todos_on_user_id"
  end

  create_table "users", force: :cascade do |t|
    t.string "name"
    t.string "email", null: false
    t.string "crypted_password", null: false
    t.string "salt", null: false
    t.datetime "created_at", null: false
    t.datetime "updated_at", null: false
    t.index ["email"], name: "index_users_on_email", unique: true
  end

  add_foreign_key "learning_records", "users"
  add_foreign_key "record_tags", "learning_records"
  add_foreign_key "record_tags", "tags"
  add_foreign_key "tags", "users"
  add_foreign_key "todo_tags", "tags"
  add_foreign_key "todo_tags", "todos"
  add_foreign_key "todos", "users"
end
```

---

## 1️⃣ バリデーションの設定

```ruby
class LearningRecord < ApplicationRecord
  ・・・
  validates :study_date, presence: true
  validates :content, presence: true
  validates :duration_minutes, allow_nil: true, numericality: { only_integer: true, greater_than: 0 }
  validates :started_at, allow_nil: true
  validates :ended_at, allow_nil: true
end
```

👉 *ストップウォッチによる自動計測機能は、「手入力」を許可したいので `allow_nil: true` とする*

👉 *`duration_minuets` は「正の整数」のみ許可*

---

## 2️⃣
