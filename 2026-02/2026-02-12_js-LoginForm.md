# JavaScript でログインフォームを操作する

## パスワードの隠し文字を表示する

- `<input type="password">` を `<input type="text">` に変更すればよい。
- `input` 要素の `type` 属性を取得するには `要素.type` を使う。

  ```js
  // 「目」のアイコンをクリックしたら隠し文字を展開する実装例
    eye.addEventListener('click', () => {
      const pw = document.getElementById('password');
      pw.type = pw.type === 'password' ? 'text' : 'password';
    });
  ```

  ```js
  // パスワード入力欄が空になったら強制的に戻すと安全
    pw.addEventListener('input', () => {
      if (pw.value.length === 0) {
        pw.type = "password";
      }
      …
    });
  ```

---

## 入力欄の状態によってアイコンの表示／非表示を切り替える

- アイコンの `classList.add()` / `classList.remove()` は、必ず反対側のクラスとセットで行う。

  👉 *`classList.replace()` を使うと remove + add を1行で書ける*

  ```js
  eye.classList.replace('bi-eye-fill', 'bi-eye-slash-fill');
  ```

---

## ログインボタンの 有効／無効 を切り替える

- `disabled` 属性の `true` / `false` を指定する。

  ```js
  document.getElementById('login').disabled = false; // ログインボタン有効
  document.getElementById('login').disabled = true; // ログインボタン無効
  ```

- `email` と `password` の両方が入力された時のみ「ログインボタン」を有効にする実装例。

  ```js
    email.addEventListener('input', () => {
      if (email.value.length > 0 && pw.value.length > 0) {
        login.disabled = false;
      } else {
      login.disabled = true;
      }
    });

    pw.addEventListener('input', () => {
      if (email.value.length > 0 && pw.value.length > 0) {
        login.disabled = false;
      } else {
      login.disabled = true;
      }
    });
  ```

    👉 *判定ロジックは「現在の全体状態を評価する」ため、`email` と `password` の両方の状態を管理する必要がある*

  ### リファクタリング例（重複したコードを整理する）

  - 関数に切り出して、ロジックを１箇所に集約する

    ```js
    function updateButton() {
      login.disabled = !(email.value && pw.value);
    }

    email.addEventListener('input', updateButton);
    pw.addEventListener('input', updateButton);
    ```

    👉 *`"`（空文字）は `false` 扱いになる*

---
