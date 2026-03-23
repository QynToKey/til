# [卒制](https://github.com/QynToKey/HowLongWillItLast) (MVP 後)：TOP ページをリニューアル

## 0️⃣ 実装方針

TOP ページを実務的かつシンプルなものに更新する。

- アプリの機能や特徴を、未登録ユーザーにも簡潔に伝える
- 説明文を簡素に
- ログインユーザーに「総学習時間」を視認性高く表示する
- マイルストーンを４つ設定する
  - 1000時間(初級)
  - 2500時間(中級)
  - 5000時間(上級)
  - 10000時間(エキスパート)

---

## 1️⃣ User モデルに「マイルストーン」のメソッドを実装

```ruby
# app/models/user.rb
class User < ApplicationRecord
  ・・・

  # 総学習時間のマイルストーンを設定
  THRESHOLDS = [
    { hours: 1000, label: "初級" },
    { hours: 2500, label: "中級" },
    { hours: 5000, label: "上級" },
    { hours: 10000, label: "エキスパート" },
  ].freeze

  ・・・

  # 現在の総学習時間を超える最初の閾値を取得
  def next_threshold
    total_hours = total_learning_minutes / 60.0
    THRESHOLDS.find { |t| t[:hours] > total_hours }
  end
```

📝 `.freeze` ：
定数を凍結（変更不可）するメソッド。
Rubyでは定数でも破壊的メソッドで中身を変えられてしまうので、`.freeze` をつけることで意図しない変更を防ぐ。

  ⬇️ ターミナルで動作確認

```bash
>> current_user = User.first
  User Load (2.0ms)  SELECT "users".* FROM "users" ORDER BY "users"."id" ASC LIMIT $1  [["LIMIT", 1]]
=>
#<User:0x0000ffffa6fdb410
...
>> current_user.next_threshold
  LearningRecord Sum (2.7ms)  SELECT SUM("learning_records"."duration_minutes") FROM "learning_records" WHERE "learning_records"."user_id" = $1  [["user_id", 2]]
=> {:hours=>1000, :label=>"初級"}
```

---

## 2️⃣
