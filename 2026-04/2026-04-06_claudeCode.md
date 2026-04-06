# [卒制](https://github.com/QynToKey/HowLongWillItLast) (MVP 後)： Claude Code を使ってリファクタリングしてみる

## 0️⃣ Claude Code の設定

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
