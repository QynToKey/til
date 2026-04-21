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

## 3️⃣ `Admin::ReadingSessions` コントローラーを作成

```bash
touch app/controllers/admin/reading_sessions_controller.rb
```

```ruby
# app/controllers/admin/reading_sessions_controller.rb
class Admin::ReadingSessionsController < ApplicationController
  # 管理者専用のコントローラー
  before_action :require_admin

  def index
    @reading_sessions = ReadingSession.includes(:creator, :texts).order(created_at: :desc)
  end

  def new
    @reading_session = ReadingSession.new
    @texts = Text.order(:title)
  end

  def create
    @reading_session = ReadingSession.new(reading_session_params)
    @reading_session.creator = current_user
    # セッションの招待トークンを生成
    @reading_session.invite_token = SecureRandom.urlsafe_base64

    if @reading_session.save
      redirect_to admin_reading_sessions_path, notice: "セッションを作成しました"
    else
      @texts = Text.order(:title)
      render :new, status: :unprocessable_entity
    end
  end

  private

  # ストロングパラメーター: セッション作成に必要なパラメーターを許可する
  def reading_session_params
    params.require(:reading_session).permit(:name, text_ids: [])
  end
end
```

---

## 4️⃣

### i18n を追記

```ruby
# config/locales/views/ja.yml
    reading_sessions:
      index:
        title: "セッション管理"
        new_button: "セッションを作成"
        table_name: "セッション名"
        table_texts_count: "テキスト数"
        table_created_by: "作成者"
        table_created_at: "作成日時"
        table_invite_token: "招待トークン"
        no_name: "（名称なし）"
      new:
        title: "セッションを作成"
      form:
        name_label: "セッション名（任意）"
        texts_label: "テキストを選択（1件以上）"
        submit: "作成する"
        cancel: "キャンセル"
      create:
        success: "セッションを作成しました"
```
