# 会話型Dev-Agent 設計書

本書は、Discord ↔ LangGraph ↔ Dev-Agent の会話型アーキテクチャへの移行設計を定義する。

---

## 1. 背景と目的

### 1.1 現状の問題

現状の LangGraph（GLM glm-4.5-flash）は「ルーティング」と「要件ヒアリング＋Issue起票」を一手に担っている。
結果として：

- Issue 起票の精度が低い（チャット内容のコピーペースト・中身の空白）
- LangGraph を強力なモデルに変えるとコストが増大する
- 将来的に営業・事務・コンサル等のエージェントを追加するときに責務が混在する

### 1.2 設計方針

**LangGraph は「宅配便」に徹する。**
メッセージを受け取り、宛先を判断して転送するだけ。会話の中身には関与しない。

**各専門エージェントが Discord と直接会話する。**
Dev-Agent が自ら Discord に質問し、ユーザーの返答を受け取り、要件を詰める。

---

## 2. アーキテクチャ全体

### 2.1 コンポーネント構成

```
agent-intake リポジトリ
├── src/handler/          # Discord Lambda（変更なし）
├── src/langgraph/        # ルーター（大幅縮小）
├── src/dev-agent/        # 新規: 会話型Dev-Agent
├── src/worker/           # ジョブ型Worker（変更なし）
└── src/orchestrator/     # CI/PR制御（変更なし）
```

### 2.2 全体フロー

```
Discord /chat コマンド
  ↓
Lambda: agent-intake-handler（変更なし）
  SQS に job_type=chat を投入
  ↓
SQS → sqs-to-ecs Lambda
  ↓
ECS: LangGraph ルーター（GLM・安価なまま）
  ① DynamoDB でアクティブセッション確認
  ② セッションあり → 対応エージェントの SQS に転送
  ③ セッションなし → 意図分類 → 対応エージェントの SQS に投入 + セッション記録
  ↓
ECS: Dev-Agent（Claude Sonnet・新規）
  Discord と多ターン会話（要件ヒアリング）
  要件確定 → GitHub Issue 起票（構造化）
  セッション終了
  ↓
GitHub Issue（「要件定義作成」ラベル付与）
  ↓
GitHub Actions → SQS → ECS Worker（既存パイプライン・変更なし）
```

### 2.3 SQS キュー構成

| キュー | 用途 | 状態 |
|---|---|---|
| `chat-queue` | handler → LangGraph | 既存 |
| `dev-agent-queue` | LangGraph → Dev-Agent | 新規 |
| `job-queue` | GitHub Actions → Worker | 既存 |

---

## 3. コンポーネント設計

### 3.1 LangGraph（ルーター）

**責務：意図分類とメッセージ転送のみ。会話しない。**

#### 3.1.1 処理フロー

```
メッセージ受信
  ↓
DynamoDB: SESSION#{user_id} を確認
  ├─ セッションあり → 対応エージェントの SQS に転送（分類スキップ）
  └─ セッションなし → GLM で意図分類
                          ├─ "development" → dev-agent-queue に投入 + セッション記録
                          └─ "self_resolve" → GLM で直接回答して終了
                          （将来: "sales", "consulting", ... を追加）
```

#### 3.1.2 廃止する機能

| 機能 | 移管先 |
|---|---|
| 会話履歴管理（`THREAD#{user_id}`） | Dev-Agent |
| `create_github_issue` ツール | Dev-Agent |
| `web_search` ツール | 要検討（self_resolve 用に残すか） |

---

### 3.2 Dev-Agent（新規）

**責務：Discord との多ターン会話による要件ヒアリング → 構造化 Issue 起票。**

#### 3.2.1 ディレクトリ構成

```
src/dev-agent/
├── entrypoint.py      # ECS エントリーポイント・会話状態管理
├── agent.py           # Claude ベース会話エンジン
└── tools/
    ├── discord.py     # Discord 返信（Bot token 使用）
    └── github.py      # 構造化 Issue 起票
```

#### 3.2.2 会話フロー

```
[ユーザー] 「〇〇を作って」
  ↓
[Dev-Agent] 「どんな機能が必要ですか？」→ Discord
  ↓
[ユーザー] 「XXX機能が必要」
  ↓
[Dev-Agent] 「非機能要件はありますか？」→ Discord
  ↓
[ユーザー] 「特になし」
  ↓
[Dev-Agent] 要件を構造化して確認提示 → Discord
  「以下の内容で Issue を起票します。よろしいですか？
   タイトル: ...
   背景: ...
   機能要件: ...
   非機能要件: ...」
  ↓
[ユーザー] 「OK」
  ↓
[Dev-Agent] GitHub Issue 起票
  「Issue #N を起票しました。開発を開始します。」→ Discord
  DynamoDB から SESSION#{user_id} を削除（セッション終了）
```

#### 3.2.3 Issue 構造化スキーマ

```python
class StructuredIssue(BaseModel):
    title: str                    # 簡潔なタイトル（80文字以内）
    background: str               # 背景・目的（1〜3文）
    features: List[str]           # 機能要件（箇条書き）
    non_functional: List[str]     # 非機能要件（箇条書き）
    out_of_scope: List[str]       # 対象外（明示された場合）
    open_questions: List[str]     # 未解決事項（あれば）
```

`create_github_issue` ツールのスキーマとして定義し、Claude に強制することで Issue の中身を保証する。

#### 3.2.4 会話履歴管理

```
DynamoDB キー: THREAD#dev#{user_id}
{
  "job_id":     "THREAD#dev#<user_id>",
  "messages":   [...],   # 会話履歴（最大30ターン）
  "updated_at": "..."
}
```

ECS タスクは起動・終了を繰り返すが、会話履歴は DynamoDB に永続化するため多ターン会話が成立する。

#### 3.2.5 会話エンジンの実装方式

既存 Worker と同様に、**Claude Code CLI（`claude` コマンド）を subprocess で呼び出す。**
`CLAUDE_CODE_OAUTH_TOKEN` は ECS secrets injection で渡す（Secrets Manager 経由）。

```python
import subprocess

def call_claude(prompt: str, model: str = "claude-sonnet-4-6") -> str:
    result = subprocess.run(
        ["claude", "--print", "--dangerously-skip-permissions", "--model", model],
        input=prompt,
        capture_output=True,
        text=True,
        timeout=300,
    )
    if result.returncode != 0:
        raise RuntimeError(f"claude CLI failed: {result.stderr}")
    return result.stdout.strip()
```

多ターン会話は DynamoDB から読み込んだ会話履歴をプロンプトに組み込む形で実現する。

```
[System]
あなたは開発要件ヒアリングの専門家です。...

[会話履歴]
User: 〇〇を作って
Assistant: どんな機能が必要ですか？
User: XXX機能が必要

[新しいメッセージ]
User: {content}

次の返答を JSON で出力してください。
{"action": "question"|"confirm", "message": "...", "issue": {...}}
```

#### 3.2.6 ECS タスク定義

| 項目 | 値 |
|---|---|
| イメージ | `agent-intake/dev-agent`（新規） |
| 起動トリガー | `dev-agent-queue` SQS |
| 環境変数 | `CLAUDE_CODE_OAUTH_TOKEN`（Secrets Manager）、`DISCORD_BOT_TOKEN`、`GITHUB_TOKEN`、`STATE_TABLE_NAME` |
| タイムアウト | 最大15分（ECS タスク） |

---

## 4. DynamoDB スキーマ変更

### 4.1 追加：セッションレコード

| 属性 | 型 | 説明 |
|---|---|---|
| `job_id` | String（PK） | `SESSION#{user_id}` |
| `agent` | String | `"dev-agent"` / 将来: `"sales-agent"` 等 |
| `queue_url` | String | 転送先 SQS キュー URL |
| `channel_id` | String | Discord チャンネル ID |
| `started_at` | String | ISO 8601 |
| `ttl` | Number | Unix 時間（24時間後、DynamoDB TTL で自動削除） |

### 4.2 追加：Dev-Agent 会話履歴レコード

| 属性 | 型 | 説明 |
|---|---|---|
| `job_id` | String（PK） | `THREAD#dev#{user_id}` |
| `messages` | List | 会話履歴（最大30ターン） |
| `updated_at` | String | ISO 8601 |
| `ttl` | Number | Unix 時間（7日後） |

### 4.3 変更：LangGraph 会話履歴

| 現状 | 変更後 |
|---|---|
| `THREAD#{user_id}`（LangGraph が管理） | 不要（Dev-Agent に移管）→ 削除 |

---

## 5. Discord Bot token への切り替え（必須対応）

### 5.1 問題

現状の `send_followup` は Discord の interaction token を使用している。
interaction token は **15分で失効** するため、多ターン会話の2ターン目以降でメッセージ送信が失敗する。

### 5.2 対応

Dev-Agent では interaction token ではなく、**Bot token で直接チャンネルに送信する。**

```
# 現状（interaction token・15分制限）
PATCH /webhooks/{application_id}/{token}/messages/@original

# 変更後（Bot token・制限なし）
POST /channels/{channel_id}/messages
Authorization: Bot {DISCORD_BOT_TOKEN}
```

`channel_id` は handler Lambda が SQS メッセージに含めており（既存）、そのまま利用できる。

---

## 6. 将来拡張

### 6.1 新規エージェント追加手順

営業・事務・コンサル等を追加するとき：

1. `src/<agent-name>/` を agent-intake に追加（Dev-Agent と同じ構造）
2. 専用 SQS キューを作成（Terraform）
3. LangGraph の意図分類ラベルに追加（1行）
4. LangGraph のルーティングテーブルに新キュー URL を追加（1行）

**LangGraph の変更は最小限。各エージェントが独立して開発・デプロイできる。**

### 6.2 エージェント間引き継ぎ（将来）

例：営業エージェントが「開発依頼に発展した」と判断した場合、DynamoDB のセッションを書き換えて Dev-Agent に引き継げる。

```python
# 営業エージェントがセッションを書き換え
put_session(user_id, agent="dev-agent", queue_url=DEV_AGENT_QUEUE_URL)
# 会話サマリーを Dev-Agent に引き継ぎ
forward_to_dev_agent(summary=conversation_summary)
```

---

## 7. 移行フェーズ

### Phase 1: Dev-Agent 新規実装

| タスク | 内容 | 規模 |
|---|---|---|
| D-1 | `src/dev-agent/` 基本構造・`entrypoint.py` 実装 | S |
| D-2 | `agent.py`：Claude ベース会話エンジン | M |
| D-3 | `tools/github.py`：構造化 Issue 起票ツール | S |
| D-4 | `tools/discord.py`：Bot token での Discord 送信 | S |
| D-5 | DynamoDB 会話履歴管理の実装 | S |
| D-6 | ECS タスク定義・`dev-agent-queue` 作成（Terraform） | M |

### Phase 2: LangGraph 縮小

| タスク | 内容 | 規模 |
|---|---|---|
| L-1 | セッション管理（`SESSION#{user_id}` の読み書き）実装 | S |
| L-2 | 意図分類のみに縮小（会話ループ・ツール削除） | M |
| L-3 | `dev-agent-queue` への転送処理実装 | S |

### Phase 3: 統合テスト・切り替え

| タスク | 内容 | 規模 |
|---|---|---|
| T-1 | Dev-Agent 単体テスト（Discord 模擬） | M |
| T-2 | LangGraph → Dev-Agent 転送テスト | S |
| T-3 | 多ターン会話テスト | M |
| T-4 | Issue 起票 → Worker パイプライン E2E テスト | M |
| T-5 | 旧 LangGraph フロー（GLM 会話部分）の削除 | S |

---

## 8. 未解決事項

| 事項 | 内容 |
|---|---|
| セッション TTL | 24時間で良いか。長時間返答がない場合の扱い（タイムアウト通知など） |
| 意図分類の精度 | GLM で「開発 vs 自己解決」を安定して分類できるか。Few-shot プロンプトの整備が必要 |
| `web_search` ツール | self_resolve 時に LangGraph が使うか、廃止するか |
| セッションキャンセル | ユーザーが会話を途中でやめたい場合の手段（`/cancel` コマンド等） |
