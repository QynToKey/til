# [卒制](https://github.com/QynToKey/HowLongWillItLast) (MVP 後)：`LearningTheme` 再実装のための準備

## 行ったこと

- Issue 設計
- 関連ドキュメント（README / ER図）の更新

## 背景・設計方針

- このアプリが最優先すべきスコアは「総学習時間」である
  👉 *「総学習時間」の表示は実装済み*
- それに伴い、タグの上位カテゴリとして「学習テーマ」 `LearningTheme` を段階的に再実装する

---

### Issue 設計

#### 0️⃣ ドキュメント更新 : 2sp

👉 *着手前に設計ドキュメントを LearningTheme 対応版に更新*

- `README` 更新（ER図・機能候補・テーブル詳細の記述を修正） : 1sp
- `docs/er_diagram.md` 更新（エンティティ・リレーション・制約設計を修正） : 1sp

#### 1️⃣ `LearningTheme` 基盤構築 : 6sp

👉 *Todo 機能・CRUD 実装の前提となる DB 基盤を整える*

```md
① LearningTheme モデルの作成 : 2sp
- `learning_themes` テーブル作成
- `User` との関連付け

② 既存データの移行 : 4sp
- `learning_themes` テーブルへ既存データをコピー
- `learning_records` に `learning_theme_id` を追加し紐付け
- `tags` に `learning_theme_id` を追加し紐付け
- `users.learning_theme` カラムを削除
```

> ✅ フェーズ 1 完了後、フェーズ 2（Todo 機能）に着手可能になる。

#### 2️⃣ Todo 機能 : 5sp

👉 *`learning_themes` テーブルが存在する状態で実装する*
👉 *Todo 作成時のテーマ紐づけは、この時点では唯一のテーマに自動紐づけとする（テーマ選択 UI は後続フェーズで追加）*

- Todo モデルに `learning_theme_id` を追加
- Todo 作成機能（テーマ自動紐づけ）
- Todo 一覧表示
- Todo 編集機能
- Todo 削除機能

> 📝 既存の Todo モデルがあるため、マイグレーション追加と CRUD 実装が主な作業。

#### 3️⃣ LearningTheme CRUD : 5sp

👉 *複数テーマの作成・管理機能を実装する*

📝 **複数テーマ追加の導線（リンク・ボタン等）はビューに配置しない**（リリース保留）。ルーティング・コントローラー・ビューは実装済みの状態にしておく。

- LearningTheme 作成機能
- LearningTheme 一覧機能
- LearningTheme 編集機能
- LearningTheme 削除機能
- ユーザー登録時に 1 つ作成する導線

> 🔒 複数テーマ追加の導線：実装済み・公開保留

#### 4️⃣ マイページ対応 : 3sp

👉 *テーマ別の累計時間表示と、複数テーマへの対応を実装*

- テーマ別累計時間表示
- 複数テーマへの対応（テーマ切り替え・集計ロジック）

#### 5️⃣ 目標時間カスタム機能 : TBD

👉 *10000 時間（最終マイルストーン）到達後のユーザー向けに、目標時間を自由設定できる機能を実装*

📝 **ユーザーへの開放はリリース保留**。ルーティング・コントローラー・ビューは実装済みの状態にしておく。

- 目標時間カスタム設定のモデル・ロジック実装
- プログレスバーのカスタム目標対応
- ビュー実装（導線は非表示）

> 🔒 ユーザー開放：実装済み・公開保留

---

### ドキュメントの更新 （`README` / `docs/er_diagram.md`）

- README：テーブル詳細に `learning_themes` を追加、各テーブルに `learning_theme_id` を追記
- README：機能候補に「学習テーマの管理機能」を追加
- ER図：`learning_themes` エンティティの追加とリレーション・制約設計を更新

---

#### 総学習時間 : 1142.6 時間
