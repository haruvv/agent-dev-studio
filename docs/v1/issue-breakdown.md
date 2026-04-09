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
フェーズC は フェーズA（テンプレート類）と フェーズB（API Gateway・Lambda 実行環境・State DB）が整い次第着手可能。
フェーズD は フェーズB（SQS・ECS 基盤・State DB）が整い次第着手可能。
フェーズE は フェーズB（SQS・State DB）と フェーズC（agent-intake Lambda URL の確定）が整い次第着手可能。C-2 の Webhook 登録ステップは E-1 の完了後に有効になる。

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
- SQS 送信メッセージ形式：`{ job_id, job_type, issue_number, repo }`
  - `job_id` は `${{ github.run_id }}-${{ github.run_attempt }}` を使用する（`design.md` § 2.3.1 のワークフロー例と統一。再実行ごとに一意になるため UUID 生成は不要）
- 各ワークフローは `permissions: id-token: write` を設定し、GitHub Actions OIDC で `SQS_SENDER_ROLE_ARN`（repo secret）を assume して一時クレデンシャルを取得。`AWS_ACCESS_KEY_ID`/`AWS_SECRET_ACCESS_KEY` は使用しない
- `waiting-for-answer` 後の再エントリー方針（MVP）
  - 人間がコメントで回答後、手動で `waiting-for-answer` を外し `要件定義作成` を再付与する
  - `requirements-trigger.yml` がラベル付与を検知して requirements ジョブを再実行する
  - この再エントリー手順を app repo の `AGENTS.md` に明記する

**完了条件**
- actionlint でワークフロー構文が通ること
- ラベル名が `docs/design.md` § 3.1 のラベル定義と一致すること
- `要件定義作成` の再付与で requirements ジョブが再実行されること

---

### A-2：CI ワークフロー雛形

**内容**
- `ci.yml` を `haruvv/dev-agents` に追加する
  - トリガー：`pull_request: branches: [main]` / `push: branches: [main]`
  - MVP 初期 CI：actionlint による構文チェックのみ
- `push: branches: [main]` トリガーにより、requirements/design Worker が `docs/` を main に直接 push した場合にも同じ CI が実行される（軽量チェックのみのため問題なし）
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
- 同一テーブルに3種類のアイテムを格納する（`job_id` の prefix でアイテム種別を判別）

| アイテム種別 | `job_id` の形式 | 用途 | key 例 |
|---|---|---|---|
| ① ジョブレコード（canonical PR record） | `${run_id}-${run_attempt}` | パイプライン実行状態の追跡。**このアイテムの `job_id` が canonical record ID**（implement 完了後も repair が同レコードを更新し続ける） | `9876543210-1` |
| ② ロックアイテム | `LOCK#{repo}#{issue_number}#{job_type}` または `LOCK#{repo}#{pr_number}#repair` | 二重起動防止 | `LOCK#haruvv/my-app#42#implement` |
| ③ リポジトリレジストリ | `REPO#{repo}` | Webhook 登録状態・repo 一覧管理 | `REPO#haruvv/my-app` |

- TTL 属性を設定する（ロックアイテムの孤立防止・`done`/`failed` レコードの自動削除に使用）
- GSI を2つ作成する（ロックアイテム・レジストリはキー prefix で判別するため GSI 不要）
  - `pr-number-index`：PK = `repo`、SK = `pr_number`（Webhook の `pull_request` イベント照合）
  - `head-sha-index`：PK = `head_sha`（Webhook の `workflow_run` イベント照合）

**完了条件**
- テーブルが作成され、Lambda / ECS タスクから読み書きできること
- ロックアイテムの条件付き書き込みで同時起動を防止できること
- リポジトリレジストリアイテムで登録済み repo 一覧を取得できること
- GSI 経由で `pr_number` / `head_sha` でジョブレコードを取得できること

---

### B-2：SQS キュー + sqs-to-ecs Lambda（EventBridge Pipes 代替）

**内容**
- SQS キュー（標準キュー）を作成する
  - Dead Letter Queue を設定する（最大受信回数：3）
- `sqs-to-ecs` Lambda を SQS トリガーとして設定し、ECS Fargate タスクを起動する
  - EventBridge Pipes は ECS コンテナオーバーライドへの動的参照が機能しないため Lambda に変更
  - Lambda が `record["body"]` を `JOB_PAYLOAD` 環境変数としてタスクに渡す
- GitHub Actions OIDC プロバイダーを IAM に登録し、app repo のワークフローが assume できる IAM Role（`SQS_SENDER_ROLE_ARN`）を作成する
  - 権限：`sqs:SendMessage` を対象 SQS ARN のみに制限
  - 信頼ポリシー：GitHub Actions OIDC の `sub` クレームを `repo:haruvv/*:ref:refs/heads/*` に限定（任意の `haruvv` 配下 app repo を許可。MVP暫定：org 内の全 repo を信頼する broad policy。将来フェーズで `/create` 時に app repo ごとの専用 Role または条件を追加する）

**完了条件**
- テストメッセージを SQS に送信すると ECS タスクが起動すること
- app repo のワークフローが OIDC クレデンシャルで SQS にメッセージを送信できること

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
- EventBridge Scheduler で5分ごとに handler を ping してウォームアップを維持する（handler は `event.source == "aws.events"` を検出したら Ed25519 検証をスキップして即座に 200 を返す warmup ブランチを持つ）

**環境変数**：`DISCORD_PUBLIC_KEY`, `PROCESSOR_FUNCTION_NAME`

**完了条件**
- Discord Developer Portal から Interactions Endpoint URL として登録できること
- /issue コマンド送信時に type 5 ACK が3秒以内に返ること

---

### C-2：Lambda processor（/issue コマンド）

**内容**
- `agent-intake-processor` Lambda の骨格と `/issue` コマンドを実装する
  - Gemma API で自然言語を解釈する
  - 曖昧な場合は Discord に確認メッセージを返信する
  - 明確な場合は対象 app repo に GitHub Issue を起票し、`要件定義作成` ラベルを付与する
  - Discord に Issue URL を返信する
  - processor の Lambda 非同期リトライは 0 回に設定する（重複起票防止）
- `/issue [<repo>] <description>` のコマンド解釈を実装する
  - repo 指定あり → 該当 app repo に Issue 起票
  - repo 省略 → 登録済み app repo 一覧を Discord に提示し、選択を促す（`DEFAULT_GITHUB_REPO` へのサイレントフォールバックは行わない）

**環境変数**：`GEMMA_API_KEY`, `GITHUB_TOKEN`, `STATE_TABLE_NAME`

**完了条件**
- Discord で `/issue <repo> <description>` を送ると対象 repo に Issue が起票されること
- repo を省略した場合に登録済み repo 一覧が返ること
- 曖昧な入力で確認メッセージが返ること

---

### C-3：/create コマンド

**内容**
- `/create <app-name>` のコマンド解釈を実装する（`docs/design.md` § 2.1.2 の手順に従う）
  1. GitHub API で新規リポジトリを作成（`auto_init: true`）。デフォルトブランチが `main` でない場合は `GET refs` → `POST git/refs` → `PATCH repo` で `main` に切り替える
  2. `dev-agents` からワークフロー雛形・docs 雛形・ラベル定義・`AGENTS.md` テンプレートを取得して配置
  3. ラベルを一括作成
  4. Secrets（`JOB_QUEUE_URL`・`SQS_SENDER_ROLE_ARN`・`AWS_REGION`）を登録
  5. Actions 権限を有効化
  6. State DB にリポジトリレジストリアイテムを登録（Webhook 登録の条件分岐は `docs/design.md § 2.1.2` の「GitHub Webhook の登録タイミング」に従う）
     - Orchestrator デプロイ済みの場合：即座に Webhook 登録 → `webhook_active: true` で書き込む
     - Orchestrator 未デプロイの場合：`webhook_active: false` で書き込み、E-1 完了時に一括登録
  7. Discord に完了通知

**環境変数**：`GITHUB_TOKEN`, `STATE_TABLE_NAME`, `WEBHOOK_ENDPOINT_URL`, `GITHUB_WEBHOOK_SECRET`, `ORCHESTRATOR_FUNCTION_NAME`

> **注意**：`ORCHESTRATOR_FUNCTION_NAME` は E-1 デプロイ前は未設定のため、未設定なら「未デプロイ」扱いにする。

**完了条件**
- `/create <app-name>` でリポジトリ作成・テンプレート配置・ラベル作成・Secrets 登録・State DB 登録が完了すること（Webhook 登録の動作確認は E-1 完了後に F-1 で行う）

---

## フェーズD：ECS Worker

Worker 基盤を実装してから、各ジョブ種別の Worker を実装する。

> **MVP 前提：app repo の `main` ブランチに branch protection を設定しない。** requirements Worker・design Worker は Worker の PAT で `main` に直接 push する。branch protection が必要な場合は将来フェーズで docs ブランチ + PR マージ戦略に移行する。

### D-1：Worker 基盤（エントリーポイント・State DB 連携）

**内容**
- ECS タスクのエントリーポイントスクリプトを実装する
  - 環境変数からジョブメッセージ（`job_type`, `issue_number`, `repo`, `job_id` 等）を取得する
  - 冪等性チェック（`docs/design.md § 2.4.1` の job_type 別ロジックに従う）：implement は `implement_completed` フラグ確認、repair は `repair_attempt` 比較、その他は `status == done / failed` 確認。条件一致で破棄して終了
  - lock item を DynamoDB 条件付き書き込みで挿入（repair 以外：`LOCK#{repo}#{issue_number}#{job_type}`、repair：`LOCK#{repo}#{pr_number}#repair`）。既存の場合はジョブを破棄して終了
  - `job_type` に応じてジョブハンドラーを呼び出す
  - 完了・失敗時に State DB を更新する
- `blocked` ラベルチェックの共通ユーティリティを実装する
- GitHub API クライアント（ラベル付与・コメント投稿・Issue 起票）の共通実装

**完了条件**
- SQS 経由でタスクが起動し、State DB が更新されること
- `blocked` ラベルがある場合に処理がスキップされること
- 同一 `job_id` のメッセージを2回 SQS に送信しても、State DB の更新・ラベル操作・コメント投稿などの副作用が1回分のみ発生すること（冪等性）
- 同一 `{repo, issue_number, job_type}` のタスクが並行起動しないこと（二重起動防止）

---

### D-2：requirements Worker

**内容**
- `job_type: requirements` を処理する Worker を実装する
  - `docs/design.md` § 4.1 の処理フローに従う
  - Issue 本文・コメント履歴を読み込み、LLM で要件整理
  - 曖昧な場合：質問コメント投稿 → `waiting-for-answer` ラベル付与。人間の回答後に `要件定義作成` が再付与されると requirements ジョブが再実行される（A-1 の trigger workflow で自動検知）
  - 確定できる場合：対象 app repo の `main` ブランチに `docs/<issue-number>-requirements.md` を直接コミット・プッシュ → `詳細設計作成` ラベル付与（PAT）
  - 質問往復上限：2往復（超過時は強制確定）

**完了条件**
- requirements.md が対象 app repo の `main` ブランチ `docs/` にコミットされること
- 次フェーズのラベル付与で GitHub Actions が起動すること
- `waiting-for-answer` → 人間回答 → `要件定義作成` 再付与 → ジョブ再実行の流れが動くこと

---

### D-3：design Worker

**内容**
- `job_type: design` を処理する Worker を実装する
  - `docs/design.md` § 4.2 の処理フローに従う
  - `main` ブランチの `docs/<issue-number>-requirements.md` を読み込み、設計書を生成して `main` に `docs/<issue-number>-design.md` を直接コミット・プッシュ → `タスク分割` ラベル付与（PAT）

**完了条件**
- design.md が対象 app repo の `main` ブランチ `docs/` にコミットされること

---

### D-4：task-split Worker

**内容**
- `job_type: task-split` を処理する Worker を実装する
  - `docs/design.md` § 4.3 の処理フローに従う
  - 設計書の作業分割方針を読み込み、子 Issue を起票する
  - 各子 Issue 本文の**1行目**に `parent_issue: <番号>` 形式で記載する（例：`parent_issue: 42`）。implement Worker が正規表現 `^parent_issue:\s*(\d+)` で解析する
  - 各子 Issue に `ready-for-impl` ラベルを付与（PAT）

**完了条件**
- 子 Issue が起票され、各 Issue に `ready-for-impl` が付与されること

---

### D-5：implement Worker

**内容**
- `job_type: implement` を処理する Worker を実装する
  - `docs/design.md` § 5.1 の処理フローに従う
  - 対象 app repo をクローン
  - `impl/issue-<番号>-<slug>` ブランチを作成・プッシュ。State DB に `branch_name` を記録
  - `main` ブランチの `docs/<親issue番号>-design.md` を参照して Claude Code で実装
  - コミット・プッシュ後、State DB に `head_sha` を記録
  - PR 作成（`ai-generated` ラベル付与）。State DB に `pr_number` を記録（`ready-for-human-review` は CI 成功後に Orchestrator が付与するため、ここでは付与しない）

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
  - `head_sha` と `workflow_run_id` で CI 失敗ログを取得（対象は `name: CI` のワークフロー実行のみ。他のワークフローの失敗は repair のトリガーにしない）
  - Claude Code でエラーを修正・コミット・プッシュ。State DB の `head_sha` を更新
  - PR synchronize により `name: CI` ワークフローが自動再実行される

**完了条件**
- 修正 push 後に CI（`name: CI`）が再実行されること
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
  - synchronize：最新 `head_sha` を State DB に反映 → PR から `ready-for-human-review` を除去（CI が再実行されるため）
  - closed：State DB を終了系ステータスに更新

- デプロイ完了後、State DB の `webhook_active: false` なリポジトリレジストリアイテムを全件スキャンし、各 app repo に GitHub Webhook を一括登録する（`GITHUB_WEBHOOK_SECRET` を署名シークレットとして設定）
- 登録完了後、各アイテムの `webhook_active` を `true` に更新する

**完了条件**
- GitHub Webhook が届き、State DB が更新されること
- `/create` で作成済みの app repo に Webhook が登録され `webhook_active: true` になること

---

### E-2：CI 結果処理・repair ジョブ投入

**内容**
- `workflow_run` イベントの処理を実装する
  - イベントに含まれる `head_sha` または `pr_number` で State DB のレコードを特定する
  - 特定後、`workflow_run_id` を State DB に書き込む（以降の repair job でログ取得に使用）
  - CI 成功：State DB を `ci-passed` に更新 → PR に `ready-for-human-review` ラベルを付与（PAT: MVP暫定）。**このラベルが存在する = 「最新コミットで CI が通過済み」を意味する**
  - CI 失敗（repair 上限内）：State DB を `ci-failed` に更新 → PR から `ready-for-human-review` を除去 → SQS に repair job を積む（`job_id` は canonical PR record の元 `job_id` をそのまま使用。repair Worker は同レコードを in-place で更新する）
  - CI 失敗（repair 上限超過）：`needs-human` ラベル付与 + Discord 通知

**完了条件**
- CI 成功 → `ready-for-human-review` ラベルが付与されること
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
- `docs/requirement.md` § 17（MVPの到達条件）を全て満たすこと

---

### F-2：スタールジョブ検出（Worker ハング）

**内容**（`docs/requirement.md` § 13 対応・前半）
- EventBridge Scheduler で定期実行する Lambda を実装する
  - State DB をスキャンし、`status == in-progress` または `ci-running` かつ `updated_at` が30分以上前のレコードを検知
  - 検知時：対象 Issue に `needs-human` ラベルを付与し、Discord に通知する

**完了条件**
- State DB の `status` を `in-progress` に固定したレコードを用意し、設定閾値経過後に `needs-human` が発火すること（検証時は閾値を2〜3分に短縮して確認してよい）

---

### F-3：スタールジョブ検出（waiting-for-answer タイムアウト）

**内容**（`docs/requirement.md` § 13 対応・後半）
- F-2 の Lambda に `waiting-for-answer` 検出を追加する
  - GitHub API で `waiting-for-answer` ラベルが付いた Issue を取得し、Issue の `updated_at` が24時間以上前のものを検知（State DB にこのラベル状態は格納しないため GitHub API を参照）
  - 検知時：対象 Issue に `needs-human` ラベルを付与し、Discord に通知する

**完了条件**
- `waiting-for-answer` ラベルを設定閾値変化させずに放置すると `needs-human` が発火すること（検証時は閾値を短縮して確認してよい）

---

## Issue 一覧サマリ

| # | フェーズ | タイトル | リポジトリ | 状態 |
|---|---|---|---|---|
| A-1 | A | トリガーワークフロー雛形 | dev-agents | ✅ 完了 |
| A-2 | A | CI ワークフロー雛形 | dev-agents | ✅ 完了 |
| A-3 | A | ラベル定義・bootstrap スクリプト | dev-agents | ✅ 完了 |
| A-4 | A | ドキュメント雛形・AGENTS.md テンプレート | dev-agents | ✅ 完了 |
| B-1 | B | DynamoDB State DB | agent-intake | ✅ 完了 |
| B-2 | B | SQS キュー + sqs-to-ecs Lambda（EventBridge Pipes 代替） | agent-intake | ✅ 完了 |
| B-3 | B | ECS Fargate クラスター・タスク定義基盤 | agent-intake | ✅ 完了 |
| C-1 | C | Lambda handler（Discord ACK） | agent-intake | ✅ 完了 |
| C-2 | C | Lambda processor（/issue コマンド） | agent-intake | ✅ 完了 |
| C-3 | C | /create コマンド | agent-intake | ✅ 完了 |
| D-1 | D | Worker 基盤（エントリーポイント・State DB 連携） | agent-intake | ✅ 完了 |
| D-2 | D | requirements Worker | agent-intake | ✅ 完了・動作確認済み |
| D-3 | D | design Worker | agent-intake | ✅ 完了・動作確認済み |
| D-4 | D | task-split Worker | agent-intake | ✅ 完了・動作確認済み |
| D-5 | D | implement Worker | agent-intake | ✅ 完了・動作確認済み（PR作成まで確認） |
| D-6 | D | repair Worker | agent-intake | ✅ 実装完了（E2E 未確認） |
| E-1 | E | Orchestrator Lambda（Webhook 受信） | agent-intake | ✅ 完了 |
| E-2 | E | Orchestrator Lambda（CI 結果処理・repair 投入） | agent-intake | ✅ 完了 |
| F-1 | F | E2E シナリオ検証 | - | 🔄 進行中（正規フロー未確認） |
| F-2 | F | スタールジョブ検出（Worker ハング） | agent-intake | 未着手 |
| F-3 | F | スタールジョブ検出（waiting-for-answer タイムアウト） | agent-intake | 未着手 |
