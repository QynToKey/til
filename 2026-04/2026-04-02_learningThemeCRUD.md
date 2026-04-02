# [卒制](https://github.com/QynToKey/HowLongWillItLast) (MVP 後)： LearningTheme CRUD

## 0️⃣ 実装方針

| やること | 方針 |
| --- | --- |
| テーマの「編集」 | `users/edit` から切り出し、`learning_themes/edit` へ移す |
| テーマの「作成」 | 複数テーマの導線は保留。`users/create` の初回作成はそのまま残す |
| テーマの「一覧」 | `learning_themes/index` を実装（マイページからリンク） |
| テーマの「削除」 | `learning_themes/destroy` を実装 |
| `users/edit` | テーマのフォームを削除し、「テーマを編集」リンクに置き換える |

---
