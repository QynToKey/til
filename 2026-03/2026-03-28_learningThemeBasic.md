# [卒制](https://github.com/QynToKey/HowLongWillItLast) (MVP 後)：`LearningTheme` 再実装（基盤構築）

## 0️⃣ 実装方針の確認

以前に一度生成した `LearningTheme` と同名モデルを再実装するため、差分を整理しておく。

| | 古いファイル | 今回の設計 |
| --- | --- | --- |
| `name` | NOT NULL | *※ NULL許可* |
| `description` | あり | なし |
| UNIQUE制約 | `(user_id, name)` | *※ なし* |
| `on_delete: :cascade` | あり | あり |

📝 `name` のUNIQUE制約について：
ユーザーが２つ以上の「学習テーマ」を登録する場合は、無題のテーマを許可してしまうと混乱を招く可能性があるので NOT NULL としたい。

👉 *DB ではなく、アプリケーション側のバリデーションで管理することとする*

```ruby
# モデル側で制御
validates :name, presence: true, if: -> { user.learning_themes.count >= 1 }
```

---

## 1️⃣ `LearningTheme` モデルの生成
