# v3 設計書：自律事業運営プラットフォーム

## アーキテクチャ概要

```
[Telegram]
    ↕ チャット
[OpenClaw Gateway]  ─────────────────────────────────────────────────┐
│  事業オペレーター                                                     │
│  - 仕事の発見・評価                                                   │
│  - 要件定義・発注                                                     │
│  - 成果物の検証                                                       │
│  - ユーザーへの報告                                                   │
└────────────────┬──────────────────────────────────────────────────┘
                 │ HTTP MCP (run_task / get_status)
                 ↓
[dev team MCP server]  ← openclaw-dev リポジトリで管理
│  - タスク受付・管理
│  - Claude Code への委譲
│  - デプロイ
└────────────────┬──────────────────────────────────────────────────
                 │ webhook (完了通知)
                 ↓
[OpenClaw /hooks/wake]  → OpenClaw が検証ループへ
```

---

## コンポーネント詳細

### 1. OpenClaw Gateway（事業オペレーター）

**実行基盤**: Cloudflare MoltWorker（既存）  
**設定**: `start-openclaw.sh` で dev team MCP server を登録

#### 責務

| 責務 | 実装方法 |
|------|---------|
| ユーザー指示の受付 | Telegram チャネル（既存） |
| タスクの要件化 | SOUL.md の指示に従い Claude が生成 |
| dev team への発注 | MCP tool `run_task` を呼ぶ |
| 進捗・完了の把握 | webhook wake → 自動ハートビート |
| 成果物の検証 | ブラウザツール / sandbox.exec |
| ユーザーへの報告 | Telegram へ直接 deliver |
| 再発注判断 | 検証失敗時に `run_task` を再呼び出し |

#### MCP server 登録（start-openclaw.sh に追加）

```js
config.plugins.entries.acpx.config.mcpServers['dev-team'] = {
    url: process.env.DEV_TEAM_MCP_URL,       // https://dev-team.xxx.workers.dev/mcp
    transport: 'streamable-http',
    headers: {
        Authorization: `Bearer ${process.env.DEV_TEAM_MCP_TOKEN}`,
    },
};
```

#### SOUL.md への追記

```markdown
# Task Delegation

開発・実装を伴うタスクは必ず dev team に委譲する。
自分でコードを書かない。

委譲の手順:
1. ユーザーの意図から実装仕様（spec）を日本語で整理する
2. run_task(spec) を呼んで dev team に発注する
3. 「着手しました。完了したら通知します」とユーザーに返す
4. 完了通知を受け取ったら成果物を確認し、問題があれば再発注する
5. 問題なければユーザーに成果物 URL と概要を報告する
```

---

### 2. dev team MCP server

**実行基盤**: Cloudflare Workers  
**リポジトリ**: `haruvv/openclaw-dev`  
**エンドポイント**: `https://dev-team.{account}.workers.dev/mcp`

#### 公開ツール

```typescript
// run_task: タスクの発注
{
  name: 'run_task',
  description: '実装タスクを受け付けて Claude Code に委譲する',
  inputSchema: {
    spec: string,       // 要件テキスト（自然言語）
    callback_url: string, // 完了時に通知する OpenClaw webhook URL
    callback_token: string, // webhook 認証トークン
    task_label?: string,  // 任意のラベル（ログ・追跡用）
  },
  returns: {
    task_id: string,    // 追跡用 ID
    status: 'accepted' | 'rejected',
    message?: string,
  }
}

// get_task_status: 進捗確認
{
  name: 'get_task_status',
  description: 'タスクの現在の状態を返す',
  inputSchema: {
    task_id: string,
  },
  returns: {
    task_id: string,
    status: 'pending' | 'running' | 'done' | 'failed',
    summary?: string,   // 完了時の概要
    artifact_url?: string, // 成果物 URL（デプロイ済みの場合）
    error?: string,     // 失敗時のエラー内容
  }
}

// cancel_task: キャンセル
{
  name: 'cancel_task',
  inputSchema: { task_id: string },
  returns: { cancelled: boolean }
}
```

#### 内部フロー

```
run_task(spec) 受信
    ↓
タスクを KV に保存（status: pending）
    ↓
GitHub Actions をトリガー（repository_dispatch）
    ↓
Actions: Claude Code が spec をもとに実装
    ↓
Actions: デプロイ（Cloudflare Workers/Pages）
    ↓
Actions: KV の status を更新（done / failed）
    ↓
callback_url に POST（完了通知）
```

**タスク状態の保存**: Cloudflare KV（`DEV_TEAM_TASKS`）

```typescript
interface Task {
  id: string;
  spec: string;
  status: 'pending' | 'running' | 'done' | 'failed';
  callback_url: string;
  callback_token: string;
  created_at: string;
  updated_at: string;
  summary?: string;
  artifact_url?: string;
  error?: string;
}
```

---

### 3. 完了通知フロー

dev team → OpenClaw の非同期通知。

#### dev team 側（GitHub Actions の最終ステップ）

```bash
curl -X POST "${CALLBACK_URL}/hooks/wake" \
  -H "Authorization: Bearer ${CALLBACK_TOKEN}" \
  -H "Content-Type: application/json" \
  -d "{
    \"text\": \"タスク完了: ${TASK_ID}\n成果物: ${ARTIFACT_URL}\n概要: ${SUMMARY}\",
    \"mode\": \"now\"
  }"
```

#### OpenClaw 側の動き

`/hooks/wake` を受信すると OpenClaw がハートビートで起動し、テキスト内容をもとにエージェントが後続処理（検証・ユーザー通知）を行う。

---

### 4. 検証ループ

成果物が存在する場合、OpenClaw が自律的に検証を行う。

```
完了通知受信
    ↓
artifact_url にアクセス（browser tool）
    ↓
動作を確認・spec との照合
    ↓
OK → ユーザーに報告（Telegram deliver）
NG → run_task(追加要求) を再発注 → ループ
```

再発注の上限は SOUL.md で指定する（例：最大3回まで自動リトライ）。

---

## データフロー図

### UC-3: ユーザー直接指示

```
ユーザー「〇〇を作って」（Telegram）
    │
    ▼
OpenClaw: spec を生成
    │
    ▼ MCP: run_task(spec, callback_url, callback_token)
dev team MCP server
    │ task_id を返す
    ▼
OpenClaw → ユーザー「着手しました (task: abc123)」
    │
    ▼（非同期）
GitHub Actions: Claude Code が実装・デプロイ
    │
    ▼ POST /hooks/wake
OpenClaw: 完了通知受信
    │
    ▼ browser tool で検証
    │
    ├─ OK ─→ ユーザー「完成: https://... 」
    └─ NG ─→ run_task(追加要求) → ループ
```

---

## API 設計

### dev team MCP server エンドポイント

| メソッド | パス | 説明 |
|---------|------|------|
| POST | `/mcp` | MCP JSON-RPC エンドポイント（streamable-http） |
| GET | `/health` | ヘルスチェック |

#### 認証

`Authorization: Bearer <DEV_TEAM_MCP_TOKEN>` ヘッダーで検証。  
トークンは Cloudflare Workers の Secret として管理。

#### MCP JSON-RPC 例

```json
// リクエスト
{
  "jsonrpc": "2.0",
  "method": "tools/call",
  "params": {
    "name": "run_task",
    "arguments": {
      "spec": "ランディングページを作成する。\n- タイトル: 〇〇サービス\n- CTA ボタン: 「今すぐ始める」\n- Cloudflare Pages にデプロイ",
      "callback_url": "https://openclaw-gateway.haruki-ito0044.workers.dev",
      "callback_token": "...",
      "task_label": "LP作成-20260410"
    }
  },
  "id": 1
}

// レスポンス
{
  "jsonrpc": "2.0",
  "result": {
    "content": [{ "type": "text", "text": "{\"task_id\":\"abc123\",\"status\":\"accepted\"}" }]
  },
  "id": 1
}
```

---

## 環境変数・シークレット管理

### OpenClaw Gateway（Cloudflare Workers secrets）

| 変数名 | 内容 |
|--------|------|
| `DEV_TEAM_MCP_URL` | dev team MCP server の URL |
| `DEV_TEAM_MCP_TOKEN` | dev team 認証トークン |

### dev team MCP server（Cloudflare Workers secrets）

| 変数名 | 内容 |
|--------|------|
| `DEV_TEAM_MCP_TOKEN` | MCP 認証トークン（受信側） |
| `GITHUB_TOKEN` | Actions トリガー用 PAT |
| `CLOUDFLARE_API_TOKEN` | デプロイ先の API トークン |
| `ANTHROPIC_API_KEY` | Claude Code 用 |

---

## 設計上の判断

### なぜ HTTP MCP server か

- OpenClaw の mcpServers は HTTP streamable-http transport に対応済み
- 別リポジトリで独立して管理・デプロイできる
- 将来的に dev 以外のエージェント（research、consult）も同じパターンで追加できる

### なぜ webhook（非同期）か

- Claude Code の実行は数分〜数十分かかる
- OpenClaw のリクエストをブロックしたくない
- `/hooks/wake` で OpenClaw を起動する仕組みがすでに存在する

### なぜ GitHub Actions か（dev team の実装選択）

- Claude Code の公式 Actions が提供されている
- CI（lint・test）と同一環境で動かせる
- PR・コミット履歴が自動的に残る

### エージェント追加の拡張方法

新しい専門エージェント（research、consult）を追加する場合：

1. 新しい MCP server リポジトリを作成（または同一 Workers に tools を追加）
2. `start-openclaw.sh` の `mcpServers` に URL を追加
3. SOUL.md に委譲ルールを追記

OpenClaw Gateway 本体は変更しない。

---

## v3 実装順序

```
Phase 1 ─── dev team MCP server の実装
             └── Cloudflare Workers + KV
             └── run_task / get_task_status / cancel_task
             └── GitHub Actions トリガー

Phase 1 ─── GitHub Actions の実装（openclaw-dev）
             └── Claude Code で spec を実装
             └── Cloudflare Pages/Workers にデプロイ
             └── callback_url に完了通知

Phase 1 ─── OpenClaw への接続
             └── start-openclaw.sh に mcpServers 追加
             └── SOUL.md に委譲ルール追記
             └── E2E テスト（Telegram → 実装 → 通知）

Phase 2 ─── 検証ループ
             └── ブラウザツールで成果物確認
             └── 自動再発注フロー

Phase 3 ─── 自律案件取得
             └── クラウドワークス API 連携
             └── 案件評価・承認フロー
```
