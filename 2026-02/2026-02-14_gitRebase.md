# `fetch` と `rebase` はセットで使う

```bash
git fetch origin
git rebase origin/main
```

- `fetch` で `origin/main` を最新に更新
- その最新の先端に `rebase` する

👉 *`git rebase origin/main` は「今いるブランチ」に対して実行される*

  | 操作 | 何が動く？ |
  | --- | --- |
  | `git rebase origin/main` | **今いるブランチ**が動く |
  | `git merge origin/main` | **今いるブランチ**が動く |
  | `git checkout main` | **HEAD** が動く |

⚠️ *`rebase` は履歴を書き換えるため、すでに共有済みのブランチには注意が必要*

- `rebase` は、たんに「既存のコミットを移動」するのではなく、「新しいコミット」を作り直している *（コミットの`SHA`が更新される）*
- そのため、既にリモートへ `push` 済みのブランチを`rebase` した後は `force push` が求められる

## `git fetch` は何をしているか

```bash
git fetch origin
```

👉 *リモートの**最新状態**を「ローカルのリモート追跡ブランチ」に反映する*

👉 *「作業ブランチ」も「ワーキングツリー」も変わらない*

👉 *自分のコミットを「最新の`origin/main`」に乗せようとするなら、`fetch` して「比較基準」を確認する必要がある*

### 「ローカル」と「リモート」の位置関係を確認する

- 「現在チェックアウトしているブランチ」と「対応する origin/ブランチ」を確認する

```bash
git fetch origin
git status
```

👉 *`git fetch`は「ブランチ情報」を取得するが、更新がなければほぼ何も表示されない*

👉 *`git status` は、現在のブランチと対応する「リモート追跡ブランチとの差分」を要約表示する*

↓

```bash
# ブランチが一直線になっている場合の例
Your branch is ahead of 'origin/main' by 1 commit.

# リモートが進んでいる場合
Your branch is behind 'origin/main' by 2 commits.

# ブランチが分岐している場合
have diverged
```

- `git log` で履歴を確認する

```bash
git log --oneline --graph --decorate --all
```

  👉 *`--decorate`: コミットに付いている「参照（ref）」を表示するオプション*

  ```bash
  # --decorate アリのときの表示例
  7e8005c (HEAD -> main, origin/main, origin/HEAD) Merge pull request #3
  8c66206 (origin/pageTransition) docs: delete screenshot
  ```

```bash
# --decorate ナシのときの表示例
7e8005c Merge pull request #3
8c66206 docs: delete screenshot
```

---

## 分岐したブランチを合流させる

### ブランチが分岐している例

```bash
A---B---C   ← (origin/main) # リモートが先へ進んでいる
     \
      D---E     ← (feature) # Bから分岐
```

#### 1️⃣ `rebase` で「履歴」をキレイにする

```bash
git rebase origin/main
```

👉 *現在チェックアウトしているブランチの「分岐点以降のコミット」を、 `origin/main` の最新コミットに付け替える*

```bash
# リベースされたブランチ例
A---B---C---D'---E'
```

👉 *このとき `D`と `E` は作り直される*

⚠️ *ただし `fetch`していないと、ローカルに保存されている「古い `origin/main`」にリベースされる*

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

  | 操作 | 既存コミット | 履歴の連続性 | `force push`？ |
  | --- | --- | --- | --- |
  | `rebase` | 作り直す | 切れる | 必要になる場合あり |
  | `merge` | そのまま | 続いている | 不要 |
