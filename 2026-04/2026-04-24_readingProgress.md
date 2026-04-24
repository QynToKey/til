# [co-READER](https://github.com/QynToKey/co_reader)（day: 17_3）： マイページ（読中 / 読了 ステータス表示機能）

## 0️⃣ 現状と実装方針

> 現状

`memberships` テーブルに `completed` も `reading_progress` も未実装

> 実装方針

これらのカラムを追加するマイグレーションから行う

```text
User（ユーザー）
  └─ Membership（参加記録）※セッションごとに1件
       ├─ reading_session_id  → どのセッションか
       ├─ completed           → セッション全体の読了マーク（#57）
       └─ reading_progress    → 進捗%（#31）
            ↑
            TextReading（テキスト読了記録）の件数から自動計算
```

> 処理の流れ

- **進捗更新**（#31）

1. ユーザーがテキストを読んで「読了にする」を押す
2. `TextReadingsController#create` が `text_reading` を作成
3. 同じタイミングで `reading_progress` を計算・更新
  例）3本中2本読んだ → `(2/3.0 * 100).round` → 67%

- **読了マーク切り替え**（#57）

1. ユーザーがセッション内の全テキストを「読了にする」
2. `TextReadingsController#create` で `progress == 100` になった時点で `completed: true` に自動更新
3. マイページに「読了」バッジが表示される（手動操作・Turbo Stream なし）

---

## 1️⃣ マイグレーション

`completed` と `reading_progress` を `memberships` に追加

```bash
$ docker compose exec web bin/rails g migration AddProgressFieldsToMemberships
      invoke  active_record
      create    db/migrate/20260424095831_add_progress_fields_to_memberships.rb
```

  ⬇️

```ruby
# 20260424095831_add_progress_fields_to_memberships.rb
class AddProgressFieldsToMemberships < ActiveRecord::Migration[8.1]
  def change
    # ユーザーがコースの進捗を追跡できるようにするためのフィールドを追加
    add_column :memberships, :completed, :boolean, null: false, default: false
    # ユーザーがコース内の特定のセクションやレッスンの進捗を追跡できるようにするためのフィールドを追加
    add_column :memberships, :reading_progress, :integer
  end
end
```

  ⬇️

```bash
$ docker compose exec web bin/rails db:migrate
== 20260424095831 AddProgressFieldsToMemberships: migrating ===================
-- add_column(:memberships, :completed, :boolean, {:null=>false, :default=>false})
   -> 0.0220s
-- add_column(:memberships, :reading_progress, :integer)
   -> 0.0019s
== 20260424095831 AddProgressFieldsToMemberships: migrated (0.0241s) ==========
```

---

## 2️⃣ `TextReadingsController#create` に進捗計算を追加

```ruby
# app/controllers/text_readings_controller.rb
    # 進捗を自動更新
    total = @reading_session.texts.count
    read = membership.text_readings.count
    progress = (read.to_f / total * 100).round
    # 進捗と完了状態を更新
    membership.update(
      reading_progress: progress,
      completed: progress == 100
    )
```

👉 *「読了にする」ボタン押下時に `reading_progress` を自動更新*

---

## 3️⃣ `ProfilesController#show` にデータを追加

```ruby
# app/controllers/profiles_controller.rb
  def show
+   @memberships = current_user.memberships
+                           .includes(:reading_session)
+                           .order(created_at: :desc)
  end
```

---

## 4️⃣ 「マイページ」ビュー を更新

👉 *[HowLongWillItLast](https://github.com/QynToKey/HowLongWillItLast)のレイアウトを参照*

```erb
<%# app/views/profiles/show.html.erb &>
<div class="container mt-5" style="max-width: 600px;">
  <h1 class="h5 mb-4"><%= t("profiles.show.title") %> : '<%= current_user.name %>' さん</h1>

  <div class="card mb-4">
    <div class="card-body">
      <h2 class="card-title h6 text-muted"><%= t("profiles.show.profile_section") %></h2>
      <p class="mb-1 small ps-3"><%= t("profiles.show.label_name") %>： <%= current_user.name %></p>
      <p class="mb-3 small ps-3"><%= t("profiles.show.label_email") %>： <%= current_user.email %></p>
      <%= link_to t("profiles.show.edit"), edit_profile_path, class: "btn btn-sm btn-outline-primary" %>
    </div>
  </div>

  <h2 class="h6 text-muted mb-3"><%= t("profiles.show.sessions_title") %></h2>
  <% if @memberships.empty? %>
    <p class="text-muted small"><%= t("profiles.show.no_sessions") %></p>
  <% else %>
    <% @memberships.each do |membership| %>
      <div class="card mb-3" id="<%= dom_id(membership) %>">
        <div class="card-body">
          <h3 class="card-title h6">
            <%= link_to membership.reading_session.name.presence || t("profiles.show.no_name"),
                        reading_session_path(membership.reading_session) %>
          </h3>
          <% if membership.reading_progress.present? %>
            <div class="progress mb-2" style="height: 6px;">
              <div class="progress-bar" role="progressbar"
                   style="width: <%= membership.reading_progress %>%"
                   aria-valuenow="<%= membership.reading_progress %>"
                   aria-valuemin="0" aria-valuemax="100"></div>
            </div>
            <p class="text-muted small mb-2"><%= membership.reading_progress %>%</p>
          <% end %>
          <% if membership.completed? %>
            <span class="badge bg-success"><%= t("profiles.show.completed") %></span>
          <% end %>
        </div>
      </div>
    <% end %>
  <% end %>
</div>
```

---

## 5️⃣ i18n に関連語彙を追加

```ruby
    show:
      title: "マイページ"
      edit: "プロフィールを編集"
+     sessions_title: "参加セッション"
+     no_sessions: "参加しているセッションはありません"
+     no_name: "（名称なし）"
+     completed: "読了"
    edit:
      title: "プロフィール編集"
      password_hint: "パスワード（変更する場合のみ入力）"
```

---

### 総学習時間： 1250.2 時間
