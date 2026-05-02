# API 一覧

| 項目 | 内容 |
|---|---|
| プロジェクト名 | engineer-career-ai |
| バージョン | 1.0 |
| 作成日 | 2026-05-02 |
| ベース URL | `/api/v1`(Next.js Route Handlers) |

※ 詳細仕様(リクエスト/レスポンスのスキーマ)は `03_detailed_design/api_specification.md`(OpenAPI 形式)で定義する。

---

## 1. 認証方式

| 項目 | 方針 |
|---|---|
| 方式 | Supabase Auth が発行する JWT を HttpOnly Cookie に保存 |
| Cookie 名 | Supabase Auth 標準(`sb-<projectRef>-auth-token`) |
| Server Actions/Route Handlers | `@supabase/ssr` の `createServerClient` で Cookie から自動取得 |
| トークン有効期限 | アクセストークン: 1時間 / リフレッシュトークン: 30日 |
| 認証エラー | 401 `ERR_AUTH_UNAUTHORIZED` |
| 認可エラー | 403 `ERR_AUTH_FORBIDDEN` |

---

## 2. API 一覧(全 25 エンドポイント)

### 2.1 認証(Authentication)

| API ID | メソッド | パス | 概要 | 認証 | 関連機能ID | 関連画面ID |
|---|:--:|---|---|:--:|---|---|
| API-001 | POST | `/api/v1/auth/signup` | メール+PWでサインアップ。reCAPTCHA 検証・13歳未満チェック含む | 不要 | F-001, F-035, F-041 | SC-002 |
| API-002 | POST | `/api/v1/auth/verify-email` | メール確認トークンの検証 | 不要 | F-001-2 | SC-004 |
| API-003 | POST | `/api/v1/auth/login` | メール+PWでログイン | 不要 | F-001-3 | SC-005 |
| API-004 | POST | `/api/v1/auth/password-reset/request` | パスワードリセットメールの送信 | 不要 | F-001-4 | SC-006 |
| API-005 | POST | `/api/v1/auth/password-reset/confirm` | 新パスワードの設定 | 不要 | F-001-4 | SC-007 |
| API-006 | POST | `/api/v1/auth/logout` | ログアウト(Cookie 削除・セッション無効化) | 必要 | F-001-5 | - |
| API-007 | GET | `/api/v1/auth/oauth/google` | Google OAuth 認証フロー開始 | 不要 | F-002 | SC-002, SC-005 |
| API-008 | GET | `/api/v1/auth/oauth/github` | GitHub OAuth 認証フロー開始 | 不要 | F-003 | SC-002, SC-005 |

### 2.2 プロフィール(Profile)

| API ID | メソッド | パス | 概要 | 認証 | 関連機能ID | 関連画面ID |
|---|:--:|---|---|:--:|---|---|
| API-009 | POST | `/api/v1/profile` | プロフィール初期登録(初回のみ) | 必要 | F-004, F-041 | SC-008 |
| API-010 | GET | `/api/v1/profile` | 自分のプロフィール取得 | 必要 | F-005 | SC-009 |
| API-011 | PATCH | `/api/v1/profile` | プロフィール更新 | 必要 | F-005 | SC-009 |
| API-012 | DELETE | `/api/v1/account` | 退会申請(`deleted_at` 設定、30日後完全削除予約) | 必要 | F-006 | SC-010 |

### 2.3 壁打ちチャット(Chat)

| API ID | メソッド | パス | 概要 | 認証 | 関連機能ID | 関連画面ID |
|---|:--:|---|---|:--:|---|---|
| API-013 | POST | `/api/v1/chat/guest` | 未ログイン壁打ち(SSE ストリーミング)。セッションID でレート制限 | 不要 | F-007, F-009, F-011, F-033 | SC-001 |
| API-014 | POST | `/api/v1/chat/sessions/:sessionId/messages` | ログイン後壁打ち(SSE ストリーミング)。履歴・プロフィール連動、自動保存 | 必要 | F-008, F-009, F-011, F-014, F-034 | SC-011 |
| API-015 | PATCH | `/api/v1/chat/sessions/:sessionId/theme` | チャットセッションのテーマ変更 | 必要(ゲストはCookie) | F-010 | SC-001, SC-011 |
| API-016 | GET | `/api/v1/chat/sessions` | 会話履歴一覧取得(ページネーション) | 必要 | F-015 | SC-012 |
| API-017 | GET | `/api/v1/chat/sessions/:sessionId` | 特定セッションのメッセージ取得(再開用) | 必要 | F-016 | SC-011 |
| API-018 | DELETE | `/api/v1/chat/sessions` | 会話履歴削除(単体: `/:sessionId` / 一括: クエリで複数ID指定) | 必要 | F-017 | SC-012 |

### 2.4 キャリアプラン(Career Plan)

| API ID | メソッド | パス | 概要 | 認証 | 関連機能ID | 関連画面ID |
|---|:--:|---|---|:--:|---|---|
| API-019 | POST | `/api/v1/career-plans` | キャリアプラン作成・保存 | 必要 | F-018 | SC-013 |
| API-020 | GET | `/api/v1/career-plans/:planId` | キャリアプラン詳細取得 | 必要 | F-019 | SC-014 |
| API-021 | PATCH | `/api/v1/career-plans/:planId` | キャリアプラン更新 | 必要 | F-019 | SC-014 |
| API-022 | GET | `/api/v1/career-plans` | キャリアプラン一覧取得 | 必要 | F-020 | SC-015 |

### 2.5 スキル棚卸し(Skill Inventory)

| API ID | メソッド | パス | 概要 | 認証 | 関連機能ID | 関連画面ID |
|---|:--:|---|---|:--:|---|---|
| API-023 | POST | `/api/v1/skill-inventories` | スキル棚卸しフォームの保存 | 必要 | F-021 | SC-016 |
| API-024 | POST | `/api/v1/skill-inventories/:inventoryId/analyze` | 保存済みフォームを AI が分析(SSE ストリーミング) | 必要 | F-022 | SC-017 |

### 2.6 同意管理(Consent)

| API ID | メソッド | パス | 概要 | 認証 | 関連機能ID | 関連画面ID |
|---|:--:|---|---|:--:|---|---|
| API-025 | POST | `/api/v1/consent` | Cookie 同意状態の記録 | 不要 | F-040 | 全画面(CMP-005) |

---

## 3. 共通仕様

### 3.1 レスポンス形式

**成功時:**
```json
{
  "success": true,
  "data": { ... }
}
```

**失敗時:**
```json
{
  "success": false,
  "error": {
    "code": "ERR_AUTH_UNAUTHORIZED",
    "message": "ログインが必要です"
  }
}
```

**ストリーミング(SSE)時:**
```
Content-Type: text/event-stream

data: {"type":"delta","content":"こんにちは"}
data: {"type":"delta","content":"！どんな"}
data: {"type":"suggestion","buttons":["強みを整理する","転職市場を調べる"]}
data: {"type":"done","sessionId":"xxx","messageId":"yyy"}
```

### 3.2 共通エラーコード

| HTTP ステータス | エラーコード | 意味 |
|:--:|---|---|
| 400 | `ERR_VALIDATION` | 入力値不正 |
| 401 | `ERR_AUTH_UNAUTHORIZED` | 未認証(ログイン必要) |
| 403 | `ERR_AUTH_FORBIDDEN` | 認可エラー(他人のリソース) |
| 404 | `ERR_NOT_FOUND` | リソース未存在 |
| 409 | `ERR_CONFLICT` | 競合(例: メールアドレス重複) |
| 422 | `ERR_AGE_RESTRICTION` | 13歳未満のため登録不可 |
| 429 | `ERR_RATE_LIMIT` | レート制限(メッセージ上限到達) |
| 503 | `ERR_BUDGET_EXCEEDED` | 月額APIコスト上限到達 |
| 500 | `ERR_INTERNAL` | サーバー内部エラー |

### 3.3 共通リクエストヘッダ

| ヘッダ | 必須 | 説明 |
|---|:--:|---|
| `Content-Type` | ○ | `application/json` |
| `Cookie` | △ | 認証必要 API で必須(自動付与) |
| `X-Request-Id` | - | リクエスト追跡 ID(任意、デバッグ用) |

### 3.4 ページネーション(一覧系 API)

**クエリパラメータ:**
- `page`: ページ番号(1始まり、デフォルト 1)
- `limit`: 件数(デフォルト 20、最大 50)
- `order`: `asc` or `desc`(デフォルト `desc`)

**レスポンス:**
```json
{
  "success": true,
  "data": [...],
  "pagination": {
    "page": 1,
    "limit": 20,
    "total": 42,
    "totalPages": 3
  }
}
```

### 3.5 SSE(ストリーミング)共通仕様

| API | エンドポイント |
|---|---|
| 未ログイン壁打き | API-013 |
| ログイン後壁打き | API-014 |
| スキル棚卸しAI分析 | API-024 |

**イベント種別:**

| type | 意味 | payload |
|---|---|---|
| `delta` | AI 応答テキストの差分 | `{ content: string }` |
| `suggestion` | ハイブリッドUI の提案ボタン | `{ buttons: string[] }` |
| `expert` | 専門家紹介カード表示指示 | `{ message: string, links: [...] }` |
| `done` | 応答完了 | `{ sessionId, messageId, tokensUsed }` |
| `error` | ストリーミング中エラー | `{ code, message }` |

---

## 4. レート制限仕様

| 対象 | 識別方法 | 上限 | 超過時 |
|---|---|---|---|
| 未ログインユーザー | Cookie の匿名セッション ID | 5 メッセージ / セッション | 401 + ログイン誘導 |
| ログインユーザー(無料) | `userId` | 30 メッセージ / 日(JST 0:00 リセット) | 429 |
| サインアップ | IP アドレス(ハッシュ) | 10 アカウント / 時 | 429 |
| PWリセット要求 | メールアドレス | 3 回 / 時 | 429 |

---

## 5. 外部 API 連携

| 連携先 | 用途 | API ID | 認証方式 | 備考 |
|---|---|---|---|---|
| Anthropic API | チャット応答生成・AI 分析 | API-013, API-014, API-024 | API キー(環境変数) | Infrastructure 層の AI アダプタ経由 |
| Google reCAPTCHA v3 | サインアップ・連続送信 Bot 対策 | API-001, API-013, API-014 | サイトキー+シークレット | サーバー側で verify API を必ず呼ぶ |
| Supabase Auth | OAuth(Google/GitHub)リダイレクト | API-007, API-008 | OAuth 2.0 | Supabase Auth が代行 |

---

## 6. Server Actions vs Route Handlers の使い分け

| 用途 | 実装方式 | 理由 |
|---|---|---|
| フォーム送信(サインアップ・プロフィール登録等) | Server Actions | フォームと同じファイルに定義でき、型安全 |
| SSE ストリーミング(チャット・AI 分析) | Route Handlers(`/api/v1/...`) | Server Actions は Streaming SSE に非対応 |
| OAuth リダイレクト(コールバック) | Route Handlers | GET リクエストのハンドリングが必要 |
| CRUD 操作(履歴削除・プラン更新等) | Server Actions | フォームアクションまたは `useTransition` と組み合わせて型安全 |
