# Component Methods — Pandora

**Note**: 詳細なビジネスルール（バリデーション規則、エラーケースの境界条件など）は Functional Design（Construction フェーズ）で定義する。本ドキュメントは **シグネチャと高レベル目的** に限定する。

---

## 1. features/auth

### AuthService
```ts
class AuthService {
  // Authorization ヘッダの JWT を検証して認証済みユーザーを返す
  getCurrentUser(req: Request): Promise<AuthUser | null>;

  // ユーザーIDが存在するかを確認（プロフィール表示や所有権チェックに使用）
  requireUserId(req: Request): Promise<string>; // 401 throws if missing
}
```

### CognitoAuthAdapter
```ts
verifyToken(token: string): Promise<JwtPayload>;  // Cognito JWKS でVerify
```

---

## 2. features/research

### ResearchService
```ts
class ResearchService {
  // 題材投稿 → AI による実験設計生成（同期）
  submitTopic(input: { userId: string; topic: string }): Promise<{ research: Research; spec: AIExperimentSpec }>;

  // 設計確定（Spec の取捨選択を反映して保存）
  startResearch(input: { researchId: string; userId: string; chosenSpec: ExperimentSpec }): Promise<Research>;

  // 公開タイムライン用の一覧
  listPublicResearches(input: { limit?: number; cursor?: string }): Promise<Page<Research>>;

  // 研究詳細
  getResearch(researchId: string): Promise<Research | null>;

  // 自分の進行中研究の一覧
  listMyResearches(userId: string): Promise<Research[]>;
}
```

### ResearchRepository
```ts
saveResearch(r: Research): Promise<void>;
getById(researchId: string): Promise<Research | null>;
listByUser(userId: string): Promise<Research[]>;
listPublic(limit: number, cursor?: string): Promise<Page<Research>>;
```

---

## 3. features/daily-record

### DailyRecordService
```ts
class DailyRecordService {
  // 日次記録の保存（同日重複時は上書き）
  saveRecord(input: { researchId: string; userId: string; date: string; values: CheckItemValue[] }): Promise<DailyRecord>;

  // 自分の研究の記録を時系列で取得
  listRecords(input: { researchId: string; userId: string }): Promise<DailyRecord[]>;

  // 画像アップロード用の署名付き URL を発行
  issueImageUploadUrl(input: { researchId: string; userId: string; contentType: string }): Promise<{ uploadUrl: string; objectKey: string }>;
}
```

### DailyRecordRepository
```ts
upsert(record: DailyRecord): Promise<void>;
listByResearchAndUser(researchId: string, userId: string): Promise<DailyRecord[]>;
listByResearchAllUsers(researchId: string): Promise<DailyRecord[]>; // グループ集計用
```

### ImageStorageAdapter
```ts
createPresignedPutUrl(input: { bucket: string; key: string; contentType: string; expiresInSec: number }): Promise<string>;
```

---

## 4. features/group

### GroupService
```ts
class GroupService {
  // 同題材で実施中の全研究と各日次記録を集約取得
  getGroupSnapshot(topicId: string): Promise<GroupSnapshot>;

  // コメント
  listComments(topicId: string, limit?: number): Promise<Comment[]>;
  postComment(input: { topicId: string; userId: string; targetType: 'topic' | 'record'; targetId: string; body: string }): Promise<Comment>;

  // リアクション（既存と同一なら取消し）
  toggleReaction(input: { topicId: string; userId: string; targetType: 'topic' | 'record'; targetId: string; kind: ReactionKind }): Promise<{ state: 'added' | 'removed' }>;
}
```

### CommentRepository
```ts
list(topicId: string, limit: number): Promise<Comment[]>;
save(c: Comment): Promise<void>;
```

### ReactionRepository
```ts
findByUserAndTarget(input: { userId: string; targetId: string; kind: ReactionKind }): Promise<Reaction | null>;
save(r: Reaction): Promise<void>;
delete(r: Reaction): Promise<void>;
countByTarget(targetId: string): Promise<Record<ReactionKind, number>>;
```

---

## 5. features/ai

### AIExperimentDesignService
```ts
class AIExperimentDesignService {
  // 題材から確からしさ + 実験設計（手順候補/スケジュール/チェック項目）を生成
  generateForTopic(input: { topic: string }): Promise<AIExperimentSpec>;
}
```

### BedrockClient
```ts
invokeModel(input: { modelId: string; prompt: string; maxTokens?: number }): Promise<string>; // text out
```

> プロンプト構築・出力パースは `AIExperimentDesignService` 内のヘルパーで実施（Q4 = B / 2層方針）。

---

## 6. lib/auth (Cross-cutting)

```ts
// middleware.ts から呼び出される
export async function verifyJwtFromRequest(req: NextRequest): Promise<{ userId: string } | { error: 'unauthorized' }>;
```

---

## 7. インターフェース型（抜粋）

```ts
type AuthUser = { userId: string; email?: string };

type Research = {
  id: string;
  ownerUserId: string;
  topicId: string;            // 同題材の集約用キー
  title: string;
  spec: ExperimentSpec;
  startedAt: string;          // ISO
  visibility: 'public';        // MVP は public 固定
};

type ExperimentSpec = {
  methods: { id: string; label: string; selected: boolean }[];
  schedule: { frequency: 'daily' | 'weekly'; timeOfDayHint?: string };
  measurements: string[];
  checkItems: CheckItem[];
};

type CheckItem =
  | { id: string; type: 'number'; label: string; unit?: string }
  | { id: string; type: 'select'; label: string; options: string[] }
  | { id: string; type: 'checkbox'; label: string }
  | { id: string; type: 'slider'; label: string; min: number; max: number }
  | { id: string; type: 'image'; label: string }
  | { id: string; type: 'text'; label: string };

type CheckItemValue = { itemId: string; value: number | string | boolean | { imageObjectKey: string } };

type DailyRecord = {
  researchId: string;
  userId: string;
  date: string;               // YYYY-MM-DD
  values: CheckItemValue[];
  createdAt: string;
};

type AIExperimentSpec = {
  evidence: { rating: 'strong' | 'moderate' | 'weak' | 'unknown'; summary: string; references?: string[] };
  spec: ExperimentSpec;
};

type Comment = { id: string; topicId: string; targetType: 'topic' | 'record'; targetId: string; userId: string; body: string; createdAt: string };
type ReactionKind = '👍' | '🔥' | '🤔' | '🎉';
type Reaction = { topicId: string; targetId: string; userId: string; kind: ReactionKind; createdAt: string };

type Page<T> = { items: T[]; nextCursor?: string };
type GroupSnapshot = { topicId: string; researches: Research[]; recordsByUser: Record<string, DailyRecord[]> };
```
