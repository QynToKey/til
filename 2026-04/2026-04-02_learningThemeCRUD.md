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

📝 `validates` と `validate` :

- `validates` → 属性名を渡すメソッド
- `validate` → カスタムメソッド名を渡すメソッド

  ⬇️ コンソールで確認

```bash
>> user.learning_themes.create(name: "テスト1")
=>
#<LearningTheme:0x0000ffffb4b277d0
 id: 5,
 user_id: 2,
 name: "テスト1",
 created_at: "2026-04-02 09:42:00.738102000 +0000",
 updated_at: "2026-04-02 09:42:00.738102000 +0000">

>> user.learning_themes.create(name: "テスト2")
=>
#<LearningTheme:0x0000ffffb2b13700
 id: 6,
 user_id: 2,
 name: "テスト2",
 created_at: "2026-04-02 09:42:09.959705000 +0000",
 updated_at: "2026-04-02 09:42:09.959705000 +0000">

>> result = user.learning_themes.create(name: "テスト3")
  TRANSACTION (0.6ms)  ROLLBACK
=>
#<LearningTheme:0x0000ffffb41d2090

>> result.errors.full_messages
=> ["学習テーマは３件までしか登録できません"]
```

---

## 2️⃣ コントローラーを作成

👉 *今回はビューも作らず、中身もシンプルなので `touch` コマンドで作成する（`g` コマンドだと余計なファイルまで生成してしまうので、かえってオプション指定が複雑になってしまう）*

```bash
touch app/controllers/learning_themes_controller.rb

# g コマンドを使う場合

docker compose exec web rails generate controller learning_themes --no-helper --no-assets --no-view-specs --skip-template-engine
```

```ruby
# app/controllers/learning_themes_controller.rb
class LearningThemesController < ApplicationController
  before_action :set_learning_theme, only: %i[destroy]

  def destroy
    @learning_theme.destroy
    redirect_to user_path(current_user), notice: "学習テーマを削除しました"
  end

  private

  def set_learning_theme
    # current_user のテーマの中からのみ検索し、他ユーザーのテーマを操作できないようにする
    @learning_theme = current_user.learning_themes.find(params[:id])
  end
end
```

---
