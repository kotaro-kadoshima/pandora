# Component Dependency — Pandora

---

## 1. 依存マトリクス（行 → 列：行が列に依存する）

|  | Auth | Research | DailyRecord | Group | AI | Repos | Adapters |
|--|:----:|:--------:|:-----------:|:-----:|:--:|:-----:|:--------:|
| **Pages (UI)** | ✓ | ✓ | ✓ | ✓ | - | - | - |
| **Route Handlers** | ✓ | ✓ | ✓ | ✓ | - | - | - |
| **AuthService** | - | - | - | - | - | - | CognitoAuthAdapter |
| **ResearchService** | - | self | - | - | ✓ | ResearchRepo | - |
| **DailyRecordService** | - | - | self | - | - | DailyRecordRepo | ImageStorageAdapter |
| **GroupService** | - | - | - | self | - | Research/Record/Comment/Reaction Repo | - |
| **AIExperimentDesignService** | - | - | - | - | self | - | BedrockClient |

横方向: 縦サービス間の参照は **Service Layer 内で発生（ResearchService → AIExperimentDesignService のみ）**。それ以外のサービス間直接依存は禁止。

---

## 2. データフロー図（題材投稿 → 実験設計生成）

```
┌────────┐  POST /api/research   ┌────────────────┐
│ Client │ ────────────────────▶ │ Route Handler  │
└────────┘                       └─────┬──────────┘
                                       │
                                       ▼
                       ┌──────────────────────────┐
                       │ AuthService              │
                       │  CognitoAuthAdapter ─▶JWKS│
                       └──────────────────────────┘
                                       │ userId
                                       ▼
                       ┌──────────────────────────┐
                       │ ResearchService          │
                       │ ┌──────────────────────┐ │
                       │ │ AIExperimentDesign   │ │
                       │ │ Service              │ │
                       │ │   └─▶ BedrockClient ─┼─┼──▶ AWS Bedrock
                       │ └──────────────────────┘ │
                       │ └─▶ ResearchRepo ────────┼──▶ DynamoDB
                       └──────────────────────────┘
                                       │
                                       ▼
                                  Response
```

## 3. データフロー図（日次記録 + 画像アップロード）

```
[Client]
   │ 1. POST /api/daily-records/upload-url { contentType }
   ▼
[Route Handler] → DailyRecordService.issueImageUploadUrl
                    └─▶ ImageStorageAdapter.createPresignedPutUrl ─▶ S3
   │ ◀───────── { uploadUrl, objectKey }
   │
   │ 2. PUT (uploadUrl) [画像本体]  ─────────────────────────────▶ S3
   │
   │ 3. POST /api/daily-records { researchId, date, values:[..., { itemId, value:{ imageObjectKey } }] }
   ▼
[Route Handler] → DailyRecordService.saveRecord
                    └─▶ DailyRecordRepo.upsert ─▶ DynamoDB
```

## 4. 依存ルール

- **逆流禁止**: Repository → Service の呼び出しは禁止
- **クロスドメイン参照は Service 経由**: `GroupService` が必要に応じて他ドメインの Repo を直接参照する例外は許容（凝集度のため）。これ以外は Service-to-Service もしくは型のみ参照
- **Cross-cutting** の `lib/auth/jwt.ts` は middleware からのみ呼び出す
- **テストの境界**: Service レイヤーで Repository / Adapter をモックして単体テスト可能な構造

---

## 5. ランタイム依存（外部）

| 内部 | 依存外部サービス |
|------|------------------|
| AuthService | AWS Cognito (JWKS) |
| ResearchRepository / DailyRecordRepository / CommentRepository / ReactionRepository | Amazon DynamoDB |
| ImageStorageAdapter | Amazon S3 |
| BedrockClient | Amazon Bedrock |

> Vercel から AWS への認証は Infrastructure Design ステージで確定（Vercel-AWS OIDC vs IAM ユーザー）。
