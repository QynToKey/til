# [卒制](https://github.com/QynToKey/HowLongWillItLast) (MVP 後)： Claude Code を使ってリファクタリングしてみる

## 0️⃣ 設定と運用方針

### Claude Code の設定

当面は学習目的で運用したいため、
Claude Code をインストールする前に `settings.json` でグローバルの `deny` （禁止）ルールを設定する。*（ひとまず Git 操作を禁止に設定）*

👉 *allow（自動許可）・ask（確認する）・deny（禁止）の3種類のルールがあり、評価順は deny → ask → allow で、最初にマッチしたルールが優先される*

```bash
$ cat ~/.claude/settings.json
{
  "permissions": {
    "deny": [
      "Bash(git push *)",
      "Bash(git commit *)",
      "Bash(git merge *)",
      "Bash(git rebase *)",
      "Bash(git reset *)"
    ]
  }
}
```

### 今日の issue

`learning_records` / `tags` / `todos` テーブルは `learning_theme_id` を持っており、`learning_theme` を経由して `user` を特定できる。現状 `user_id` を重複保持しているため、これを削除して設計を整理する。

- `learning_records` / `tags` / `todos` から `user_id` カラムを削除するマイグレーション作成
- 各モデルの `belongs_to :user` を削除
- `user_id` を参照しているクエリ・コントローラー・ビューを `learning_theme` 経由に修正
- 動作確認

### issueの進め方

| ステップ | 自分がやること | Claude Codeに頼むこと |
| --- | --- | --- |
| ① | 影響範囲を把握する | リストアップのみ依頼 |
| ② | 修正方針を自分で考える | 方針の妥当性を確認 |
| ③ | 自分でコードを書く | 書いたコードのレビュー依頼 |
| ④ | マイグレーション作成 | 構文の確認のみ |

### Clode Code を起動

```bash
claude
```

---

## 1️⃣ 影響範囲の把握

```bash

> user_idを参照しているファイルと行番号をリストアップして。ファイルの修正は一切しないこと。

 ---
  user_id を参照しているファイルと行番号

  db/schema.rb

  ┌─────┬────────────────────────────────────────────────────┐
  │ 行  │                        内容                        │
  ├─────┼────────────────────────────────────────────────────┤
  │ 24  │ t.bigint "user_id", null: false (learning_records) │
  ├─────┼────────────────────────────────────────────────────┤
  │ 26  │ t.index ["user_id"], ... (learning_records)        │
  ├─────┼────────────────────────────────────────────────────┤
  │ 33  │ t.bigint "user_id", null: false (learning_themes)  │
  ├─────┼────────────────────────────────────────────────────┤
  │ 34  │ t.index ["user_id", "name"], ... (learning_themes) │
  ├─────┼────────────────────────────────────────────────────┤
  │ 35  │ t.index ["user_id"], ... (learning_themes)         │
  ├─────┼────────────────────────────────────────────────────┤
  │ 53  │ t.bigint "user_id", null: false (tags)             │
  ├─────┼────────────────────────────────────────────────────┤
  │ 56  │ t.index ["user_id"], ... (tags)                    │
  ├─────┼────────────────────────────────────────────────────┤
  │ 76  │ t.bigint "user_id", null: false (todos)            │
  ├─────┼────────────────────────────────────────────────────┤
  │ 78  │ t.index ["user_id"], ... (todos)                   │
  └─────┴────────────────────────────────────────────────────┘

  db/migrate/

  ファイル: 20260307024210_create_tags.rb
  行: 10
  内容: add_index :tags, [ :user_id, :name ], unique: true
  ────────────────────────────────────────
  ファイル: 20260328045747_create_learning_themes.rb
  行: 10
  内容: add_index :learning_themes, [ :user_id, :name ], unique:
    true, ...
  ────────────────────────────────────────
  ファイル:
  20260328060103_migrate_learning_theme_to_learning_themes.rb
  行: 6
  内容: user_id: user.id,
  ────────────────────────────────────────
  ファイル:
  20260328060103_migrate_learning_theme_to_learning_themes.rb
  行: 20
  内容: User.find(theme.user_id).update!(...)
  ────────────────────────────────────────
  ファイル: 20260329072729_add_learning_theme_id_to_learning_records
  _and_tags.rb
  行: 8
  内容: LearningTheme.find_by(user_id: record.user_id)
  ────────────────────────────────────────
  ファイル: 20260329072729_add_learning_theme_id_to_learning_records
  _and_tags.rb
  行: 17
  内容: LearningTheme.find_by(user_id: tag.user_id)
  ────────────────────────────────────────
  ファイル:
  20260403152308_fix_learning_records_missing_learning_theme_id.rb
  行: 4
  内容: LearningTheme.find_by(user_id: record.user_id)
  ────────────────────────────────────────
  ファイル: 20260404084417_change_unique_index_on_tags.rb
  行: 3
  内容: remove_index :tags, [ :user_id, :name ]

  config/initializers/sorcery.rb
  ・・・

  test/channels/application_cable/connection_test.rb
  ・・・

  README.md

  ┌─────┬───────────────────────────────────────────────────────┐
  │ 行  │                         内容                          │
  ├─────┼───────────────────────────────────────────────────────┤
  │ 142 │ user_id: bigint / usersテーブルの外部キー（NOT NULL） │
  ├─────┼───────────────────────────────────────────────────────┤
  │ 149 │ user_id: bigint / usersテーブルの外部キー             │
  ├─────┼───────────────────────────────────────────────────────┤
  │ 158 │ user_id: bigint / usersテーブルの外部キー             │
  ├─────┼───────────────────────────────────────────────────────┤
  │ 166 │ user_id: bigint / usersテーブルの外部キー             │
  └─────┴───────────────────────────────────────────────────────┘

  docs/er_diagram.md

  ┌─────┬──────────────────────────────────────┐
  │ 行  │                 内容                 │
  ├─────┼──────────────────────────────────────┤
  │ 77  │ learning_records.user_id は NOT NULL │
  ├─────┼──────────────────────────────────────┤
  │ 131 │ bigint user_id FK "NOT NULL"         │
  ├─────┼──────────────────────────────────────┤
  │ 139 │ bigint user_id FK                    │
  ├─────┼──────────────────────────────────────┤
  │ 150 │ bigint user_id FK                    │
  ├─────┼──────────────────────────────────────┤
  │ 159 │ bigint user_id FK "NOT NULL"         │
  └─────┴──────────────────────────────────────┘

  ---
  app/ 配下には user_id の参照はありませんでした（モデルの
  belongs_to :user から自動解決されているためです）。
  ```

---

## 2️⃣ 修正方針

- `sorcery.rb` と `test.rb` はコメント行なので無視
- ドキュメント類(`README` / `docs/er_diagram.md`)はリファクタリング後に手書きで修正
- `db/schema.rb` に対してマイグレーションを実行（自動で修正）

---

## 3️⃣ マイグレーションファイルを作成

```bash
$ docker compose exec web rails g migration RemoveUserIdFromLearningRecordsTagsTodos
      invoke  active_record
      create    db/migrate/20260406053128_remove_user_id_from_learning_records_tags_todos.rb
```

  ⬇️

```ruby
# db/migrate/20260406053128_remove_user_id_from_learning_records_tags_todos.rb

class RemoveUserIdFromLearningRecordsTagsTodos < ActiveRecord::Migration[8.1]
  def change
    remove_index :learning_records, :user_id # ⬅️ index を先に削除する
    remove_index :tags, :user_id
    remove_index :todos, :user_id

    remove_column :learning_records, :user_id
    remove_column :tags, :user_id
    remove_column :todos, :user_id
  end
end
```

📝 なぜインデックスを先に削除するのか：
インデックスは「カラムを参照して」作られるため、
カラムを先に消すと、インデックスが参照先を失うのでエラーになる。

```markdown
カラム : `user_id`
  └── インデックス `index_tags_on_user_id`  ← `user_id` を参照している
```

※ 依存関係のある順序で削除する、という考え方は外部キーの削除でも同じ

---

## 4️⃣ マイグレーションを実行

```bash
$ docker compose exec web rails db:migrate
== 20260406053128 RemoveUserIdFromLearningRecordsTagsTodos: migrating =========
-- remove_index(:learning_records, :user_id)
   -> 0.0300s
-- remove_index(:tags, :user_id)
   -> 0.0038s
-- remove_index(:todos, :user_id)
   -> 0.0034s
-- remove_column(:learning_records, :user_id)
   -> 0.0074s
-- remove_column(:tags, :user_id)
   -> 0.0022s
-- remove_column(:todos, :user_id)
   -> 0.0020s
== 20260406053128 RemoveUserIdFromLearningRecordsTagsTodos: migrated (0.0492s)
```

  ⬇️

```bash
$ git diff
diff --git a/db/schema.rb b/db/schema.rb
index 2bc35b6..06aa080 100644
--- a/db/schema.rb
+++ b/db/schema.rb
@@ -10,7 +10,7 @@
 #
 # It's strongly recommended that you check this file into your version control system.

-ActiveRecord::Schema[8.1].define(version: 2026_04_04_084417) do
+ActiveRecord::Schema[8.1].define(version: 2026_04_06_053128) do
   # These are extensions that must be enabled in order to support this database
   enable_extension "pg_catalog.plpgsql"

@@ -21,9 +21,7 @@ ActiveRecord::Schema[8.1].define(version: 2026_04_04_084417) do
     t.bigint "learning_theme_id"
     t.date "study_date", null: false
     t.datetime "updated_at", null: false
-    t.bigint "user_id", null: false
     t.index ["learning_theme_id"], name: "index_learning_records_on_learning_theme_id"
-    t.index ["user_id"], name: "index_learning_records_on_user_id"
   end

   create_table "learning_themes", force: :cascade do |t|
@@ -50,10 +48,8 @@ ActiveRecord::Schema[8.1].define(version: 2026_04
_04_084417) do
     t.bigint "learning_theme_id"
     t.string "name", null: false
     t.datetime "updated_at", null: false
-    t.bigint "user_id", null: false
     t.index ["learning_theme_id", "name"], name: "index_tags_on_lea
rning_theme_id_and_name", unique: true
     t.index ["learning_theme_id"], name: "index_tags_on_learning_th
eme_id"
-    t.index ["user_id"], name: "index_tags_on_user_id"
   end

   create_table "todo_tags", force: :cascade do |t|
@@ -73,9 +69,7 @@ ActiveRecord::Schema[8.1].define(version: 2026_04_
04_084417) do
     t.bigint "learning_theme_id", null: false
     t.string "title"
     t.datetime "updated_at", null: false
-    t.bigint "user_id", null: false
     t.index ["learning_theme_id"], name: "index_todos_on_learning_t
heme_id"
-    t.index ["user_id"], name: "index_todos_on_user_id"
   end

   create_table "users", force: :cascade do |t|
@@ -91,14 +85,11 @@ ActiveRecord::Schema[8.1].define(version: 2026_0
4_04_084417) do
   end

   add_foreign_key "learning_records", "learning_themes"
-  add_foreign_key "learning_records", "users"
   add_foreign_key "learning_themes", "users", on_delete: :cascade
   add_foreign_key "record_tags", "learning_records"
   add_foreign_key "record_tags", "tags"
   add_foreign_key "tags", "learning_themes"
-  add_foreign_key "tags", "users"
   add_foreign_key "todo_tags", "tags"
   add_foreign_key "todo_tags", "todos"
   add_foreign_key "todos", "learning_themes"
-  add_foreign_key "todos", "users"
 end
```

---

## 5️⃣ `LearningRecord` / `tag` / `todo` モデルから `belongs_to :user` を削除

```bash
❯ `belongs_to :user`
が残っているファイルと行数をリストアップしてください。

  Searched for 1 pattern (ctrl+o to expand)

⏺ ┌───────────────────────────────┬──────┐
  │           ファイル            │ 行数 │
  ├───────────────────────────────┼──────┤
  │ app/models/tag.rb             │ 2    │
  ├───────────────────────────────┼──────┤
  │ app/models/learning_record.rb │ 2    │
  ├───────────────────────────────┼──────┤
  │ app/models/todo.rb            │ 2    │
  ├───────────────────────────────┼──────┤
  │ app/models/learning_theme.rb  │ 2    │
  └───────────────────────────────┴──────┘

  4ファイルに belongs_to :user が残っています。
```

  ⬇️

| ファイル | 対応 |
| --- | --- |
| `app/models/learning_record.rb` | 削除 |
| `app/models/tag.rb` | 削除 |
| `app/models/todo.rb` | 削除 |
| `app/models/learning_theme.rb` | **残す** |

---

## 6️⃣
