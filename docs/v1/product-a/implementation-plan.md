# 実装計画: プロダクトA

## アーキテクチャ決定

### LangGraph Server（HTTP）を使わない理由

設計書では LangGraph Platform を ECS 上に立てて handler Lambda から HTTP で叩く案を検討したが、
handler Lambda が VPC 外に存在するため内部 ALB に到達できない問題がある。

→ **既存の SQS → ECS タスクパターンを踏襲する**。LangGraph はライブラリとして使い、
サーバーは立てない。interrupt()/resume() の代わりに DynamoDB で Discord コンテキストを管理する。

### 確定アーキテクチャ

```
Discord /chat <message>
  ↓
handler Lambda（ACK + SQS 送信）
  ↓ {user_id, content, application_id, token}
product-a ECS タスク（LangGraph エージェント）
  ├─ DynamoDB から会話履歴を読み込む（thread_id = user_id）
  ├─ LangGraph エージェント実行
  ├─ 雑談・調査 → Discord にフォローアップメッセージ送信
  └─ 開発タスク → GitHub Issue 起票
        DynamoDB に {issue_number → discord_context} を保存
        Discord に「依頼受け付けました」送信

CI 成功 →
Orchestrator Lambda
  ├─ ready-for-human-review 付与（既存処理）
  └─ DynamoDB から discord_context を取得 → Discord に完了通知送信
```

### 変更点サマリー

| コンポーネント | 変更内容 |
|---|---|
| handler Lambda | SQS 送信に変更（processor Lambda invoke を置き換え） |
| processor Lambda | **廃止** |
| product-a ECS タスク | **新規追加**（LangGraph エージェント） |
| Orchestrator Lambda | CI 成功時に Discord 通知を追加 |
| DynamoDB | 会話履歴・Discord コンテキスト保存カラムを追加（スキーマレスのため追加不要） |
| Discord コマンド | `/issue` → `/chat` に変更 |
| Terraform | product-a タスク定義追加、processor Lambda 削除 |

---

## 依存グラフ

```
Task 1: ディレクトリ・設定ファイル
  ↓
Task 2: tools/discord.py      Task 3: tools/web_search.py    Task 4: tools/dev_agents.py
  └──────────────────────────────────┬───────────────────────────────┘
                                     ↓
                            Task 5: agent.py（ツールを統合）
                                     ↓
                            Task 6: entrypoint.py（ECS エントリーポイント）
                                     ↓
Task 7: Terraform（product-a ECS タスク定義）    Task 8: Terraform（processor 廃止）
  └───────────────────────────────────┬──────────────────────────────┘
                                      ↓
                            Task 9: handler Lambda 変更
                                      ↓
                            Task 10: Orchestrator Lambda 変更
                                      ↓
                            Task 11: Discord コマンド更新 + ビルドスクリプト
```

---

## タスク一覧

### Phase 1: product-a コード

---

#### Task 1: ディレクトリ構造・設定ファイル

**説明**: `agent-intake/src/product-a/` を作成し、Dockerfile・langgraph.json（将来用）・requirements.txt を配置する。

**Acceptance criteria:**
- [ ] `src/product-a/Dockerfile` が存在し、Python 3.12 ベースで LangGraph・langchain-anthropic・duckduckgo-search をインストールする
- [ ] `src/product-a/requirements.txt` に依存パッケージが列挙されている
- [ ] `src/product-a/tools/` ディレクトリが存在する

**Verification:**
- [ ] `docker build` が成功する

**Dependencies:** なし

**Files:**
- `src/product-a/Dockerfile`
- `src/product-a/requirements.txt`
- `src/product-a/tools/__init__.py`

**Scope:** S

---

#### Task 2: tools/discord.py

**説明**: Discord の follow-up メッセージ送信ツール。handler Lambda が ACK を返した後、エージェントが Discord に追加メッセージを送るための関数。

**Acceptance criteria:**
- [ ] `send_followup(application_id, token, content)` が Discord Webhook API を叩いてメッセージを送信する
- [ ] `send_message(channel_id, content)` が Discord Bot API を叩いて通常メッセージを送信する（Orchestrator からも使用）
- [ ] HTTP エラー時は例外を raise する

**Verification:**
- [ ] 手動で関数を呼び出してメッセージが届くことを確認

**Dependencies:** Task 1

**Files:**
- `src/product-a/tools/discord.py`

**Scope:** S

---

#### Task 3: tools/web_search.py

**説明**: DuckDuckGo 検索ツール（API キー不要）。調査タスクをエージェントが処理するために使用。

**Acceptance criteria:**
- [ ] `web_search(query: str) -> str` が DuckDuckGo で検索し、上位5件の結果（タイトル + URL + スニペット）を返す
- [ ] LangGraph `@tool` デコレーターで定義されている

**Verification:**
- [ ] `web_search("LangGraph")` を呼んで結果が返ることを確認

**Dependencies:** Task 1

**Files:**
- `src/product-a/tools/web_search.py`

**Scope:** S

---

#### Task 4: tools/dev_agents.py

**説明**: dev-agents パイプラインに開発タスクを委譲するツール。GitHub Issue を起票し、Discord コンテキスト（user_id・application_id・token）を DynamoDB に保存する。

**Acceptance criteria:**
- [ ] `create_dev_issue(task: str, user_id: str, application_id: str, token: str) -> str` が dev-agents に Issue を起票する
- [ ] 起票した Issue 番号をキーに DynamoDB へ discord_context を保存する（`DISCORD#<issue_number>` アイテム）
- [ ] 戻り値は GitHub Issue の URL

**Verification:**
- [ ] Issue が dev-agents に起票されることを確認
- [ ] DynamoDB に discord_context が保存されることを確認

**Dependencies:** Task 1

**Files:**
- `src/product-a/tools/dev_agents.py`

**Scope:** M

---

#### Task 5: agent.py

**説明**: LangGraph エージェント本体。Claude を LLM として使用し、上記ツールをバインドした Supervisor パターンのエージェント。会話履歴は DynamoDB から読み込む。

**Acceptance criteria:**
- [ ] Claude（claude-sonnet-4-6）を LLM として使用する
- [ ] `web_search` / `create_dev_issue` / `reply_to_discord` ツールを持つ
- [ ] `run(user_id, content, application_id, token)` 関数が会話を1ターン処理して Discord に返答する
- [ ] 会話履歴を DynamoDB の `THREAD#<user_id>` アイテムに保存・読み込みする（最大20件保持）

**Verification:**
- [ ] ローカルで `run()` を呼んでエージェントが動作することを確認

**Dependencies:** Task 2, 3, 4

**Files:**
- `src/product-a/agent.py`

**Scope:** M

---

#### Task 6: entrypoint.py

**説明**: ECS タスクのエントリーポイント。`JOB_PAYLOAD` 環境変数から Discord メッセージのペイロードを受け取り、エージェントを起動する。既存の Worker と同じ構造。

**Acceptance criteria:**
- [ ] `JOB_PAYLOAD` を JSON パースして `user_id`, `content`, `application_id`, `token` を取得する
- [ ] `agent.run()` を呼び出す
- [ ] 例外発生時は Discord にエラーメッセージを送信して exit(1) する

**Verification:**
- [ ] `JOB_PAYLOAD` を環境変数にセットして起動確認

**Dependencies:** Task 5

**Files:**
- `src/product-a/entrypoint.py`

**Scope:** S

---

### Checkpoint: Phase 1

- [ ] `docker build src/product-a` が成功する
- [ ] ローカルで `JOB_PAYLOAD` を渡してエージェントが動作する
- [ ] Discord にメッセージが届く

---

### Phase 2: インフラ（Terraform）

---

#### Task 7: product-a ECS タスク定義

**説明**: product-a 用の ECS タスク定義・ECR リポジトリ・セキュリティグループを Terraform に追加。既存の worker タスク定義を参考に作成。

**Acceptance criteria:**
- [ ] `aws_ecr_repository.product_a` が定義されている
- [ ] `aws_ecs_task_definition.product_a` が定義されている（Secrets Manager から `GITHUB_TOKEN` と `CLAUDE_CODE_OAUTH_TOKEN` を取得）
- [ ] `aws_security_group.product_a` が定義されている（HTTPS アウトバウンドのみ）
- [ ] `terraform plan` でエラーが出ない

**Verification:**
- [ ] `terraform apply` で ECS タスク定義が作成される

**Dependencies:** Task 6

**Files:**
- `infra/terraform/ecs.tf`（product-a 追記）

**Scope:** M

---

#### Task 8: processor Lambda 廃止・handler Lambda 環境変数変更

**説明**: processor Lambda リソースを Terraform から削除。handler Lambda の環境変数を `PROCESSOR_FUNCTION_NAME` → `JOB_QUEUE_URL` に変更。IAM ポリシーから不要な Lambda invoke 権限を削除。

**Acceptance criteria:**
- [ ] `aws_lambda_function.processor` が terraform から削除されている
- [ ] `aws_lambda_function_event_invoke_config.processor` が削除されている
- [ ] handler Lambda の環境変数に `JOB_QUEUE_URL` が設定されている
- [ ] `terraform plan` でエラーが出ない

**Verification:**
- [ ] `terraform apply` で processor Lambda が削除される

**Dependencies:** なし（Task 7 と並列可）

**Files:**
- `infra/terraform/lambda.tf`

**Scope:** S

---

### Checkpoint: Phase 2

- [ ] `terraform apply` が成功する
- [ ] product-a の ECR リポジトリが存在する
- [ ] processor Lambda が削除されている

---

### Phase 3: 既存コンポーネント変更

---

#### Task 9: handler Lambda 変更

**説明**: handler Lambda が processor Lambda を invoke する代わりに、SQS に Discord メッセージを投入するよう変更。`/chat` コマンドのペイロードを整形して送信する。

**Acceptance criteria:**
- [ ] `PROCESSOR_FUNCTION_NAME` の代わりに `JOB_QUEUE_URL` 環境変数を参照する
- [ ] `boto3.client("sqs").send_message()` で SQS に `{job_type: "chat", user_id, content, application_id, token}` を送信する
- [ ] `/chat` コマンドの `message` オプションの値を `content` として取得する
- [ ] 既存の PING/PONG・署名検証ロジックは変更なし

**Verification:**
- [ ] Discord から `/chat` を送って SQS にメッセージが届くことを確認

**Dependencies:** Task 8

**Files:**
- `src/handler/handler.py`

**Scope:** S

---

#### Task 10: Orchestrator Lambda 変更

**説明**: CI 成功（`ready-for-human-review` 付与）時に DynamoDB から discord_context を読み込み、Discord にPR完了通知を送信する。

**Acceptance criteria:**
- [ ] `_handle_ci_success()` 内で `DISCORD#<issue_number>` アイテムを DynamoDB から取得する
- [ ] discord_context が存在する場合、`send_message(channel_id, content)` で Discord に PR URL を通知する
- [ ] discord_context が存在しない場合は従来通り（ラベル付与のみ）
- [ ] 既存の repair ジョブ投入・ラベル操作ロジックは変更なし

**Verification:**
- [ ] dev-agents に Issue を起票してパイプラインを走らせ、CI 成功後に Discord に通知が来ることを確認

**Dependencies:** Task 4, Task 8

**Files:**
- `src/orchestrator/orchestrator.py`

**Scope:** S

---

### Checkpoint: Phase 3

- [ ] handler Lambda のビルド・デプロイ成功
- [ ] orchestrator Lambda のビルド・デプロイ成功
- [ ] `/chat` → SQS → product-a ECS → Discord 返答 の E2E が動作する

---

### Phase 4: Discord コマンド更新・ビルド整備

---

#### Task 11: Discord コマンド更新・ビルドスクリプト追加

**説明**: Discord コマンドを `/issue` から `/chat` に変更する（`register_commands.py` を更新）。product-a のビルド・ECR push スクリプトを追加または既存スクリプトを拡張する。

**Acceptance criteria:**
- [ ] `register_commands.py` が `/chat message:string` コマンドを登録する
- [ ] `scripts/build_product_a.sh` が product-a イメージをビルドして ECR に push する
- [ ] `python scripts/register_commands.py` を実行してコマンドが更新される

**Verification:**
- [ ] Discord で `/chat` が候補に出る
- [ ] ECR に `product-a:latest` イメージが push されている

**Dependencies:** Task 9

**Files:**
- `scripts/register_commands.py`
- `scripts/build_product_a.sh`

**Scope:** S

---

### Checkpoint: 完了

- [ ] Discord `/chat こんにちは` → エージェントが返答する
- [ ] Discord `/chat LangGraphについて調べて` → Web 検索して返答する
- [ ] Discord `/chat ○○を作って` → dev-agents に Issue が起票され、CI 完了後に PR URL が通知される

---

## リスクと対策

| リスク | 影響 | 対策 |
|---|---|---|
| sqs-to-ecs Lambda が `chat` job_type を知らない | ECS タスクが起動しない | sqs-to-ecs の分岐に `chat` を追加（Task 7 で対応） |
| DuckDuckGo の非公式 API が壊れる | 検索ツールが機能しない | フォールバックメッセージを返す |
| Claude の tool_choice が意図しないツールを選ぶ | 誤動作 | システムプロンプトで用途を明記 |
| 会話履歴が DynamoDB に蓄積し続ける | コスト増・レイテンシ増 | 最大 20 件保持（古いものは削除）で対処 |

## 未決定事項

| 項目 | 内容 |
|---|---|
| sqs-to-ecs Lambda の job_type 分岐 | `chat` タイプを追加して product-a タスク定義に向ける必要がある（Task 7 で確認） |
| LLM | Gemini 2.0 Flash（`langchain-google-genai`）を使用。既存の `GEMMA_API_KEY` をそのまま流用するため新規キー不要 |
