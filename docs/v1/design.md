# AI自律開発エージェントパイプライン 設計書

本書は要件定義書（`docs/requirement.md`）に基づく自律開発パイプラインの設計を定義する。

---

## 1. システム全体設計

### 1.1 設計思想

パイプラインを「受付層」「オーケストレーション層」「実行層」の3層に分離する。

| 層 | 担当 | 主要コンポーネント |
|---|---|---|
| 受付層 | 人間からの要求受付・GitHub Issue起票 | agent-intake（Lambda） |
| オーケストレーション層 | ステート管理・ジョブ発行・CI結果処理 | Orchestrator（Lambda）/ GitHub Actions（トリガー） |
| 実行層 | 実際の要件整理・設計・実装・修正 | ECS Fargate Worker |

**GitHub Actions はトリガーと CI に限定する。** 実装・修正などの重処理は ECS Fargate Worker に委譲する。これにより Actions のツール制限・タイムアウト・イベント発火制約を回避し、Worker 側で自由な実行環境を確保する。

**知識を最大限活用するため、実際にパイプラインが動作するワークフロー、Issue/PR、`docs/`、実装コードは各 app repo に集約する。**
これにより、要件定義・設計・意思決定・制約・PR履歴が同一 repo に蓄積され、後続 Worker が局所文脈を失わずに参照できる。

agent-intake と Orchestrator は同一リポジトリ内の別 Lambda 関数として実装するが、責務は明確に分離する。agent-intake は人間からの入力を受け取る受付専用、Orchestrator は CI・PR イベントを処理するパイプライン制御専用とする。

### 1.2 全体フロー

Human
  │ Discord /chat コマンド（自然言語）
  ▼
Lambda: agent-intake-handler
  │ Ed25519 署名検証・即時 ACK（type 5）
  │ SQS に job_type=chat メッセージを投入
  ▼
SQS → sqs-to-ecs Lambda（シム）
  ▼
ECS: LangGraph エージェント（GLM glm-4.5-flash / z.ai）
  │ DynamoDB に会話履歴を保持しながら多ターン会話
  │ 要件が曖昧な場合は Discord 上で確認
  │ 要件が十分揃ったら create_github_issue ツール呼び出し
  ▼
対象 app repo に GitHub Issue 起票（2ステップ）
  │ ① POST /issues（ラベルなし）
  │ ② POST /issues/{number}/labels（「要件定義作成」付与）
  ▼
対象 app repo の GitHub Actions（issues: labeled イベント検知 → SQS にジョブ送信）
  ▼
SQS → sqs-to-ecs Lambda（シム）
  ▼
ECS Fargate Worker（ECC: Everything Claude Code）
  ├─ 要件定義・設計・タスク分割フェーズ：
  │    Issue・コメント履歴を読み込み
  │    claude CLI（--print モード）で成果物生成
  │    成果物を対象 app repo の docs/ にコミット
  │    次フェーズのラベルを PAT（MVP暫定）で付与
  │    → 対象 app repo の GitHub Actions が次ジョブを発行
  │
  └─ 実装フェーズ：
       対象 app repo をクローン、実装ブランチ作成
       claude CLI（エージェントモード）でコード実装
         ├─ planner サブエージェントで実装計画策定
         └─ code-reviewer サブエージェントでレビュー
       コミット → プッシュ → PR 作成（`ai-generated` ラベルのみ付与）
  ▼
対象 app repo の GitHub Actions（CI: lint / test）
  ▼
GitHub Webhook → Orchestrator（Lambda）
  ├─ CI 成功 → State DB 更新（ci-passed）→ `ready-for-human-review` 付与 → 人間レビュー待ち
  └─ CI 失敗 → State DB 更新（ci-failed）
              → SQS に repair job を積む
              → ECS Fargate Worker（修正・再コミット）
              → PR synchronize により CI 再実行
              → 上限到達で needs-human エスカレーション
  ▼
人間レビュー → approve → マージ

### 1.3 リポジトリ構成

| リポジトリ | 役割 |
|---|---|
| `haruvv/agent-intake` | agent-intake（受付 Lambda）・Orchestrator（Webhook Lambda）を同梱 |
| `haruvv/dev-agents` | パイプライン定義テンプレートの配布元（ワークフロー雛形・ラベル定義・初期ドキュメント雛形・ECC インフラ） |
| `haruvv/<app-name>` | 実際の開発対象 repo。workflow・docs・Issue/PR・実装コードを集約する実行現場 |

`dev-agents` はテンプレート配布元であり、**実際にワークフローが動作する場所ではない。**
実際の GitHub Actions workflow は `haruvv/<app-name>` に配置し、その repo の `docs/`、Issue、PR、コードと密接に連携して動作する。

`dev-agents` が提供する ECC（Everything Claude Code）インフラは以下の通り。Worker がリポジトリを参照することで利用できる。

| ディレクトリ | 内容 |
|---|---|
| `.claude/agents/` | サブエージェント定義（planner, architect, code-reviewer, security-reviewer, code-explorer, doc-updater） |
| `.claude/skills/` | スキル定義（coding-standards, api-design, python-patterns, tdd-workflow 等） |
| `.claude/rules/` | 共通ルール（coding-style, git-workflow, security, testing 等） |

---

## 2. コンポーネント設計

### 2.1 agent-intake（受付層）

**リポジトリ**：`haruvv/agent-intake`  
**責務**：人間からの Discord コマンドを受け取り、対象 app repo に GitHub Issue を起票する。CI 結果などパイプライン内部のイベントは処理しない。

#### 2.1.1 Discord Bot（handler Lambda + ECS LangGraph エージェント構成）

Discord のインタラクション受信から3秒以内の ACK が必要なため、handler Lambda は即時 ACK のみを担い、実処理は ECS LangGraph エージェントに委譲する。

Discord POST /interactions
  ↓
API Gateway
  ↓
Lambda: agent-intake-handler
  署名検証（Ed25519）
  type 1 → PONG 即時返答
  type 2 (/chat) → type 5 ACK 即時返答 + SQS に job_type=chat メッセージを投入
  ↓
SQS → sqs-to-ecs Lambda
  ↓
ECS: LangGraph エージェント（GLM glm-4.5-flash / z.ai API）
  DynamoDB に会話履歴（最大20ターン）を保存
  ツール: web_search / create_github_issue
  会話を重ねて要件が揃ったら create_github_issue を呼び出し
  → 対象 app repo に GitHub Issue 起票（「要件定義作成」ラベル付与）
  → Discord に応答を返信（send_followup）

| コンポーネント | 役割 | タイムアウト目安 |
|---|---|---|
| `agent-intake-handler` | 署名検証・即時 ACK・SQS 投入 | 3秒以内 |
| ECS LangGraph エージェント | 会話・ルーティング・Issue 起票 | 最大15分（ECS タスク） |

> **注**: `src/processor/processor.py`（Gemma を使う `/issue` スラッシュコマンド実装）は旧フローの残骸であり、現行の handler は processor Lambda を呼び出していない。

#### 2.1.2 /create コマンド（マルチリポジトリ対応）

`/create <app-name>` 受信時に processor が以下を実行する。

1. GitHub API で新規リポジトリ `haruvv/<app-name>` を作成（`auto_init: true` で初期コミットを作成）。デフォルトブランチが `main` でない場合は以下の手順で `main` に切り替える：
   1. `GET /repos/{owner}/{repo}/git/refs/heads/{current_default}` で初期コミットの SHA を取得
   2. `POST /repos/{owner}/{repo}/git/refs` に `{ "ref": "refs/heads/main", "sha": "<sha>" }` を送信して `main` ブランチを作成
   3. `PATCH /repos/{owner}/{repo}` に `{ "default_branch": "main" }` を送信してデフォルトブランチを切り替え
2. `dev-agents` からワークフロー雛形・初期ドキュメント雛形・ラベル定義・`AGENTS.md` テンプレートを取得
3. 対象 app repo に `.github/workflows/`、`docs/`、`AGENTS.md`、初期設定を配置
4. ラベルを一括作成
5. Secrets を一括登録（app repo のワークフローが SQS 送信するために必要な最小限のみ）
   - `JOB_QUEUE_URL`（トリガーワークフローが SQS へジョブ送信するために必要）
   - `AWS_REGION`（OIDC で取得したセッションクレデンシャルで使用するリージョン）
   - `AWS_ACCESS_KEY_ID`・`AWS_SECRET_ACCESS_KEY` は **app repo には配布しない**。app repo のトリガーワークフローは GitHub Actions OIDC を使用して一時クレデンシャルを取得し SQS 送信する。`/create` 時に app repo に IAM Role ARN（`SQS_SENDER_ROLE_ARN`）を Secret として登録し、ワークフローはこの ARN を assume する（B-2 で IAM Role を作成し、GitHub Actions OIDC プロバイダーを信頼条件に設定する）。
   - `CLAUDE_CODE_OAUTH_TOKEN`・`GH_PAT` は **app repo には配布しない**。これらは Worker / agent-intake Lambda が保持し、ECS タスクや Lambda の環境変数として管理する。
6. Actions 権限（PR 作成許可）を有効化
7. State DB にリポジトリレジストリアイテムを登録（後述の条件で `webhook_active` を設定）
8. Discord に完了通知

**GitHub Webhook の登録タイミング（条件分岐）**：
- **Orchestrator がすでにデプロイ済みの場合**：`/create` 時に即座に Webhook を登録し、`webhook_active: true` で State DB に書き込む。
- **Orchestrator 未デプロイ（初回セットアップ段階）の場合**：`webhook_active: false` で登録し、E-1（Orchestrator Lambda デプロイ）完了後の起動時スキャンで Orchestrator が一括登録し `webhook_active: true` に更新する。

`webhook_active` の確認方法：agent-intake が Orchestrator の Lambda 関数名（`ORCHESTRATOR_FUNCTION_NAME` 環境変数）に対して `GetFunctionConfiguration` を呼び出し、`State == Active` であれば「デプロイ済み」と判断する。

**Webhook 登録時のイベントサブスクリプション**：Webhook API の `events` フィールドに `["workflow_run", "pull_request"]` を明示的に指定する。`events` を省略するとデフォルトで `push` のみが対象となるため必須。

#### 2.1.3 /issue コマンド

`/issue [<repo>] <description>` を受信した場合、processor は以下を実行する。

- repo 指定あり → 該当 app repo に Issue 起票
- repo 省略 → 登録済み app repo 一覧を提示し、選択を促す
- 曖昧な要求 → Discord 上で確認を行い、確定後に対象 app repo へ起票

#### 2.1.4 環境変数

| 変数 | 対象関数 | 内容 |
|---|---|---|
| `DISCORD_PUBLIC_KEY` | handler | Discord アプリ公開鍵 |
| `JOB_QUEUE_URL` | handler | SQS キュー URL（chat メッセージ投入先） |
| `GLM_API_KEY` | LangGraph ECS | GLM API キー（z.ai） |
| `GITHUB_TOKEN` | LangGraph ECS / processor（旧） | GitHub PAT（MVP暫定） |
| `CLAUDE_CODE_OAUTH_TOKEN` | Worker ECS | Claude Code OAuth トークン（ECS secrets injection 経由） |
| `GH_PAT` | Worker ECS | GitHub PAT（ラベル操作用。app repo には配布しない） |
| `STATE_TABLE_NAME` | LangGraph ECS / Worker ECS | DynamoDB テーブル名 |
| `WEBHOOK_ENDPOINT_URL` | processor（旧） | Orchestrator の `/webhook` エンドポイント URL |
| `GITHUB_WEBHOOK_SECRET` | processor（旧） | GitHub Webhook 署名シークレット |

#### 2.1.5 エラー処理方針

- Discord トークン期限切れ（404）はエラーを無視・リトライしない
- processor の Lambda 非同期リトライは 0 回（重複 Issue 起票防止）
- 曖昧な要求は Issue を起票せず Discord 上で確認する
- repo 指定が不正な場合は Issue を起票せず、候補 repo を提示する

#### 2.1.6 ウォームアップ戦略

Lambda コールドスタートが Discord の3秒制限に近いため、EventBridge Scheduler で5分ごとに handler を ping してウォーム状態を維持する。handler の冒頭でイベントソースを判定し（`event.source == "aws.events"` の場合）、Ed25519 検証をスキップして即座に 200 を返す warmup ブランチを実装する。Discord リクエストと EventBridge scheduled event はペイロード形式が異なるため、この分岐が必須。

---

### 2.2 Orchestrator（オーケストレーション層）

**リポジトリ**：`haruvv/agent-intake`（同梱）  
**責務**：GitHub Webhook を受け取り、CI 結果・PR イベントに応じて State DB を更新し、repair ジョブを SQS に積む。人間からの入力は処理しない。

#### 2.2.1 GitHub Webhook エンドポイント

`/webhook` エンドポイントを持つ独立した Lambda 関数として実装する。

**受信イベント**
- `workflow_run`（CI 完了）
- `pull_request`（PR オープン / synchronize / クローズ）

**処理内容**

| イベント | アクション |
|---|---|
| CI 成功（`workflow.name == "CI"` かつ `event == "pull_request"` のみ対象） | State DB を `ci-passed` に更新 → PR に `ready-for-human-review` ラベル付与（PAT: MVP暫定） |
| CI 失敗（同条件、repair 上限内） | State DB を `ci-failed` に更新 → PR から `ready-for-human-review` を除去 → SQS に repair job を積む（メッセージには `job_id`・`pr_number`・`repo`・`repair_attempt` を含める） |
| CI 失敗（同条件、repair 上限超過） | `needs-human` ラベル付与 + 通知 |
| PR 作成 | `pr_number` と `head_sha` を State DB に反映 |
| PR synchronize | 最新 `head_sha` を State DB に反映 → PR から `ready-for-human-review` を除去（CI が再実行されるため） |
| PR closed | 状態を終了系へ更新 |

#### 2.2.2 環境変数

| 変数 | 内容 |
|---|---|
| `GITHUB_WEBHOOK_SECRET` | Webhook 署名検証シークレット |
| `GITHUB_TOKEN` | GitHub PAT（ラベル操作用、MVP暫定） |
| `JOB_QUEUE_URL` | SQS キュー URL |
| `STATE_TABLE_NAME` | DynamoDB テーブル名 |
| `DISCORD_BOT_TOKEN` | Discord Bot トークン（`needs-human` エスカレーション通知用） |
| `DISCORD_ALERT_CHANNEL_ID` | Discord 通知先チャンネル ID |

---

### 2.3 GitHub Actions（オーケストレーション補助）

**対象**：各 `haruvv/<app-name>` repo  
**役割**：ラベルイベント検知と SQS へのジョブ送信、および CI 実行に限定する。処理時間は数秒以内。

#### 2.3.1 トリガーワークフロー構成

各 app repo に、各エージェントに対応する workflow を配置する。`issues: labeled` / `pull_request: labeled` イベントを検知し、SQS にジョブメッセージを送信する。

例：`.github/workflows/requirements-trigger.yml`

on:
  issues:
    types: [labeled]

jobs:
  dispatch:
    if: github.event.label.name == '要件定義作成'
    runs-on: ubuntu-latest
    steps:
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.SQS_SENDER_ROLE_ARN }}
          aws-region: ${{ secrets.AWS_REGION }}
      - name: Send job to SQS
        run: |
          aws sqs send-message \
            --queue-url ${{ secrets.JOB_QUEUE_URL }} \
            --message-body '{
              "job_id": "${{ github.run_id }}-${{ github.run_attempt }}",
              "job_type": "requirements",
              "issue_number": "${{ github.event.issue.number }}",
              "repo": "${{ github.repository }}"
            }'

#### 2.3.2 CI ワークフロー

各 app repo に CI workflow を配置する。

name: CI

on:
  pull_request:
    branches: [main]
  push:
    branches: [main]

**運用方針**
- `pull_request` は PR 作成時および PR ブランチ更新時（synchronize）に実行される
- `push: main` は merge 後の main 保護用に実行する
- feature ブランチ単独 push を CI の主軸にはしない
- repair Worker は PR ブランチへ修正 push を行い、PR synchronize により CI を再実行させる

**MVP フェーズの CI 構成**

| フェーズ | CI 内容 |
|---|---|
| MVP 初期 | ワークフロー構文チェック（actionlint） |
| 実装開始後 | リント・型検査 |
| テスト追加後 | 単体テスト |

---

### 2.4 ECS Fargate Worker（実行層）

Claude Code をネイティブ実行する自律エージェント（ECC: Everything Claude Code）。ジョブ種別に応じた処理を実行し、完了後に次フェーズへバトンを渡す。

Worker は `dev-agents` が提供する ECC インフラ（`.claude/agents/`・`.claude/skills/`・`.claude/rules/`）を参照し、サブエージェントやスキルを活用して処理を行う。

#### 2.4.1 起動フロー

SQS メッセージ受信
  ↓ Lambda: sqs-to-ecs（EventBridge Pipes 代替シム）
    ※ EventBridge Pipes は ECS コンテナオーバーライドへの動的参照（`<$.Body>` 等）が機能しないため Lambda に変更
    ※ Lambda が `record["body"]` を `JOB_PAYLOAD` 環境変数としてタスクに渡す
ECS Fargate タスク起動（ジョブメッセージをパラメータとして受け取る）
  ↓
冪等性チェック（SQS at-least-once 対策）
  job_type == implement の場合：job_id で State DB を GetItem し、`implement_completed == true` ならこのメッセージを破棄して終了
  （implement 完了後に同レコードが repair で ci-failed/ci-running に書き換えられても、implement_completed フラグは不変のため遅延再配信を正しく棄却できる）
  job_type == repair の場合：job_id で State DB を GetItem し、`repair_attempt` が現メッセージの attempt 番号より大きければ破棄して終了
  その他（requirements / design / task-split）：job_id で State DB を GetItem し、status == done / failed ならこのメッセージを破棄して終了
  ↓
二重起動防止（ロックアイテム方式）
  job_type が repair 以外の場合：PK = `LOCK#{repo}#{issue_number}#{job_type}` で条件付き書き込み
  job_type が repair の場合：PK = `LOCK#{repo}#{pr_number}#repair` で条件付き書き込み（SQS メッセージに pr_number を含める必要がある）
  アイテムが既存の場合（attribute_not_exists 条件が false）はジョブを破棄して終了
  成功した場合は job_id で通常レコードを status=in-progress で作成（または repair の場合は既存レコードを更新）する
  ジョブ完了・失敗時にロックアイテムを削除する（TTL も設定して孤立防止）
  ↓
ジョブ種別に応じた処理を実行
  ↓
State DB を done / failed に更新（ロックアイテムを削除）
  ↓
タスク終了（使い捨て）

#### 2.4.2 ジョブ種別と処理内容

| ジョブ種別 | トリガーラベル | Worker の処理 | 次フェーズへの引き渡し |
|---|---|---|---|
| `requirements` | 要件定義作成 | 対象 app repo の Issue・コメント読み込み → `claude --print` CLI で要件整理 → 質問コメント投稿 or 要件確定・成果物コミット | ラベル更新（PAT: MVP暫定）で次ジョブ発行 |
| `design` | 詳細設計作成 | 対象 app repo の要件成果物を読み込み → `claude --print` CLI で設計書生成 → 同 repo にコミット | ラベル更新（PAT: MVP暫定） |
| `task-split` | タスク分割 | 対象 app repo の設計書読み込み → `claude --print` CLI でタスク分割 → 子 Issue 起票 → 各子 Issue に ready-for-impl 付与 | ラベル付与（PAT: MVP暫定）→ impl ジョブ発行 |
| `implement` | ready-for-impl | 対象 app repo をクローン → ブランチ作成 → `claude` CLI（エージェントモード）で実装 → コミット → プッシュ → PR 作成（`ai-generated` ラベルのみ付与） | Orchestrator が CI 成功時に `ready-for-human-review` を付与 |
| `repair` | CI 失敗（Orchestrator から） | 対象 app repo の PR ブランチをクローン → CI 失敗ログ取得 → `claude` CLI（エージェントモード）で修正 → コミット → プッシュ | Orchestrator が次回 CI 成功時に `ready-for-human-review` を付与 |

#### 2.4.3 claude CLI の実行方式

requirements / design / task-split Worker は `claude --print --dangerously-skip-permissions --model <model>` を subprocess で呼び出す。コンテナが非 root ユーザー（`worker`, uid=1000）で動作するため `--dangerously-skip-permissions` を使用できる。

認証は `CLAUDE_CODE_OAUTH_TOKEN` 環境変数を ECS secrets injection で渡す（Secrets Manager 経由）。claude CLI がこの環境変数を直接参照する。`ANTHROPIC_API_KEY` への転写は不要・行わない（OAuth トークンを API キーとして使用するとエラーになるため）。

implement / repair Worker は claude CLI をエージェントモードで実行し、リポジトリのクローン・実装・コミット・プッシュを自律的に行う。

#### 2.4.4 ラベル更新の PAT 必須理由（MVP暫定）

GitHub Actions の `GITHUB_TOKEN` によるラベル変更は `labeled` イベントを発火しない（無限ループ防止の仕様）。Worker が PAT を使用してラベルを更新することで、次のトリガーワークフローが正常に発火する。

#### 2.4.5 Issue 起票のラベル付与方式（2ステップ）

GitHub API でラベルを `POST /issues` のボディに含めて Issue 作成しても `issues: labeled` イベントが発火しない。そのため processor は以下の2ステップで Issue を起票する：
1. `POST /repos/{repo}/issues`（ラベルなし）
2. `POST /repos/{repo}/issues/{number}/labels`（ラベルを別途追加）

ステップ2の追加により `issues: labeled` イベントが発火し、トリガーワークフローが起動する。

**MVP暫定**  
PAT は管理コストが高く、スコープ制限が粗い。将来フェーズで GitHub Apps に移行し、最小権限・per-installation トークンに置き換える。

---

### 2.5 State DB（DynamoDB）

パイプラインの状態を永続管理し、二重起動防止・repair 試行回数管理・観測可能性を提供する。

テーブルには以下の2種類のアイテムを格納する。

**① ジョブレコード**

| 属性 | 型 | 説明 |
|---|---|---|
| `job_id` | PK（String） | `implement`/`requirements`/`design`/`task-split` ジョブ：`${run_id}-${run_attempt}`（GitHub Actions が生成）。`repair` ジョブ：元の `implement` ジョブの `job_id` を使用（canonical PR record を更新するため。repair ロック用に lock item PK は `LOCK#{repo}#{pr_number}#repair` を別途使用） |
| `issue_number` | Number | GitHub Issue 番号 |
| `parent_issue_number` | Number | 親 Issue 番号（子 Issue の場合） |
| `repo` | String | 対象 app repo（例: `haruvv/my-app`） |
| `job_type` | String | `requirements` / `design` / `task-split` / `implement` / `repair` |
| `status` | String | `queued` / `in-progress` / `pr-created` / `ci-running` / `ci-passed` / `ci-failed` / `done` / `failed` |
| `implement_completed` | Boolean | implement Worker が PR 作成まで完了したことを示す不変フラグ（一度 `true` に設定後は repair 等で変更しない） |
| `pr_number` | Number | 作成された PR 番号（GSI キー） |
| `branch_name` | String | 実装ブランチ名（例: `impl/issue-42-add-login`） |
| `head_sha` | String | 最新コミットの SHA（GSI キー） |
| `workflow_run_id` | String | CI 結果とジョブを紐付けるための workflow run ID |
| `ci_attempt` | Number | CI 試行回数 |
| `repair_attempt` | Number | repair 試行回数（上限 3） |
| `created_at` | String | ISO 8601 |
| `updated_at` | String | ISO 8601 |

**② ロックアイテム（二重起動防止用）**

| 属性 | 型 | 説明 |
|---|---|---|
| `job_id` | PK（String） | `LOCK#{repo}#{issue_number}#{job_type}` |
| `ttl` | Number | Unix タイムスタンプ（孤立ロック自動削除用） |

**③ リポジトリレジストリアイテム（`/issue` の repo 一覧用）**

| 属性 | 型 | 説明 |
|---|---|---|
| `job_id` | PK（String） | `REPO#{repo}`（例: `REPO#haruvv/my-app`） |
| `repo` | String | 対象 app repo 名 |
| `webhook_active` | Boolean | Webhook 登録済みかどうか（E-1 完了後に `true` に更新） |
| `registered_at` | String | ISO 8601 |

`/create` 実行時にこのアイテムを `webhook_active: false` で書き込む。Webhook 登録（E-1 完了後）に `true` に更新する。`/issue` で repo 省略時は `webhook_active: true` のものだけ一覧表示する。

**PR ごとの canonical レコード方針**：`implement` Worker が作成するジョブレコードを、そのPRのlifecycle全体を通じた唯一のレコードとする（canonical PR record）。repair 時は新しいジョブレコードを作成せず、Orchestrator が既存レコードの `head_sha`・`repair_attempt`・`status` を in-place で更新する。これにより `pr-number-index` / `head-sha-index` で常に1件のみ返ることを保証する。repair ジョブの SQS メッセージには元の `job_id`（`implement` Worker が生成したもの）を含め、Worker はそのIDで canonical レコードを参照・更新する。

**GSI（Global Secondary Index）**

| GSI 名 | PK | SK | 用途 |
|---|---|---|---|
| `pr-number-index` | `repo`（String） | `pr_number`（Number） | Webhook の `pull_request` イベントから canonical PR レコードを特定（PR 1件につき1行） |
| `head-sha-index` | `head_sha`（String） | — | Webhook の `workflow_run` および `pull_request.synchronize` イベントから canonical PR レコードを特定（`head_sha` は repair のたびに更新される） |

---

## 3. パイプラインステート設計

### 3.1 Issue ラベル（ステート遷移）

| ラベル | 意味 | 付与主体 |
|---|---|---|
| `要件定義作成` | 要件定義ジョブのトリガー | 人間 |
| `in-requirements` | 要件定義 Worker 実行中 | Worker |
| `waiting-for-answer` | 人間の回答待ち | Worker |
| `詳細設計作成` | 設計ジョブのトリガー | Worker |
| `in-design` | 設計 Worker 実行中 | Worker |
| `タスク分割` | タスク分割ジョブのトリガー | Worker |
| `in-task-split` | タスク分割 Worker 実行中 | Worker |
| `ready-for-impl` | 実装ジョブのトリガー | Worker |
| `in-impl` | 実装 Worker 実行中 | Worker |
| `blocked` | 自動処理停止・要人間対応 | Worker / Orchestrator |
| `needs-human` | 上限超過・人間介入必要 | Orchestrator |

### 3.2 PR ラベル

| ラベル | 意味 | 付与主体 | MVP |
|---|---|---|---|
| `ai-generated` | Worker が生成した PR | Worker | ○ |
| `ready-for-human-review` | 人間レビュー待ち（CI 成功後のみ付与） | Orchestrator | ○ |
| `ready-for-auto-review` | レビュー Worker トリガー | Worker | 将来 |
| `needs-fix` | 差し戻し（同一 PR ブランチで修正継続） | Worker | 将来 |
| `in-review` | レビュー Worker 実行中 | Worker | 将来 |
| `review-passed` | レビュー承認済み | Worker | 将来 |
| `ready-to-merge` | マージ条件を全て満たした | Orchestrator | 将来 |
| `auto-merge` | 自動マージ許可 | 人間 | 将来 |
| `blocked` | 自動処理停止 | Worker / Orchestrator | ○ |

### 3.3 Issue ステートマシン

                        人間が /issue
                             │
                             ▼
                       要件定義作成
                             │
              ┌──────────────┤
              │              │
         質問あり         質問なし
              │              │
      waiting-for-answer   詳細設計作成
              │              │
       人間が回答            ▼
              │          in-design
       要件定義作成            │
              │          タスク分割
              │              │
              └──────────────┘
                             │
                        in-task-split
                             │
                    子 Issue × N 起票
                    各子 Issue に ready-for-impl
                             │
                         in-impl
                             │
                          PR 作成
                ready-for-human-review (PR)

---

## 4. 要件定義・設計フェーズ詳細

### 4.1 要件定義 Worker

**ジョブ種別**：`requirements`  
**トリガーラベル**：`要件定義作成`  
**コミット先**：対象 app repo  
**成果物置き場**：`docs/<issue-number>-requirements.md`

**ドキュメントフォーマット**

# 要件定義：<タイトル>

## 背景・目的
## 機能要件
## 非機能要件
## 制約条件
## 未解決事項（質問往復の記録）

**処理フロー**
1. `blocked` チェック。付いていればスキップ。
2. Issue ラベルを `in-requirements` に遷移。
3. Issue 本文・コメント履歴を読み込み、LLM で要件整理。
4. 曖昧点ありかつ質問往復 < 2 の場合：質問コメントを投稿 → `in-requirements` を除去し `waiting-for-answer` を付与。
5. 要件確定できる場合（曖昧点なし、または質問往復 ≥ 2）：
   - 対象 app repo の `main` ブランチに `docs/<issue-number>-requirements.md` を直接コミット・プッシュ（MVP前提：app repo の `main` には branch protection を設定しない。Worker の PAT が直接プッシュできる状態を想定）
   - Issue に成果物 URL をコメント
   - `in-requirements` を除去し `詳細設計作成` を付与
6. 失敗時：`blocked` を付与しコメントで通知。

**質問往復上限**：2往復。超過時は現時点の情報で強制確定。

### 4.2 設計 Worker

**ジョブ種別**：`design`  
**トリガーラベル**：`詳細設計作成`  
**コミット先**：対象 app repo  
**成果物置き場**：対象 app repo の `main` ブランチ `docs/<issue-number>-design.md`

**処理フロー**
1. `blocked` チェック。
2. `in-design` に遷移。
3. 対象 app repo の `main` ブランチから `docs/<issue-number>-requirements.md` を読み込み、設計書を生成して `main` に `docs/<issue-number>-design.md` を直接コミット・プッシュ。
4. Issue に設計書 URL をコメント → `in-design` を除去し `タスク分割` を付与。
5. 失敗時：`blocked` を付与。

**設計書フォーマット**

# 設計書：<タイトル>

## 目的・背景
## 実装方針
## 制約条件
## 作業分割方針（タスク分割 Worker への指示）
## 検証方法

### 4.3 タスク分割 Worker

**ジョブ種別**：`task-split`  
**トリガーラベル**：`タスク分割`  
**対象**：対象 app repo の Issue / docs

**処理フロー**
1. `blocked` チェック。
2. `in-task-split` に遷移。
3. 設計書の作業分割方針を読み込み、1 Worker が 1 PR で完結できる粒度に分割。
4. 各タスクを子 Issue として対象 app repo に起票。本文の**1行目**に `parent_issue: <番号>` という形式で親 Issue 番号を記載する（例：`parent_issue: 42`）。implement Worker は正規表現 `^parent_issue:\s*(\d+)` で解析する。
5. 各子 Issue に `ready-for-impl` を付与。
6. 親 Issue に分割完了コメントを投稿 → `in-task-split` を除去。
7. 失敗時：`blocked` を付与。

---

## 5. 実装・修正フェーズ詳細（MVP）

### 5.1 実装 Worker

**ジョブ種別**：`implement`  
**トリガーラベル**：`ready-for-impl`  
**対象**：対象 app repo

**処理フロー**
1. `blocked` チェック。
2. Issue の `ready-for-impl` を除去し `in-impl` を付与。
3. 子 Issue の本文1行目から正規表現 `^parent_issue:\s*(\d+)` で親 Issue 番号を取得する（タスク分割 Worker が子 Issue 作成時に記載。マッチしない場合は `blocked` を付与して終了）。取得後、State DB のジョブレコードに `parent_issue_number` として書き込む（後続処理のログ用）。
4. 対象 app repo をクローン。
5. `impl/issue-<番号>-<slug>` ブランチを作成・プッシュ。State DB に `branch_name` を記録。
6. `main` ブランチの `docs/<親issue番号>-design.md` を参照して Claude Code で実装。
7. 変更をコミット・プッシュ。State DB に `head_sha` を記録。`implement_completed` を `true` に設定。
8. PR を作成（`ai-generated` ラベル付与）。State DB に `pr_number` を記録。
9. Issue の `in-impl` を除去。
10. 失敗時：`blocked` を付与。

### 5.2 repair Worker

**ジョブ種別**：`repair`  
**トリガー**：Orchestrator が CI 失敗を検知して SQS に積む  
**対象**：対象 app repo の PR ブランチ

**処理フロー**
1. State DB で repair_attempt を確認。上限（3回）超過で `needs-human` に切り替え。
2. PR ブランチをクローン。
3. CI 失敗ログを取得（GitHub API）。`head_sha` と `workflow_run_id` で対象実行を特定。
4. Claude Code でエラーを修正・コミット・プッシュ。State DB の `head_sha` を更新。
5. State DB を `ci-running` に更新。
6. PR synchronize により GitHub Actions CI が自動再実行される。

---

## 6. 将来フェーズ（MVP対象外）

### 6.1 レビュー Worker

**ジョブ種別**：`review`  
**トリガーラベル**：`ready-for-auto-review`（PR）

MVPでは `ready-for-human-review` は人間レビューのシグナルとしてのみ機能する。将来フェーズで自動レビュー Worker を追加し、以下を実装する。

**処理フロー**
1. `blocked` チェック。レビュー試行上限（3回）チェック。
2. `in-review` に遷移。
3. 差分・PR 概要・設計書を読み込み、レビューコメントを投稿。
4. `ACTION: APPROVE` → `in-review` を除去、`review-passed` を付与。
5. `ACTION: REQUEST_CHANGES` → `in-review` を除去、`needs-fix` を付与。
6. 上限超過 / 判定不能：`blocked` + `needs-human` を付与。

**レビューコメントフォーマット**

## 自動レビュー結果

### P1（必須修正）
### P2（推奨修正）
### 総評

ACTION: APPROVE | REQUEST_CHANGES

**差し戻し方針**：PR はクローズしない。同一ブランチ・同一 PR 上で修正を継続。

### 6.2 GitHub Apps への移行

MVPでは PAT でラベル操作を行うが、以下の課題がある。

- スコープが粗く、組織全体に権限が及ぶリスク
- トークン管理・ローテーションのコスト
- per-repo 権限制御が困難

将来フェーズで GitHub Apps に移行し、per-installation トークンで最小権限を実現する。

### 6.3 統合条件（レビュー Worker 導入後）

レビュー Worker 導入後は以下の条件をすべて満たした場合にのみマージを実施する。

| 条件 | 検証方法 |
|---|---|
| `review-passed` ラベルが付いている | ラベル存在確認 |
| CI（lint / test）がすべて成功 | `gh pr checks` で全件 success 確認 |
| `blocked` ラベルが付いていない | ラベル不在確認 |
| `auto-merge` ラベルがある場合のみ自動マージ | ラベル存在確認 |

`auto-merge` がない場合は `ready-to-merge` を付与して人間の判断を待つ。

---

## 7. マルチリポジトリ設計

### 7.1 /create コマンド

Discord: /create <app-name>
  ↓
processor:
  1. gh repo create haruvv/<app-name> --public
  2. dev-agents からワークフロー雛形・初期 docs 雛形・ラベル定義を取得
  3. それらを対象 app repo に配置
  4. Secrets を登録（MVP暫定）
  5. Actions 権限を有効化
  6. State DB にリポジトリを登録
  7. Discord に完了通知

### 7.2 各 app repo に集約するもの

各 `haruvv/<app-name>` repo には少なくとも以下を配置する。

- `.github/workflows/`
- `docs/`
- 実装コード
- Issue / PR
- ラベル
- AGENTS.md または CLAUDE.md
- repo 固有の制約・運用ルール

### 7.3 dev-agents に置くもの

`haruvv/dev-agents` には少なくとも以下を置く。

- workflow 雛形
- ラベル定義の元データ
- docs 雛形
- AGENTS.md / CLAUDE.md の初期テンプレート
- bootstrap / 同期用スクリプト

`dev-agents` は配布元であり、app repo の履歴・知識蓄積先を代替しない。

---

## 8. 観測可能性設計

| 手段 | 内容 |
|---|---|
| GitHub Issue / PR コメント | Worker が処理開始・主要アクション・結果・次アクションをコメントとして記録 |
| State DB | 全ジョブのステータス・試行回数・タイムスタンプを管理 |
| GitHub Actions ログ | トリガー検知・SQS 送信・CI 実行の記録 |
| ECS タスクログ | Worker の実行ログ（CloudWatch Logs） |
| CI 結果 | GitHub Checks API |

---

## 9. 移行フェーズ

| フェーズ | 内容 | 状態 |
|---|---|---|
| フェーズ1 | GitHub Actions のみでパイプライン動作確認（sandbox repo） | 完了（v1 検証） |
| フェーズ2 | ECS Fargate Worker 実装・各 app repo に workflow 集約 | **完了** |
| フェーズ3 | Orchestrator に Webhook エンドポイント追加・State DB 構築 | **完了** |
| フェーズ4 | `/create` コマンド実装・テンプレート配布自動化 | **完了** |
| フェーズ5 | E2E 動作確認（requirements → design → task-split → implement → PR） | **動作確認済み** |
| フェーズ6 | レビュー Worker 実装・GitHub Apps 移行 | 将来 |

---

## 10. 制約一覧

| 制約 | 内容 |
|---|---|
| docs/ 記録の必須化 | 設計判断・意思決定は必ず対象 app repo の `docs/` に記録する |
| 成果物の明示的コミット | 要件定義・設計書は対象 app repo の `main` ブランチ `docs/<issue-number>-{requirements,design}.md` に直接コミットする（MVP前提：`main` に branch protection なし） |
| blocked の遵守 | `blocked` ラベルを無視した処理続行は禁止 |
| 秘密情報の禁止 | API キー・トークン等をコードやコミットに含めない |
| 自動マージ制限 | `auto-merge` ラベルの明示的付与なしに自動マージしない（将来フェーズ） |
| 再試行上限 | 各 Worker の再試行は上限を設け、無制限な試行を禁止する |
| PAT 使用は MVP 暫定 | Worker・Orchestrator からのラベル変更は PAT を使用（MVP暫定）。将来フェーズで GitHub Apps に移行する |
