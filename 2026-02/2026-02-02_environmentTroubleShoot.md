## Railsアプリをクローン後、`Dev Container`の環境構築が正常に行われなかった

### 想定される理由と対処法：

  - `.devcontainer` が壊れたりバージョン差異で動かなくなるケースはよくある
  -  その際は`devcontainer.json`を「最小構成で確認 → 復旧」

### トラブルシュートの手順

- コンテナが起動しない場合の確認点：

  ```bash
  # devcontainer.json が存在するか
    ls -la .devcontainer/

  # Docker イメージがあるか
    docker images | grep ruby

  # コンテナが残っていないか
    docker ps -a
  ```

  👉 *VS Code が古いコンテナを参照している場合は `Rebuild` や `Reopen in Container` を試す*

### 復旧手順：

  1. 別フォルダで **最小構成 `.devcontainer`** を作ってテスト

  ```bash
  # テスト用フォルダ作成
    mkdir ~/devcontainer-test
    cd ~/devcontainer-test

  # 最小構成の .devcontainer.json を作成
    mkdir .devcontainer
    cat > .devcontainer/devcontainer.json <<EOL
    {
      "name": "railstutorial",
      "image": "mcr.microsoft.com/devcontainers/ruby:3.2.9"
    }
    EOL
  ```

  2. 問題なければ、元のプロジェクトにコピー

  ```bash
    cd ~/toy_app  # 元プロジェクトに移動
    cp -r ~/devcontainer-test/.devcontainer ./
  ```

  3. 不要なバックアップを整理

  ```bash
  # icon.svg などを残したい場合は退避
    mkdir ~/devcontainer-backup
    mv .devcontainer.bak/icon.svg ~/devcontainer-backup/

  # bak フォルダ削除
    rm -rf .devcontainer.bak
  ```

  4. VS Code で **Reopen in Container** → `bundle install` → `bin/rails server -b 0.0.0.0`

  ```bash
  # ターミナルがコンテナ内に入る
    pwd
    # /workspaces/toy_app

  # Gemfile に従ってインストール
    bundle install

  # Rails サーバー起動
    bin/rails server
  ```

  👉 *ブラウザで http://localhost:3000 が開けば 環境構築完了*

---

#### Rails Server の起動コマンドについて

```
$ bin/rails server -b 0.0.0.0
```

- `-b`: <br>Rails サーバーがどの IP アドレスにバインド（待ち受け）するか を指定するオプション
- `0.0.0.0`: <br>**全てのネットワークインターフェース**で待ち受ける *(コンテナ内の Rails サーバーを**ホストや外部からアクセス可能** にする)* <br> *デフォルトでは `127.0.0.1` にバインドされる (コンテナ内からしかアクセスできない)*
