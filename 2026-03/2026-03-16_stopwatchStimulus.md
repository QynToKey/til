# [卒制](https://github.com/QynToKey/HowLongWillItLast) (day 18)：ストップウォッチ機能の実装：Stimulus (3/3)

## Stimulus とは

- Rails 7 以降ではデフォルトで導入されている HTML 主導の軽量 JS フレームワーク
- HTML に属性を追加するだけで JavaScript の動きを加えられる
- DOM 中心のシンプルな設計
- Turbo との親和性が高く、組み合わせることで部分的なページ更新が可能

### Stimulus の実装イメージ

```markdown
1. HTMLに `data-controller="stopwatch"` を書く
  ⬇️
2. Stimulusが自動的に `stopwatch_controller.js` を見つける
  ⬇️
3. `stopwatch_controller.js` の処理が動く
```

---

## 4️⃣ `stopwatch_controller.js` を実装

### DOM 要素を取得

- ビュー側 'app/views/learning_records/_stopwatch.html.erb'

```html
<div class="container" data-controller="stopwatch">
  <div data-stopwatch-target="timer">00:00</div>
  <div class="controllers">
    <div class="btn" data-stopwatch-target="start" data-action="click->stopwatch#start">START</div>
    <div class="btn" data-stopwatch-target="stop" data-action="click->stopwatch#stop">STOP</div>
    <div class="btn" data-stopwatch-target="reset" data-action="click->stopwatch#reset">RESET</div>
  </div>
</div>
```

👉 *ID 指定 `id='xxxx` は不要*

| 属性 | 働き | 静的サイトでの JS コード |
| --- | --- | --- |
| `data-controller="stopwatch"` | この HTML に `stopwatch_controller.js` を紐づける | `<script src="js/main.js"></script>` に相当 |
| `data-stopwatch-target="要素"` | JS側で `this.[要素]Target` として参照できる | `document.getElementById("要素")` に相当 |
| `data-action="click->stopwatch#start"` | `[イベント名]->[コントローラ名]#[メソッド名]` でイベントを設定 | `addEventListener()` に相当 |

- JavaScript 側 `app/javascript/controllers/stopwatch_controller.js`

```javascript
export default class extends Controller {
  // DOM要素を取得するためのターゲットを定義
  static targets = ["timer", "start", "stop", "reset"]

  // ストップウォッチの状態を管理するためのプロパティを定義
  startTime = undefined
  timeoutId = undefined
  elapsedTime = 0

  connect() {
    // ⬇️ 動作確認のため
    console.log(this.timerTarget)
    console.log(this.startTarget)
    console.log(this.stopTarget)
    console.log(this.resetTarget)
  }
}
```

👉 *一旦、`console.log()` で要素が取得できていることをコンソールで確認*

---

### 基本動作を実装

👉 *別途作成済みの [StopwatchApp](https://github.com/QynToKey/MyStopwatch) を Stimulus の書式に変換して移植*

```javascript
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  // DOM要素を取得するためのターゲットを定義
  static targets = ["timer", "start", "stop", "reset"]

  // ストップウォッチの状態を管理するためのプロパティを定義
  startTime = undefined
  timeoutId = undefined
  elapsedTime = 0

  // コントローラーが接続されたときに呼び出されるメソッド
  connect() {
    this.setButtonStateInitial();
  }

  setButtonStateInitial() {
    this.startTarget.classList.remove("inactive")
    this.stopTarget.classList.add("inactive")
    this.resetTarget.classList.add("inactive")
  }

  // スタートボタンがクリックされたときに呼び出されるメソッド
  start() {
    if (this.startTime) return; // すでにスタートしている場合は何もしない

    this.startTime = Date.now() - this.elapsedTime; // 経過時間を考慮して開始時間を設定
    this.updateTimer(); // タイマーを更新
    this.setButtonStateRunning(); // ボタンの状態を更新
  }

  // ストップボタンがクリックされたときに呼び出されるメソッド
  stop() {
    if (!this.startTime) return; // ストップウォッチがスタートしていない場合は何もしない

    clearTimeout(this.timeoutId); // タイマーの更新を停止
    this.elapsedTime = Date.now() - this.startTime; // 経過時間を保存
    this.startTime = undefined; // 開始時間をリセット
    this.setButtonStateStopped(); // ボタンの状態を更新
  }

  // リセットボタンがクリックされたときに呼び出されるメソッド
  reset() {
    clearTimeout(this.timeoutId); // タイマーの更新を停止
    this.startTime = undefined; // 開始時間をリセット
    this.elapsedTime = 0; // 経過時間をリセット
    this.timerTarget.textContent = "00:00"; // タイマー表示をリセット
    this.setButtonStateInitial(); // ボタンの状態を初期化
  }

  // タイマーを更新するためのメソッド
  updateTimer() {
    const elapsedTime = Date.now() - this.startTime; // 経過時間を計算
    const minutes = String(Math.floor(elapsedTime / 60000)).padStart(2, "0"); // 分を計算してゼロ埋め
    const seconds = String(Math.floor((elapsedTime % 60000) / 1000)).padStart(2, "0"); // 秒を計算してゼロ埋め
    this.timerTarget.textContent = `${minutes}:${seconds}`; // タイマー表示を更新

    this.timeoutId = setTimeout(() => this.updateTimer(), 1000); // 1秒ごとにタイマーを更新
  }

  // ボタンの状態を更新するためのメソッド
  setButtonStateRunning() {
    this.startTarget.classList.add("inactive")
    this.stopTarget.classList.remove("inactive")
    this.resetTarget.classList.add("inactive")
  }

  // ボタンの状態を更新するためのメソッド
  setButtonStateStopped() {
    this.startTarget.classList.remove("inactive")
    this.stopTarget.classList.add("inactive")
    this.resetTarget.classList.remove("inactive")
  }
}
```

---
