# ユーザーモデルの初期設定(例)

```ruby
# example_user.rb (例)

class User
  attr_accessor :name, :email

  def initialize(attributes = {})
    @name  = attributes[:name]
    @email = attributes[:email]
  end

  def formatted_email
    "#{@name} <#{@email}>"
  end
end
```

---

## アトリビュート・アクセサー (Attribute Accessor)

👉 *「属性: attribute」に対応する「アクセサー（accessor）」を作成する*

```ruby
attr_accessor :name, :email
```

- アクセサーを作成すると、そのデータ（ここでは `name` と `email`）を取り出すメソッド（`getter`）と、データに代入するメソッド（`setter`）がそれぞれ定義される。

  👉 *インスタンス変数 `@name` とインスタンス変数 `@email` を操作するためのメソッドが用意される*

  - getter : `@name` を「読む」
  - setter : `@name = value` を「書く」

---

## `initialize` メソッド

- `initialize` という名前のインスタンスメソッドを定義することで、これを「コンストラクタ」として扱うことができる
  👉 *`.new` (クラスメソッド)が呼ばれたとき、`new` が生成するインスタンスに対して `initialize` (インスタンスメソッド)が自動的に呼ばれる*

- `initialize` は Ruby の言語仕様に含まれる「特殊メソッド」だが、その中身は 自分で `def` して定義する必要がある

```ruby
def initialize(attributes = {})
  @name  = attributes[:name]
  @email = attributes[:email]
end
```

👇 この場合...

- `initialize` メソッドは、`attributes` という引数を1つ取る
- `attributes` 変数は「空のハッシュをデフォルトの値として持つ」ため、名前やメールアドレスのないユーザーを作ることができる
  👉 *デフォルト値を持たないハッシュでは、存在しないキーを参照すると `nil` が返る*
