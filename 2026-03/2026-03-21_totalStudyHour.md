# [卒制](https://github.com/QynToKey/HowLongWillItLast) (MVP 後)：「総学習時間」を表示

## 0️⃣

---

### `users` テーブルに `learning_theme` カラム追加

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

  ⬇️

```bash
$ docker compose exec web rails db:migrate
== 20260321093259 AddLearningThemeToUsers: migrating ==========================
-- add_column(:users, :learning_theme, :string)
   -> 0.0090s
== 20260321093259 AddLearningThemeToUsers: migrated (0.0090s) =================
```
