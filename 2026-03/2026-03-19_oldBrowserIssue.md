# [卒制](https://github.com/QynToKey/HowLongWillItLast) (MVP 後)：古い iOS Safari でアクセスできない問題を修正

## 概要

以下のフィードバックを受け、対応した。

- 機種 ： `iPhone 8`
- ブラウザ ： `Safari iOS 16.7.15`
- エラーメッセージ ： 「ブラウザがサポートできず、アップグレードが必要」

---

### 修正内容

`ApplicationController` から `allow_browser versions: :modern` を削除

### 原因特定

👉 *エラーメッセージの出所から判断した。*

```bash
$ grep -r "サポート\|support\|upgrade\|アップグレード\|browser" app/ --include="*.js" --include="*.erb" --include="*.rb" -l
app/controllers/application_controller.rb
```

- `ApplicationController` に `allow_browser versions: :modern` が設定されており、`iOS 16` の Safari がモダンブラウザの条件を満たせず Rails がアクセスを拒否していた。
- なお本設定は `rails new` で自動生成されるボイラープレートであり、このアプリの機能には不要なものだった。

---

### 確認

本修正をデプロイした後、当該ユーザーにアクセスを確認してもらった。

- [x] `iPhone 8` / `Safari iOS 16.7.15` からアクセスできる
