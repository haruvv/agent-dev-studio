# v2 実装計画

## 全体目標

OpenClaw をチャットに常駐させ、特化エージェントにタスクを委譲するプラットフォームを構築する。
v2 では開発エージェントを最初の特化エージェントとして実装する。

---

## リポジトリ

| リポジトリ | 役割 | 状態 |
|-----------|------|------|
| `haruvv/openclaw-gateway` | OpenClaw 設定・カスタムツール | 初期構成済み |
| `haruvv/openclaw-dev` | 開発エージェント（GitHub Actions + Claude Code） | 未着手 |

---

## Phase 1: openclaw-gateway のセットアップ

### 1-1. Cloudflare へのデプロイ

- [ ] Cloudflare アカウントで Workers Paid プランを有効化
- [ ] R2 バケット `openclaw-gateway-data` を作成
- [ ] `wrangler secret put` で必要な Secrets を設定
  - `ZAI_API_KEY`
  - `TELEGRAM_BOT_TOKEN`
  - `TELEGRAM_CHAT_ID`（dev-intake routing 用）
  - `GITHUB_PERSONAL_ACCESS_TOKEN`
  - `MOLTBOT_GATEWAY_TOKEN`
- [ ] `npm run deploy` でデプロイ
- [ ] Telegram Bot を作成・動作確認（`@BotFather` で作成）

### 1-2. dev-intake エージェントの調整

- [x] dev-intake のエージェント定義（`start-openclaw.sh` の config patch で実装済み）
  - `agents.list` に `dev-intake` エージェントを追加
  - `DISCORD_GUILD_ID` による Discord メッセージの routing
  - GitHub MCP サーバー（`@modelcontextprotocol/server-github`）の設定
- [ ] GitHub MCP サーバーの動作確認（Issue 起票）

---

## Phase 2: openclaw-dev のセットアップ

### 2-1. リポジトリ構成

- [x] `haruvv/openclaw-dev` に GitHub Actions ワークフローを作成
  - トリガー：Issue への `ai-dev` ラベル付与
  - Claude Code（`anthropics/claude-code-action@v1`）で実装・PR 作成
  - CI（lint・型検査・テスト）
  - Discord webhook で結果通知

### 2-2. Claude Code の設定

- [ ] `ANTHROPIC_API_KEY` Secret を `haruvv/openclaw-dev` に設定
- [ ] 実装対象リポジトリへのアクセス権設定（PAT or GitHub App）

### 2-3. Discord 通知

- [x] GitHub Actions から Discord webhook でイベント通知（実装済み）
  - 実装完了時（success）
  - 実装失敗時（failure）
  - CI 失敗時
- [ ] `DISCORD_WEBHOOK_URL` Secret を `haruvv/openclaw-dev` に設定

---

## Phase 3: エンドツーエンド動作確認

- [ ] Discord でユーザーが開発依頼を送る
- [ ] OpenClaw が要件をヒアリングし、承認後に GitHub Issue を起票する
- [ ] `openclaw-dev` の GitHub Actions が起動し、Claude Code が実装して PR を作成する
- [ ] CI が実行され、結果が Discord に通知される
- [ ] ユーザーが PR をレビューして統合する

---

## 今後の拡張（Phase 4 以降）

- コンサルエージェント（`openclaw-consult`）の追加
- 顧客調整エージェント（`openclaw-customer`）の追加
- OpenClaw の routing ルール拡充
