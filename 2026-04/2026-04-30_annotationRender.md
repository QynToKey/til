# [co-READER](https://github.com/QynToKey/co_reader)（day: 23）： アノテーション表示レンダリング

## 0️⃣ 実装計画

- 保存済みオフセットから DOM 範囲を復元し `<span>` を挿入してハイライト表示
- 同じ仕組みでアンダーライン表示
- ツールバーに色スウォッチを追加し color カラムに保存

---

## 1️⃣ `TextsController` に `@annotations` を追加

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

## 2️⃣ `AnnotationsController` の JSON レスポンスを拡充

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

## 3️⃣ 「テキスト詳細」ビュー を更新

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
```

---

## 4️⃣ `text_selection_controller.js` を更新

```js
// app/javascript/controllers/text_input_controller.js

import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  // このコントローラーが操作する HTML 要素を宣言する
  // data-text-selection-target="body" / "toolbar" に対応
  static targets = ["body", "toolbar"]

  // HTML の data 属性から値を受け取る宣言
  // data-text-selection-url-value / data-text-selection-annotations-value に対応
  static values  = { url: String, annotations: Array }

  // JS のプライベートフィールド（# が付くとクラス外から触れない）
  // 選択範囲の開始・終了オフセットと選択中の色を保持する
  #startPosition = null
  #endPosition   = null
  #selectedColor = "#ffeb3b"   // デフォルト色：黄

  // コントローラーが DOM に接続されたときに自動で呼ばれる
  connect() {
    // mouseup イベントのリスナーを登録する
    // bind(this) で「this がコントローラー自身」に固定する
    this._onMouseup = this.#handleMouseup.bind(this)
    this.bodyTarget.addEventListener("mouseup", this._onMouseup)
  }

  // コントローラーが DOM から切り離されたときに自動で呼ばれる
  // リスナーを解除してメモリリークを防ぐ
  disconnect() {
    this.bodyTarget.removeEventListener("mouseup", this._onMouseup)
  }

  // Stimulus の "Value コールバック" — annotationsValue が変化すると自動で呼ばれる
  // ページロード時に HTML の data 属性から値が読み込まれたときも発火するので
  // 「ページを開いたら既存アノテーションを描画する」という処理に使える
  annotationsValueChanged() {
    if (this.annotationsValue.length === 0) return
    this.annotationsValue.forEach(a => this.#renderAnnotation(a))
  }

  // 色スウォッチをクリックしたときに呼ばれる
  selectColor(event) {
    // クリックされたボタンの data-color 属性を読んで選択色を更新する
    this.#selectedColor = event.currentTarget.dataset.color

    // すべてのスウォッチを走査して、選択中のものだけ selected クラスを付ける
    this.element.querySelectorAll(".annotation-color-swatch").forEach(el => {
      el.classList.toggle("selected", el.dataset.color === this.#selectedColor)
    })
  }

  // テキスト本文でマウスボタンが離されたときに呼ばれる
  #handleMouseup() {
    const sel = window.getSelection()

    // 選択範囲がなければツールバーを隠して終了
    if (!sel || sel.isCollapsed) {
      this.#hideToolbar()
      return
    }

    const range = sel.getRangeAt(0)
    const container = this.bodyTarget

    // 選択範囲がテキスト本文コンテナの外なら無視する
    if (!container.contains(range.commonAncestorContainer)) {
      this.#hideToolbar()
      return
    }

    // 選択範囲の開始・終了を「コンテナ内の文字オフセット」に変換して保存
    this.#startPosition = this.#getTextOffset(container, range.startContainer, range.startOffset)
    this.#endPosition   = this.#getTextOffset(container, range.endContainer,   range.endOffset)

    // 選択範囲の画面上の位置を取得してツールバーをその上に表示する
    const rect = range.getBoundingClientRect()
    const toolbar = this.toolbarTarget
    toolbar.style.position  = "absolute"
    toolbar.style.top       = `${rect.top + window.scrollY - toolbar.offsetHeight - 8}px`
    toolbar.style.left      = `${rect.left + window.scrollX + rect.width / 2}px`
    toolbar.style.transform = "translateX(-50%)"  // 水平方向に中央揃え
    toolbar.classList.remove("d-none")
  }

  // ハイライト／アンダーラインボタンをクリックしたときに呼ばれる
  annotate(event) {
    const annotationType = event.currentTarget.dataset.type  // "highlight" or "underline"
    const token = document.querySelector('meta[name="csrf-token"]').getAttribute("content")
    const start = this.#startPosition
    const end   = this.#endPosition
    const color = this.#selectedColor

    // Rails に POST して保存する
    fetch(this.urlValue, {
      method: "POST",
      headers: { "Content-Type": "application/json", "X-CSRF-Token": token },
      body: JSON.stringify({
        annotation: {
          start_position:  start,
          end_position:    end,
          annotation_type: annotationType,
          color:           color
        }
      })
    })
    .then(res => res.json())
    .then(data => {
      if (data.id) {
        // 保存成功したら即座に DOM へレンダリングする（ページリロード不要）
        this.#renderAnnotation({ ...data, start_position: start, end_position: end, color })
        this.#hideToolbar()
        window.getSelection()?.removeAllRanges()
      }
    })
  }

  // ツールバーを隠し、保存していたオフセットをリセットする
  #hideToolbar() {
    this.toolbarTarget.classList.add("d-none")
    this.#startPosition = null
    this.#endPosition   = null
  }

  // オフセット情報をもとに DOM へ <span> を挿入してアノテーションを描画する
  #renderAnnotation(annotation) {
    // オフセット → DOM Range に変換する（#offsetToRange の逆）
    const range = this.#offsetToRange(
      this.bodyTarget, annotation.start_position, annotation.end_position
    )
    if (!range) return

    // アノテーション種別に応じた CSS クラスを持つ <span> を作成する
    const span = document.createElement("span")
    span.classList.add(`annotation-${annotation.annotation_type}`)
    span.dataset.annotationId = annotation.id  // 後で削除などに使える

    // 色をインラインスタイルで適用する
    if (annotation.color) {
      if (annotation.annotation_type === "highlight") {
        span.style.backgroundColor = annotation.color
      } else {
        span.style.borderBottomColor = annotation.color
      }
    }

    try {
      // 単純なケース（単一テキストノード内の選択）はこちらで処理
      range.surroundContents(span)
    } catch (e) {
      // <p> をまたぐ選択や重複アノテーションはこちらにフォールバック
      // extractContents でコンテンツを取り出し span に入れてから挿入する
      const fragment = range.extractContents()
      span.appendChild(fragment)
      range.insertNode(span)
    }
  }

  // DOM の選択範囲（startContainer + startOffset）をコンテナ内の「テキスト全体の何文字目か」に変換する
  // TreeWalker でテキストノードを順に辿り、目的のノードに到達したらそれまでの累積文字数 + ノード内オフセット を返す
  #getTextOffset(container, targetNode, targetOffset) {
    let offset = 0
    const walker = document.createTreeWalker(container, NodeFilter.SHOW_TEXT)
    while (walker.nextNode()) {
      if (walker.currentNode === targetNode) return offset + targetOffset
      offset += walker.currentNode.length
    }
    return offset
  }

  // #getTextOffset の逆操作
  // 「コンテナ内の何文字目か」という数値から DOM Range を復元する
  //  TreeWalker でテキストノードを辿りながら文字数を累積し start / end それぞれが含まれるノードを見つけて Range を組み立てる
  #offsetToRange(container, start, end) {
    const range = document.createRange()
    let offset = 0
    let startSet = false

    const walker = document.createTreeWalker(container, NodeFilter.SHOW_TEXT)
    while (walker.nextNode()) {
      const node = walker.currentNode
      const len  = node.length

      // このノードの範囲に start が含まれるなら range の開始点を設定する
      if (!startSet && offset + len >= start) {
        range.setStart(node, start - offset)
        startSet = true
      }
      // このノードの範囲に end が含まれるなら range の終了点を設定して返す
      if (startSet && offset + len >= end) {
        range.setEnd(node, end - offset)
        return range
      }
      offset += len
    }
    return null  // 対応するノードが見つからなかった場合
  }
}
```

---

## 5️⃣ `application.css` にスタイルを追加

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

### 動作確認

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

#### 総学習時間： 1276.9 時間
