# Ruby の クラス と 継承

## コンストラクタ

- Ruby のオブジェクト（インスタンス）はクラスから作成される
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

  👉 *Rubyでは、`new` メソッドがオブジェクトを生成し、その内部で `initialize` が呼ばれて初期化が行われる。*

### 「配列」の場合

- 配列のコンストラクタ（`Array.new`）は、引数に配列を渡すことで新しい配列を生成する

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

⚠️ *ただし、引数の与え方によっては「配列のサイズ」を指定する*

```ruby
# 第1引数は「配列のサイズ」を指定する
  >> Array.new(3)        # => [nil, nil, nil]

# 第2引数は「各要素の初期値」を指定する
  >> Array.new(3, 0)     # => [0, 0, 0]

# 引数に配列を渡す
  >> Array.new([1,2,3])  # => [1,2,3]
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

- Ruby では、全てのオブジェクトは最終的に `BasicObject` クラスを継承している *(Rubyではあらゆるものがオブジェクトである)*

### クラスを生成するクラス

- `String` クラスの継承構造

```ruby
# 文字列オブジェクトは String クラスのインスタンス
  >> "さすけちゃん".class
  => String

# String クラスは Object クラスを継承している
  > "さすけちゃん".class.superclass
  => Object
```

- 他方、`String` クラスは `Class` クラスのインスタンスでもある

```ruby
# String クラスは Class クラスのインスタンス
  >> "さすけちゃん".class.class
  => Class

# Class クラスは Module クラスを継承している
  >> "さすけちゃん".class.class.superclass
  => Module

# Module クラスは Object クラスを継承している
  >> "さすけちゃん".class.class.superclass.superclass
  => Object
```

  👇

  ```ruby
  # Class クラスの継承階層
  BasicObject
    ↑
  Object
    ↑
  Module
    ↑
  Class
  ```

#### `self` キーワード

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

#### コントローラークラス の継承階層

```ruby
>>controller = StaticPagesController.new
=> #<StaticPagesController:0x000000000df458>

>> controller.class
=> StaticPagesController

>> controller.class.superclass
=> ApplicationController

>> controller.class.superclass.superclass
=> ActionController::Base

>> controller.class.superclass.superclass.superclass
=> ActionController::Metal

>> controller.class.superclass.superclass.superclass.superclass
=> AbstractController::Base

>> controller.class.superclass.superclass.superclass.superclass.superclass
=> Object
```

👇 *ここでコントローラのアクション（メソッド）を呼ぶこともできる*

```ruby
# home アクションを読んでみる
  >> controller.home
  => nil   # メソッドの中身は空なので nil が返る
```

⚠️ *Rails は Ruby で書かれているが、普通の Ruby オブジェクトと同様に振る舞うとは限らない (フレームワークによるメタプログラミングの影響を受けている)*

#### ユーザークラスの継承階層

```ruby
> user = User.new
=>
#<User:0x0000ffff8fb814a8
...
>> user.class
=> User(id: integer, name: string, email: string, created_at: datetime, updated_at: datetime)

>> user.class.superclass
=> ApplicationRecord(abstract)

>> user.class.superclass.superclass
=> ActiveRecord::Base

>> user.class.superclass.superclass.superclass
=> Object
```

---

## 組み込みクラスの変更

👉 *Rubyに組み込まれている基本クラスは拡張可能*

```ruby
# 上で例示した palindrome? メソッドは、「文字列リテラル」に対して直接実行できない
  >> "level".palindrome?
  undefined method `palindrome?' for "level":String (NoMethodError)
```

👇 *継承を使わずに、`String` クラスを拡張する*

```ruby
# Word クラスを生成せず、palindrome?メソッドをStringクラスに直接追加する
  >> class String

# 文字列が回文であればtrueを返す
  >>   def palindrome?
  >>     self == self.reverse
  >>   end
  >> end
  => :palindrome?

# 文字列リテラルに対してメソッドを直接実行できる
  >> "deified".palindrome?
  => true
```

⚠️ *組み込みクラスの変更は強力なテクニックだが、真に正当な理由がない限り、組み込みクラスにメソッドを追加することは無作法とされる*
