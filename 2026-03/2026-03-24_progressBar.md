# [卒制](https://github.com/QynToKey/HowLongWillItLast) (MVP 後)：プログレスバーのリファクタリング

## 0️⃣ 実装計画

- 両端の表示が切れる問題を修正
- マイルストーンごとに表示を切り替え（ブログレスバーをパーシャル化）
- マイページにも実装

---

## 1️⃣ 両端の表示が切れる問題を修正

- 左端の目盛りを右寄せ `translateX(0)` に
- 右端の目盛りを左寄せ `translateX(-100)` に

```erb
<%# app/views/home/index.html.erb %>
  <div style="position: relative; height: 28px; margin-top: 6px;">
    <span class="text-muted" style="position: absolute; left: 0%; font-size: 11px; transform: translateX(0);">0</span>
    ・・・
    <span class="text-muted text-center" style="position: absolute; left: 100%; font-size: 11px; transform: translateX(-100%);">10,000h<br>エキスパート</span>
  </div>
```

---

## 2️⃣ プログレスバーをパーシャル化して TOP ページに実装

```bash
touch app/views/shared/_progressbar.html.erb
```

```erb
<%# app/views/shared/_progressbar.html.erb %>

<% progress = [(@total_hours / 10000.0 * 100), 100].min.round(1) %>
<div class="progress mb-1" style="height: 6px;">
  <div class="progress-bar bg-dark" style="width: <%= progress %>%;"></div>
</div>

<div style="position: relative; height: 28px; margin-top: 6px;">
  ・・・
</div>
```

```erb
<%# app/views/home/index.html.erb %>
      <%= render "shared/progressbar" %>
      <%= yield %>
```

---

## 3️⃣ プログレスバーをマイルストーンごとに条件分岐

```erb
<%# 各ステージの上限を 100% として進捗状況を描画 %>
<% max_hours = if current_user.total_learning_minutes <= 1000 * 60
                 1000.0
               elsif current_user.total_learning_minutes <= 2500 * 60
                 2500.0
               elsif current_user.total_learning_minutes <= 5000 * 60
                 5000.0
               else
                 10000.0
               end %>
<% progress = [(@total_hours / max_hours * 100), 100].min.round(1) %>

<%# 各ステージごとに目盛りの配置を切り替える %>
<div style="position: relative; height: 28px; margin-top: 6px;">
  <% if current_user.total_learning_minutes <= 1000 * 60 %>
    <span class="text-muted" style="position: absolute; left: 0%; font-size: 11px; transform: translateX(0);">0</span>
    <span class="text-muted text-center" style="position: absolute; left: 100%; font-size: 11px; transform: translateX(-100%);">1,000h<br>初級</span>
  <% elsif (current_user.total_learning_minutes > 1000 * 60) && (current_user.total_learning_minutes <= 2500 * 60) %>
    <span class="text-muted" style="position: absolute; left: 0%; font-size: 11px; transform: translateX(0);">0</span>
    <span class="text-muted text-center" style="position: absolute; left: 40%; font-size: 11px; transform: translateX(-50%);">1,000h<br>初級</span>
    <span class="text-muted text-center" style="position: absolute; left: 100%; font-size: 11px; transform: translateX(-100%);">2,500h<br>中級</span>
  <% elsif (current_user.total_learning_minutes > 2500 * 60) && (current_user.total_learning_minutes <= 5000 * 60) %>
    <span class="text-muted" style="position: absolute; left: 0%; font-size: 11px; transform: translateX(0);">0</span>
    <span class="text-muted text-center" style="position: absolute; left: 20%; font-size: 11px; transform: translateX(-50%);">1,000h<br>初級</span>
    <span class="text-muted text-center" style="position: absolute; left: 50%; font-size: 11px; transform: translateX(-50%);">2,500h<br>中級</span>
    <span class="text-muted text-center" style="position: absolute; left: 100%; font-size: 11px; transform: translateX(-100%);">5,000h<br>上級</span>
  <% else %>
    <span class="text-muted" style="position: absolute; left: 0%; font-size: 11px; transform: translateX(0);">0</span>
    <span class="text-muted text-center" style="position: absolute; left: 10%; font-size: 11px; transform: translateX(-50%);">1,000h<br>初級</span>
    <span class="text-muted text-center" style="position: absolute; left: 25%; font-size: 11px; transform: translateX(-50%);">2,500h<br>中級</span>
    <span class="text-muted text-center" style="position: absolute; left: 50%; font-size: 11px; transform: translateX(-50%);">5,000h<br>上級</span>
    <span class="text-muted text-center" style="position: absolute; left: 100%; font-size: 11px; transform: translateX(-100%);">10,000h<br>エキスパート</span>
  <% end %>
</div>
```

---

## 4️⃣ マイページにプログレスバーを実装

### `UserController` にプログレスバーを表示するための変数を追加

```ruby
# app/controllers/users_controller.rb

  def show
    # プログレスバーを表示するための変数
    @total_hours = (current_user.total_learning_minutes / 60.0).round(1)
    @next_threshold = current_user.next_threshold

    ・・・
  end
```

### ビューに実装

```erb
<%# app/views/users/show.html.erb %>
      <p class='small mt-3'>総学習時間： <%= (current_user.total_learning_minutes / 60.0).round(1) %> 時間</p>

      <%= render "shared/progressbar" %>
      <%= yield %>
```

---

## 5️⃣ 10000時間到達時に表示するメッセージを追加

10000時間に到達するとプログレスバーが振り切ってしまうため、以下のように対応する。

```erb
<%# app/views/shared/_progressbar.html.erb %>

<% if @next_threshold.nil? %>
  <p class="small mt-2">🎉 エキスパートレベル達成！</p>
<% else %>
  <%# 既存のプログレスバー %>
<% end %>
```

👉 *プログレスバーを非表示にしてメッセージだけを表示*

### 備考

現実的にはエキスパート到達者がこのアプリに現れるのは数年後なので、
その後の「継続化」に対応する実装も検討する。

📝 実装案：
今後実装予定の `LearningTheme` にユーザー独自のゴール時間を設定できるようにする

---

#### 総学習時間：1139.3 時間
