# [卒制](https://github.com/QynToKey/HowLongWillItLast) (day 2)：認証基盤構築

## gem Sorcery 導入

- `Gemfile`に `gem 'sorcery', '0.16.3'` を記述

  👉 *バージョン固定は将来的な破壊的変更を避けるため*

- `docker compose exec web bundle install`

  ⬇️

  ```bash
  # sorcery が正しく入っているか確認
  $ docker compose exec web bundle info sorcery
  * sorcery (0.16.3)
        Summary: Magical authentication for Rails applications
        Homepage: https://github.com/Sorcery/sorcery
        Path: /usr/local/bundle/gems/sorcery-0.16.3
  ```

- `docker compose exec web rails g sorcery:install`

  ⬇️ initializer を生成

  ```bash
  config/initializers/sorcery.rb
  ```

- `docker compose exec web rails generate sorcery:install user`

  ⬇️ migration ファイル生成

  ```ruby
  # app/models/user.rb
  class User < ApplicationRecord
    authenticates_with_sorcery!
  end
  ```

- `docker compose exec web rails g model User name:string email:string`

  ⬇️ 問題発生⚠️

  ```bash
  $ docker compose exec web rails db:migrate:status
  no configuration file provided: not found
  ```

  ※ 状況整理：

  - `app/models/user.rb` ✅ 存在
  - `db/migrate` ❌ 存在しない
  - `schema.rb` ✅ 存在

  ⬇️ User テーブルをつくる migration を作成

  ```bash
  $ docker compose exec web rails g migration CreateUsers
      invoke  active_record
      create    db/migrate/20260227045255_create_users.rb
  ```

  ⬇️ 生成された migration に手書きでカラム定義を記述

  ```ruby
  class CreateUsers < ActiveRecord::Migration[7.2]
    def change
      create_table :users do |t|
        t.string :name
        t.string :email, null: false
        t.string :crypted_password, null: false
        t.string :salt, null: false

        t.timestamps
      end

      add_index :users, :email, unique: true
    end
  end
  ```

- `docker compose exec web rails db:migrate`

  ⬇️

  ```bash
  == 20260227045255 CreateUsers: migrating ======================================
  -- create_table(:users)
    -> 0.0338s
  -- add_index(:users, :email, {:unique=>true})
    -> 0.0027s
  == 20260227045255 CreateUsers: migrated (0.0366s) =============================
  ```

- `docker compose restart`

---

## User モデルにバリデーションを設定

```ruby
class User < ApplicationRecord
  authenticates_with_sorcery!

  before_validation { email&.downcase! } # 保存時に downcase して正規化

  validates :email,
            presence: true,
            uniqueness: { case_sensitive: false } # Email の重複判定で大文字小文字を区別しない
  validates :password,
            presence:true,
            confirmation: true,
            length: { minimum: 6 },
            if: -> { new_record? || changes[:crypted_password] }  # 新規作成時、またはパスワード変更時のみ実行
  validates :password_confirmation,
            presence: true,
            if: -> { new_record? || changes[:crypted_password] }
end
```

⬇️ `rails c` でテスト

```bash
# エラー発生
>> User.create(email: "", password: "123456", password_confirmation: "123456") app/models/user.rb:2:in <class:User>': wrong argument type Class (expected Module) (TypeError) include Submodules.const_get(mod.to_s.split('_').map(&:capitalize).join)
:
```

⚠️ *Sorcery が「有効化されていないサブモジュールを読み込もうとしている」時に出る典型例 （Sorcery が `Module` を期待しているのに `Class` を読もうとして落ちる）*

⬇️ `config/initializers/sorcery.rb` を確認

```ruby
# この行↓をコメントアウト
Rails.application.config.sorcery.submodules = [:user]
```

👉 *`submodule` をオフにして、Sorcery が読み込みに行かないようにする*

---

## 正常に認証が行われることを確認

### `email`が空白だと認証失敗

```bash
how-long-will-it-last(dev)> User.create(email: "", password: "123456", password_confirmation: "123456")
  TRANSACTION (0.1ms)  BEGIN
  User Exists? (2.1ms)  SELECT 1 AS one FROM "users" WHERE "users"."email" = $1 LIMIT $2  [["email", ""], ["LIMIT", 1]]
  TRANSACTION (0.4ms)  ROLLBACK
=>
#<User:0x0000ffff85f11820
 id: nil,
 name: nil,
 email: "",
 crypted_password:
  "$2a$10$8SzlegzvCzdtyWNIFIRKV.74.sUBQgK9saWrH99B17b...",
 salt: "B1fvEobW3FDFvr5bbeYP",
 created_at: nil,
 updated_at: nil> # 保存失敗
```

👉 *Sorceryはバリデーション前に暗号化するが、保存はされない*

### 正常に入力すると認証成功

```bash
how-long-will-it-last(dev)> User.create(email: "test@test.com", password: "123456", password_confirmation: "123456")
  TRANSACTION (0.3ms)  BEGIN
  User Exists? (2.4ms)  SELECT 1 AS one FROM "users" WHERE "users"."email" = $1 LIMIT $2  [["email", "test@test.com"], ["LIMIT", 1]]
  User Create (4.3ms)  INSERT INTO "users" ("name", "email", "crypted_password", "salt", "created_at", "updated_at") VALUES ($1, $2, $3, $4, $5, $6) RETURNING "id"  [["name", nil], ["email", "test@test.com"], ["crypted_password", "$2a$10$zD3b9QKm6fmuLJaHsVdHQemR9sH5HrWWyS2X4IGoEJe3GoxpWbG6K"], ["salt", "87vKTYQgKsCqTyzGmqhe"], ["created_at", "2026-02-27 06:50:06.360656"], ["updated_at", "2026-02-27 06:50:06.360656"]]
  TRANSACTION (1.0ms)  COMMIT
=>
#<User:0x0000ffff85d1e4c8
 id: 1,
 name: nil,
 email: "test@test.com",
 crypted_password:
  "$2a$10$zD3b9QKm6fmuLJaHsVdHQemR9sH5HrWWyS2X4IGoEJe...",
 salt: "87vKTYQgKsCqTyzGmqhe",
 created_at: "2026-02-27 06:50:06.360656000 +0000",
 updated_at: "2026-02-27 06:50:06.360656000 +0000">
how-long-will-it-last(dev)> User.authenticate("test@test.com", "123456")
  User Load (1.2ms)  SELECT "users".* FROM "users" WHERE "users"."email" = 'test@test.com' ORDER BY "users"."id" ASC LIMIT $1  [["LIMIT", 1]]
=>
#<User:0x0000ffff85d13d48
 id: 1,
 name: nil,
 email: "test@test.com",
 crypted_password:
  "$2a$10$zD3b9QKm6fmuLJaHsVdHQemR9sH5HrWWyS2X4IGoEJe...",
 salt: "87vKTYQgKsCqTyzGmqhe",
 created_at: "2026-02-27 06:50:06.360656000 +0000",
 updated_at: "2026-02-27 06:50:06.360656000 +0000">
```

👉 *パスワード暗号化 ⭕️ / バリデーション通過 ⭕️ / 保存成功 ⭕️*

### 間違ったパスワードを入力したとき

```bash
how-long-will-it-last(dev)> User.authenticate("test@test.com", "wrong")
  User Load (3.1ms)  SELECT "users".* FROM "users" WHERE "users"."email" = 'test@test.com' ORDER BY "users"."id" ASC LIMIT $1  [["LIMIT", 1]]
=> nil # 認証失敗
```

---

## Git tips

### 📝 `run` と `exec`

#### `docker compose run`

- 新しい一時コンテナを作って実行 *(サーバーが起動していなくても実行可能)*
- ポートやネットワーク設定が異なることがある *(状態が分離される可能性あり)*
- 実行後コンテナは終了する

#### `docker compose exec`

- 今動いているコンテナの中で実行
- 通常の開発ではこちらが正解
- Railsサーバと同じ環境で実行できる

---

### リポジトリをリセットする

| コマンド | 役割り |
| --- | --- |
| `git restore .` | 変更済みファイルを最後のcommit状態に戻す |
| `git clean -fd` | Git管理外ファイルを削除する |
| `git clean -fdn` | 削除対象を確認する |
