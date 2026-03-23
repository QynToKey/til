# [卒制](https://github.com/QynToKey/HowLongWillItLast) (MVP 後)：ダッシュボード機能を実装

## 0️⃣ 背景

`learning_theme` カラムを再実装したが、
「マイページ」が編集フォームと同居している状態だったため、
`users#show` を新規作成して「ダッシュボード」として整備する。

---

## 1️⃣ `users#show` を新規作成

### ルーティングに `show` を追加

```ruby
# config/routes.rb
resources :users, only: %i[new create edit show update destroy]
```

### ビューを実装 `app/views/users/show.html.erb`

```ruby
# app/views/users/show.html.erb
<div class="container mt-5" style="max-width: 600px;">
  <h1 class="h5 mb-4">マイページ : '<%= current_user.name %>' さん</h1>

  <div class="card mb-3">
    <div class="card-body">
      <h2 class="card-title h6 text-muted">プロフィール</h2>
      <p class="mb-1 small ps-3">
        お名前： <%= current_user.name %>
      </p>
      <p class="mb-1 small ps-3">
        email： <%= current_user.email %>
      </p>
      <p class="mb-0 small ps-3">
        パスワード： (変更する場合は<%= link_to 'こちら', edit_user_path(current_user) %>)
      </p>
    </div>
  </div>

  <div class="card mb-3">
    <div class="card-body">
    <h2 class="card-title h6 text-muted">
      学習テーマ： <%= current_user.learning_theme %>
    </h2>
      <p class='small mt-3'>総学習時間： <%= (current_user.total_learning_minutes / 60.0).round(1) %> 時間</p>
      <% if @tag_summaries.any? %>
        <ul class="small mt-2 ps^3">
          <% @tag_summaries.each do |summary| %>
            <li><%= summary[:name] %>： <%= summary[:hours] %> 時間</li>
          <% end %>
        </ul>
      <% end %>
    </div>
  </div>

  <div class="card mb-4">
    <div class="card-body">
      <h2 class="card-title h6 text-muted">タグ</h2>
        <% if current_user.tags.present? %>
          <p class='small mt-3'>設定済み： <%= current_user.tags.map(&:name).join(', ') %></p>
        <% else %>
          <p class="text-muted small mt-3">※ 学習記録にタグを設定すると、タグごとに学習ログを管理できます</p>
        <% end %>
    </div>
  </div>

  <div class="d-flex gap-2 mb-3 ps-2">
    <%= link_to '編集', edit_user_path(current_user), class: "btn btn-sm btn-primary" %>
    <%= link_to '今日の記録', new_learning_record_path(study_date: Date.current), class: "btn btn-sm btn-outline-primary" %>
    <%= link_to 'ログ一覧', learning_records_path, class: "btn btn-sm btn-outline-primary" %>
    <%= link_to 'タグ管理', tags_path, class: "btn btn-sm btn-outline-primary" %>
  </div>
</div>
```

### `UsersController` の 'set_user' コールバックに `show` を追加

```ruby
class UsersController < ApplicationController
  skip_before_action :require_login, only: [ :new, :create ]
  before_action :set_user, only: %i[edit update destroy]
  before_action :set_user, only: %i[show edit update destroy]
```

---

## 2️⃣ リンクの整理

### ヘッダーの「マイページ」リンクを `users#show` に変更

```ruby
# app/views/shared/_header.html.erb
<%= link_to "マイページ", user_path(current_user), class: "nav-link" %>
```

👉 *パスは `user_path(current_user)`*

### ユーザー登録 / ログイン 後の遷移先を `users#show` に変更

```ruby
# app/controllers/users_controller.rb
  def create
    @user = User.new(user_params)

    if @user.save
      auto_login(@user)
      redirect_to user_path(current_user), notice: "登録が完了しました"
    ・・・
  end
```

```ruby
# app/controllers/user_sessions_controller.rb
  def create
    @user = login(params[:email], params[:password])

    if @user
      redirect_to user_path(current_user), notice: "ログインしました"
    ・・・
  end
```

---

#### 総学習時間： 1129.8 時間
