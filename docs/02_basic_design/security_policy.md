# セキュリティ方針

| 項目 | 内容 |
|---|---|
| プロジェクト名 | engineer-career-ai |
| バージョン | 1.1 |
| 作成日 | 2026-04-28 |
| 更新日 | 2026-05-02 |

---

## 0. 本書の位置付け

本書は要件定義書 §4.3「セキュリティ」を満たすための実装方針を定める。
- 個人情報を扱う一般公開Webサービス
- 13歳未満利用不可、18歳未満は保護者同意必要
- AI回答に起因するクレーム・訴訟リスクへの対応(免責の徹底)
- AI APIコスト爆発の防止(レート制限+CAPTCHA+月額上限)

---

## 1. 守るべき情報資産

| 資産 | 機密性 | 完全性 | 可用性 | 関連法規・備考 |
|---|:--:|:--:|:--:|---|
| ユーザー個人情報(メール/年齢/経歴/年収) | 高 | 高 | 中 | 個人情報保護法、改正個情法 |
| 認証情報(パスワードハッシュ / セッショントークン) | 高 | 高 | 高 | 漏洩時はインシデント |
| 会話履歴・キャリアプラン | 高 | 高 | 中 | キャリア相談という性質上、漏洩リスクは大 |
| AIプロンプト・APIキー | 高 | 高 | 高 | 漏洩=コスト直撃 |
| アクセスログ・監査ログ | 中 | 高 | 中 | 90日保管 |

---

## 2. 認証(Authentication)

### 2.1 認証方式

| 認証手段 | 採用 | 機能ID | 備考 |
|---|:--:|---|---|
| メール+パスワード | ○ | F-001, F-001-3 | Supabase Auth(emailパスワードプロバイダ) |
| Google OAuth 2.0 | ○ | F-002 | Supabase Auth(googleプロバイダ) |
| GitHub OAuth 2.0 | ○ | F-003 | Supabase Auth(githubプロバイダ) |
| 2要素認証(TOTP) | ✕(MVP外) | - | フェーズ2で検討。MVPは見送り(MAU 5想定で UX 重視) |

### 2.2 パスワードポリシー

| 項目 | 方針 |
|---|---|
| 最小長 | 12文字以上 |
| 文字種 | 英大文字・英小文字・数字・記号のうち**3種以上**を含む |
| 既知漏洩PWの拒否 | Supabase Auth の HaveIBeenPwned 連携を有効化(可能な場合) |
| ハッシュ | Supabase Auth 内部で bcrypt(運用上はマネージド任せ) |
| パスワード変更 | プロフィール画面から実施可(MVP対象、F-005 に内包) |
| パスワードリセット | F-001-4(メール送信→ワンタイムリンク) |

### 2.3 アカウントロック・ブルートフォース対策

| 項目 | 方針 |
|---|---|
| ログイン失敗 | 5回連続失敗で15分ロック(Supabase Auth の rate-limiting 設定で実現) |
| パスワードリセット要求 | 1分間に1回まで(Supabase 標準) |
| サインアップ | 1IPあたり10アカウント/時 を上限(リバースプロキシ層 or Supabase設定) |

### 2.4 セッション管理

| 項目 | 方針 |
|---|---|
| トークン形式 | Supabase Auth が発行する JWT |
| 配置場所 | HttpOnly Cookie(`@supabase/ssr` を使用) |
| Secure 属性 | 本番環境で必須 |
| SameSite | `Lax`(OAuth フローと両立) |
| アクセストークン有効期限 | 1時間 |
| リフレッシュトークン有効期限 | 30日(操作時に延長) |
| ログアウト | サーバ側でリフレッシュトークン無効化、Cookie 削除 |

### 2.5 メール確認・初期登録フロー

```mermaid
sequenceDiagram
    participant U as ユーザー
    participant N as Next.js
    participant S as Supabase Auth
    participant M as メール

    U->>N: サインアップ(email, password, 生年月日)
    N->>N: 13歳未満チェック<br/>18歳未満:保護者同意チェック
    N->>S: signUp(email, password)
    S->>M: 確認メール送信
    M-->>U: 確認リンク
    U->>N: 確認リンククリック
    N->>S: verifyOtp
    S-->>N: メール確認済セッション
    N-->>U: プロフィール初期登録画面へ誘導
```

---

## 3. 多層防御アーキテクチャ(4層 + Edge Functions)

### 3.0 全体マップ

```
ブラウザ
  ↓
【第1層】Vercel / Next.js（アプリ層）
  ├─ middleware: 未認証リダイレクト
  ├─ Server Action/Route Handler: user_roles テーブルで権限チェック
  └─ getUser() で認証状態を確認(getSession() は使わない)
  ↓
【第2層】Supabase API キー管理
  ├─ フロントエンド: anon key(RLS 適用)
  └─ サーバサイドのみ: service_role key(RLS バイパス、管理・バッチのみ)
  ↓
【第3層】RLS（行レベルセキュリティ）
  ├─ 全テーブルで ENABLE ROW LEVEL SECURITY 必須
  ├─ デフォルト全拒否ポリシー(deny_all)
  └─ 操作ごとに明示許可(SELECT/INSERT/UPDATE/DELETE)
  ↓
【第4層】PostgreSQL + PgBouncer（接続管理）
  └─ ポート 6543(PgBouncer 経由)で接続枯渇を防ぐ

【Edge Functions】(Supabase / Deno)
  └─ Vercel 60秒制限を超える重い処理を外部委譲
     例: スキル棚卸し AI 分析、キャリアプラン一括生成
```

---

### 3.1 第1層: アプリ層(Vercel / Next.js)

#### 3.1.1 認証状態の取得

```typescript
// ✅ 正しい: getUser()を使う(サーバーに毎回問い合わせ)
const { data: { user } } = await supabase.auth.getUser()

// ❌ 禁止: getSession()はJWTのローカル検証のみ(改ざんリスク)
// const { data: { session } } = await supabase.auth.getSession()
```

#### 3.1.2 権限チェック(user_roles テーブル)

```typescript
export async function DELETE(req: Request) {
  const { data: { user } } = await supabase.auth.getUser()
  if (!user) return Response.json({ error: '未認証' }, { status: 401 })

  // ✅ user_roles テーブルで権限確認(JWT の raw_user_meta_data は使わない)
  const { data: roleRow } = await supabase
    .from('user_roles')
    .select('role')
    .eq('user_id', user.id)
    .single()

  if (roleRow?.role !== 'admin') {
    return Response.json({ error: '権限なし' }, { status: 403 })
  }
  // 処理続行...
}
```

**禁止事項**:
- `raw_user_meta_data`(JWT内)で権限判定しない → 改ざん可能
- `userId` をクライアント送信パラメータから受け取らない → 必ずサーバでセッションから取得

#### 3.1.3 middleware によるルート保護

```typescript
// middleware.ts
export async function middleware(request: NextRequest) {
  const { user } = await supabase.auth.getUser()
  if (!user && request.nextUrl.pathname.startsWith('/(authed)')) {
    return NextResponse.redirect(new URL('/login', request.url))
  }
}
```

---

### 3.2 第2層: Supabase API キー管理

```typescript
// ✅ フロントエンド → anon key(RLS 適用、NEXT_PUBLIC_ 可)
const supabase = createClient(
  process.env.NEXT_PUBLIC_SUPABASE_URL!,
  process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!
)

// ✅ サーバサイドのみ → service_role key(RLS バイパス、管理・バッチのみ)
// infrastructure/supabase/admin-client.ts に閉じ込める
const supabaseAdmin = createClient(
  process.env.SUPABASE_URL!,
  process.env.SUPABASE_SERVICE_ROLE_KEY!  // ❌ NEXT_PUBLIC_ に置かない
)
```

**禁止事項**:
- `NEXT_PUBLIC_SUPABASE_SERVICE_ROLE_KEY` は絶対に作らない → クライアントに漏洩
- `supabaseAdmin` は `infrastructure/supabase/admin-client.ts` 内のみ。Service 層からも直接 import 禁止

---

### 3.3 第3層: RLS(行レベルセキュリティ)— 最後の砦

#### 3.3.1 基本方針

```sql
-- ① 全テーブルで必ず有効化
ALTER TABLE chat_sessions ENABLE ROW LEVEL SECURITY;

-- ② デフォルト全拒否(まずこれを置く)
CREATE POLICY "deny_all" ON chat_sessions USING (false);

-- ③ 操作ごとに明示許可(4操作を個別設定)
CREATE POLICY "select_own" ON chat_sessions
  FOR SELECT USING (auth.uid() = user_id);

CREATE POLICY "insert_own" ON chat_sessions
  FOR INSERT WITH CHECK (auth.uid() = user_id);

CREATE POLICY "update_own" ON chat_sessions
  FOR UPDATE USING (auth.uid() = user_id);

CREATE POLICY "delete_own" ON chat_sessions
  FOR DELETE USING (auth.uid() = user_id);
```

#### 3.3.2 View への対応

```sql
-- RLS はデフォルトで View に効かない → security_invoker 必須
CREATE VIEW safe_user_profiles
  WITH (security_invoker = true)
AS
  SELECT id, user_id, handle_name, career_orientation FROM user_profiles;
```

#### 3.3.3 RLS 自動有効化トリガー

新規テーブル作成時に RLS と `deny_all` ポリシーを自動適用:

```sql
CREATE OR REPLACE FUNCTION auto_enable_rls()
RETURNS event_trigger AS $$
DECLARE obj record;
BEGIN
  FOR obj IN SELECT * FROM pg_event_trigger_ddl_commands()
  WHERE command_tag = 'CREATE TABLE'
  LOOP
    EXECUTE format('ALTER TABLE %s ENABLE ROW LEVEL SECURITY', obj.object_identity);
    EXECUTE format('CREATE POLICY "deny_all" ON %s USING (false)', obj.object_identity);
  END LOOP;
END;
$$ LANGUAGE plpgsql;

CREATE EVENT TRIGGER auto_rls_trigger
  ON ddl_command_end
  WHEN TAG IN ('CREATE TABLE')
  EXECUTE FUNCTION auto_enable_rls();
```

#### 3.3.4 テーブル別 RLS ポリシー一覧

| テーブル | SELECT | INSERT | UPDATE | DELETE | 備考 |
|---|:--:|:--:|:--:|:--:|---|
| users | 自分のみ | Auth トリガー自動 | 自分のみ | ✕(退会は論理削除) | |
| user_roles | 自分のみ | ✕(サーバのみ) | ✕(サーバのみ) | ✕(サーバのみ) | service_role で管理 |
| user_profiles | 自分のみ | 自分のみ | 自分のみ | ✕(退会時 cascade) | |
| chat_sessions | 自分のみ | 自分のみ | 自分のみ | 自分のみ | |
| chat_messages | 親 session 所有者 | 親 session 所有者 | ✕ | 親 session 所有者 | |
| career_plans | 自分のみ | 自分のみ | 自分のみ | 自分のみ | |
| skill_inventories | 自分のみ | 自分のみ | 自分のみ | 自分のみ | |
| usage_quotas | 自分のみ | ✕(サーバのみ) | ✕(サーバのみ) | ✕ | service_role で upsert |
| api_usage_logs | 自分のみ | ✕(サーバのみ) | ✕ | ✕ | |
| consent_logs | 自分のみ | anon 可 | ✕ | ✕ | |
| audit_logs | 自分のみ | ✕(サーバのみ) | ✕ | ✕ | |
| anon_sessions | ✕ | ✕(サーバのみ) | ✕(サーバのみ) | ✕ | service_role のみ |
| anon_messages | ✕ | ✕(サーバのみ) | ✕ | ✕(サーバのみ) | service_role のみ |

---

### 3.4 第4層: PgBouncer(接続管理)

```typescript
// ✅ ポート 6543(PgBouncer 経由)を使用
// Supabase の接続文字列でポートを 5432 → 6543 に変更するだけ
const connectionString =
  'postgresql://user:pass@db.xxx.supabase.co:6543/postgres?pgbouncer=true'
```

**目的**: サーバレス環境(Vercel)は関数ごとに DB コネクションを張るため、PgBouncer なしでは接続が枯渇する。

---

### 3.5 Edge Functions(Supabase / Deno)

**目的**: Vercel の 60 秒制限を超える重い処理を Supabase Edge Functions に委譲する。

#### 3.5.1 対象処理

| 処理 | 理由 |
|---|---|
| スキル棚卸し AI 分析(API-024) | プロフィール全体を踏まえた長文生成で 60 秒超えの可能性 |
| キャリアプラン一括生成 | 複数ステップの AI 推論で処理時間が長い |
| 退会時の完全削除バッチ | `scheduled_purge_at` 到達ユーザーの全データ削除 |
| 匿名セッションのクリーンアップ | `expires_at` 超過の `anon_sessions` 定期削除 |

#### 3.5.2 フロー(Fire-and-Forget)

```
ブラウザ
  ↓ ① リクエスト
Next.js(Route Handler)
  ↓ ② Edge Function に投げる(await しない)
  ↓ ③ job_id を即返す(HTTP 202)
Supabase Edge Function(Deno)
  ↓ 重い処理をバックグラウンドで実行
  ↓ 結果を DB に保存
DB
  ↑ ④ ポーリング or WebSocket で結果を受け取る
ブラウザ
```

```typescript
// ✅ Vercel 側: await しない(投げっぱなし)
fetch(process.env.SUPABASE_EDGE_FUNCTION_URL + '/skill-analysis', {
  method: 'POST',
  headers: { Authorization: `Bearer ${process.env.SUPABASE_SERVICE_ROLE_KEY}` },
  body: JSON.stringify({ job_id, inventory_id })
})  // ← await しない

return Response.json({ job_id }, { status: 202 })

// ✅ Edge Function 側: waitUntil でバックグラウンド処理
Deno.serve(async (req) => {
  const { job_id, inventory_id } = await req.json()

  EdgeRuntime.waitUntil((async () => {
    // AI 分析(時間がかかる)
    const analysis = await generateAnalysis(inventory_id)
    await supabase.from('skill_inventories')
      .update({ ai_analysis: analysis, analyzed_at: new Date() })
      .eq('id', inventory_id)
    await supabase.from('jobs')
      .update({ status: 'done' })
      .eq('id', job_id)
  })())

  return new Response('ok')  // すぐ返す
})
```

#### 3.5.3 結果受け取り方式

| 方式 | 採用 | 理由 |
|---|:--:|---|
| ポーリング(DB polling) | ✅ MVP | シンプル実装。スキル分析は即時性より安定性 |
| WebSocket(Supabase Realtime) | フェーズ2 | 即時通知が必要になった時に追加 |
| SSE | ✕ | Vercel では使えない。Edge Function 側なら可だが複雑 |

---

### 3.6 ゲスト(未ログイン)の状態管理

- 未ログイン壁打ちは **匿名セッションID(httpOnly Cookie に保存)** でメッセージ数を管理
- セッションIDはサーバで発行し、クライアントから書き換え不可
- カウント管理: `anon_sessions.message_count` を service_role で upsert(Cookie 改ざんで突破不可)
- メッセージ内容: `anon_messages` テーブルに保存(将来ログイン後引き継ぎ機能に備える)

---

## 4. 通信

| 項目 | 方針 |
|---|---|
| プロトコル | HTTPS のみ(TLS 1.2 以上、推奨 1.3) |
| HTTP→HTTPS リダイレクト | Vercel 標準で有効 |
| HSTS | `Strict-Transport-Security: max-age=63072000; includeSubDomains; preload` |
| 証明書 | Vercel / Supabase のマネージド証明書(自動更新) |
| ミックスドコンテンツ | 全アセットを HTTPS で配信、CSP で `upgrade-insecure-requests` |

### 4.1 セキュリティヘッダ(Next.js middleware で設定)

| ヘッダ | 値 | 目的 |
|---|---|---|
| Content-Security-Policy | `default-src 'self'; img-src 'self' data: https://*.supabase.co; script-src 'self' https://www.google.com/recaptcha/ https://www.gstatic.com/recaptcha/ 'unsafe-inline'; connect-src 'self' https://*.supabase.co https://api.anthropic.com https://www.google.com/recaptcha/; frame-src https://www.google.com/recaptcha/;` | XSS, データ流出抑制 |
| X-Frame-Options | `DENY` | クリックジャッキング |
| X-Content-Type-Options | `nosniff` | MIMEスニッフィング |
| Referrer-Policy | `strict-origin-when-cross-origin` | Referer 漏洩抑制 |
| Permissions-Policy | `camera=(), microphone=(), geolocation=()` | 不要権限の遮断 |

CSP は reCAPTCHA を許可する都合で `script-src` に `unsafe-inline` を含む(reCAPTCHA v3 の動的スクリプト読み込みのため)。詳細設計時に nonce ベースに見直し可能。

---

## 5. データ保護

### 5.1 暗号化

| 種類 | 方針 |
|---|---|
| 通信時暗号化 | TLS(§4) |
| 保管時暗号化(at rest) | Supabase 標準で AES-256(マネージド) |
| 個別カラム暗号化 | MVP では追加せず、Supabase標準のat-rest暗号化に依存。フェーズ2で年収等の機微列に追加検討 |
| バックアップ暗号化 | Supabase の自動バックアップは標準で暗号化 |

### 5.2 個人情報の取り扱い(F-037 プライバシーポリシー対応)

| 項目 | 取得 | 必須/任意 | 削除可否 |
|---|---|---|---|
| メールアドレス | ○ | 必須 | 退会で削除 |
| ハンドルネーム | ○ | 必須 | 退会で削除 |
| 年齢(生年月日) | ○ | 必須 | 退会で削除 |
| 現職・経歴 | ○ | 任意 | プロフィール編集でも削除可 |
| 現年収・目標年収 | ○ | 任意 | プロフィール編集でも削除可 |
| スキル・志向 | ○ | 必須 | プロフィール編集で更新可 |
| 会話履歴 | ○ | 自動保存 | ユーザー個別/一括削除可 |
| 本名 / 住所 / 電話 | ✕ | 取得しない | — |
| IPアドレス | ○ | アクセスログ | 90日後自動削除 |

### 5.3 ログ・画面でのマスキング

| 場所 | マスキング方針 |
|---|---|
| アプリログ(Vercel Logs) | パスワード、認証トークン、API キー、ユーザー入力本文を出力しない |
| エラーレポート | スタックトレースは可、リクエストボディは個人情報マスク |
| 画面 | 年収は本人にのみ表示(他者画面は MVP 範囲では存在しない) |

### 5.4 データ削除(退会)

| フェーズ | 内容 |
|---|---|
| 退会申請(F-006) | `users.deleted_at` を設定、`users.scheduled_purge_at = now() + 30日` |
| 30日経過後 | バッチ(Supabase Edge Functions の Scheduled Trigger)で関連データを物理削除 |
| 削除対象 | users, user_profiles, chat_sessions, chat_messages, career_plans, skill_inventories, usage_quotas |
| 残すもの | 監査ログのうち IP / userId をハッシュ化したもの(法定保管要請対応) |

---

## 6. 入力検証

| 観点 | 対策 |
|---|---|
| SQLインジェクション | Supabase Client(パラメータバインディング)を Repository 層で必ず使用。生 SQL は基本禁止 |
| XSS | Next.js の自動エスケープに依存。`dangerouslySetInnerHTML` は原則禁止、AIの応答も Markdown レンダラ(react-markdown 等)経由で表示 |
| CSRF | Server Actions / Route Handlers は SameSite=Lax Cookie + Origin ヘッダ検証。GET 以外は Origin が自ドメインかチェック |
| Open Redirect | リダイレクト先パラメータは自ドメインのパスのみ許可(ホワイトリスト) |
| SSRF | サーバから外部 URL を fetch する箇所は Anthropic API / reCAPTCHA Verify のみ。動的な URL fetch は実装しない |
| プロンプトインジェクション | AI への入力は「ユーザーメッセージ」枠で渡し、システムプロンプトと分離。ユーザー入力に「あなたはこれから〜」等の指示があっても、システムプロンプトで「役割はキャリア相談AIに固定」と明示 |
| 大量入力 | メッセージ最大長(例: 4,000 文字)、添付不可、サーバ側で長さチェック |

### 6.1 入力長・形式制限(主要なもの)

| 項目 | 最大長 / 形式 |
|---|---|
| メールアドレス | 254 文字、RFC 5322 準拠 |
| パスワード | 12〜128 文字 |
| ハンドルネーム | 1〜30 文字、絵文字可、改行不可 |
| チャットメッセージ | 1〜4,000 文字 |
| キャリアプラン タイトル | 1〜100 文字 |
| キャリアプラン 本文 | 〜10,000 文字 |
| スキル棚卸し 各項目 | 〜2,000 文字 |

---

## 7. レート制限・コスト管理

要件定義書 R-001(AI APIコスト爆発)対策。多層防御。

### 7.1 機能ごとのレート制限

| 対象 | 上限 | 機能ID |
|---|---|---|
| 未ログインユーザー | 1セッションあたり 5メッセージ | F-007, F-033 |
| ログインユーザー(無料) | 1日あたり 30メッセージ | F-008, F-034 |
| サインアップ(同一IP) | 10アカウント / 時 | F-001 |
| パスワードリセット要求 | 3回 / 時(同一メール) | F-001-4 |
| reCAPTCHA score 閾値 | < 0.5 で拒否(または2要素認証要求) | F-035 |

### 7.2 月額APIコスト上限ガード(F-036)

| 項目 | 方針 |
|---|---|
| 月額予算 | 環境変数で設定(初期 $50 / 月) |
| 計測方法 | AI 呼び出し時に消費トークン数をDB記録(`api_usage_logs`)、月集計で予算判定 |
| 超過時の挙動 | 全AI機能を503エラーで一時停止、TOPに「API予算到達のため一時停止中」表示 |
| 復旧 | 翌月1日 0:00 JST で予算リセット(Supabase Edge Functions の Scheduled Trigger) |
| 警告 | 80%到達時に運用者にメール通知(Resend or Supabase Webhook) |

### 7.3 CAPTCHA(F-035)

| 場面 | 実装 |
|---|---|
| サインアップ | reCAPTCHA v3 を必須(score < 0.5 で拒否) |
| 連続短時間送信(チャット) | 1分間に5メッセージ以上で reCAPTCHA v3 を再評価 |
| パスワードリセット要求 | reCAPTCHA v3 を必須 |
| 検証 | サーバー側で `https://www.google.com/recaptcha/api/siteverify` を必ず叩く(クライアント検証だけでは不十分) |

---

## 8. ログと監査

### 8.1 ログ種別と保存期間

| ログ種別 | 保存場所 | 保存期間 | 内容 |
|---|---|:--:|---|
| アクセスログ | Vercel Logs(自動)+ 重要操作のみ DB(`audit_logs`) | 90日 | リクエスト URL, ステータス, IP(ハッシュ化), userId, タイムスタンプ |
| 認証ログ | Supabase Auth Logs | 90日 | ログイン成功/失敗, OAuth プロバイダ |
| 操作ログ | DB(`audit_logs`) | 90日 | 退会, 履歴削除, プロフィール変更 |
| API利用ログ | DB(`api_usage_logs`) | 90日 | 消費トークン, モデル名, userId, タイムスタンプ |
| エラーログ | Vercel Logs | 90日 | スタックトレース(個人情報マスク) |
| 同意ログ | DB(`consent_logs`) | 退会後30日まで | プラポリ・利用規約・Cookie 同意の取得時刻 |

### 8.2 ログに含めてはいけないもの

- パスワード(平文・ハッシュとも)
- 認証トークン(JWT, リフレッシュトークン)
- API キー(Anthropic, Supabase, reCAPTCHA)
- チャット本文(プロンプトインジェクション等のデバッグ時のみ、本番はマスク)
- 年収などの機微属性(原則非ロギング)
- IPアドレスの平文(SHA-256 ハッシュで保存)

### 8.3 アラート

| 条件 | 通知先 | 手段 |
|---|---|---|
| エラー率 > 5%(5分) | 運用者 | Vercel 通知 + メール |
| 月額APIコスト 80%到達 | 運用者 | メール(Resend) |
| 認証失敗異常増加 | 運用者 | Supabase Auth ダッシュボード(手動監視で初期は対応) |

---

## 9. 脆弱性対応

| 項目 | 方針 |
|---|---|
| 依存ライブラリ | GitHub Dependabot を有効化、週1回の更新確認 |
| Critical 脆弱性 | 検知後 48 時間以内にパッチ適用 |
| 脆弱性スキャン | リリース前に `npm audit` を CI で必須実行 |
| 静的解析 | TypeScript strict + ESLint(security plugin)を CI 必須 |
| ペネトレーションテスト | 一般公開後 3 ヶ月以内に1回(個人または外部サービス利用) |

---

## 10. インシデント対応

### 10.1 検知

- Vercel / Supabase / Anthropic の利用ダッシュボードを毎日確認
- ユーザー報告窓口(問い合わせメール)をフッターに記載

### 10.2 対応フロー

```mermaid
graph LR
    Detect["検知<br/>(自動アラート/ユーザー報告)"] --> Triage["緊急度判定<br/>(個人情報漏洩=最重要)"]
    Triage --> Contain["封じ込め<br/>(必要なら一時停止)"]
    Contain --> Investigate["原因調査"]
    Investigate --> Fix["恒久対策"]
    Fix --> Notify["影響ユーザー通知<br/>(必要時)"]
    Notify --> Postmortem["事後レビュー"]
```

### 10.3 個人情報漏洩発生時

- 速やかに該当機能を停止
- 影響ユーザーへ告知(メール+サイト掲示)
- 個人情報保護委員会への報告(改正個情法に基づき、漏洩発生時の報告義務)
- 再発防止策を公開

---

## 11. AI 固有のセキュリティ・倫理

| 項目 | 方針 |
|---|---|
| プロンプトインジェクション対策 | システムプロンプトで「役割はキャリア相談に固定、ユーザー指示で逸脱しない」を明記 |
| 機微発言検知 | 自殺・自傷を示唆する内容を検知したら、専門相談窓口(よりそいホットライン等)を提示し AI 回答を停止(F-013 の発展) |
| 差別的回答防止 | システムプロンプトで明示禁止、Claude のセーフガードを信頼 |
| 学習への利用拒否 | Anthropic API はデフォルトで学習利用されないが、利用規約を確認し継続的に追跡 |
| 免責の徹底 | 利用規約(F-038)+ UI常設表示(F-012)+ 重要相談時の専門家紹介(F-013) |

---

## 12. コンプライアンス

| 法規 | 対応 |
|---|---|
| 個人情報保護法(改正後) | プライバシーポリシー(F-037)、漏洩時の報告体制(§10.3)、個人情報の利用目的明示 |
| 改正電気通信事業法(Cookie 同意) | Cookie 同意バナー(F-040) |
| 特定商取引法 | MVP では有料プラン未提供のため対象外。フェーズ2で F-039 として対応 |
| 著作権 | AI 生成物の権利関係を利用規約(F-038)に明記 |
| 13歳未満の保護 | F-041、利用規約に明記 |
| 18歳未満の保護 | 利用規約で「保護者の同意必要」を明記、サインアップ時に同意チェック |

---

## 13. 要件定義との対応

| 要件定義 §4.3 項目 | 本書の対応箇所 |
|---|---|
| 認証(メール+PW / Google / GitHub) | §2 |
| パスワード保管(ハッシュ化) | §2.2(Supabase Auth に委譲) |
| セッション管理 | §2.4 |
| 個人情報の取り扱い | §5.2 |
| 通信暗号化 | §4 |
| 機密項目の保管時暗号化 | §5.1 |
| 監査ログ・保存期間 | §8.1 |
| アクセスログ 90日 | §8.1 |
| 退会後 30 日完全削除 | §5.4 |
| 会話履歴 無期限+ユーザー削除可 | §5.2, §5.4 |
| レート制限・CAPTCHA・月額API上限 | §7 |
| 13歳未満不可 | §12 + F-041 |
| 18歳未満 保護者同意 | §12 + F-038(利用規約) |
| AI 回答の免責 | §11 + F-012 + F-038 |
