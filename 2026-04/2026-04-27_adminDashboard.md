# [co-READER](https://github.com/QynToKey/co_reader)（day: 20）： 管理者ダッシュボード

## 0️⃣ 背景

- admin ユーザーのヘッダーに散在する3リンク（招待URL管理・メンバー一覧・セッション管理）を整理し、起点となる「管理者ダッシュボード」を実装する。
- ダッシュボードから各管理画面へ遷移できる構成にし、ヘッダーをシンプルにする。

---

## 1️⃣ ルーティングに「管理者ダッシュボード」を追加

```ruby
# config/routes.rb
resource :dashboard, only: %i[show]
```

---

## 2️⃣ `Admin::DashboardController` を作成

```bash
touch app/controllers/admin/dashboards_controller.rb
```

```ruby
# app/controllers/admin/dashboards_controller.rb
class Admin::DashboardsController < ApplicationController
  # 管理者ダッシュボードのコントローラー
  before_action :require_admin

  # 管理者が作成したReadingSessionの一覧と、未使用のAdminRegistrationTokenの数を表示
  def show
    @reading_sessions = ReadingSession
                          .where(created_by_id: current_user.id)
                          .includes(:memberships)
                          .order(created_at: :desc)
    @unused_tokens_count = AdminRegistrationToken.where(used_at: nil).count
  end
end
```

---

## 3️⃣ セッションごとの参加者管理機能 を実装

### `Admin::MembersController#index` にセッションフィルタを追加

👉 *管理者が作成したReadingSessionのメンバーの一覧を、セッションごとに絞り込んで表示*

```ruby
# app/controllers/admin/members_controller.rb
  def index
-    @users = User.joins(memberships: :reading_session).where(reading_sessions: { created_by_id: current_user.id }).where(role: :member).distinct.order(created_at: :desc) # current_user が作成したセッションに所属するメンバーを取得し、作成日時の降順で表示する
+    sessions_scope = ReadingSession.where(created_by_id: current_user.id)
+
+    if params[:session_id].present?
+      @current_session = sessions_scope.find_by(id: params[:session_id])
+      @users = @current_session&.members&.where(role: :member)&.order(created_at: :desc) || User.none
+    else
+      @users = User.joins(memberships: :reading_session).where(reading_sessions: { created_by_id: current_user.id }).where(role: :member).distinct.order(created_at: :desc)
+    end
  end
```

- `@current_session` はビューでセッション名を表示するために使う
- `find_by` で見つからない場合（他人のセッションIDを渡された場合）は `User.none` を返し安全に処理する
- `ReadingSession#members` は既存のアソシエーション（`has_many :members, through: :memberships, source: :user`）を利用

### 「メンバー一覧」ビューにセッション名を表示

```erb
<%# app/views/admin/members/index.html.erb %>
   <div class="d-flex justify-content-between align-items-center mb-3">
-    <h1 class="h4"><%= t("admin.members.index.title") %></h1>
+    <h1 class="h4">
+      <% if @current_session %>
+        <%= t("admin.members.index.title_with_session", session_name: @current_session.name.presence || t("admin.members.index.no_name")) %>
+      <% else %>
+        <%= t("admin.members.index.title") %>
+      <% end %>
+    </h1>
```

---

## 4️⃣「管理者ダッシュボード」ビューを作成

```bash
mkdir -p app/views/admin/dashboard/ && touch app/views/admin/dashboard/show.html.erb
```

```erb
<div class="container mt-5" style="max-width: 600px;">
  <h1 class="h5 mb-4"><%= t("admin.dashboard.show.title") %></h1>

  <div class="card mb-4">
    <div class="card-body">
      <h2 class="card-title h6 text-muted"><%= t("admin.dashboard.show.links_title") %></h2>
      <div class="d-flex gap-2 mt-2">
        <%= link_to t("admin.dashboard.show.invitations"), admin_invitations_path, class: "btn btn-sm btn-outline-secondary" %>
        <%= link_to t("admin.dashboard.show.members"),     admin_members_path,     class: "btn btn-sm btn-outline-secondary" %>
      </div>
    </div>
  </div>

  <div class="card mb-4">
    <div class="card-body">
      <h2 class="card-title h6 text-muted"><%= t("admin.dashboard.show.invitations_status") %></h2>
      <p class="small ps-3 mb-0">
        <%= t("admin.dashboard.show.unused_tokens", count: @unused_tokens_count) %>
      </p>
    </div>
  </div>

  <h2 class="h6 text-muted mb-3"><%= t("admin.dashboard.show.sessions_title") %></h2>
  <% if @reading_sessions.empty? %>
    <p class="text-muted small"><%= t("admin.dashboard.show.no_sessions") %></p>
  <% else %>
    <% @reading_sessions.each do |rs| %>
      <div class="card mb-3">
        <div class="card-body">
          <h3 class="card-title h6"><%= rs.name.presence || t("admin.dashboard.show.no_name") %></h3>
          <p class="small text-muted mb-2">
            <%= t("admin.dashboard.show.participants", count: rs.memberships.size) %>
          </p>
          <%= link_to t("admin.dashboard.show.members"), admin_members_path(session_id: rs.id), class: "btn btn-sm btn-outline-secondary" %>
          <%= link_to t("admin.dashboard.show.edit"), edit_admin_reading_session_path(rs), class: "btn btn-sm btn-outline-primary" %>
        </div>
      </div>
    <% end %>
  <% end %>
</div>
```

---

## 5️⃣ ヘッダーを修正

👉 *`admin?` ブロック内の3リンクを1リンクに置き換え*

```erb
<%# app/views/shared/_header.html.erb %>
       <% if current_user&.admin? %>
-        <%= link_to t("layouts.application.invitations"), admin_invitations_path, class: "nav-link" %>
-        <%= link_to t("layouts.application.members"), admin_members_path, class: "nav-link" %>
-        <%= link_to t("layouts.application.reading_sessions"), admin_reading_sessions_path, class: "nav-link" %>
+        <%= link_to t("layouts.application.dashboard"), admin_dashboard_path, class: "nav-link" %>
       <% elsif current_user&.superadmin? %>
         <%= link_to t("layouts.application.admins"), admin_admins_path, class: "nav-link" %>
       <% end %>
```

---

## 6️⃣ i18n に関連語彙を追加

```ruby
# config/locales/views/ja.yml
  layouts:
    application:
      ・・・
+       dashboard: "管理者ダッシュボード"

  admin:
+   dashboard:
+     show:
+       title: "管理者ダッシュボード"
+       links_title: "管理メニュー"
+       invitations: "招待URL管理"
+       members: "メンバー一覧"
+       invitations_status: "招待URL"
+       unused_tokens: "未使用の招待URL：%{count} 件"
+       sessions_title: "セッション一覧"
+       no_sessions: "作成したセッションはありません"
+       no_name: "（名称なし）"
+       participants: "参加者数：%{count} 人"
+       members: "メンバー管理"
+       edit: "編集"

    members:
      index:
        title: "メンバー一覧"
+       title_with_session: "%{session_name} のメンバー"
+       no_name: "（名称なし）"
```

---

### 総学習時間： 1263.2 時間
