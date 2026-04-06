# [卒制](https://github.com/QynToKey/HowLongWillItLast) (MVP 後)： Claude Code を使ってリファクタリングしてみる

## 0️⃣ Claude Code の設定

当面は学習目的で運用したいため、
Claude Code をインストールする前に `settings.json` でグローバルの `deny` （禁止）ルールを設定する。*（ひとまず Git 操作を禁止に設定）*

👉 *allow（自動許可）・ask（確認する）・deny（禁止）の3種類のルールがあり、評価順は deny → ask → allow で、最初にマッチしたルールが優先される*

```bash
$ cat ~/.claude/settings.json
{
  "permissions": {
    "deny": [
      "Bash(git push *)",
      "Bash(git commit *)",
      "Bash(git merge *)",
      "Bash(git rebase *)",
      "Bash(git reset *)"
    ]
  }
}
```

---

## 1️⃣ 今日やろうとしていること

`learning_records` / `tags` / `todos` テーブルは `learning_theme_id` を持っており、`learning_theme` を経由して `user` を特定できる。現状 `user_id` を重複保持しているため、これを削除して設計を整理する。

- `learning_records` / `tags` / `todos` から `user_id` カラムを削除するマイグレーション作成
- 各モデルの `belongs_to :user` を削除
- `user_id` を参照しているクエリ・コントローラー・ビューを `learning_theme` 経由に修正
- 動作確認

---

## 2️⃣
