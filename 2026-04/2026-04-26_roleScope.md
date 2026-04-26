# [co-READER](https://github.com/QynToKey/co_reader)（day: 19）： 権限スコープとロール設計の整備

## 1️⃣ Admin コントローラーのスコープ制限

👉 *セッション内のメンバー（= 管理者が招待したメンバー）のみをスコープに置く*

### `Admin::InvitationsController`

```ruby
# app/controllers/admin/invitations_controller.rb
   def index
-   @invitations = Invitation.order(created_at: :desc)
+   @invitations = Invitation.where(invited_by: current_user).order(created_at: :desc)
   end
```

### `Admin::ReadingSessionsController`

```ruby
# app/controllers/admin/reading_sessions_controller.rb
  def index
-   @reading_sessions = ReadingSession.includes(:creator, :texts).order(created_at: :desc)
+   @reading_sessions = ReadingSession.where(created_by_id: current_user.id).includes(:creator, :texts).order(created_at: :desc)
  end

  def set_reading_session
-   @reading_session = ReadingSession.find(params[:id])
+   @reading_session = ReadingSession.where(created_by_id: current_user.id).find(params[:id])
  end
```

### `Admin::TextsController`

```ruby
# app/controllers/admin/texts_controller.rb
def index
-   @texts = Text.includes(:uploader).order(created_at: :desc)
+   @texts = Text.where(uploaded_by: current_user.id).includes(:uploader).order(created_at: :desc)
end

def set_text
-   @text = Text.find(params[:id])
+   @text = Text.where(uploaded_by: current_user.id).find(params[:id])
```

### `Admin::MembersController`

```ruby
# app/controllers/admin/members_controller.rb
  def index
-   @texts = Text.includes(:uploader).order(created_at: :desc)
+   @texts = Text.where(uploaded_by: current_user.id).includes(:uploader).order(created_at: :desc)
  end

  def set_text
-   @text = Text.find(params[:id])
+   @text = Text.where(uploaded_by: current_user.id).find(params[:id])
```

---

## 2️⃣ ヘッダーの「マイページ」リンクを修正

👉 *`unless` ブロックを削除して全 `current_user` に「マイページ」を表示*

```erb
<&# app/views/shared/_header.html.erb %>
      <% if current_user %>
-       <% unless current_user.admin? || current_user.superadmin? %>
        <%= link_to t("layouts.application.mypage"), profile_path, class: "nav-link" %>
-     <% end %>
```

---

## 「開発者」`superadmin` の「管理者管理」機能

> 設計

- 「開発者」は「管理者」を招待登録する。
- 「開発者」は「管理者」の情報を編集できない。*（`index` / 招待方式による `create` / `destroy` のみ）*
- 「開発者」は「管理者によるセッション」の内容を関知しない。

---

### 3️⃣ `ApplicationController` に `require_superadmin` を追加

```ruby
# app/controllers/application_controller.rb
  # 管理者権限が必要なアクションの前に呼び出されるメソッド
  def require_admin
    redirect_to root_path, alert: t("errors.not_authorized") unless current_user&.admin? || current_user&.admin?
  end

  # 開発者権限が必要なアクションの前に呼び出されるメソッド
  def require_superadmin
    redirect_to root_path, alert: t("errors.not_authorized") unless current_user&.superadmin?
  end
```

---

### 4️⃣ ルーティングに `resources :admins` を追加

```ruby
# config/routes.rb
  namespace :admin do
    get  "register", to: "registrations#new",    as: :register
    post "register", to: "registrations#create"
    resources :invitations, only: %i[ index create ]
    resources :members, only: %i[ index new create edit update destroy ]
    resources :texts, only: %i[ index show new create edit update destroy ]
    resources :reading_sessions, only: %i[ index new create edit update destroy ]
+   resources :admins, only: %i[ index create destroy ]
  end
```

---

### 5️⃣ `Admin::AdminsController` を作成

```bash
touch app/controllers/admin/admins_controller.rb
```

```ruby
# app/controllers/admin/admins_controller.rb
class Admin::AdminsController < ApplicationController
  # 開発者が管理者を管理するためのコントローラー
  before_action :require_superadmin
  before_action :set_admin_user, only: %i[destroy]

  def index
    @admins = User.where(role: :admin).order(created_at: :desc)
  end

  # AdminRegistrationTokenを生成し、そのトークンを含む招待URLを作成して、管理者一覧ページにリダイレクト。
  def create
    token = AdminRegistrationToken.create!
    invite_url = admin_register_url(token: token.token)
    redirect_to admin_admins_path, notice: "招待URL: #{invite_url}"
  end

  def destroy
    @admin_user.destroy
    redirect_to admin_admins_path, notice: t("admin.admins.destroy.success")
  end

  private

  # 管理者ユーザーをIDで検索してインスタンス変数@admin_userにセットするためのメソッド
  def set_admin_user
    @admin_user = User.where(role: :admin).find(params[:id])
  end
end
```

---

### 6️⃣ 開発者用「管理者一覧」ビューを作成

```bash
mkdir -p app/views/admin/admins/ && touch app/views/admin/admins/index.html.erb
```

```erb
<%# app/views/admin/admins/index.html.erb %>
<div class="container mt-5">
  <div class="d-flex justify-content-between align-items-center mb-3">
    <h1 class="h4"><%= t("admin.admins.index.title") %></h1>
    <%= button_to t("admin.admins.index.invite"), admin_admins_path, class: "btn btn-primary" %>
  </div>

  <table class="table">
    <thead>
      <tr>
        <th><%= t("admin.admins.index.table_name") %></th>
        <th><%= t("admin.admins.index.table_email") %></th>
        <th><%= t("admin.admins.index.table_created_at") %></th>
        <th></th>
      </tr>
    </thead>
    <tbody>
      <% @admins.each do |admin_user| %>
        <tr>
          <td><%= admin_user.name %></td>
          <td><%= admin_user.email %></td>
          <td><%= admin_user.created_at.strftime("%Y/%m/%d %H:%M") %></td>
          <td>
            <%= button_to t("admin.admins.index.destroy"), admin_admin_path(admin_user), method: :delete, class: "btn btn-sm btn-outline-danger", form: { class: "d-inline" }, data: { turbo_confirm: t("admin.admins.index.confirm_destroy") } %>
          </td>
        </tr>
      <% end %>
    </tbody>
  </table>
</div>
```

---

### 7️⃣ ヘッダーに `superadmin` の分岐を追加

```erb
<%# app/views/shared/_header.html.erb %>
-      <% if current_user&.admin? || current_user&.superadmin? %>
+      <% if current_user&.admin? %>
         <%= link_to t("layouts.application.invitations"), admin_invitations_path, class: "nav-link" %>
         <%= link_to t("layouts.application.members"), admin_members_path, class: "nav-link" %>
         <%= link_to t("layouts.application.reading_sessions"), admin_reading_sessions_path, class: "nav-link" %>
+      <% elsif current_user&.superadmin? %>
+        <%= link_to t("layouts.application.admins"), admin_admins_path, class: "nav-link" %>
       <% end %>
```

---

### 8️⃣ i18n に関連語彙を追加

```ruby
# config/locales/views/ja.yml
  layouts:
    application:
      reading_sessions: "セッション管理（開発用）"
+     admins: "管理者管理"
   sessions:
     new:
       title: "ログイン"

+   admins:
+     index:
+       title: "管理者一覧"
+       table_name: "登録名"
+       table_email: "メールアドレス"
+       table_created_at: "登録日時"
+       destroy: "削除"
+       confirm_destroy: "本当に削除しますか？"
+     destroy:
+       success: "管理者を削除しました"
```

---

#### 総学習時間： 1260.5 時間
