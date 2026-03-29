# [卒制](https://github.com/QynToKey/HowLongWillItLast) (MVP 後)： 既存データの移行

## 0️⃣ 前提

`LearningTheme` 再実装に伴う既存データの移行作業の積み残し分を行う。

- `learning_records` に `learning_theme_id` を追加し紐付け
- `tags` に `learning_theme_id` を追加し紐付け

👉 *[実行済みのデータ移行マイグレーション](https://github.com/QynToKey/til/blob/main/2026-03/2026-03-28_learningThemeBasic.md)（`MigrateLearningThemeToLearningThemes`）の `up` メソッドで `User.find_each` して全ユーザーに `learning_themes` レコードを作成しているので、全ユーザーが1件以上 `learning_themes` を持っている。*

---
