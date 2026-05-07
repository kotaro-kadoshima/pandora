# Application Design Plan — Pandora

**Stage**: INCEPTION / Application Design
**Inputs**: requirements.md / stories.md / personas.md / execution-plan.md / idea.md

---

## 0. プラン概要

Pandora MVP の **コンポーネント・サービス層・依存関係** を設計する。
詳細なビジネスルールは後段の Functional Design（Construction フェーズ）で扱うため、本ステージでは責務分離・インターフェース・通信パターンの確定にフォーカスする。

---

## 1. 実行チェックリスト（PART 2 で消化）

- [x] フロントエンドのコンポーネント領域（Pages / UI部品）の境界定義
- [x] バックエンド（Next.js Route Handlers）のドメイン別モジュール定義
- [x] AWS リソースとの接続点（Service Adapter）定義
- [x] AI サービス抽象（Bedrock 呼び出し）の責務定義
- [x] サービス層（Use Case 相当）の orchestration 設計
- [x] コンポーネント依存マトリクスの作成
- [x] components.md / component-methods.md / services.md / component-dependency.md / application-design.md を生成

---

## 2. 暫定方針（質問への回答で確定）

- **アーキテクチャスタイル**: Next.js モノレポ（単一リポジトリ）/ レイヤード（UI / API / Service / Adapter / Domain Model）
- **モジュール境界**: ドメインで分割（auth / research / daily-record / group / ai-experiment）
- **データアクセス**: DynamoDB は **単一テーブル設計** を第一候補（PK/SK 設計はFunctional Designで詳細化）
- **API スタイル**: REST 風 Route Handlers（`/api/<domain>/...`）
- **FE データ取得**: Server Components 優先、必要に応じて TanStack Query をクライアント側で使用
- **AI 抽象**: `AIExperimentDesignService` がドメインを意識し、`BedrockClient` adapter を内部利用

---

## 3. 確認のための質問

### Q1. ディレクトリ構造の方針
Next.js プロジェクトのディレクトリ構造はどれを採用しますか？

- A) **App Router 標準** — `app/` 配下に画面と Route Handlers、`lib/` に共通ロジック
- B) **App Router + Feature-Sliced** — `app/` の下に画面、`features/<domain>/` 配下に各ドメインの service/repo/components 等を集約
- C) **Layered（src 分割）** — `src/{ui,api,services,repositories,domain}/` のように水平レイヤーで切る
- X) その他（自由記述）

[Answer]: 

---

### Q2. ドメインモジュールの粒度
バックエンド（Route Handlers + Service）のドメイン分割はどうしますか？

- A) **ドメイン別**: `auth / research / daily-record / group / ai`（推奨。要件と素直に対応）
- B) **エンティティ別**: User / Research / DailyRecord / Comment / Reaction を全部独立モジュールに
- C) **粗く**: `core / public / community` の 3 つでまとめる
- X) その他

[Answer]: 

---

### Q3. データアクセスパターン
DynamoDB へのアクセス層をどう設計しますか？

- A) **Repository パターン**: 各エンティティに `XxxRepository` を作り、Service から呼ぶ（推奨）
- B) **直接呼び出し**: Service から AWS SDK を直接叩く（最速だがテスト性低）
- C) **ORM / Mapper 層**（DynamoDB DocumentClient + 軽量 Mapper）
- X) その他

[Answer]: 

---

### Q4. AI サービス層の抽象度
Bedrock を含む AI 機能はどう抽象化しますか？

- A) **ドメインサービスのみ**: `AIExperimentDesignService` が直接 Bedrock SDK を呼ぶ
- B) **2層**: `AIExperimentDesignService`（ドメイン）→ `BedrockClient`（インフラ adapter）（推奨。差し替え容易）
- C) **3層**: 上記＋ promptビルダー / 出力パーサーを別モジュール化
- X) その他

[Answer]: 

---

### Q5. 認証連携の置き場所
Cognito JWT の検証はどこで行いますか？

- A) **Next.js Middleware**（`middleware.ts`）で API ルートを横断的に検証（推奨）
- B) 各 Route Handler 冒頭で個別に検証
- C) **NextAuth** を導入して Cognito Provider を使う
- X) その他（自由記述）

[Answer]: 

---

### Q6. フロントエンドの状態管理
グローバル状態管理は何を採用しますか？

- A) **Server Components 中心 + 必要箇所のみ TanStack Query** （推奨）
- B) Zustand を入れる
- C) Redux Toolkit
- D) なし（useState / useReducer のみで完結させる）
- X) その他

[Answer]: 

---

### Q7. 画像ストレージのアクセスパターン
S3 への画像アップロードはどう実装しますか？

- A) **署名付きURL方式**: クライアントが S3 に直接 PUT し、メタ情報のみ Route Handler 経由で保存（推奨。Vercel 帯域節約）
- B) **API 経由**: クライアント → Route Handler → S3 にプロキシ（実装は単純）
- X) その他

[Answer]: 

---

### Q8. ユニット境界と本ステージのスコープ
Workflow Planning では Unit を「Web App」と「AWS Infra」に分ける案を採用しました。本 Application Design では:

- A) **Web App ユニット中心** に設計し、Infra ユニットは Infrastructure Design 段階で詳細化（推奨）
- B) Web App / Infra の両方を本ステージで詳細にカバー
- X) その他

[Answer]: 

---

### Q9. その他要望
特記事項（設計上の制約、参考にしたい構造、絶対に避けたいもの等）があれば自由に記述してください。

[Answer]: 

---

## 4. PART 2 の生成物（質問回答後に作成）

- `aidlc-docs/inception/application-design/components.md`
- `aidlc-docs/inception/application-design/component-methods.md`
- `aidlc-docs/inception/application-design/services.md`
- `aidlc-docs/inception/application-design/component-dependency.md`
- `aidlc-docs/inception/application-design/application-design.md`（統合ドキュメント）
