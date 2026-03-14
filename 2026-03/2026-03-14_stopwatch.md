# [卒制](https://github.com/QynToKey/HowLongWillItLast) (day 16)：ストップウォッチ機能の実装

## 0️⃣ 設計の見直し

### 現状と対応方針

1. 現行は `started_at` / `ended_at` を Rails 側で管理する設計
2. 「経過時間計測機能」は JavaScript 側の責務として分離可能
3. 2 の場合、Rails は「経過時間 `elapsedTime`」のデータのみを管理すればよい

⬇️ 以上の判断から、以下のように設計変更する

1. `started_at` / `ended_at` カラムを廃止 *（LearningRecord のデータは「日」を跨ぐことがないので、将来的な機能拡張を見込んでも不要と判断）*
2. `calculate_duration` メソッド *（JavaScript 側が責務を担うため不要）*
3. `ended_at_after_started_at` バリデーション *（JavaScript 側が責務を担うため不要）*
