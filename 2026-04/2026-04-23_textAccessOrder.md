# [co-READER](https://github.com/QynToKey/co_reader)（day: 16）： セッション内テキストのアクセス順序制御

## 0️⃣ 実装方針

> 「誰が（`membership_id`）・どのテキストを（`text_id`）・読んだか」というレコード `text_readings` が存在すれば「既読」、なければ「未読」と判定する。

- `enforce_order: true` のセッションでテキスト X を開こうとしたとき
  - X より `position` が小さいテキストの `text_readings` レコードがすべて存在するか？
  - 存在しなければリダイレクト

---

## 1️⃣ `TextReadings` テーブルを作成

```bash
$ docker compose exec web bin/rails g migration CreateTextReadings membership_id:references text_id:references read_at:datetime
      invoke  active_record
      create    db/migrate/20260423034802_create_text_readings.rb
```

  ⬇️

```ruby
# db/migrate/20260423034802_create_text_readings.rb
class CreateTextReadings < ActiveRecord::Migration[8.1]
  def change
    create_table :text_readings do |t|
      t.references :membership, null: false, foreign_key: true
      t.references :text, null: false, foreign_key: true
      t.datetime :read_at, null: false

      # t.timestamps
      t.index [:membership_id, :text_id], unique: true
    end
  end
end
```

  ⬇️

```bash
$ docker compose exec web bin/rails db:migrate
== 20260423034802 CreateTextReadings: migrating ===============================
-- create_table(:text_readings)
   -> 0.0578s
== 20260423034802 CreateTextReadings: migrated (0.0579s) ======================
```

---

## 2️⃣ `TextReading` モデルを作成

```bash
touch app/models/text_reading.rb
```

```ruby
# app/models/text_reading.rb
class TextReading < ApplicationRecord
  # ユーザーが特定のテキストをいつ読んだかを記録するためのモデル。
  belongs_to :membership
  belongs_to :text

  validates :read_at, presence: true
  validates :text_id, uniqueness: { scope: :membership_id }
end
```

---

## 3️⃣
