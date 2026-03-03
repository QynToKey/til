# Render + PostgreSQL + SECRET_KEY_BASE 設定手順

## 0️⃣ 前提

- Rails アプリを Docker でデプロイ
- Render Web Service を使用
- 本番 DB は Render PostgreSQL ⬅️ 未設定❗️

---

## 1️⃣ Render で PostgreSQL を作成

1. Render ダッシュボード
2. 「New +」→ **PostgreSQL**
3. 必要項目を入力（Name / Region など）
4. Status が `Available` になるまで待つ

---

## 2️⃣ DATABASE_URL の設定

### 2-1. `Internal URL` を使用

- PostgreSQL の「Connect」から `Internal Database URL` をコピー

例：

```bash
postgresql://user:password@dpg-xxxx/how_long_will_it_last_db
```

⬇️

### 2-2. Web Service 側に設定

1. 「Web Service」>「Settings」>「Environment」> `Add Environment Variable`

```bash
Key: DATABASE_URL
Value: （Internal URL）
```

⬇️

保存後、再デプロイ

---

## 3️⃣ SECRET_KEY_BASE の設定

### 3-1. ローカルで生成

```bash
docker compose web bin/rails secret
```

⬇️

出力された長い文字列をコピーして、「Environment Variables」 に追加

```bash
Key: SECRET_KEY_BASE
Value: （生成した文字列）
```

⬇️

保存 → 再デプロイ

---

## 4️⃣ Render の build 時にエラー発生 ⚠️

### `assets:precompile` のエラー

```bash
# build 時のログより
ActiveRecord::AdapterNotSpecified:
database configuration does not specify adapter
```

👉 *原因： build 時に参照する `DATABASE_URL` が存在しなかったため、 production 環境で DB 設定が完了できなかった (接続エラーではない)*

---

### `assets:precompile` のエラーの対応

Render のログに `Dockerfile` のコードが行番号付きで表示されていたので、そこを見る。

#### 1. ローカルで `Dockerfile` を修正

```dockerfile
# デフォルトの設定
RUN SECRET_KEY_BASE_DUMMY=1 ./bin/rails assets:precompile

# 以下に更新
RUN SECRET_KEY_BASE_DUMMY=1 \
    RAILS_ENV=production \
    DATABASE_URL=postgresql://dummy:dummy@localhost:5432/dummy \
    ./bin/rails assets:precompile
```

📝 ポイント：
    - `assets:precompile` は DB に接続しない
    - adapter が解決できれば OK
    - 本番では、 runtime 時に本物の ENV が使われる

<!-- build 時だけダミー `DATABASE_URL` を渡す（実際の本番 DB は実行時の `ENV` で上書きされる）* -->

---

### （代替案）

```bash
production:
  url: <%= ENV.fetch("DATABASE_URL", "postgresql://dummy:dummy@localhost/dummy") %>
```

👉 *`database.yml` に default を持たせる (Dockerfile を汚さずに解決可能)*

---

## 5️⃣ デプロイ確認

本番 URL で確認：

☑️ ユーザー登録
☑️ ログイン
☑️ ログアウト

---

## まとめ

- これまでは静的ページをデプロイするだけだったので**DB 設定**が不要に見えたが、Rails は production 起動時および `assets:precompile` 実行時に DB 設定を解決するため、**DB を使わない処理でも設定は必要**
    👉 *(DB 設定が解決できないと ActiveRecord が起動できない)*

- Render は build 時と runtime で**環境変数**の扱いが異なる
    👉 *(`assets:precompile` は production 環境で実行される)*

---
