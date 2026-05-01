# engineer-career-ai

エンジニアのキャリアを AI と壁打ちする Web システム。

## 特徴

- **未ログインで体験**: セッション内 5 メッセージまでログイン不要で壁打ち可能
- **ログイン後は履歴連動**: 過去のやり取り・プロフィールを踏まえた継続的なキャリア相談
- **エンジニア特化**: 技術系の文脈に最適化された AI 応答(Claude Haiku 4.5)
- **キャリアプラン管理**: AI 対話の結果を構造化して保存・編集
- **スキル棚卸し**: テンプレートに沿って経験を整理し AI が分析
- **求人サイト連携**: Findy / LAPRAS / Wantedly 等への導線

## ターゲット

- 若手エンジニア(キャリアを見つめ直したい)
- シニアエンジニア(技術スペシャリスト or マネジメント転向で悩んでいる)
- 転職エージェントの利害相反を避けて中立的な相談相手が欲しい人

## 技術スタック(計画)

| カテゴリ | 採用技術 |
|---|---|
| フロントエンド | Next.js 15 (TypeScript, App Router) |
| UIスタイル | Tailwind CSS |
| 認証 + DB | Supabase (Auth + PostgreSQL) |
| AI | Anthropic API (Claude Haiku 4.5) |
| ホスティング | Vercel |
| Bot対策 | Google reCAPTCHA v3 |

## アーキテクチャ

レイヤードアーキテクチャ(4層):

- **Presentation**: `src/app/`, `src/components/`(Next.js App Router)
- **Application/Service**: `src/services/`(ユースケース)
- **Domain**: `src/domain/`(エンティティ・値オブジェクト・Repository interface、純粋 TypeScript)
- **Infrastructure**: `src/infrastructure/`(Supabase / AI / Auth アダプタ)

依存方向: `Presentation → Service → Domain ← Infrastructure`(依存性逆転)

詳細は [`docs/02_basic_design/system_architecture.md`](docs/02_basic_design/system_architecture.md) を参照。

## 開発状況

| 工程 | 状態 | 成果物 |
|---|---|---|
| 1. 要件定義 | ✅ 完了(承認済み) | [`docs/01_requirements/system_requirements.md`](docs/01_requirements/system_requirements.md) |
| 2. 基本設計 | 🟡 進行中(3/6) | [`docs/02_basic_design/`](docs/02_basic_design/) |
| 3. 詳細設計 | ⏳ 未着手 | — |
| 4. 実装 | ⏳ 未着手 | — |
| 5. テスト | ⏳ 未着手 | — |

### 基本設計の進捗

- [x] system_architecture.md(システム構成)
- [x] function_list.md(機能一覧)
- [x] security_policy.md(セキュリティ方針)
- [ ] screen_design.md(画面設計)
- [ ] api_list.md(API 一覧)
- [ ] er_diagram.md(ER図)

## ディレクトリ構成

```
engineer-career-ai/
├── docs/
│   ├── 01_requirements/        # 要件定義
│   ├── 02_basic_design/        # 基本設計
│   ├── 03_detailed_design/     # 詳細設計(予定)
│   ├── 04_implementation/      # 実装メモ(予定)
│   └── 05_testing/             # テスト計画(予定)
├── src/                        # アプリケーションコード(実装フェーズで作成)
│   ├── app/                    # Next.js App Router
│   ├── components/             # UI コンポーネント
│   ├── services/               # Application/Service 層
│   ├── domain/                 # Domain 層
│   └── infrastructure/         # Infrastructure 層
├── tests/                      # テスト
├── public/                     # 静的アセット
├── README.md
└── .gitignore
```

## 設計駆動開発の方針

- 設計書を「正」とする(コードと設計書がズレたら設計書を先に直す)
- 工程を飛ばさない(要件 → 基本設計 → 詳細設計 → 実装 → テスト)
- 各工程の終わりにレビュー・承認を取ってから次工程へ

## ライセンス

未定(MVP リリース前に決定)
