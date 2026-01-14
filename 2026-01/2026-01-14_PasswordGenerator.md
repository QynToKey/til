- JavaScriptで「パスワードジェネレーター」を試作

  - `legend`タグ: HTMLの要素で、その親要素`fieldset`の内容やキャプションを表す。

  - `password-length`をスライダーで可変表示させる。
  ```
  # html (最小値と最大値を設定し初期値は'8'を表示)
  <input type="range" id="slider" min="8" max="24" value="8">

  # js (それぞれDOMを取得しスライダーの値をテキストに代入)
  passwordLength.textContent = slider.value;
  ```
