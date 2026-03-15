# [卒制](https://github.com/QynToKey/HowLongWillItLast) (day 16)：ストップウォッチ機能の実装 (1/2)

## 0️⃣ 設計の見直し

### 現状と対応方針

1. 現行は `started_at` / `ended_at` を Rails 側で管理する設計
2. 「経過時間計測機能」は JavaScript 側の責務として分離可能
3. 2 の場合、Rails は「経過時間 `duration_minuets`」のデータのみを管理すればよい

⬇️ 以上の判断から、以下のように設計変更する

1. `started_at` / `ended_at` カラムを廃止 *（LearningRecord のデータは「日」を跨ぐことがないので、将来的な機能拡張を見込んでも不要と判断）*
2. `calculate_duration` メソッド *（JavaScript 側が責務を担うため不要）*
3. `ended_at_after_started_at` バリデーション *（JavaScript 側が責務を担うため不要）*

### Issue 設計（変更後）

```md
- ストップウォッチ機能 : 5sp
  - 設計の見直し / 不要となる実装済みコードの整理 : 1sp
  - JavaScript でストップウォッチ実装（含 ビュー） : 2sp
  - `duration_minutes` の受け渡し処理 : 1sp
  - `duration_minutes` の加算処理 : 1sp
```

---

### 関連ドキュメントの修正

- `README.md`

```md
#### Bテーブル：learning_records

- `user_id` : bigint / usersテーブルの外部キー
- `study_date` : date / 学習日（NOT NULL）
- `duration_minutes` : integer / 学習時間（分単位）
- `content` : text / 学習内容の記録
- `started_at` : datetime / 学習開始時間（ストップウォッチ用・任意） ⬅️ 削除
- `ended_at` : datetime / 学習終了時間（ストップウォッチ用・任意） ⬅️ 削除
- `created_at` : datetime / 作成日時
- `updated_at` : datetime / 更新日時
```

- `docs/er_diagram.md`

```md
- 記録オプションについて
  - `learning_records.duration_minutes` は NULL 許可
  - `learning_records.started_at` は NULL 許可 ⬅️ 削除
  - `learning_records.ended_at` は NULL 許可 ⬅️ 削除
```

```md
  設計方針:

  - `learning_records.duration_minutes` については本アプリの主たる機能に関するが、ユーザー判断で使わないことを敢えて許容したい
  - 「手入力」での記録を許容するため `duration_minutes` は NULL を許可するが、「ストップウォッチ」使用時は `duration_minutes` によって集計する ⬅️ 修正後
  - 各カラムの整合性はアプリケーションレイヤーで担保する
```

```md
    LEARNING_RECORDS {
        bigint id PK
        bigint user_id FK
        date study_date "NOT NULL"
        integer duration_minutes
        text content "NOT NULL"
        datetime started_at ⬅️ 削除
        datetime ended_at ⬅️ 削除
        datetime created_at
        datetime updated_at
    }
```

---

## 1️⃣ 不要となる「実装済みコード」の整理

### `app/models/learning_record.rb`

```ruby
⬇️ 削除
  # 開始時間と終了時間の両方が存在する場合、終了時間は開始時間より後でなければならない
  validate :ended_at_after_started_at
```

```ruby
⬇️ 丸ごと削除
  private

  # 開始時間と終了時間から学習時間を計算して duration_minutes にセットする
  def calculate_duration
    if started_at.present? && ended_at.present?
      self.duration_minutes = ((ended_at - started_at) / 60).to_i
    end
  end

  # 終了時間は開始時間より後でなければならない
  def ended_at_after_started_at
    return if started_at.blank? || ended_at.blank?

    if ended_at <= started_at
      errors.add(:ended_at, "は開始時間より後の時刻を入力してください")
    end
  end
```

---

### `app/controllers/learning_records_controller.rb`

```ruby
  def learning_record_params
    params.require(:learning_record).permit(:study_date, :content, :duration_minutes, tag_ids: [])
  end
```

👉 *カラム `:started_at` / `:ended_at` を削除*

---

### `started_at` / `ended_at` カラムを削除するマイグレーション

```bash
$ docker compose exec web rails g migration RemoveStartedAtAndEndedAtFromLearningRecords started_at:datetime ended_at:datetime
      invoke  active_record
      create    db/migrate/20260314094024_remove_started_at_and_ended_at_from_learning_records.rb
```

👉 *`rails g migration Removeカラム名Fromテーブル名 カラム名:型名`*

⬇️ 生成されたマイグレーションファイルを確認

```ruby
class RemoveStartedAtAndEndedAtFromLearningRecords < ActiveRecord::Migration[7.2]
  def change
    remove_column :learning_records, :started_at, :datetime
    remove_column :learning_records, :ended_at, :datetime
  end
end
```

⬇️ `db:migrate` を実行

```bash
$ docker compose exec web ra
ils db:migrate
== 20260314094024 RemoveStartedAtAndEndedAtFromLearningRecords: migrating =====
-- remove_column(:learning_records, :started_at, :datetime)
   -> 0.0161s
-- remove_column(:learning_records, :ended_at, :datetime)
   -> 0.0014s
== 20260314094024 RemoveStartedAtAndEndedAtFromLearningRecords: migrated (0.0176s)
```

⬇️ `db/schema.rb` で LearningRecord テーブルから `started_at` / `ended_at` カラムが消えていることを確認

```ruby
# db/schema.rb
  create_table "learning_records", force: :cascade do |t|
    t.bigint "user_id", null: false
    t.date "study_date", null: false
    t.integer "duration_minutes"
    t.text "content", null: false
    t.datetime "created_at", null: false
    t.datetime "updated_at", null: false
    t.index ["user_id"], name: "index_learning_records_on_user_id"
  end
```

---
