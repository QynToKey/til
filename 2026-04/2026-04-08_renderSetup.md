# [co-READER](https://github.com/QynToKey/co_reader)（day: 2_2）： Render × NEON へのデプロイ設定

## やったこと全体像

co-reader を Render（ホスティング）+ NEON（PostgreSQL）構成でデプロイするための設定を行った。3回のPRに分けて解決した。

---

## PR #35：Render デプロイ設定の追加

### 変更内容

> `render.yaml`（新規）

- Render Web Service の定義ファイル。Docker ランタイム・Free プラン。
- `/up` をヘルスチェックエンドポイントに指定。
- 環境変数 `DATABASE_URL` と `RAILS_MASTER_KEY` はダッシュボードで手動設定。
- デプロイのたびに `bin/rails db:prepare` を自動実行するよう設定。

> `db:prepare` とは

Render のデプロイコマンドとして指定した Rails のコマンド。状況に応じて自動で判断してくれる：

- `schema.rb` なし → `db:migrate` を実行（初回デプロイ）
- `schema.rb` あり → `schema:load` で高速セットアップ
- DB が既に最新 → 何もしない

何度実行しても安全で、デプロイコマンドとして使いやすい。`db:create` も内包している。  
**PR #35〜#37 を通じて `db:prepare` 自体は成功していた。** 問題は接続名の欠落やポートのズレという別レイヤーで発生していた。

> `config/database.yml`

- Rails 8 デフォルトの「cache / queue / cable 用に別 DB を用意する」構成を、NEON（単一 DB）に合わせて統合。
- `production` を `DATABASE_URL` 環境変数のみで接続する形に変更。

> `config/environments/production.rb`

- Render は SSL を終端するリバースプロキシ経由でアクセスされるため、`config.assume_ssl = true` を有効化。

> Render ダッシュボードの手動設定（Environment タブ）

| キー | 値 |
| --- | --- |
| DATABASE_URL | NEON の接続文字列（`?sslmode=require` 付き） |
| RAILS_MASTER_KEY | `config/master.key` の内容 |

---

## PR #36：Solid Queue/Cache/Cable の接続名を復元

### 問題

PR #35 で `database.yml` を単一DB構成に変更したことで、`Solid Queue` / `Cache` / `Cable` が参照する接続名（`queue` / `cache` / `cable`）が消えてしまい、起動時にエラーが発生した。

### 原因

Solid Queue は内部で `connects_to database: { writing: :queue }` を持っており、`database.yml` の production セクションに `queue` という接続定義がないと起動できない。Solid Cache・Solid Cable も同様。

### 解決策

接続名（primary / cache / queue / cable）を残したまま、全て同じ `DATABASE_URL` を向けることで単一の NEON DB に集約した。

---

## render_issue ブランチ：502 エラーの解消

### 問題

PR #36 マージ・再デプロイ後、`/up` が `HTTP/2 502` を返していたが、

Render のログを確認すると Puma は正常に起動していた：

```bash
[79] * Listening on http://0.0.0.0:3000
[79] - Worker 0 (PID: 89) booted in 0.09s, phase: 0
[79] - Worker 1 (PID: 98) booted in 0.0s, phase: 0
```

### 原因

Render はデフォルトで port `10000` にリクエストをルーティングするが、Puma は `3000` で起動していたためリクエストがアプリに届いていなかった。

### 解決策

`render.yaml` に `port: 3000` を追記した。

```yaml
port: 3000
```

再デプロイ後、`/up` が `HTTP/2 200` を返すことを確認。

---

## 学んだこと

- `curl -I <URL>` で HTTP レスポンスのヘッダーだけ確認できる（`-I` は headers only）
- Render は `render.yaml` で `port` を指定しないとデフォルト `10000` を使う
- アプリのログが正常でも 502 になる場合はポートの不一致を疑う
- Rails 8 の Solid Queue/Cache/Cable は `database.yml` に対応する接続名が必要
- NEON（単一 DB）に全接続名を向けることで Free プランの制約をクリアできる
- `db:prepare` はデプロイコマンドとして安全に使える。問題が起きても「`db:prepare` が失敗した」のか「アプリ起動時に失敗した」のかをログで切り分けることが大事

---

### 総学習時間： 1195.1 時間
