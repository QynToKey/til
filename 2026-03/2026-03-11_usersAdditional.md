# [卒制](https://github.com/QynToKey/HowLongWillItLast) (day 13)：Users 機能 CRUD

---

## Users 機能の残りを実装

### 1️⃣ ルーティングに `edit` / `update` / `destroy` を追加

```ruby
resources :users, only: %i[new create edit update destroy]
```

---

### 2️⃣ コントローラーに `edit` / `update` / `destroy` を追加

```ruby
class UsersController < ApplicationController
  skip_before_action :require_login, only: [ :new, :create ]
  before_action :set_user, only: %i[edit update destroy]

  def new
    @user = User.new
  end

  def create
    @user = User.new(user_params)

    if @user.save
      # 新規登録後は「今日の記録」へ遷移する
      redirect_to new_learning_record_path, notice: "登録が完了しました"
    else
      render :new, status: :unprocessable_entity
    end
  end

  def edit
  end

  def update
    if @user.update(user_params)
      redirect_to root_path, notice: "ユーザー情報を更新しました"
    else
      render :edit, status: :unprocessable_entity
    end
  end

  def destroy
    @user.destroy
    redirect_to root_path, notice: "ユーザーを削除しました"
  end

  private

  def user_params
    params.require(:user).permit(:name, :email, :password, :password_confirmation)
  end

  def set_user
    @user = current_user
  end
end
```

---

### 3️⃣ ログインユーザーへ表示するヘッダーに「マイページ」への動線を追加

```ruby
# app/views/shared/_header.html.erb
  <nav>
    <% if logged_in? %>
      <%= link_to "HOME", root_path %> |
      # id には current_user を渡す
      <%= link_to "マイページ", edit_user_path(current_user) %> |
      <%= link_to "ログアウト", logout_path, data: { turbo_method: :delete } %>
    <% else %>
      <%= link_to "HOME", root_path %> |
      <%= link_to "ログイン", login_path %> |
      <%= link_to "新規登録", new_user_path %>
    <% end %>
  </nav>
```

👉 *「ユーザー情報編集」ページへの動線は、当該ページでも表示されるため、違和感のない呼称として「マイページ」としておく*

---

### 4️⃣ 「ユーザー設定」画面を作成

```bash
touch app/views/users/edit.html.erb
```

⬇️

```ruby
<h1>マイページ</h1>

<% if @user.errors.any? %>
  <div style="color: red;">
    <h2><%= @user.errors.count %>件のエラーがあります</h2>
    <ul>
      <% @user.errors.full_messages.each do |message| %>
        <li><%= message %></li>
      <% end %>
    </ul>
  </div>
<% end %>

<%= form_with model: @user do |f| %>
  <div>
    <%= f.label :name %><small>（ニックネーム可）</small><br>
    <%= f.text_field :name %>
  </div>

  <div>
    <%= f.label :email %><br>
    <%= f.email_field :email %>
  </div>

  <div>
    <%= f.label :password %><br>
    <%= f.password_field :password %>
  </div>

  <div>
    <%= f.label :password_confirmation %><br>
    <%= f.password_field :password_confirmation %>
  </div>

  <div>
    <%= f.submit "更新" %>
  </div>

    <%= button_to "学習ログへ戻る",  learning_records_path, method: :get %>
    <%= button_to "削除", user_path(@user), method: :delete,
    data: { turbo_confirm: "本当に削除しますか？" } %>
<% end %>

```

---

### 5️⃣ `destroy` アクションを修正

```ruby
  def destroy
    @user.destroy
    # ⬇️ Sorcery でログアウトするメソッドを追加
    logout
    redirect_to root_path, notice: "ユーザーを削除しました"
  end
```

---

## DB の保存先を Neon に移行

Render の PostgreSQL の無料枠には期限があるため、Neon に移行した。

### 1️⃣ Neon の `Connection string` を取得

- GitHub アカウントでサインイン
- 初期設定（PostgreSQL のバージョンは最新を指定）
- Project dashboard から `Connection string` を開き、文字列をコピー

### 2️⃣ Render の `DATABASE_URL` に文字列をペースト

- Render の **Dashboard > 当該 Project > MANAGE > Environment** を開く
- **Environment Variables** の `DATABASE_URL` を編集し、Neon で取得した `Connection string` をペースト

⬇️

「Save」すると自動でデプロイが始まるので、エラーにならなければ成功

⚠️ *新しい DB を使うため既存データは引き継がれないので注意*
