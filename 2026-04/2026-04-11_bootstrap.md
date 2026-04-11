# [co-READER](https://github.com/QynToKey/co_reader)（day: 5）： Bootstrap を導入

## 1️⃣ Bootstrap の導入

> **導入理由**：
[前プロジェクト](https://github.com/QynToKey/til/blob/main/2026-03/2026-03-17_chores.md)で実績があることに加え、
一連の関連アプリ群として統一感を持たせたい。

👉 *[Bootstrap](https://getbootstrap.com/) が提供している CDN タグを `app/views/layouts/application.html.erb` に貼る*

```erb
  <head>
    ・・・
    <!-- ⬇️ stylesheet_link_tag の前に置く -->
    <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.8/dist/css/bootstrap.min.css" rel="stylesheet" integrity="sha384-sRIl4kxILFvY47J16cr9ZwB07vP4J8+LH7qKQnuqkuIAvNWLzeN8tE5YBujZqJLB" crossorigin="anonymous">
    <%= stylesheet_link_tag :app, "data-turbo-track": "reload" %>
    <%= javascript_importmap_tags %>
    <!-- ⬇️ javascript_import_tag の後ろに置く -->
    <script src="https://cdn.jsdelivr.net/npm/bootstrap@5.3.8/dist/js/bootstrap.bundle.min.js" integrity="sha384-FKyoEForCGlyvwx9Hj09JcYn3nv7wiPVlz7YYwJrWVcXK/BmnVDxM+D2scQbITxI" crossorigin="anonymous"></script>
  </head>
```

---
