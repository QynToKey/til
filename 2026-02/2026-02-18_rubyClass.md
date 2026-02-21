# Ruby の クラス と 継承チェーン

## コンストラクタ

- Ruby のクラスはインスタンスから作成される
- クラスからインスタンスを生成する処理 = **コンストラクタ**

### 文字列クラスのインスタンスを生成する

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

  👉 *正確には、`String.new` を実行する際に呼び出される初期化メソッド`initialize`がコンストラクタに相当する*

### 「配列」の場合

- 配列のコンストラクタ（`Array.new`）は配列の初期値を引数に取る

```ruby
# 配列も同様（省略せずに配列クラスからインスタンスを生成）
  >> a = Array.new([1, 3, 2])
  => [1, 3, 2]

# new メソッドを省略して、配列クラスからインスタンスを生成
  >> b = [1, 3, 2]
  => [1, 3, 2]

# 生成した配列オブジェクト同士を == メソッドで比較
  >> a == b
  => true
```

### 「ハッシュ」の場合

- ハッシュのコンストラクタ（`Hash.new`）はハッシュのキーが存在しなかったときのデフォルト値を引数に取る

```ruby
# コンストラクタの引数は空（デフォルト値）のままにする
  >> h1 = Hash.new()
  => {}

# 存在しないキー「:foo」の値にアクセスしてみると..
  >> h1[:foo]
  => nil

# コンストラクタの引数に「0」を渡して、存在しないキーの値にアクセスしてみると..
  => {}
  >> h2 = Hash.new(0)
  >> h2[:foo]
  => 0
```

  👉 *ハッシュのコンストラクタ（`Hash.new`）に渡す引数が、存在しないキーを参照したときのデフォルト値となる*

---

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
