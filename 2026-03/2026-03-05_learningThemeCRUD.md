# [卒制](https://github.com/QynToKey/HowLongWillItLast) (day 8)：LearningTheme CRUD

---

## 実装

### 0️⃣ 実装手順

1. `index`
2. `create`
3. `update`
4. `destroy`
5. `new` / `edit`
6. view

- 📝 `create` アクションから定義することで、

  - 必要なパラメータ
  - バリデーション
  - リダイレクト先

  が確定するため、Controller のロジックが安定する。

- 📝 view は controller のインスタンス変数に依存するため、controller を先に完成させる方がスムーズ。

- 📝 `show` は MVP 段階では不要と判断。

  ---

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

### 3️⃣ `create` アクションを定義

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

| コード | 役割 |
| --- | --- |
| `require_login` | 未ログインアクセス防止 |
| `set_learning_theme` | 対象となるレコードを取得 |
| `learning_theme_params` | strong parameters |
| `current_user.learning_themes` | ユーザーとデータのドメイン関係を保証 |

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

### 7️⃣ `update` / `destroy` アクションを追加

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

  ---

8️⃣ `new` アクションを追加

```ruby
def new
  @learning_theme = current_user.learning_themes.build
end
```

  ---

### ９ view を作成し、LearningThemes への動線を追加

- `index.html.erb`

```ruby
<h1>学習テーマ</h1>

<%= link_to "新しいテーマ", new_learning_theme_path %>

<ul>
  <% @learning_themes.each do |theme| %>
    <li>
      <strong><%= theme.name %></strong>
      <p><%= theme.description %></p>

      <%= link_to "編集", edit_learning_theme_path(theme) %>
      <%= link_to "削除", learning_theme_path(theme),
            data: { turbo_method: :delete, turbo_confirm: "削除しますか？" } %>
    </li>
  <% end %>
</ul>
```

- `new.html.erb`

```ruby
<h1>学習テーマを作成</h1>

<%= render "form", learning_theme: @learning_theme %>

<%= link_to "戻る", learning_themes_path %>
```

- `edit.html.erb`

```ruby

<h1>学習テーマを編集</h1>

<%= render "form", learning_theme: @learning_theme %>

<%= link_to "戻る", learning_themes_path %>
```

- `_form.html.erb`

```bash
touch app/views/learning_themes/_form.html.erb
```

```ruby
<%= form_with model: learning_theme do |f| %>

  <% if learning_theme.errors.any? %>
    <div>
      <h2><%= learning_theme.errors.count %>件のエラーがあります</h2>
      <ul>
        <% learning_theme.errors.full_messages.each do |msg| %>
          <li><%= msg %></li>
        <% end %>
      </ul>
    </div>
  <% end %>

  <div>
    <%= f.label :name, "テーマ名" %><br>
    <%= f.text_field :name %>
  </div>

  <div>
    <%= f.label :description, "説明" %><br>
    <%= f.text_area :description %>
  </div>

  <div>
    <%= f.submit %>
  </div>

<% end %>
```

- LearningThemes への動線を追加

```ruby
# app/controllers/user_sessions_controller.rb
def create
  @user = login(params[:email], params[:password])

  if @user
    redirect_to learning_themes_path, notice: "ログインしました"
・・・
end
```

📝 設計メモ：

- 本来なら「ダッシュボード」を用意すべきだろうが、MVP 後に検討するものとする。
- 構想としては、TOP ページから「今日の記録」ヘダイレクトに飛ぶくらいのシンプルな構成を現段階ではイメージしている。

---
