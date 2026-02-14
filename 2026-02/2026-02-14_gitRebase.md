# `fetch` と `rebase`

## `git fetch` は何をしているか

```bash
git fetch origin
```

👉 *リモートの**最新状態**を「ローカルのリモート追跡ブランチ」に反映する*

👉 *「作業ブランチ」も「ワーキングツリー」も変わらない*

👉 *自分のコミットを「最新の`origin/main`」に乗せようとするなら、`fetch` して「比較基準」を確認する必要がある*

### ブランチの状態を確認する

```bash
git log --oneline --graph --decorate --all
```

👉 *ここで「ローカル」と「リモート」の位置関係を確認する*


### ブランチが分岐していた場合

```css
A---B---C   ← (origin/main) リモートが先へ進んでいる
     \
      D---E     ← (feature)
```

#### 1️⃣ `rebase` で「履歴」をキレイにする

```bash
git rebase origin/main
```

```css
A---B---C---D'---E'
```

👉 *分岐元を付け替える*

👉 *このとき `D`と `E` は作り直される*

#### 2️⃣ `merge` で履歴が分岐したまま統合する

```bash
git merge origin/main
```

```css
A---B---C
     \    \
      D-----E-----M   ← (feature)
```

👉 *`M` は「マージコミット」で、２つの親を持つ（履歴は直線にならない）*

👉 *すべての履歴を保持したまま、分岐が「合流」される*
