- JavaScriptで「パスワードジェネレーター」を試作

  - `legend`タグ: HTMLの要素で、その親要素`fieldset`の内容やキャプションを表す。

  - `password-length`をスライダーで可変表示させる。
    ```
    # html (最小値と最大値を設定し初期値は'8'を表示)
    <input type="range" id="slider" min="8" max="24" value="8">

    # js (それぞれDOMを取得しスライダーの値をテキストに代入)
    passwordLength.textContent = slider.value;
    ```

  - 文字列をランダムに生成する。
    ```
    # 小文字列と大文字列を用意
    const letters = 'abcdefghijklmnopqrstuvwxyz';
    const seedLetters = letters + letters.toUpperCase();

    # 文字列からランダムに取得した値を変数に代入
    let password = '';

    for (let i =0; i < slider.value; i++) {
      password += seedLetters[Math.floor(Math.random() * 52)];
    }

    # 変数を表示させる
    result.textContent = password;
    ```

    - 取得元の配列に「数字」や「記号」を追加する。
    ```
    # 「数字」の配列を取得元に加える実装例
    const numbersCheckbox = document.getElementById('numbers-checkbox');
    const numbers = '0123456789';

    if (numbersCheckbox.checked) {
      seed += numbers;
    }
    ```


- `Figma`

  - プラグイン

    - フリーの画像素材: アクション > プラグインとウィジェット > [**Unsplash**](https://unsplash.com/ja)

    - **地図**: アクション > プラグインとウィジェット > `Google Map`

  - スタイル

    - テキストスタイル: テキストの「書式設定」を登録できる
      - 「タイポグラフィ」の４つ丸アイコンを開いて設定

    - カラースタイル: 色のスタイルを登録できる
      - 「塗り」の４つ丸アイコンを開いて設定

  - プロトタイプ: 画面遷移などを設定し、「プレビュー」で動作を確認できる
