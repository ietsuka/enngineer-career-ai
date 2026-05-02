# ER 図(論理データモデル)

| 項目 | 内容 |
|---|---|
| プロジェクト名 | engineer-career-ai |
| バージョン | 1.1 |
| 作成日 | 2026-05-02 |
| 更新日 | 2026-05-02 |

※ 物理設計(型・インデックス・DDL)は `03_detailed_design/db_physical_design.md` で定義する。

---

## 1. ER 図

```mermaid
erDiagram
    USERS ||--|| USER_ROLES : "1:1 ロール管理"
    USERS ||--o| USER_PROFILES : "1:1 プロフィール"
    USERS ||--o{ CHAT_SESSIONS : "1:N 会話セッション"
    USERS ||--o{ CAREER_PLANS : "1:N キャリアプラン"
    USERS ||--o{ SKILL_INVENTORIES : "1:N スキル棚卸し"
    USERS ||--o{ USAGE_QUOTAS : "1:N 利用量管理"
    USERS ||--o{ CONSENT_LOGS : "1:N 同意ログ"
    CHAT_SESSIONS ||--o{ CHAT_MESSAGES : "1:N メッセージ"
    USERS ||--o{ AUDIT_LOGS : "1:N 操作ログ"
    USERS ||--o{ API_USAGE_LOGS : "1:N API利用ログ"
    ANON_SESSIONS ||--o{ ANON_MESSAGES : "1:N 未ログインメッセージ"

    USERS {
        uuid id PK
        string email UK
        string provider
        timestamp deleted_at
        timestamp scheduled_purge_at
        timestamp created_at
        timestamp updated_at
    }

    USER_ROLES {
        uuid id PK
        uuid user_id FK UK
        string role
        timestamp created_at
        timestamp updated_at
    }

    USER_PROFILES {
        uuid id PK
        uuid user_id FK UK
        string handle_name
        int age
        string current_job
        text career_history
        int current_salary
        int target_salary
        json skills
        string career_orientation
        boolean profile_completed
        timestamp created_at
        timestamp updated_at
    }

    CHAT_SESSIONS {
        uuid id PK
        uuid user_id FK
        string theme
        string title
        timestamp last_message_at
        timestamp created_at
        timestamp updated_at
    }

    CHAT_MESSAGES {
        uuid id PK
        uuid session_id FK
        string role
        text content
        json suggestion_buttons
        boolean expert_card_shown
        int tokens_used
        timestamp created_at
    }

    CAREER_PLANS {
        uuid id PK
        uuid user_id FK
        string title
        text goal
        json action_items
        date target_date
        uuid source_session_id FK
        timestamp created_at
        timestamp updated_at
    }

    SKILL_INVENTORIES {
        uuid id PK
        uuid user_id FK
        text experiences
        text strengths
        text weaknesses
        text interests
        text work_style
        text ai_analysis
        timestamp analyzed_at
        timestamp created_at
        timestamp updated_at
    }

    USAGE_QUOTAS {
        uuid id PK
        uuid user_id FK
        date quota_date
        int message_count
        timestamp updated_at
    }

    API_USAGE_LOGS {
        uuid id PK
        uuid user_id FK
        string model
        int input_tokens
        int output_tokens
        string endpoint
        timestamp created_at
    }

    CONSENT_LOGS {
        uuid id PK
        uuid user_id FK
        string consent_type
        boolean accepted
        string ip_hash
        timestamp created_at
    }

    AUDIT_LOGS {
        uuid id PK
        uuid user_id FK
        string action
        json metadata
        string ip_hash
        timestamp created_at
    }

    ANON_SESSIONS {
        uuid id PK
        string session_token UK
        string theme
        int message_count
        string ip_hash
        timestamp expires_at
        timestamp created_at
    }

    ANON_MESSAGES {
        uuid id PK
        uuid anon_session_id FK
        string role
        text content
        timestamp created_at
    }
```

---

## 2. エンティティ一覧

| エンティティ名(論理) | テーブル名(物理) | 概要 |
|---|---|---|
| ユーザー | users | 認証ユーザー。Supabase Auth と連動 |
| ユーザーロール | user_roles | ロール管理(free/admin)。JWT ではなく DB で管理 |
| ユーザープロフィール | user_profiles | キャリア属性(ハンドルネーム/年齢/経歴/年収/スキル/志向) |
| チャットセッション | chat_sessions | 壁打きの1スレッド(テーマ・タイトル) |
| チャットメッセージ | chat_messages | スレッド内の1発言(user or assistant) |
| キャリアプラン | career_plans | AI対話から作成した構造化キャリアプラン |
| スキル棚卸し | skill_inventories | 5項目フォーム入力 + AI分析結果 |
| 利用量クォータ | usage_quotas | ログインユーザーの1日あたりのメッセージ数管理 |
| API利用ログ | api_usage_logs | 消費トークン数の追跡(月額コスト管理用) |
| 同意ログ | consent_logs | Cookie同意・利用規約同意の取得記録 |
| 操作ログ | audit_logs | 退会・削除等の重要操作の監査証跡 |
| 匿名セッション | anon_sessions | 未ログインユーザーのセッション管理(レート制限用) |
| 匿名メッセージ | anon_messages | 匿名セッション内の発言(ログイン誘導後に破棄) |

---

## 3. エンティティ詳細(論理)

### 3.1 users(ユーザー)

| 論理名 | 物理名 | 型 | 必須 | 説明 |
|---|---|---|:--:|---|
| ユーザーID | id | UUID | ○ | PK。Supabase Auth の auth.users.id と一致 |
| メールアドレス | email | 文字列 | ○ | UK。認証用 |
| 認証プロバイダ | provider | 文字列 | ○ | `email` / `google` / `github` |
| 削除日時 | deleted_at | 日時 | - | 退会申請時にセット |
| 完全削除予定日時 | scheduled_purge_at | 日時 | - | `deleted_at + 30日` |
| 作成日時 | created_at | 日時 | ○ | |
| 更新日時 | updated_at | 日時 | ○ | |

### 3.2 user_roles(ユーザーロール)

| 論理名 | 物理名 | 型 | 必須 | 説明 |
|---|---|---|:--:|---|
| ロールID | id | UUID | ○ | PK |
| ユーザーID | user_id | UUID | ○ | FK(users.id)、UK(1:1) |
| ロール | role | 文字列 | ○ | `free`(MVP 全員) / `admin`(将来) / `premium`(フェーズ2) |
| 作成日時 | created_at | 日時 | ○ | |
| 更新日時 | updated_at | 日時 | ○ | |

**設計理由**: JWT(`raw_user_meta_data`)でのロール判定は改ざんリスクがあるため、必ず DB の `user_roles` テーブルを参照する。更新は `service_role key` を使うサーバサイドのみ。

### 3.3 user_profiles(ユーザープロフィール)

| 論理名 | 物理名 | 型 | 必須 | 説明 |
|---|---|---|:--:|---|
| プロフィールID | id | UUID | ○ | PK |
| ユーザーID | user_id | UUID | ○ | FK(users.id)、UK(1:1) |
| ハンドルネーム | handle_name | 文字列 | ○ | 1〜30文字 |
| 年齢 | age | 整数 | ○ | 13歳以上のみ登録可 |
| 現職・肩書き | current_job | 文字列 | - | 任意 |
| 経歴 | career_history | テキスト | - | 任意 |
| 現在の年収 | current_salary | 整数 | - | 万円単位、任意 |
| 目標年収 | target_salary | 整数 | - | 万円単位、任意 |
| スキル | skills | JSON | ○ | `["JavaScript","React",...]` |
| キャリア志向性 | career_orientation | 文字列 | ○ | `specialist` / `management` / `undecided` |
| プロフィール完了フラグ | profile_completed | 真偽値 | ○ | 初回登録完了後 true |
| 作成日時 | created_at | 日時 | ○ | |
| 更新日時 | updated_at | 日時 | ○ | |

### 3.4 chat_sessions(チャットセッション)

| 論理名 | 物理名 | 型 | 必須 | 説明 |
|---|---|---|:--:|---|
| セッションID | id | UUID | ○ | PK |
| ユーザーID | user_id | UUID | ○ | FK(users.id) |
| テーマ | theme | 文字列 | ○ | `job_change` / `skill_up` / `management` / `other` |
| タイトル | title | 文字列 | - | 最初のメッセージから自動生成(最大50文字) |
| 最終メッセージ日時 | last_message_at | 日時 | - | 一覧の並び順に使用 |
| 作成日時 | created_at | 日時 | ○ | |
| 更新日時 | updated_at | 日時 | ○ | |

### 3.5 chat_messages(チャットメッセージ)

| 論理名 | 物理名 | 型 | 必須 | 説明 |
|---|---|---|:--:|---|
| メッセージID | id | UUID | ○ | PK |
| セッションID | session_id | UUID | ○ | FK(chat_sessions.id) |
| 発言者ロール | role | 文字列 | ○ | `user` / `assistant` |
| 本文 | content | テキスト | ○ | 最大 4,000 文字(user) / 制限なし(assistant) |
| 提案ボタン | suggestion_buttons | JSON | - | `["強みを整理する","転職市場を調べる"]`。assistant のみ |
| 専門家カード表示フラグ | expert_card_shown | 真偽値 | - | AI が専門家紹介を表示した場合 true |
| 消費トークン数 | tokens_used | 整数 | - | assistant メッセージのみ |
| 作成日時 | created_at | 日時 | ○ | |

### 3.6 career_plans(キャリアプラン)

| 論理名 | 物理名 | 型 | 必須 | 説明 |
|---|---|---|:--:|---|
| プランID | id | UUID | ○ | PK |
| ユーザーID | user_id | UUID | ○ | FK(users.id) |
| タイトル | title | 文字列 | ○ | 最大100文字 |
| ゴール | goal | テキスト | ○ | キャリアの目標 |
| アクション項目 | action_items | JSON | - | `[{"text":"...","done":false}]` |
| 目標達成期限 | target_date | 日付 | - | 任意 |
| 元セッションID | source_session_id | UUID | - | FK(chat_sessions.id)。プランを作成した壁打きセッション |
| 作成日時 | created_at | 日時 | ○ | |
| 更新日時 | updated_at | 日時 | ○ | |

### 3.7 skill_inventories(スキル棚卸し)

| 論理名 | 物理名 | 型 | 必須 | 説明 |
|---|---|---|:--:|---|
| 棚卸しID | id | UUID | ○ | PK |
| ユーザーID | user_id | UUID | ○ | FK(users.id) |
| 経験 | experiences | テキスト | - | 最大2,000文字 |
| 強み | strengths | テキスト | - | 最大2,000文字 |
| 弱み | weaknesses | テキスト | - | 最大2,000文字 |
| 興味・やりたいこと | interests | テキスト | - | 最大2,000文字 |
| 働き方の希望 | work_style | テキスト | - | 最大2,000文字 |
| AI分析結果 | ai_analysis | テキスト | - | AI が生成した要約・提案 |
| 分析日時 | analyzed_at | 日時 | - | AI分析実行時刻 |
| 作成日時 | created_at | 日時 | ○ | |
| 更新日時 | updated_at | 日時 | ○ | |

### 3.8 usage_quotas(利用量クォータ)

| 論理名 | 物理名 | 型 | 必須 | 説明 |
|---|---|---|:--:|---|
| クォータID | id | UUID | ○ | PK |
| ユーザーID | user_id | UUID | ○ | FK(users.id) |
| 集計日 | quota_date | 日付 | ○ | JST 基準(UK: user_id + quota_date) |
| メッセージ数 | message_count | 整数 | ○ | 当日の送信数。上限 30 |
| 更新日時 | updated_at | 日時 | ○ | |

### 3.9 anon_sessions(匿名セッション)

| 論理名 | 物理名 | 型 | 必須 | 説明 |
|---|---|---|:--:|---|
| 匿名セッションID | id | UUID | ○ | PK |
| セッショントークン | session_token | 文字列 | ○ | UK。HttpOnly Cookie に保存する値 |
| テーマ | theme | 文字列 | - | 選択されたテーマ |
| メッセージ数 | message_count | 整数 | ○ | 0〜5。5到達でログイン誘導 |
| IP(ハッシュ) | ip_hash | 文字列 | - | 不正利用検知用 |
| 有効期限 | expires_at | 日時 | ○ | セッション開始から24時間後 |
| 作成日時 | created_at | 日時 | ○ | |

---

## 4. リレーション一覧

| 親エンティティ | 子エンティティ | カーディナリティ | 削除時の挙動 |
|---|---|---|---|
| users | user_roles | 1:1 | カスケード削除 |
| users | user_profiles | 1:1 | カスケード削除 |
| users | chat_sessions | 1:N | カスケード削除 |
| chat_sessions | chat_messages | 1:N | カスケード削除 |
| users | career_plans | 1:N | カスケード削除 |
| chat_sessions | career_plans(source) | 1:N | SET NULL(元セッション削除時) |
| users | skill_inventories | 1:N | カスケード削除 |
| users | usage_quotas | 1:N | カスケード削除 |
| users | api_usage_logs | 1:N | カスケード削除 |
| users | consent_logs | 1:N | カスケード削除 |
| users | audit_logs | 1:N | カスケード削除 |
| anon_sessions | anon_messages | 1:N | カスケード削除 |

---

## 5. 設計方針

| 項目 | 方針 |
|---|---|
| 主キー | 全テーブルで UUID v4 を採用(連番は推測されやすく外部公開 API に不向き) |
| 監査列 | 全テーブルに `created_at` を持たせる。更新があるテーブルは `updated_at` も必須 |
| 論理削除 | `users` のみ `deleted_at` + `scheduled_purge_at` で論理削除。その他テーブルは物理削除 |
| 文字コード | UTF-8(Supabase/PostgreSQL 標準) |
| JSON 列 | `skills`(配列)、`action_items`(配列)、`suggestion_buttons`(配列)は JSON 型で柔軟に扱う |
| RLS | 全テーブルで `user_id = auth.uid()` を基本ポリシーとして設定(security_policy.md §3.2 参照) |
| 退会時の完全削除 | `users.scheduled_purge_at` を起点に Supabase Edge Functions の Scheduled Trigger が全関連データを物理削除 |
| 匿名セッション有効期限 | 24時間で失効。定期 Scheduled Trigger でクリーンアップ |
