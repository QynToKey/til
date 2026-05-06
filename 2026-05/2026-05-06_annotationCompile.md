# [co-READER](https://github.com/QynToKey/co_reader)（day: 28）： アノテーションの集約表示

## 0️⃣ 現状と実装方針

> 現状

- 共読モードで複数ユーザーが同じ箇所に書き込むと、重なったカラースパンで視認性が下がる。

> 実装方針

バッジ（件数）でグループを示し、クリックでサイドパネルに一覧を展開する。

---

## 1️⃣ 「テキスト詳細」ビューを修正

👉 *アノテーションデータに `user_id`・`user_name`・`created_at`（Unix タイムスタンプ）を追加。（バッジとパネルで「誰が」「いつ」書いたかを表示するため）。*

```erb
<%# app/views/texts/show.html %>
            data-text-selection-annotations-value="<%= (
-             @own_annotations.map { |a|
-               { id: a.id, annotation_type: a.annotation_type,
-                 start_position: a.start_position, end_position: a.end_position,
-                 color: a.color, fill_style: a.fill_style, is_own: true }
-             } +
-             @other_annotations.map { |a|
-               { id: a.id, annotation_type: a.annotation_type,
-                 start_position: a.start_position, end_position: a.end_position,
-                 color: a.color, fill_style: a.fill_style, is_own: false }
-             }
-           ).to_json %>">
+              @own_annotations.map { |a|
+                { id: a.id, annotation_type: a.annotation_type,
+                  start_position: a.start_position, end_position: a.end_position,
+                  color: a.color, fill_style: a.fill_style, is_own: true,
+                  user_id: a.user_id, user_name: a.user.name, created_at: a.created_at.to_i }
+              } +
+              @other_annotations.map { |a|
+                { id: a.id, annotation_type: a.annotation_type,
+                  start_position: a.start_position, end_position: a.end_position,
+                  color: a.color, fill_style: a.fill_style, is_own: false,
+                  user_id: a.user_id, user_name: a.user.name, created_at: a.created_at.to_i }
+              }
+            ).to_json %>">
```

👉 *共読モードのとき、`data-text-selection-favorite-user-ids-value` も追加する*

```erb
<%# app/views/texts/show.html %>
            data-text-selection-text-id-value="<%= @text.id %>"
            data-text-selection-mode-value="<%= @membership.mode %>"
            data-text-selection-current-user-id-value="<%= current_user.id %>"
+           <% if @membership.co_reading? %>
+           data-text-selection-favorite-user-ids-value="<%= @my_favorite_user_ids.to_a.to_json %>"
+           <% end %>
```

---

## 2️⃣ `text_selection_controller.js` を修正：バッジ生成ロジック

### `static values` に `favoriteUserIds: Array` を追加

```js
// text_selection_controller.js
static values  = { url: String, annotations: Array, deleteBaseUrl: String, commentBaseUrl: String,
                   readingSessionId: Number, textId: Number,
                   mode: String, currentUserId: Number,
                   favoriteUserIds: Array }
```

### プライベートフィールドを追加

```js
// text_selection_controller.js
#annotationMeta = {}   // annotationId → メタデータ（パネル表示・バッジ生成で参照）
```

### `annotationsValueChanged()` の末尾

```js
// text_selection_controller.js
// メタデータを蓄積する（バッジ・パネル表示に使用）
this.annotationsValue.forEach(a => {
  this.#annotationMeta[a.id] = {
    user_id: a.user_id, user_name: a.user_name, created_at: a.created_at,
    start_position: a.start_position, end_position: a.end_position,
    annotation_type: a.annotation_type, color: a.color
  }
})
if (this.modeValue === "co_reading") this.#badgeOverlappingGroups()
```

### クラス末尾（`#isFullyCovered` の後）に2つのメソッドを追加

```js
// text_selection_controller.js
// 共読モードで重なり合うアノテーションを検出してバッジを付ける
#badgeOverlappingGroups() {
  // bodyTarget 内の全 annotation span を start_position でソート
  const spans = [...this.bodyTarget.querySelectorAll("[data-annotation-id]")]
  const entries = spans.map(span => ({
    id: parseInt(span.dataset.annotationId),
    start: parseInt(span.dataset.startPosition),
    end: parseInt(span.dataset.endPosition),
    span
  })).sort((a, b) => a.start - b.start)

  // 重なり判定でグループ化
  const groups = []
  for (const entry of entries) {
    let placed = false
    for (const group of groups) {
      if (group.some(g => g.start < entry.end && entry.start < g.end)) {
        group.push(entry)
        placed = true
        break
      }
    }
    if (!placed) groups.push([entry])
  }

  // 2件以上のグループにバッジを付ける
  for (const group of groups) {
    if (group.length < 2) continue
    const badge = document.createElement("sup")
    badge.className = "annotation-badge"
    badge.textContent = group.length
    badge.dataset.group = JSON.stringify(group.map(e => e.id))
    badge.addEventListener("click", e => {
      e.stopPropagation()
      this.#handleBadgeClick(JSON.parse(e.currentTarget.dataset.group))
    })
    // 最初の span の直後に挿入する
    group[0].span.insertAdjacentElement("afterend", badge)
  }
}

// バッジクリック：書き込み一覧をサイドパネルに展開する
#handleBadgeClick(annotationIds) {
  const text = this.bodyTarget.textContent
  const annotations = annotationIds.map(id => {
    const meta = this.#annotationMeta[id]
    if (!meta) return null
    return {
      ...meta,
      excerpt: text.slice(meta.start_position, meta.end_position).substring(0, 30)
    }
  }).filter(Boolean)

  window.dispatchEvent(new CustomEvent("annotationGroupSelected", {
    detail: { annotations, favoriteUserIds: this.favoriteUserIdsValue }
  }))
}
```

---

## 3️⃣ 「サイドパネル」パーシャルに書き込み一覧セクション追加

```erb
<%# app/views/texts/_side_panel.html.erb%>

<div class="border rounded p-3 mt-3"
     data-controller="annotation-panel">
  <h6 class="mb-2">書き込み一覧</h6>
  <p class="text-muted small mb-0"
     data-annotation-panel-target="placeholder">
    テキスト上のバッジをクリックしてください
  </p>
  <div data-annotation-panel-target="list" class="d-none"></div>
</div>
```

---

## 4️⃣ `annotation_panel_controller.js` を作成

`` を新規作成：

```js
// app/javascript/controllers/annotation_panel_controller.js

import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static targets = ["list", "placeholder"]

  connect() {
    // window に発火されたカスタムイベントを受け取るため、バインド済み関数を保持しておく
    // disconnect 時に同じ参照で removeEventListener できるようにするための慣用パターン
    this._onSelect = this.#handleGroupSelected.bind(this)
    window.addEventListener("annotationGroupSelected", this._onSelect)
  }

  disconnect() {
    window.removeEventListener("annotationGroupSelected", this._onSelect)
  }

  // text_selection_controller の #handleBadgeClick から発火されるイベントを受け取る
  // event.detail には annotations（メタデータ配列）と favoriteUserIds が入っている
  #handleGroupSelected(event) {
    const { annotations, favoriteUserIds } = event.detail

    // Set にしておくことで has() が O(1) になる
    const fav = new Set(favoriteUserIds)

    // ソート順：① お気に入りユーザー優先 → ② 新しい順（created_at 降順）→ ③ テキスト上の位置順（start_position 昇順）
    const sorted = [...annotations].sort((a, b) => {
      const af = fav.has(a.user_id) ? 0 : 1
      const bf = fav.has(b.user_id) ? 0 : 1
      if (af !== bf) return af - bf
      if (b.created_at !== a.created_at) return b.created_at - a.created_at
      return a.start_position - b.start_position
    })

    // innerHTML でまとめて書き換える（件数が少ないため逐次 appendChild より簡潔）
    // XSS 対策：user_name と excerpt はサーバーから来た値なので、将来的にエスケープを検討すること
    this.listTarget.innerHTML = sorted.map(a => `
      <div class="border rounded p-2 mb-2 small">
        <span class="fw-bold">${a.user_name}</span>
        ${fav.has(a.user_id) ? '<span class="text-warning ms-1">★</span>' : ''}
        <span class="badge bg-secondary ms-1">${a.annotation_type === "highlight" ? "HL" : "UL"}</span>
        <div class="text-muted mt-1">「${a.excerpt}…」</div>
      </div>
    `).join("")

    // 一覧を表示し、案内文を隠す
    this.listTarget.classList.remove("d-none")
    this.placeholderTarget.classList.add("d-none")
  }
}
```

---

## 5️⃣ CSS にバッジのスタイルを追加

```css
/* app/assets/stylesheets/application.css` */
.annotation-badge {
  display: inline-block;
  background: #6c757d;
  color: white;
  border-radius: 50%;
  font-size: 0.65rem;
  width: 1.2em;
  height: 1.2em;
  line-height: 1.2em;
  text-align: center;
  cursor: pointer;
  vertical-align: super;
  margin-left: 1px;
}
```

---

### 動作確認

- [x] 共読モードで2人以上が同じ箇所に書き込みがある状態でページを開く → 重なった箇所に小さな数字バッジが表示されること
- [x] バッジをクリック → サイドパネルの「書き込み一覧」に一覧が展開されること
- [x] お気に入りユーザーの書き込みが先頭に来ていること
- [x] 孤読モードではバッジが表示されないこと（modeValue === "co_reading" ガード）

---

### CSS tips

📝 **`vertical-align`**

バッジの表示位置は以下を参照（今回は `bottom` を選択）

| 値 | 位置 |
| --- | --- |
| `super` | 上付き（現状） |
| `sub` | 下付き |
| `top` | 行の上端に揃える |
| `bottom` | 行の下端に揃える |
| `middle` | 行の中央 |
| `baseline` | ベースライン（通常の文字と同じ高さ） |
| `text-top` | 親要素のフォントの上端 |
| `text-bottom` | 親要素のフォントの下端 |
| `0.5em` など | 数値で直接指定（プラスで上・マイナスで下） |

👉 *細かく調整したい場合は数値指定が一番自由度が高い。たとえば `vertical-align: -0.3em` なら「ベースラインより少し下」に表示*

---

### 「書き込み一覧」の仕様

以下の仕様にリファクタリングする。

- バッジ表示条件：他ユーザーのアノテーションが1件以上
- バッジの数字：他ユーザーの件数のみ
- パネルの並び順：① 自分 → ② お気に入り → ③ 新しい順 → ④ 位置順

> 修正

#### `#annotationMeta` に `is_own` を追加

```js
// text_selection_controller.js`
this.#annotationMeta[a.id] = {
  user_id: a.user_id, user_name: a.user_name, created_at: a.created_at,
  start_position: a.start_position, end_position: a.end_position,
  annotation_type: a.annotation_type, color: a.color,
  is_own: a.is_own   // ← 追加
}
```

#### `#badgeOverlappingGroups()` のバッジ生成条件を変更

```js
//text_selection_controller.js`
for (const group of groups) {
  // 他ユーザーのアノテーションだけを抽出する
  const otherCount = group.filter(e => !this.#annotationMeta[e.id]?.is_own).length
  // 他ユーザーが1件もいなければバッジを付けない
  if (otherCount < 1) continue

  const badge = document.createElement("sup")
  badge.className = "annotation-badge"
  badge.dataset.count = otherCount   // 他ユーザーの件数のみ表示
  // 以下は既存コードのまま
```

#### ソート順を変更

```js
// annotation_panel_controller.js
const sorted = [...annotations].sort((a, b) => {
  // ① 自分のアノテーションを最上部
  const aOwn = a.is_own ? 0 : 1
  const bOwn = b.is_own ? 0 : 1
  if (aOwn !== bOwn) return aOwn - bOwn
  // ② 他ユーザー内：お気に入り優先
  const af = fav.has(a.user_id) ? 0 : 1
  const bf = fav.has(b.user_id) ? 0 : 1
  if (af !== bf) return af - bf
  // ③ 新しい順
  if (b.created_at !== a.created_at) return b.created_at - a.created_at
  // ④ テキスト上の位置順
  return a.start_position - b.start_position
})
```

---

##### 総学習時間： 1310.7 時間
