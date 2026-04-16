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

## 3️⃣ ルーティングを追加

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

## 4️⃣

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
