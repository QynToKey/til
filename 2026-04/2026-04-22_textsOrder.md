# [co-READER](https://github.com/QynToKey/co_reader)（day: 15）： セッション内テキストの並び順制御機能

## 1️⃣ `position` / `enforce_order` カラムを追加

```bash
$ docker compose exec web bin/rails g migration AddPositionToReadingSessionTexts position:integer
      invoke  active_record
      create    db/migrate/20260422015628_add_position_to_reading_session_texts.rb
```

```bash
$ docker compose exec web bin/rails g migration AddEnforceOrderToReadingSessions enforce_order:boolean
      invoke  active_record
      create    db/migrate/20260422015933_add_enforce_order_to_reading_sessions.rb
```

  ⬇️

```ruby
# db/migrate/20260422015628_add_position_to_reading_session_texts.rb
class AddPositionToReadingSessionTexts < ActiveRecord::Migration[8.1]
  def change
    add_column :reading_session_texts, :position, :integer
  end
end
```

```ruby
# db/migrate/20260422015933_add_enforce_order_to_reading_sessions.rb
class AddEnforceOrderToReadingSessions < ActiveRecord::Migration[8.1]
  def change
    add_column :reading_sessions, :enforce_order, :boolean, default: false, null: false
  end
end
```

  ⬇️

```bash
$ docker compose exec web bin/rails db:migrate
== 20260422015628 AddPositionToReadingSessionTexts: migrating =================
-- add_column(:reading_session_texts, :position, :integer)
   -> 0.0094s
== 20260422015628 AddPositionToReadingSessionTexts: migrated (0.0095s) ========

== 20260422015933 AddEnforceOrderToReadingSessions: migrating =================
-- add_column(:reading_sessions, :enforce_order, :boolean, {:default=>false, :null=>false})
   -> 0.0186s
== 20260422015933 AddEnforceOrderToReadingSessions: migrated (0.0187s) ========
```

---

## 2️⃣ 「管理者フォーム」を更新

```erb
<%# app/views/admin/reading_sessions/_form.html.erb %>

-  <div class="mb-3">
-    <label class="form-label"><%= t("admin.reading_sessions.form.texts_label") %></label>
-    <%= f.collection_check_boxes :text_ids, @texts, :id, :title do |b| %>
-      <div class="form-check">
-        <%= b.check_box class: "form-check-input" %>
-        <%= b.label class: "form-check-label" %>
-      </div>
-    <% end %>
-  </div>
+  <div class="mb-3">
+    <label class="form-label"><%= t("admin.reading_sessions.form.texts_label") %></label>
+    <% @texts.each do |text| %>
+      <div class="d-flex align-items-center gap-3 mb-2">
+        <div class="form-check mb-0">
+          <%= check_box_tag "reading_session[text_ids][]",
+                            text.id,
+                            @reading_session.texts.include?(text),
+                            id: "text_#{text.id}",
+                            class: "form-check-input" %>
+          <%= label_tag "text_#{text.id}", text.title, class: "form-check-label" %>
+        </div>
+        <%= number_field_tag "reading_session[text_positions][#{text.id}]",
+                             @reading_session.reading_session_texts.find_by(text: text)&.position,
+                             min: 1,
+                             class: "form-control form-control-sm",
+                             style: "width: 80px;",
+                             placeholder: "順番" %>
+      </div>
+    <% end %>
+  </div>
+
+  <div class="mb-3">
+    <div class="form-check">
+      <%= f.check_box :enforce_order, class: "form-check-input" %>
+      <%= f.label :enforce_order, "順番を固定する（読者は指定順にのみ閲覧可）", class: "form-check-label" %>
+    </div>
+  </div>
```

```ruby
# app/assets/stylesheets/application.css
::placeholder {
  font-size: 0.8rem;
  color: #bbb !important;
}
```

---
