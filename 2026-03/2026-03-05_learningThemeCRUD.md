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

👉 *`user_id` > `learning_theme` というドメイン関係をコードで表現することで、データが常に正しい所有関係のもとに生成される*

  ---

### 4️⃣ `before_action` / `strong parameters` を追加

```ruby
class LearningThemesController < ApplicationController
  before_action :require_login # current_user = nil を許可しない
  ・・・
  private

  def learning_theme_params
    params.require(:learning_theme).permit(:name, :description)
  end
end
```

👉 *未ログインなら `login_path` にリダイレクト*

⚠️ *`user_id` は絶対に permit しない*

  ---

### 5️⃣ `index` アクション

```ruby
  def index
    @learning_themes = current_user.learning_themes.order(created_at: :desc)
  end
```

👉 *`current_user.learning_themes` とすることで、他のユーザーのデータを取得しない*

  ➡️ SQL に `WHERE` が付く

  ```bash
  WHERE user_id = current_user.id
  ```

  ---

### 6️⃣ `set_learning_theme` メソッドを追加

```ruby
class LearningThemesController < ApplicationController
  before_action :set_learning_theme, only: %i[edit update destroy]

  private

  def set_learning_theme
    @learning_theme = current_user.learning_themes.find(params[:id])
  end
end
```

👉 *これにより `edit` / `update` / `destroy` で毎回同じコードを書かなくて済むようになる （ここで `index` を外していることに留意❗️）*

  📝 アクションごとに責務をわける

  | アクション | 呼び出すデータ |
  | --- | --- |
  | `edit` / `update` / `destroy` | @learning_theme |
  | `index` | @learning_themes |

  ---

### 7️⃣ update` / `destroy` アクションを追加

```ruby
  def update
    if @learning_theme.update(learning_theme_params)
      redirect_to learning_themes_path, notice: "テーマを更新しました"
    else
      render :edit, status: :unprocessable_entity
    end
  end

  def destroy
    @learning_theme.destroy
    redirect_to learning_themes_path, notice: "テーマを削除しました"
  end
```
