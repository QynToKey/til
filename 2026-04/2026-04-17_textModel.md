# [co-READER](https://github.com/QynToKey/co_reader)（day: 10）： Text モデル

## 0️⃣ 実装内容

- 管理者がテキスト（本文）を登録・編集・削除できる機能。
- ER 図に定義された `texts` テーブルを作成し、`admin namespace` に CRUD を実装する。
- テキスト入力方法は「ファイルアップロード」か「直接入力」をフォーム上で選べるようにする。

---

## 1️⃣ `text` テーブルを作成

```bash
$ docker compose exec web bin/rails g migration CreateTexts
      invoke  active_record
      create    db/migrate/20260417060240_create_texts.rb
```

```ruby
# db/migrate/20260417060240_create_texts.rb
class CreateTexts < ActiveRecord::Migration[8.1]
  def change
    create_table :texts do |t|
      t.bigint :uploaded_by, null: false
      t.string :title, null: false
      t.text :body, null: false
      t.timestamps
    end
    # 外部キー制約を追加するために、uploaded_byカラムにインデックスを作成し、usersテーブルのidカラムと関連付ける
    add_index :texts, :uploaded_by
    add_foreign_key :texts, :users, column: :uploaded_by
  end
end
```

```bash
$ docker compose exec web bin/rails db:migrate
== 20260417060240 CreateTexts: migrating ======================================
-- create_table(:texts)
   -> 0.0481s
-- add_index(:texts, :uploaded_by)
   -> 0.0047s
-- add_foreign_key(:texts, :users, {:column=>:uploaded_by})
   -> 0.0100s
== 20260417060240 CreateTexts: migrated (0.0631s) =============================
```

---

## 2️⃣ `Text` モデルを作成

```ruby
class Text < ApplicationRecord
  belongs_to :uploader, class_name: "User", foreign_key: :uploaded_by
  validates :title, presence: true
  validates :body, presence: true
end
```

---

## 3️⃣ `texts` をルーティングに追加

```ruby
# config/routes.rb
  namespace :admin do
    get  "register", to: "registrations#new",    as: :register
    post "register", to: "registrations#create"
    resources :invitations, only: %i[ index create ]
    resources :members, only: %i[ index new create ]
    resources :texts, only: %i[ index show new create edit update destroy ] # ⬅️ 追加
  end
```

---

## 4️⃣ `Admin::TextsController` を作成

```bash
touch app/controllers/admin/texts_controller.rb
```

```ruby
# app/controllers/admin/texts_controller.rb
class Admin::TextsController < ApplicationController
  # 管理者専用のテキスト管理機能を提供するコントローラー。テキストは、ユーザーがアップロードした内容を保存するモデルで、タイトルと本文を持ちます。
  before_action :require_admin
  before_action :set_text, only: %i[ show edit update destroy ]

  def index
    @texts = Text.includes(:uploader).order(created_at: :desc)
  end

  def show
  end

  def new
    @text = Text.new
  end

  def create
    @text = Text.new(text_params)
    @text.uploaded_by = current_user.id
    if params[:file].present?
      # ファイルがアップロードされた場合は、テキストの内容を UTF-8 として読み込む
      @text.body = params[:file].read.force_encoding("UTF-8")
    end
    if @text.save
      redirect_to admin_texts_path, notice: t("admin.texts.create.success")
    else
      render :new, status: :unprocessable_entity
    end
  end

  def edit
  end

  def update
    if params[:file].present?
      # ファイルがアップロードされた場合は、テキストの内容を UTF-8 として読み込む
      @text.body = params[:file].read.force_encoding("UTF-8")
    end
    if @text.update(text_params)
      redirect_to admin_text_path(@text), notice: t("admin.texts.update.success")
    else
      render :edit, status: :unprocessable_entity
    end
  end

  def destroy
    @text.destroy
    redirect_to admin_texts_path, notice: t("admin.texts.destroy.success")
  end

  private

  # コールバックを使用して、共通のセットアップや制約をまとめておく
  def set_text
    @text = Text.find(params[:id])
  end

  # ストロングパラメーターを使用して、許可された属性のみを受け取るようにする
  def text_params
    params.expect(text: [ :title, :body ])
  end
end
```

👉 *`create` / `update` でファイルアップロード処理： `params[:file]` があれば `params[:file].read` を `body` に使用、なければ `text_params[:body]` を使用*

---

## 5️⃣ ビューの作成

| ファイル | 内容 |
| --- | --- |
| `app/views/admin/texts/index.html.erb` | タイトル・登録者・登録日・操作リンク（詳細/編集/削除）のテーブル |
| `app/views/admin/texts/show.html.erb` | タイトル・本文・登録者・日時の表示 |
| `app/views/admin/texts/new.html.erb` | フォームパーシャルを呼ぶだけ |
| `app/views/admin/texts/edit.html.erb` | フォームパーシャルを呼ぶだけ |
| `app/views/admin/texts/_form.html.erb` | タイトル入力、入力方式の選択 UI（※）、エラー表示 |

👉 *ファイルアップロード UI：ラジオボタンで「ファイル」「直接入力」を切り替える。*

---

## 6️⃣ Stimulus コントローラーを作成

👉 *`Javascript で、テキストの入力方法を切り替えるコントローラーを実装*

```bash
touch app/javascript/controllers/text_input_controller.js
```

```js
// app/javascript/controllers/text_input_controller.js
import { Controller } from "@hotwired/stimulus"

// ユーザーは、直接テキストを入力するか、ファイルからテキストをアップロードするかを選択できます。
export default class extends Controller {
  static targets = ["directArea", "fileArea", "radio"]

  toggle() {
    const mode = this.radioTargets.find(r => r.checked).value
    this.directAreaTarget.classList.toggle("d-none", mode !== "direct")
    this.fileAreaTarget.classList.toggle("d-none", mode !== "file")
  }
}
```

> 📝 **Stimulus** について ：

「HTML に書いた属性でJavaScriptの動作を紐付ける」ための軽量フレームワーク。
*（複雑なDOM操作を書かずに済む）*

> コントローラーの登録

- `_form.html.erb` の `form_with` に書いた `data: { controller: "text-input" }` が起点となる。
- この属性があるHTML要素に、`text_input_controller.js` が自動的に紐付く。 *（ファイル名のアンダースコアがハイフンに変換される）*

> `targets` — 操作対象の要素を名前で参照

```js
static targets = ["directArea", "fileArea", "radio"]
```

- `data-text-input-target="directArea"` という属性を持つ要素を `this.directAreaTarget` で取得する。 *（`document.querySelector(...)` を自分で書く代わり）*

 > `toggle()` — ラジオボタンが変わったときの処理

```js
toggle() {
  const mode = this.radioTargets.find(r => r.checked).value
  this.directAreaTarget.classList.toggle("d-none", mode !== "direct")
  this.fileAreaTarget.classList.toggle("d-none", mode !== "file")
}
```

1. チェックされているラジオボタンの `value`（`"direct"` or `"file"`）を取得
2. Bootstrap の `d-none` クラスを付け外しして表示切替

- ラジオボタンに書いた `data: { action: "change->text-input#toggle" }` が「値が変わったら `toggle()` を呼ぶ」という紐付け。

---

## 7️⃣ i18n に `texts` 関連のワードを追加

```ruby
$ cat config/locales/views/ja.yml
ja:
  ・・・
  layouts:
    application:
      ・・・
      texts: "テキスト管理"
  ・・・
  admin:
    ・・・
    texts:
      index:
        title: "テキスト管理"
        new_button: "テキストを登録"
        table_title: "タイトル"
        table_uploaded_by: "登録者"
        table_created_at: "登録日時"
        show: "詳細"
        edit: "編集"
        destroy: "削除"
        confirm_destroy: "本当に削除しますか？"
      show:
        edit: "編集"
        uploaded_by: "登録者"
        back: "一覧に戻る"
      new:
        title: "テキスト登録"
      edit:
        title: "テキスト編集"
      form:
        input_mode: "入力方式"
        mode_direct: "直接入力"
        mode_file: "ファイルアップロード"
        file_label: "テキストファイル（.txt）"
        file_hint: "UTF-8 形式のファイルを選択してください"
        submit: "保存する"
        cancel: "キャンセル"
      create:
        success: "テキストを登録しました"
      update:
        success: "テキストを更新しました"
      destroy:
        success: "テキストを削除しました"
```

---

## 8️⃣
