# [卒制](https://github.com/QynToKey/HowLongWillItLast) (MVP 後)： `Tags` のリファクタリング

## 0️⃣ 前提

`LearningTheme` の実装に伴ってテーブル構造が変更されたため、`tags` を`learning_themes` と紐づける必要が生じた。

### 影響範囲の確認

```bash
$ grep -r "tags_path\|tag_path\|edit_tag_path\|new_tag_path" app/views
app/views/learning_records/index.html.erb:  <%= link_to 'タグ管理', tags_path, class: "btn btn-sm btn-outline-primary" %>
app/views/learning_records/index.html.erb:      <%= link_to 'タグを追加する', new_tag_path, class: "btn btn-sm btn-outline-primary" %>
app/views/learning_records/_form.html.erb:        <p class="text-muted small">※ タグがありません。<%= link_to 'タグ管理', tags_path %>から追加してください。</p>
app/views/learning_records/_form.html.erb:      <%= link_to "タグ管理", tags_path, class: "btn btn-sm btn-outline-primary" %>
app/views/tags/index.html.erb:  <%= link_to '新規タグを追加', new_tag_path, class: "btn btn-sm btn-primary" %>
app/views/tags/index.html.erb:            <%= link_to '編集', edit_tag_path(tag), class: "btn btn-sm btn-outline-secondary" %>
app/views/tags/index.html.erb:            <%= link_to '削除', tag_path(tag), class: "btn btn-sm btn-outline-danger", data: { turbo_method: :delete, turbo_confirm: '本当に削除しますか？' } %>
app/views/tags/edit.html.erb:  <%= render 'form', tag: @tag, return_path: tags_path %>
app/views/tags/new.html.erb:  <%= render 'form', tag: @tag, return_path: tags_path %>
app/views/users/edit.html.erb:    <%= link_to "タグ管理", tags_path, method: :get, class: "btn btn-sm btn-outline-primary" %>
app/views/users/show.html.erb:          <%= link_to 'タグを編集する', tags_path, class: "btn btn-sm btn-outline-primary" %>
app/views/users/show.html.erb:          <%= link_to 'タグを追加する', tags_path, class: "btn btn-sm btn-outline-primary" %>
app/views/users/show.html.erb:    <%= link_to 'タグ管理', tags_path, class: "btn btn-sm btn-outline-primary" %>
```

> ルーティング変更の影響範囲

- `app/views/tags/` 内 → 4ファイル
- `app/views/learning_records/` 内 → 2ファイル
- `app/views/users/` 内 → 2ファイル
- `app/controllers/tags_controller.rb` → リダイレクト先など

> 修正後のパスヘルパーの対応表

| 変更前 | 変更後 |
| --- | --- |
| `tags_path` | `learning_theme_tags_path(@learning_theme)` |
| `new_tag_path` | `new_learning_theme_tag_path(@learning_theme)` |
| `edit_tag_path(tag)` | `edit_learning_theme_tag_path(@learning_theme, tag)` |
| `tag_path(tag)` | `learning_theme_tag_path(@learning_theme, tag)` |

---

## 1️⃣ `tags` のルーティングを `learning_themes` 内にネストする

```ruby
# config/routes.rb
  resources :learning_themes do
    resources :tags, only: %i[index new create edit update destroy]
    resources :todos
  end
```

  ⬇️ 確認

```bash
$ docker compose exec web rails routes | grep tag
      learning_theme_tags GET    /learning_themes/:learning_theme_id/tags(.:format)                                                tags#index
                          POST   /learning_themes/:learning_theme_id/tags(.:format)                                                tags#create
  new_learning_theme_tag GET    /learning_themes/:learning_theme_id/tags/new(.:format)                                            tags#new
  edit_learning_theme_tag GET    /learning_themes/:learning_theme_id/tags/:id/edit(.:format)                                       tags#edit
      learning_theme_tag PATCH  /learning_themes/:learning_theme_id/tags/:id(.:format)                                            tags#update
                          PUT    /learning_themes/:learning_theme_id/tags/:id(.:format)                                            tags#update
                          DELETE /learning_themes/:learning_theme_id/tags/:id(.:format)                                            tags#destroy
```

---

## 2️⃣ `tags_controller.rb` を修正

- `set_learning_theme` を追加して `before_action` に設定
- リダイレクト先の `tags_path` を `learning_theme_tags_path(@learning_theme)` に変更

```ruby
# app/controllers/tags_controller.rb
class TagsController < ApplicationController
  before_action :set_learning_theme
  before_action :set_tag, only: %i[edit update destroy]

  def index
    @tags = @learning_theme.tags.order(:name)
  end

  def new
    @tag = Tag.new
  end

  def create
    @tag = @learning_theme.tags.build(tag_params)

    if @tag.save
      redirect_to learning_theme_tags_path(@learning_theme), notice: "タグを保存しました"
    else
      flash.now[:alert] = "タグの保存に失敗しました"
      render :new, status: :unprocessable_entity
    end
  end

  def edit
  end

  def update
    if @tag.update(tag_params)
      redirect_to learning_theme_tags_path(@learning_theme), notice: "タグを更新しました"
    else
      flash.now[:alert] = "タグの更新に失敗しました"
      render :edit, status: :unprocessable_entity
    end
  end

  def destroy
    @tag.destroy
    redirect_to learning_theme_tags_path(@learning_theme), notice: "タグを削除しました"
  end

  private

  def tag_params
    params.require(:tag).permit(:name)
  end

  # @learning_theme の tags の中からのみ検索することで、他ユーザーの tag にアクセスできないようにする
  def set_tag
    @tag = @learning_theme.tags.find(params[:id])
  end

  # current_user の learning_themes の中からのみ検索することで、他ユーザーの learning_theme にアクセスできないようにする
  def set_learning_theme
    @learning_theme = current_user.learning_themes.find(params[:learning_theme_id])
  end
end
```

---

## 3️⃣ `tags/` ディレクトリ内のファイルのリンクを修正

```erb
<%# app/views/tags/_form.html.erb %>
<%= form_with model: [@learning_theme, @tag], class: "container mt-4" do |f| %>
```

```erb
<%# app/views/tags/index.html.erb %>
    <%= link_to '編集', edit_learning_theme_tag_path(@learning_theme, tag), class: "btn btn-sm btn-outline-secondary" %>
    <%= link_to '削除', learning_theme_tag_path(@learning_theme, tag), class: "btn btn-sm btn-outline-danger", data: { turbo_method: :delete, turbo_confirm: '本当に削除しますか？' } %>
```

```erb
<%# app/views/tags/new.html.erb %>
<%# app/views/tags/edit.html.erb %>
  <%= render 'form', tag: @tag, return_path: learning_theme_tags_path(@learning_theme) %>
```

---

## 4️⃣ `learning_records_controller` にも `@learning_theme` を追加

```ruby
# app/controllers/learning_records_controller.rb
class LearningRecordsController < ApplicationController
  ・・・
  before_action :set_learning_theme, only: %i[index create edit update] # ⬅️追加

  ・・・

  private
  ・・・
  def set_learning_theme # ⬅️追加
    # current_user の learning_themes の中からのみ検索することで、他ユーザーの learning_theme にアクセスできないようにする
    @learning_theme = current_user.learning_themes.first
  end
end
```

📝 `before_action :set_learning_theme` ：
`index` / `create` / `edit` / `update` のビューで `@learning_theme` を使うリンクがあるため、これらのアクション実行前に `@learning_theme` をセット *（`new` を除外しているのは、未ログインユーザーによるアクセスを許容するため）*

📝 `set_learning_theme` メソッド ：
`tags_controller` の `set_learning_theme` と異なり、URLに `learning_theme_id` が含まれないので `params[:learning_theme_id]` は使えない。代わりに `.first` でユーザーの唯一のテーマを取得。

---

## 5️⃣ `UsersController` に `@learning_theme` を追加

※ 同上

```ruby
# app/controllers/users_controller.rb
class UsersController < ApplicationController
  ・・・
  before_action :set_learning_theme, only: %i[show edit] # ⬅️追加

  ・・・

  private

  ・・・

  def set_learning_theme # ⬅️追加
    # current_user の learning_themes の中からのみ検索することで、他ユーザーの learning_theme にアクセスできないようにする
    @learning_theme = current_user.learning_themes.first
  end
end
```

---

## 6️⃣ `users/` ディレクトリ内のビューのリンクを修正

```erb
<%# app/views/users/show.html.erb %>
          <%= link_to 'タグを編集する', learning_theme_tags_path(@learning_theme), class: "btn btn-sm btn-outline-primary" %>

          <%= link_to 'タグを追加する', learning_theme_tags_path(@learning_theme), class: "btn btn-sm btn-outline-primary" %>

    <%= link_to 'タグ管理', learning_theme_tags_path(@learning_theme), class: "btn btn-sm btn-outline-primary" %>
```

```erb
<%# app/views/users/edit.html.erb %>
    <%= link_to "タグ管理", learning_theme_tags_path(@learning_theme), method: :get, class: "btn btn-sm btn-outline-primary" %>
```

---

## 7️⃣ `app/views/learning_records/` 内のファイルのリンクを修正

```erb
<%# app/views/learning_records/index.html.erb %>
  <%= link_to 'タグ管理', learning_theme_tags_path(@learning_theme), class: "btn btn-sm btn-outline-primary" %>

      <%= link_to 'タグを追加する', new_learning_theme_tag_path(@learning_theme), class: "btn btn-sm btn-outline-primary" %>
```

```erb
<%# app/views/learning_records/_form.html.erb %>
      <%= link_to "タグ管理", learning_theme_tags_path(@learning_theme), class: "btn btn-sm btn-outline-primary" %>
```

---

## 8️⃣ `LearningRecords#new` ビューの UrlGeneration エラーに対応

> エラーの原因：

未ログインユーザーに `new` アクションを許容したいのだが、
未ログイン時は `current_user` が `nil` なのでエラーとなる。

> 対応：

`LearningRecordsController` の `set_learning_theme` メソッドを修正

```ruby
# app/controllers/learning_records_controller.rb
class LearningRecordsController < ApplicationController
  ・・・
  before_action :set_learning_theme, only: %i[index new create edit update] # ⬅️ new を追加
  ・・・

  private

  def set_learning_theme
    # ⬇️ 「ぼっち演算子」を追加
    @learning_theme = current_user&.learning_themes&.first
  end
```

📝 **ぼっち演算子** `&.` ：
左辺が `nil` の場合はエラーにならず `nil` を返す。

---

### 総学習時間： 1165.3 時間
