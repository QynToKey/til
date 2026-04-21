# [co-READER](https://github.com/QynToKey/co_reader)（day: 14_1）： セッション作成フロー

## 0️⃣ 現状と実装方針

> 現状

`ReadingSessionsController` には `index` / `join` のみで、セッション作成機能が未実装

> 実装方針

- セッション作成は管理者のみ可能
- `reading_session_texts` 中間テーブル（#81）を経由して複数テキストを紐づける
- `InviteRegistrationsController#create` の不備（自動ログインなし）も合わせて修正

---

## 1️⃣ ルーティングに `reading_sessions` を追加

```ruby
# config/routes.rb
   namespace :admin do
     get  "register", to: "registrations#new",    as: :register
     post "register", to: "registrations#create"
     resources :invitations, only: %i[ index create ]
     resources :members, only: %i[ index new create ]
     resources :texts, only: %i[ index show new create edit update destroy ]
+    resources :reading_sessions, only: %i[ index new create edit update destroy ]
   end
```

---

## 2️⃣ `ReadingSession` モデルにバリデーションを追加

```ruby
# app/models/reading_session.rb
   validates :invite_token, presence: true, uniqueness: true
   validates :creator, presence: true
+
+  validate :at_least_one_text

   # セッション作成後に、作成者をセッション管理者として memberships テーブルに自動的に追加するコールバック
   after_create :assign_creator_as_admin
+
+  private
+
+  def at_least_one_text
+    errors.add(:texts, "を1件以上選択してください") if text_ids.blank?
+  end
```

---

## 3️⃣ `Admin::ReadingSessions` コントローラーを作成

```bash
touch app/controllers/admin/reading_sessions_controller.rb
```

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

  # ストロングパラメーター: セッション作成に必要なパラメーターを許可する
  def reading_session_params
    params.require(:reading_session).permit(:name, text_ids: [])
  end
end
```

---

## 4️⃣ `ja.yml` に `reading_sessions` 関連の語彙を追記

```ruby
# config/locales/views/ja.yml
    reading_sessions:
      index:
        title: "セッション管理"
        new_button: "セッションを作成"
        table_name: "セッション名"
        table_texts_count: "テキスト数"
        table_created_by: "作成者"
        table_created_at: "作成日時"
        table_invite_token: "招待トークン"
        no_name: "（名称なし）"
        edit: "編集"
        destroy: "削除"
        confirm_destroy: "本当に削除しますか？"
      new:
        title: "セッションを作成"
      form:
        name_label: "セッション名（任意）"
        texts_label: "テキストを選択（1件以上）"
        submit: "作成する"
        cancel: "キャンセル"
      edit:
        title: "セッションを編集"
      create:
        success: "セッションを作成しました"
      update:
        success: "セッションを更新しました"
      destroy:
        success: "セッションを削除しました"
```

---

## 5️⃣ 管理者用「セッション作成」画面を作成

```bash
mkdir app/views/admin/reading_sessions/ && touch app/views/admin/reading_sessions/index.html.erb app/views/admin/reading_sessions/new.html.erb app/views/admin/reading_sessions/edit.html.erb app/views/admin/reading_sessions/_form.html.erb
```

- `app/views/admin/reading_sessions/index.html.erb`

```erb
<div class="container mt-5">
  <div class="d-flex justify-content-between align-items-center mb-3">
    <h1 class="h4"><%= t("admin.reading_sessions.index.title") %></h1>
    <%= link_to t("admin.reading_sessions.index.new_button"), new_admin_reading_session_path, class: "btn btn-primary" %>
  </div>

  <table class="table">
    <thead>
      <tr>
        <th><%= t("admin.reading_sessions.index.table_name") %></th>
        <th><%= t("admin.reading_sessions.index.table_texts_count") %></th>
        <th><%= t("admin.reading_sessions.index.table_created_by") %></th>
        <th><%= t("admin.reading_sessions.index.table_created_at") %></th>
        <th><%= t("admin.reading_sessions.index.table_invite_token") %></th>
        <th></th>
      </tr>
    </thead>
    <tbody>
      <% @reading_sessions.each do |session| %>
        <tr>
          <td><%= session.name.presence || t("admin.reading_sessions.index.no_name") %></td>
          <td><%= session.texts.size %></td>
          <td><%= session.creator.email %></td>
          <td><%= session.created_at.strftime("%Y/%m/%d %H:%M") %></td>
          <td><code><%= session.invite_token %></code></td>
          <td class="text-end">
            <%= link_to t("admin.reading_sessions.index.edit"), edit_admin_reading_session_path(session), class: "btn btn-sm btn-outline-primary" %>
            <%= button_to t("admin.reading_sessions.index.destroy"), admin_reading_session_path(session), method: :delete, class: "btn btn-sm btn-outline-danger", form: { class: "d-inline" }, data: { turbo_confirm: t("admin.reading_sessions.index.confirm_destroy") } %>
          </td>
        </tr>
      <% end %>
    </tbody>
  </table>
</div>
```

- `app/views/admin/reading_sessions/new.html.erb`

```erb
<div class="container mt-5" style="max-width: 600px;">
  <h1 class="h4 mb-4"><%= t("admin.reading_sessions.new.title") %></h1>
  <%= render "form", reading_session: @reading_session %>
</div>
```

- `app/views/admin/reading_sessions/edit.html.erb`

```erb
<div class="container mt-5" style="max-width: 600px;">
  <h1 class="h4 mb-4"><%= t("admin.reading_sessions.edit.title") %></h1>
  <%= render "form", reading_session: @reading_session %>
</div>
```

- `app/views/admin/reading_sessions/_form.html.erb`

```erb
<%= form_with model: [:admin, @reading_session] do |f| %>
  <% if @reading_session.errors.any? %>
    <div class="alert alert-danger">
      <ul class="mb-0">
        <% @reading_session.errors.full_messages.each do |msg| %>
          <li><%= msg %></li>
        <% end %>
      </ul>
    </div>
  <% end %>

  <div class="mb-3">
    <%= f.label :name, t("admin.reading_sessions.form.name_label"), class: "form-label" %>
    <%= f.text_field :name, class: "form-control" %>
  </div>

  <div class="mb-3">
    <label class="form-label"><%= t("admin.reading_sessions.form.texts_label") %></label>
    <%= f.collection_check_boxes :text_ids, @texts, :id, :title do |b| %>
      <div class="form-check">
        <%= b.check_box class: "form-check-input" %>
        <%= b.label class: "form-check-label" %>
      </div>
    <% end %>
  </div>

  <%= f.submit t("admin.reading_sessions.form.submit"), class: "btn btn-primary" %>
  <%= link_to t("admin.reading_sessions.form.cancel"), admin_reading_sessions_path, class: "btn btn-outline-secondary ms-2" %>
<% end %>
```

---

## 6️⃣ `InviteRegistrationsController#create` のパスを修正

```ruby
# app/controllers/invite_registrations_controller.rb
-      redirect_to root_path, notice: "登録が完了しました"
+      redirect_to reading_sessions_path, notice: "登録が完了しました。参加できるセッションをご確認ください。"
```

---

### 動作確認

- [x] 管理者でログインし `/admin/reading_sessions/new` にアクセスできること
- [x] テキストを選択せずに送信 → バリデーションエラーが表示されること
- [x] テキストを選択して送信 → 一覧にリダイレクトされ、セッションが表示されること
- [x] 編集ボタンから名前・テキストを変更できること
- [x] 削除ボタンで確認ダイアログが出て、削除できること
- [x] 新規ユーザーが招待 URL から登録後、セッション一覧にリダイレクトされること

---

#### 総学習時間： 1228.5 時間
