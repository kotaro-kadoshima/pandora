# Components — Pandora

**作成日**: 2026-05-07
**スコープ**: Web App ユニット（Q8 = A）
**構造方針**: App Router + Feature-Sliced（Q1 = B）/ ドメイン別 (Q2 = A) / Repository パターン (Q3 = A)

---

## 0. レイヤー構成（俯瞰図）

```
┌──────────────────────────────────────────────────┐
│  app/                  (Pages / Layouts / RSC)   │  UI Layer
│  app/api/              (Route Handlers)          │  API Layer
├──────────────────────────────────────────────────┤
│  features/<domain>/services/  (Use Cases)        │  Service Layer
│  features/<domain>/repositories/                 │  Repository Layer
│  features/<domain>/types/, schemas/              │  Domain Models
├──────────────────────────────────────────────────┤
│  lib/aws/  (DynamoDB / S3 / Bedrock client)      │  Infra Adapter
│  lib/auth/ (Cognito JWT verifier)                │
│  middleware.ts                                   │  Cross-cutting
└──────────────────────────────────────────────────┘
```

---

## 1. UI コンポーネント (`app/`)

| コンポーネント | パス | 責務 |
|--------------|------|------|
| RootLayout | `app/layout.tsx` | 共通レイアウト・ヘッダ・テーマ |
| HomePage | `app/page.tsx` | 公開タイムライン（ゲスト/ログイン両対応） |
| ResearchDetailPage | `app/research/[id]/page.tsx` | 研究詳細（題材・実験設計・日次記録） |
| LoginPage | `app/(auth)/login/page.tsx` | ログイン画面 |
| SignupPage | `app/(auth)/signup/page.tsx` | サインアップ画面 |
| MyHomePage | `app/(authed)/my/page.tsx` | 自分の研究一覧 |
| NewResearchPage | `app/(authed)/my/research/new/page.tsx` | 題材投稿 + AI 設計確認 |
| MyResearchDetailPage | `app/(authed)/my/research/[id]/page.tsx` | 自分の研究の経過とグラフ |
| DailyRecordPage | `app/(authed)/my/research/[id]/record/page.tsx` | 日次記録フォーム |
| GroupPage | `app/(authed)/group/[topicId]/page.tsx` | 題材グループ一覧+比較 |

### 共通 UI 部品（`components/`）

| 部品 | 責務 |
|------|------|
| `<Header />` | ロゴ・ナビ・ログイン状態表示 |
| `<ResearchCard />` | タイムライン用カード |
| `<DailyRecordForm />` | チェック項目（数値/セレクト/...）の動的フォーム |
| `<ProgressChart />` | 数値項目のライングラフ |
| `<ImageGallery />` | 画像時系列ギャラリー |
| `<CommentList />` / `<CommentInput />` | コメント表示・投稿 |
| `<ReactionBar />` | 絵文字リアクションバー |
| `<AIExperimentSpecPreview />` | AI 生成された実験設計のプレビュー UI |

---

## 2. API コンポーネント (`app/api/`)

Route Handlers をドメイン別に配置（Q2 = A）。すべて `middleware.ts` で JWT 検証（Q5 = A）。

| Route | メソッド | 責務 |
|-------|---------|------|
| `/api/research` | POST | 題材投稿 + AI 実験設計生成 |
| `/api/research` | GET | 公開研究の一覧取得 |
| `/api/research/[id]` | GET | 研究詳細取得 |
| `/api/research/[id]/start` | POST | 設計を確定して研究を開始 |
| `/api/daily-records` | POST | 日次記録の登録/更新 |
| `/api/daily-records/upload-url` | POST | 画像アップロード署名付きURL発行（Q7 = A） |
| `/api/daily-records?researchId=` | GET | 自分の研究の記録一覧 |
| `/api/group/[topicId]` | GET | 同題材グループの記録一覧 |
| `/api/group/[topicId]/comments` | GET / POST | コメント一覧/投稿 |
| `/api/group/[topicId]/reactions` | POST | リアクション送信（トグル） |
| `/api/me` | GET | ログインユーザー情報 |

> 認証エンドポイント（サインアップ/ログイン）は Cognito Hosted UI もしくは AWS SDK 経由でクライアントが直接実行（必要に応じて `/api/auth/*` を追加）。

---

## 3. Feature モジュール（`features/<domain>/`）

各ドメインに **service / repository / types** をまとめる。

### 3.1 features/auth
- **AuthService**: トークン検証、現在ユーザー取得
- **CognitoAuthAdapter**: Cognito JWT 検証 (JWKS) / ユーザー情報取得

### 3.2 features/research
- **ResearchService**: 題材投稿、設計確定、公開取得
- **ResearchRepository**: DynamoDB の Research/Topic 永続化
- **types**: `Research`, `Topic`, `ExperimentSpec`

### 3.3 features/daily-record
- **DailyRecordService**: 日次記録の保存、読み取り、画像URL発行
- **DailyRecordRepository**: DynamoDB の DailyRecord 永続化
- **ImageStorageAdapter**: S3 署名付きURL 発行（Q7 = A）
- **types**: `DailyRecord`, `CheckItemValue`

### 3.4 features/group
- **GroupService**: 題材グループの集約取得 / コメント / リアクション
- **CommentRepository**, **ReactionRepository**
- **types**: `Comment`, `Reaction`

### 3.5 features/ai
- **AIExperimentDesignService**: 題材を入力に確からしさ + 実験設計を生成（ドメインサービス）
- **BedrockClient**: AWS Bedrock 呼び出し adapter（Q4 = B / 2層）
- **types**: `AIExperimentSpec`, `EvidenceSummary`

---

## 4. Cross-cutting (`lib/`)

| モジュール | 責務 |
|-----------|------|
| `lib/aws/dynamodb.ts` | DocumentClient のシングルトン提供 |
| `lib/aws/s3.ts` | S3 Client のシングルトン提供 |
| `lib/aws/bedrock.ts` | Bedrock Runtime Client のシングルトン提供 |
| `lib/auth/jwt.ts` | Cognito JWT verifier（JWKS キャッシュ） |
| `lib/db/single-table.ts` | DynamoDB 単一テーブル設計のキー生成ユーティリティ |
| `middleware.ts` | 認証必須パス（`/api/*` の保護対象 / `/(authed)/**`）に対する JWT verify |

---

## 5. 命名・配置のルール

- ファイル/フォルダ: kebab-case
- TypeScript の export: 名前付き export（default export は Pages のみ）
- スキーマ検証: zod を `features/<domain>/schemas.ts` に集約
- フロントの状態取得: Server Component で `await service.xxx()` を直接呼ぶ。クライアント主体画面は **TanStack Query** を限定的に利用（Q6 = A）
