# Render + PostgreSQL + SECRET_KEY_BASE 設定手順

## 0️⃣ 前提

* Rails アプリを Docker でデプロイ
* Render Web Service を使用
* 本番 DB は Render PostgreSQL

---

## 1️⃣ Render で PostgreSQL を作成

1. Render ダッシュボード
2. 「New +」→ PostgreSQL
3. 必要項目を入力（Name / Region など）
4. Status が **Available** になるまで待つ

---

## 2️⃣ DATABASE_URL の設定

### 2-1. Internal URL を使用

* PostgreSQL の「Connect」から
* **Internal Database URL** をコピー

例：

```bash
postgresql://user:password@dpg-xxxx/how_long_will_it_last_db
```

---

### 2-2. Web Service 側に設定

1. Web Service → Settings
2. Environment
3. Add Environment Variable

```bash
Key: DATABASE_URL
Value: （Internal URL）
```

保存後、再デプロイ

---

## 3️⃣ SECRET_KEY_BASE の設定

### 3-1. ローカルで生成

```bash
bin/rails secret
```

出力された長い文字列をコピー

---

### 3-2. Render に登録

Environment Variables に追加：

```bash
Key: SECRET_KEY_BASE
Value: （生成した文字列）
```

保存 → 再デプロイ

---

## 4️⃣ assets:precompile 失敗対策（重要）

Docker build 時に発生したエラー：

```bash
ActiveRecord::AdapterNotSpecified:
database configuration does not specify adapter
```

原因：

* build 時には DATABASE_URL が存在しない
* そのため production 環境で DB 設定が解決できない

---

### 4-1. Dockerfile 修正

```dockerfile
RUN SECRET_KEY_BASE_DUMMY=1 \
    RAILS_ENV=production \
    DATABASE_URL=postgresql://dummy:dummy@localhost:5432/dummy \
    ./bin/rails assets:precompile
```

ポイント：

* build 時だけダミー DATABASE_URL を渡す
* 実際の本番 DB は実行時の ENV で上書きされる

---

## 5️⃣ デプロイ確認

本番 URL で確認：

* ユーザー登録
* ログイン
* ログアウト

---

## 6️⃣ 学び

* Render は build 時と runtime で環境変数の扱いが異なる
* assets:precompile は production 環境で実行される
* DB 未設定だと ActiveRecord が起動できない
* Docker build 時にも最低限の DB 設定が必要

---

## 明日やること

* 本当に Dockerfile 修正が最適解か検討
* `config/database.yml` 側で解決できるか検証
* `config.require_master_key` の扱い確認
* ドキュメントとして公開レベルまで整える

---
