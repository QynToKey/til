# [co-READER](https://github.com/QynToKey/co_reader)（day: 17_2）： マイページ（プロフィール編集機能）

## 0️⃣ 現状と実装方針

- ヘッダーの「セッション一覧」リンクが管理者・メンバー両方に表示されている。
- マイページ実装後にこのリンクを『マイページ』へ置き換える。

---

## 1️⃣ ルーティングを追加

```ruby
# config/routes.rb
resource :profile, only: %i[ show edit update ]
```

---

## 2️⃣ `ProfileController` を作成

```bash
touch app/controllers/profiles_controller.rb
```

```ruby
# app/controllers/profiles_controller.rb
class ProfilesController < ApplicationController
  before_action :require_login

  def show
  end

  def edit
  end

  def update
    if current_user.update(profile_params)
      redirect_to profile_path, notice: t("profiles.update.success")
    else
      render :edit, status: :unprocessable_entity
    end
  end

  private

  # ユーザーがプロフィールを更新する際に許可されたパラメータをフィルタリング
  def profile_params
    params.require(:user).permit(:name, :email, :password, :password_confirmation).reject do |_, v|
      v.blank?
    end
  end
end
```

---

## 3️⃣ 「マイページ」ビューを作成

👉 *ここでは最低限の実装。後続 issue で肉付けする*

```bash
mkdir -p app/views/profiles/ && touch app/views/profiles/show.html.erb app/views/profiles/edit.html
.erb
```

```erb
<%# app/views/profiles/show.html.erb %>
<div class="container mt-5" style="max-width: 600px;">
  <h1 class="h4 mb-3"><%= t("profiles.show.title") %></h1>
  <p><%= current_user.name %></p>
  <p><%= current_user.email %></p>
  <%= link_to t("profiles.show.edit"), edit_profile_path, class: "btn btn-outline-primary" %>
</div>
```

```erb
<%# app/views/profiles/edit.html.erb %>
<div class="container mt-5" style="max-width: 600px;">
  <h1 class="h4 mb-3"><%= t("profiles.edit.title") %></h1>
  <%= form_with model: current_user, url: profile_path do |f| %>
    <% if current_user.errors.any? %>
      <div class="alert alert-danger">
        <ul>
          <% current_user.errors.full_messages.each do |msg| %>
            <li><%= msg %></li>
          <% end %>
        </ul>
      </div>
    <% end %>
    <div class="mb-3">
      <%= f.label :name, class: "form-label" %>
      <%= f.text_field :name, class: "form-control" %>
    </div>
    <div class="mb-3">
      <%= f.label :email, class: "form-label" %>
      <%= f.email_field :email, class: "form-control" %>
    </div>
    <div class="mb-3">
      <%= f.label :password, t("profiles.edit.password_hint"), class: "form-label" %>
      <%= f.password_field :password, class: "form-control" %>
    </div>
    <div class="mb-3">
      <%= f.label :password_confirmation, class: "form-label" %>
      <%= f.password_field :password_confirmation, class: "form-control" %>
    </div>
    <%= f.submit t("profiles.edit.submit"), class: "btn btn-primary" %>
    <%= link_to t("profiles.edit.cancel"), profile_path, class: "btn btn-secondary ms-2" %>
  <% end %>
</div>
```

---

## 4️⃣ ヘッダーを修正

```erb
<%# app/views/shared/_header.html.erb %>
-      <% if current_user %>
-        <%= link_to "セッション一覧", reading_sessions_path, class: "nav-link" %>
-        <%= button_to t("layouts.application.logout"), session_path, method: :delete, class: "btn btn-outline-danger ms-2" %>
-      <% end %>
+      <% if current_user %>
+        <% unless current_user.admin? %>
+          <%= link_to t("layouts.application.mypage"), profile_path, class: "nav-link" %>
+        <% end %>
+        <%= button_to t("layouts.application.logout"), session_path, method: :delete, class: "btn btn-outline-danger ms-2" %>
+      <% end %>
```

---

## 5️⃣ i18n に関連語彙を追加

```ruby
# /config/locales/views/ja.yml
   layouts:
     application:
+      mypage: "マイページ"
       logout: "ログアウト"
       ・・・
       destroy:
         success: "セッションを削除しました"
+  profiles:
+    show:
+      title: "マイページ"
+      edit: "プロフィールを編集"
+    edit:
+      title: "プロフィール編集"
+      password_hint: "パスワード（変更する場合のみ入力）"
+      submit: "保存する"
+      cancel: "キャンセル"
+    update:
+      success: "プロフィールを更新しました"
   invite_registrations:
     new:
       title: "メンバー登録"
```

---

### 以上をデプロイ後、#103 の積み残し issue を実行

```text
開発者自身の name を登録後、マイグレーション②で default を削除
```

---

## 6️⃣ `UsersController` / `InviteRegistrationsController' を修正

```ruby
# app/controllers/users_controller.rb
  def new
    @user = User.new
-   @user.name = nil # プレースホルダーが表示されるように、nameをnil
に設定する
   end

  ・・・

  def user_params
-    params.require(:user).permit(:email, :password, :password_confirmation)
+    params.require(:user).permit(:name, :email, :password, :password_
confirmation)
  end
end
```

```ruby
# app/controllers/invite_registrations_controller.rb
  def new
    @user = User.new
-   @user.name = nil # プレースホルダーが表示されるように、nameをnil
に設定する
  end

  ・・・

  def user_params
-   params.require(:user).permit(:email, :password, :password_confirm
ation)
+   params.require(:user).permit(:name, :email, :password, :password_
confirmation)
  end
end
```

---

## 7️⃣ `Users` モデルから `default: "未設定"` を削除

```bash
$ docker compose exec web bin/rails g migration RemoveDefaultFromUsersName
      invoke  active_record
      create    db/migrate/20260424073849_remove_default_from_users_name.rb
```

  ⬇️

```ruby
# 20260424073849_remove_default_from_users_name.rb
class RemoveDefaultFromUsersName < ActiveRecord::Migration[8.1]
  def change
    change_column_default :users, :name, from: "未設定", to: nil
  end
end
```

  ⬇️

```bash
$ docker compose exec web bin/rails db:migrate
== 20260424073849 RemoveDefaultFromUsersName: migrating =======================
-- change_column_default(:users, :name, {:from=>"未設定", :to=>nil})
   -> 0.0423s
== 20260424073849 RemoveDefaultFromUsersName: migrated (0.0424s) ==============
```

---

### 総学習時間： 1247.0 時間
