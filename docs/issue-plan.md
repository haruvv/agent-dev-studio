# Issue計画草案

本ドキュメントは `docs/harness_engineering_mvp.md` を設計書として、
ai-agent-pipeline の実装issueを整理したもの。

---

## Issue 1: ラベルセットの定義と整備

### 目的
エージェントチームのバトンリレーを成立させるラベルを定義する。

### 背景
設計書3章のバトンリレー方式はラベル付与をトリガーとする。
ラベルが未定義だとワークフローが機能しない。

### 実装内容
以下のラベルをリポジトリに作成するスクリプトを用意する：
- `要件定義作成` — 要件定義エージェントのトリガー
- `詳細設計作成` — 詳細設計エージェントのトリガー
- `タスク分割` — タスク分割エージェントのトリガー
- `ready-for-impl` — 実装エージェントのトリガー
- `in-progress` — エージェント実行中
- `blocked` — エージェント失敗・依存未解決
- `ai-generated` — エージェント生成物の識別
- `ready-for-review` — レビュー待ち
- `review-passed` — レビュー完了（自動マージのゲート）

### 完了条件
- [ ] 上記ラベルがすべてリポジトリに存在する
- [ ] 各ラベルに説明と色が設定されている
- [ ] `scripts/setup-labels.sh` でコンシューマーリポジトリにも適用できる

### 対象ファイル / 想定影響範囲
- `scripts/setup-labels.sh`（新規）

### 制約 / 禁止事項
- ラベル名はワークフローのトリガー条件と完全一致させること

### 依存 Issue
なし

---

## Issue 2: IssueテンプレートとPRテンプレートの作成

### 目的
エージェントへの入力品質を担保する。

### 背景
設計書4.2より、エージェントの認知負荷を抑えるために構造化された入力が必要。
テンプレートがないと必須情報が欠けたまま実装が走るリスクがある。

### 実装内容
- `.github/ISSUE_TEMPLATE/implementation.yml` — 実装タスク用テンプレート（必須項目: 目的・背景・実装内容・完了条件・対象ファイル・制約・依存Issue）
- `.github/pull_request_template.md` — PR用テンプレート（必須項目: 変更概要・実施内容・テスト実施内容・懸念点・レビュー観点）

### 完了条件
- [ ] IssueテンプレートからIssueを作成できる
- [ ] PRテンプレートがGitHub UIで自動適用される

### 対象ファイル / 想定影響範囲
- `.github/ISSUE_TEMPLATE/implementation.yml`（新規）
- `.github/pull_request_template.md`（新規）

### 制約 / 禁止事項
- IssueテンプレートにはラベルをAUTO付与しない（依存Issue未解決のまま実装が走ることを防ぐ）

### 依存 Issue
Issue 1

---

## Issue 3: CLAUDE.md の作成（リポジトリ地図）

### 目的
エージェントがリポジトリ構造・行動原則を素早く把握できるようにする。

### 背景
設計書4.2より、CLAUDE.md は百科事典ではなく地図として機能させる必要がある。
エージェントの認知負荷を下げることが実装品質に直結する。

### 実装内容
`CLAUDE.md` を作成し、以下を記載する：
- リポジトリの目的と構成
- 各ワークフロー・エージェントの役割と呼び出し方
- エージェントが守るべき黄金原則（docs/への記録、blockedラベルの使い方等）
- 禁止事項

### 完了条件
- [ ] `CLAUDE.md` がリポジトリルートに存在する
- [ ] エージェントが読んで行動できる内容になっている

### 対象ファイル / 想定影響範囲
- `CLAUDE.md`（新規）

### 制約 / 禁止事項
- 情報を詰め込みすぎない。地図として機能する分量に留める

### 依存 Issue
なし

---

## Issue 4: 実装ワークフローの実装（④ claude-implement）

### 目的
`ready-for-impl` ラベルをトリガーにClaude Codeが実装からPR作成まで行うワークフローを構築する。

### 背景
設計書3章④に相当する中核ワークフロー。
Reusable Workflowと、コンシューマーリポジトリ用のcaller workflowの両方を実装する。

### 実装内容
1. `.github/workflows/claude-impl-reusable.yml`（Reusable Workflow）
   - `blocked` ラベルがある場合はスキップ
   - 実行中は `in-progress` ラベルを付与
   - `claude-code-action` でブランチ作成・実装・コミット
   - PR作成（`ai-generated` / `ready-for-review` ラベル付与）
   - 失敗時は `blocked` ラベルを付与
   - 完了時は `in-progress` ラベルを除去

2. `.github/workflows/claude-impl.yml`（Caller Workflow / コンシューマーリポジトリ用テンプレート）
   - `issues: labeled` イベント、`ready-for-impl` ラベルでトリガー
   - `claude-impl-reusable.yml` を `uses` で呼び出す

### 完了条件
- [ ] `ready-for-impl` ラベル付与でワークフローが起動する
- [ ] `blocked` ラベルがある場合はスキップされる
- [ ] 実行中に `in-progress` ラベルが付く
- [ ] PR が作成される
- [ ] 失敗時に `blocked` ラベルが付く

### 対象ファイル / 想定影響範囲
- `.github/workflows/claude-impl-reusable.yml`（新規）
- `.github/workflows/claude-impl.yml`（新規）

### 制約 / 禁止事項
- PR本文はインラインで定義する（コンシューマーリポジトリのファイル参照は不可）

### 依存 Issue
Issue 1, 2, 3

---

## Issue 5: CIワークフローの実装（⑥ CI）

### 目的
リンター・テストを自動実行し、マージ前の品質を担保する。

### 背景
設計書4.3より、制約はCIによって機械的に強制する。
エラーメッセージに修復手順を含めることでエージェントが自律的に修正できる。

### 実装内容
1. `.github/workflows/ci-reusable.yml`（Reusable Workflow）
   - `install_command` / `lint_command` / `test_command` / `build_command` をinputsで受け取る
   - 各コマンドが未設定の場合はスキップ（warningを出力）
   - エラーメッセージに修復ヒントを含める

2. `.github/workflows/ci.yml`（Caller Workflow / コンシューマーリポジトリ用テンプレート）
   - `pull_request` イベントでトリガー
   - `ci-reusable.yml` を `uses` で呼び出す

### 完了条件
- [ ] PRに対してCIが自動実行される
- [ ] lint/test/buildがそれぞれ独立して制御できる
- [ ] 未設定コマンドはスキップされる

### 対象ファイル / 想定影響範囲
- `.github/workflows/ci-reusable.yml`（新規）
- `.github/workflows/ci.yml`（新規）

### 制約 / 禁止事項
- Reusable Workflowとして定義する

### 依存 Issue
なし

---

## Issue 6: 自動レビューワークフローの実装（⑤ claude-auto-review）

### 目的
PR作成後にコードを自動レビューし、品質を担保する。

### 背景
設計書3章⑤に相当。指摘がある場合は実装エージェントへ差し戻す修正ループを実現する。

### 実装内容
1. `.github/workflows/claude-auto-review.yml`
   - `ready-for-review` ラベル付与でトリガー
   - レビューエージェント（`.claude/agents/reviewer.md`）を使いPRをレビュー
   - P1/P2指摘がある場合は実装エージェントへ差し戻し、修正ループを回す（上限: 3回）
   - 指摘なし・上限到達時は `review-passed` ラベルを付与

2. `.claude/agents/reviewer.md`（レビューエージェント定義）

### 完了条件
- [ ] `ready-for-review` ラベルでレビューが起動する
- [ ] P1/P2指摘時に差し戻しが行われる
- [ ] 指摘なし時に `review-passed` ラベルが付く
- [ ] 差し戻しループに上限（3回）がある

### 対象ファイル / 想定影響範囲
- `.github/workflows/claude-auto-review.yml`（新規）
- `.claude/agents/reviewer.md`（新規）

### 制約 / 禁止事項
- レビューループの上限回数を設ける（無限ループ防止）

### 依存 Issue
Issue 4, 5

---

## Issue 7: 自動マージワークフローの実装（⑥ claude-auto-merge）

### 目的
レビュー完了かつCI通過後にPRを自動マージし、開発フローを完結させる。

### 背景
設計書3章⑥に相当。CIのみをゲートとすると、レビュー未完でマージされるリスクがある。

### 実装内容
`.github/workflows/claude-auto-merge.yml`
- `review-passed` ラベル付与でトリガー
- CI全チェック通過を確認してからauto-mergeを有効化（`gh pr merge --auto`）
- マージ後にIssueをクローズ

### 完了条件
- [ ] `review-passed` ラベル付与でマージフローが起動する
- [ ] CIが通過していない場合はCI完了まで待機する
- [ ] マージ後にIssueがクローズされる

### 対象ファイル / 想定影響範囲
- `.github/workflows/claude-auto-merge.yml`（新規）

### 制約 / 禁止事項
- `review-passed` ラベルとCI通過の両方を満たすまでマージしない

### 依存 Issue
Issue 5, 6

---

## Issue 8: 要件定義ワークフローの実装（① claude-requirements）

### 目的
人間が起票した要求Issueに対し、エージェントが壁打ちを行い要件を確定する。

### 背景
設計書3章①に相当。上流フェーズが存在しないと「Issue起票からマージまで」のフローが成立しない。

### 実装内容
1. `.github/workflows/claude-requirements.yml`
   - `要件定義作成` ラベル付与でトリガー
   - 要件定義エージェント（`.claude/agents/requirements.md`）を起動
   - IssueにコメントしながらQ&Aを行い要件を確定
   - 完了後に `詳細設計作成` ラベルを付与
   - 失敗時は `blocked` ラベルを付与

2. `.claude/agents/requirements.md`（要件定義エージェント定義）

### 完了条件
- [ ] `要件定義作成` ラベルでエージェントが起動する
- [ ] Issue上で要件の壁打ちが行われる
- [ ] 完了後に `詳細設計作成` ラベルが付く

### 対象ファイル / 想定影響範囲
- `.github/workflows/claude-requirements.yml`（新規）
- `.claude/agents/requirements.md`（新規）

### 制約 / 禁止事項
- 要件が未確定のまま次フェーズへ進まない

### 依存 Issue
Issue 1, 3

---

## Issue 9: 詳細設計ワークフローの実装（② claude-detailed-design）

### 目的
確定した要件をもとに設計書を `docs/` 配下に作成する。

### 背景
設計書3章②に相当。設計書がないとタスク分割エージェントが動けない。
設計書は4.1に従いリポジトリ内に集約・バージョン管理する。

### 実装内容
1. `.github/workflows/claude-detailed-design.yml`
   - `詳細設計作成` ラベル付与でトリガー
   - 詳細設計エージェント（`.claude/agents/designer.md`）を起動
   - `docs/` 配下に設計書を作成してコミット
   - 完了後に `タスク分割` ラベルを付与
   - 失敗時は `blocked` ラベルを付与

2. `.claude/agents/designer.md`（詳細設計エージェント定義）

### 完了条件
- [ ] `詳細設計作成` ラベルでエージェントが起動する
- [ ] `docs/` 配下に設計書が作成される
- [ ] 完了後に `タスク分割` ラベルが付く

### 対象ファイル / 想定影響範囲
- `.github/workflows/claude-detailed-design.yml`（新規）
- `.claude/agents/designer.md`（新規）

### 制約 / 禁止事項
- 設計書は `docs/` 配下に保存する（外部サービス不可）

### 依存 Issue
Issue 1, 3, 8

---

## Issue 10: タスク分割ワークフローの実装（③ claude-task-split）

### 目的
設計書のPR分割計画をもとに実装用サブIssue群を自動生成する。

### 背景
設計書3章③に相当。タスク分割が自動化されることで実装フェーズへのバトンリレーが完成する。

### 実装内容
1. `.github/workflows/claude-task-split.yml`
   - `タスク分割` ラベル付与でトリガー
   - タスク分割エージェント（`.claude/agents/task-splitter.md`）を起動
   - `docs/` 配下の設計書を読み込みサブIssueを生成
   - 依存関係がある場合は `blocked` ラベルを付与した上で生成
   - 失敗時は `blocked` ラベルを付与

2. `.claude/agents/task-splitter.md`（タスク分割エージェント定義）

### 完了条件
- [ ] `タスク分割` ラベルでエージェントが起動する
- [ ] 設計書からサブIssueが自動生成される
- [ ] 依存関係のあるIssueは `blocked` ラベルが付く

### 対象ファイル / 想定影響範囲
- `.github/workflows/claude-task-split.yml`（新規）
- `.claude/agents/task-splitter.md`（新規）

### 制約 / 禁止事項
- 設計書が存在しない場合は処理を中断して `blocked` を付与する

### 依存 Issue
Issue 1, 3, 9
