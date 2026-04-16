# [co-READER](https://github.com/QynToKey/co_reader)（day: 9）： ユーザー登録フロー

## 0️⃣ 実装方針

管理者が参加者（member）を登録できる仕組みを2方式で構築する。

- 方式A（招待URL）:
 管理者がトークンを発行 → URLをメンバーに共有 → メンバーが自分で登録
- 方式B（直接登録）:
管理者がメールアドレス＋仮パスワードを入力してメンバーを直接作成
- 既存の `AdminRegistrationToken` /  `Admin::RegistrationsController` と同じパターンで実装する。

> 実装フロー

- 方式A（招待URL）
 [管理者] /admin/invitations → 招待URL生成 → URLをコピーして共有
 [参加者] /invite/register?token=xxx → 登録フォーム → 登録完了（member role）

- 方式B（直接登録）
 [管理者] /admin/members/new → メール＋仮パスワードを入力 → 登録完了
 [参加者] /login → 受け取った情報でログイン

---

## 1️⃣

---

## 検証手順

> 方式A（招待URL）

- [ ] 管理者でログイン → /admin/invitations にアクセス
- [ ] 「招待URLを発行」ボタン → 一覧に表示されることを確認
- [ ] URLをコピー → シークレットウィンドウで開く
- [ ] 登録フォームが表示される → 登録後に member ロールでログインを確認
- [ ] トークンが「使用済み」になることを確認
- [ ] 同じURLに再アクセス → リダイレクトされることを確認

> 方式B（直接登録）

- [ ] 管理者でログイン → /admin/members/new にアクセス
- [ ] メール＋パスワードを入力 → 登録
- [ ] シークレットウィンドウでそのメール＋パスワードでログイン → member ロールで入れることを確認
