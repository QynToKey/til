# Ruby の継承チェーン

## コンストラクタ

- Ruby のクラスはインスタンスから作成される

```ruby
# ダブルクォートは文字列クラスのインスタンス生成の省略形
  >> s = "foobar"
  => "foobar"

# 生成したインスタンスオブジェクトのクラスをclassメソッドで確認
  >> s.class
  => String
```

```ruby
# 省略せずに文字列クラスからインスタンスを生成
  >> s = String.new("foobar")
  => "foobar"
  >> s.class
  => String
```

  👉  *`new` メソッドなどを省略して直接記述できる値のことを「リテラル」と呼ぶ*

- Ruby のオブジェクトは `BasicObject` を祖先に持つ

## 1️⃣ 何も書かないクラス

```ruby
class A
end
```

このときの「継承ツリー」：

  ```text
  BasicObject
    ↑
    Object
    ↑
    A
  ```

  👉 *デフォルトで `Object` を継承する*

---

## 2️⃣ 上位構造

IRBで確認：

```ruby
Object.superclass
# => BasicObject

Class.superclass
# => Module
```

このときの「継承ツリー」：

  ```text
  BasicObject
    ↑
    Object
    ↑
    Module
    ↑
    Class
  ```

---

## 3️⃣ Class もオブジェクト

`Class` もまた `BasicObject` を祖先に持つオブジェクトである

```ruby
User.class
# => Class
```

- このとき `class` は `Class` のインスタンス
- `new` は `Class` に定義されている
- 定義メソッドは「継承チェーン」を上にたどって探索される（メソッド探索は上から順に見つかった時点で停止する）

```text
特異クラス（eigenclass）
↓
クラス
↓
include された module
↓
親クラス
↓
...
```

---

## 4️⃣ initialize の挙動

- `new` は `Class` に定義されている
- `new` の中で `initialize` が呼ばれる
- `initialize` がなければ祖先を探す

---

## 5️⃣ Railsとの接続イメージ

```ruby
class User < ApplicationRecord
```

このとき `User` は巨大な継承チェーンの一部

  ```text
  User
    ↑
  ApplicationRecord
    ↑
  ActiveRecord::Base
    ↑
  ...
    ↑
  Object
    ↑
  BasicObject
  ```

  👉 *`Rails` はこの「継承構造」を利用して機能を提供している*

---
