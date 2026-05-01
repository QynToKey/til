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
+        <% [["#ffeb3b","黄"],["#86d188","緑"],["#ffb3b3","ピンク"],["#b3d4ff","青"]].each do |color, label| %>
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

// スウォッチ色（ハイライト用）に対応するアンダーライン用の色を返す
// factor で一律に暗くすると彩度も落ちてグレーに見えるため、色ごとに絶対値で指定する
// ?? hex は「マップに存在しない色が渡された場合はそのまま返す」という意味（将来色を追加した場合の安全弁）
function underlineColor(hex) {
  const map = {
    "#ffb3b3": "#dc3545",  // Bootstrap danger（ピンク → 赤）
    "#b3d4ff": "#0d6efd",  // Bootstrap primary（薄青 → 青）
  }
  return map[hex] ?? hex
}

export default class extends Controller {
  static targets = ["body", "toolbar", "editPopup"]
  static values  = { url: String, annotations: Array, deleteBaseUrl: String }

  #startPosition      = null      // 新規アノテーション：選択範囲の開始オフセット
  #endPosition        = null      // 新規アノテーション：選択範囲の終了オフセット
  #selectedColor      = "#ffeb3b" // 新規アノテーション：選択中の色
  #toolbarFocusedType = null      // ツールバー内で選択中の型（null = 未選択）
  #toolbarAnnotations = {}        // ツールバーセッション内で作成済みの type→span マップ
                                  // 2周目以降は POST ではなく PATCH で色を更新するために使う
  #activeAnnotations  = {}        // 編集中の位置にある type→span のマップ
  #focusedType        = null      // ポップアップ内で選択中の型（null = 未選択）
  #editStart          = null      // 編集中の位置の開始オフセット
  #editEnd            = null      // 編集中の位置の終了オフセット

  connect() {
    this._onMouseup   = this.#handleMouseup.bind(this)
    this._onClick     = this.#handleClick.bind(this)
    this._onDocClick  = this.#handleDocumentClick.bind(this)
    // editPopup は Stimulus action を使わず、直接リスナーを登録する
    // stopPropagation() を確実に呼ぶことで document リスナーへの干渉を防ぐ
    this._onPopupClick = this.#handlePopupClick.bind(this)
    this.bodyTarget.addEventListener("mouseup", this._onMouseup)
    this.bodyTarget.addEventListener("click",   this._onClick)
    document.addEventListener("click",          this._onDocClick)
    this.editPopupTarget.addEventListener("click", this._onPopupClick)
  }

  disconnect() {
    this.bodyTarget.removeEventListener("mouseup", this._onMouseup)
    this.bodyTarget.removeEventListener("click",   this._onClick)
    document.removeEventListener("click",          this._onDocClick)
    this.editPopupTarget.removeEventListener("click", this._onPopupClick)
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

  // ツールバーの色スウォッチをクリックする
  // 型が未選択の場合は選択色の更新だけ行う
  // 型が選択済みの場合：
  //   - 今回のツールバーセッションでまだ作成していない型 → POST で新規作成
  //   - すでに作成済みの型 → PATCH で色だけ更新（重複作成を防ぎ、試行錯誤を可能にする）
  selectColor(event) {
    const color = event.currentTarget.dataset.color
    this.#selectedColor = color
    // ツールバー内のスウォッチのみ .selected を更新する（editPopup に影響させない）
    this.toolbarTarget.querySelectorAll(".annotation-color-swatch").forEach(el => {
      el.classList.toggle("selected", el.dataset.color === color)
    })

    // 型が選択されていなければここで終了
    if (!this.#toolbarFocusedType) return

    const type  = this.#toolbarFocusedType
    const token = document.querySelector('meta[name="csrf-token"]').getAttribute("content")

    if (this.#toolbarAnnotations[type]) {
      // ── 作成済みの型 → 色を更新する（PATCH） ──────────────────────────
      const span = this.#toolbarAnnotations[type]
      const id   = span.dataset.annotationId
      fetch(`${this.deleteBaseUrlValue}${id}`, {
        method: "PATCH",
        headers: { "Content-Type": "application/json", "X-CSRF-Token": token },
        body: JSON.stringify({ annotation: { color } })
      })
      .then(res => {
        if (res.ok) {
          // 両プロパティをいったんクリアしてから設定する（スタイル残りを防ぐ）
          span.style.backgroundColor   = ""
          span.style.borderBottomColor = ""
          if (type === "highlight") {
            span.style.backgroundColor = color
          } else {
            // アンダーラインはスウォッチ色をそのまま使うと細い線では薄く見えるため、暗くした色を適用する
            span.style.borderBottomColor = underlineColor(color)
          }
          this.#resetToolbarTypeSelection()
        }
      })

    } else {
      // ── 未作成の型 → 新規作成する（POST） ────────────────────────────
      const start = this.#startPosition
      const end   = this.#endPosition
      fetch(this.urlValue, {
        method: "POST",
        headers: { "Content-Type": "application/json", "X-CSRF-Token": token },
        body: JSON.stringify({
          annotation: { start_position: start, end_position: end, annotation_type: type, color }
        })
      })
      .then(res => res.json())
      .then(data => {
        if (data.id) {
          const newSpan = this.#renderAnnotation({ ...data, start_position: start, end_position: end, color })
          // 作成した span を記録しておく（2周目以降で PATCH に切り替えるため）
          if (newSpan) this.#toolbarAnnotations[type] = newSpan
          this.#resetToolbarTypeSelection()
        }
      })
    }
  }

  // ツールバーのHL／ULボタンをクリックして編集対象の型を選択する
  // 同じ型を再クリックすると選択解除になる（トグル）
  annotate(event) {
    const type = event.currentTarget.dataset.type
    this.#toolbarFocusedType = (this.#toolbarFocusedType === type) ? null : type
    // 選択中のボタンに白い box-shadow を付ける
    // Bootstrap が outline: 0 で上書きするため、インラインスタイルで確実に表示する
    this.toolbarTarget.querySelectorAll("[data-type]").forEach(btn => {
      const isFocused = btn.dataset.type === this.#toolbarFocusedType
      btn.classList.toggle("active", isFocused)
      btn.style.boxShadow = isFocused ? "0 0 0 3px white" : ""
    })
  }

  // 既存アノテーションをクリックしたときに編集ポップアップを表示する
  #handleClick(event) {
    if (this.toolbarTarget.contains(event.target)) return
    if (this.editPopupTarget.contains(event.target)) return
    // ドラッグ選択直後の click はテキスト選択が残っているのでツールバーを閉じない
    // isCollapsed が false = 選択範囲あり = mouseup で表示したツールバーを維持する
    const sel = window.getSelection()
    if (sel && !sel.isCollapsed) return
    // アノテーションをクリックした時点でツールバーが開いていれば閉じる
    this.#hideToolbar()

    // クリック位置から annotation span を内側→外側へ遡って全件収集する
    // 同一範囲に HL と UL が重なっている場合、入れ子になっているので両方取得できる
    this.#activeAnnotations = {}
    this.#focusedType = null
    this.#editStart   = null
    this.#editEnd     = null

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
      target = target.parentElement?.closest("[data-annotation-id]")
    }

    if (Object.keys(this.#activeAnnotations).length === 0) {
      this.#hideEditPopup()
      return
    }

    // span の上端を基準にポップアップを配置する（ツールバーと表示位置を統一）
    // d-none を先に外して offsetHeight を確定させてから top を計算する（ツールバーと同じ方式）
    const anySpan = Object.values(this.#activeAnnotations)[0]
    const rect    = anySpan.getBoundingClientRect()
    const popup   = this.editPopupTarget
    // 初期状態はどちらの型も非アクティブ（ユーザーが選択してから編集を始める）
    // Bootstrap が outline を上書きするため、box-shadow インラインスタイルをリセットする
    this.editPopupTarget.querySelectorAll("[data-popup-action='type']").forEach(btn => {
      btn.classList.remove("active")
      btn.style.boxShadow = ""
    })
    popup.classList.remove("d-none")
    popup.style.position  = "absolute"
    popup.style.top       = `${rect.top + window.scrollY - popup.offsetHeight - 8}px`
    popup.style.left      = `${rect.left + window.scrollX + rect.width / 2}px`
    popup.style.transform = "translateX(-50%)"
  }

  // コントローラー要素の外をクリックしたらポップアップとツールバーを閉じる
  #handleDocumentClick(event) {
    if (this.element.contains(event.target)) return
    this.#hideEditPopup()
    this.#hideToolbar()
  }

  // editPopup 内のクリックをすべてここで処理する
  // Stimulus action を使わず直接リスナーを登録することで、
  // document への伝播を確実に止め #handleDocumentClick との干渉を防ぐ
  #handlePopupClick(event) {
    // ポップアップ内クリックが document リスナーに届かないようにする
    event.stopPropagation()

    // クリックされた要素から data-popup-action を持つボタンを探す
    const btn = event.target.closest("[data-popup-action]")
    if (!btn) return

    const action = btn.dataset.popupAction
    const token  = document.querySelector('meta[name="csrf-token"]').getAttribute("content")

    if (action === "type") {
      // ── 型ボタン：選択中の型を切り替える（トグル） ─────────────────
      // 同じ型を再クリックしたら選択解除
      const type = btn.dataset.type
      this.#focusedType = (this.#focusedType === type) ? null : type
      // 選択中のボタンに白い box-shadow を付ける
      // Bootstrap が outline: 0 で上書きするため、インラインスタイルで確実に表示する
      this.editPopupTarget.querySelectorAll("[data-popup-action='type']").forEach(b => {
        const isFocused = b.dataset.type === this.#focusedType
        b.classList.toggle("active", isFocused)
        b.style.boxShadow = isFocused ? "0 0 0 3px white" : ""
      })

    } else if (action === "color") {
      // ── 色スウォッチ：選択中の型に色を適用する ──────────────────────
      // 型が選択されていない場合は何もしない
      if (!this.#focusedType) return

      const type  = this.#focusedType
      const color = btn.dataset.color

      if (this.#activeAnnotations[type]) {
        // 既存アノテーションの色を更新する（PATCH）
        const span = this.#activeAnnotations[type]
        const id   = span.dataset.annotationId
        fetch(`${this.deleteBaseUrlValue}${id}`, {
          method: "PATCH",
          headers: { "Content-Type": "application/json", "X-CSRF-Token": token },
          body: JSON.stringify({ annotation: { color } })
        })
        .then(res => {
          if (res.ok) {
            // 両プロパティをいったんクリアしてから設定する
            // 型変更を経た span に古いスタイルが残っている場合の上書き漏れを防ぐ
            span.style.backgroundColor   = ""
            span.style.borderBottomColor = ""
            if (type === "highlight") {
              span.style.backgroundColor = color
            } else {
              // アンダーラインはスウォッチ色をそのまま使うと細い線では薄く見えるため、暗くした色を適用する
              span.style.borderBottomColor = underlineColor(color)
            }
            // ポップアップは閉じない（引き続き他の型の編集ができる）
          }
        })
      } else {
        // 選択した色で新しいアノテーションを作成する（POST）
        // オフセットが取得できていない場合は何もしない
        if (this.#editStart === null || isNaN(this.#editStart)) return
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
            // DOM に描画し、返ってきた span を activeAnnotations に登録する
            const newSpan = this.#renderAnnotation({
              ...data,
              start_position: this.#editStart,
              end_position:   this.#editEnd,
              color
            })
            if (newSpan) this.#activeAnnotations[type] = newSpan
            // ポップアップは閉じない（引き続き色の変更や他の型の追加ができる）
          }
        })
      }

    } else if (action === "delete") {
      // ── 削除ボタン：選択中の型のアノテーションだけを削除する ──────────
      // 型が選択されていない場合は何もしない
      if (!this.#focusedType) return
      const type = this.#focusedType
      const span = this.#activeAnnotations[type]
      if (!span) return

      const id = span.dataset.annotationId
      fetch(`${this.deleteBaseUrlValue}${id}`, {
        method: "DELETE",
        headers: { "X-CSRF-Token": token }
      })
      .then(res => {
        if (res.ok) {
          // span の中身を親ノードに戻してから span を除去する
          const parent = span.parentNode
          while (span.firstChild) parent.insertBefore(span.firstChild, span)
          parent.removeChild(span)
          // 隣接テキストノードを結合し、次回のオフセット計算を正確にする
          this.bodyTarget.normalize()
          // 削除した型を activeAnnotations から除く
          delete this.#activeAnnotations[type]
          this.#focusedType = null
          this.editPopupTarget.querySelectorAll("[data-popup-action='type']").forEach(b => {
            b.classList.remove("active")
            b.style.boxShadow = ""
          })
          // 残りのアノテーションがなければポップアップを閉じる
          if (Object.keys(this.#activeAnnotations).length === 0) {
            this.#hideEditPopup()
          }
        }
      })
    }
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

    this.#startPosition      = this.#getTextOffset(container, range.startContainer, range.startOffset)
    this.#endPosition        = this.#getTextOffset(container, range.endContainer,   range.endOffset)
    this.#toolbarAnnotations = {}   // 新しい選択のたびにセッションをリセットする

    // 選択範囲と重なる既存アノテーション span を収集する
    // ・toolbarAnnotations に登録 → selectColor で同型を選んだとき POST ではなく PATCH になり重複作成を防ぐ
    // ・overlappingSpans に収集 → 後続のカバレッジ判定に使う
    // 重なり判定：span の範囲と選択範囲が 1 文字でも重なっていれば対象とする
    const selStart = this.#startPosition
    const selEnd   = this.#endPosition
    const overlappingSpans = []
    container.querySelectorAll("[data-annotation-id]").forEach(span => {
      const spanStart = parseInt(span.dataset.startPosition)
      const spanEnd   = parseInt(span.dataset.endPosition)
      if (spanStart < selEnd && spanEnd > selStart) {
        const type = [...span.classList].find(c => c.startsWith("annotation-"))?.replace("annotation-", "")
        if (type && !this.#toolbarAnnotations[type]) {
          this.#toolbarAnnotations[type] = span
          overlappingSpans.push(span)
        }
      }
    })

    // 選択範囲が既存アノテーションで完全にカバーされている場合はポップアップを表示する
    // 一部でもカバーされていない場合（未注釈箇所を含む、またはアノテーションがない）はツールバーを表示する
    if (overlappingSpans.length > 0 && this.#isFullyCovered(selStart, selEnd, overlappingSpans)) {
      this.#hideToolbar()
      this.#activeAnnotations = { ...this.#toolbarAnnotations }
      this.#focusedType = null
      this.#editStart   = selStart
      this.#editEnd     = selEnd
      this.#toolbarAnnotations = {}
      const anySpan = overlappingSpans[0]
      const popupRect = anySpan.getBoundingClientRect()
      const popup = this.editPopupTarget
      this.editPopupTarget.querySelectorAll("[data-popup-action='type']").forEach(btn => {
        btn.classList.remove("active")
        btn.style.boxShadow = ""
      })
      popup.classList.remove("d-none")
      popup.style.position  = "absolute"
      popup.style.top       = `${popupRect.top + window.scrollY - popup.offsetHeight - 8}px`
      popup.style.left      = `${popupRect.left + window.scrollX + popupRect.width / 2}px`
      popup.style.transform = "translateX(-50%)"
      return
    }

    const rect = range.getBoundingClientRect()
    const toolbar = this.toolbarTarget
    // 先に d-none を外して offsetHeight を確定させてから top を計算する
    // d-none のまま offsetHeight を読むと 0 になり、ツールバーがテキスト上に重なってしまう
    this.#hideEditPopup()
    toolbar.classList.remove("d-none")
    toolbar.style.position  = "absolute"
    toolbar.style.top       = `${rect.top + window.scrollY - toolbar.offsetHeight - 8}px`
    toolbar.style.left      = `${rect.left + window.scrollX + rect.width / 2}px`
    toolbar.style.transform = "translateX(-50%)"
  }

  #hideToolbar() {
    this.toolbarTarget.classList.add("d-none")
    this.#startPosition      = null
    this.#endPosition        = null
    this.#toolbarAnnotations = {}   // セッション内の作成記録をリセットする
    // 型選択状態と box-shadow をリセットする
    this.#toolbarFocusedType = null
    this.toolbarTarget.querySelectorAll("[data-type]").forEach(btn => {
      btn.classList.remove("active")
      btn.style.boxShadow = ""
    })
  }

  // ツールバーの型選択状態だけをリセットする（ツールバー自体は閉じない）
  // POST・PATCH の完了後に呼び、次の型選択を受け付けられる状態にする
  // Note: removeAllRanges() は省略している。
  // contenteditable 要素がページ内にある環境で removeAllRanges() を呼ぶと選択範囲のテキストが DOM から消える場合があるため。
  #resetToolbarTypeSelection() {
    this.#toolbarFocusedType = null
    this.toolbarTarget.querySelectorAll("[data-type]").forEach(btn => {
      btn.classList.remove("active")
      btn.style.boxShadow = ""
    })
  }

  #hideEditPopup() {
    this.editPopupTarget.classList.add("d-none")
    // 編集状態をすべてリセットする
    this.#activeAnnotations = {}
    this.#focusedType = null
    this.#editStart   = null
    this.#editEnd     = null
  }

  // annotation を DOM に描画し、作成した span 要素を返す
  #renderAnnotation(annotation) {
    const range = this.#offsetToRange(
      this.bodyTarget, annotation.start_position, annotation.end_position
    )
    if (!range) return null

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
        // アンダーラインはスウォッチ色をそのまま使うと細い線では薄く見えるため、暗くした色を適用する
        span.style.borderBottomColor = underlineColor(annotation.color)
      }
    }

    const startParent = range.startContainer.nodeType === Node.TEXT_NODE
      ? range.startContainer.parentElement
      : range.startContainer
    const endParent = range.endContainer.nodeType === Node.TEXT_NODE
      ? range.endContainer.parentElement
      : range.endContainer

    const insideExisting = !!(startParent.dataset.annotationId && startParent === endParent)

    if (insideExisting) {
      const fragment = range.extractContents()
      span.appendChild(fragment)
      startParent.appendChild(span)
    } else {
      try {
        range.surroundContents(span)
      } catch (e) {
        const fragment = range.extractContents()
        span.appendChild(fragment)
        range.insertNode(span)
      }
    }

    return span
  }

  // TreeWalker でテキストノードを順に走査し、targetNode に到達するまでの文字数を累積することで、DOM 上の位置を「文字オフセット」に変換する
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
  // TreeWalker でテキストノードを走査し、累積長が start/end を超えた時点で setStart / setEnd を呼ぶ
  //
  // 注意：start 判定は >= ではなく > を使う。
  // surroundContents でテキストノードが分割されると、split 前の末尾ノードの累積長が start とぴったり一致する場合がある。このとき >= で判定すると、span の外にあるノードの末尾（= span の直前）を開始点に設定してしまい、次に同位置へ別の型のアノテーションを追加しようとすると挿入位置がズレる。
  // > にすることで、境界ぴったりの場合は次のノード（span 内テキストノードの先頭）を開始点として使う。
  #offsetToRange(container, start, end) {
    const range = document.createRange()
    let offset = 0
    let startSet = false

    const walker = document.createTreeWalker(container, NodeFilter.SHOW_TEXT)
    while (walker.nextNode()) {
      const node = walker.currentNode
      const len  = node.length

      if (!startSet && offset + len > start) {
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

  // 選択範囲 [start, end] が spans の和集合で完全にカバーされているか判定する
  // spans を開始位置でソートし、先頭から順に「ここまでカバー済み」を伸ばしていく
  // 途中でギャップ（カバーされていない箇所）が見つかれば false を返す
  #isFullyCovered(start, end, spans) {
    const sorted = spans
      .map(s => ({ start: parseInt(s.dataset.startPosition), end: parseInt(s.dataset.endPosition) }))
      .sort((a, b) => a.start - b.start)
    let covered = start
    for (const s of sorted) {
      if (s.start > covered) return false  // ギャップあり
      covered = Math.max(covered, s.end)
      if (covered >= end) return true
    }
    return covered >= end
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

##### 総学習時間： 1286.3 時間
