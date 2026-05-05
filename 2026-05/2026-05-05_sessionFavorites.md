# [co-READER](https://github.com/QynToKey/co_reader)（day: 27_2）： SessionFavorites 機能

## 0️⃣ 実装計画

- `session_favorites` テーブル・モデルを新設
- 共読モード時にサイドパネルでお気に入りトグルを操作できる UI を実装

---

## 1️⃣ `session_favorites` テーブルを作成

```bash
$ docker compose exec web bin/rails g migration CreateSessionFavorites
      invoke  active_record
      create    db/migrate/20260505080416_create_session_favorites.rb
```

  ⬇️

```ruby
# db/migrate/20260505080416_create_session_favorites.rb
class CreateSessionFavorites < ActiveRecord::Migration[8.0]
  def change
    create_table :session_favorites do |t|
      t.references :membership,    null: false, foreign_key: true
      t.references :favorite_user, null: false, foreign_key: { to_table: :users }
      t.datetime   :created_at,    null: false
    end

    add_index :session_favorites, [:membership_id, :favorite_user_id], unique: true
  end
end
```

  ⬇️

```bash
$ docker compose exec web bin/rails db:migrate
== 20260505080416 CreateSessionFavorites: migrating ===========================
-- create_table(:session_favorites)
   -> 0.0586s
-- add_index(:session_favorites, [:membership_id, :favorite_user_id], {:unique=>true})
   -> 0.0065s
== 20260505080416 CreateSessionFavorites: migrated (0.0652s) ==================
```

---

## 2️⃣ `SessionFavorite` モデル

### モデル作成

```bash
touch app/models/session_favorite.rb
```

```ruby
# app/models/session_favorite.rb
class SessionFavorite < ApplicationRecord
  belongs_to :membership
  belongs_to :favorite_user, class_name: 'User'

  validates :favorite_user_id, uniqueness: { scope: :membership_id }
end
```

### アソシエーション追加

```ruby
# app/models/membership.rb
   belongs_to :user
   belongs_to :reading_session
   has_many :text_readings, dependent: :destroy
+  has_many :session_favorites, dependent: :destroy
```

```ruby
# app/models/user.rb
  has_many :memberships, dependent: :destroy
 has_many :annotations, dependent: :destroy
+ has_many :favorited_by, class_name: 'SessionFavorite',
+                       foreign_key: :favorite_user_id,
+                       dependent: :destroy
 has_many :reading_sessions, through: :memberships
```

### ルーティング

```ruby
# config/routes.rb
  resources :reading_sessions, only: %i[ index show ] do
    member do
     patch :update_mode
    end
    get :join, on: :collection
+   resources :session_favorites, only: %i[create destroy]
    resources :texts, only: %i[ show ] do
      resource :text_reading, only: %i[ create ]
      resources :annotations, only: %i[ create ]
```

  ⬇️ 確認

```bash
$ docker compose exec web bin/rails routes | grep session_favorites
    reading_session_session_favorites POST   /reading_sessions/:reading_session_id/session_favorites(.:format)                                 session_favorites#create
    reading_session_session_favorite DELETE /reading_sessions/:reading_session_id/session_favorites/:id(.:format)                             session_favorites#destroy
```

---

## 3️⃣ `SessionFavoritesController` を作成

```bash
touch app/controllers/session_favorites_controller.rb
```

```ruby
# app/controllers/session_favorites_controller.rb
class SessionFavoritesController < ApplicationController
  before_action :set_reading_session
  before_action :set_membership

  def create
    @favorite_user = User.find(params[:favorite_user_id])
    @favorite = @membership.session_favorites.build(favorite_user: @favorite_user)

    if @favorite.save
      # 成功した場合は、Turbo Stream で更新を返す
      respond_to do |format|
        format.turbo_stream
        format.html { redirect_back fallback_location: reading_session_path(@reading_session) }
      end
    end
  end

  def destroy
    @favorite = @membership.session_favorites.find(params[:id])
    @favorite_user = @favorite.favorite_user
    @favorite.destroy

    respond_to do |format|
      format.turbo_stream
      format.html { redirect_back fallback_location: reading_session_path(@reading_session) }
    end
  end

  private

  def set_reading_session
    @reading_session = ReadingSession.find(params[:reading_session_id])
  end

  def set_membership
    @membership = @reading_session.memberships.find_by!(user: current_user)
  end
end
```

---

## 4️⃣ `TextsController#show` にメンバー情報を追加

```ruby
# app/controllers/texts_controller.rb
  def show

  # 共読モードの場合、他の参加者と自分のお気に入りユーザーを取得する
+   if @membership.co_reading?
+     @other_members = @reading_session.memberships
+                                      .includes(:user, :session_favorites)
+                                      .where.not(user_id: current_user.id)
+     @my_favorite_user_ids = @membership.session_favorites.pluck(:favorite_user_id).to_set
+   end
  end
```

---

## 5️⃣ 「お気に入り操作」UI を実装

### 「テキスト詳細」を修正

👉 *共読モード時に２カラム構成に*

```erb
<%# app/views/texts/show.html.erb %>

-<div class="container mt-5" style="max-width: 1000px;">
+<div class="container mt-5" style="max-width: <%= @membership.co_reading? ? '1220px' : '1000px' %>;">
+  <div class="d-flex gap-3 align-items-start">
+  <div style="flex: 1; min-width: 0;">
   <h1 class="h4 mb-2"><%= @text.title %></h1>

   <p class="text-muted small mb-4">
@@ -263,4 +265,16 @@
       <% end %>
     </div>
   </div>
-</div>
+
+    </div><%# flex: 1 を閉じる %>
+    <% if @membership.co_reading? %>
+      <div style="width: 220px; flex-shrink: 0;">
+        <%= render 'texts/side_panel',
+              reading_session: @reading_session,
+              other_members: @other_members,
+              my_favorite_user_ids: @my_favorite_user_ids,
+              membership: @membership %>
+      </div>
+    <% end %>
+  </div><%# d-flex を閉じる %>
+</div><%# container を閉じる %>
```

### 「サイドパネル」パーシャルを作成

```bash
touch app/views/texts/_side_panel.html.erb
```

```erb
<%# app/views/texts/_side_panel.html.erb %>
<div class="border rounded p-3" style="position: sticky; top: 1rem;">
  <h6 class="mb-3">参加者</h6>
  <% other_members.each do |m| %>
    <%= render 'texts/side_panel_user',
          reading_session: reading_session,
          user: m.user,
          favorited: my_favorite_user_ids.include?(m.user.id),
          favorite: membership.session_favorites.find { |f| f.favorite_user_id == m.user.id } %>
  <% end %>
</div>
```

### 「ユーザー行」パーシャルを作成

```bash
touch app/views/texts/_side_panel_user.html.erb
```

```erb
<&# app/views/texts/_side_panel_user.html.erb $>
<div id="session_favorite_user_<%= user.id %>"
     class="d-flex align-items-center justify-content-between mb-2">
  <span class="small"><%= user.name %></span>
  <% if favorited %>
    <%= button_to "★",
          reading_session_session_favorite_path(reading_session, favorite),
          method: :delete,
          class: "btn btn-sm btn-warning p-0 px-1",
          title: "お気に入り解除" %>
  <% else %>
    <%= button_to "☆",
          reading_session_session_favorites_path(reading_session),
          params: { favorite_user_id: user.id },
          class: "btn btn-sm btn-outline-warning p-0 px-1",
          title: "お気に入り追加" %>
  <% end %>
</div>
```

---

## 6️⃣ Turbo Stream ビューを作成

### `create`

```bash
mkdir -p app/views/session_favorites/ && touch app/views/session_favorites/create.turbo_stream.erb
```

```erb
<%# app/views/session_favorites/create.turbo_stream.erb %>
<%= turbo_stream.replace "session_favorite_user_#{@favorite_user.id}" do %>
  <%= render 'texts/side_panel_user',
        reading_session: @reading_session,
        user: @favorite_user,
        favorited: true,
        favorite: @favorite %>
<% end %>
```

### `destroy`

```bash
touch app/views/session_favorites/destroy.turbo_stream.erb
```

```erb
<%# app/views/session_favorites/destroy.turbo_stream.erb %>
<%= turbo_stream.replace "session_favorite_user_#{@favorite_user.id}" do %>
  <%= render 'texts/side_panel_user',
        reading_session: @reading_session,
        user: @favorite_user,
        favorited: false,
        favorite: nil %>
<% end %>
```

---

### 📝 Turbo Stream

- Turbo Stream は Hotwire の一部で、**サーバーからHTMLの断片を返し、ページの特定部分だけを更新する仕組み**。
- 通常のリクエストはページ全体を返すが、Turbo Stream は「このIDの要素をこのHTMLで置き換えろ」という命令をサーバーから送る。

#### 仕組み

1. リクエスト時、ブラウザは `Accept: text/vnd.turbo-stream.html` というヘッダーを付けて送る。

2. コントローラーの `respond_to` で `format.turbo_stream` がこれを受け取り、`.turbo_stream.erb` ファイルを返す：

```ruby
respond_to do |format|
  format.turbo_stream   # → create.turbo_stream.erb を返す
  format.html { redirect_back ... }
end
```

#### **turbo_stream.replace**

```erb
<%= turbo_stream.replace "session_favorite_user_3" do %>
  ...新しいHTML...
<% end %>
```

👉 *ブラウザに対して「`id="session_favorite_user_3"` の要素をこの中身で置き換えろ」という命令*

  ⬇️

```html
<turbo-stream action="replace" target="session_favorite_user_3">
  <template>
    ...新しいHTML...
  </template>
</turbo-stream>
```

👉 *ブラウザ側の Turbo ライブラリがこのタグを解釈して、DOM を更新*

#### replace 以外のアクション

| アクション | 意味 |
| --- | --- |
| `replace` | 要素ごと置き換え |
| `update` | 要素の中身だけ置き換え |
| `append` | 要素の末尾に追加 |
| `prepend` | 要素の先頭に追加 |
| `remove` | 要素を削除 |

👉 *今回は★↔☆ボタンの切り替えなので `replace` が最適*

---

### 動作確認

- [x] 共読モードに切り替えてサイドパネルに参加者が表示されることを確認
- [x] ★☆ボタンでページリロードなしにボタン状態が切り替わることを確認
- [x] docker compose exec web bin/rails console で SessionFavorite.all を確認

```bash
app(dev):001> SessionFavorite.all
=>
[#<SessionFavorite:0x0000ffff57a496b8
  id: 2,
  membership_id: 4,
  favorite_user_id: 8,
  created_at: "2026-05-05 18:21:14.187328000 +0900">]
```

---

#### 総学習時間： 1308.8 時間
