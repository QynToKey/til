# [卒制](https://github.com/QynToKey/HowLongWillItLast) (day 5)：ログアウト機能 / 未ログイン時のアクセス制限

## 実装

### ログイン / ログアウト / 新規登録の動線を追加

```ruby
# application.html.erb
  <% if logged_in? %>
    <%= link_to "ログアウト", logout_path, data: { turbo_method: :delete } %>
  <% else %>
    <%= link_to "ログイン", login_path %>
    <%= link_to "新規登録", new_user_path %>
  <% end %>
  ```

---

### 未ログイン時のアクセス制限

#### 1️⃣ 実装内容の確認

最終仕様：

1. 未ログインでも入力できる
2. 保存ボタン押すと「ログインが必要です」と表示
3. ログイン後、入力内容を保持したまま保存

👉 *3 については `LearningRecord` 機能実装後の作業とし、まずは `require_login` を標準実装する*

📝 *セキュリティはデフォルトで「拒否」とし、必要に応じて例外的に開放するのが安全*

##### アクセス制御ポリシー

| 機能 | 未ログイン可否 |
| --- | --- |
| `LearningRecord new` | ✅ 可（体験用） |
| `LearningRecord create` | ❌ 不可（保存はログイン必須） |
| `LearningTheme` 全般 | ❌ 不可（ログイン必須） |
| 各リソースの `index` アクション | ❌ 不可（ログイン必須） |
| `home#index` | ✅ 可 |
| `Todo` 全般 | ❌ 不可（ログイン必須） |
| ストップウォッチ | ✅ 可 |

### 2️⃣ `require_login` を実装

#### デフォルトは「ログイン必須」

```ruby
class ApplicationController < ActionController::Base
  before_action :require_login
  ・・・
end
```

👉 *アクセス制御はアプリ全体のポリシーのため `ApplicationController` に定義*

#### コントローラーごとにアクセス制限を解除

```ruby
# users_controller.rb / user_sessions_controller.rb
skip_before_action :require_login, only: [:new, :create]
```

👉 *「ユーザー登録」と「ログイン認証」の際はアクセス制限を解除*

```ruby
# home_controller.rb
skip_before_action :require_login, only: [:index]
```

👉 *TOP ページはアクセス制限を解除*

---

## 今回の学び

- `before_action` は「デフォルト拒否」の方針で設計する
- `skip_before_action` は最小限に限定する
- 特殊仕様は後回しにする判断も重要

---

以上で [Render 設定](https://github.com/QynToKey/til/blob/main/2026-01/2026-01-31_render.md) して PR へ

⬇️ Render ビルド中にエラー発生❗️

## 本番環境のDB接続エラー

### エラー内容

```bash
could not translate host name "db"
```

### 原因

Docker用 `host: db` 設定が本番に残っていた。

```ruby
# config/database.yml のデフォルト設定
production:
  <<: *default
  database: how_long_will_it_last_production
  username: how_long_will_it_last
  password: <%= ENV["HOW_LONG_WILL_IT_LAST_DATABASE_PASSWORD"] %>
```

### 対応

```ruby
# config/database.yml を以下に修正
production:
  url: <%= ENV['DATABASE_URL'] %>
```

👉 *PaaSでは `DATABASE_URL` を使う*
