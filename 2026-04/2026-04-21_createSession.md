# [co-READER](https://github.com/QynToKey/co_reader)（day: 14）： セッション作成フロー

## 0️⃣ 現状と実装方針

> 現状
`ReadingSessionsController` には `index` / `join` のみで、セッション作成機能が未実装

> 実装方針

- セッション作成は管理者のみ可能
- `reading_session_texts` 中間テーブル（#81）を経由して複数テキストを紐づける
- `InviteRegistrationsController#create` の不備（自動ログインなし）も合わせて修正

---

## 1️⃣ ルーティングに `reading_sessions` を追加

```ruby
# config/routes.rb
   namespace :admin do
     get  "register", to: "registrations#new",    as: :register
     post "register", to: "registrations#create"
     resources :invitations, only: %i[ index create ]
     resources :members, only: %i[ index new create ]
     resources :texts, only: %i[ index show new create edit update destroy ]
+    resources :reading_sessions, only: %i[ index new create ]
   end
```

---

## 2️⃣ `ReadingSession` モデルにバリデーションを追加

```ruby
# app/models/reading_session.rb
   validates :invite_token, presence: true, uniqueness: true
   validates :creator, presence: true
+
+  validate :at_least_one_text

   # セッション作成後に、作成者をセッション管理者として memberships テーブルに自動的に追加するコールバック
   after_create :assign_creator_as_admin
+
+  private
+
+  def at_least_one_text
+    errors.add(:texts, "を1件以上選択してください") if text_ids.blank?
+  end
```

---

## 3️⃣
