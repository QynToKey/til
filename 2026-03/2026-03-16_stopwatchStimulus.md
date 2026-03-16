# [卒制](https://github.com/QynToKey/HowLongWillItLast) (day 18)：ストップウォッチ機能の実装：Stimulus (3/3)

## Stimulus とは

- HTML に属性を追加するだけで JavaScript の動きを加えられる
- DOM 中心のシンプルな設計
- Turbo と組み合わせることで、部分的なページ更新が可能
- Rails 7 以降ではデフォルトで導入済み

### Stimulus の実装イメージ

```markdown
1. HTMLに `data-controller="stopwatch"` を書く
  ⬇️
2. Stimulusが自動的に `stopwatch_controller.js` を見つける
  ⬇️
3. `stopwatch_controller.js` の処理が動く
```
