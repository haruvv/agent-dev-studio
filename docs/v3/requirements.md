# v3 要件定義：自律事業運営プラットフォーム

## ビジョン

OpenClaw が事業の「脳みそ」として機能し、仕事の受注・要件定義・開発委託・検証・改善を自律的に回す。
ユーザーはチャットから方針を与えるだけで、実務の大部分を AI エージェント群が担う。

---

## ユースケース

### UC-1: 受注型（クラウドソーシング）

1. OpenClaw がクラウドワークスなどの案件を検索・評価する
2. 受注可能と判断した案件をユーザーに報告し、承認を得る
3. 要件を整理して開発チームに発注する
4. 成果物を受け取り、品質を検証する
5. 問題があれば追加要求を開発チームに投げて修正させる
6. 完成物を納品し、報酬を受領する

### UC-2: 自社プロダクト型（アフィリエイト・広告収益）

1. ユーザーまたは OpenClaw 自身がプロダクトアイデアを提案する
2. 要件を定義して開発チームに発注する
3. 開発チームが実装・デプロイする
4. OpenClaw が動作確認・収益モニタリングを行う
5. 改善が必要であればユーザーへ報告し、追加開発を依頼する

### UC-3: ユーザー直接指示

1. ユーザーが Telegram でタスクを指示する
2. OpenClaw が内容を整理し、開発チームに発注する
3. 進捗をユーザーに報告する
4. ユーザーが追加指示を出せば OpenClaw が開発チームへ反映する

---

## コンポーネント

### OpenClaw（事業オペレーター）

チャットに常駐する中枢エージェント。すべての判断と委譲を担う。

| 機能 | 内容 |
|------|------|
| 仕事の取得 | クラウドワークス API / Web スクレイピングで案件を探す |
| 要件定義 | 案件や指示から実装仕様を整理する |
| 開発チームへの発注 | MCP tool 経由で dev team に要件を渡す |
| 成果物の検証 | 実際に動かして確認する（ブラウザツール等） |
| ユーザーへの報告 | Telegram で進捗・完了・問題を通知する |
| 再発注 | 検証で問題が見つかれば追加要求を投げる |

### 開発チーム（dev team MCP server）

openclaw-dev リポジトリで管理する HTTP MCP サーバー。

| 機能 | 内容 |
|------|------|
| タスク受付 | `run_task(spec)` ツールで要件テキストを受け取る |
| 実装 | Claude Code が要件に基づいてコードを書く |
| デプロイ | 実装後に自動デプロイまたはデプロイ可能な状態にする |
| 結果報告 | 完了・失敗・成果物 URL を OpenClaw に返す |

### ユーザー（介入ポイント）

| タイミング | 内容 |
|-----------|------|
| 案件承認 | 受注前に OpenClaw から確認を受けて承認・却下 |
| 方針変更 | チャットで追加指示・方向修正 |
| 品質確認 | 最終成果物のレビュー |

---

## インターフェース定義

### OpenClaw → dev team

HTTP MCP プロトコルで通信する。

```
POST https://dev-team.example.workers.dev/mcp
```

**ツール一覧（dev team が公開）**

| ツール名 | 引数 | 説明 |
|---------|------|------|
| `run_task` | `spec: string` | 要件テキストを渡して実装を依頼する |
| `get_task_status` | `task_id: string` | タスクの進行状況を確認する |
| `cancel_task` | `task_id: string` | 実行中のタスクをキャンセルする |

### dev team → OpenClaw（完了通知）

OpenClaw の webhook エンドポイントに POST する。

```
POST https://openclaw-gateway.haruki-ito0044.workers.dev/hooks/wake
```

```json
{
  "text": "タスク完了: {task_id}\n成果物: {url}\n概要: {summary}"
}
```

---

## 実装フロー（開発タスクの例）

```
[ユーザー or OpenClaw] 「〇〇を作ってほしい」
    ↓
[OpenClaw] 要件を整理して spec テキストを生成
    ↓
[OpenClaw → dev team] run_task(spec) を呼ぶ
    ↓
[dev team] Claude Code が実装・デプロイ
    ↓
[dev team → OpenClaw] webhook で完了通知
    ↓
[OpenClaw] 成果物を検証（ブラウザ等）
    ↓ 問題なし
[OpenClaw → ユーザー] 「完成しました: {url}」
    ↓ 問題あり
[OpenClaw → dev team] run_task(追加要求) を再度呼ぶ
```

---

## v3 のスコープ（最初の実装対象）

v3 では UC-3（ユーザー直接指示 → 開発チームへ委譲）を最初のターゲットとする。
自律的な案件取得（UC-1, UC-2）はその後に追加する。

### Phase 1: 開発チーム連携

- [ ] dev team MCP server の実装（openclaw-dev リポジトリ）
  - `run_task` ツールの実装
  - Claude Code との接続
  - 完了時の webhook 通知
- [ ] OpenClaw への dev team MCP server 登録（start-openclaw.sh）
- [ ] E2E 動作確認：Telegram 指示 → 開発 → 完了通知

### Phase 2: 検証ループ

- [ ] OpenClaw が成果物を自動検証する仕組み（ブラウザツール活用）
- [ ] 問題検出時の自動再発注フロー
- [ ] ユーザーへの進捗通知テンプレート

### Phase 3: 自律案件取得

- [ ] クラウドワークス API 連携ツール
- [ ] 案件評価ロジック（OpenClaw の判断）
- [ ] ユーザー承認フロー

---

## 技術スタック

| レイヤ | 技術 |
|--------|------|
| 事業オペレーター | OpenClaw（Cloudflare MoltWorker） |
| チャット UI | Telegram |
| dev team サーバー | Cloudflare Workers（HTTP MCP server） |
| 実装エージェント | Claude Code（GitHub Actions or ACP） |
| 案件取得 | クラウドワークス API / Playwright |
| デプロイ先 | Cloudflare Workers / Pages（想定） |

---

## 未解決の問題

| 問題 | 備考 |
|------|------|
| dev team の非同期完了をどう受け取るか | webhook で OpenClaw を wake する方法は確認済み |
| Claude Code の実行環境 | GitHub Actions か Cloudflare Container か未定 |
| 案件の支払い・口座管理 | スコープ外（人間が対応） |
| デプロイ先の認証情報管理 | Cloudflare API token 等を dev team に渡す設計が必要 |
