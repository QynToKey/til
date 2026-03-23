# [卒制](https://github.com/QynToKey/HowLongWillItLast) (MVP 後)：TOP ページをリニューアル

## 0️⃣ 実装方針

TOP ページを実務的かつシンプルなものに更新する。

- アプリの機能や特徴を、未登録ユーザーにも簡潔に伝える
- 説明文を簡素に
- ログインユーザーに「総学習時間」を視認性高く表示する
- マイルストーンを４つ設定する
  - 1000時間(初級)
  - 2500時間(中級)
  - 5000時間(上級)
  - 10000時間(エキスパート)

---

## 1️⃣ モデル側：User モデルで「マイルストーン」を定義

```ruby
# app/models/user.rb
class User < ApplicationRecord
  ・・・

  # 総学習時間のマイルストーンを設定
  THRESHOLDS = [
    { hours: 1000, label: "初級" },
    { hours: 2500, label: "中級" },
    { hours: 5000, label: "上級" },
    { hours: 10000, label: "エキスパート" },
  ].freeze

  ・・・

  # 現在の総学習時間を超える最初の閾値を取得
  def next_threshold
    total_hours = total_learning_minutes / 60.0
    THRESHOLDS.find { |t| t[:hours] > total_hours }
  end
```

📝 `.freeze` ：
定数を凍結（変更不可）するメソッド。
Rubyでは定数でも破壊的メソッドで中身を変えられてしまうので、`.freeze` をつけることで意図しない変更を防ぐ。

  ⬇️ ターミナルで動作確認

```bash
>> current_user = User.first
  User Load (2.0ms)  SELECT "users".* FROM "users" ORDER BY "users"."id" ASC LIMIT $1  [["LIMIT", 1]]
=>
#<User:0x0000ffffa6fdb410
...
>> current_user.next_threshold
  LearningRecord Sum (2.7ms)  SELECT SUM("learning_records"."duration_minutes") FROM "learning_records" WHERE "learning_records"."user_id" = $1  [["user_id", 2]]
=> {:hours=>1000, :label=>"初級"}
```

---

## 2️⃣ コントローラー側：TOP ページで「マイルストーン」を使えるようにする

```ruby
# app/controllers/home_controller.rb
class HomeController < ApplicationController
  skip_before_action :require_login, only: [ :index ]

  def index
    # ログインユーザーのみ「総学習時間」「次のマイルストーン」を用意
    if logged_in?
      @total_hours = (current_user.total_learning_minutes / 60.0).round(1)
      @next_threshold = current_user.next_threshold
    end
  end
end
```

---

## 3️⃣ TOP ページを更新

👉 *プログレスバーを実装して、進捗状況を可視化*

```html
<div class="container mt-5">
  <p class="text-muted mb-1" style="font-size: 11px; letter-spacing: 0.08em;">LEARNING TRACKER</p>
  <h1 class="h4 mb-3">How Long Will It Last</h1>

  <p class="text-muted small">
    学習時間と学習内容を記録し、長期的な自己変容の足跡を残すツールです。</p>

  <% if logged_in? %>
    <div class="mt-5 mb-4">
      <% if current_user.learning_theme.present? %>
        <p class='small mt-3'>'<%= current_user.learning_theme %>' の総学習時間</p>
      <% else %>
        <p class='small mt-3'>総学習時間</p>
      <% end %>

      <p class="h3 mb-2"><%= @total_hours %> 時間</p>

      <% progress = [(@total_hours / 10000.0 * 100), 100].min.round(1) %>
      <div class="progress mb-1" style="height: 6px;">
        <div class="progress-bar bg-dark" style="width: <%= progress %>%;"></div>
      </div>

      <div class="d-flex justify-content-between">
        <span class="text-muted" style="font-size: 11px;">0</span>
        <span class="text-muted text-center" style="font-size: 11px;">1,000h<br>初級</span>
        <span class="text-muted text-center" style="font-size: 11px;">2,500h<br>中級</span>
        <span class="text-muted text-center" style="font-size: 11px;">5,000h<br>上級</span>
        <span class="text-muted text-center" style="font-size: 11px;">10,000h<br>エキスパート</span>
      </div>

      <% if @next_threshold %>
        <p class="text-muted small mt-2">
          次のマイルストーンまで <%= (@next_threshold[:hours] - @total_hours).round(1) %> 時間
          （<%= @next_threshold[:label] %>: <%= number_with_delimiter(@next_threshold[:hours]) %>時間）
        </p>
      <% else %>
        <p class="text-muted small mt-2">エキスパートレベル達成！</p>
      <% end %>
    </div>

    <%= link_to '今日の記録', new_learning_record_path(study_date: Date.current), class: "btn btn-sm btn-primary" %>
    <%= link_to '学習ログ', learning_records_path, class: "btn btn-sm btn-outline-primary" %>

  <% else %>
    <div class="d-flex gap-2 mt-5 mb-3">
      <%= link_to '試してみる', new_learning_record_path, class: "btn btn-sm btn-primary" %>
      <%= link_to 'ユーザー登録', new_user_path, class: "btn btn-sm btn-outline-secondary" %>
      <%= link_to 'ログイン', login_path, class: "btn btn-sm btn-outline-secondary" %>
    </div>

    <p class="text-muted small mt-3">
      ※ 記録するには <%= link_to 'ユーザー登録', new_user_path %> /
      <%= link_to 'ログイン', login_path %> してください。
    </p>
  <% end %>

  <div class="text-muted small mt-5 text-center">
    <%= link_to 'このアプリについて', about_path %>  |
    <%= link_to '開発ログ', logs_path %>  |
    <%= link_to 'ご意見 / お問合せ', 'https://forms.gle/FF7vLDtC6qdt7m2P7', target: "_blank", rel: "noopener noreferrer" %>
  </div>
</div>
```

---

## プログレスバーの実装について

### ① 進捗の割合を計算する

```erb
<% progress = [(@total_hours / 10000.0 * 100), 100].min.round(1) %>
```

- `@total_hours / 10000.0 * 100` で「10,000時間を100%としたときの現在地（%）」を計算
- `10000.0` と小数にしているのは整数同士の割り算にすると小数点以下が切り捨てられるため
- `[計算結果, 100].min`は「計算結果と100のうち小さい方を取る」という意味で、10,000時間を超えてもバーが100%を超えないようにしている

### ② バーの幅にinlineで適用する

```erb
<div class="progress mb-1" style="height: 6px;">
  <div class="progress-bar bg-dark" style="width: <%= progress %>%;"></div>
</div>
```

- Bootstrapの `progress`/`progress-bar` クラスでバーの外枠と中身を作る
- `style="width: <%= progress %>%;"` でERBの変数をCSSに埋め込む
- `width`の値が動的に変わることでバーの長さが変わる

---

#### 総学習時間：
