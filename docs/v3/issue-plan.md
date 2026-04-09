# v3 Issue 計画

## リポジトリ構成

| リポジトリ | 役割 |
|-----------|------|
| `haruvv/openclaw-gateway` | OpenClaw 設定・MCP server 登録 |
| `haruvv/openclaw-dev` | dev team MCP server + GitHub Actions |

---

## Epic 一覧

| Epic | 内容 | Phase |
|------|------|-------|
| E1: dev team MCP server | Cloudflare Workers で MCP エンドポイントを実装 | 1 |
| E2: GitHub Actions 連携 | MCP server から Actions をトリガーして Claude Code を動かす | 1 |
| E3: OpenClaw 接続 | start-openclaw.sh に MCP server を登録・SOUL.md 更新 | 1 |
| E4: 検証ループ | 成果物の自動検証と再発注 | 2 |
| E5: 自律案件取得 | クラウドワークス連携 | 3 |

---

## Phase 1: 開発チーム連携の確立

### E1: dev team MCP server（openclaw-dev）

---

**Issue #1: Cloudflare Workers プロジェクトのセットアップ**

```
リポジトリ: haruvv/openclaw-dev
ラベル: setup

## 内容
openclaw-dev を Cloudflare Workers として初期化する。

## タスク
- [ ] `npm create cloudflare@latest` で Hono ベースの Workers を初期化
- [ ] wrangler.jsonc に KV バインディング（DEV_TEAM_TASKS）を設定
- [ ] ヘルスチェックエンドポイント GET /health を実装
- [ ] 初回デプロイ確認

## 完了条件
`curl https://dev-team.{account}.workers.dev/health` が 200 を返す
```

---

**Issue #2: MCP エンドポイントの実装**

```
リポジトリ: haruvv/openclaw-dev
ラベル: feature
依存: #1

## 内容
OpenClaw から呼ばれる MCP JSON-RPC エンドポイントを実装する。

## タスク
- [ ] POST /mcp エンドポイントを実装（streamable-http transport）
- [ ] Bearer トークン認証を実装（DEV_TEAM_MCP_TOKEN シークレット）
- [ ] MCP プロトコルハンドシェイク（initialize / tools/list）を実装

## 完了条件
MCP クライアントから initialize → tools/list が成功する
```

---

**Issue #3: run_task ツールの実装**

```
リポジトリ: haruvv/openclaw-dev
ラベル: feature
依存: #2

## 内容
タスク受付ツールを実装する。

## タスク
- [ ] run_task ツールを MCP に登録
  - 引数: spec, callback_url, callback_token, task_label?
  - 戻り値: task_id, status
- [ ] タスクを Cloudflare KV に保存（status: pending）
- [ ] GitHub Actions を repository_dispatch でトリガー
  - event_type: run-dev-task
  - client_payload: { task_id, spec, callback_url, callback_token }

## 完了条件
run_task を呼ぶと KV にタスクが作成され Actions がトリガーされる
```

---

**Issue #4: get_task_status / cancel_task ツールの実装**

```
リポジトリ: haruvv/openclaw-dev
ラベル: feature
依存: #2

## 内容
タスク状態確認とキャンセルツールを実装する。

## タスク
- [ ] get_task_status ツールを実装（KV からステータスを返す）
- [ ] cancel_task ツールを実装（KV のステータスを cancelled に更新）

## 完了条件
run_task 後に get_task_status で pending が返る
```

---

### E2: GitHub Actions 連携（openclaw-dev）

---

**Issue #5: Claude Code Actions ワークフローの実装**

```
リポジトリ: haruvv/openclaw-dev
ラベル: feature
依存: #3

## 内容
run-dev-task イベントを受け取り Claude Code で実装するワークフローを作成する。

## タスク
- [ ] .github/workflows/run-dev-task.yml を作成
  - トリガー: repository_dispatch (event_type: run-dev-task)
  - ステップ:
    1. client_payload から task_id / spec / callback_url / callback_token を取得
    2. KV の status を running に更新（Workers API 経由）
    3. Claude Code で spec をもとに実装
    4. Cloudflare Pages/Workers にデプロイ
    5. 完了通知（callback_url に POST）
    6. KV の status を done / failed に更新

## Secrets（haruvv/openclaw-dev に設定）
- ANTHROPIC_API_KEY
- CLOUDFLARE_API_TOKEN
- DEV_TEAM_WORKERS_API_URL  ← KV 更新用
- DEV_TEAM_WORKERS_API_TOKEN

## 完了条件
ワークフローが正常終了し callback_url に完了通知が届く
```

---

**Issue #6: 完了通知の実装**

```
リポジトリ: haruvv/openclaw-dev
ラベル: feature
依存: #5

## 内容
GitHub Actions の最終ステップで OpenClaw に完了を通知する。

## タスク
- [ ] 成功時: callback_url/hooks/wake に POST
  ```json
  {
    "text": "タスク完了: {task_id}\n成果物: {artifact_url}\n概要: {summary}",
    "mode": "now"
  }
  ```
- [ ] 失敗時: 同様に失敗内容を POST
- [ ] KV のステータスと artifact_url / error を更新

## 完了条件
Actions 完了後に OpenClaw のエージェントが応答する（Telegram に通知が来る）
```

---

### E3: OpenClaw 接続（openclaw-gateway）

---

**Issue #7: dev team MCP server の登録**

```
リポジトリ: haruvv/openclaw-gateway
ラベル: feature
依存: openclaw-dev #2

## 内容
start-openclaw.sh に dev team MCP server を登録する。

## タスク
- [ ] start-openclaw.sh の mcpServers に dev-team を追加
  ```js
  config.plugins.entries.acpx.config.mcpServers['dev-team'] = {
      url: process.env.DEV_TEAM_MCP_URL,
      transport: 'streamable-http',
      headers: { Authorization: `Bearer ${process.env.DEV_TEAM_MCP_TOKEN}` },
  };
  ```
- [ ] wrangler secret put DEV_TEAM_MCP_URL
- [ ] wrangler secret put DEV_TEAM_MCP_TOKEN
- [ ] デプロイ

## 完了条件
OpenClaw のツール一覧に run_task が表示される
```

---

**Issue #8: SOUL.md への委譲ルール追記**

```
リポジトリ: haruvv/openclaw-gateway
ラベル: config
依存: #7

## 内容
OpenClaw が自律的に dev team へ委譲するよう SOUL.md を更新する。

## タスク
- [ ] SOUL.md に Task Delegation セクションを追記
  - 開発・実装タスクの委譲手順
  - spec の書き方ガイドライン
  - 再発注の判断基準（最大 N 回）
- [ ] デプロイ・動作確認

## 完了条件
「〇〇を作って」と Telegram で送ると OpenClaw が run_task を呼ぶ
```

---

**Issue #9: E2E 動作確認**

```
リポジトリ: haruvv/openclaw-gateway
ラベル: testing
依存: #8, openclaw-dev #6

## 内容
Telegram から指示 → 実装 → 通知の全体フローを確認する。

## テストケース
- [ ] 「シンプルな HTML ページを作って Cloudflare Pages にデプロイして」
  → run_task が呼ばれる
  → Actions が起動する
  → 実装・デプロイが完了する
  → Telegram に完了通知が届く
  → URL にアクセスできる

## 完了条件
上記テストケースが一気通貫で動作する
```

---

## Phase 2: 検証ループ

**Issue #10: 成果物の自動検証**

```
リポジトリ: haruvv/openclaw-gateway
ラベル: feature
依存: Phase 1 完了

## 内容
完了通知受信後に OpenClaw が成果物を自動検証する。

## タスク
- [ ] SOUL.md に検証手順を追記（ブラウザツールで URL にアクセス・確認）
- [ ] 検証失敗時の再発注フローを定義（追加要求を spec に加えて run_task 再呼び出し）
- [ ] 最大リトライ回数の設定（デフォルト 3 回）
```

---

**Issue #11: ユーザーへの進捗通知の整備**

```
リポジトリ: haruvv/openclaw-gateway
ラベル: feature
依存: #10

## 内容
各ステージでの Telegram 通知メッセージを統一する。

## 通知タイミング
- [ ] 発注時：「着手しました。しばらくお待ちください。」
- [ ] 完了時：「完成しました。\n{url}\n\n{概要}」
- [ ] 検証NG再発注時：「改善点を見つけたので修正を依頼します。」
- [ ] 最終失敗時：「実装に失敗しました。\n{エラー内容}」
```

---

## Phase 3: 自律案件取得（将来）

**Issue #12: クラウドワークス API 連携ツール**

```
リポジトリ: haruvv/openclaw-dev (または openclaw-gateway)
ラベル: feature, future

## 内容
OpenClaw がクラウドワークスの案件を自律的に取得・評価できるようにする。

## タスク
- [ ] クラウドワークス API の調査
- [ ] 案件検索 MCP tool の実装
- [ ] 案件評価ロジック（予算・スキルセット・納期の適合判定）
- [ ] ユーザー承認フロー（Telegram でリストを提示）
```

---

## 着手順序

```
openclaw-dev:  #1 → #2 → #3 → #4（並行） → #5 → #6
                                                    ↓
openclaw-gateway:                          #7 → #8 → #9
                                                    ↓
                                           Phase 2: #10 → #11
```

Issue #3 と #4 は依存なし・並行作業可能。