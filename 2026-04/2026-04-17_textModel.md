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

## 4️⃣
