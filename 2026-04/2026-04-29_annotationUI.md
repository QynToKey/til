# [co-READER](https://github.com/QynToKey/co_reader)（day: 22_2）： ハイライト / アンダーライン UI

## 0️⃣ 実装方針

- `Annotation` モデル（実装済み）を活用し、ユーザーがテキスト本文を選択したときにハイライト／アンダーラインの種別とオフセットを保存する UI を構築する。
- ハイライト表示 / アンダーライン表示）はこの issue の保存処理に依存するため、本 issue で土台を作る。

---

## 1️⃣ ルーティングに `annotations` を追加

👉 *`Annotation` は「どのセッションのどのテキストに対する書き込みか」が必要なので、2段階ネストにする*

```ruby
# config/routes.rb
resources :texts, only: %i[ show ] do
  resource :text_reading, only: %i[ create ]
  resources :annotations, only: %i[ create ]
end
```

  ⬇️ 確認

```bash
$ docker compose exec web bin/rails routes | grep annotation
        reading_session_text_annotations POST   /reading_sessions/:reading_session_id/texts/:text_id/annotations(.:format)                        annotations#create
```

---

## 2️⃣ `AnnotationsController` を作成

```bash
touch app/controllers/annotations_controller.rb
```

```ruby
# app/controllers/annotations_controller.rb
class AnnotationsController < ApplicationController
  before_action :require_login

  def create
    # ログインユーザーを自動で紐付け
    annotation = current_user.annotations.build(annotation_params)
    annotation.reading_session = ReadingSession.find(params[:reading_session_id])
    annotation.text = Text.find(params[:text_id])

    # Stimulus からの fetch リクエストに対して JSON を返す
    if annotation.save
      render json: { id: annotation.id }, status: :created
    else
      render json: { errors: annotation.errors.full_messages }, status: :unprocessable_entity
    end
  end

  private

  # 受け取るパラメーターを明示的に許可
  def annotation_params
    params.require(:annotation).permit(:start_position, :end_position, :annotation_type)
  end
end
```

---

## 3️⃣ Stimulus コントローラーを作成

```bash
touch app/javascript/controllers/text_selection_controller.js
```

```js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static targets = ["body", "toolbar"]
  static values  = { url: String }

  #startPosition = null
  #endPosition   = null

  connect() {
    this._onMouseup = this.#handleMouseup.bind(this)
    this.bodyTarget.addEventListener("mouseup", this._onMouseup)
  }

  disconnect() {
    this.bodyTarget.removeEventListener("mouseup", this._onMouseup)
  }

  #handleMouseup() {
    const sel = window.getSelection()
    if (!sel || sel.isCollapsed) {
      this.#hideToolbar()
      return
    }

    // mouseup 後
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
    // toolbar を rect 上部に配置（position: fixed）
    toolbar.style.top       = `${rect.top + window.scrollY - toolbar.offsetHeight - 8}px`
    toolbar.style.left      = `${rect.left + window.scrollX + rect.width / 2}px`
    toolbar.style.transform = "translateX(-50%)"
    toolbar.classList.remove("d-none")
  }

  annotate(event) {
    const annotationType = event.currentTarget.dataset.type
    const token = document.querySelector('meta[name="csrf-token"]').getAttribute("content")

    fetch(this.urlValue, {
      method: "POST",
      headers: { "Content-Type": "application/json", "X-CSRF-Token": token },
      body: JSON.stringify({
        annotation: {
          start_position:  this.#startPosition,
          end_position:    this.#endPosition,
          annotation_type: annotationType
        }
      })
    })
    .then(res => {
      if (res.ok) {
        this.#hideToolbar()
        window.getSelection()?.removeAllRanges()
      }
    })
  }

  #hideToolbar() {
    this.toolbarTarget.classList.add("d-none")
    this.#startPosition = null
    this.#endPosition   = null
  }

  #getTextOffset(container, targetNode, targetOffset) {
    let offset = 0
    const walker = document.createTreeWalker(container, NodeFilter.SHOW_TEXT)
    while (walker.nextNode()) {
      if (walker.currentNode === targetNode) return offset + targetOffset
      offset += walker.currentNode.length
    }
    return offset
  }
}
```

📝 `#startPosition / #endPosition`
JS のプライベートフィールド（クラス外から触れない）

📝 `#getTextOffset`
DOM のテキストノードを順に辿り、選択位置を「テキスト全体の何文字目か」に変換

📝 `annotate(event)`
ボタンの `data-type` 属性（"highlight" or "underline"）を読み取って POST する

📝 `#hideToolbar`
ツールバーを隠し、保存済みオフセットをリセットする

> Values

| value | 用途 |
| --- | --- |
| url | POST 先 URL（ビューで生成） |

> Targets

| target | 用途 |
| --- | --- |
| body | テキスト本文コンテナ（オフセット計算の基準） |
| toolbar | フローティングツールバー要素 |

---

### 📝 **Stimulus**

Rails が作った**JS フレームワークの名前**。
Rails に限らず使えるが、Rails（Hotwire）と一緒に使うことが多い。

> 普通の JS との違い

- 普通の JS は「この要素を取得して、このイベントを登録して…」と自分で書く必要がある。

```javascript
// 普通の JS
document.querySelectorAll("[data-copy]").forEach(el => {
  el.addEventListener("click", () => { ... })
})
```

- Stimulus は HTML の `data` 属性と JS を自動で繋いでくれる。

```html
<!-- HTML 側で「このコントローラーを使う」と宣言 -->
<div data-controller="clipboard">
```

```javascript
// Stimulus 側は「接続されたら何をするか」だけ書けばいい
connect() { ... }
```

> Rails との相性

Rails の設計思想は「HTML を中心に置く」で、Stimulus も同じ考えから JS の動作を HTML の属性として表現する。

- `data-controller` — どのコントローラーか
- `data-action` — どのメソッドを呼ぶか
- `data-target` — どの要素を参照するか

この「HTML に意味を持たせる」スタイルが Rails と相性よく、Hotwire（Turbo + Stimulus）として Rails 7 から標準採用された。

---

## 4️⃣ 「テキスト詳細」ビューを修正

```erb
<%# app/views/texts/show.html.erb %>

-  <div class="lh-lg">
-    <%= simple_format(@text.body) %>
+  <div data-controller="text-selection"
+       data-text-selection-url-value="<%= reading_session_text_annotations_path(@reading_session, @text) %>">
+
+    <div data-text-selection-target="toolbar"
+         class="d-none position-absolute bg-dark text-white rounded px-2 py-1"
+         style="z-index: 1000; white-space: nowrap;">
+      <button data-action="click->text-selection#annotate"
+              data-type="highlight"
+              class="btn btn-sm btn-warning me-1">ハイライト</button>
+      <button data-action="click->text-selection#annotate"
+              data-type="underline"
+              class="btn btn-sm btn-info">アンダーライン</button>
+    </div>
+
+    <div class="lh-lg" data-text-selection-target="body">
+      <%= simple_format(@text.body) %>
+    </div>
   </div>
```

📝 `data-controller="text-selection"`
Stimulus が `text_selection_controller.js` を自動でロード（`_` が `-` になる命名規則）
📝 `data-text-selection-url-value`
コントローラーの `urlValue` に POST 先 URL を渡す
📝 `data-text-selection-target="toolbar"`
コントローラーの `toolbarTarget` として参照される
📝 `data-text-selection-target="body"`
テキスト本文コンテナ。オフセット計算の基準になる

---

## 5️⃣ i18n に関連語彙を追加

```ruby
# config/locales/views/ja.yml
     show:
       completed: "読了"
       mark_as_read: "読了にする"
+      annotations:
+        create:
+          success: "書き込みを保存しました"
+          error: "書き込みの保存に失敗しました"
```

---

### 動作確認

- [ ] Docker 環境で `bin/rails db:migrate` 実行済みであることを確認（migration は既存）
- [ ] サーバー起動 → ログインしてテキスト表示ページへアクセス
テキストを選択 → ツールバーが表示されることを確認
- [ ] 「ハイライト」ボタンをクリック → ネットワークタブで `201 Created` が返ることを確認
- [ ] DB で `SELECT * FROM annotations;` を実行し、`start_position / end_position` が保存されていることを確認

---

#### 総学習時間： 1274.6 時間
