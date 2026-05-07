# Unit of Work Plan — Pandora

**Stage**: INCEPTION / Units Generation
**Inputs**: requirements.md / stories.md / application-design.md / execution-plan.md

---

## 0. プラン概要

Workflow Planning で **2 ユニット構成（Web App / AWS Infra）** を提案し、Application Design でも前提として進めた。本ステージでは:

1. ユニット定義・責務・コード配置を確定
2. ユニット間依存マトリクスを作成
3. ストーリーをユニットへ割り当て

を実施する。

---

## 1. 実行チェックリスト（PART 2 で消化）

- [x] `unit-of-work.md` を生成（ユニット定義・責務・配置）
- [x] `unit-of-work-dependency.md` を生成（依存マトリクス）
- [x] `unit-of-work-story-map.md` を生成（ストーリー → ユニット割当）
- [x] Greenfield コード組織方針を `unit-of-work.md` に記載
- [x] `aidlc-state.md` を更新

---

## 2. 暫定方針（質問への回答で確定）

| 項目 | 暫定 |
|------|------|
| ユニット数 | 2（Unit 1: Web App / Unit 2: AWS Infra） |
| リポジトリ形態 | 単一リポジトリ（モノレポ） |
| ディレクトリ | `pandora/` 直下に Web App、`infra/` に CDK |
| デプロイ | Web App = Vercel、Infra = AWS（CDK deploy） |
| 共通型 | Web App 内の `features/<domain>/types.ts` に閉じる（CDK は AWS リソース型のみ扱う） |
| 開発順序 | Infra → Web App（リソース ID を Web App の env に流す） |

---

## 3. 確認のための質問

### Q1. リポジトリ形態
ユニットを格納するリポジトリの形態は？

- A) **単一リポジトリ（モノレポ）** — `pandora/` に Web App、`pandora/infra/` に CDK（推奨）
- B) **2 リポジトリに分離** — Web App と Infra を別レポに
- C) ワークスペース化（pnpm/npm workspaces で `apps/web` `infra` を管理）
- X) その他

[Answer]: 

---

### Q2. CDK のディレクトリ配置
モノレポ採用時、CDK プロジェクトはどこに置きますか？

- A) **`infra/`** を Web App と同階層に配置（推奨。シンプル）
- B) `apps/web` `apps/infra` のように `apps/` 配下に揃える
- X) その他

[Answer]: 

---

### Q3. ユニット間の契約（共有定義）
リソース ID（DynamoDB テーブル名 / S3 バケット名 / Cognito UserPool ID 等）の Web App への伝達方式は？

- A) **環境変数経由**（Vercel に CDK デプロイ後の出力値を設定。手動 or 自動化スクリプト）（推奨）
- B) AWS Secrets Manager / SSM Parameter Store を Web App から参照
- C) CDK の `cdk.out` から TypeScript 定義をエクスポートして Web App が import
- X) その他

[Answer]: 

---

### Q4. 開発・実装順序
2 ユニットの実装順序は？

- A) **Infra 先行 → Web App** — リソースを先に立てて開発（推奨）
- B) **Web App 先行（モック）→ Infra** — UI/UX を先に整える
- C) 並行（チーム分担前提）
- X) その他

[Answer]: 

---

### Q5. その他要望
特記事項（リポジトリ命名、追加ユニットの希望、絶対避けたい構造など）があれば自由記述。

[Answer]: 

---

## 4. PART 2 の生成物（質問回答後に作成）

- `aidlc-docs/inception/application-design/unit-of-work.md`
- `aidlc-docs/inception/application-design/unit-of-work-dependency.md`
- `aidlc-docs/inception/application-design/unit-of-work-story-map.md`
