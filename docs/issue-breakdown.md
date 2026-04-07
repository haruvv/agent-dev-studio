# MVP 実装 Issue 分割計画

設計書（`docs/design.md`）に基づき、MVPを GitHub Issue 単位に分割する。
各 Issue は「1 Worker が 1 PR で完結できる粒度」を原則とする。

---

## 対象リポジトリ

| リポジトリ | 用途 |
|---|---|
| `haruvv/dev-agents` | ワークフロー雛形・ラベル定義・テンプレート類 |
| `haruvv/agent-intake` | agent-intake Lambda・Orchestrator Lambda・インフラ |

---

## 実装順序と依存関係

```
[フェーズA] dev-agents テンプレート整備
    ↓
[フェーズB] 基盤インフラ（SQS / DynamoDB / ECS）
    ↓
[フェーズC] agent-intake Lambda
    ↓
[フェーズD] ECS Worker（基盤 → 各 Worker）
    ↓
[フェーズE] Orchestrator Lambda
    ↓
[フェーズF] 結合・E2E 検証
```

フェーズA と フェーズB は並行着手可能。
フェーズC は フェーズB（API Gateway・Lambda 実行環境）が整い次第着手可能。
フェーズD は フェーズB（SQS・ECS 基盤・State DB）が整い次第着手可能。

---

## フェーズA：dev-agents テンプレート整備

`haruvv/dev-agents` に配置するワークフロー雛形・ラベル・ドキュメントテンプレートを整備する。

### A-1：トリガーワークフロー雛形

**内容**
- 以下の trigger workflow ファイルを `haruvv/dev-agents` に追加する
  - `requirements-trigger.yml`（ラベル `要件定義作成` 検知 → SQS 送信）
  - `design-trigger.yml`（ラベル `詳細設計作成` 検知 → SQS 送信）
  - `task-split-trigger.yml`（ラベル `タスク分割` 検知 → SQS 送信）
  - `implement-trigger.yml`（ラベル `ready-for-impl` 検知 → SQS 送信）
- SQS 送信メッセージ形式：`{ job_type, issue_number, repo }`

**完了条件**
- actionlint でワークフロー構文が通ること
- ラベル名が `docs/design.md` § 3.1 のラベル定義と一致すること

---

### A-2：CI ワークフロー雛形

**内容**
- `ci.yml` を `haruvv/dev-agents` に追加する
  - トリガー：`pull_request: branches: [main]` / `push: branches: [main]`
  - MVP 初期 CI：actionlint による構文チェックのみ
- repair Worker が PR ブランチに push した場合に CI が再実行されること（PR synchronize で自動発火）

**完了条件**
- actionlint が通ること

---

### A-3：ラベル定義・bootstrap スクリプト

**内容**
- `labels.yml` にラベル名・色・説明を定義する（Issue ラベル + PR ラベル）
  - `docs/design.md` § 3.1・§ 3.2 記載のラベルを網羅する（将来フェーズ分も定義しておく）
- `scripts/bootstrap.sh` を作成する
  - `gh label create` で `labels.yml` の内容を対象 repo に一括作成する

**完了条件**
- 手動実行で対象 repo にラベルが作成されること

---

### A-4：ドキュメント雛形・AGENTS.md テンプレート

**内容**
- `templates/docs/requirements.md` および `templates/docs/design.md` の雛形を作成する
  - `docs/design.md` § 4.1・§ 4.2 に記載のフォーマットに従う
- `templates/AGENTS.md`（または `CLAUDE.md`）の初期テンプレートを作成する
  - Worker がリポジトリの文脈・制約を把握するための案内文書
  - 主要な参照先（`docs/`、ラベル一覧、Issue 起票先）を記載する

**完了条件**
- テンプレートが `/create` 時にコピーできる形式になっていること

---

## フェーズB：基盤インフラ

`haruvv/agent-intake` に IaC（Terraform または CDK）を配置し、共有インフラを構築する。

### B-1：DynamoDB State DB

**内容**
- テーブル名：`agent-state`
- パーティションキー：`job_id`（String）
- 属性：`docs/design.md` § 2.5 記載の全フィールドを定義する
  - `job_id`, `issue_number`, `parent_issue_number`, `repo`, `job_type`, `status`, `pr_number`, `branch_name`, `head_sha`, `workflow_run_id`, `ci_attempt`, `repair_attempt`, `created_at`, `updated_at`
- TTL 属性を設定し、`done` / `failed` レコードを自動削除できるようにする

**完了条件**
- テーブルが作成され、Lambda / ECS タスクから読み書きできること

---

### B-2：SQS キュー + EventBridge Pipes

**内容**
- SQS キュー（標準キュー）を作成する
  - Dead Letter Queue を設定する（最大受信回数：3）
- EventBridge Pipes でキューをポーリングし、ECS Fargate タスクを起動する設定を行う
  - メッセージボディをタスクのコンテナ環境変数に渡す

**完了条件**
- テストメッセージを SQS に送信すると ECS タスクが起動すること

---

### B-3：ECS Fargate クラスター・タスク定義 基盤

**内容**
- ECS クラスターを作成する
- Worker 共通のタスク定義を作成する
  - Claude Code が実行できる Docker image（`claude` CLI を含む）
  - 必要な IAM ロール（DynamoDB・SQS・Secrets Manager アクセス）
  - CloudWatch Logs へのログ出力設定
- Secrets Manager に以下を登録する
  - `GITHUB_TOKEN`（PAT: MVP暫定）
  - `CLAUDE_CODE_OAUTH_TOKEN`

**完了条件**
- タスクが起動し、CloudWatch Logs にログが出力されること

---

## フェーズC：agent-intake Lambda

`haruvv/agent-intake` に Discord Bot の Lambda 関数を実装する。

### C-1：Lambda handler（Discord ACK）

**内容**
- `agent-intake-handler` Lambda を実装する
  - Discord の Ed25519 署名検証
  - type 1 → PONG 即時返答
  - type 2 → type 5 ACK 返答 + `agent-intake-processor` を非同期起動
- API Gateway（HTTP API）と Lambda を接続する
- EventBridge ルールで5分ごとに ping してウォームアップを維持する

**環境変数**：`DISCORD_PUBLIC_KEY`, `PROCESSOR_FUNCTION_NAME`

**完了条件**
- Discord Developer Portal から Interactions Endpoint URL として登録できること
- /issue コマンド送信時に type 5 ACK が3秒以内に返ること

---

### C-2：Lambda processor（LLM 解釈・Issue 起票）

**内容**
- `agent-intake-processor` Lambda を実装する
  - Gemma API で自然言語を解釈する
  - 曖昧な場合は Discord に確認メッセージを返信する
  - 明確な場合は対象 app repo に GitHub Issue を起票し、`要件定義作成` ラベルを付与する
  - Discord に Issue URL を返信する
  - processor の Lambda 非同期リトライは 0 回に設定する（重複起票防止）
- `/issue [<repo>] <description>` のコマンド解釈を実装する
  - repo 省略時は `DEFAULT_GITHUB_REPO` に起票する

**環境変数**：`GEMMA_API_KEY`, `GITHUB_TOKEN`, `DEFAULT_GITHUB_REPO`, `STATE_TABLE_NAME`

**完了条件**
- Discord で `/issue` コマンドを送ると対象 repo に Issue が起票されること
- 曖昧な入力で確認メッセージが返ること

---

## フェーズD：ECS Worker

Worker 基盤を実装してから、各ジョブ種別の Worker を実装する。

### D-1：Worker 基盤（エントリーポイント・State DB 連携）

**内容**
- ECS タスクのエントリーポイントスクリプトを実装する
  - 環境変数からジョブメッセージ（`job_type`, `issue_number`, `repo` 等）を取得する
  - State DB で `job_id` を `in-progress` に更新する（二重起動防止）
  - `job_type` に応じてジョブハンドラーを呼び出す
  - 完了・失敗時に State DB を更新する
- `blocked` ラベルチェックの共通ユーティリティを実装する
- GitHub API クライアント（ラベル付与・コメント投稿・Issue 起票）の共通実装

**完了条件**
- SQS 経由でタスクが起動し、State DB が更新されること
- `blocked` ラベルがある場合に処理がスキップされること

---

### D-2：requirements Worker

**内容**
- `job_type: requirements` を処理する Worker を実装する
  - `docs/design.md` § 4.1 の処理フローに従う
  - Issue 本文・コメント履歴を読み込み、LLM で要件整理
  - 曖昧な場合：質問コメント投稿 → `waiting-for-answer` ラベル付与
  - 確定できる場合：`docs/<issue-number>-requirements.md` を生成してコミット → `詳細設計作成` ラベル付与（PAT）
  - 質問往復上限：2往復

**完了条件**
- requirements.md が対象 app repo にコミットされること
- 次フェーズのラベル付与で GitHub Actions が起動すること

---

### D-3：design Worker

**内容**
- `job_type: design` を処理する Worker を実装する
  - `docs/design.md` § 4.2 の処理フローに従う
  - `docs/<issue-number>-requirements.md` を読み込み、設計書を生成
  - `docs/<issue-number>-design.md` にコミット → `タスク分割` ラベル付与（PAT）

**完了条件**
- design.md が対象 app repo にコミットされること

---

### D-4：task-split Worker

**内容**
- `job_type: task-split` を処理する Worker を実装する
  - `docs/design.md` § 4.3 の処理フローに従う
  - 設計書の作業分割方針を読み込み、子 Issue を起票する
  - 各子 Issue に `ready-for-impl` ラベルを付与（PAT）
  - State DB に `parent_issue_number` を記録する

**完了条件**
- 子 Issue が起票され、各 Issue に `ready-for-impl` が付与されること

---

### D-5：implement Worker

**内容**
- `job_type: implement` を処理する Worker を実装する
  - `docs/design.md` § 5.1 の処理フローに従う
  - 対象 app repo をクローン
  - `impl/issue-<番号>-<slug>` ブランチを作成・プッシュ。State DB に `branch_name` を記録
  - `docs/<親issue番号>-design.md` を参照して Claude Code で実装
  - コミット・プッシュ後、State DB に `head_sha` を記録
  - PR 作成（`ai-generated` ラベル付与）。State DB に `pr_number` を記録
  - `ready-for-human-review` ラベルを付与（PAT）

**完了条件**
- PR が作成され、CI が起動すること
- State DB に `branch_name`・`head_sha`・`pr_number` が記録されること

---

### D-6：repair Worker

**内容**
- `job_type: repair` を処理する Worker を実装する
  - `docs/design.md` § 5.2 の処理フローに従う
  - `repair_attempt` を確認（上限3回）
  - PR ブランチをクローン
  - `head_sha` と `workflow_run_id` で CI 失敗ログを取得
  - Claude Code でエラーを修正・コミット・プッシュ。State DB の `head_sha` を更新
  - PR synchronize により CI が自動再実行される

**完了条件**
- 修正 push 後に CI が再実行されること
- 上限超過で `needs-human` ラベルが付与されること

---

## フェーズE：Orchestrator Lambda

`haruvv/agent-intake` に Orchestrator を実装する。

### E-1：Orchestrator Lambda（Webhook 受信・イベントルーティング）

**内容**
- `/webhook` エンドポイントを持つ Lambda を実装する（`agent-intake` と別関数）
  - GitHub Webhook の HMAC 署名検証（`GITHUB_WEBHOOK_SECRET`）
  - 受信イベントを `workflow_run` / `pull_request` でルーティングする
- `pull_request` イベントの処理
  - opened：`pr_number`・`head_sha` を State DB に反映
  - synchronize：最新 `head_sha` を State DB に反映
  - closed：State DB を終了系ステータスに更新

**完了条件**
- GitHub Webhook が届き、State DB が更新されること

---

### E-2：CI 結果処理・repair ジョブ投入

**内容**
- `workflow_run` イベントの処理を実装する
  - `workflow_run_id` で State DB のレコードを特定する
  - CI 成功：State DB を `ci-passed` に更新
  - CI 失敗（repair 上限内）：State DB を `ci-failed` に更新 → SQS に repair job を積む
  - CI 失敗（repair 上限超過）：`needs-human` ラベル付与 + Discord 通知

**完了条件**
- CI 失敗 → repair ジョブ投入 → CI 再実行の一連のループが動くこと

---

## フェーズF：結合・E2E 検証

### F-1：E2E シナリオ検証

**内容**
- 以下のシナリオを手動で通しで確認する
  1. Discord `/issue` → Issue 起票 → requirements → design → task-split → implement → PR 作成 → CI 通過
  2. CI 失敗 → repair → CI 再実行
  3. repair 上限超過 → `needs-human` エスカレーション
- 各ステップで State DB の状態が正しく更新されていることを確認する

**完了条件**
- `docs/design.md` § 17（要件定義書の到達条件）を全て満たすこと

---

## Issue 一覧サマリ

| # | フェーズ | タイトル | リポジトリ |
|---|---|---|---|
| A-1 | A | トリガーワークフロー雛形 | dev-agents |
| A-2 | A | CI ワークフロー雛形 | dev-agents |
| A-3 | A | ラベル定義・bootstrap スクリプト | dev-agents |
| A-4 | A | ドキュメント雛形・AGENTS.md テンプレート | dev-agents |
| B-1 | B | DynamoDB State DB | agent-intake |
| B-2 | B | SQS キュー + EventBridge Pipes | agent-intake |
| B-3 | B | ECS Fargate クラスター・タスク定義基盤 | agent-intake |
| C-1 | C | Lambda handler（Discord ACK） | agent-intake |
| C-2 | C | Lambda processor（LLM 解釈・Issue 起票） | agent-intake |
| D-1 | D | Worker 基盤（エントリーポイント・State DB 連携） | agent-intake |
| D-2 | D | requirements Worker | agent-intake |
| D-3 | D | design Worker | agent-intake |
| D-4 | D | task-split Worker | agent-intake |
| D-5 | D | implement Worker | agent-intake |
| D-6 | D | repair Worker | agent-intake |
| E-1 | E | Orchestrator Lambda（Webhook 受信） | agent-intake |
| E-2 | E | Orchestrator Lambda（CI 結果処理・repair 投入） | agent-intake |
| F-1 | F | E2E シナリオ検証 | - |
