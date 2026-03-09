# [卒制](https://github.com/QynToKey/HowLongWillItLast) (day 12)：LearningRecord 機能 (`index` / `show` / `edit` / `destroy`)

---

## 1️⃣ 一覧機能

### コントローラーに `index` アクションを追加

```ruby
# app/controllers/learning_records_controller.rb
  def index
    @learning_records = current_user.learning_records.order(study_date: :desc)
  end
```

👉 *自分の学習記録（= `current_user`）のみを取得し、新しい記録から降順に並べる）*

### ビューを作成

```bash
touch app/views/learning_records/index.html.erb
```

```ruby
# touch app/views/learning_records/index.html.erb
<h1>学習ログ</h1>

<%= link_to '記録を追加する', new_learning_record_path %>

<table>
  <thead>
    <tr>
      <th>学習日</th>
      <th>内容</th>
      <th>時間（分）</th>
      <th>アクション</th>
    </tr>
  </thead>
  <tbody>
    <% @learning_records.each do |record| %>
      <tr>
        <td><%= record.study_date.strftime('%Y-%m-%d') %></td>
        <td><%= record.content %></td>
        <td><%= record.duration_minutes %></td>
        <td>
          <%= link_to '編集', edit_learning_record_path(record) %>
          # ⬇️ Rails 7からTurboが導入されたためこう書く
          <%= link_to '削除', learning_record_path(record), data: { turbo_method: :delete, turbo_confirm: '本当に削除しますか？' } %>
        </td>
      </tr>
    <% end %>
  </tbody>
</table>
```

---
