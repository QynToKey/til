# [卒制](https://github.com/QynToKey/HowLongWillItLast) (MVP 後)： タグの UNIQUENESS を修正

## 0️⃣ 現状

同一名のタグを、テーマを超えて登録できない。

  ⬇️

複数テーマ対応に伴って、
`tags` テーブルのUNIQUE制約を
 `(user_id, name)` → `(learning_theme_id, name)` に変更する。

## 1️⃣ マイグレーションファイルを生成

```bash
$ docker compose exec web rails generate migration ChangeUniqueIndexOnTags
      invoke  active_record
      create    db/migrate/20260404084417_change_unique_index_on_tags.rb
```

   ⬇️

```ruby
class ChangeUniqueIndexOnTags < ActiveRecord::Migration[8.1]
  def change
    remove_index :tags, [:user_id, :name]
    add_index :tags, [:learning_theme_id, :name], unique: true
  end
end
```

  ⬇️

```bash
$ docker compose exec web rails db:migrate
== 20260404084417 ChangeUniqueIndexOnTags: migrating ==========================
-- remove_index(:tags, [:user_id, :name])
   -> 0.0112s
-- add_index(:tags, [:learning_theme_id, :name], {:unique=>true})
   -> 0.0096s
== 20260404084417 ChangeUniqueIndexOnTags: migrated (0.0210s) =================

---

### 総学習時間： 1182.3 時間
