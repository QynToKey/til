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

---

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

> 実装方針

- 「学習ログ」 `learning_records/index` は、テーマごとに別々のページを遷移する仕様にする
- `theme_id` と紐づけることで、テーマごとに別々の「学習ログ」を表示させる
- タグ / TODO は `theme_id` に紐づいているため現状のまま

### マイページ `app/views/users/show.html.erb`

```erb
  <% @learning_themes.each do |theme| %>
    <div class="card mb-3">
      <div class="card-body">
       ・・・
        <%= link_to '学習ログ', learning_records_path(theme_id: theme.id), class: "btn btn-sm btn-outline-primary" %>
```

### 学習ログ `app/views/learning_records/index.html.erb`

```erb
  <div>
    <%# テーマごとの総学習時間を表示 %>
    <p class='small mt-3'>総学習時間（<%= @learning_theme.name %>）： <%= (@learning_theme.total_learning_minutes / 60.0).round(1) %> 時間</p>
  </div>
```

### TOP `app/views/home/index.html.erb`

```erb
  <% if logged_in? %>
    <div class="mt-5 mb-4">
      <% @learning_themes.each do |theme| %>
        <p class='small mt-3 mb-1'>
          '<%= theme.name.presence || '未設定' %>' の総学習時間
        </p>
        <p class="h3 mb-2"><%= theme.total_learning_minutes_in_hours %> 時間</p>
        <%= render "shared/progressbar", theme: theme %>
        <div class="d-flex gap-2 mt-1 mb-3">
          <%= link_to '今日の学習を記録する', new_learning_record_path(study_date: Date.current, theme_id: theme.id), class: "btn btn-sm btn-outline-primary" %>
          <%= link_to '学習ログ', learning_records_path(theme_id: theme.id), class: "btn btn-sm btn-outline-secondary" %>
        </div>
        <hr class="my-3">
      <% end %>
    </div>
```

📝 その他、関連する全てのビューの「今日の記録」「学習ログ」ボタンのリンクに `theme_id` を紐付け

- `learning_records/index.html.erb`
- `tags/index.html.erb`
- `users/edit.html.erb`
- `users/show.html.erb`

📝 その他、関連する全てのビューでタグの参照先に `@learning_theme` を指定

- `app/views/learning_records/_form.html.erb`
- `app/views/learning_records/index.html.erb`

### コントローラー `LearningRecordsController`

```ruby
# app/controllers/learning_records_controller.rb
  def index
    # ユーザーが所有する学習記録をタグ情報とともに取得し、学習日が新しい順に並べる
    @learning_records = if params[:theme_id].present?
                          # テーマIDが指定されている場合は、そのテーマに該当する学習記録のみを表示する
                          current_user.learning_records.where(learning_theme_id: @learning_theme.id).includes(:tags).order(study_date: :desc)
                        else
                          # テーマIDが指定されていない場合は、すべての学習記録を表示する
                          current_user.learning_records.includes(:tags).order(study_date: :desc)
                        end
      if params[:tag_id].present?
      # ユーザーが所有するタグの中から、指定されたタグIDに該当するタグを見つける
      @current_tag = current_user.tags.find(params[:tag_id])
      # タグIDが指定されている場合は、そのタグが付いている学習記録のみを表示する
      @learning_records = @learning_records.joins(:tags).where(tags: { id: @current_tag.id })
    elsif params[:date].present?
      # 学習日が指定されている場合は、その日に該当する学習記録のみを表示する
      @learning_records = @learning_records.where(study_date: params[:date])
    end
  end

  def set_learning_theme
    # テーマIDが指定されている場合は、そのテーマをセットする。指定されていない場合は、ユーザーの最初のテーマをセットする。
    @learning_theme = if params[:theme_id].present?
                        current_user.learning_themes.find(params[:theme_id])
                      else
                        current_user&.learning_themes&.first
                      end
  end
```

---

## 6️⃣ ２つ目以降の「学習テーマ」が正常に記録されないバグに対応

- destroy` アクション に `set_learning_theme` を追加し `theme_id` を引き継ぐ
- `learning_records` の各ビューに `theme_id` を引き継ぐリンクを追加

```ruby
class LearningRecordsController < ApplicationController
  skip_before_action :require_login, only: %i[new]
  before_action :set_learning_record, only: %i[show edit update destroy]
  before_action :set_learning_theme, only: %i[index new create edit update destroy] # ⬅️ destroy を追加

  ・・・

  def destroy
    @learning_record.destroy
    redirect_to learning_records_path(theme_id: @learning_theme.id), notice: "学習記録を削除しました"
  end
```

---

### 総学習時間： 1181.3 時間
