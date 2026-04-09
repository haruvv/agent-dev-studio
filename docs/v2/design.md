# 自律エージェント基盤 v2 設計書

## ビジョン

チャットに常駐する AI が、あらゆるタスクを受け取り、適切な特化エージェントに委譲する。
開発・コンサル・顧客調整など、業務の種類を問わず同一のインターフェースから操作できるプラットフォームを構築する。

---

## 基本構造

```
[Discord / Slack / その他チャット]
            ↓
    [OpenClaw Gateway]          ← チャットに常駐する中枢
    (Cloudflare MoltWorker)
            │
            ├─→ 開発タスク      → 開発エージェント
            ├─→ コンサルタスク  → コンサルエージェント
            ├─→ 顧客調整タスク  → 調整エージェント
            └─→ 軽微なタスク    → OpenClaw 自身が処理
```

OpenClaw はチャットプラットフォームとエージェント群の**ゲートウェイ**として機能する。
タスクの受付・分類・委譲・返答をすべて担い、ユーザーは OpenClaw とのチャットだけを意識すればよい。

---

## コンポーネント

### OpenClaw Gateway

| 項目 | 内容 |
|------|------|
| 実行基盤 | Cloudflare MoltWorker（Cloudflare Sandbox） |
| 永続化 | Cloudflare R2（会話履歴・workspace） |
| 対応チャット | Discord、Slack など OpenClaw がサポートする任意のプラットフォーム |
| 役割 | タスク受付、intent 分類、エージェント委譲、返答 |

特化エージェントはそれぞれ OpenClaw の **multi-agent routing** または **カスタムツール** として登録する。
タスクの複雑度や性質に応じて、OpenClaw 自身が処理するか特化エージェントに委譲するかを判断する。

---

### 開発エージェント（v2 最初の特化エージェント）

ユーザーから開発依頼を受けた OpenClaw が委譲するエージェント。
要件ヒアリングから実装・PR 作成までを担う。

#### フロー

```
OpenClaw
    ↓ 開発タスクと判断
[dev-intake agent]（OpenClaw 管理下）
    ↓ 要件ヒアリング（複数ターン）
    ↓ ユーザー承認
    ↓ GitHub Issue 起票
[GitHub]
    ↓ Issue ラベル → GitHub Actions トリガー
[Claude Code]（GitHub Actions 上）
    ↓ 実装・コミット・PR 作成
[GitHub PR]
    ↓ CI
    ↓ Discord 通知（GitHub Actions → Discord webhook）
[ユーザー]
```

#### 各コンポーネント

**dev-intake agent**

OpenClaw が管理する Claude エージェント。
要件ヒアリングと Issue 起票に特化する。

| ツール | 機能 |
|--------|------|
| `create_github_issue` | 構造化 Issue を GitHub に起票する |

会話履歴は OpenClaw の workspace（R2）で管理される。
session_db のような独自実装は持たない。

**GitHub Actions + Claude Code**

Issue 起票をトリガーに実行される実装パイプライン。

| ステップ | 内容 |
|----------|------|
| トリガー | Issue への特定ラベル付与 |
| 実装 | Claude Code（Anthropic 公式 Actions）がコードを書く |
| PR 作成 | Claude Code が自動で PR を作成 |
| CI | lint・型検査・テスト |
| 通知 | Discord webhook で結果をチャットに返す |

---

## 状態管理

| 状態の種類 | 管理場所 |
|-----------|---------|
| 会話履歴・セッション | Cloudflare R2（OpenClaw workspace） |
| Issue・PR の進行状態 | GitHub（ラベル・コメント） |
| CI 結果 | GitHub Actions |
| Discord 通知 | GitHub Actions → Discord webhook |

専用のデータベースは持たない。GitHub と Cloudflare R2 で完結する。

---

## 拡張方針

新しい業務領域への対応は、**特化エージェントを追加**することで行う。
OpenClaw Gateway 自体は変えない。

```
# 将来追加される特化エージェントのイメージ
bindings:
  - agentId: "dev"        # 開発（v2 で実装）
  - agentId: "consult"    # コンサル（将来）
  - agentId: "customer"   # 顧客調整（将来）
```

各特化エージェントは独立して設計・実装・改善できる。
OpenClaw のルーティング設定を更新するだけで追加される。

---

## 技術スタック

| レイヤ | 技術 |
|--------|------|
| チャット常駐 | OpenClaw（Cloudflare MoltWorker） |
| 会話状態 | Cloudflare R2 |
| Issue・PR 管理 | GitHub |
| 実装エージェント | Claude Code（GitHub Actions） |
| CI | GitHub Actions |
| 通知 | Discord webhook（GitHub Actions から） |

---

## v2 実装スコープ

v2 では開発エージェントを最初の特化エージェントとして実装し、以下の動作を成立させる。

1. Discord でユーザーが開発依頼を送る
2. OpenClaw が要件をヒアリングし、承認後に GitHub Issue を起票する
3. GitHub Actions が起動し、Claude Code が実装して PR を作成する
4. CI が実行され、結果が Discord に通知される
5. ユーザーが PR をレビューして統合する
