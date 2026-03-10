# [卒制](https://github.com/QynToKey/HowLongWillItLast) (day 11)：LearningRecord 機能 (`new` / `create`)

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
end
```

👉 *ストップウォッチによる自動計測機能は、「手入力」を許可したいので `allow_nil: true` とする*

👉 *`duration_minuets` は「正の整数」のみ許可*

---

## 2️⃣ 自動計測機能に関するロジックを実装

- `started_at` と `ended_at` から `duration_minutes` を計算するロジックを実装する
- このロジックは「データに関する計算処理」なので、**モデルの責務**

```ruby
class LearningRecord < ApplicationRecord
  ・・・

  # ⬇️ バリデーションの前に学習時間を計算するコールバック
  before_validation :calculate_duration

  ・・・

  # ⬇️ 開始時間と終了時間の両方が存在する場合、終了時間は開始時間より後でなければならない
  validate :ended_at_after_started_at

  private

  # ⬇️ 開始時間と終了時間から学習時間を計算して duration_minutes にセットする
  def calculate_duration
    if started_at.present? && ended_at.present?
      self.duration_minutes = ((ended_at - started_at) / 60).to_i
    end
  end

  # ⬇️ 終了時間は開始時間より後でなければならない
  def ended_at_after_started_at
    return if started_at.blank? || ended_at.blank?

    if ended_at <= started_at
      errors.add(:ended_at, "は開始時間より後の時刻を入力してください")
    end
  end
end
```

👉 *Railsの「コールバック実行順序」は、コードの書き順に関係なく**Railsが決めた順番**で動く* ⇨ [Active Record コールバック](https://railsguides.jp/v7.1/active_record_callbacks.html)

---

## 3️⃣ LearningRecord をルーティングに追加

```ruby
Rails.application.routes.draw do
  ・・・
  resources :learning_records
  ・・・
end
```

---

## 4️⃣ `learning_records_controller` を作成

### `rails g controller LearningRecords`

```bash
$ docker compose exec web rails g controller LearningRecords
      create  app/controllers/learning_records_controller.rb
      invoke  erb
      create    app/views/learning_records
      invoke  helper
      create    app/helpers/learning_records_helper.rb
```

⚠️ *ここでアクション名を指定すると、すでに `routes.rb` に `resources :learning_records` を実装済みなので、ルーティングが重複してしまう*

⬇️

### コントローラーを編集

```ruby
class LearningRecordsController < ApplicationController
  skip_before_action :require_login, only: %i[new]

  def new
    # デフォルトで今日の日付をセットする
    @learning_record = LearningRecord.new(study_date: Date.today)
  end

  def create
    # ⬇️ 本来必要なコードだが、ビューで制御するため不要
    # unless logged_in?
    # redirect_to login_path, alert: "ログインしてください"
    # return

    @learning_record = current_user.learning_records.build(learning_record_params)

    if @learning_record.save
      # 記録後は詳細ページ( show )へリダイレクト
      redirect_to @learning_record, notice: "学習記録を保存しました"
    else
      flash.now[:alert] = "学習記録の保存に失敗しました"
      render :new, status: :unprocessable_entity
    end
  end

  private

  def learning_record_params
    params.require(:learning_record).permit(:study_date, :content, :duration_minutes, :started_at, :ended_at, tag_ids: [])
  end
end
```

📝 ストロングパラメーターに含めるべきカラム

| カラム | フォームから入力する？ |
| --- | --- |
| `study_date` | ✅ |
| `content` | ✅ |
| `duration_minutes` | ✅ 手入力の場合 |
| `started_at` | ✅ ストップウォッチの場合 |
| `ended_at` | ✅ ストップウォッチの場合 |
| `user_id` | ❌ `current_user` から取得 |

*(ストロングパラメーターは **ユーザーがフォームから入力するカラム** を許可する)*

---

## 5️⃣ 「今日の記録画面」を作成

```bash
touch app/views/learning_records/new.html.erb
```

⬇️

```ruby
# app/views/learning_records/new.html.erb
<h1>今日の学習記録</h1>

<% if @learning_record.errors.any? %>
  <div style="color: red;">
    <h2><%= @learning_record.errors.count %>件のエラーがあります</h2>
    <ul>
      <% @learning_record.errors.full_messages.each do |message| %>
        <li><%= message %></li>
      <% end %>
    </ul>
  </div>
<% end %>

<%= form_with model: @learning_record do |f| %>
  <div>
    <%= f.label :content, "学習内容" %><br>
    <%= f.text_area :content, rows: 4 %>
  </div>

  <div>
    <%= f.label :duration_minutes, "学習時間（分）" %><br>
    <%= f.number_field :duration_minutes, min: 0 %>
  </div>

  <div>
    <%= f.label :study_date, "日付" %><br>
    <%= f.date_field :study_date %>
  </div>

  <div>
    <%= f.submit "記録する" %>
  </div>
<% end %>
```

---

## 6️⃣ 動作確認

### 動作確認のために `show` アクションとビューを追加

```ruby
# app/controllers/learning_records_controller.rb
  def show
    @learning_record = current_user.learning_records.find(params[:id])
  end
```

```bash
touch app/views/learning_records/show.html.erb
```

⬇️

### [ブラウザ](http://localhost:3000/learning_records/new) で動作確認

☑️ 正常に記録されると `show` ページへ遷移し、"学習記録を保存しました" と表示される

☑️ NOT NULL 制約に違反するとエラーメッセージがフラッシュで表示される

---
