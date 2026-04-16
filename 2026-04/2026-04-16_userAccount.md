# [co-READER](https://github.com/QynToKey/co_reader)（day: 9）： ユーザー登録フロー

## 0️⃣ 実装方針

管理者が参加者（member）を登録できる仕組みを2方式で構築する。

- 方式A（招待URL）:
 管理者がトークンを発行 → URLをメンバーに共有 → メンバーが自分で登録
- 方式B（直接登録）:
管理者がメールアドレス＋仮パスワードを入力してメンバーを直接作成
- 既存の `AdminRegistrationToken` /  `Admin::RegistrationsController` と同じパターンで実装する。

> 実装フロー

- 方式A（招待URL）
 [管理者] /admin/invitations → 招待URL生成 → URLをコピーして共有
 [参加者] /invite/register?token=xxx → 登録フォーム → 登録完了（member role）

- 方式B（直接登録）
 [管理者] /admin/members/new → メール＋仮パスワードを入力 → 登録完了
 [参加者] /login → 受け取った情報でログイン

---

## 1️⃣ `CreateInvitations` テーブルを作成

```bash
$ docker compose exec web bin/rails g migration CreateInvitations
      invoke  active_record
      create    db/migrate/20260416052918_create_invitations.rb
```

  ⬇️

```ruby
# db/migrate/20260416052918_create_invitations.rb
class CreateInvitations < ActiveRecord::Migration[8.1]
  def change
    create_table :invitations do |t|
      t.string :token, null: false
      t.datetime :used_at

      t.timestamps
    end

    # トークンは一意である必要があるため、インデックスを追加
    add_index :invitations, :token, unique: true
  end
end
```

  ⬇️

```bash
$ docker compose exec web bin/rails db:migrate
== 20260416052918 CreateInvitations: migrating ================================
-- create_table(:invitations)
   -> 0.0316s
-- add_index(:invitations, :token, {:unique=>true})
   -> 0.0035s
== 20260416052918 CreateInvitations: migrated (0.0352s) =======================
```

---

## 2️⃣ `Invitation` モデルを作成

```bash
touch app/models/invitation.rb
```

  ⬇️

```ruby
# app/models/invitation.rb
class Invitation < ApplicationRecord
  # トークンは一度使用されたら再利用できないようにするため、used_atカラムで使用済みかどうかを管理する
  before_create :generate_token

  # トークンがまだ使用されていないかを確認するメソッド
  def available?
    used_at.nil?
  end

  # トークンを使用済みにするメソッド
  def use!
    update!(used_at: Time.current)
  end

  private

  # トークンを生成するメソッド
  def generate_token
    self.token = SecureRandom.urlsafe_base64(32)
  end
end
```

---

## 3️⃣ 招待URL / メンバー登録 のルーティングを追加

```ruby
# config/routes.rb
Rails.application.routes.draw do
  get "up" => "rails/health#show", as: :rails_health_check

  root "sessions#new"

  resource  :session,  only: %i[ new create destroy ],
                        path_names: { new: "login" }
  resources :users,    only: %i[ new create ],
                        path_names: { new: "register" }

  namespace :admin do
    get  "register", to: "registrations#new",    as: :register
    post "register", to: "registrations#create"
    resources :invitations, only: %i[ index create ]
    resources :members, only: %i[ index new create ]
  end

  get "invite/register", to: "invite_registrations#new", as: :new_invite_registration
  post "invite/register", to: "invite_registrations#create", as: :invite_registration
end
```

📝 Rubyのシンボル配列リテラル `%i[]` と `%w[]`

- `%w[]` : 文字列の配列（word）

```ruby
  %w[foo bar baz]  # => ["foo", "bar", "baz"]
```

- `%i[]` : シンボルの配列（integer/identifier）

```ruby
  %i[foo bar baz]  # => [:foo, :bar, :baz]
```

  👉 *区切り文字はスペース（カンマ不可）*

```ruby
  %i[new, create]  # => [:"new,", :create]  ← "new," がそのままシンボル化される
  %i[new create]   # => [:new, :create]      ← 正しい
```

  👉 *`[ ]` を使った通常の配列リテラルはカンマ区切り*

```ruby
  only: [:new, :create, :destroy]   # 通常の書き方（カンマ区切り）
  only: %i[new create destroy]      # %i を使った書き方（スペース区切り）
```

 ⬇️ 生成されたルーティング

```bash
$ docker compose exec web bin/rails routes | grep -E  "invite|invitation|member"
        admin_invitations GET    /admin/invitations(.:format)                                                                      admin/invitations#index
                          POST   /admin/invitations(.:format)                                                                      admin/invitations#create
            admin_members GET    /admin/members(.:format)                                                                          admin/members#index
                          POST   /admin/members(.:format)                                                                          admin/members#create
        new_admin_member GET    /admin/members/new(.:format)                                                                      admin/members#new
  new_invite_registration GET    /invite/register(.:format)                                                                        invite_registrations#new
      invite_registration POST   /invite/register(.:format)                                                                        invite_registrations#create
```

---

## 4️⃣ `Admin::Invitations` コントローラーを作成

```bash
touch app/controllers/admin/invitations_controller.rb
```

```ruby
# app/controllers/admin/invitations_controller.rb
class Admin::InvitationsController < ApplicationController
  # Admin::InvitationsControllerは、管理者がユーザーを招待するための招待URLを発行する機能を提供します。
  before_action :require_admin

  def index
    @invitations = Invitation.order(created_at: :desc)
  end

  def create
    Invitation.create!
    redirect_to admin_invitations_path, notice: "招待URLを発行しました"
  end
end
```

---

## 5️⃣ 「招待URL管理」ビューを作成

> i18n を追加

```ruby
# config/locales/views/ja.yml
ja:
  ・・・
  admin:
    registrations:
      ・・・
    invitations:
      index:
        title: "招待URL管理"
        new_button: "招待URLを発行"
        table_url: "招待URL"
        table_issued_at: "発行日時"
        table_status: "状態"
        status_available: "未使用"
        status_used: "使用済み"
    members:
      index:
        title: "メンバー一覧"
      new:
        title: "メンバー登録"
        password_hint: "パスワード（8文字以上）"
        submit: "登録する"
  invite_registrations:
    new:
      title: "メンバー登録"
      password_hint: "パスワード（8文字以上）"
      submit: "登録する"
```

> 「招待URL管理」画面

```bash
mkdir -p app/views/admin/invitations/ && touch app/views/admin/invitations/index.html.erb
```

```erb
<div class="container mt-5">
  <div class="d-flex justify-content-between align-items-center mb-3">
    <h1 class="h4"><%= t("admin.invitations.index.title") %></h1>
    <%= button_to t("admin.invitations.index.new_button"),
                  admin_invitations_path,
                  method: :post,
                  class: "btn btn-primary" %>
  </div>

  <table class="table">
    <thead>
      <tr>
        <th><%= t("admin.invitations.index.table_url") %></th>
        <th><%= t("admin.invitations.index.table_issued_at") %></th>
        <th><%= t("admin.invitations.index.table_status") %></th>
      </tr>
    </thead>
    <tbody>
      <% @invitations.each do |inv| %>
        <tr>
          <td>
            <% if inv.available? %>
              <%= new_invite_registration_url(token: inv.token) %>
            <% else %>
              <span class="text-muted">—</span>
            <% end %>
          </td>
          <td><%= inv.created_at.strftime("%Y/%m/%d %H:%M") %></td>
          <td>
            <% if inv.available? %>
              <span class="badge text-bg-success"><%= t("admin.invitations.index.status_available") %></span>
            <% else %>
              <span class="badge text-bg-secondary"><%= t("admin.invitations.index.status_used") %></span>
            <% end %>
          </td>
        </tr>
      <% end %>
    </tbody>
  </table>
</div>
```

---

## 6️⃣ `Admin::Members` コントローラーを作成

```bash
touch app/controllers/admin/members_controller.rb
```

```ruby
# app/controllers/admin/members_controller.rb
class Admin::MembersController < ApplicationController
  # Admin::MembersControllerは、管理者がメンバーを管理するための機能を提供します。
  before_action :require_admin

  def index
    @users = User.where(role: :member).order(created_at: :desc)
  end

  def new
    @user = User.new
  end

  def create
    # user_paramsにrole: :memberをマージして、新しいユーザーを作成します。これにより、管理者がメンバーを登録する際に、常にroleがmemberになるようになります。
    @user = User.new(user_params.merge(role: :member))
    if @user.save
      redirect_to admin_members_path, notice: "メンバーを登録しました"
    else
      render :new, status: :unprocessable_entity
    end
  end

  private

  # Strong Parametersを使用して、ユーザーの登録に必要なパラメータを許可するメソッド
  def user_params
    params.require(:user).permit(:email, :password, :password_confirmation)
  end
end
```

---

## 7️⃣ `member/index` / `member/new` 画面を作成

- `member/index`

```bash
mkdir -p app/views/admin/members/ && touch app/views/admin/members/index.html.erb
```

```erb
<div class="container mt-5">
  <div class="d-flex justify-content-between align-items-center mb-3">
    <h1 class="h4"><%= t("admin.members.index.title") %></h1>
    <%= link_to "メンバーを直接登録", new_admin_member_path, class: "btn btn-primary" %>
  </div>
</div>
```

 👉 *現状は空の一覧*

- `member/new`

```bash
touch  app/views/admin/members/new.html.erb
```

```erb
<div class="container mt-5" style="max-width: 600px;">
  <h1 class="h4 mb-3"><%= t("admin.members.new.title") %></h1>

  <%= form_with model: @user, url: admin_members_path do |f| %>
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
      <%= f.label :password, t("admin.members.new.password_hint"), class: "form-label" %>
      <%= f.password_field :password, class: "form-control" %>
    </div>
    <div class="mb-3">
      <%= f.label :password_confirmation, class: "form-label" %>
      <%= f.password_field :password_confirmation, class: "form-control" %>
    </div>
    <%= f.submit t("admin.members.new.submit"), class: "btn btn-primary" %>
  <% end %>
</div>
```

---

## 8️⃣ `InviteRegistrations` コントローラーを作成

```bash
touch app/controllers/invite_registrations_controller.rb
```

```ruby
# app/controllers/invite_registrations_controller.rb
class InviteRegistrationsController < ApplicationController
  # InviteRegistrationsControllerは、招待URLを使用してユーザーが登録するための機能を提供します。
  skip_before_action :require_login

  def new
    @token_record = find_available_token
    unless @token_record
      redirect_to new_session_path, alert: "無効または使用済みのトークンです"
      return
    end
    @user = User.new
  end

  def create
    @token_record = find_available_token
    unless @token_record
      redirect_to new_session_path, alert: "無効または使用済みのトークンです"
      return
    end

    @user = User.new(user_params.merge(role: :member))
    if @user.save
      @token_record.use!
      auto_login(@user)
      redirect_to root_path, notice: "登録が完了しました"
    else
      render :new, status: :unprocessable_entity
    end
  end

  private

  # 招待トークンが有効であればそのレコードを返し、無効であればnilを返すメソッド
  def find_available_token
    record = Invitation.find_by(token: params[:token])
    record&.available? ? record : nil
  end

  # Strong Parametersを使用して、ユーザーの登録に必要なパラメータを許可するメソッド
  def user_params
    params.require(:user).permit(:email, :password, :password_confirmation)
  end
end
```

---

## 9️⃣ 「メンバー登録」画面を作成

```bash
mkdir -p app/views/invite_registrations/ && touch app/views/invite_registrations/new.html.erb
```

```erb
<div class="container mt-5" style="max-width: 600px;">
  <h1 class="h4 mb-3"><%= t("invite_registrations.new.title") %></h1>

  <%= form_with model: @user, url: invite_registration_path(token: params[:token]) do |f| %>
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
      <%= f.label :password, t("invite_registrations.new.password_hint"), class: "form-label" %>
      <%= f.password_field :password, class: "form-control" %>
    </div>
    <div class="mb-3">
      <%= f.label :password_confirmation, class: "form-label" %>
      <%= f.password_field :password_confirmation, class: "form-control" %>
    </div>
    <%= f.submit t("invite_registrations.new.submit"), class: "btn btn-primary" %>
  <% end %>
</div>
```

---

## 検証手順

> 方式A（招待URL）

- [ ] 管理者でログイン → /admin/invitations にアクセス
- [ ] 「招待URLを発行」ボタン → 一覧に表示されることを確認
- [ ] URLをコピー → シークレットウィンドウで開く
- [ ] 登録フォームが表示される → 登録後に member ロールでログインを確認
- [ ] トークンが「使用済み」になることを確認
- [ ] 同じURLに再アクセス → リダイレクトされることを確認

> 方式B（直接登録）

- [ ] 管理者でログイン → /admin/members/new にアクセス
- [ ] メール＋パスワードを入力 → 登録
- [ ] シークレットウィンドウでそのメール＋パスワードでログイン → member ロールで入れることを確認
