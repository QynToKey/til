# [卒制](https://github.com/QynToKey/HowLongWillItLast) (MVP 後)： LearningTheme CRUD

## 0️⃣ 実装方針

1. `LearningTheme` モデルにテーマ上限（3件）のバリデーションを追加
2. `LearningThemesController` を作成し `destroy` アクションだけ実装
3. `users#show`（マイページ）の「学習テーマ」カードを複数テーマ対応にリニューアル

| やること | 方針 |
| --- | --- |
| テーマの「編集」 | `users/edit` のまま維持（現状どおり） |
| テーマの「作成」 | 複数テーマの導線は保留。`users/create` の初回作成はそのまま |
| テーマの「一覧」 | `users/show`（マイページ）に表示 |
| テーマの「削除」 | `users/show` からボタンで削除 |
| `LearningTheme` のビュー | 作らない |
| `LearningThemesController` | `destroy` のみ実装 |

---

## 1️⃣ `LearningTheme` に上限3件のバリデーションを追加

> 上限を指定する理由：

- このアプリは多数の学習テーマを管理する目的ではない
- 登録件数を制限することで UI が煩雑になることを回避する

```ruby
class LearningTheme < ApplicationRecord
  belongs_to :user

  ・・・
  validate :within_theme_limit

  ・・・

  private

  def within_theme_limit
    return unless user
    # 新規作成時のみチェック（編集時はスキップ）
    if new_record? && user.learning_themes.count >= 3
      errors.add(:base, "学習テーマは３件までしか登録できません")
    end
  end
end
```
