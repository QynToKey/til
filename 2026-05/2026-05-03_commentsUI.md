# [co-READER](https://github.com/QynToKey/co_reader)（day: 25_3）： コメント一覧 / UI 整備

## 現状と実装方針

> 現状

- コメントはDBに保存できるが（`CommentsController#create` 実装済み）、テキスト詳細ページ（`texts/show`）にコメント一覧が表示されない。
- アノテーションとコメントの視覚的紐付けもない。

> 実装方針

- PC・スマホ共通でテキスト本文の直下にコメント UI を配置
- デフォルトでは非表示。「コメント（n件）▼」ボタンで展開/折りたたむ
- コメントが0件のときはボタン自体を非表示
- コメント一覧の各項目をクリックすると対応アノテーションにスクロール ＋ 一時強調

---

## 1️⃣ i18n に関連語彙を追加

```ruby
# config/locales/views/ja.yml
        comment:
          toggle: "コメント"
          placeholder: "メモを書く..."
          submit: "保存"
          cancel: "キャンセル"
+         list_toggle: "コメント（%{count}件）"
+         annotation_type:
+           highlight: "HL"
+           underline: "UL"
```

---

## 2️⃣ `TextsController` を修正

👉 *`@annotations` に `includes(:comments)` を追加してN+1を防ぐ*

```ruby
# app/controllers/texts_controller.rb
def show
  ・・・
-    @annotations = current_user.annotations.where(reading_session: @reading_session, text: @text)
+    @annotations = current_user.annotations.includes(:comments).where(reading_session: @reading_session, text: @text)
end
```

---

## 3️⃣ 「テキスト詳細」ビューに「コメント一覧」を追加

```erb
<%# app/views/texts/show.html.erb %>

+  <%# コメント一覧 %>
+  <% annotations_with_comments = @annotations.select { |a| a.comments.any? } %>
+  <% unless annotations_with_comments.empty? %>
+    <div class="mt-4">
+      <button class="btn btn-outline-secondary btn-sm"
+              data-action="click->text-selection#toggleCommentList"
+              data-text-selection-target="commentListToggle">
+        <%= t("texts.show.annotations.comment.list_toggle", count: annotations_with_comments.sum { |a| a.comments.count }) %> ▼
+      </button>
+
+      <div data-text-selection-target="commentList" class="d-none mt-3">
+        <% annotations_with_comments.each do |annotation| %>
+          <% annotation.comments.each do |comment| %>
+            <div class="border rounded p-2 mb-2"
+                data-annotation-id="<%= annotation.id %>"
+                data-action="click->text-selection#scrollToAnnotation"
+                style="cursor: pointer;">
+              <span class="badge me-1"
+                    style="background-color: <%= annotation.color %>; color: #333; font-size: 0.7rem;">
+                <%= t("texts.show.annotations.comment.annotation_type.#{annotation.annotation_type}") %>
+              </span>
+              <span class="text-muted small me-2">
+                "...<%= @text.body[annotation.start_position, 20] %>..."
+              </span>
+              <p class="mb-0 mt-1"><%= comment.body %></p>
+            </div>
+          <% end %>
+        <% end %>
+      </div>
+    </div>
+  <% end %>
```

---

## 4️⃣ Stimulus を更新

### `static targets` の行を変更

```js
// 変更前
static targets = ["body", "toolbar", "editPopup", "commentForm", "commentBody"]

// 変更後
static targets = ["body", "toolbar", "editPopup", "commentForm", "commentBody", "commentList", "commentListToggle"]
```

### `disconnect()` の後（約46行目）に2メソッドを追加**

```js
  toggleCommentList() {
    const list = this.commentListTarget
    const btn  = this.commentListToggleTarget
    const isHidden = list.classList.toggle("d-none")
    btn.textContent = btn.textContent.replace(isHidden ? "▲" : "▼", isHidden ? "▼" : "▲")
  }

  scrollToAnnotation(event) {
    const annotationId = event.currentTarget.dataset.annotationId
    const span = this.bodyTarget.querySelector(`[data-annotation-id="${annotationId}"]`)
    if (!span) return
    span.scrollIntoView({ behavior: "smooth", block: "center" })
    span.classList.add("annotation-focus")
    setTimeout(() => span.classList.remove("annotation-focus"), 1500)
  }
```

---

## 5️⃣ CSS を更新

👉 *コメント一覧からアノテーション span にジャンプしたときの一時強調リング*

```css
/* app/assets/stylesheets/application.css */
.annotation-focus {
  box-shadow: 0 0 0 3px #ffc107;
  border-radius: 2px;
  transition: box-shadow 0.3s ease;
}
```

---

### 動作確認

- [x] テキスト本文の直下に「コメント（n件）▼」ボタンが表示される
- [x] ボタンクリックで一覧を展開/折りたたみ
- [x] 一覧の項目をクリックすると対応するアノテーション span にスクロール＋一時強調

---

#### 総学習時間： 1295.4 時間
