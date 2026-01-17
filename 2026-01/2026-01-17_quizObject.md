- 「三択クイズ」アプリの完成

  - 問題文を配列に格納する
    ```
    const quizzez = [
      ['1の正解は？', '選択肢 A', '選択肢 B', '選択肢 C', 0],
      ['2の正解は？', '選択肢 A', '選択肢 B', '選択肢 C', 1],
      ['3の正解は？', '選択肢 A', '選択肢 B', '選択肢 C', 2],
    ];
    ```

  - 出題処理を`forEach()`でまとめる
    ```
    quizzez.forEach((quiz) => {
      render(quiz);
    });
    ```
      └ `render関数`は別途用意する: `createElement()`と`appendChild()`を活用

  - 問題文は**オブジェクト**に格納した方が整理できるのでは？
    ```
    // 問題文をオブジェクトに再編
      const quizzez = [
        {
          question: '1の正解は？',
          choices: ['選択肢 A', '選択肢 B', '選択肢 C'],
          answer: 0
        },
      ];
    ```
      └ 「正解」の値がそのままインデックスとして機能する

    ```
    // li 要素を反復処理で生成
      quiz.choices.forEach((choice, index) => {
        const li = document.createElement('li');
        li.textContent = choice;
        li.addEventListener('click', () => {
          li.classList.add(
            index === quiz.answer ? 'correct' : 'wrong'
          );
        });
        ul.appendChild(li);
      });
    ```
      └ コードが軽量化するうえに可読性や拡張性も高まる
