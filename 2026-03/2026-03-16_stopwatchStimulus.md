# [卒制](https://github.com/QynToKey/HowLongWillItLast) (day 18)：ストップウォッチ機能の実装：Stimulus (3/3)

## Stimulus とは

- Rails 7 以降ではデフォルトで導入されている HTML 主導の軽量 JS フレームワーク
- HTML に属性を追加するだけで JavaScript の動きを加えられる
- DOM 中心のシンプルな設計
- Turbo との親和性が高く、組み合わせることで部分的なページ更新が可能

### Stimulus の実装イメージ

```markdown
1. HTMLに `data-controller="stopwatch"` を書く
  ⬇️
2. Stimulusが自動的に `stopwatch_controller.js` を見つける
  ⬇️
3. `stopwatch_controller.js` の処理が動く
```
