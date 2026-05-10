---
name: update-readme
description: aidlc-docs 配下のドキュメント（要件・ユーザーストーリー・アプリケーション設計・Unit of Work 等）と idea.md・image/pandora-introduction1.png を参照し、Pandora プロジェクトのルート README.md を最新の状態に更新する。ドキュメント更新後の同期、機能追加・技術スタック変更・開発ステータス変更が発生したときに使う。
---

# update-readme — Pandora README 同期スキル

Pandora プロジェクトの `README.md` を、`aidlc-docs/` の正本ドキュメントと同期させるためのスキル。

## いつ使うか

- `aidlc-docs/` 配下のファイルが更新されたとき
- `idea.md` のコンセプトが書き換わったとき
- 開発フェーズ（Inception / Construction / Operations）が進んだとき
- 技術スタック・アーキテクチャ・ユニット構成が変わったとき
- ヒーロー画像（`image/pandora-introduction1.png` 等）を差し替えたとき

## 参照するファイル（優先度順）

1. **`idea.md`** — コンセプト・キャッチコピー・「ダメになる」設計思想・ピッチ
2. **`image/pandora-introduction1.png`** — ヒーロービジュアル（README 冒頭に配置）。画像内の文言（「人類は、研究をやめられない」「探究心中毒者のための、共同研究プラットフォーム」など）と機能アイコンの並びを README の概要・主要機能に反映する
3. **`aidlc-docs/aidlc-state.md`** — 現在のフェーズと進捗（Inception / Construction / Operations のチェックボックス）
4. **`aidlc-docs/inception/requirements/requirements.md`** — プロジェクト概要・機能要件（FR-1〜FR-6）・技術スタック・MVP スコープ
5. **`aidlc-docs/inception/user-stories/stories.md`** — ユーザーストーリー（機能の言い換えに使う）
6. **`aidlc-docs/inception/application-design/components.md`** — レイヤー構成・ディレクトリ構造
7. **`aidlc-docs/inception/application-design/unit-of-work.md`** — モノレポ構成・U1/U2 ユニット定義・デプロイ先
8. その他 `aidlc-docs/inception/application-design/` 配下（services / component-methods 等）は必要に応じて参照

## README の標準構成

以下のセクション順を維持する。情報がなければそのセクションは省略してよいが、勝手に順序を変えない。

1. **ヒーロー** — `<div align="center">` で `image/pandora-introduction1.png` を貼る + プロジェクト名 + キャッチコピー
2. **コンセプト見出し（画像の主要文言を H2 として再掲）** — 「人類は、研究をやめられない。」など
3. **概要本文** — `idea.md` のコアフィロソフィーを 2〜4 段落で要約。画像内の箇条書き（テーマを出し合う／仮説を立てる／検証して共有する／一緒に深掘りする）を含める
4. **主要機能（MVP）** — 画像下部の 5 アイコン（研究テーマ投稿／議論&フィードバック／仮説&実験記録／共同プロジェクト／成果を共有）に対応する表形式
5. **技術スタック** — `requirements.md` §5 をベースに表形式
6. **アーキテクチャ概要** — `unit-of-work.md` の U1/U2 構成 + `components.md` の主要ディレクトリ
7. **開発ステータス** — `aidlc-state.md` のチェックボックス状態を反映
8. **ドキュメント** — 主要ファイルへの相対リンク

## 更新手順

1. 上記参照ファイルを読み込む（変更されていそうなものから優先的に）
2. 既存 `README.md` を読む
3. **差分のあるセクションだけを Edit ツールで更新する**。全文書き換え（Write）は構成自体を変える時だけにする
4. 画像のキャッチコピーや機能アイコンが画像内のテキストと食い違っていないか確認する
5. 機能名・技術名は `requirements.md` の表記を正とする（例: 「Amazon DynamoDB」「AWS Bedrock」「Next.js Route Handlers」）
6. 開発ステータスのチェックボックスは `aidlc-state.md` の Stage Progress と完全一致させる

## スタイルルール

- 日本語。敬体ではなく常体に近いトーン（`idea.md` と統一）
- 表は Markdown table。絵文字は機能アイコン列など演出目的でのみ使う（過剰禁止）
- 内部リンクは相対パス（`./aidlc-docs/...`）
- README の長さは 100〜140 行程度を目安（読み手が一画面でスクロールしきれる量）
- 画像参照は HTML `<img>` で `width="100%"` 指定（GitHub での表示崩れ防止）

## トーン上の禁止事項

- **ハッカソンへの言及をしない**。MVP であることや提出先イベントは README に書かない（プロダクトとして自立した紹介にする）
- **他サービスを下げる比較をしない**。TikTok / SNS など実在サービスを引き合いに出して「あちらは〇〇で人をダメにする」のような対比は書かない。Pandora 自身の価値で語る

## やらないこと

- README に詳細な API 仕様や DB スキーマを書く（それらは `aidlc-docs/` 側の責務）
- `aidlc-docs/` 配下のファイルを書き換える（README の同期が目的であり、正本側を変えない）
- 未確定情報を断定的に書く（`requirements.md` §10 の未決事項は README に書かない）
