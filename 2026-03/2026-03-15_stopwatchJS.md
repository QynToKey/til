# [卒制](https://github.com/QynToKey/HowLongWillItLast) (day 17)：ストップウォッチ機能の実装 (2/3)

## 2️⃣ 「ストップウォッチ」ビューの作成

👉 *ストップウォッチはパーシャル化して実装する (責務を分離して、アプリ設計に柔軟性を持たせたいため)*

```bash
touch app/views/learning_records/_stopwatch.html.erb
```

⬇️

```ruby
# app/views/learning_records/_stopwatch.html.erb
<div class="container">
  <div id="timer">00:00</div>
  <div class="controllers">
    <div class="btn" id="start">START</div>
    <div class="btn" id="stop">STOP</div>
    <div class="btn" id="reset">RESET</div>
  </div>
</div>
```

👉 *必要な要素を仮置き*

```ruby
# app/views/learning_records/new.html.erb
<%= render 'stopwatch' %>
```

👉 *記録作成画面から読み込む*

---

## 3️⃣ `stopwatch_controller.js`ファイルを作成

👉 *「ストップウォッチ」は学習で作成した JavaScript で実装する*

```bash
touch app/javascript/controllers/stopwatch_controller.js
```

```ruby
# app/javascript/controllers/stopwatch_controller.js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  connect() {
    console.log("stopwatch connected!") # ⬅️ 動作確認用（仮置き）
  }
}
```

```ruby
# app/views/learning_records/_stopwatch.html.erb
<div class="container" data-controller="stopwatch"> # ⬅️ stopwatch_controller.js を読み込む
  ・・・
</div>
```

👉 *ビュー側に `data-controller="stopwatch"` と書くことで、HTML と JS が紐付く*

---
