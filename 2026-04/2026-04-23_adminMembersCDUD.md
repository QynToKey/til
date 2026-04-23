# [co-READER](https://github.com/QynToKey/co_reader)（day: 16_2）： 管理者による参加メンバー CRUD 機能

## 0️⃣ 現状と実装方針

> 現状

`Admin::MembersController` には `index` / `new` / `create` のみ実装されており、編集・削除が未対応。

> 実装方針

管理者がメンバー（一般ユーザー）の情報を編集・削除できる機能を実装する。

---

## 1️⃣ ルーティング

`admin` の `resources :member` に `edit` / `update` / `destroy` を追加

```ruby
# config/routes.rb
resources :members, only: %i[ index new create edit update destroy ]
```

---

## 2️⃣ `Admin::MembersController` に `edit` / `update` / `destroy` を追加

```ruby
# app/controllers/admin/members_controller.rb

class Admin::MembersController < ApplicationController
  before_action :require_admin
  before_action :set_user, only: %i[ edit update destroy ]

  def index
    @users = User.where(role: :member).order(created_at: :desc)
  end

  def new
    @user = User.new
  end

  def create
    @user = User.new(user_params.merge(role: :member))
    if @user.save
      redirect_to admin_members_path, notice: t("admin.members.create.success")
    else
      render :new, status: :unprocessable_entity
    end
  end

  def edit
  end

  def update
    if @user.update(update_params)
      redirect_to admin_members_path, notice: t("admin.members.update.success")
    else
      render :edit, status: :unprocessable_entity
    end
  end

  def destroy
    @user.destroy
    redirect_to admin_members_path, notice: t("admin.members.destroy.success")
  end

  private

  def set_user
    @user = User.find(params[:id])
  end

  def user_params
    params.require(:user).permit(:email, :password, :password_confirmation)
  end

  def update_params
    params.require(:user).permit(:email, :password, :password_confirmation).reject do |_, v|
      v.blank?
    end
  end
end
```

---

## 3️⃣ 「参加者管理」画面を作成 / 更新

- フォーム `app/views/admin/members/_form.html.erb`

```erb
<%= form_with model: @user, url: @user.new_record? ? admin_members_path : admin_member_path(@user) do |f| %>
  <% if @user.errors.any? %>
    <div class="alert alert-danger">
      <ul>
        <% @user.errors.full_messages.each do |msg| %>
          <li><%= msg %></li>
        <% end %>
      </ul>
    </div>
  <% end %>

  <div class="mb-3">
    <%= f.label :email, class: "form-label" %>
    <%= f.email_field :email, class: "form-control" %>
  </div>
  <div class="mb-3">
    <%= f.label :password, t("admin.members.form.password_hint"), class: "form-label" %>
    <%= f.password_field :password, class: "form-control" %>
  </div>
  <div class="mb-3">
    <%= f.label :password_confirmation, class: "form-label" %>
    <%= f.password_field :password_confirmation, class: "form-control" %>
  </div>
  <%= f.submit t("admin.members.form.submit"), class: "btn btn-primary" %>
  <%= link_to t("admin.members.form.cancel"), admin_members_path, class: "btn btn-secondary ms-2" %>
<% end %>
```

- 編集画面 `app/views/admin/members/edit.html.erb`

```erb
<div class="container mt-5" style="max-width: 600px;">
  <h1 class="h4 mb-3"><%= t("admin.members.edit.title") %></h1>
  <%= render "form" %>
</div>
```

- 新規作成画面 `app/views/admin/members/new.html.erb`

```erb
<%# パーシャル化 %>
<div class="container mt-5" style="max-width: 600px;">
  <h1 class="h4 mb-3"><%= t("admin.members.new.title") %></h1>
  <%= render "form" %>
</div>
```

- 一覧画面 `app/views/admin/members/index.html.erb`

```erb
<%# ハードコードされた文字列を i18n に統一 %>
<div class="container mt-5">
  <div class="d-flex justify-content-between align-items-center mb-3">
    <h1 class="h4"><%= t("admin.members.index.title") %></h1>
    <%= link_to t("admin.members.index.new_member"), new_admin_member_path, class: "btn btn-primary" %>
  </div>

  <table class="table">
    <thead>
      <tr>
        <th><%= t("admin.members.index.table_email") %></th>
        <th><%= t("admin.members.index.table_created_at") %></th>
      </tr>
    </thead>
    <tbody>
      <% @users.each do |user| %>
        <tr>
          <td><%= user.email %></td>
          <td><%= user.created_at.strftime("%Y/%m/%d %H:%M") %></td>
          <%# 編集 / 削除ボタンを追加 %>
          <td>
            <%= link_to t("admin.members.index.edit"), edit_admin_member_path(user), class: "btn btn-sm btn-outline-primary" %>
            <%= button_to t("admin.members.index.destroy"), admin_member_path(user), method: :delete, class: "btn btn-sm btn-outline-danger", form: { class: "d-inline" }, data: { turbo_confirm: t("admin.members.index.confirm_destroy") } %>
          </td>
        </tr>
      <% end %>
    </tbody>
  </table>
</div>
```

---

## 4️⃣ `ja.yml` に語彙を追加

```ruby
# config/locales/views/ja.yml
     members:
       index:
         title: "メンバー一覧"
+        new_member: "メンバーを直接登録"
+        table_email: "メールアドレス"
+        table_created_at: "登録日時"
+        edit: "編集"
+        destroy: "削除"
+        confirm_destroy: "本当に削除しますか？"
       new:
         title: "メンバー登録"
-        password_hint: "パスワード（8文字以上）"
-        submit: "登録する"
+      edit:
+        title: "メンバー編集"
+      form:
+        password_hint: "パスワード（変更する場合のみ入力）"
+        submit: "保存する"
+        cancel: "キャンセル"
+      create:
+        success: "メンバーを登録しました"
+      update:
+        success: "メンバー情報を更新しました"
+      destroy:
+        success: "メンバーを削除しました"
```

---

### 総学習時間： 1240.6 時間
