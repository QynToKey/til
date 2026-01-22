## JavaScriptで「[カウントダウンタイマー](https://github.com/QynToKey/MyCountdownApp)」を試作

### 実装の流れ

  1. 終了時刻を求める
      ```
      # クリックしたら「現在時刻」を取得し、経過ミリ秒に変換し、終了までのミリ秒を加算
        btn.addEventListener('click', () => {
          const endTime = new Date().getTime() + 3 *1000;
        });
      ```

      - `new Date()`: 現在の日時についてのデータを作成
      - `toLocaleString()`: 日時データを読みやすい形式に変換
      - `getFullYear()`: 年を取得
      - `getMonth()`: 月を取得
      - `getDate()`: 日を取得
      - `getHours()`: 時間を取得
      - `getMinutes()`: 分を取得
      - `getSeconds()`: 秒を取得
      - `getMilliseconds()`: ミリ秒を取得
      - `getTime()`: `1970/01/01`からの経過ミリ秒を取得(`Unix Timestamp`)


  2. 残り時間を求める
      ```
      # 残り時間 = 終了時刻 - 現在時刻
        function check() {
          const countdown = endTime - new Date().getTime();
        }

      # 0.1秒毎に呼び出す
        setInterval(check, 100);
      ```


  3. タイマーを終了させる
      ```
      # タイマーのID( serInterval() の返り値 )を取得して渡す
        if (countdown < 3) {
          clearInterval(setIntervalID);
        }
      ```


- タイマーの表示を整形する

    ```
    # 「ミリ秒」を「分」と「秒」に変換
      const totalSeconds = Math.floor(countdown / 1000);
      const minutes = Math.floor(totalSeconds / 60);
      const seconds = totalSeconds % 60;

    # 「二桁」で表示
      const minutesFormated = String(minutes).padStart(2, '0');
      const secondsFormated = String(seconds).padStart(2, '0');

    # テンプレートリテラルで要素へ代入
      timer.textContent = `${minutesFormated}:${secondsFormated}`;
    ```
