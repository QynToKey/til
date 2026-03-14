# [卒制](https://github.com/QynToKey/HowLongWillItLast) (day 16)：ストップウォッチ機能の実装

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
