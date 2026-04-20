# [co-READER](https://github.com/QynToKey/co_reader)（day: 13）： 複数テキストに対応

## 0️⃣ 現状と実装方針

> 現状

- `reading_sessions.text_id`（単一 FK）で「1 セッション = 1 テキスト」になっている。

> 実装方針

- 1 セッションに複数の `Text` を紐づけるため、`reading_session_texts` 中間テーブルを追加し `text_id` を削除する。
- スキーマ変更により既存コードが 2 箇所で壊れるため、最小限の修正も合わせて行う。

---
