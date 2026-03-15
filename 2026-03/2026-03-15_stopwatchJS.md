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

```ruby
# app/views/learning_records/new.html.erb
<%= render 'stopwatch' %>
```

---
