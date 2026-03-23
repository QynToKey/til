# [卒制](https://github.com/QynToKey/HowLongWillItLast) (MVP 後)：「総学習時間」を表示

## 0️⃣ 背景

- このアプリが最優先すべきスコアは「総学習時間」であることを再確認した。
  それに伴い、タグの上位カテゴリとして「学習テーマ」を段階的に再実装する。

- その基盤整備の第一弾として、以下を実装する。
  - 「学習テーマ」カラムを追加し、将来的な複数テーマ対応への基盤を整える
  - このアプリの最重要スコアである「総学習時間」をTOPページおよび学習ログ一覧に表示

---

## 1️⃣ `users` テーブルに `learning_theme` カラム追加

```bash
$ docker compose exec web rails g migration AddLearningThemeToUsers learning_theme:string
      invoke  active_record
      create    db/migrate/20260321093259_add_learning_theme_to_users.rb
```

  ⬇️ 生成されたマイグレーションファイルを確認

```ruby
# db/migrate/20260321093259_add_learning_theme_to_users.rb
class AddLearningThemeToUsers < ActiveRecord::Migration[8.1]
  def change
    add_column :users, :learning_theme, :string
  end
end
```

👉 *Railsのデフォルトは `null: true`（NULL許可）なのでそのまま*

  ⬇️ `db:migrate`

```bash
$ docker compose exec web rails db:migrate
== 20260321093259 AddLearningThemeToUsers: migrating ==========================
-- add_column(:users, :learning_theme, :string)
   -> 0.0090s
== 20260321093259 AddLearningThemeToUsers: migrated (0.0090s) =================
```

---

## 2️⃣ ビューに実装

### ユーザー登録フォーム

```ruby
# app/views/users/new.html.erb および app/views/users/edit.html.erb
    <div class="mb-3">
      <%= f.label :learning_theme, class: "form-label" %>
      <%= f.text_field :learning_theme, class: "form-control",
      placeholder: "※ 学習テーマを入力してください（例：英語、ギター）" %>
    </div>
```

### TOP ページ

```ruby
# app/views/home/index.html.erb
    <p class='small mt-3'>総学習時間： <%= (current_user.total_learning_minutes / 60.0).round(1) %> 時間</p>
```

### 学習ログ

```ruby
# app/views/learning_records/index.html.erb
  <div>
    <p class='small mt-3'>総学習時間： <%= (current_user.total_learning_minutes / 60.0).round(1) %> 時間</p>
  </div>
```

---

## 3️⃣ i18n 対応

```yml
# config/locales/views/ja.yml
ja:
  activerecord:
    ・・・
    attributes:
      user:
        ・・・
        learning_theme: 学習テーマ
```

---
