# [卒制](https://github.com/QynToKey/HowLongWillItLast) (day 3)：ユーザー登録機能

##

---

## 覚書

### `rails new` の際に `README.md` が上書きされて初期化してしまった

### 対策 1️⃣

`main` ブランチに存在する `README.md` だけを作業ブランチにコピーする

```bash
git checkout main -- README.md
```

👉 *対象は `README.md `だけ*

### 対策 2️⃣

現在のブランチの最新コミット（`HEAD`）にある `README.md` の状態に戻す

```bash
git restore README.md
```

👉 *これも対象は `README.md `だけ*

### 現実的な運用

1.作業ブランチで `rails new` 実行
2.直後に「上書きされた必要ファイル」だけ戻す

  ```bash
  git status
  ```

3.必要な差分だけをコミット

  ```bash
  git restore README.md
  ```
