# [co-READER](https://github.com/QynToKey/co_reader)（day: 21）：「管理者ダッシュボード」に招待登録機能を実装

## 0️⃣ 実装方針

- 招待フローを `invite_token` に一本化する。
- 未ログインユーザーがセッション招待URLにアクセスした際に、
ログイン・登録後に自動でセッションへ参加できるようにする。
- 不要になった `Invitation` 関連コードは削除する。

---

## 1️⃣ `ReadingSessionsController#index` を修正

👉 *`join` アクションを未ログイン対応にする*

```ruby
# app/controllers/reading_sessions_controller.rb
 class ReadingSessionsController < ApplicationController
   before_action :require_login
+  skip_before_action :require_login, only: [:join]

   def index
-    session = ReadingSession.find_by(invite_token: params[:token])
+    reading_session = ReadingSession.find_by(invite_token: params[:token])

-    if session.nil?
+    if reading_session.nil?
       redirect_to root_path, alert: "無効な招待リンクです"
       return
     end

+    # session[:invite_token] にトークンを保持してログインページへ誘導
+    unless logged_in?
+      session[:invite_token] = params[:token]
+      redirect_to new_session_path, notice: "参加するにはログインまたは登録が必要です"
+      return
+    end
+
-    if session.users.include?(current_user)
+    if reading_session.users.include?(current_user)
       redirect_to reading_session_path(reading_session), notice: "すでに参加しています"
       return
     end

-    session.add_member(current_user)
+    reading_session.add_member(current_user)
     redirect_to reading_session_path(reading_session), notice: "セッションに参加しました"
   end
 end
```

---

## 2️⃣ `SessionsController#create` を修正

👉 *ログイン後はセッションに自動参加*

```ruby
# app/controllers/sessions_controller.rb
  def create
    user = login(params[:email], params[:password], params[:remember_me])
    if user
-     redirect_to root_path, notice: "ログインしました"
+     if (token = session.delete(:invite_token))
+       redirect_to join_reading_sessions_path(token: token), notice: "セッションに参加しました"
+     else
+       redirect_to root_path, notice: "ログインしました"
+     end
    else
      flash.now[:alert] = "メールアドレスまたはパスワードが正しくありません"
      render :new, status: :unprocessable_entity
    end
  end
```

📝 `session[:invite_token]` ：
 「ログインするまでの一時的な預かり場所」で、ログイン完了と同時に消える。

- HTTPセッションから `:invite_token` キーを取り出して削除する
  - キーが存在した → 値（トークン文字列）を返す → if が真 → join へリダイレクト
  - キーが存在しない → nil を返す → if が偽 → 通常のトップへリダイレクト

---

## 3️⃣ `UsersController#create` を修正

👉 *ユーザー登録後はセッションに自動参加*

```ruby
# app/controllers/users_controller.rb
  def create
    @user = User.new(user_params)
    if @user.save
      auto_login(@user)
-     redirect_to root_path, notice: "登録が完了しました"
+     if (token = session.delete(:invite_token))
+      redirect_to join_reading_session_path(token: token),, notice: "セッションに参加しました"
      else
+      redirect_to root_path, notice: "登録が完了しました"
+     end
    else
      render :new, status: :unprocessable_entity
    end
```

---

## 4️⃣ 不要ファイル / コード を削除

### 不要ファイルを削除

- `app/controllers/invite_registrations_controller.rb`
- `app/views/invite_registrations/new.html.erb`
- `app/controllers/admin/invitations_controller.rb`
- `app/views/admin/invitations/index.html.erb`
- `app/models/invitation.rb`

```bash
rm app/controllers/invite_registrations_controller.rb app/views/invite_registrations/new.html.erb app/controllers/admin/invitations_controller.rb app/views/admin/invitations/index.html.erb app/models/invitation.rb
```

### ルーティング

```ruby
# config/routes.rb
Rails.application.routes.draw do
    get  "register", to: "registrations#new",    as: :register
    post "register", to: "registrations#create"
    resource :dashboard, only: %i[show]
-   resources :invitations, only: %i[ index create ]
    resources :members, only: %i[ index new create edit update destroy ]
    resources :texts, only: %i[ index show new create edit update destroy ]
    resources :reading_sessions, only: %i[ index new create edit update destroy ]
        resources :admins, only: %i[ index create destroy ]
  end

-  get "invite/register", to: "invite_registrations#new", as: :new_invite_registration
-  post "invite/register", to: "invite_registrations#create", as: :invite_registration
```

### `Invitations` テーブルを削除

```bash
$ docker compose exec web bin/rails g migration DropInvitations
      invoke  active_record
      create    db/migrate/20260428020941_drop_invitations.rb
```

  ⬇️

```ruby
# db/migrate/20260428020941_drop_invitations.rb
class DropInvitations < ActiveRecord::Migration[8.1]
  def change
    drop_table :invitations
  end
end
```

  ⬇️

```bash
$ docker compose exec web bin/rails db:migrate
== 20260428020941 DropInvitations: migrating ==================================
-- drop_table(:invitations)
   -> 0.0262s
== 20260428020941 DropInvitations: migrated (0.0262s) =========================
```

---

#### そう隔週時間： 1270.1 時間
