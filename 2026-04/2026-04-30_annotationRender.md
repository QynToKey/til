# [co-READER](https://github.com/QynToKey/co_reader)（day: 23）： アノテーション表示レンダリング

## 実装計画

- 保存済みオフセットから DOM 範囲を復元し `<span>` を挿入してハイライト表示
- 同じ仕組みでアンダーライン表示
- ツールバーに色スウォッチを追加し color カラムに保存

---

### 1️⃣ `TextsController` に `@annotations` を追加

👉 *ページロード時に既存のアノテーションを取得し、ビュー経由で JS に渡す*

```ruby
# app/controllers/texts_controller.rb
  def show
    ・・・
     current_index = session_text.index(@text)
     @prev_text = session_text[current_index - 1] if current_index > 0
     @next_text = session_text[current_index + 1]
+    @annotations = current_user.annotations.where(reading_session: @reading_session, text: @text)
   end
```

---

### 2️⃣ `AnnotationsController` の JSON レスポンスを拡充

👉 *保存直後に JS 側でレンダリングするため全フィールドを返す。（`color` を許可しないと色が保存されない）*

```ruby
# app/controllers/annotations_controller.rb
     if annotation.save
-      render json: { id: annotation.id }, status: :created
+      render json: {
+        id:              annotation.id,
+        annotation_type: annotation.annotation_type,
+        start_position:  annotation.start_position,
+        end_position:    annotation.end_position,
+        color:           annotation.color
+      }, status: :created
     else
       render json: { errors: annotation.errors.full_messages }, status: :unprocessable_entity
     end

     ・・・

   def annotation_params
-    params.require(:annotation).permit(:start_position, :end_position
, :annotation_type)
+    params.require(:annotation).permit(:start_position, :end_position, :annotation_type, :color)
   end
```

---

### 3️⃣ 「テキスト詳細」ビュー を更新

```erb
<%# app/views/texts/show.html.erb%>

   <div data-controller="text-selection"
-       data-text-selection-url-value="<%= reading_session_text_annotations_path(@reading_session, @text) %>">
+       data-text-selection-url-value="<%= reading_session_text_annotations_path(@reading_session, @text) %>"
+       data-text-selection-annotations-value="<%= @annotations.map { |a|
+         { id: a.id, annotation_type: a.annotation_type,
+           start_position: a.start_position, end_position: a.end_position,
+           color: a.color }
+       }.to_json %>">

:...skipping...

   <div data-controller="text-selection"
-       data-text-selection-url-value="<%= reading_session_text_annotations_path(@reading_session, @text) %>">
+       data-text-selection-url-value="<%= reading_session_text_annotations_path(@reading_session, @text) %>"
+       data-text-selection-annotations-value="<%= @annotations.map { |a|
+         { id: a.id, annotation_type: a.annotation_type,
+           start_position: a.start_position, end_position: a.end_position,
+           color: a.color }
+       }.to_json %>">

     <div data-text-selection-target="toolbar"
          class="d-none position-absolute bg-dark text-white rounded px-2 py-1"
          style="z-index: 1000; white-space: nowrap;">
+      <span class="me-2">
+        <% [["#ffeb3b","黄"],["#a8d8a8","緑"],["#ffb3b3","ピンク"],["#b3d4ff","青"]].each do |color, label| %>
+          <button data-action="click->text-selection#selectColor"
+                  data-color="<%= color %>"
+                  class="annotation-color-swatch<%= color == '#ffeb3b' ? ' selected' : '' %>"
+                  style="background:<%= color %>;"
+                  title="<%= label %>"></button>
+        <% end %>
+      </span>
       <button data-action="click->text-selection#annotate"
               data-type="highlight"
-              class="btn btn-sm btn-warning me-1">ハイライト</button>
+              class="btn btn-sm btn-warning me-1"><%= t("texts.show.annotations.toolbar.highlight") %></button>
       <button data-action="click->text-selection#annotate"
               data-type="underline"
-              class="btn btn-sm btn-info">アンダーライン</button>
+              class="btn btn-sm btn-info"><%= t("texts.show.annotations.toolbar.underline") %></button>
     </div>
```

📝 `data-text-selection-annotations-value`
data-controller 要素に `@annotations` を JSON で渡す。

📝 既存のハイライト／アンダーラインボタンの前に色スウォッチを挿入する。

```ruby
# config/locales/views/ja.yml
       annotations:
+        toolbar:
+          highlight: "ハイライト"
+          underline: "アンダーライン"
         create:
           success: "書き込みを保存しました"
-          error: "書き込みの保存に失敗しました"
+          error: "保存に失敗しました"
+        delete: "削除"
```

---

### 4️⃣ `text_selection_controller.js` を更新

※ リファクタリング後（後述）のコードのみ記録する

```js
// app/javascript/controllers/text_input_controller.js

import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static targets = ["body", "toolbar", "editPopup"]
  static values  = { url: String, annotations: Array, deleteBaseUrl: String }

  #startPosition     = null  // 新規アノテーション：選択範囲の開始オフセット
  #endPosition       = null  // 新規アノテーション：選択範囲の終了オフセット
  #selectedColor     = "#ffeb3b"  // 新規アノテーション：選択中の色
  #activeAnnotations = {}    // 編集中の位置にある type→span のマップ例: { highlight: span, underline: span }
  #editStart         = null  // 編集中の位置の開始オフセット
  #editEnd           = null  // 編集中の位置の終了オフセット

  connect() {
    this._onMouseup  = this.#handleMouseup.bind(this)
    this._onClick    = this.#handleClick.bind(this)
    this._onDocClick = this.#handleDocumentClick.bind(this)
    this.bodyTarget.addEventListener("mouseup", this._onMouseup)
    this.bodyTarget.addEventListener("click",   this._onClick)
    document.addEventListener("click",          this._onDocClick)
  }

  disconnect() {
    this.bodyTarget.removeEventListener("mouseup", this._onMouseup)
    this.bodyTarget.removeEventListener("click",   this._onClick)
    document.removeEventListener("click",          this._onDocClick)
  }

  // ページロード時に既存アノテーションを自動描画する（Value コールバック）
  // 同じ種別・同じ位置の重複は1件だけ描画する
  annotationsValueChanged() {
    if (this.annotationsValue.length === 0) return
    const seen = new Set()
    this.annotationsValue.forEach(a => {
      const key = `${a.annotation_type}:${a.start_position}:${a.end_position}`
      if (seen.has(key)) return
      seen.add(key)
      this.#renderAnnotation(a)
    })
  }

  // 新規アノテーション作成時の色スウォッチ選択
  selectColor(event) {
    this.#selectedColor = event.currentTarget.dataset.color
    this.element.querySelectorAll(".annotation-color-swatch").forEach(el => {
      el.classList.toggle("selected", el.dataset.color === this.#selectedColor)
    })
  }

  // ハイライト／アンダーラインボタンをクリックして保存する
  annotate(event) {
    const annotationType = event.currentTarget.dataset.type
    const token = document.querySelector('meta[name="csrf-token"]').getAttribute("content")
    const start = this.#startPosition
    const end   = this.#endPosition
    const color = this.#selectedColor

    fetch(this.urlValue, {
      method: "POST",
      headers: { "Content-Type": "application/json", "X-CSRF-Token": token },
      body: JSON.stringify({
        annotation: { start_position: start, end_position: end, annotation_type: annotationType, color }
      })
    })
    .then(res => res.json())
    .then(data => {
      if (data.id) {
        this.#renderAnnotation({ ...data, start_position: start, end_position: end, color })
        this.#hideToolbar()
        window.getSelection()?.removeAllRanges()
      }
    })
  }

  // 既存アノテーションをクリックしたときに編集ポップアップを表示する
  #handleClick(event) {
    if (this.toolbarTarget.contains(event.target)) return
    if (this.editPopupTarget.contains(event.target)) return

    // クリック位置から annotation span を内側→外側へ遡って全件収集する
    // 同一範囲に HL と UL が重なっている場合、入れ子になっているので両方取得できる
    this.#activeAnnotations = {}
    this.#editStart = null
    this.#editEnd   = null

    let target = event.target.closest("[data-annotation-id]")
    while (target) {
      const type = [...target.classList]
        .find(c => c.startsWith("annotation-"))
        ?.replace("annotation-", "")
      if (type && !this.#activeAnnotations[type]) {
        this.#activeAnnotations[type] = target
        // 最初に見つけた span のオフセットを編集用位置として保存する
        if (this.#editStart === null) {
          this.#editStart = parseInt(target.dataset.startPosition)
          this.#editEnd   = parseInt(target.dataset.endPosition)
        }
      }
      // 親要素に annotation span があればそちらも収集する
      target = target.parentElement?.closest("[data-annotation-id]")
    }

    if (Object.keys(this.#activeAnnotations).length === 0) {
      this.#hideEditPopup()
      return
    }

    // ポップアップを span の直下に配置する
    const anySpan = Object.values(this.#activeAnnotations)[0]
    const rect    = anySpan.getBoundingClientRect()
    const popup   = this.editPopupTarget
    popup.style.position  = "absolute"
    popup.style.top       = `${rect.bottom + window.scrollY + 8}px`
    popup.style.left      = `${rect.left + window.scrollX + rect.width / 2}px`
    popup.style.transform = "translateX(-50%)"

    // 現在存在する型のボタンに .active を付けて視覚的に示す
    this.editPopupTarget.querySelectorAll("[data-type]").forEach(btn => {
      btn.classList.toggle("active", btn.dataset.type in this.#activeAnnotations)
    })
    popup.classList.remove("d-none")
  }

  // コントローラー要素の外をクリックしたらポップアップを閉じる
  #handleDocumentClick(event) {
    if (this.element.contains(event.target)) return
    this.#hideEditPopup()
  }

  // 編集ポップアップの色スウォッチをクリックして色を変更する
  // 同一位置に複数の型がある場合はすべてに同じ色を適用する
  changeColor(event) {
    if (Object.keys(this.#activeAnnotations).length === 0) return

    const color = event.currentTarget.dataset.color
    const token = document.querySelector('meta[name="csrf-token"]').getAttribute("content")

    // 各 span に対して PATCH リクエストを並列で送る
    const promises = Object.entries(this.#activeAnnotations).map(([type, span]) => {
      const id = span.dataset.annotationId
      return fetch(`${this.deleteBaseUrlValue}${id}`, {
        method: "PATCH",
        headers: { "Content-Type": "application/json", "X-CSRF-Token": token },
        body: JSON.stringify({ annotation: { color } })
      })
      .then(res => {
        if (res.ok) {
          // 型に応じたスタイルプロパティを更新する
          if (type === "highlight") {
            span.style.backgroundColor = color
          } else {
            span.style.borderBottomColor = color
          }
        }
      })
    })

    // 全リクエストが完了したらポップアップを閉じる
    Promise.all(promises).then(() => this.#hideEditPopup())
  }

  // 編集ポップアップの種別ボタンをクリックする
  // 既に存在する型 → 削除（ボタンをオフ）
  // 存在しない型  → 新規追加（ボタンをオン）
  changeType(event) {
    const type  = event.currentTarget.dataset.type
    const token = document.querySelector('meta[name="csrf-token"]').getAttribute("content")

    if (this.#activeAnnotations[type]) {
      // ── 削除パス ──────────────────────────────────────────────
      const span = this.#activeAnnotations[type]
      const id   = span.dataset.annotationId

      fetch(`${this.deleteBaseUrlValue}${id}`, {
        method: "DELETE",
        headers: { "X-CSRF-Token": token }
      })
      .then(res => {
        if (res.ok) {
          // span の中身を親に移してから span 要素を取り除く
          const parent = span.parentNode
          while (span.firstChild) parent.insertBefore(span.firstChild, span)
          parent.removeChild(span)
          parent.normalize()

          delete this.#activeAnnotations[type]

          // 別の型がまだ残っていればボタン状態を更新、なければポップアップを閉じる
          if (Object.keys(this.#activeAnnotations).length > 0) {
            this.editPopupTarget.querySelectorAll("[data-type]").forEach(btn => {
              btn.classList.toggle("active", btn.dataset.type in this.#activeAnnotations)
            })
          } else {
            this.#hideEditPopup()
          }
        }
      })
    } else {
      // ── 追加パス ──────────────────────────────────────────────
      // 型ごとのデフォルト色で新しいアノテーションを作成する
      const defaultColors = { highlight: "#ffeb3b", underline: "#2196f3" }
      const color = defaultColors[type] ?? "#ffeb3b"

      fetch(this.urlValue, {
        method: "POST",
        headers: { "Content-Type": "application/json", "X-CSRF-Token": token },
        body: JSON.stringify({
          annotation: {
            start_position:  this.#editStart,
            end_position:    this.#editEnd,
            annotation_type: type,
            color
          }
        })
      })
      .then(res => res.json())
      .then(data => {
        if (data.id) {
          this.#renderAnnotation({
            ...data,
            start_position: this.#editStart,
            end_position:   this.#editEnd,
            color
          })
          // DOM が更新されたのでポップアップを閉じる
          // 再クリックすると両方の型がアクティブな状態で開く
          this.#hideEditPopup()
        }
      })
    }
  }

  // 削除ボタンをクリックして、同一位置の全アノテーションを削除する
  deleteAnnotation() {
    if (Object.keys(this.#activeAnnotations).length === 0) return

    const token = document.querySelector('meta[name="csrf-token"]').getAttribute("content")

    // 全 span に DELETE リクエストを並列で送る
    const promises = Object.entries(this.#activeAnnotations).map(([_type, span]) => {
      const id = span.dataset.annotationId
      return fetch(`${this.deleteBaseUrlValue}${id}`, {
        method: "DELETE",
        headers: { "X-CSRF-Token": token }
      })
      .then(res => {
        if (res.ok) {
          const parent = span.parentNode
          while (span.firstChild) parent.insertBefore(span.firstChild, span)
          parent.removeChild(span)
        }
      })
    })

    // 全削除完了後にテキストノードを結合してからポップアップを閉じる
    // normalize() で隣接テキストノードを1つに統合し、次回のオフセット計算を正確にする
    Promise.all(promises).then(() => {
      this.bodyTarget.normalize()
      this.#hideEditPopup()
    })
  }

  // テキスト選択時にツールバーを表示し、編集ポップアップを閉じる
  #handleMouseup() {
    const sel = window.getSelection()
    if (!sel || sel.isCollapsed) {
      this.#hideToolbar()
      return
    }

    const range = sel.getRangeAt(0)
    const container = this.bodyTarget

    if (!container.contains(range.commonAncestorContainer)) {
      this.#hideToolbar()
      return
    }

    this.#startPosition = this.#getTextOffset(container, range.startContainer, range.startOffset)
    this.#endPosition   = this.#getTextOffset(container, range.endContainer,   range.endOffset)

    const rect = range.getBoundingClientRect()
    const toolbar = this.toolbarTarget
    toolbar.style.position  = "absolute"
    toolbar.style.top       = `${rect.top + window.scrollY - toolbar.offsetHeight - 8}px`
    toolbar.style.left      = `${rect.left + window.scrollX + rect.width / 2}px`
    toolbar.style.transform = "translateX(-50%)"
    // 編集ポップアップが開いていれば閉じる
    this.#hideEditPopup()
    toolbar.classList.remove("d-none")
  }

  #hideToolbar() {
    this.toolbarTarget.classList.add("d-none")
    this.#startPosition = null
    this.#endPosition   = null
  }

  #hideEditPopup() {
    this.editPopupTarget.classList.add("d-none")
    // 編集状態をすべてリセットする
    this.#activeAnnotations = {}
    this.#editStart = null
    this.#editEnd   = null
  }

  #renderAnnotation(annotation) {
    const range = this.#offsetToRange(
      this.bodyTarget, annotation.start_position, annotation.end_position
    )
    if (!range) return

    const span = document.createElement("span")
    span.classList.add(`annotation-${annotation.annotation_type}`)
    span.dataset.annotationId  = annotation.id
    // クリック時に追加・削除の判断に使うオフセットを data 属性として持たせる
    span.dataset.startPosition = annotation.start_position
    span.dataset.endPosition   = annotation.end_position

    if (annotation.color) {
      if (annotation.annotation_type === "highlight") {
        span.style.backgroundColor = annotation.color
      } else {
        span.style.borderBottomColor = annotation.color
      }
    }

    try {
      // 選択範囲が単一テキストノード内に収まる場合はこちら
      range.surroundContents(span)
    } catch (e) {
      // 選択範囲が複数ノードをまたぐ場合（例：既存 span の境界）は
      // extractContents で中身を取り出して span に入れてから挿入する
      const fragment = range.extractContents()
      span.appendChild(fragment)
      range.insertNode(span)
    }
  }

  // TreeWalker でテキストノードを順に走査し、targetNode に到達するまでの
  // 文字数を累積することで、DOM 上の位置を「文字オフセット」に変換する
  #getTextOffset(container, targetNode, targetOffset) {
    let offset = 0
    const walker = document.createTreeWalker(container, NodeFilter.SHOW_TEXT)
    while (walker.nextNode()) {
      if (walker.currentNode === targetNode) return offset + targetOffset
      offset += walker.currentNode.length
    }
    return offset
  }

  // #getTextOffset の逆：文字オフセット（start/end）を DOM の Range に変換する
  // TreeWalker でテキストノードを走査し、累積長が start/end を超えた時点で
  // setStart / setEnd を呼ぶ
  #offsetToRange(container, start, end) {
    const range = document.createRange()
    let offset = 0
    let startSet = false

    const walker = document.createTreeWalker(container, NodeFilter.SHOW_TEXT)
    while (walker.nextNode()) {
      const node = walker.currentNode
      const len  = node.length

      if (!startSet && offset + len >= start) {
        range.setStart(node, start - offset)
        startSet = true
      }
      if (startSet && offset + len >= end) {
        range.setEnd(node, end - offset)
        return range
      }
      offset += len
    }
    return null
  }
}
```

---

### 5️⃣ `application.css` にスタイルを追加

```js
/* アノテーション */
.annotation-highlight {
  background-color: #ffeb3b;  /* color 未指定時のデフォルト */
  border-radius: 2px;
  padding: 0 1px;
}

.annotation-underline {
  border-bottom: 2px solid #2196f3;
  padding-bottom: 1px;
}

/* 色スウォッチ */
.annotation-color-swatch {
  display: inline-block;
  width: 18px;
  height: 18px;
  border-radius: 50%;
  border: 2px solid transparent;
  cursor: pointer;
  vertical-align: middle;
  padding: 0;
}

.annotation-color-swatch.selected {
  border-color: #fff;
  outline: 2px solid #555;
}
```

---

#### 動作確認

- [x] テキストページをリロード → 既存のアノテーション（ハイライト）が黄色で表示される
- [x] テキストを選択 → ツールバーに色スウォッチが表示される
- [x] 色スウォッチをクリック → 選択色が切り替わる（枠が付く）
- [x] 「ハイライト」クリック → 選択した色で即座に背景色が付く
- [x] 「アンダーライン」クリック → 選択した色で下線が付く
`DB: SELECT id, annotation_type, color FROM annotations ORDER BY id DESC LIMIT 3;` で `color` が保存されていること

```bash
$ docker compose exec db psql -U postgres -d co_reader_development -c "SELECT id, annotation_type, color FROM annotations ORDER BY id DESC LIMIT 5;"
 id | annotation_type |  color
----+-----------------+---------
 19 |               1 | #a8d8a8
 18 |               0 | #ffeb3b
 17 |               1 | #ffb3b3
 16 |               0 | #ffb3b3
 15 |               1 | #b3d4ff
(5 rows)
```

---

## リファクタリング

- 以上の実装で、既存アノテーション上に重ねて書き込もうとすると `DOM` が壊れる問題が判明した。
- アノテーションをクリックすると色変更・削除ができるポップアップを表示することで解決する。
- 同一範囲に HL と UL を両立させ、ポップアップで個別にトグルする。

| 状態 | HL ボタン | UL ボタン |
| --- | --- | --- |
| HL のみ | active（クリックで削除） | inactive（クリックで追加） |
| UL のみ | inactive（クリックで追加） | active（クリックで削除） |
| HL + UL | active（クリックで削除） | active（クリックで削除） |

---

### 6️⃣ ルーティングに `annotations` を追加

```ruby
# config/routes.rb
  resources :annotations, only: %i[ update destroy ]
```

```bash
$ docker compose exec web bin/rails routes | grep "annotation"
                        annotation PATCH  /annotations/:id(.:format)                                                                        annotations#update
                                    PUT    /annotations/:id(.:format)                                                                        annotations#update
                                    DELETE /annotations/:id(.:format)                                                                        annotations#destroy
  reading_session_text_annotations POST   /reading_sessions/:reading_session_id/texts/:text_id/annotations(.:format)                        annotations#create
```

---

### 7️⃣ `AnnotationsController` に `update` / `destroy` を追加

```ruby
# app/controllers/annotations_controller.rb
  def update
    annotation = current_user.annotations.find(params[:id])
    if annotation.update(annotation_params)
      render json: { color: annotation.color }, status: :ok
    else
      render json: { errors: annotation.errors.full_messages }, status: :unprocessable_entity
    end
  rescue ActiveRecord::RecordNotFound
    render json: { error: "Not found" }, status: :not_found
  end

  def destroy
    annotation = current_user.annotations.find(params[:id])
    annotation.destroy
    render json: {}, status: :ok
  rescue ActiveRecord::RecordNotFound
    render json: { error: "Not found" }, status: :not_found
  end
```

📝 `current_user.annotations.find`
自分以外のアノテーションを操作しようとすると `RecordNotFound` になりセキュア

📝 `rescue`
404を返すのは Rails の標準的なパターン

---

### 8️⃣ 「テキスト詳細」ビューを修正

👉 *`data-controller` 要素に `value` を追加*
👉 *編集ポップアップを追加*

```erb
<%# app/views/texts/show.html.erb %>
   <div data-controller="text-selection"
        data-text-selection-url-value="<%= reading_session_text_annotations_path(@reading_session, @text) %>"
+       data-text-selection-delete-base-url-value="/annotations/"
        data-text-selection-annotations-value="<%= @annotations.map { |a|
          { id: a.id, annotation_type: a.annotation_type,
            start_position: a.start_position, end_position: a.end_position,
           color: a.color }
       }.to_json %>">
     </div>

+      <div data-text-selection-target="editPopup"
+         class="d-none position-absolute bg-dark text-white rounded px-2 py-1"
+         style="z-index: 1001; white-space: nowrap;">
+      <span class="me-2">
+        <% [["#ffeb3b","黄"],["#a8d8a8","緑"],["#ffb3b3","ピンク"],["#b3d4ff","青"]].each do |color, label| %>
+          <button data-action="click->text-selection#changeCo
lor"
+                  data-color="<%= color %>"
+                  class="annotation-color-swatch"
+                  style="background:<%= color %>;"
+                  title="<%= label %>"></button>
+        <% end %>
+      </span>
+      <button data-action="click->text-selection#deleteAnnota
tion"
+              class="btn btn-sm btn-danger">
+        <%= t("texts.show.annotations.delete") %>
+      </button>
+    </div>
```

---

#### リファクタリングの動作確認

- [ ] ハイライト済み箇所をクリック → 編集ポップアップが表示される
- [ ] 別の色スウォッチをクリック → 色が変わる（DB も更新される）
- [ ] 「削除」をクリック → ハイライトが消える（DB からも削除される）
- [ ] 削除後、同じ箇所に再度ハイライト → 正常に表示される

---

##### 総学習時間： 1276.9 時間
