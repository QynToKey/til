# [卒制](https://github.com/QynToKey/HowLongWillItLast) (MVP 後)： `Tags` のリファクタリング

## 0️⃣ 前提

`LearningTheme` の実装に伴ってテーブル構造が変更されたため、`tags` を`learning_themes` と紐づける必要が生じた。

### 影響範囲の確認

```bash
$ grep -r "tags_path\|tag_path\|edit_tag_path\|new_tag_path" app/views
app/views/learning_records/index.html.erb:  <%= link_to 'タグ管理', tags_path, class: "btn btn-sm btn-outline-primary" %>
app/views/learning_records/index.html.erb:      <%= link_to 'タグを追加する', new_tag_path, class: "btn btn-sm btn-outline-primary" %>
app/views/learning_records/_form.html.erb:        <p class="text-muted small">※ タグがありません。<%= link_to 'タグ管理', tags_path %>から追加してください。</p>
app/views/learning_records/_form.html.erb:      <%= link_to "タグ管理", tags_path, class: "btn btn-sm btn-outline-primary" %>
app/views/tags/index.html.erb:  <%= link_to '新規タグを追加', new_tag_path, class: "btn btn-sm btn-primary" %>
app/views/tags/index.html.erb:            <%= link_to '編集', edit_tag_path(tag), class: "btn btn-sm btn-outline-secondary" %>
app/views/tags/index.html.erb:            <%= link_to '削除', tag_path(tag), class: "btn btn-sm btn-outline-danger", data: { turbo_method: :delete, turbo_confirm: '本当に削除しますか？' } %>
app/views/tags/edit.html.erb:  <%= render 'form', tag: @tag, return_path: tags_path %>
app/views/tags/new.html.erb:  <%= render 'form', tag: @tag, return_path: tags_path %>
app/views/users/edit.html.erb:    <%= link_to "タグ管理", tags_path, method: :get, class: "btn btn-sm btn-outline-primary" %>
app/views/users/show.html.erb:          <%= link_to 'タグを編集する', tags_path, class: "btn btn-sm btn-outline-primary" %>
app/views/users/show.html.erb:          <%= link_to 'タグを追加する', tags_path, class: "btn btn-sm btn-outline-primary" %>
app/views/users/show.html.erb:    <%= link_to 'タグ管理', tags_path, class: "btn btn-sm btn-outline-primary" %>
```

> ルーティング変更の影響範囲

- `app/views/tags/` 内 → 4ファイル
- `app/views/learning_records/` 内 → 2ファイル
- `app/views/users/` 内 → 2ファイル
- `app/controllers/tags_controller.rb` → リダイレクト先など

> 修正後のパスヘルパーの対応表

| 変更前 | 変更後 |
| --- | --- |
| `tags_path` | `learning_theme_tags_path(@learning_theme)` |
| `new_tag_path` | `new_learning_theme_tag_path(@learning_theme)` |
| `edit_tag_path(tag)` | `edit_learning_theme_tag_path(@learning_theme, tag)` |
| `tag_path(tag)` | `learning_theme_tag_path(@learning_theme, tag)` |

---
