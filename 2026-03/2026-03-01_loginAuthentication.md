# [卒制](https://github.com/QynToKey/HowLongWillItLast) (day 4)：ログイン認証

## 設計方針

### 動線について (`require_login` の適用について)

- 「未登録ユーザー」および「非ログインユーザー」も学習記録画面に入力できるようにしたい。
  - アプリを実際に触ってみるお試し体験を提供したい
  - ただし、データを保存するためにはログイン必須とする
  ➡️ 未登録ユーザーには「ユーザー登録画面」への動線を、非ログインユーザーへは「ログイン」への動線を用意する

- 「未ログインユーザー」が `create` アクションで弾かれた場合、それまでの入力内容が破棄されないようにしたい。
  ➡️ ログイン画面へたんにリダイレクトせず、ログインを促すフラッシュメッセージは必須

### ログインURL について

- `UserSessionsController` を作成してセッション管理を行うが、UX 設計はユーザーフレンドリーとしたいのでログインURLは `/login` とする。

---

## 実装

### 1️⃣ `UserSessionsController' を作成

```bash
rails g controller UserSessions new
```

---

### 2️⃣ ルーティング

```ruby
# config/routes.rb
get 'login',  to: 'user_sessions#new'
post 'login',  to: 'user_sessions#create'
delete 'logout', to: 'user_sessions#destroy'
```

⬇️ `login_path` がつくられる

`localhost:3000/rails/info/routes` で確認 ⭕️

| Helper | HTTP Verb | Path | Controller#Action |
| --- | --- | --- | --- |
| login_path | GET | /login(.:format) | user_sessions#new |
| login_path | POST | /login(.:format) | user_sessions#create |

---

### 3️⃣ ログイン画面を作成

```ruby
# user_sessions/new.html.erb
<h1>ログイン</h1>

# ↓セッションは model ではないので url を指定
<%= form_with url: login_path, local: true do |f| %>
  <div>
    <%= label_tag :email %>
    <%= text_field_tag :email %>
  </div>

  <div>
    <%= label_tag :password %>
    <%= password_field_tag :password %>
  </div>

  <div>
    <%= submit_tag "ログイン" %>
  </div>
<% end %>
```

⭕️ `localhost:3000/login` で表示確認

❌ ログインの動作確認でエラー発生 *(ログインボタンを押下しても何も起こらない)*

#### エラー対応

##### 1. ログイン情報入力フォームに `autocomplete` 属性を付加

  ```bash
  # コンソールの警告
  [DOM] Input elements should have autocomplete attributes (suggested: "current-password"): (More info: https://goo.gl/9p2vKq) <input type=​"password" name=​"password" id=​"password">​
  ```

  ⬇️ ログインフォームに `autocomplete` 属性を付加

  ```ruby
  # user_sessions/new.html.erb
  <%= text_field_tag :email, nil, autocomplete: "email" %>
  :
  <%= password_field_tag :password, nil, autocomplete: "current-password" %>
  ```

##### 2. PWA 関連のメタタグを修正

  📝 **PWA (Progressive Web Apps)** ：
    WebサイトをiOSアプリやAndroidアプリのように使えるようにする技術のこと

  ```bash
  # コンソールの警告
  <meta name="apple-mobile-web-app-capable" content="yes"> is deprecated. Please include <meta name="mobile-web-app-capable" content="yes">
  ```

  ⬇️ `application.html.erb` のメタタグを修正

  ```ruby
  # application.html.erb
  <!-- <meta name="apple-mobile-web-app-capable" content="yes"> -->
  <!-- PWA関連のメタタグ警告を解消するために上記の行を修正 -->
  <meta name="mobile-web-app-capable" content="yes">
  ```

  👉 *AppleのPWA対応が不完全なため、`mobile-web-app-capable` を使用することで警告を回避*

---

### 4️⃣ `UserSessionsController' の実装

```ruby
class UserSessionsController < ApplicationController
  def new
  end

  def create
    @user = login(params[:email], params[:password]) # login メソッド

    if @user
      redirect_to root_path, notice: "ログインしました"
    else
      flash.now[:alert] = "メールアドレスまたはパスワードが違います"
      render :new, status: :unprocessable_entity # 失敗時は 422 を返す
    end
  end

  def destroy
    logout
    redirect_to root_path, notice: "ログアウトしました"
  end
end
```

### 5️⃣ ログイン失敗時のフラッシュメッセージを実装

```ruby
# application.html.erb
  <body>
  <% flash.each do |type, message| %>
    <div class="flash <%= type %>">
      <%= message %>
    </div>
  <% end %>

    <%= yield %>
  </body>
  ```

---
