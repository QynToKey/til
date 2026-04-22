# [co-READER](https://github.com/QynToKey/co_reader)（day: 15）： セッション内テキストの並び順制御機能

## 1️⃣ `position` / `enforce_order` カラムを追加

```bash
$ docker compose exec web bin/rails g migration AddPositionToReadingSessionTexts position:integer
      invoke  active_record
      create    db/migrate/20260422015628_add_position_to_reading_session_texts.rb
```

```bash
$ docker compose exec web bin/rails g migration AddEnforceOrderToReadingSessions enforce_order:boolean
      invoke  active_record
      create    db/migrate/20260422015933_add_enforce_order_to_reading_sessions.rb
```

  ⬇️

```ruby
# db/migrate/20260422015628_add_position_to_reading_session_texts.rb
class AddPositionToReadingSessionTexts < ActiveRecord::Migration[8.1]
  def change
    add_column :reading_session_texts, :position, :integer
  end
end
```

```ruby
# db/migrate/20260422015933_add_enforce_order_to_reading_sessions.rb
class AddEnforceOrderToReadingSessions < ActiveRecord::Migration[8.1]
  def change
    add_column :reading_sessions, :enforce_order, :boolean, default: false, null: false
  end
end
```

  ⬇️

```bash
$ docker compose exec web bin/rails db:migrate
== 20260422015628 AddPositionToReadingSessionTexts: migrating =================
-- add_column(:reading_session_texts, :position, :integer)
   -> 0.0094s
== 20260422015628 AddPositionToReadingSessionTexts: migrated (0.0095s) ========

== 20260422015933 AddEnforceOrderToReadingSessions: migrating =================
-- add_column(:reading_sessions, :enforce_order, :boolean, {:default=>false, :null=>false})
   -> 0.0186s
== 20260422015933 AddEnforceOrderToReadingSessions: migrated (0.0187s) ========
```

---

## 2️⃣ 「管理者フォーム」を更新

```erb
<%# app/views/admin/reading_sessions/_form.html.erb %>

-  <div class="mb-3">
-    <label class="form-label"><%= t("admin.reading_sessions.form.texts_label") %></label>
-    <%= f.collection_check_boxes :text_ids, @texts, :id, :title do |b| %>
-      <div class="form-check">
-        <%= b.check_box class: "form-check-input" %>
-        <%= b.label class: "form-check-label" %>
-      </div>
-    <% end %>
-  </div>
+  <div class="mb-3">
+    <label class="form-label"><%= t("admin.reading_sessions.form.texts_label") %></label>
+    <% @texts.each do |text| %>
+      <div class="d-flex align-items-center gap-3 mb-2">
+        <div class="form-check mb-0">
+          <%= check_box_tag "reading_session[text_ids][]",
+                            text.id,
+                            @reading_session.texts.include?(text),
+                            id: "text_#{text.id}",
+                            class: "form-check-input" %>
+          <%= label_tag "text_#{text.id}", text.title, class: "form-check-label" %>
+        </div>
+        <%= number_field_tag "reading_session[text_positions][#{text.id}]",
+                             @reading_session.reading_session_texts.find_by(text: text)&.position,
+                             min: 1,
+                             class: "form-control form-control-sm",
+                             style: "width: 80px;",
+                             placeholder: "順番" %>
+      </div>
+    <% end %>
+  </div>
+
+  <div class="mb-3">
+    <div class="form-check">
+      <%= f.check_box :enforce_order, class: "form-check-input" %>
+      <%= f.label :enforce_order, "順番を固定する（読者は指定順にのみ閲覧可）", class: "form-check-label" %>
+    </div>
+  </div>
```

```ruby
# app/assets/stylesheets/application.css
::placeholder {
  font-size: 0.8rem;
  color: #bbb !important;
}
```

---

## 3️⃣ `Admin::ReadingSessions` コントローラーを更新（管理者側）

👉 *`create` と `update` の保存後に `save_positions` `を呼ぶ処理を追加し、reading_session_params` と `private` メソッドを更新*

```ruby
# app/controllers/admin/reading_sessions_controller.rb
class Admin::ReadingSessionsController < ApplicationController
  # 管理者専用のコントローラー
  before_action :require_admin
  before_action :set_reading_session, only: %i[ edit update destroy ]

  def index
    @reading_sessions = ReadingSession.includes(:creator, :texts).order(created_at: :desc)
  end

  def new
    @reading_session = ReadingSession.new
    @texts = Text.order(:title)
  end

  def create
    @reading_session = ReadingSession.new(reading_session_params)
    @reading_session.creator = current_user
    # セッションの招待トークンを生成
    @reading_session.invite_token = SecureRandom.urlsafe_base64

    if @reading_session.save
      save_positions
      redirect_to admin_reading_sessions_path, notice: "セッションを作成しました"
    else
      @texts = Text.order(:title)
      render :new, status: :unprocessable_entity
    end
  end

  def edit
    @texts = Text.order(:title)
  end

  def update
    if @reading_session.update(reading_session_params)
      save_positions
      redirect_to admin_reading_sessions_path, notice: t("admin.reading_sessions.update.success")
    else
      @texts = Text.order(:title)
      render :edit, status: :unprocessable_entity
    end
  end

  def destroy
    @reading_session.destroy
    redirect_to admin_reading_sessions_path, notice: t("admin.reading_sessions.destroy.success")
  end

  private

  # 管理者でないユーザーのアクセスを制限するメソッド
  def set_reading_session
    @reading_session = ReadingSession.find(params[:id])
  end

  # ストロングパラメーター: セッション更新に必要なパラメーターを許可する
  def reading_session_params
    params.require(:reading_session).permit(:name, :enforce_order, text_ids: [])
  end

  # テキストの順番を保存するメソッド
  def save_positions
    # テキストIDと位置のペアを params から取得
    text_positions = params.dig(:reading_session, :text_positions)
    return if params[:reading_session][:text_positions].blank?
    # テキストIDと位置のペアをループして保存
    params[:reading_session][:text_positions].each do |text_id, position|
      # 位置が空の場合はスキップ
      next if position.blank?
      @reading_session.reading_session_texts.find_by(text_id: text_id)&.update(position: position.to_i)
    end
  end
end
```

---

## 4️⃣ `Reading_sessions` / `Texts` コントローラーを更新（一般ユーザー側）

```ruby
# app/controllers/reading_sessions_controller.rb
  def show
    ・・・
    # セッションのテキストを順序付きで取得する。順序が指定されている場合は position カラムでソートする
    @texts = if @reading_session.enforce_order?
      @reading_session.texts.order("reading_session_texts.position ASC NULLS LAST")
    else
      @reading_session.texts
    end
  end
```

```ruby
# app/controllers/texts_controller.rb
  def show
    # セッションのテキストを順序付きで取得する。順序が指定されている場合は position カラムでソートする
    session_text = if @reading_session.enforce_order?
      @reading_session.texts.order("reading_session_texts.position ASC NULLS LAST")
    else
      @reading_session.texts.order(:id)
    end
    current_index = session_text.index(@text)
    @prev_text = session_text[current_index - 1] if current_index > 0
    @next_text = session_text[current_index + 1]
  end
```

---

## 5️⃣ ER 図を更新

### `reading_sessions` の責務説明（32〜33行目）**

```diff
 - 特定のテキストを対象とした読書セッション
 - 招待トークン（`invite_token`）による URL 招待方式で参加者を管理する
+- `enforce_order` でテキストの閲覧順を固定するかどうかを管理する
```

### `reading_session_texts` の責務説明（36〜38行目）**

```diff
 - セッションとテキストの多対多の関係を管理する中間テーブル
 - 1 セッションに複数のテキストを紐づけることができる
+- `position` でセッション内のテキスト表示順を管理する
```

### Mermaid ER 図の2テーブル

```diff
     READING_SESSIONS {
         bigint id PK
         bigint created_by FK "NOT NULL"
         string name
         string invite_token "NOT NULL, UNIQUE"
+        boolean enforce_order "NOT NULL, DEFAULT false"
         datetime created_at
         datetime updated_at
     }

     READING_SESSION_TEXTS {
         bigint id PK
         bigint reading_session_id FK "NOT NULL"
         bigint text_id FK "NOT NULL"
+        integer position
         datetime created_at
         datetime updated_at
     }
```

---

#### 総学習時間： 1235.0 時間
