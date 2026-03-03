# [卒制](https://github.com/QynToKey/HowLongWillItLast) (day 6)：UI整備

## 1️⃣ パーシャル `_header.html.erb` を作成

```bash
mkdir -p app/views/shared # 必要に応じて親ディレクトリも作成する
touch app/views/shared/_header.html.erb
```

⬇️

```ruby
# _header.html.erb
<header class="app-header">
  <h1><%= link_to "How Long Will It Last", root_path %></h1>

  <nav>
    <% if logged_in? %>
      <%= link_to "HOME", home_index_path %> |
      <%= link_to "ログアウト", logout_path, data: { turbo_method: :delete } %>
    <% else %>
      <%= link_to "HOME", home_index_path %> |
      <%= link_to "ログイン", login_path %> |
      <%= link_to "新規登録", new_user_path %>
    <% end %>
  </nav>

  <div class="flash-messages">
    <% flash.each do |type, message| %>
      <div class="flash <%= type %>">
        <%= message %>
      </div>
    <% end %>
  </div>
</header>
```

---

## 2️⃣ パーシャルを `application.html.erb` に実装

```ruby
  <body>
    <%= render "shared/header" %>
    <%= yield %>
  </body>
```

⚠️ *既存のログイン／flash の分岐は削除する*

---

## 3️⃣ `application_controller.rb' にヘルパーメソッドを追加

```ruby
class ApplicationController < ActionController::Base
  before_action :require_login
  helper_method :logged_in? # ログイン状態をビューで確認できるようにする
 ・・・
end
```

📝 `logged_in?` メソッド：

- 明示的に `helper_method :logged_in?` を書くことで、 view 側から安全に呼び出せる保証 が得られる
- 今後コードをリファクタリングしたり、別のコントローラで view を表示したりする場合に安全
- Rails の「推奨パターン」に沿うので、メンテナンス性が向上

---

## 4️⃣ サーバーを再起動

```bash
docker compose restart web
```

👉 *`web` はRails アプリのサービス名。再起動後にブラウザで挙動を確認*

---

## 開発環境 / 本番環境で動作確認

[☑️] ログイン状態でヘッダーが正しく切り替わる

[☑️] flash メッセージがヘッダー内で表示される

[☑️] パーシャル呼び出しは 1 回のみ

[☑️] CSS 未設定でもレイアウト崩れがないことを確認
