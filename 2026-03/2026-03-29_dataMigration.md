# [卒制](https://github.com/QynToKey/HowLongWillItLast) (MVP 後)： 既存データの移行

## 0️⃣ 前提

`LearningTheme` 再実装に伴う既存データの移行作業の積み残し分を行う。

- `learning_records` に `learning_theme_id` を追加し紐付け
- `tags` に `learning_theme_id` を追加し紐付け

👉 *[実行済みのデータ移行マイグレーション](https://github.com/QynToKey/til/blob/main/2026-03/2026-03-28_learningThemeBasic.md)（`MigrateLearningThemeToLearningThemes`）の `up` メソッドで `User.find_each` して全ユーザーに `learning_themes` レコードを作成しているので、全ユーザーが1件以上 `learning_themes` を持っている。*

---

## 1️⃣ マイグレーションファイルを作成 / 修正 / 実行

```bash
$ docker compose exec web rails g migration AddLearningThemeIdToLearningRecordsAndTags
      invoke  active_record
      create    db/migrate/20260329072729_add_learning_theme_id_to_learning_records_and_tags.rb
```

```ruby
# db/migrate/20260329072729_add_learning_theme_id_to_learning_records_and_tags.rb
class AddLearningThemeIdToLearningRecordsAndTags < ActiveRecord::Migration[8.1]
  def up
    # learning_records に learning_theme_id を追加
    add_reference :learning_records, :learning_theme, foreign_key: true

    # 既存の learning_records を user の最初の learning_theme に紐付け
    LearningRecord.find_each do |record|
      theme = LearningTheme.find_by(user_id: record.user_id)
      record.update_columns(learning_theme_id: theme.id) if theme
    end

    # tags に learning_theme_id を追加
    add_reference :tags, :learning_theme, foreign_key: true

    # 既存の tags を user の最初の learning_theme に紐付け
    Tag.find_each do |tag|
      theme = LearningTheme.find_by(user_id: tag.user_id)
      tag.update_columns(learning_theme_id: theme.id) if theme
    end
  end

  def down
    remove_reference :learning_records, :learning_theme
    remove_reference :tags, :learning_theme
  end
end
```

```bash
$ docker compose exec web rails db:migrate
== 20260329072729 AddLearningThemeIdToLearningRecordsAndTags: migrating =======
-- add_reference(:learning_records, :learning_theme, {:foreign_key=>true})
   -> 0.0257s
-- add_reference(:tags, :learning_theme, {:foreign_key=>true})
   -> 0.0070s
== 20260329072729 AddLearningThemeIdToLearningRecordsAndTags: migrated (0.1102s)
```

---
