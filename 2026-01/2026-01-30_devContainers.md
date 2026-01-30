
## Dockerfileを書かずにDocker環境を構築する（Dev Containers編）

### 前提

  - Rails アプリのリポジトリが既に存在する
  - `Gemfile` / `Gemfile.lock` がある
  - Docker Desktop がインストール・起動済み
  - VS Code + Dev Containers 拡張を使用

---

## やったこと

### 1. リポジトリをローカルに clone

  ```bash
  git clone <repository>
  cd <repository>
  ```

- この時点では **まだ Docker は使っていない**
- ただのローカル Rails プロジェクト

---

### 2. VS Code で「Reopen in Container」を実行

- VS Code 左下のステータスバー
- もしくは
  `Cmd + Shift + P` → `Dev Containers: Reopen in Container`

👉 *ここで初めて`Docker`が裏で使われ始める*

---

### 3. `.devcontainer/devcontainer.json` が作成される

今回のポイント：

  ```json
  {
    "image": "mcr.microsoft.com/devcontainers/ruby:3.2"
  }
  ```

  - **Dockerfile は書かない**
  - 既製の Ruby 環境イメージをそのまま使っている
  - VS Code が内部的に：
    - Docker イメージを取得（`docker pull`）
    - コンテナを起動（`docker run`）
    している（自分では叩かない）

---

### 4. VS Code が Docker 操作を「代行」する

この時点の構造はこうなっている：

  ```
  自分
  ↓
  VS Code
  ↓
  Dev Containers 拡張
  ↓
  Docker Engine
  ↓
  Linux + Ruby + Rails 実行環境
  ```

👉 *Dockerコマンドはすべて`VS Code`が管理*

---

### 5. コンテナ内ターミナルで作業開始

ターミナル表示例：

  ```bash
  vscode ➜ /workspaces/hello_app (main) $
  ```

ここで打つコマンドは：

- **ローカル Mac ではなく**
- **Docker コンテナ内の Linux** 上で実行される

---

### 6. bundle install → rails server

  ```bash
  bundle install
  rails server
  ```

- `rails server`: <br>アプリを「起動する」（＝ **環境構築完了の確認**）

- ブラウザで Rails が表示されれば成功 (`http://localhost:3000`)

🤛 *このとき VS Code 内でも小さなブラウザが開くが外部ブラウザを使って問題ない*

---

## 学んだこと

### Dockerfileを書かなくても Docker は使える

####  Dev Containers とは

  - `Dockerfile`を用意しなくても、Docker を **IDE の実装詳細**にする仕組み

  - `Dockerコマンド`を直接叩く必要がない（`vscode`が代行して裏で動いている）


####  Dev Containers の終了と再開

  - 終了時はVS Code を閉じるだけ <br>🤛 *コンテナは自動的に停止する*

  - 再開時は `Reopen in Container` <br>🤛 *`rails server` は毎回必要*
    ```bash
    rails server
    ```

####  Dev Containers 使用時の Git操作

  - Dev Container 内は「まっさらな Linux」状態なので、GitHub からは「別の PC」として扱われる 🤛 *「SSH 鍵なし」の状態*

  - `git push`するために、今回は「**SSH 公開鍵認証を使わない方式**」に切り替えた <br>🤛 *`SSH認証`でも可能だが、初学者にはやや難解*

    - 手順
    ```
    # リモートとの接続状況を確認
      git remote -v

    # SSHで連携されている場合
      origin  git@github.com:ユーザー名/リポジトリ.git (fetch)
      origin  git@github.com:ユーザー名/リポジトリ.git (push)
    ```
    ↓
    ```
    # HTTPS方式に変更
      git remote set-url origin https://github.com/ユーザー名/リポジトリ.git
    ```
    🤛 *`HTTPS` に切り替えることで`git push`できるようになる*


### HTTPS と SSH

- `HTTPS` は ⭕️「**OAuth / Token 認証**」 ❌「認証なし」
- `SSH` は「**SSH 公開鍵認証**」

🤛 *`HTTPS`では`VS Code`が認証を仲介しているため、パスワードを毎回聞かれないだけ*


---

### Dockerfileを書く構成との違い（整理）

| 観点         | Dev Containers | docker-compose |
| ---------- | -------------- | -------------- |
| Dockerfile | 書かない           | 書く             |
| 起動         | VS Code が管理    | 自分で管理          |
| Docker操作   | 不要             | 必須             |
| 学習コスト      | 低い             | 高い             |
| 制御性        | 低め             | 高い             |

🤛 *同じ Docker でも「Dockerfileを書く構成」とは責務分担がまったく異なる*
