# [co-READER](https://github.com/QynToKey/co_reader)（day: 15）： セッション内テキストの並び順制御機能

## 1️⃣ `position` / `enforce_order` カラムを追加

```bash
$ docker compose exec web bin/rails g migration AddPositionToReadingSessionTexts position:integer
      invoke  active_record
      create    db/migrate/20260422015628_add_position_to_reading_session_texts.rb
```

```bash
$ docker compose exec web bin/rails g migration AddEnforceOrderToReadingSessions enforce_order:boolean
      invoke  active_record
      create    db/migrate/20260422015933_add_enforce_order_to_reading_sessions.rb
```

  ⬇️

```ruby
# db/migrate/20260422015628_add_position_to_reading_session_texts.rb
class AddPositionToReadingSessionTexts < ActiveRecord::Migration[8.1]
  def change
    add_column :reading_session_texts, :position, :integer
  end
end
```

```ruby
# db/migrate/20260422015933_add_enforce_order_to_reading_sessions.rb
class AddEnforceOrderToReadingSessions < ActiveRecord::Migration[8.1]
  def change
    add_column :reading_sessions, :enforce_order, :boolean, default: false, null: false
  end
end
```

  ⬇️

```bash
$ docker compose exec web bin/rails db:migrate
== 20260422015628 AddPositionToReadingSessionTexts: migrating =================
-- add_column(:reading_session_texts, :position, :integer)
   -> 0.0094s
== 20260422015628 AddPositionToReadingSessionTexts: migrated (0.0095s) ========

== 20260422015933 AddEnforceOrderToReadingSessions: migrating =================
-- add_column(:reading_sessions, :enforce_order, :boolean, {:default=>false, :null=>false})
   -> 0.0186s
== 20260422015933 AddEnforceOrderToReadingSessions: migrated (0.0187s) ========
```

---

## 2️⃣
