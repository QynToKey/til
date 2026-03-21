# JS でお札カウンターを実装

---

## 課題

- `+` / `-`ボタンで枚数・合計金額を更新
- 合計金額をカンマ区切りで表示する

---

## 学んだこと

- `querySelectorAll` + `forEach` で複数要素にイベントリスナーを登録できる
- `btn.parentNode.querySelector()` で関連する要素をたどれる
- 表示用と計算用の変数を分けることで `toLocaleString()` と併用できる

---

## 重要ポイント / ハイライト

- `let count = 0` の宣言位置がポイント。`forEach` のループ内・イベントリスナーの外に置くことで、各お札ごとに独立したカウントを保持できる
- `toLocaleString()` を使うとき、`textContent` から数値に戻す際にカンマが邪魔になる。計算用変数 `totalAmount` を別途保持することで解決できる

---

## プロセス

- 合計額の計算までは順調に実装できたが、`count` が更新できずに悩み込んだ。
  - `let count = 0;` をループ内に置くことで、`+` / `-` と紐づけることができた。
  - さらに、`-` ボタンの処理も同じ `+` ボタンと同じループ内に置くことで、 `count` の状態を `+` / `-` で共有した。

- 最後の最後に、`toLocaleString()` の実装で躓いた。
  - 合計額の変数 `totalAmount` を宣言し、計算と表示の責務を分離して実装した。

```js
const total = document.getElementById('total');
const plusBtns = document.querySelectorAll('.plus');

let totalAmount = 0;  // 合計額

plusBtns.forEach((btn) => {
  let count = 0;  // ループの内側 かつ イベントリスナーの外側
  const minusBtn = btn.parentNode.querySelector('.minus');  // 親要素を取得することで + と - のボタンを紐づける

  btn.addEventListener('click', () => {
    const countEl = btn.parentNode.querySelector('.count'); // 親要素を取得することで ボタンと count を紐づける
    count ++;
    countEl.textContent = count;

    totalAmount += Number(btn.dataset.price);  // 合計額の計算式
    total.textContent = totalAmount.toLocaleString();  // 合計額の表示方法

    minusBtn.disabled = false;
  });

  minusBtn.addEventListener('click', () => {
    const countEl = minusBtn.parentNode.querySelector('.count');
    count --;
    countEl.textContent = count;

    totalAmount -= Number(btn.dataset.price);
    total.textContent = totalAmount.toLocaleString();

    if (count === 0) {
      minusBtn.disabled = true;  // count が 0 になったら - ボタンを無効化
    }
  });
});
```

---

### ここまでの総学習時間 ： 1122.1 時間
