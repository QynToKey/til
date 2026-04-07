# Docker 前提の Rails プロジェクト立ち上げ

## やったこと

Rails 8.x + PostgreSQL + Docker Compose の構成で、新規 Rails プロジェクト（co-READER）を立ち上げた。合わせて GitHub リポジトリの作成・push、GitHub CLI のセットアップ、issue 設計までを行った。

---

## 手順まとめ

### 1. プロジェクトディレクトリと Docker ファイルを用意する

```
Gemfile          # gem "rails", "~> 8.0" のみ記載
Gemfile.lock     # 空ファイル（bundle install の前に必要）
Dockerfile
compose.yml
entrypoint.sh
.dockerignore
```

Dockerfile では `libyaml-dev` が必要。ないと `psych` gem のビルドが失敗する。

```dockerfile
RUN apt-get install -y --no-install-recommends \
      build-essential \
      libpq-dev \
      libyaml-dev \
      ...
```

### 2. Rails アプリを生成する

```bash
docker compose run --no-deps --rm web rails new . --force --database=postgresql --skip-test
```

- `--no-deps`：依存サービス（db）を起動せずに実行
- `--force`：既存の Gemfile を上書き
- `--database=postgresql`：DB を PostgreSQL に設定
- `--skip-test`：デフォルトの Minitest をスキップ

`rails new .` はカレントディレクトリ（`/app`）に Rails アプリを生成する。ボリュームマウント（`. :/app`）によってホスト側にも反映される。

### 3. DB を作成する

```bash
docker compose run --rm web rails db:create
```

### 4. 起動する

```bash
docker compose up
```

`http://localhost:3000` で Rails のウェルカム画面が表示されれば成功。

---

## ハマったポイント

### `psych` gem のビルドエラー

```
yaml.h not found
```

**原因**：`libyaml-dev` が Dockerfile に含まれていなかった。  
**対処**：apt-get のインストールリストに `libyaml-dev` を追加。

### Rails アプリが生成されていなかった

`docker compose up` で `rails new` のヘルプテキストが表示されてコンテナが終了した。

**原因**：イメージビルドが失敗していた状態で `docker compose up` を実行したため、`rails new .`（Step 1）が実行されておらず、アプリのファイルが存在しなかった。  
**対処**：Dockerfile を修正後、`docker compose build --no-cache` でリビルドし、改めて `rails new .` を実行。

---

## GitHub CLI のセットアップ

```bash
brew install gh
gh auth login --with-token <<< "ghp_xxxx"
gh auth status  # ログイン確認
```

- 対話式の `gh auth login` が効かない場合は `--with-token` でトークンを直接渡す
- トークンは GitHub → Settings → Developer settings → Personal access tokens (classic) で発行
- 必要なスコープ：`repo`、`read:org`

---

## issue 設計

MVP 機能を以下の Epic に分類し、SP（ストーリーポイント）を付けて GitHub issue を作成した。

| Epic | 内容 |
|---|---|
| 基盤整備 | Render デプロイ、CI |
| 認証 | Sorcery、ロール管理、招待URL |
| テキスト管理 | テキストのCRUD・表示 |
| 書き込み機能 | ハイライト・アンダーライン・コメント・Hotwire |
| モード | 孤読/共読の切り替えと制御 |
| その他MVP | 進捗表示、メンション |

SP が 3 を超える issue は Sub-issue に分節した。ラベルは `epic:〇〇` の形式で作成。

```bash
gh label create "epic:認証" --color "e4e669" --repo QynToKey/co_reader
gh issue create --title "タイトル [SP:3]" --body "..." --label "epic:認証"
```
