# [卒制](https://github.com/QynToKey/HowLongWillItLast) (MVP 後)： 複数の `LearningThemes` に対応

## 1️⃣ `LearningTheme` モデルに `total_learning_minutes` / `next_threshold` メソッドを追加

👉 *`User` モデルに定義したものを転用する (`THRESHOLDS` は `User` モデルで定義されているので `User::THRESHOLDS` と書く)*

```ruby
# app/models/learning_theme.rb

  # 総学習時間
  def total_learning_minutes
    learning_records.sum(:duration_minutes)
  end

  # 現在の総学習時間を超える最初の閾値を取得
  def next_threshold
    total_hours = total_learning_minutes / 60.0
    user: :THRESHOLDS.find { |t| t[:hours] > total_hours }
  end
```

---

## 補足

- 現状 `learning_records` は `user` に直接紐づくのではなく、`learning_theme` を介して紐づいているため、`learning_records` は `user_id` と `learning_theme_id` の両方を持っている。
- 設計上は `learning_theme_id` だけで user を特定できるので、`user_id` は冗長とも言える。

  ⬇️ リファクタリング issue として積んでおく

> [learning_records / tags / todos の user_id を learning_theme 経由に統一する#177](https://github.com/users/QynToKey/projects/5/views/1?pane=issue&itemId=172175287&issue=QynToKey%7CHowLongWillItLast%7C177)
