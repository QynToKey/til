# [卒制](https://github.com/QynToKey/HowLongWillItLast) (MVP 後)：プログレスバーのリファクタリング

## 0️⃣ 実装計画

- 両端の表示が切れる問題を修正
- マイルストーンごとに表示を切り替え（ブログレスバーをパーシャル化）
- マイページにも実装

---

## 1️⃣ 両端の表示が切れる問題を修正

- 左端の目盛りを右寄せ `translateX(0)` に
- 右端の目盛りを左寄せ `translateX(1000)` に

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
<%# app/views/home/index.html.erb %>
      <%= render "shared/progressbar" %>
      <%= yield %>
```

---

## 3️⃣ プログレスバーをマイルストーンごとに条件分岐

```erb
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
