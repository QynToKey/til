# [卒制](https://github.com/QynToKey/HowLongWillItLast) (day 3)：ユーザー登録機能

## ルーティングを追加

`routes.rb` に `resources` を追加

```ruby
Rails.application.routes.draw do
  get "home/index"
  root "home#index"

  resources :users, only: %i[new create]

  ・・・
end
```

⬇️ ターミナルで確認

```bash
$ docker compose exec web rails routes | grep users
    users    POST /users        users#create
    new_user GET  /users/new    users#new
```

---

## Users コントローラーを作成

```bash
# rails g controller コントローラ名 アクション名
docker compose exec web bin/rails g controller Users new
```

👉 *まずは「ユーザー登録機能」の実装を先行させるため、`new` アクションのみを指定*

### ルーティングの二重定義を修正

```bash
# routes.rb に自動で追加される
get "users/new"
```

⚠️ *既に `resources :users` が定義されているため、`get "users/new"` は「二重定義」になってしまうので、これを削除*

### Users コントローラーの定義

```ruby
# users_controller.rb
class UsersController < ApplicationController
  def new
    @user = User.new
  end

  def create
    @user = User.new(user_params)

    if @user.save
      redirect_to root_path, notice: "登録が完了しました"
    else
      render :new, status: :unprocessable_entity # バリデーション失敗時は 422 を返す
    end
  end

  private

  def user_params
    params.require(:user).permit(:email, :password, :password_confirmation)
  end
end
```

👉 *ビューで `form_with model: @user` が使えるようにするため、空の `User` を渡す*

---

## ユーザー登録フォーム を作成

```ruby
<h1>新規登録</h1>

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
    <%= f.submit "登録" %>
  </div>
<% end %>
```

---

### 動作確認

1.ブラウザから模擬登録
2.`rails c` で確認

```bash
how-long-will-it-last(dev)> User.last
  User Load (0.3ms)  SELECT "users".* FROM "users" ORDER BY "users"."id" DESC LIMIT $1  [["LIMIT", 1]]
=>
#<User:0x0000ffffad265888
 id: 2,
 name: nil,
 email: "test@example.com",
 crypted_password:
  "$2a$10$aAUIIK0s5/ruZQ8T17RVyuewgo9DYMowmlAzq462Fq8...",
 salt: "sDmu_9-TUUkyr-uuMExo",
 created_at: "2026-02-28 08:30:20.942096000 +0000",
 updated_at: "2026-02-28 08:30:20.942096000 +0000">
 ```

✔ `new` → `create` 動作
✔ `strong parameters` OK
✔ DB保存OK
✔ 暗号化OK

---

### バリデーションエラー時のメッセージ

```ruby
# new.html.erb
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
```

---

## Rails tips

### `rails new` の際に `README.md` が上書きされて初期化してしまった

### 対策 1️⃣

`main` ブランチに存在する `README.md` だけを作業ブランチにコピーする

```bash
git checkout main -- README.md
```

👉 *対象は `README.md` だけ*

### 対策 2️⃣

現在のブランチの最新コミット（`HEAD`）にある `README.md` の状態に戻す

```bash
git restore README.md
```

👉 *これも対象は `README.md` だけ*
⚠️ *ただし、ブランチを切る前の `main` の状態とは限らない*

### 現実的な運用

1.作業ブランチで `rails new` 実行
2.直後に「上書きされた必要ファイル」だけ戻す

  ```bash
  git status
  ```

3.必要な差分だけをコミット

  ```bash
  git restore README.md
  ```
