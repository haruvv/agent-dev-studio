# プロダクトA 設計書

## 概要

チャット（Discord）を入口とする中央エージェント。ユーザーの意図を理解し、自分で処理するか、専門エージェントチームに委譲するかを判断する。

**技術スタック**: LangGraph（Python）/ LangGraph Self-Hosted Lite（既存 ECS に同居）

---

## 背景・動機

### 旧構成の制約

- Discord `/issue` スラッシュコマンドのみが入口
- 固定パイプライン（requirements → design → implement）しか起動できない
- 雑談・調査・開発以外のタスクに対応不可

### 新構成で解決すること

- 自然言語チャットで何でも受け付ける
- 意図に応じて動的にルーティング（直接処理 or エージェントチームに委譲）
- 会話コンテキストを保持したまま長時間タスクを処理できる

---

## アーキテクチャ

```
Discord メッセージ
  ↓
handler Lambda（3秒以内に ACK 返却）
  ↓ POST /runs（thread_id = Discord channel_id）
LangGraph Platform（プロダクトA）
  ├─ 雑談・質問 → Claude が直接返答 → Discord API
  ├─ 調査タスク → web_search ツール → Discord API
  └─ 開発タスク → create_issue ツール
        ↓ dev-agents に GitHub Issue 起票
        ↓ interrupt() で待機
              ↓（非同期）
        dev-agents pipeline（既存のまま）
        CI 成功 → Orchestrator Lambda が resume API 呼び出し
              ↓
        エージェント再開 → Discord に完了 + PR URL 通知
```

---

## コンポーネント詳細

### handler Lambda（変更あり）

- 役割：Discord Interactions の 3 秒 ACK のみ
- **変更点**：送り先を processor Lambda → LangGraph Platform API に変更
- `thread_id` に Discord の `channel_id`（または `user_id`）を使用し、会話を継続

### processor Lambda（廃止）

- 役割を LangGraph エージェントが引き継ぐため廃止
- Gemma による明確さチェックもエージェントの判断に統合

### プロダクトA（LangGraph Agent）

LangGraph の Supervisor パターンで実装。

**ツール一覧**

| ツール | 説明 |
|---|---|
| `web_search` | ウェブ検索（調査タスク） |
| `create_dev_issue` | dev-agents に GitHub Issue を起票し、パイプラインを起動 |
| `reply_to_discord` | Discord にメッセージを送信 |
| （将来）`delegate_to_*` | 他エージェントチームへの委譲 |

**状態管理**

- LangGraph Platform の thread ベースで会話履歴を保持
- `thread_id = Discord channel_id` でチャンネルごとにコンテキスト分離

**待機・再開フロー（開発タスク）**

```python
@tool
def create_dev_issue(task: str) -> str:
    """開発タスクを dev-agents パイプラインに投げる"""
    issue_url = _create_github_issue(task)
    # PR 完了まで待機（Orchestrator Lambda が resume する）
    pr_url = interrupt({"waiting_for": "pr", "issue_url": issue_url})
    return pr_url
```

### Orchestrator Lambda（変更あり）

- 既存の役割（CI 結果処理・repair ジョブ投入）はそのまま継続
- **追加**：CI 成功 + `ready-for-human-review` 付与時に LangGraph Platform API を呼んで該当スレッドを resume

```python
# CI 成功時の追加処理
def _handle_ci_success(repo, job_id, pr_number):
    ...（既存処理）
    # LangGraph スレッドを再開
    _resume_langgraph_thread(job_record, pr_url)
```

### dev-agents pipeline（変更なし）

既存の requirements → design → task-split → implement → repair フローをそのまま使用。

---

## LangGraph ホスティング

### プラン選定：Self-Hosted Lite（無料）

| プラン | 料金 | ノード上限 | ホスティング |
|---|---|---|---|
| **Self-Hosted Lite** | **無料** | 100万ノード/年 | 自前インフラ（Docker/ECS） |
| Developer（クラウド） | 無料 | 10万ノード/月 | LangGraph Platform |
| Plus（クラウド） | $0.001/ノード + LangSmith $39/月 | 無制限 | LangGraph Platform |

### 選定理由

1メッセージあたりのノード数（LLM呼び出し + ツール呼び出し + 返答）は約 5〜15 ノード。

- 1日 50 メッセージ想定 → 約 500 ノード/日 → **約 18 万ノード/年**
- Self-Hosted Lite の上限 100 万ノード/年 に対して十分な余裕がある
- Plus の $39/月（LangSmith 必須）はこのスケールでは割に合わない
- 既存の ECS インフラに Docker コンテナとして同居させることで追加インフラ不要

### デプロイ構成

LangGraph Server を ECS 上の独立コンテナとして稼働させる。

```
agent-intake/
  src/
    product-a/               ← プロダクトA のコードを配置（予定）
      Dockerfile             ← LangGraph Server コンテナ
      langgraph.json         ← LangGraph 設定
      agent.py               ← エージェント本体
      tools/
        web_search.py
        dev_agents.py
        discord.py
      requirements.txt
```

---

## 既存コンポーネントへの影響

| コンポーネント | 変更内容 |
|---|---|
| handler Lambda | 送り先を LangGraph Platform API に変更（コード小変更） |
| processor Lambda | **廃止** |
| Orchestrator Lambda | CI 成功時に LangGraph resume API 呼び出しを追加 |
| dev-agents pipeline | **変更なし** |
| ECS Worker（dev-agents 用） | **変更なし** |
| Terraform | processor Lambda 削除、LangGraph Platform の API キー管理を追加 |

---

## 未解決事項・今後の検討

| 項目 | 内容 |
|---|---|
| LangGraph Platform の料金 | Self-Hosted Lite（無料）を選定。100万ノード/年まで無償。現在の想定使用量（約18万ノード/年）は余裕で収まる |
| Discord → LangGraph の認証 | VPC 内 ALB 経由で通信。外部公開しないため認証不要 |
| resume 時のスレッド特定 | Issue 起票時に State DB の job レコードへ `thread_id` を追加保存。CI 成功時に Orchestrator が参照して resume |
| thread_id 設計 | `user_id`（Discord ユーザー ID）を使用。会話コンテキストはユーザーごとに分離。成果物（Issue/PR）は dev-agents リポジトリに集約されるため、複数ユーザーが同一プロジェクトを触ることは GitHub 上で可能 |
| 複数タスクの並行処理 | 同一ユーザーが複数の開発タスクを並行依頼した場合の挙動は今後検討 |
| エラー通知 | repair 上限超過時の Discord 通知（既存 Orchestrator の機能を活用） |
| 将来のエージェントチーム | 調査特化・レビュー特化など、dev-agents 以外のチームの追加方針 |
