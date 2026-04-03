# [卒制](https://github.com/QynToKey/HowLongWillItLast) (MVP 後)： 複数の `LearningThemes` に対応

## 1️⃣ `LearningTheme` モデルに学習時間計算メソッドを追加

👉 *`User` モデルに定義した `total_learning_minutes` / `next_threshold` を転用する (`THRESHOLDS` は `User` モデルで定義されているので `User::THRESHOLDS` と書く)*

```ruby
# app/models/learning_theme.rb

  # 総学習時間
  def total_learning_minutes
    learning_records.sum(:duration_minutes)
  end

  # 現在の総学習時間を超える最初の閾値を取得
  def next_threshold
    total_hours = total_learning_minutes / 60.0
    User::THRESHOLDS.find { |t| t[:hours] > total_hours }
  end

  # 総学習時間を時間単位で取得（小数点1位まで） ⬅️ 新規追加（時間への変換をモデル側に持たせた方がビューがすっきりするため）
  def total_learning_minutes_in_hours
    (total_learning_minutes / 60.0).round(1)
  end
```

  ⬇️ 動作確認

```bash
>> theme = User.first.learning_themes.first
=>
#<LearningTheme:0x0000ffff99928e18
...
>> theme.total_learning_minutes
=> 60616
>> theme.next_threshold
=> {:hours=>2500, :label=>"中級"}
```

---

## 2️⃣ `UsersController#show` の修正

```ruby
# app/controllers/users_controller.rb
$ cat app/controllers/users_controller.rb
class UsersController < ApplicationController
  ・・・
  # before_action :set_learning_theme, only: %i[edit] ⬅️ edit だけなので set_learning_theme メソッドごと削除

  def show
    # ユーザープロフィール画面では、ユーザーの学習テーマとタグごとの学習時間を表示するためにデータを準備する
    @learning_themes = current_user.learning_themes

    # 各 learning_theme ごとにタグの学習時間を計算してまとめる
    @tag_summaries_by_theme = @learning_themes.each_with_object({}) do |theme, hash|
      hash[theme.id] = theme.tags.map do |tag|
        {
          name: tag.name,
          hours: (current_user.total_learning_minutes_by_tag(tag) / 60.0).round(1)
        }
      end
    end
  end

  def edit
    # 編集画面では、最初の learning_theme を表示する（複数登録されている場合も最初のものだけ編集可能）
    @learning_theme = current_user.learning_themes.first
  end
```

---

## 3️⃣ ビューの更新

### マイページ `users#show`

👉 *タグ・Todoカードの `@learning_theme` 参照は、複数テーマ対応後は「どのテーマのタグ・Todoか」をテーマごとに表示する構造になる。*

👉 *併せて `total_learning_minutes_in_hours` メソッドを全ての該当箇所で実装*

```erb
<%# app/views/users/show.html.erb %>

<div class="container mt-5" style="max-width: 600px;">
  <h1 class="h5 mb-4">マイページ : '<%= current_user.name %>' さん</h1>

  <div class="card mb-3">
    <div class="card-body">
      <h2 class="card-title h6 text-muted">プロフィール</h2>
      <p class="mb-1 small ps-3">
        お名前： <%= current_user.name %>
      </p>
      <p class="mb-1 small ps-3">
        email： <%= current_user.email %>
      </p>
      <p class="mb-3 small ps-3">
        パスワード： (変更する場合は<%= link_to 'こちら', edit_user_path(current_user) %>)
      </p>
    <%= link_to 'プロフィールを編集する', edit_user_path(current_user), class: "btn btn-sm btn-outline-primary" %>
    </div>
  </div>

  <% @learning_themes.each do |theme| %>
    <div class="card mb-3">
      <div class="card-body">
        <h2 class="card-title h6 text-muted">
          学習テーマ： <%= theme.name.presence || '未設定' %>
        </h2>

        <%# テーマ削除は、複数テーマ公開時にコメントアウトを外す %>
        <%# <%= button_to "このテーマを削除する", learning_theme_path(theme), method: :delete, data: { turbo_confirm: "このテーマを削除すると、関連する記録 / タグ / TODO も全て削除されます。よろしいですか？" }, class: "btn btn-sm btn-outline-danger" %>

        <p class='small mt-3'>総学習時間： <%= theme.total_learning_minutes_in_hours %> 時間</p>

        <%= render "shared/progressbar", theme: theme %>

        <% tag_summaries = @tag_summaries_by_theme[theme.id] %>
        <% if tag_summaries&.any? %>
          <ul class="small mt-2 ps^3">
            <% tag_summaries.each do |summary| %>
              <li><%= summary[:name] %>： <%= summary[:hours] %> 時間</li>
            <% end %>
          </ul>
        <% end %>

        <%= link_to '今日の学習を記録する', new_learning_record_path(study_date: Date.current), class: "btn btn-sm btn-outline-primary" %>
        <%= link_to '学習ログ', learning_records_path, class: "btn btn-sm btn-outline-primary" %>

    <hr class="my-3">

    <h3 class="h6 text-muted">タグ</h3>
      <% if theme.tags.present? %>
        <p class='small mt-2'>設定済み： <%= theme.tags.map(&:name).join(', ') %></p>
        <%= link_to 'タグを編集する', learning_theme_tags_path(theme), class: "btn btn-sm btn-outline-primary" %>
      <% else %>
        <p class="text-muted small mt-3">※ 学習記録にタグを設定すると、タグごとに学習ログを管理できます</p>
        <%= link_to 'タグを追加する', learning_theme_tags_path(theme), class: "btn btn-sm btn-outline-primary" %>
      <% end %>

    <hr class="my-3">

    <h3 class="h6 text-muted">TODO</h3>
      <% if theme.todos.present? %>
        <p class='small mt-2'><%= theme.todos.count %>件の TODO が登録されています。</p>
        <%= link_to 'TODO 一覧へ', learning_theme_todos_path(theme), class: "btn btn-sm btn-outline-primary" %>
      <% else %>
        <p class="text-muted small mt-2">※ 学習計画や参照資料を TODO リストで管理できます</p>
        <%= link_to 'TODO を追加する', learning_theme_todos_path(theme), class: "btn btn-sm btn-outline-primary" %>
      <% end %>
      </div>
    </div>
  <% end %>
</div>

```

### プログレスバー（パーシャル）

👉 *インスタンス変数をパーシャル内で使うと、どこから値が来ているか追いにくくなるので `@total_hours` と `@next_threshold` をローカル変数に*

```erb
<%# app/views/shared/_progressbar.html.erb %>

<%# 各ステージの上限を 100% として進捗状況を描画 %>
<% total_hours = theme.total_learning_minutes_in_hours %>
<% max_hours = if theme.total_learning_minutes <= 1000 * 60
                1000.0
              elsif theme.total_learning_minutes <= 2500 * 60
                2500.0
              elsif theme.total_learning_minutes <= 5000 * 60
                5000.0
              else
                10000.0
              end %>
<% progress = [(total_hours / max_hours * 100), 100].min.round(1) %>

<div class="progress mb-1" style="height: 6px;">
  <div class="progress-bar bg-dark" style="width: <%= progress %>%;"></div>
</div>

<%# 各ステージごとに目盛りの配置を切り替える %>
<div style="position: relative; height: 28px; margin-top: 6px;">
  <% if theme.total_learning_minutes <= 1000 * 60 %>
    <span class="text-muted" style="position: absolute; left: 0%; font-size: 11px; transform: translateX(0);">0</span>
    <span class="text-muted text-center" style="position: absolute; left: 100%; font-size: 11px; transform: translateX(-100%);">1,000h<br>初級</span>
  <% elsif (theme.total_learning_minutes > 1000 * 60) && (theme.total_learning_minutes <= 2500 * 60) %>
    <span class="text-muted" style="position: absolute; left: 0%; font-size: 11px; transform: translateX(0);">0</span>
    <span class="text-muted text-center" style="position: absolute; left: 40%; font-size: 11px; transform: translateX(-50%);">1,000h<br>初級</span>
    <span class="text-muted text-center" style="position: absolute; left: 100%; font-size: 11px; transform: translateX(-100%);">2,500h<br>中級</span>
  <% elsif (theme.total_learning_minutes > 2500 * 60) && (theme.total_learning_minutes <= 5000 * 60) %>
    <span class="text-muted" style="position: absolute; left: 0%; font-size: 11px; transform: translateX(0);">0</span>
    <span class="text-muted text-center" style="position: absolute; left: 20%; font-size: 11px; transform: translateX(-50%);">1,000h<br>初級</span>
    <span class="text-muted text-center" style="position: absolute; left: 50%; font-size: 11px; transform: translateX(-50%);">2,500h<br>中級</span>
    <span class="text-muted text-center" style="position: absolute; left: 100%; font-size: 11px; transform: translateX(-100%);">5,000h<br>上級</span>
  <% else %>
    <span class="text-muted" style="position: absolute; left: 0%; font-size: 11px; transform: translateX(0);">0</span>
    <span class="text-muted text-center" style="position: absolute; left: 10%; font-size: 11px; transform: translateX(-50%);">1,000h<br>初級</span>
    <span class="text-muted text-center" style="position: absolute; left: 25%; font-size: 11px; transform: translateX(-50%);">2,500h<br>中級</span>
    <span class="text-muted text-center" style="position: absolute; left: 50%; font-size: 11px; transform: translateX(-50%);">5,000h<br>上級</span>
    <span class="text-muted text-center" style="position: absolute; left: 100%; font-size: 11px; transform: translateX(-100%);">10,000h<br>エキスパート</span>
  <% end %>
</div>

<% next_threshold = theme.next_threshold %>
<% if next_threshold %>
  <p class="text-muted small mt-2">
    次のマイルストーンまで <%= (next_threshold[:hours] - total_hours).round(1) %> 時間
    （<%= next_threshold[:label] %>: <%= number_with_delimiter(next_threshold[:hours]) %>時間）
  </p>
<% else %>
  <p class="text-muted small mt-2">🎉 エキスパートレベル達成！</p>
<% end %>
```

---

## 補足

- 現状 `learning_records` は `user` に直接紐づくのではなく、`learning_theme` を介して紐づいているため、`learning_records` は `user_id` と `learning_theme_id` の両方を持っている。
- 設計上は `learning_theme_id` だけで user を特定できるので、`user_id` は冗長とも言える。

  ⬇️ リファクタリング issue として積んでおく

> [learning_records / tags / todos の user_id を learning_theme 経由に統一する#177](https://github.com/users/QynToKey/projects/5/views/1?pane=issue&itemId=172175287&issue=QynToKey%7CHowLongWillItLast%7C177)
