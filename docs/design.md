# AI自律開発エージェントパイプライン 設計書

本書は要件定義書（`docs/requirement.md`）に基づく自律開発パイプラインの設計を定義する。

---

## 1. システム全体設計

### 1.1 設計思想

パイプラインを「受付層」「オーケストレーション層」「実行層」の3層に分離する。

| 層 | 担当 | 主要コンポーネント |
|---|---|---|
| 受付層 | 人間からの要求受付・GitHub Issue起票 | agent-intake（Lambda） |
| オーケストレーション層 | ステート管理・ジョブ発行・CI結果処理 | GitHub Actions（トリガー）/ Orchestrator |
| 実行層 | 実際のコード実装・コミット・PR作成 | ECS Fargate Worker |

**GitHub Actions はトリガーと CI に限定する。** 実装・修正などの重処理は ECS Fargate Worker に委譲する。これにより Actions のツール制限・タイムアウト・イベント発火制約を回避し、Worker 側で自由な実行環境を確保する。

### 1.2 全体フロー

```
Human
  │ Discord /issue または /create コマンド（自然言語）
  ▼
agent-intake（Lambda: handler + processor）
  │ LLM（Gemma）で自然言語を解釈
  │ 曖昧な場合は Discord 上で確認
  ▼
GitHub Issue 起票
  │ 「要件定義作成」ラベル付与
  ▼
GitHub Actions（ラベルイベント検知 → SQS にジョブ送信）
  ▼
SQS → EventBridge Pipes
  ▼
ECS Fargate Worker
  ├─ 要件定義・設計・タスク分割フェーズ：
  │    Issue 上で LLM を実行、コメント投稿・ラベル更新
  │    次フェーズのラベルを PAT で付与 → GitHub Actionsが次ジョブを発行
  │
  └─ 実装フェーズ：
       対象リポジトリをクローン
       実装ブランチ作成
       Claude Code でコード実装・コミット・プッシュ
       PR 作成 + ready-for-review ラベル付与（PAT）
  ▼
GitHub Actions（CI: lint / test）
  ▼
GitHub Webhook → Orchestrator
  ├─ CI 成功 → State DB 更新（ci-passed）→ 人間レビュー待ち
  └─ CI 失敗 → State DB 更新（ci-failed）
              → SQS に repair job を積む
              → ECS Fargate Worker（修正・再コミット）
              → CI 再実行
              → 上限到達で needs-human エスカレーション
  ▼
人間レビュー → approve → マージ
```

### 1.3 リポジトリ構成

| リポジトリ | 役割 |
|---|---|
| `haruvv/agent-intake` | Discord Bot・Orchestrator（Lambda） |
| `haruvv/dev-agents` | パイプライン定義テンプレート（ワークフロー・ラベル定義） |
| `haruvv/<app-name>` | 実装先（`/create` で動的生成） |

`dev-agents` はテンプレートリポジトリとして機能し、`/create` 実行時にワークフロー・ラベル・Secrets が新リポジトリにコピーされる。

---

## 2. コンポーネント設計

### 2.1 agent-intake（受付層 + Orchestrator）

**リポジトリ**：`haruvv/agent-intake`

#### 2.1.1 Discord Bot（Lambda 2関数構成）

Discord のインタラクション受信から3秒以内の ACK が必要なため、handler と processor を分離する。

```
Discord POST /interactions
  ↓
API Gateway
  ↓
Lambda: handler
  署名検証（Ed25519）
  type 1 → PONG 即時返答
  type 2 → type 5 ACK 即時返答 + processor を非同期起動
  ↓
Lambda: processor
  Gemma で自然言語解釈
  曖昧 → Discord に確認メッセージ返信
  明確 → GitHub Issue 起票（「要件定義作成」ラベル付与）
       → Discord に Issue URL を返信
```

| 関数 | 役割 | タイムアウト目安 |
|---|---|---|
| `agent-intake-handler` | 署名検証・即時 ACK・processor 起動 | 3秒以内 |
| `agent-intake-processor` | LLM 解釈・Issue 起票・Discord 返信 | 最大30秒 |

#### 2.1.2 Orchestrator（GitHub Webhook 処理）

agent-intake に `/webhook` エンドポイントを追加し、CI 結果・PR イベントをハンドリングする。

**受信イベント**：
- `workflow_run`（CI 完了）
- `pull_request`（PR オープン / クローズ）

**処理内容**：

| イベント | アクション |
|---|---|
| CI 成功 | State DB を `ci-passed` に更新 |
| CI 失敗（repair 上限内） | State DB を `ci-failed` に更新 → SQS に repair job を積む |
| CI 失敗（repair 上限超過） | `needs-human` ラベル付与 + 通知 |

#### 2.1.3 /create コマンド（マルチリポジトリ対応）

`/create <app-name>` 受信時に processor が以下を実行する：

1. GitHub API で新規リポジトリ作成
2. `dev-agents` のワークフローファイルをコピー
3. ラベルを一括作成
4. `CLAUDE_CODE_OAUTH_TOKEN`・`GH_PAT` を Secrets として登録
5. Actions 権限（PR 作成許可）を有効化
6. State DB にリポジトリ情報を登録

#### 2.1.4 環境変数

| 変数 | 対象関数 | 内容 |
|---|---|---|
| `DISCORD_PUBLIC_KEY` | handler | Discord アプリ公開鍵 |
| `PROCESSOR_FUNCTION_NAME` | handler | processor の Lambda 関数名 |
| `GEMMA_API_KEY` | processor | Gemma API キー |
| `GITHUB_TOKEN` | processor | GitHub PAT |
| `GITHUB_REPO` | processor | デフォルト Issue 起票先 |
| `CLAUDE_CODE_OAUTH_TOKEN` | processor | 新リポジトリへの Secret 転写用 |
| `GH_PAT` | processor | 新リポジトリへの Secret 転写用 |
| `JOB_QUEUE_URL` | processor | SQS キュー URL |
| `STATE_TABLE_NAME` | processor / orchestrator | DynamoDB テーブル名 |

#### 2.1.5 エラー処理方針

- Discord トークン期限切れ（404）はエラーを無視・リトライしない
- processor の Lambda 非同期リトライは 0 回（重複 Issue 起票防止）
- 曖昧な要求は Issue を起票せず Discord 上で確認

#### 2.1.6 ウォームアップ戦略

Lambda コールドスタートが Discord の3秒制限に近いため、EventBridge で5分ごとに handler を ping してウォーム状態を維持する。

---

### 2.2 GitHub Actions（オーケストレーション層）

**役割をラベルイベント検知と SQS へのジョブ送信に限定する。** 処理時間は数秒以内。

#### 2.2.1 トリガーワークフロー構成

各エージェントに対応するワークフローが `issues: labeled` / `pull_request: labeled` イベントを検知し、SQS にジョブメッセージを送信する。

```yaml
# 例：requirements-trigger.yml
on:
  issues:
    types: [labeled]

jobs:
  dispatch:
    if: github.event.label.name == '要件定義作成'
    runs-on: ubuntu-latest
    steps:
      - name: Send job to SQS
        run: |
          aws sqs send-message \
            --queue-url ${{ secrets.JOB_QUEUE_URL }} \
            --message-body '{
              "job_type": "requirements",
              "issue_number": "${{ github.event.issue.number }}",
              "repo": "${{ github.repository }}"
            }'
```

#### 2.2.2 CI ワークフロー

```yaml
on:
  pull_request:
    branches: [main]
  push:
    branches: [main]
```

PR マージ前検証と main 保護を最低限とする。feature ブランチへの push では起動しない。

**MVP フェーズの CI 構成**：

| フェーズ | CI 内容 |
|---|---|
| MVP 初期 | ワークフロー構文チェック（actionlint） |
| 実装開始後 | リント・型検査 |
| テスト追加後 | 単体テスト |

---

### 2.3 ECS Fargate Worker（実行層）

Claude Code をネイティブ実行する自律エージェント。ジョブ種別に応じた処理を実行し、完了後に次フェーズへバトンを渡す。

#### 2.3.1 起動フロー

```
SQS メッセージ受信
  ↓ EventBridge Pipes
ECS Fargate タスク起動（ジョブメッセージをパラメータとして受け取る）
  ↓
State DB で job_id を in-progress に更新（二重起動防止）
  ↓
ジョブ種別に応じた処理を実行
  ↓
State DB を completed / failed に更新
  ↓
タスク終了（使い捨て）
```

#### 2.3.2 ジョブ種別と処理内容

| ジョブ種別 | トリガーラベル | Worker の処理 | 次フェーズへの引き渡し |
|---|---|---|---|
| `requirements` | 要件定義作成 | Issue・コメント読み込み → LLM で要件整理 → 質問コメント投稿 or 要件確定コメント投稿 | ラベル更新（PAT）で次ジョブ発行 |
| `design` | 詳細設計作成 | 要件確定コメント読み込み → 設計書生成 → リポジトリにコミット | ラベル更新（PAT） |
| `task-split` | タスク分割 | 設計書読み込み → 子 Issue 起票 → 各子 Issue に ready-for-impl 付与 | ラベル付与（PAT）→ impl ジョブ発行 |
| `implement` | ready-for-impl | リポジトリクローン → ブランチ作成 → Claude Code で実装 → コミット → プッシュ → PR 作成 | PR に ready-for-review ラベル付与（PAT） |
| `repair` | CI 失敗（Orchestrator から） | PR ブランチをクローン → CI 失敗ログ取得 → Claude Code で修正 → コミット → プッシュ | PR に ready-for-review ラベル付与（PAT） |

#### 2.3.3 ラベル更新の PAT 必須理由

GitHub Actions の `GITHUB_TOKEN` によるラベル変更は `labeled` イベントを発火しない（無限ループ防止の仕様）。Worker が PAT を使用してラベルを更新することで、次のトリガーワークフローが正常に発火する。

---

### 2.4 State DB（DynamoDB）

パイプラインの状態を永続管理し、二重起動防止・repair 試行回数管理・観測可能性を提供する。

| 属性 | 型 | 説明 |
|---|---|---|
| `job_id` | PK（String） | UUID |
| `issue_number` | Number | GitHub Issue 番号 |
| `repo` | String | 対象リポジトリ（例: `haruvv/my-app`） |
| `job_type` | String | `requirements` / `design` / `task-split` / `implement` / `repair` |
| `status` | String | `queued` / `in-progress` / `pr-created` / `ci-running` / `ci-passed` / `ci-failed` / `done` / `failed` |
| `pr_number` | Number | 作成された PR 番号 |
| `ci_attempt` | Number | CI 試行回数 |
| `repair_attempt` | Number | repair 試行回数（上限 3） |
| `created_at` | String | ISO 8601 |
| `updated_at` | String | ISO 8601 |

---

## 3. パイプラインステート設計

### 3.1 Issue ラベル（ステート遷移）

| ラベル | 意味 | 付与主体 |
|---|---|---|
| `要件定義作成` | 要件定義ジョブのトリガー | 人間 / タスク分割 Worker |
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

| ラベル | 意味 | 付与主体 |
|---|---|---|
| `ai-generated` | Worker が生成した PR | Worker |
| `ready-for-review` | レビュージョブのトリガー | Worker |
| `in-review` | レビュー Worker 実行中 | Worker |
| `needs-fix` | 差し戻し（同一 PR ブランチで修正継続） | Worker |
| `review-passed` | レビュー承認済み | Worker |
| `ready-to-merge` | マージ条件を全て満たした | Orchestrator |
| `auto-merge` | 自動マージ許可 | 人間 |
| `blocked` | 自動処理停止 | Worker / Orchestrator |

### 3.3 Issue ステートマシン

```
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
                    ready-for-review (PR)
```

---

## 4. 要件定義・設計フェーズ詳細

### 4.1 要件定義 Worker

**ジョブ種別**：`requirements`  
**トリガーラベル**：`要件定義作成`

**処理フロー**：
1. `blocked` チェック。付いていればスキップ。
2. Issue ラベルを `in-requirements` に遷移。
3. Issue 本文・コメント履歴を読み込み、LLM で要件整理。
4. 曖昧点ありかつ質問往復 < 2 の場合：質問コメントを投稿 → `waiting-for-answer` に遷移。
5. 要件確定できる場合（曖昧点なし、または質問往復 ≥ 2）：要件確定コメントを投稿 → `詳細設計作成` を付与。
6. 失敗時：`blocked` を付与しコメントで通知。

**質問往復上限**：2往復。超過時は現時点の情報で強制確定。

### 4.2 設計 Worker

**ジョブ種別**：`design`  
**トリガーラベル**：`詳細設計作成`

**処理フロー**：
1. `blocked` チェック。
2. `in-design` に遷移。
3. 要件確定コメントを読み込み、設計書を生成して `docs/<issue-number>-design.md` にコミット。
4. Issue に設計書 URL をコメント → `タスク分割` を付与。
5. 失敗時：`blocked` を付与。

**設計書フォーマット**：
```markdown
# 設計書：<タイトル>

## 目的・背景
## 実装方針
## 制約条件
## 作業分割方針（タスク分割 Worker への指示）
## 検証方法
```

### 4.3 タスク分割 Worker

**ジョブ種別**：`task-split`  
**トリガーラベル**：`タスク分割`

**処理フロー**：
1. `blocked` チェック。
2. `in-task-split` に遷移。
3. 設計書の作業分割方針を読み込み、1 Worker が 1 PR で完結できる粒度に分割。
4. 各タスクを子 Issue として起票（親 Issue 参照を本文に含める）。
5. 各子 Issue に `ready-for-impl` を付与。
6. 親 Issue に分割完了コメントを投稿 → `in-task-split` を除去。
7. 失敗時：`blocked` を付与。

---

## 5. 実装・修正フェーズ詳細

### 5.1 実装 Worker

**ジョブ種別**：`implement`  
**トリガーラベル**：`ready-for-impl`

**処理フロー**：
1. `blocked` チェック。
2. Issue に `in-impl` を付与。
3. 対象リポジトリをクローン。
4. `impl/issue-<番号>-<slug>` ブランチを作成・プッシュ。
5. 設計書（`docs/<親issue番号>-design.md`）を参照して Claude Code で実装。
6. 変更をコミット・プッシュ。
7. PR を作成（`ai-generated` ラベル付与）。
8. `ready-for-review` を付与（PAT）→ レビュージョブ発行。
9. Issue の `in-impl` を除去。
10. 失敗時：`blocked` を付与。

### 5.2 レビュー Worker

**ジョブ種別**：`review`  
**トリガーラベル**：`ready-for-review`（PR）

**処理フロー**：
1. `blocked` チェック。レビュー試行上限（3回）チェック。
2. `in-review` に遷移。
3. 差分・PR 概要・設計書を読み込み、レビューコメントを投稿。
4. `ACTION: APPROVE` → `in-review` を除去、`review-passed` を付与。
5. `ACTION: REQUEST_CHANGES` → `in-review` を除去、`needs-fix` を付与。
6. 上限超過 / 判定不能：`blocked` + `needs-human` を付与。

**レビューコメントフォーマット**：
```
## 🔍 自動レビュー結果

### P1（必須修正）
### P2（推奨修正）
### 総評

ACTION: APPROVE | REQUEST_CHANGES
```

**差し戻し方針**：PR はクローズしない。同一ブランチ・同一 PR 上で修正を継続。

### 5.3 repair Worker

**ジョブ種別**：`repair`  
**トリガー**：Orchestrator が CI 失敗を検知して SQS に積む

**処理フロー**：
1. State DB で repair_attempt を確認。上限（3回）超過で `needs-human` に切り替え。
2. PR ブランチをクローン。
3. CI 失敗ログを取得（GitHub API）。
4. Claude Code でエラーを修正・コミット・プッシュ。
5. State DB を `ci-running` に更新。
6. GitHub Actions CI が自動再実行される。

---

## 6. マルチリポジトリ設計

### 6.1 /create コマンド

```
Discord: /create <app-name>
  ↓
processor:
  1. gh repo create haruvv/<app-name> --public
  2. dev-agents のワークフローを取得してプッシュ
  3. ラベルを一括作成（dev-agents と同一セット）
  4. CLAUDE_CODE_OAUTH_TOKEN / GH_PAT を Secrets に登録
  5. Actions 権限（PR 作成許可）を有効化
  6. State DB にリポジトリを登録
  7. Discord に完了通知
```

### 6.2 /issue コマンド（拡張後）

```
Discord: /issue [<repo>] <description>
  ↓
processor:
  repo 指定あり → 該当リポジトリに Issue 起票
  repo 省略    → 登録済みリポジトリ一覧を返信して選択を促す
```

---

## 7. 統合条件設計

PR のマージは以下の条件をすべて満たした場合にのみ実施する。

| 条件 | 検証方法 |
|---|---|
| `review-passed` ラベルが付いている | ラベル存在確認 |
| CI（lint / test）がすべて成功 | `gh pr checks` で全件 success 確認 |
| `blocked` ラベルが付いていない | ラベル不在確認 |
| `auto-merge` ラベルがある場合のみ自動マージ | ラベル存在確認 |

`auto-merge` がない場合は `ready-to-merge` を付与して人間の判断を待つ。

---

## 8. 観測可能性設計

| 手段 | 内容 |
|---|---|
| GitHub Issue / PR コメント | Worker が処理開始・主要アクション・結果・次アクションをコメントとして記録 |
| State DB | 全ジョブのステータス・試行回数・タイムスタンプを管理 |
| GitHub Actions ログ | トリガー検知・SQS 送信の記録 |
| ECS タスクログ | Worker の実行ログ（CloudWatch Logs） |
| CI 結果 | GitHub Checks API |

---

## 9. 移行フェーズ

| フェーズ | 内容 | 状態 |
|---|---|---|
| フェーズ1 | GitHub Actions のみでパイプライン動作確認（dev-agents-sandbox） | 完了（v1 検証） |
| フェーズ2 | ECS Fargate Worker 実装・GitHub Actions はトリガーのみに変更 | 次のステップ |
| フェーズ3 | Orchestrator に Webhook エンドポイント追加・State DB 構築 | 未着手 |
| フェーズ4 | `/create` コマンド実装・マルチリポジトリ対応 | 未着手 |

---

## 10. 制約一覧

| 制約 | 内容 |
|---|---|
| docs/ 記録の必須化 | 設計判断・意思決定は必ず `docs/` に記録する |
| blocked の遵守 | `blocked` ラベルを無視した処理続行は禁止 |
| 秘密情報の禁止 | API キー・トークン等をコードやコミットに含めない |
| 自動マージ制限 | `auto-merge` ラベルの明示的付与なしに自動マージしない |
| 再試行上限 | 各 Worker の再試行は上限を設け、無制限な試行を禁止する |
| ラベル付与は PAT で行う | Worker・Orchestrator からのラベル変更は PAT を使用し、イベント発火を保証する |
