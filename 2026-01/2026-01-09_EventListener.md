- サロンHPのバグ修正など

  - Safari（特に iOS）では、非インタラクティブ要素を「ユーザー操作イベントの起点」にするとイベントが発火しないことがある。
    - iOS Safari が「タップ対象」として扱う要素
      - `a`
      - `button`
      - `input`
      - `label`
      - 明示的に `role` `tabindex` が付いた要素

  - `querySelector` はDOMを取得するだけなので安全
  
  - `addEventListener` は「どの要素をイベントの起点にするか」に注意が必要
