# [co-READER](https://github.com/QynToKey/co_reader)（day: 22_1）： 書き込みモデル設計

---

## 0️⃣ 実装方針

- コア機能の土台となる `Annotation` モデルを設計・実装する。
- ER 図では `text_id` が抜けていたため、今回の実装に合わせて ER 図も修正する。
- 合わせて `texts/show.html.erb` のハードコード文字列を i18n 化する。

---

## 1️⃣ ER 図の修正

👉 *`text_id` を追加*

```mermaid
erDiagram

    USERS {
        bigint id PK
        string name "NOT NULL"
        string email "NOT NULL, UNIQUE"
        string crypted_password "NOT NULL"
        string salt "NOT NULL"
        string role "NOT NULL"
        datetime created_at
        datetime updated_at
    }

    TEXTS {
        bigint id PK
        bigint uploaded_by FK "NOT NULL"
        string title "NOT NULL"
        text body "NOT NULL"
        datetime created_at
        datetime updated_at
    }

    READING_SESSIONS {
        bigint id PK
        bigint created_by FK "NOT NULL"
        string name
        string invite_token "NOT NULL, UNIQUE"
        boolean enforce_order "NOT NULL, DEFAULT false"
        datetime created_at
        datetime updated_at
    }

    READING_SESSION_TEXTS {
        bigint id PK
        bigint reading_session_id FK "NOT NULL"
        bigint text_id FK "NOT NULL"
        integer position
        datetime created_at
        datetime updated_at
    }

    MEMBERSHIPS {
        bigint id PK
        bigint user_id FK "NOT NULL"
        bigint reading_session_id FK "NOT NULL, UNIQUE (user_id, reading_session_id)"
        string mode "NOT NULL"
        integer reading_progress
        boolean completed "NOT NULL, DEFAULT false"
        datetime joined_at "NOT NULL"
    }

    SESSION_FAVORITES {
        bigint id PK
        bigint membership_id FK "NOT NULL"
        bigint favorite_user_id FK "NOT NULL"
        datetime created_at
    }

    TEXT_READINGS {
        bigint id PK
        bigint membership_id FK "NOT NULL"
        bigint text_id FK "NOT NULL"
        datetime read_at "NOT NULL"
    }

    ANNOTATIONS {
        bigint id PK
        bigint user_id FK "NOT NULL"
        bigint reading_session_id FK "NOT NULL"
        bigint text_id FK "NOT NULL"
        string annotation_type "NOT NULL"
        integer start_position "NOT NULL"
        integer end_position "NOT NULL"
        string color
        text body
        datetime created_at
        datetime updated_at
    }

    REPLIES {
        bigint id PK
        bigint annotation_id FK "NOT NULL"
        bigint user_id FK "NOT NULL"
        text body "NOT NULL"
        datetime created_at
        datetime updated_at
    }

    USERS ||--o{ TEXTS : uploads
    USERS ||--o{ READING_SESSIONS : creates
    TEXTS ||--o{ READING_SESSION_TEXTS : has
    READING_SESSIONS ||--o{ READING_SESSION_TEXTS : has
    USERS ||--o{ MEMBERSHIPS : joins
    READING_SESSIONS ||--o{ MEMBERSHIPS : has
    MEMBERSHIPS ||--o{ SESSION_FAVORITES : has
    USERS ||--o{ SESSION_FAVORITES : favorited_by
    USERS ||--o{ ANNOTATIONS : writes
    READING_SESSIONS ||--o{ ANNOTATIONS : has
    TEXTS ||--o{ ANNOTATIONS : has
    ANNOTATIONS ||--o{ REPLIES : has
    USERS ||--o{ REPLIES : writes
    MEMBERSHIPS ||--o{ TEXT_READINGS : has
    TEXTS ||--o{ TEXT_READINGS : has
```

---

## 2️⃣ マイグレーション

👉 *`annotations` テーブルを作成

```bash
$ docker compose exec web bin/rails g migration CreateAnnotations
      invoke  active_record
      create    db/migrate/20260429004313_create_annotations.rb
```

  ⬇️

```ruby
# db/migrate/20260429004313_create_annotations.rb
class CreateAnnotations < ActiveRecord::Migration[8.0]
  def change
    create_table :annotations do |t|
      t.references :user,            null: false, foreign_key: true
      t.references :reading_session, null: false, foreign_key: true
      t.references :text,            null: false, foreign_key: true
      t.integer    :annotation_type, null: false
      t.integer    :start_position,  null: false
      t.integer    :end_position,    null: false
      t.string     :color
      t.text       :body

      t.timestamps
    end
  end
end
```

  ⬇️

```bash
$ docker compose exec web bin/rails db:migrate
== 20260429004313 CreateAnnotations: migrating ================================
-- create_table(:annotations)
   -> 0.0458s
== 20260429004313 CreateAnnotations: migrated (0.0459s) =======================
```

---

## 3️⃣ `Annotation` モデルを作成

```bash
touch app/models/annotation.rb
```

```ruby
# app/models/annotation.rb
class Annotation < ApplicationRecord
  belongs_to :user
  belongs_to :reading_session
  belongs_to :text

  enum :annotation_type, { highlight: 0, underline: 1, comment: 2 }

  validates :annotation_type, presence: true
  validates :start_position,  presence: true
  validates :end_position,    presence: true
end
```

---

## 4️⃣ 関連モデルにアソシエーション追加

### `User` モデル

```ruby
# app/models/user.rb
has_many :annotations, dependent: :destroy
```

### `ReadingSession` モデル

```ruby
# app/models/reading_session.rb
has_many :annotations, dependent: :destroy
```

### `Text` モデル

```ruby
# app/models/text.rb
has_many :annotations, dependent: :destroy
```

---

## 5️⃣ `ja.yml` の修正

```ruby
# config/locales/views/ja.yml
+  texts:
+    show:
+      completed: "読了"
+      mark_as_read: "読了にする"
  profiles:
    show:
-      not_completed: "読了にする"
```

---

## 6️⃣ 読了ボタンを i18n 化

```erb
<%# app/views/texts/show.html.erb %>
      <span class="btn btn-success disabled"><i class="bi bi-check-lg"></i> <%= t("texts.show.completed") %></span>
    <% else %>
      <%= button_to t("texts.show.mark_as_read"), reading_session_text_text_reading_path(@reading_session, @text), class: "btn btn-outline-success" %>
```

---

### 動作確認

- [x] `docker compose exec web rails db:migrate` が正常終了する
- [x] `docker compose exec web rails console` で以下を実行：

```bash
Annotation.new(annotation_type: :highlight).annotation_type
# => "highlight"
```

- [x] ブラウザでテキスト詳細を開き「読了にする」ボタンが正常に表示・動作する

---

#### 総学習時間： 1272.9 時間
