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
