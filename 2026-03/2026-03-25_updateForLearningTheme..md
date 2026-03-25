# [卒制](https://github.com/QynToKey/HowLongWillItLast) (MVP 後)：`LearningTheme` 再実装のための準備

## 行ったこと

- 関連ドキュメント（README / ER図）の更新
- Issue 設計

## 背景・設計方針

- このアプリが最優先すべきスコアは「総学習時間」である
  👉 *「総学習時間」の表示は実装済み*
- それに伴い、タグの上位カテゴリとして「学習テーマ」 `LearningTheme` を段階的に再実装する

## 変更内容

- README：テーブル詳細に `learning_themes` を追加、各テーブルに `learning_theme_id` を追記
- README：機能候補に「学習テーマの管理機能」を追加
- ER図：`learning_themes` エンティティの追加とリレーション・制約設計を更新

## 次のステップ

- フェーズ1：`LearningTheme` モデル作成・既存データ移行
- フェーズ2：Todo 機能実装
- フェーズ3：`LearningTheme` CRUD 実装
