# [卒制](https://github.com/QynToKey/HowLongWillItLast) (MVP 後)： 複数の `LearningThemes` に対応 (2/2)

## 0️⃣ やること

1. `users/edit` のフォームを配列形式に変更
2. `UsersController#update` を複数テーマ対応に変更
3. `users/show` の削除ボタンのコメントアウトを外す
4. `learning_records/index` のタグフィルタリングをテーマ別に対応

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
```
