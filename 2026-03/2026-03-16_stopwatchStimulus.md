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
  <div id="timer" data-stopwatch-target="timer">00:00</div>
  <div class="controllers">
    <div class="btn" id="start" data-stopwatch-target="start">START</div>
    <div class="btn" id="stop" data-stopwatch-target="stop">STOP</div>
    <div class="btn" id="reset" data-stopwatch-target="reset">RESET</div>
  </div>
</div>
```

| 属性 | 働き |
| --- | --- |
| `data-controller="stopwatch"` | この HTML に `stopwatch_controller.js` を紐づける |
| `data-stopwatch-target="要素"` | JS側で `this.[要素]Target` として参照できる |

- JavaScript 側 `app/javascript/controllers/stopwatch_controller.js`

```javascript
export default class extends Controller {
  static targets = ["timer", "start", "stop", "reset"]

  connect() {
    console.log(this.timerTarget)
    console.log(this.startTarget)
    console.log(this.stopTarget)
    console.log(this.resetTarget)
  }
}
```

👉 *`console.log()` で要素が取得できていることをコンソールで確認*

---
