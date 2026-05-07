# Unit of Work — Pandora

**作成日**: 2026-05-07
**形態**: モノレポ（Q1=A, Q2=A）/ Greenfield

---

## 1. ユニット一覧

| Unit ID | 名称 | 種別 | 配置 | デプロイ先 |
|---------|------|------|------|----------|
| **U1** | Web App | Service（独立デプロイ可） | `pandora/` リポジトリのルート | **Vercel** |
| **U2** | AWS Infra | Service（独立デプロイ可） | `pandora/infra/` | **AWS（CDK deploy）** |

---

## 2. Unit 1: Web App

### 責務
- Pandora MVP の全 UI（Pages / 共通部品）
- Next.js Route Handlers による API 提供
- features/<domain>/ レイヤーで Service / Repository / Adapter を実装
- AWS リソース（DynamoDB / S3 / Bedrock / Cognito JWKS）への呼び出し

### 含むモジュール（features）
- features/auth
- features/research
- features/daily-record
- features/group
- features/ai

### 主要技術
- Next.js 14+ (App Router) / TypeScript / React Server Components
- Tailwind CSS + shadcn/ui
- AWS SDK v3（DynamoDB / S3 / Bedrock）
- TanStack Query（限定的）
- zod（schema validation）

### 配置（モノレポルート）

```
pandora/
├── app/                      ← Pages + API Route Handlers
├── components/
├── features/
├── lib/
├── middleware.ts
├── public/
├── package.json
├── tsconfig.json
├── tailwind.config.ts
├── next.config.mjs
└── infra/                    ← Unit 2 (CDK)
```

### デプロイ
- Vercel に Git push 連動で自動デプロイ
- 環境変数で Unit 2 が払い出した AWS リソース ID を受け取る（Q3=A）

---

## 3. Unit 2: AWS Infra

### 責務
- AWS リソースのプロビジョニングと構成管理（IaC）
- Cognito User Pool / DynamoDB Table / S3 Bucket / IAM Policies / Bedrock 利用設定
- リソース出力値（テーブル名・バケット名・User Pool ID 等）を CDK Output として公開

### 主要技術
- AWS CDK (TypeScript)
- AWS SDK / CDK constructs (`@aws-cdk/aws-cognito`, `@aws-cdk/aws-dynamodb`, `@aws-cdk/aws-s3`, etc.)

### 配置（モノレポ infra/）

```
pandora/
└── infra/
    ├── bin/pandora.ts                      ← CDK app entrypoint
    ├── lib/
    │   ├── pandora-stack.ts                ← メインスタック
    │   ├── constructs/
    │   │   ├── auth-construct.ts            ← Cognito
    │   │   ├── data-construct.ts            ← DynamoDB
    │   │   ├── storage-construct.ts         ← S3
    │   │   └── ai-construct.ts              ← Bedrock 関連 IAM
    │   └── shared/
    ├── cdk.json
    ├── package.json
    └── tsconfig.json
```

### デプロイ
- `cdk deploy` で AWS にプロビジョン
- CloudFormation Outputs で Unit 1 が必要とする値を出力（Q3=A: 環境変数として Vercel 側に手動 or スクリプトで設定）

---

## 4. ユニット間の契約

### 4.1 共有する値（Unit 2 → Unit 1）

| 値 | 例 | 経路 |
|----|----|------|
| Cognito User Pool ID | `ap-northeast-1_xxxxx` | Vercel 環境変数 `COGNITO_USER_POOL_ID` |
| Cognito App Client ID | `xxxxxxxxxxxx` | `COGNITO_CLIENT_ID` |
| Cognito JWKS URL | `https://cognito-idp.{region}.amazonaws.com/{poolId}/.well-known/jwks.json` | `COGNITO_JWKS_URL` |
| DynamoDB Table Name | `pandora-main` | `DDB_TABLE_NAME` |
| S3 Bucket Name | `pandora-images-xxxx` | `S3_BUCKET_NAME` |
| AWS Region | `ap-northeast-1` | `AWS_REGION` |
| Bedrock Model ID | `anthropic.claude-...:0` | `BEDROCK_MODEL_ID` |

### 4.2 認証情報

- Vercel から AWS への認証は **Vercel-AWS OIDC（推奨）または IAM ユーザーアクセスキー** （Infrastructure Design ステージで確定）

---

## 5. コード組織方針（Greenfield）

- **モノレポ単一 package.json なし**: Unit 1（ルート）と Unit 2（`infra/`）はそれぞれ独自の `package.json` を持つ。pnpm/yarn workspaces は MVP では採用しない（Q1=A シンプル優先）
- **TypeScript 設定**: それぞれ独立した `tsconfig.json`
- **共有型**: 当面は **共有しない**。Unit 1 内 `features/<domain>/types.ts` で完結。Unit 2 は AWS リソース型のみ扱う
- **CI**: GitHub Actions
  - PR 時: 両ユニットの lint / type-check
  - main マージ時: U1 = Vercel が自動デプロイ、U2 = `cdk deploy` を必要に応じて手動 or workflow で実行

---

## 6. 開発・実装順序（Q4=A）

1. **Phase A — Infra 立ち上げ**
   - Unit 2 で Cognito / DynamoDB / S3 / IAM の最小構成を `cdk deploy`
   - Output 値を Vercel 環境変数に登録
2. **Phase B — Web App スケルトン**
   - Unit 1 で Next.js プロジェクト作成 / 認証 middleware / 共通レイアウト
3. **Phase C — 機能実装**
   - 機能① マイ研究所 → ② 公開 → ③ グループ の順（要件 FR-2 → FR-3 → FR-4）
4. **Phase D — Bedrock 接続 / E2E 動作確認**
5. **Phase E — Build & Test / デモ準備**

---

## 7. ユニット境界の妥当性チェック

- [x] 各ユニットが独立してデプロイ可能
- [x] 責務が明確に分離（プレゼン+ロジック vs インフラ）
- [x] ユニット間契約が環境変数経由で疎結合
- [x] すべてのストーリー（US-01〜US-13）が U1 にマップ可能（後述の story-map 参照）
- [x] U2 は U1 の前提条件として独立して開発・デプロイ可能
