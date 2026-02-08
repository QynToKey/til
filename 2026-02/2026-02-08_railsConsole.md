
### Rails コンソールでアプリケーションの状態を調べてみたら `NoMethodError` が出た

#### 行ったこと

  1. `rails g`コマンドで`User モデル` と `Micropost モデル` を生成
  2. `rails db:migrate` を実行
  3. **アソシエーション** および **バリデーション** を設定
    ```ruby
    class User < ApplicationRecord
      has_many :microposts
    end
    ```
    ```ruby
    class Micropost < ApplicationRecord
      belongs_to :user
      validates :content, length: { maximum: 140 }
    end
    ```
  4. ブラウザからテストデータを作成
  5. `rails console` で実装状況を確認
    - 成功例
    ```bash
    first_user = User.first
    first_user.microposts # => Micropost の配列が返る
    ```

#### 起きたこと
  - Rails コンソールで `NoMethodError` が出た
    ```bash
    irb(main):009> first_user.microposts
    /usr/local/rvm/gems/default/gems/activemodel-7.0.4.3/lib/active_model/attribute_methods.rb:458:in `method_missing': undefined method `microposts' for #<User id: 2, name: "さすけちゃん", email: "qyntokey@icloud.com", created_at: "2026-02-07 02:27:04.038738000 +0000", updated_at: "2026-02-07 02:27:04.038738000 +0000"> (NoMethodError)
    Did you mean?  maicroposts # 👈 ココがヒント
    ```

  - デバッグのために行ったこと
    1. コンソールのコマンドを確認
      - `Active Record` コマンドのタイポを疑った ← これは⭕️
    2. ターミナルのログを確認
      - `rails g` コマンドのタイポを疑った ← これも⭕️
    3. アソシエーションの設定を確認
      - User の `has_many` を `:maicroposts` とタイポしていた ← 今回はコレが原因❌
    <br>👉 *修正後は`Rails コンソール`の再起動が必要❗️*

#### 学び
  - `has_many` のシンボル名を1文字でも間違えると、その名前の関連メソッドが生える
    - 依存度の強い順
    ```text
    generator → DB → association → console
    ```
  - コードを修正したら「Rails コンソール」は要再起動
  - NoMethodError の `Did you mean?` は最優先で確認する
