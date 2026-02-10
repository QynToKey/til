### Rails tips

#### Rail で使える短縮形のコマンド例

  | 完全なコマンド | 短縮形 |
  | --- | --- |
  | `$ rails server`	| `$ rails s` |
  | `$ rails console`	| `$ rails c` |
  | `$ rails generate`	| `$ rails g` |
  | `$ rails destroy`	| `$ rails d` |
  | `$ rails test`	| `$ rails t` |
  | `$ bundle install`	| `$ bundle` |

#### 自動生成したファイルを元に戻す

- **生成**`generate` → **取り消し**`destroy`

  ```bash
  # 自動生成コマンド（例）
    $ rails generate controller StaticPages home help

  # 対応する取り消しコマンド
    $ rails destroy  controller StaticPages home help
  ```

  👉 *モデル及びコントローラーについては「引数」を省略できる*

- **マイグレーション**の変更を元に戻す場合

  ```bash
  # マイグレーションの変更
    $ rails db:migrate

  # 1つ前の状態に戻す
    $ rails db:rollback

  # 最初の状態に戻す
    $ rails db:migrate VERSION=0
  ```

  👉 *通常は`VERSION=20220407092210`のようにマイグレーションの作成日時のみ指定可能だが、例外的に`VERSION=0`とシリアル番号で初期化できる*
