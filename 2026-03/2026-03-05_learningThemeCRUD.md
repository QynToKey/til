# [卒制](https://github.com/QynToKey/HowLongWillItLast) (day 8)：LearningTheme CRUD

---

## 実装

### 1️⃣ ルーティング に `resources` を追加

```ruby
resources :learning_themes
```

  ---

### 2️⃣ Controller を生成

```bash
docker compose exec web rails g controller LearningThemes new create index edit update destroy
```

  ---

### 3️⃣ `create` メソッドを定義

```ruby
class LearningThemesController < ApplicationController
  ・・・

  def create
    @learning_theme = current_user.learning_themes.build(learning_theme_params)

    if @learning_theme.save
      redirect_to learning_themes_path, notice: "テーマを作成しました"
    else
      render :new, status: :unprocessable_entity
    end
  end
  ・・・
end
```

👉 *`current_user` は Sorcery が `ApplicationController` に注入しているメソッドで、明示的には定義されていないが使うことができる*

```ruby
# 実装したコード
current_user.learning_themes.build

# 内部的に実行されるプログラム
LearningTheme.new(user_id: current_user.id)
```

---

### 4️⃣
