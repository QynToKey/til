# [卒制](https://github.com/QynToKey/HowLongWillItLast) (MVP 後)： 複数の `LearningThemes` に対応 (2/2)

## 0️⃣ やること

1. `users/edit` のフォームを配列形式に変更
2. `UsersController#update` を複数テーマ対応に変更
3. マイページ `users/show` の導線を複数テーマ対応に修正
4. TOP ページを複数テーマ対応に修正
5. `learning_records/index` のタグフィルタリングをテーマ別に対応

---

## 1️⃣ `users/edit` のフォームを配列形式に変更

👉 *複数対応のため `text_field_tag :learning_theme_name` を配列形式 `learning_theme_names[]` に統一*

👉 *既存テーマは編集フィールドに表示され、空きスロットは空フィールドとして表示される*

```erb
<#% app/views/users/edit.html.erb %>
    <div class="mb-3">
      <%= label_tag :learning_theme_names, "学習テーマ", class: "form-label" %>
      <% current_user.learning_themes.each do |theme| %>
        <%= text_field_tag "learning_theme_names[]", theme.name, class: "form-control mb-2", placeholder: "※ 学習テーマを入力してください（例：英語、Rails、ピアノ など）" %>
      <% end %>
      <% (3 - current_user.learning_themes.count).times do %>
        <%= text_field_tag :"learning_theme_names[]", nil, class: "form-control mb-2", placeholder: "※ 学習テーマは３つまで登録できます" %>
      <% end %>
    </div>
```

---

## 2️⃣ `UsersController#update` を複数テーマ対応に変更

```ruby
# app/controllers/users_controller.rb
  def update
    @user.assign_attributes(user_params) # 属性を更新するが保存しない

    theme_names = params[:learning_theme_names].reject(&:blank?) # 空の入力を除外してテーマ名を更新

    ActiveRecord::Base.transaction do
      # このブロック内の処理は「全部成功」か「全部失敗」のどちらかになる
      # @user.save! が失敗 → theme.save! は実行されず、@user の変更も取り消される
      # theme.save! が失敗 → @user の変更も取り消される
      # → どちらかが失敗した場合、DBは transaction 実行前の状態に戻る（ロールバック）
      @user.save!

      # 既存テーマを更新 / 新規テーマを追加
      theme_names.each_with_index do |name, i|
        theme = current_user.learning_themes[i] || current_user.learning_themes.build
        theme.name = name
        theme.save!
      end

      # 送信されたテーマ数より多い既存テーマは削除する
      if current_user.learning_themes.size > theme_names.size
        current_user.learning_themes[theme_names.size..].each(&:destroy!)
      end
    end

    redirect_to user_path(current_user), notice: "ユーザー情報を更新しました"
  rescue ActiveRecord::RecordInvalid
    render :edit, status: :unprocessable_entity
  end
```

📝 `(&:destroy!)` ：
`&:` はシンボルをブロックに変換する記法で、
`&:メソッド名` で「各要素に対してそのメソッドを呼ぶ」という意味。

```ruby
# この2つは同じ意味
current_user.learning_themes[theme_names.size..].each(&:destroy!)
current_user.learning_themes[theme_names.size..].each { |theme| theme.destroy! }
```

---

## 3️⃣ マイページ `users/show` の導線を複数テーマ対応に修正

```erb
<div class="container mt-5" style="max-width: 600px;">
  <h1 class="h5 mb-4">マイページ : '<%= current_user.name %>' さん</h1>

  <div class="card mb-3">
    <div class="card-body">
      <h2 class="card-title h6 text-muted">プロフィール</h2>
      ・・・
      </p>
      <p class="mb-3 small ps-3">
        <% if current_user.learning_themes.present? %>
          学習テーマ： <%= current_user.learning_themes.count %>件登録されています (編集する場合は<%= link_to 'こちら', edit_user_path(current_user) %>)
        <% else %>
          学習テーマが登録されていません。
          (登録する場合は<%= link_to 'こちら', edit_user_path(current_user) %>)
        <% end %>
      </p>
    <%= link_to 'プロフィールを編集する', edit_user_path(current_user), class: "btn btn-sm btn-outline-primary" %>
    </div>
  </div>

  ・・・

      <div class="card-footer">
        <%= button_to "このテーマを削除する", learning_theme_path(theme), method: :delete, data: { turbo_confirm: "このテーマを削除すると、関連する記録 / タグ / TODO も全て削除されます。よろしいですか？" }, class: "btn btn-sm btn-outline-danger" %>
      </div>
    </div>
  <% end %>
</div>
```

## 4️⃣ TOP ページを複数テーマ対応に修正

### Home コントローラー `app/controllers/home_controller.rb`

```ruby
class HomeController < ApplicationController
  skip_before_action :require_login, only: [ :index ]

  def index
    # トップページでは、ログインユーザーの学習テーマとその学習時間を表示するためにデータを準備する
    if logged_in?
      @learning_themes = current_user.learning_themes
    end
  end
end
```

### TOP ページ `app/views/home/index.html.erb`

```erb
  <% if logged_in? %>
    <div class="mt-5 mb-4">
      <% @learning_themes.each do |theme| %>
        <p class='small mt-3 mb-1'>
          '<%= theme.name.presence || '未設定' %>' の総学習時間
        </p>
        <p class="h3 mb-2"><%= theme.total_learning_minutes_in_hours %> 時間</p>
        <%= render "shared/progressbar", theme: theme %>
        <hr class="my-3">
      <% end %>
    </div>
```

---

## 5️⃣ `learning_records/index` のタグフィルタリングをテーマ別に対応
