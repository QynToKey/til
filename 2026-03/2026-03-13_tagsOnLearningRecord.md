# [卒制](https://github.com/QynToKey/HowLongWillItLast) (day 15-2)：Tags を LearningRecord へ実装

## 6️⃣ LearningRecord 作成 / 編集 に「タグ」を追加

👉  *パーシャルへ追加する*

```ruby
# app/views/learning_records/_form.html.erb
  <div>
    <%= f.label :tag_ids, "タグ" %><br>
    <% current_user.tags.each do |tag| %>
      <%= f.check_box :tag_ids, { multiple: true, checked: learning_record.tags.include?(tag) }, tag.id, false %>
      <%= tag.name %>
    <% end %>
  </div>
```

📝 `<%= f.check_box :tag_ids, { multiple: true, checked: learning_record.tags.include?(tag) }, tag.id, false %>` ：
*タグがすでに付いていればチェック済みで表示し、チェックされたらそのIDを送信する*

- `tag_ids` ：
送信するパラメータの名前。`learning_record[tag_ids][]` という形でサーバーに送られる
- `multiple: true` ：
複数のチェックボックスから配列として値を送信するための指定
- `checked: learning_record.tags.include?(tag)` ：
編集時に「すでにこのタグが付いているか」を確認して、チェック済み状態にするかどうかを決める *（新規作成時は `false`）*
- `tag.id` ：
チェックされたときに送信される値 *（どのタグが選ばれたかをIDで識別する）*
- `false` ：
チェックされていないときに送信される値  *（通常は `0` や `false` を指定して「未選択」を明示する。 `false` のとき、未チェック時は何も送信されない）*

---

## 7️⃣ LearningRecords 一覧 へ「タグ」を追加

### `index` ビューに「タグ」を表示

```ruby
# app/views/learning_records/index.html.erb
        <td>
          <% record.tags.each do |tag| %>
            <span><%= tag.name %></span>
          <% end %>
        </td>
```

### コントローラーで「N+1問題」に対応する

```ruby
  def index
    # ユーザーが所有する学習記録をタグ情報とともに取得し、学習日が新しい順に並べる
    @learning_records = current_user.learning_records.includes(:tags).order(study_date: :desc)
  end
```

📝 **N+1 問題**

- 未対応の場合、下のような SQL が発行される

```sql
-- 1回目: 学習記録を全件取得
SELECT * FROM learning_records WHERE user_id = 1;

-- 記録の件数分だけ繰り返し発行される
SELECT * FROM tags WHERE record_id = 1;
SELECT * FROM tags WHERE record_id = 2;
SELECT * FROM tags WHERE record_id = 3;
-- ...
```

- 解決策： `includes` を使って最初に関連データをまとめて取得する
👉 *2回の SQL で完結する*

---

## 8️⃣ タグによるフィルタリング機能を実装

👉 *実装イメージ：ユーザーがタグをクリックすると、そのタグが付いた記録だけに絞り込まれる*

### `index` アクションにフィルタリング機能を実装

```ruby
# app/controllers/learning_records_controller.rb
  def index
    ・・・

    if params[:tag_id].present?
      # タグIDが指定されている場合は、そのタグが付いている学習記録のみを表示する
      @learning_records = @learning_records.joins(:tags).where(tags: { id: params[:tag_id] })
    end
  end
```

📝 `joins(:tags)` ：
 `learning_records` と `tags` を JOIN する

| メソッド | 働き |
| --- | --- |
| `includes` | 関連データの事前読み込み |
| `joins` | 絞り込みのためのSQL結合 |

📝 `.where(tags: { id: params[:tag_id] })` ：
「指定されたIDのタグが付いている記録のみ」に絞り込む

⬇️ 実行される流れはこうなる

- タグ指定あり → `includes` で取得 → `joins` で絞り込み
- タグ指定なし → `includes` で全件取得

---
