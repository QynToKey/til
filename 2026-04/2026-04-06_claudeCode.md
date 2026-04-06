# [卒制](https://github.com/QynToKey/HowLongWillItLast) (MVP 後)： Claude Code を使ってリファクタリングしてみる

## 0️⃣ Claude Code の設定

当面は学習目的で運用したいため、
Claude Code をインストールする前に `settings.json` で `deny` （禁止）ルールを設定する

👉 *allow（自動許可）・ask（確認する）・deny（禁止）の3種類のルールがあり、評価順は deny → ask → allow で、最初にマッチしたルールが優先される*

👉 *ひとまず Git 操作を禁止に設定した*

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
