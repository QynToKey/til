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

### `index` ビューを作成

```bash
touch app/views/learning_records/index.html.erb
```

⬇️

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
        # 学習した日を「年月日」で取得して表示
        <td><%= record.study_date.strftime('%Y-%m-%d') %></td>
        <td><%= record.content %></td>
        <td><%= record.duration_minutes %></td>
        <td>
          <%= link_to '編集', edit_learning_record_path(record) %>
          <%= link_to '削除', learning_record_path(record), data: { turbo_method: :delete, turbo_confirm: '本当に削除しますか？' } %> # ⬅️ Rails 7からTurboが導入されたためこう書く
        </td>
      </tr>
    <% end %>
  </tbody>
</table>
```

---

## 2️⃣ 編集機能

### コントローラーに `edit` / `update` 機能を追加

```ruby
# app/controllers/learning_records_controller.rb
  def edit
    # ユーザーが所有する学習記録のみを編集できるようにする
    @learning_record = current_user.learning_records.find(params[:id])
  end

  def update
    @learning_record = current_user.learning_records.find(params[:id])

    if @learning_record.update(learning_record_params)
      redirect_to @learning_record, notice: "学習記録を更新しました"
    else
      flash.now[:alert] = "学習記録の更新に失敗しました"
      render :edit, status: :unprocessable_entity
    end
  end
```

### `edit` ビューを作成

```bash
touch app/views/learning_records/edit.html.erb
```

⬇️

```ruby
# app/views/learning_records/edit.html.erb
<h1><%= @learning_record.study_date.strftime('%Y/%m/%d') %> の学習記録</h1>

<% if @learning_record.errors.any? %>
  <div style="color: red;">
    <h2><%= @learning_record.errors.count %>件のエラーがあります</h2>
    <ul>
      <% @learning_record.errors.full_messages.each do |message| %>
        <li><%= message %></li>
      <% end %>
    </ul>
  </div>
<% end %>

<%= form_with model: @learning_record do |f| %>
  <div>
    <%= f.label :content, "学習内容" %><br>
    <%= f.text_area :content, rows: 4 %>
  </div>

  <div>
    <%= f.label :duration_minutes, "学習時間（分）" %><br>
    <%= f.number_field :duration_minutes, min: 0 %>
  </div>

  <div>
    <%= f.label :study_date, "日付" %><br>
    <%= f.date_field :study_date %>
  </div>

  <div>
    <%= f.submit "更新する" %>
    <%= link_to 'キャンセル', learning_record_path(@learning_record) %>
  </div>
<% end %>
```

---

## 3️⃣ 仮置きした `show` ビューを更新

```ruby
<h1><%= @learning_record.study_date.strftime('%Y/%m/%d') %> の学習記録</h1>

<p>学習内容: <%= @learning_record.content %></p>
<p>学習時間: <%= @learning_record.duration_minutes %> 分</p>

<%= link_to '編集', edit_learning_record_path(@learning_record) %>
<%= link_to 'ログ一覧', learning_records_path %>
```

---

## 4️⃣ 削除機能

### コントローラーに `destroy` アクションを追加

```ruby
# app/controllers/learning_records_controller.rb
  def destroy
    @learning_record = current_user.learning_records.find(params[:id])
    @learning_record.destroy
    redirect_to learning_records_path, notice: "学習記録を削除しました"
  end
```

👉 *削除ダイアログは、`index.html.erb` の削除リンクに書いた `turbo_confirm` によって実行される*

---

## 5️⃣ ビューの共通部分をパーシャル化

### パーシャルを作成

```bash
touch app/views/learning_records/_form.html.erb
```

⬇️

```ruby
# app/views/learning_records/_form.html.erb

<%= form_with model: learning_record do |f| %>
  <div>
    <%= f.label :content, "学習内容" %><br>
    <%= f.text_area :content, rows: 4 %>
  </div>

  <div>
    <%= f.label :duration_minutes, "学習時間（分）" %><br>
    <%= f.number_field :duration_minutes, min: 0 %>
  </div>

  <div>
    <%= f.label :study_date, "日付" %><br>
    <%= f.date_field :study_date %>
  </div>

  <div>
    <%= f.submit learning_record.new_record? ? "記録する" : "更新する" %>
    <%= link_to "戻る", return_path %>
  </div>
<% end %>
```

👉 *１行目は、ローカル変数を受け取るので `@` を外して `<%= form_with model: learning_record do |f| %>` とする*

👉 *ボタンとリンクが `new` と `edit` で異なるため、パーシャルにローカル変数 `return_path` として渡す*

👉 *`f.submit` は、モデルの状態（新規か既存か）を判断してラベルを切り替えてる*

### `new` / `edit` で読み込む

```ruby
# app/views/learning_records/new.html.erb
<%= render 'form', learning_record: @learning_record, return_path: learning_records_path %>
```

👉 *`return_path`で受けて、一覧ページ `learning_records_path` へリダイレクト*

```ruby
# app/views/learning_records/edit.html.erb
<%= render 'form', learning_record: @learning_record, return_path: learning_record_path(@learning_record) %>
```

👉 *`return_path`で受けて、詳細ページ `learning_record_path(@learning_record)` へリダイレクト*

---

## 今後の作業

- `label` / `f.submit` は一通りの実装が終了した後に `i18n`
で設定する
- 「タグ別フィルタリング」および「累計時間集計」は Tag 機能実装時に対応する
