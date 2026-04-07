# 単一リポジトリ設計

## 背景・動機

マルチリポジトリ構成（`/create` で app repo を都度生成）には以下の根本的問題がある。

- Worker が app repo ごとに書き込む docs/ が孤立し、パイプライン自体の知識蓄積ループが成立しない
- dev-agents のテンプレートが静的なまま固まり、運用知見が反映されない
- 「リポジトリを記録システムとして運用する」というハーネスエンジニアリングの前提が複数リポジトリに分散して崩れる

→ **1つのリポジトリに要件・設計・実装・ナレッジを集約し、知識蓄積ループを成立させる。**

---

## 単一リポジトリの構造

`dev-agents` をパイプラインの実行現場兼ナレッジベースとして位置づける。

```
dev-agents/
├── AGENTS.md                        ← Worker 向け地図（パイプライン全体のエントリー）
├── docs/
│   ├── requirements-template.md    ← 要件定義フォーマット定義
│   ├── design-template.md          ← 設計書フォーマット定義
│   ├── constraints.md              ← アーキテクチャ制約（全 Issue 共通）
│   ├── pipeline-learnings.md       ← パイプライン運用で得た知見の蓄積
│   ├── <issue-number>-requirements.md  ← requirements Worker が生成
│   └── <issue-number>-design.md        ← design Worker が生成
├── src/                             ← 実装コード（implement Worker が書く）
├── .github/
│   └── workflows/                   ← トリガー workflow・CI（自己適用）
└── labels.yml                       ← ラベル定義（パイプライン仕様の記録）
```

---

## 旧設計との対比

| 項目 | 旧（マルチ repo） | 新（単一 repo） |
|---|---|---|
| 開発対象 | `/create` で都度生成した app repo | `dev-agents` 自身 |
| docs/ の場所 | 各 app repo に孤立 | `dev-agents/docs/` に集約・蓄積 |
| テンプレート配布 | `/create` 時にコピー | 不要（単一 repo に最初からある） |
| 知識ループ | 成立しない | dev-agents/docs/ が育ち続ける |
| AGENTS.md | app repo ごとにコピー | dev-agents に1つ存在し更新される |

---

## パイプラインフロー（変更後）

```
Human
  │ Discord /issue <description>
  ▼
agent-intake-processor
  │ Gemma で要求解釈
  │ 曖昧なら Discord で確認
  ▼
dev-agents に GitHub Issue 起票（要件定義作成ラベル）
  ▼
dev-agents の GitHub Actions（ラベルイベント検知 → SQS）
  ▼
ECS Fargate Worker
  ├─ requirements: dev-agents/docs/<n>-requirements.md を生成・コミット
  ├─ design:       dev-agents/docs/<n>-design.md を生成・コミット
  ├─ task-split:   dev-agents に子 Issue 起票
  ├─ implement:    dev-agents/src/ 配下に実装・PR 作成
  └─ repair:       CI 失敗を修正・再プッシュ
  ▼
Orchestrator（CI 結果 Webhook）
  ▼
人間レビュー → マージ
```

---

## ナレッジ蓄積の設計

単一 repo により以下のループが成立する。

```
Issue 起票
  → requirements/design Worker が docs/ に記録
  → implement Worker が docs/ を参照して実装
  → CI・repair の知見を pipeline-learnings.md に反映（人間またはGC Worker）
  → AGENTS.md・constraints.md・テンプレートが更新される
  → 次の Issue に最新の知見が適用される
```

`docs/pipeline-learnings.md` は Garbage Collection の観点から定期的に以下を記録する場所として機能する。

- 有効だった要件整理パターン
- task-split が失敗した設計書の特徴
- repair ループで頻出するエラーパターン
- アーキテクチャ制約の追加・変更履歴

---

## agent-intake への影響

### 廃止・変更

| 機能 | 変更内容 |
|---|---|
| `/create` コマンド | **廃止**。新規 app repo を生成する必要がなくなる |
| `_copy_files()` | **廃止**。テンプレートのコピーが不要になる |
| `TEMPLATE_FILES` | **廃止** |
| `_create_repo()` | **廃止** |
| `GITHUB_ORG` / `DEV_AGENTS_REPO` 環境変数 | **廃止** |
| リポジトリレジストリ（State DB の REPO# アイテム） | **廃止**。対象 repo が固定になるため不要 |

### 変更

| 機能 | 変更内容 |
|---|---|
| `/issue` コマンド | repo 選択 UI を廃止。固定の `dev-agents` repo に起票する |
| 環境変数 `TARGET_REPO` | `haruvv/dev-agents` 固定値として設定 |
| Webhook 登録 | `/create` 時ではなく初回デプロイ時に `dev-agents` に対して一度登録 |

---

## 弱点と対策

| 弱点 | 対策 |
|---|---|
| docs/ に Issue が積み重なりコンテキスト汚染 | AGENTS.md の参照ガイドを整備。Worker は issue 番号を明示して参照する |
| 複数 Worker が同時に main へコミット | ジョブ種別ごとにロックアイテムで二重起動防止（現設計を継承） |
| 実装コードと docs が同居して見通しが悪い | `src/` 配下を明確に分離。AGENTS.md にディレクトリ構造を記載 |
| 1つの CI が全変更に適用される | ジョブ種別に応じた CI パスフィルタリングで対応可能 |

---

## 移行スコープ

### フェーズ A（agent-intake 側）

- [ ] `/create` コマンドの廃止
- [ ] `/issue` から repo 選択を削除し `TARGET_REPO` 固定化
- [ ] processor.py の `TEMPLATE_FILES` / `_copy_files` / `_create_repo` 削除
- [ ] State DB のリポジトリレジストリ関連ロジック削除

### フェーズ B（dev-agents 側）

- [ ] `docs/constraints.md` 新規作成（アーキテクチャ制約の初期定義）
- [ ] `docs/pipeline-learnings.md` 新規作成（ナレッジ蓄積の起点）
- [ ] `templates/` ディレクトリ廃止（コピー配布が不要になるため）
- [ ] `scripts/bootstrap.sh` を `/issue` 登録手順に置き換えるか廃止
- [ ] Webhook を `dev-agents` に登録
- [ ] GitHub Actions workflow・ラベルを `dev-agents` 本体に適用
