# Services — Pandora

サービス層は **Use Case 相当** の orchestration を担う。Route Handler は薄く、Service へ委譲する。

---

## 1. Service 一覧

| Service | 主な責務 | 依存 |
|---------|---------|------|
| `AuthService` | JWT 検証 / ユーザー識別 | CognitoAuthAdapter |
| `ResearchService` | 題材投稿 → AI 設計生成 / 設計確定 / 一覧 / 詳細 | ResearchRepository, AIExperimentDesignService |
| `DailyRecordService` | 日次記録の保存・取得 / 画像URL発行 | DailyRecordRepository, ImageStorageAdapter |
| `GroupService` | 同題材グループの集約 / コメント / リアクション | ResearchRepository, DailyRecordRepository, CommentRepository, ReactionRepository |
| `AIExperimentDesignService` | 題材の確からしさ + 実験設計を生成 | BedrockClient |

---

## 2. 主要オーケストレーション

### 2.1 題材投稿 → AI 実験設計生成（US-03）

```
[Client] → POST /api/research { topic }
            ↓
       [Route Handler]
            ↓
       AuthService.requireUserId(req)
            ↓
       ResearchService.submitTopic({ userId, topic })
            ├─→ AIExperimentDesignService.generateForTopic({ topic })
            │       └─→ BedrockClient.invokeModel(...)
            └─→ ResearchRepository.saveResearch(researchDraft)
            ↓
       Response { research, spec }
```

### 2.2 設計確定 → 研究開始（US-04）

```
POST /api/research/[id]/start
  → AuthService.requireUserId
  → ResearchService.startResearch({ researchId, userId, chosenSpec })
        → ResearchRepository.saveResearch (status: started)
```

### 2.3 日次記録（US-05, US-07）

```
POST /api/daily-records
  → AuthService.requireUserId
  → DailyRecordService.saveRecord({ ... })
        → DailyRecordRepository.upsert

POST /api/daily-records/upload-url
  → AuthService.requireUserId
  → DailyRecordService.issueImageUploadUrl
        → ImageStorageAdapter.createPresignedPutUrl
```

### 2.4 公開タイムライン / 詳細（US-08, US-09）

```
GET /api/research                → ResearchService.listPublicResearches
GET /api/research/[id]           → ResearchService.getResearch
                                   + DailyRecordService.listRecords (本人/題材所有者の記録)
```

> ゲストでも閲覧可。middleware.ts で公開エンドポイントは認証スキップ。

### 2.5 グループ集約 / コメント / リアクション（US-10, US-11, US-12）

```
GET /api/group/[topicId]
  → GroupService.getGroupSnapshot(topicId)
       ├─→ ResearchRepository.listByTopic(topicId)
       └─→ DailyRecordRepository.listByResearchAllUsers (per research)

POST /api/group/[topicId]/comments
  → AuthService.requireUserId
  → GroupService.postComment(...)

POST /api/group/[topicId]/reactions
  → AuthService.requireUserId
  → GroupService.toggleReaction(...)
```

---

## 3. Service 設計ルール

- **Route Handler は薄く保つ**: バリデーション (zod) → AuthService → 該当 Service 呼び出し → レスポンス整形のみ
- **Service は副作用を集約**: Repository / Adapter を経由し、横断的なエラー型 (`ServiceError`) を返す
- **AI 呼び出しは 30 秒タイムアウト**（NFR-1.3）。失敗時はリトライせずクライアントへ返却
- **`features/<domain>/services/index.ts`** を window として export
- **依存方向**: Route Handler → Service → Repository / Adapter（逆流禁止）
