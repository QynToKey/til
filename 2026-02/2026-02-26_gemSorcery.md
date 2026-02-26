# [卒制](https://github.com/QynToKey/HowLongWillItLast) (day 2)：認証基盤構築

## gem Sorcery 導入

- `Gemfile`に `gem 'sorcery', '0.16.3'` を記述
- `docker compose exec web bundle install`
- `docker compose exec web rails g sorcery:install`
- `docker compose exec web rails db:migrate`
- `docker compose restart`

### 📝 `run` と `exec`

#### `docker compose run`

- 新しい一時コンテナを作って実行 *(サーバーが起動していなくても実行可能)*
- ポートやネットワーク設定が異なることがある *(状態が分離される可能性あり)*
- 実行後コンテナは終了する

#### `docker compose exec`

- 今動いているコンテナの中で実行
- 通常の開発ではこちらが正解
- Railsサーバと同じ環境で実行できる

---

## Git tips

### リポジトリをリセットする

| コマンド | 役割り |
| --- | --- |
| `git restore .` | 変更済みファイルを最後のcommit状態に戻す |
| `git clean -fd` | Git管理外ファイルを削除する |
| `git clean -fdn` | 削除対象を確認する |
