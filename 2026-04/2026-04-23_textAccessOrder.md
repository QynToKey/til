# [co-READER](https://github.com/QynToKey/co_reader)（day: 16）： セッション内テキストのアクセス順序制御

## 0️⃣ 実装方針

> テキストへのアクセスチェック：

「誰が（`membership_id`）・どのテキストを（`text_id`）・読んだか」というレコード `text_readings` が存在すれば「既読」、なければ「未読」と判定する。

- `enforce_order: true` のセッションでテキスト X を開こうとしたとき
  - X より `position` が小さいテキストの `text_readings` レコードがすべて存在するか？
  - 存在しなければリダイレクト（alert メッセージ）

> テキストの読了チェック：

- テキスト末尾に「読了」ボタン／チェックボックスを配置 → クリックで記録
- メリット：意図的な行為なので読者の能動性に合う。#31・#57 とも自然に連携できる
- デメリット：押し忘れの可能性

> 仕様のイメージ：

```text
「読了にする」ボタン
        ↓
  TextReading レコード作成
        ↓
  ├─ enforce_order のアクセス制御に使う（#99）
  └─ 進捗表示・読了ステータスに使う（#31）
```

---

## 1️⃣ `TextReadings` テーブルを作成

```bash
$ docker compose exec web bin/rails g migration CreateTextReadings membership_id:references text_id:references read_at:datetime
      invoke  active_record
      create    db/migrate/20260423034802_create_text_readings.rb
```

  ⬇️

```ruby
# db/migrate/20260423034802_create_text_readings.rb
class CreateTextReadings < ActiveRecord::Migration[8.1]
  def change
    create_table :text_readings do |t|
      t.references :membership, null: false, foreign_key: true
      t.references :text, null: false, foreign_key: true
      t.datetime :read_at, null: false

      # t.timestamps
      t.index [:membership_id, :text_id], unique: true
    end
  end
end
```

  ⬇️

```bash
$ docker compose exec web bin/rails db:migrate
== 20260423034802 CreateTextReadings: migrating ===============================
-- create_table(:text_readings)
   -> 0.0578s
== 20260423034802 CreateTextReadings: migrated (0.0579s) ======================
```

---

## 2️⃣ `TextReading` モデルを作成

```bash
touch app/models/text_reading.rb
```

```ruby
# app/models/text_reading.rb
class TextReading < ApplicationRecord
  # ユーザーが特定のテキストをいつ読んだかを記録するためのモデル。
  belongs_to :membership
  belongs_to :text

  validates :read_at, presence: true
  validates :text_id, uniqueness: { scope: :membership_id }
end
```

---

## 3️⃣ `text_reading` のルーティングを `texts` にネスト

```ruby
# config/routes.rb
resources :reading_sessions, only: %i[ index show ] do
  get :join, on: :collection
  resources :texts, only: %i[ show ] do
    resource :text_reading, only: %i[ create ] # ⬅️ resource は単数形
  end
end
```

👉 *`text_reading` はテキスト1本につき1件しか存在しないため、ID を URL に含める必要がない*

---

## 4️⃣ `TextReadings` コントローラーを作成

```bash
touch app/controllers/text_readings_controller.rb
```

```ruby
# app/controllers/text_readings_controller.rb
class TextReadingsController < ApplicationController
  # ユーザーが特定のテキストを読んだことを記録するためのコントローラー。
  before_action :require_login
  before_action :set_reading_session
  before_action :set_text

  def create
    membership = @reading_session.memberships.find_by(user: current_user)
    # すでに読んだことがある場合は更新、ない場合は新規作成
    membership.text_readings.find_or_create_by(text: @text) do |tr|
      tr.read_at = Time.current # tr は text_reading
    end
    redirect_to reading_session_text_path(@reading_session, @text)
  end

  private

  # セッションとテキストをセットするためのメソッド
  def set_reading_session
    @reading_session = ReadingSession.find(params[:reading_session_id])
  end

  # テキストをセットするためのメソッド
  def set_text
    @text = Text.find(params[:text_id])
  end
end
```

---

## 5️⃣ `Membership` モデルに `reading_text` を追加

```ruby
  class Membership < ApplicationRecord
    belongs_to :user
    belongs_to :reading_session
+   has_many :text_readings, dependent: :destroy

    enum :session_role, { member: 0, session_admin: 1 }
    enum :mode, { solo: 0, co_reading: 1 }
```

---

## 6️⃣「テキスト詳細」に「読了ボタン」を実装

```erb
<%# app/views/texts/show.html.erb %>
  <div class="lh-lg">
    <%= simple_format(@text.body) %>
  </div>

+  <% membership = @reading_session.memberships.find_by(user: current_user) %>
+  <% already_read = membership.text_readings.exists?(text: @text) %>
+
+  <div class="text-center mt-5">
+    <% if already_read %>
+      <span class="btn btn-success disabled"><i class="bi bi-check-lg"></i> 読了</span>
+    <% else %>
+      <%= button_to "読了にする",
+            reading_session_text_text_reading_path(@reading_session, @text),
+            class: "btn btn-outline-success" %>
+    <% end %>
+  </div>
+
  <div class="d-flex justify-content-center align-items-center gap-3 mt-5">
```
