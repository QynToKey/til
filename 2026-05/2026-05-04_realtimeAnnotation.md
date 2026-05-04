# [co-READER](https://github.com/QynToKey/co_reader)（day: 26_1）： 書き込みのリアルタイム反映

## 0️⃣ Hotwire について

📝 **Hotwire** = **H**TML **O**VER **T**HE **WIRE**
  Turbo + Stimulus + Strada* の総称。

> **Turbo Stream**

サーバーが HTML の断片を WebSocket（ActionCable）経由で送り、クライアントの DOM を自動で更新する仕組み。

```text
サーバー → "<turbo-stream action='append'><template>...</template></turbo-stream>" → ブラウザ
```

ブラウザ側はコードなしで DOM が更新される。

---

> 実装の狙い

共読モードで複数ユーザーが同じテキストを読んでいるとき、一方が書き込みを追加・削除すると他方の画面にもリアルタイムで反映されるようにする。

> 現在のアーキテクチャ：

- アノテーションは `Stimulus controller` が JSON fetch で作成・削除し、DOM を直接操作している
- ActionCable は未使用（app/channels/ ディレクトリも存在しない）
- `solid_cable gem` は Gemfile に含まれており `config/cable.yml` 設定済み

> 設計方針

- 既存の rendering が JS 主導のため
  ❌ HTML partial を `broadcast` する `Turbo::Broadcastable` パターン
  ⭕️ ActionCable で JSON を `broadcast` → Stimulus で受信して既存の `#renderAnnotation` を再利用する

📝 なぜ Turbo Stream より ActionCable JSON が適切か

1. 本アプリのアノテーション描画は、すでに Stimulus が **JS でゼロから DOM を構築** している（`#renderAnnotation` メソッド）。
2. ここに Turbo Stream の HTML 断片を流しても、Stimulus の座標計算ロジックを通らないため正しく描画できない。

| | Turbo Stream broadcast | ActionCable JSON broadcast（今回） |
| --- | --- | --- |
| サーバーが送るもの | HTML 断片 | JSON データ |
| クライアントの処理 | DOM 自動更新 | Stimulus が受け取り `#renderAnnotation` を呼ぶ |
| 既存コードの流用 | できない | できる |

👉 *「Hotwire のうち ActionCable の部分を使い、Stimulus で処理する」設計を採用し、最小限の変更で「リアルタイム反映」を達成する*

---

## 1️⃣ `ApplicationCable` 基盤ファイルの作成

👉 *`ActionCable`（WebSocket）の接続を管理する基盤クラスを作成*

### `connection.rb` を作成

```bash
mkdir -p app/channels/application_cable/ && touch app/channels/application_cable/connection.rb
```

```ruby
# app/channels/application_cable/connection.rb
module ApplicationCable
  class Connection < ActionCable::Connection::Base
    identified_by :current_user

    # 「誰が接続しているのか」を特定するためのコード
    def connect
      verified_user = User.find_by(id: request.session[:user_id])
      # もしユーザーが見つからなければ、接続を拒否する
      reject_unauthorized_connection unless verified_user
      self.current_user = verified_user
    end
  end
end
```

### `channel.rb` を作成

```bash
touch app/channels/application_cable/channel.rb
```

```ruby
# app/channels/application_cable/channel.rb
module ApplicationCable
  class Channel < ActionCable::Channel::Base
  end
end
```

👉 *すべてのチャンネルが継承する基底クラス。今は中身が空だが、将来チャンネル共通の処理を書く場所になる。*

---

## 2️⃣ `AnnotationChannel` を作成

```bash
touch app/channels/annotation_channel.rb
```

```ruby
# app/channels/annotation_channel.rb
class AnnotationChannel < ApplicationCable::Channel
  # 「どのテキストページの更新を受け取るか」を決める
  def subscribed
    stream_from "annotation_channel:#{params[:reading_session_id]}:#{params[:text_id]}"
  end
end
```

👉 *WebSocket では、接続した全ユーザーに同じ情報を流すのではなく「ストリーム名」を使って宛先を絞るため、 `reading_session_id` と `text_id` の組み合わせをストリーム名にする。*

📝 **`params`**
  JS 側から接続時に渡す値（次のステップで Stimulus から送る）

📝 **`stream_from`**
  指定したストリーム名の broadcast を受信し始める

---

## 3️⃣ `Annotation` モデルに `broadcast` を追加

アノテーションが保存・削除されたとき、**同じページを開いている他のユーザーに通知**する処理を追加。

```ruby
# app/models/annotation.rb

# _commit 系を使うことで、DB トランザクション確定後に broadcast する（競合状態を防ぐ）
+  after_create_commit  :broadcast_create_to_channel
+  after_destroy_commit :broadcast_destroy_to_channel
+
+  private
+
+  def broadcast_create_to_channel
+    ActionCable.server.broadcast(
+      "annotation_channel:#{reading_session_id}:#{text_id}",
+      {
+        action:     "create",
+        annotation: {
+          id:              id,
+          annotation_type: annotation_type,
+          start_position:  start_position,
+          end_position:    end_position,
+          color:           color,
+          fill_style:      fill_style
+        }
+      }
+    )
+  end

# after_destroy_commit 時点では id などの属性値はまだ参照できる
+  def broadcast_destroy_to_channel
+    ActionCable.server.broadcast(
+      "annotation_channel:#{reading_session_id}:#{text_id}",
+      { action: "destroy", annotation_id: id }
+    )
+  end
```

👉 *ストリーム名は Step 2 で定義したものと一致させる（`"annotation_channel:#{rs_id}:#{text_id}"`）*

📝 **`action: "create"` / `"destroy"`**
  次の Step で Stimulus がこの値を見て処理を分岐する

📝 コールバック **`after_create_commit`** / **`after_destroy_commit`**

- 「DB への保存が確定した直後に実行される」処理。ここで ActionCable 経由でデータを broadcast します。
- 削除後なので、`id` など属性値はまだ参照できる。

---

## 4️⃣ `importmap` に `@rails/actioncable` を追加

> 実装のねらい

- Stimulus から ActionCable を使うには、`@rails/actioncable` を JavaScript の import 対象として登録する必要がある。

- 本アプリは npm ではなく **importmap** でJS パッケージを管理しているため、`pin` を1行追加し、`import { createConsumer } from "@rails/actioncable"` が使えるようにする。

- `actioncable.esm.js` は Rails が `public/assets/` に配置済みのファイルなので、外部 CDN や npm install は不要です。

```ruby
# config/importmap.rb
  pin "@hotwired/turbo-rails", to: "turbo.min.js"
+ pin "@rails/actioncable", to: "actioncable.esm.js"
```

👉 *`importmap.rb` の pin 行は宣言の列挙であり、読み込み順序は記述順に依存しない*

---

## 5️⃣ 「テキスト詳細」ビューに data 属性を追加

> 実装のねらい

- Stimulus controller が ActionCable に接続するとき、「どの `reading_session` のどの `text` か」をチャンネルに伝える必要がある。

- その値を HTML の `data` 属性経由で Stimulus に渡す。

- Rails の **Stimulus Values** という仕組みを使って、`data-[controller]-[name]-value` という命名規則で書くと、JS 側で `this.[name]Value` として受け取ることができる。

```erb
<%# app/views/texts/show.html.erb %>
        data-text-selection-url-value="<%= reading_session_text_annotations_path(@reading_session, @text) %>"
        data-text-selection-delete-base-url-value="/annotations/"
        data-text-selection-comment-base-url-value="/annotations/"
+       data-text-selection-reading-session-id-value="<%= @reading_session.id %>"
+       data-text-selection-text-id-value="<%= @text.id %>"
        data-text-selection-annotations-value="<%= @annotations.map { |a|
          { id: a.id, annotation_type: a.annotation_type,
            start_position: a.start_position, end_position: a.end_position,
```

---

## 6️⃣ Stimulus に ActionCable subscription を追加

### import を追加

ファイル先頭に1行追加。

```js
// text_selection_controller.js
  import { Controller } from "@hotwired/stimulus"
+ import { createConsumer } from "@rails/actioncable"
```

👉 *`createConsumer` は ActionCable の WebSocket 接続を作る関数*

### `static values` に2つ追加

```js
// text_selection_controller.js
- static values = { url: String, annotations: Array, deleteBaseUrl: String, commentBaseUrl: String }
+ static values = { url: String, annotations: Array, deleteBaseUrl: String, commentBaseUrl: String, readingSessionId: Number, textId: Number }
```

👉 *HTML に追加した data 属性を、Stimulus が `this.readingSessionIdValue` / `this.textIdValue` として受け取れるようにする。*

### private フィールドに `#subscription` を追加

```js
// text_selection_controller.js
  #editStart    = null
  #editEnd      = null
+ #subscription = null   // ActionCable サブスクリプション
```

### `connect()` 末尾と `disconnect()` に追加

👉 *`connect()` の末尾（`this.editPopupTarget.addEventListener` の直後）に subscription の登録を追加*

```js
// text_selection_controller.js
this.#subscription = createConsumer().subscriptions.create(
  {
    channel:            "AnnotationChannel",
    reading_session_id: this.readingSessionIdValue,
    text_id:            this.textIdValue
  },
  {
    received: (data) => {
      if (data.action === "create") {
        // 自分が作成した場合は既に DOM に描画済みなのでスキップ
        if (this.bodyTarget.querySelector(`[data-annotation-id="${data.annotation.id}"]`)) return
        this.#renderAnnotation(data.annotation)
      } else if (data.action === "destroy") {
        const span = this.bodyTarget.querySelector(`[data-annotation-id="${data.annotation_id}"]`)
        if (!span) return
        const parent = span.parentNode
        while (span.firstChild) parent.insertBefore(span.firstChild, span)
        parent.removeChild(span)
        this.bodyTarget.normalize()
      }
    }
  }
)
```

### `disconnect()` 末尾に1行追加

```js
// text_selection_controller.js
  disconnect() {
    this.bodyTarget.removeEventListener(...)
    ...
+   this.#subscription?.unsubscribe()
  }
```

- `received: (data) => { ... }` はアロー関数。これにより `this` がStimulusコントローラーを指し続ける（通常の `function` だと `this` が変わってしまう）
- `this.#renderAnnotation` は既存のメソッドをそのまま再利用
- `destroy` 処理は既存の削除ロジック（ポップアップの delete ボタン）と同じ手順

---

### 動作確認

2つのブラウザで別アカウントでログインし、同じ `reading_session` の同じテキストページを両方で開く

- [x] 片方でハイライト作成 → もう一方に反映されるか確認
- [x] 片方で削除 → もう一方からも消えるか確認

---

## 7️⃣ `TextsController` を修正

👉 *他ユーザーの既存アノテーションを読み込めるように、`@annotations` を全ユーザー分に拡張*

```ruby
# app/controllers/texts_controller.rb
-    @annotations = current_user.annotations.where(reading_session: @reading_session, text: @text)
+    @annotations = Annotation.includes(:comments)
+                .where(reading_session: @reading_session, text: @text)
```

※ 現状、全ユーザーのアノテーションが全件表示されているが、後続 issue でこれらを制御する実装を行う

---

### 総学習時間： 1298.6 時間
