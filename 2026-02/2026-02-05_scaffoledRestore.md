- JavaScriptの復習
- figmaの復習（卒制対応）
- Railsの復習
---
## `rails g scaffold` を実行してみた

```bash
rails g scaffold User name:string email:string
```

### 起きたこと

-  controller / model / views / migration / routes / test など
  **大量のファイルが自動生成**された
- 想定より影響範囲が広く、学習目的には過剰だと判断
- `scaffold` 実行前の状態へ戻したくなった

---

### `git restore` の実行

  #### 1. 変更ファイルを元に戻す

  - コミット前だったので、そのまま `restore` コマンドを叩く

  ```bash
  git restore .
  ```

  <br>👉 * `.` は「カレントディレクトリ配下すべて」を意味する (今回は個別にファイル名を書く必要はない)*


  #### 2. 新規生成ファイル・ディレクトリを削除

  - `scaffold` が作ったファイル一式を削除できる

  ```bash
  git clean -fd
  ```

    - `-f` : 強制削除
    - `-d` : ディレクトリも対象


  #### 3. 状態確認

  ```bash
  git status
  ```

  ```text
  nothing to commit, working tree clean
  ```

### 教訓

<br> ⚠️ **`main`で試行錯誤しないこと❗️**

- 実務・検証用途では**お試し用ブランチを切って試す**べし

- もしくは `--pretend` オプションを活用
---
