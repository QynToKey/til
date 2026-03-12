# [卒制](https://github.com/QynToKey/HowLongWillItLast) (day 14)：TagsController

---

## 1️⃣ ルーティングに `resources` を追加

```ruby
# config/routes.rb
resources :tags, only: %i[index new create edit update destroy]
```

👉 *`show` アクションのみ不要 (タグ単体の詳細ページをユーザーが必要とする場面がないため)*

⬇️ 確認

```bash
$ docker compose exec web bin/rails routes | grep tags
                                    tags GET    /tags(.:format)                                                                                   tags#index
                                         POST   /tags(.:format)                                                                                   tags#create
                                 new_tag GET    /tags/new(.:format)                                                                               tags#new
                                edit_tag GET    /tags/:id/edit(.:format)                                                                          tags#edit
                                     tag PATCH  /tags/:id(.:format)                                                                               tags#update
                                         PUT    /tags/:id(.:format)                                                                               tags#update
                                         DELETE /tags/:id(.:format)                                                                               tags#destroy
```

---

## 2️⃣ TagsController を作成

```bash
$ docker compose exec web rails g controller Tags
      create  app/controllers/tags_controller.rb
      invoke  erb
      create    app/views/tags
      invoke  helper
      create    app/helpers/tags_helper.rb
```

⬇️

```ruby
# app/controllers/tags_controller.rb
class TagsController < ApplicationController
  before_action :set_tag, only: %i[edit update destroy]

  def index
    @tags = current_user.tags.order(:name)
  end

  def new
    @tag = Tag.new
  end

  def create
    @tag = current_user.tags.build(tag_params)

    if @tag.save
      redirect_to tags_path, notice: "タグを保存しました"
    else
      flash.now[:alert] = "タグの保存に失敗しました"
      render :new, status: :unprocessable_entity
    end
  end

  def edit
  end

  def update
    if @tag.update(tag_params)
      redirect_to tags_path, notice: "タグを更新しました"
    else
      flash.now[:alert] = "タグの更新に失敗しました"
      render :edit, status: :unprocessable_entity
    end
  end

  def destroy
    @tag.destroy
    redirect_to tags_path, notice: "タグを削除しました"
  end

  private

  def tag_params
    params.require(:tag).permit(:name)
  end

  def set_tag
    @tag = current_user.tags.find(params[:id])
  end
end
```

👉 *未ログインユーザーのタグへのアクセスは無用なので、`before_action` のコールバックは不要*

---

## 実装方針の確認

- 以下の各機能をパーシャルとして各ページに埋め込む
- 「マイページ」などからの動線も確保するためにヘッダーに「タグ管理」を加える

### 実装案

- `/tags`（独立ページ）
  👉 *タグの一覧・作成・編集・削除*

- `learning_records/new`
  👉 *タグを付けられる・タグ作成もできる*

- `learning_records/edit`
  👉 *タグの付け外し・タグ作成もできる*

- `learning_records/index`
  👉 *タグで絞り込める（フィルタ）*
  👉 *タグの付け外しは `edit` に集約する（UX的にも実装的にも複雑になるため）*

- ヘッダー
  👉 *「タグ管理」リンクで / `tags` へ動線を確保*
