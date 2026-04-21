# [co-READER](https://github.com/QynToKey/co_reader)（day: 14）： セッション作成フロー

## 0️⃣ 現状と実装方針

> 現状
`ReadingSessionsController` には `index` / `join` のみで、セッション作成機能が未実装

> 実装方針

- セッション作成は管理者のみ可能
- `reading_session_texts` 中間テーブル（#81）を経由して複数テキストを紐づける
- `InviteRegistrationsController#create` の不備（自動ログインなし）も合わせて修正

---

## 1️⃣
