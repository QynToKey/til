# [co-READER](https://github.com/QynToKey/co_reader)（day: 14_2）： セッションテキスト一覧

## 0️⃣ 現状と実装内容

>現状

`#81` で `reading_session_texts` 中間テーブルを追加済みで `session.texts` が使える状態。

> 実装内容

- セッションに紐づいた `Text` の一覧を表示する `show` ページを実装する。
- あわせて、管理者向けの仮リンク（開発用）をヘッダーに追加する。

---

## 1️⃣ ルーティングに `reading_sessions#show` を追加

```ruby
-  resources :reading_sessions, only: %i[ index ] do
+  resources :reading_sessions, only: %i[ index show ] do
     get :join, on: :collection
   end
```

---

## 2️⃣ `ReadingSessionsController` に `show` アクションを追加

```ruby
# app/controllers/reading_sessions_controller.rb
   def index
     @memberships = current_user.memberships.includes(reading_session: :texts).order(created_at: :desc)
   end

+  def show
+    @reading_session = ReadingSession.find(params[:id])
+    unless @reading_session.users.include?(current_user)
+      redirect_to reading_sessions_path, alert: "参加していないセッションです"
+      return
+    end
+    @texts = @reading_session.texts
+  end
+
   def join
```

---

## 3️⃣ 「セッション詳細」画面を作成

```bash
touch app/views/reading_sessions/show.html.erb
```

```erb
<div class="container mt-5" style="max-width: 800px;">
  <h1 class="h4 mb-4"><%= @reading_session.name.presence || "（名称なし）" %></h1>

  <% if @texts.empty? %>
    <p class="text-muted">テキストがありません。</p>
  <% else %>
    <ul class="list-group">
      <% @texts.each do |text| %>
        <li class="list-group-item">
          <%= link_to text.title, text_path(text) %>
        </li>
      <% end %>
    </ul>
  <% end %>

  <%= link_to "セッション一覧に戻る", reading_sessions_path, class: "btn btn-outline-secondary mt-3" %>
</div>
```

---

## 4️⃣
