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

## 4️⃣ `index.html.erb` のセッション名をリンク化

```erb
-            <%= membership.reading_session.name %>
+            <%= link_to membership.reading_session.name.presence || "（名称なし）", reading_session_path(membership.reading_session) %>
```

---

## 5️⃣ ヘッダーに管理者向け仮リンクを追加

```erb
<%# app/views/shared/_header.html.erb %>
         <%= link_to t("layouts.application.texts"), admin_texts_path, class: "nav-link" %>
+        <%= link_to t("layouts.application.reading_sessions"), admin_reading_sessions_path, class: "nav-link" %>
       <% end %>
```

---

### 動作確認

- [x] セッション一覧でセッション名がリンクになっていること
- [x] リンクをクリックするとテキスト一覧が表示されること
- [x] 参加していないセッションの URL に直アクセスするとセッション一覧にリダイレクトされること
- [x] 管理者ログイン時にヘッダーに「セッション管理」リンクが表示されること

---

## 6️⃣ 「セッション管理」画面のリファクタリング

現状「招待トークン」の URL が表示されているが、これは不要なので、「招待トークン発行」ボタンに変更する。

### Stimulus に `clipboard` コントローラーを実装

```ruby
# app/javascript/controllers/clipboard_controller.js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static targets = ["button"]
  static values = { content: String }

  copy() {
    navigator.clipboard.writeText(this.contentValue)
      .then(() => {
        const original = this.buttonTarget.textContent
        this.buttonTarget.textContent = "コピーしました！"
        setTimeout(() => { this.buttonTarget.textContent = original }, 2000)
      })
      .catch((err) => {
        console.error("コピー失敗:", err)
      })
  }
}
```

### `app/views/admin/reading_sessions/index.html.erb` を修正

```erb
-          <td>
-            <%= link_to t("admin.reading_sessions.index.invite_url_button"),
-                        join_reading_sessions_path(token: session.invite_token),
-                        class: "btn btn-sm btn-outline-secondary",
-                        target: "_blank", rel: "noopener noreferrer" %>
-          </td>
+          <td>
+            <div data-controller="clipboard"
+                 data-clipboard-content-value="<%= join_reading_sessions_url(token: session.invite_token) %>">
+              <button data-clipboard-target="button"
+                      data-action="clipboard#copy"
+                      class="btn btn-sm btn-outline-secondary">
+                <%= t("admin.reading_sessions.index.invite_url_button") %>
+              </button>
+            </div>
+          </td>
```

### i18n に追記

```ruby
-        invite_url_button: "招待URLを開く"
+        copy_url_button: "招待URLをコピー"
```

---

## 7️⃣ 「セッション作成」画面のリファクタリング

ステータスに応じてボタンの表示を「作成する / 更新する」と切り替える。

```erb
<%# app/views/admin/reading_sessions/_form.html.erb %>
- <%= f.submit t("admin.reading_sessions.form.submit"), class: "btn btn-primary" %>
+ <%= f.submit @reading_session.persisted? ? t("admin.reading_sessions.form.update") : t("admin.reading_sessions.form.submit"), class: "btn btn-primary" %>
```

---

### 総学習時間： 1230.6 時間
