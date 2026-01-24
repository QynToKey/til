## JavaScriptで「[数字タッチゲーム](https://github.com/QynToKey/MyNumbersGame)」をつくる


  ### CSS tips
  - **`box-shadow: ;`**
    - 値が２つだけのとき: `<offset-x>`および`<offset-y>`として解釈される。
    - ３つ目の値: `<blur-radius>`*(影の輪郭をぼかす)*として解釈される。
    - ４つ目の値: `<spread-radius>`*(影の広がりを指定する)*として解釈さる。
    - 任意で`inset`キーワード
    - 任意で`<color>` の値


  ### 実装について

  #### 今日のふりかえり

  - 今日はたっぷり時間があったが、集中力が散漫だった。
  - メソッドを仮置きしながら実装を進めるプロセスがスラスラ出来たら楽しいだろうなぁと思う。そのためにはアタマの中で全体像を描いておく必要があるだろう。
  - つづきは明日。

  #### **`Class構文`**についての覚書

    - `constructor()`: プロパティを初期化するための特殊なメソッド
    - `this`: そのクラスからつくられるインスタンスを、そのクラス内で表現するときのキーワード
    - **`カプセル化`**: プロパティを直接操作せず、メソッドを介して操作しようとする考え方
    - `parseInt(string, radix)`: 文字列の引数`string`を、指定された基数`radix` *(`10`ならば10進法)*に変換して返す ➡️ [MDN](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/parseInt)
