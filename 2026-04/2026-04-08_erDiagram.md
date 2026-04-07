# [co-READER](https://github.com/QynToKey/co_reader)（day: 2）： ER図

## 概要

- 6エンティティ： `users` / `texts` / `reading_sessions` / `memberships` / `annotations` / `replies`
- 孤読・共読のモードは `memberships.mode` で管理（セッション単位ではなく参加者単位）
- 書き込み3種別（ハイライト・アンダーライン・コメント）は `annotations` に統一し、`annotation_type` で区別
- テキスト位置は `start_position` / `end_position`（文字オフセット）で管理

---

## ER 図

```mermaid
erDiagram

    USERS {
        bigint id PK
        string name "NOT NULL"
        string email "NOT NULL, UNIQUE"
        string crypted_password "NOT NULL"
        string salt "NOT NULL"
        string role "NOT NULL"
        datetime created_at
        datetime updated_at
    }

    TEXTS {
        bigint id PK
        bigint uploaded_by FK "NOT NULL"
        string title "NOT NULL"
        text body "NOT NULL"
        datetime created_at
        datetime updated_at
    }

    READING_SESSIONS {
        bigint id PK
        bigint text_id FK "NOT NULL"
        bigint created_by FK "NOT NULL"
        string name
        string invite_token "NOT NULL, UNIQUE"
        datetime created_at
        datetime updated_at
    }

    MEMBERSHIPS {
        bigint id PK
        bigint user_id FK "NOT NULL"
        bigint reading_session_id FK "NOT NULL, UNIQUE (user_id, reading_session_id)"
        string mode "NOT NULL"
        integer reading_progress
        datetime joined_at "NOT NULL"
    }

    ANNOTATIONS {
        bigint id PK
        bigint user_id FK "NOT NULL"
        bigint reading_session_id FK "NOT NULL"
        string annotation_type "NOT NULL"
        integer start_position "NOT NULL"
        integer end_position "NOT NULL"
        string color
        text body
        datetime created_at
        datetime updated_at
    }

    REPLIES {
        bigint id PK
        bigint annotation_id FK "NOT NULL"
        bigint user_id FK "NOT NULL"
        text body "NOT NULL"
        datetime created_at
        datetime updated_at
    }

    USERS ||--o{ TEXTS : uploads
    USERS ||--o{ READING_SESSIONS : creates
    TEXTS ||--o{ READING_SESSIONS : has
    USERS ||--o{ MEMBERSHIPS : joins
    READING_SESSIONS ||--o{ MEMBERSHIPS : has
    USERS ||--o{ ANNOTATIONS : writes
    READING_SESSIONS ||--o{ ANNOTATIONS : has
    ANNOTATIONS ||--o{ REPLIES : has
    USERS ||--o{ REPLIES : writes
```
