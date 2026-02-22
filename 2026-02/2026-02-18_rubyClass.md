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

### 範囲オブジェクト

```ruby
# リテラル
  >> r = (1..10)
  => 1..10

# コンストラクタ
  >> ra = Range.new(1, 10)
  => 1..10

# == で確認
  >> r == ra
  => true
```

  👉 *`Range` クラスのコンストラクタでは、2つの引数を渡す*

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

## クラスメソッド と インスタンスメソッド

### クラスメソッド

- `String.new` のようにクラスが直接呼び出すメソッド
  👉 *「Rubyリファレンスマニュアル」では、`String.new` のように `.` で表記される*

### インスタンスメソッド

- `"foobar".length` のようにインスタンスが呼び出すメソッド
  👉 *「Rubyリファレンスマニュアル」では、`String#length` のように `#` と表記される*

---

## クラスの継承

👉 *クラスの機能（メソッドなど）を受け継いで別のクラスを作ることができる*

- **継承元**となったクラス ： **親クラス**（`superclass`）
- **継承先**となったクラス ： **子クラス**（`subclass`）

### 継承の階層

```ruby
  >> s = String.new("foobar")
  => "foobar"

# 変数sのクラスを調べる
  >> s.class
  => String

# Stringクラスの親クラスを調べる
  >> s.class.superclass
  => Object

# Stringクラスの親クラスの親クラスを調べる
  >> s.class.superclass.superclass
  => BasicObject

  >> s.class.superclass.superclass.superclass
  => nil
```

👇 *このときのクラスの継承階層*

  ```text
    BasicObject
    ↑
    Object
    ↑
    String
  ```

- Ruby では、全てのオブジェクトは `BasicObject` クラスを継承している
*= "Rubyではあらゆるものがオブジェクトである")*

### `self` キーワード

👉 *Rubyでは、`self` キーワードを使うと自分自身を指定できる*

```ruby
# Stringクラスを継承するWordクラスを定義
  >> class Word < String
# 文字列が回文であればtrueを返す
  >>   def palindrome?
  >>     self == self.reverse        # selfは文字列自身を表す
  >>   end
  >> end
  => :palindrome?
```

- 上の場合、`self` は `Word` クラスの中で**オブジェクト自身**を指す

  ```ruby
  # palindrome? メソッドの定義
    self == self.reverse
  ```

- `String` クラスの内部では、メソッドや属性を呼び出すときの `self.` を省略可能

  ```ruby
  # palindrome? メソッドをこう書ける
    self == reverse
  ```

  👇

  ```ruby
  >> s = Word.new("level")
  => "level"
  >> s.palindrome?
  => true
  ```

---

※ 以下編集中

---

## 1️⃣ 何も書かないクラス

```ruby
class A
end
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
