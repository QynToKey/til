# [co-READER](https://github.com/QynToKey/co_reader)（day: 10）： Text モデル

## 0️⃣ 実装内容

- 管理者がテキスト（本文）を登録・編集・削除できる機能。
- ER 図に定義された `texts` テーブルを作成し、`admin namespace` に CRUD を実装する。
- テキスト入力方法は「ファイルアップロード」か「直接入力」をフォーム上で選べるようにする。

---

## 1️⃣ `text` テーブルを作成
