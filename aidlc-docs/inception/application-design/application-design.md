# Application Design — Pandora（統合ドキュメント）

**作成日**: 2026-05-07
**スコープ**: Web App ユニット（Q8 = A）

本ドキュメントは下記 4 ファイルの内容をまとめた **統合ビュー** です。詳細は各個別ファイルを参照してください。

- [components.md](./components.md)
- [component-methods.md](./component-methods.md)
- [services.md](./services.md)
- [component-dependency.md](./component-dependency.md)

---

## 1. 設計方針サマリ

| 項目 | 採用 |
|------|------|
| ディレクトリ構造 | App Router + Feature-Sliced (`app/` + `features/<domain>/`) |
| ドメイン分割 | auth / research / daily-record / group / ai |
| データアクセス | Repository パターン |
| AI 抽象 | 2 層: AIExperimentDesignService → BedrockClient |
| 認証検証 | Next.js Middleware（Cognito JWT） |
| 状態管理 | Server Components 中心 + 必要時 TanStack Query |
| 画像 | S3 署名付きURL方式（クライアント直接 PUT） |
| 本ステージのスコープ | Web App ユニット中心。Infra は Infrastructure Design で詳細化 |

---

## 2. ディレクトリレイアウト（最終形イメージ）

```
pandora/
├── app/
│   ├── layout.tsx
│   ├── page.tsx                          # 公開タイムライン
│   ├── research/[id]/page.tsx
│   ├── (auth)/login/page.tsx
│   ├── (auth)/signup/page.tsx
│   ├── (authed)/
│   │   ├── my/page.tsx                   # 自分の研究一覧
│   │   ├── my/research/new/page.tsx
│   │   ├── my/research/[id]/page.tsx
│   │   ├── my/research/[id]/record/page.tsx
│   │   └── group/[topicId]/page.tsx
│   └── api/
│       ├── research/route.ts
│       ├── research/[id]/route.ts
│       ├── research/[id]/start/route.ts
│       ├── daily-records/route.ts
│       ├── daily-records/upload-url/route.ts
│       ├── group/[topicId]/route.ts
│       ├── group/[topicId]/comments/route.ts
│       ├── group/[topicId]/reactions/route.ts
│       └── me/route.ts
├── components/                          # 共通UI
│   ├── Header.tsx
│   ├── ResearchCard.tsx
│   ├── DailyRecordForm.tsx
│   ├── ProgressChart.tsx
│   ├── ImageGallery.tsx
│   ├── CommentList.tsx / CommentInput.tsx
│   ├── ReactionBar.tsx
│   └── AIExperimentSpecPreview.tsx
├── features/
│   ├── auth/
│   │   ├── services/auth-service.ts
│   │   ├── adapters/cognito-auth-adapter.ts
│   │   └── types.ts
│   ├── research/
│   │   ├── services/research-service.ts
│   │   ├── repositories/research-repository.ts
│   │   ├── schemas.ts
│   │   └── types.ts
│   ├── daily-record/
│   │   ├── services/daily-record-service.ts
│   │   ├── repositories/daily-record-repository.ts
│   │   ├── adapters/image-storage-adapter.ts
│   │   └── types.ts
│   ├── group/
│   │   ├── services/group-service.ts
│   │   ├── repositories/{comment,reaction}-repository.ts
│   │   └── types.ts
│   └── ai/
│       ├── services/ai-experiment-design-service.ts
│       ├── adapters/bedrock-client.ts
│       └── types.ts
├── lib/
│   ├── aws/{dynamodb,s3,bedrock}.ts
│   ├── auth/jwt.ts
│   └── db/single-table.ts
├── middleware.ts
├── package.json
└── tsconfig.json
```

---

## 3. 設計の整合チェック

### ストーリー × 主要コンポーネント

| Story | Pages | Routes | Services |
|-------|-------|--------|---------|
| US-01/02 認証 | (auth)/login,signup | /api/me | AuthService |
| US-03 題材投稿+AI生成 | my/research/new | POST /api/research | ResearchService + AIExperimentDesignService |
| US-04 設計確定 | my/research/new | POST /api/research/[id]/start | ResearchService |
| US-05 日次記録 | my/research/[id]/record | POST /api/daily-records | DailyRecordService |
| US-06 経過閲覧 | my/research/[id] | GET /api/daily-records | DailyRecordService |
| US-07 画像アップ | my/research/[id]/record | POST /api/daily-records/upload-url | DailyRecordService + ImageStorageAdapter |
| US-08 公開一覧 | / | GET /api/research | ResearchService |
| US-09 詳細閲覧 | /research/[id] | GET /api/research/[id] | ResearchService + DailyRecordService |
| US-10 グループ一覧 | (authed)/group/[topicId] | GET /api/group/[topicId] | GroupService |
| US-11 コメント | (authed)/group/[topicId] | POST /api/group/[topicId]/comments | GroupService |
| US-12 リアクション | (authed)/group/[topicId] | POST /api/group/[topicId]/reactions | GroupService |
| US-13 確からしさ | my/research/new | (US-03 と同じ) | AIExperimentDesignService |

すべての Must/Should ストーリー（US-01〜US-13）が、コンポーネント・Route・Service にマッピング可能であることを確認。

---

## 4. 後段ステージへの引き継ぎ事項

- **Functional Design (Construction)**: 各 Service / Repository のビジネスルール詳細化、エラーケースのバリデーション、エッジケース、AI プロンプト設計
- **NFR Requirements/Design**: AI 30 秒タイムアウト、画像最大サイズ、レスポンシブ、最低限のセキュリティ
- **Infrastructure Design**: DynamoDB 単一テーブル設計の PK/SK 詳細、Cognito User Pool 設定、S3 バケット + CORS、Bedrock IAM、Vercel-AWS OIDC、CDK スタック分割

---

## 5. 設計の限界・前提

- 単一の Web App ユニットとして設計しており、マイクロサービス化は MVP では行わない
- DynamoDB 単一テーブルの具体的キー設計は Functional/Infrastructure Design で確定
- Vercel ↔ AWS の認証方式は未確定（Infrastructure Design で決定）
- AI 出力の品質は実装フェーズで検証（試行錯誤前提）
