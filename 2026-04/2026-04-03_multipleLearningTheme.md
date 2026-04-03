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
    User::THRESHOLDS.find { |t| t[:hours] > total_hours }
  end
```

  ⬇️ 動作確認

```bash
>> theme = User.first.learning_themes.first
=>
#<LearningTheme:0x0000ffff99928e18
...
>> theme.total_learning_minutes
=> 60616
>> theme.next_threshold
=> {:hours=>2500, :label=>"中級"}
```

---

## 2️⃣ `UsersController#show` の修正

```ruby
# app/controllers/users_controller.rb
$ cat app/controllers/users_controller.rb
class UsersController < ApplicationController
  ・・・
  # before_action :set_learning_theme, only: %i[edit] ⬅️ edit だけなので set_learning_theme メソッドごと削除

  def show
    # ユーザープロフィール画面では、ユーザーの学習テーマとタグごとの学習時間を表示するためにデータを準備する
    @learning_themes = current_user.learning_themes

    # 各 learning_theme ごとにタグの学習時間を計算してまとめる
    @tag_summaries_by_theme = @learning_themes.each_with_object({}) do |theme, hash|
      hash[theme.id] = theme.tags.map do |tag|
        {
          name: tag.name,
          hours: (current_user.total_learning_minutes_by_tag(tag) / 60.0).round(1)
        }
      end
    end
  end

  def edit
    # 編集画面では、最初の learning_theme を表示する（複数登録されている場合も最初のものだけ編集可能）
    @learning_theme = current_user.learning_themes.first
  end
```

## 3️⃣

---

## 補足

- 現状 `learning_records` は `user` に直接紐づくのではなく、`learning_theme` を介して紐づいているため、`learning_records` は `user_id` と `learning_theme_id` の両方を持っている。
- 設計上は `learning_theme_id` だけで user を特定できるので、`user_id` は冗長とも言える。

  ⬇️ リファクタリング issue として積んでおく

> [learning_records / tags / todos の user_id を learning_theme 経由に統一する#177](https://github.com/users/QynToKey/projects/5/views/1?pane=issue&itemId=172175287&issue=QynToKey%7CHowLongWillItLast%7C177)
