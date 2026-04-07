# アーキテクチャ制約（dev-agents 単一リポジトリ）

本ファイルは `dev-agents` リポジトリ全体に適用される制約を定義する。
Worker は実装時にここに記載された制約を必ず遵守すること。

---

## ディレクトリ構造ルール

```
dev-agents/
├── AGENTS.md          ← 更新可。Worker 向け地図
├── docs/              ← 更新可。記録システム（Worker が書き込む場所）
├── src/               ← 更新可。実装コード（implement Worker が書く場所）
├── .github/workflows/ ← 更新可（CI/トリガー変更時のみ）
├── labels.yml         ← パイプライン仕様変更時のみ更新
└── scripts/           ← 手動運用スクリプト。Worker は原則触らない
```

## 禁止事項

- `docs/` 配下への実装コードの配置禁止
- `src/` 配下への設計書・要件定義書の配置禁止
- `labels.yml` の Worker による自動更新禁止（人間がキュレーションする）
- `AGENTS.md` の Worker による自動上書き禁止（人間がキュレーションする）

## docs/ 命名規則

| ファイル | 命名規則 | 生成主体 |
|---|---|---|
| 要件定義書 | `docs/<issue-number>-requirements.md` | requirements Worker |
| 設計書 | `docs/<issue-number>-design.md` | design Worker |
| 制約定義 | `docs/constraints.md`（本ファイル） | 人間 |
| 運用知見 | `docs/pipeline-learnings.md` | 人間（GC Worker が補助） |

## ブランチ・PR ルール

- 実装ブランチ：`impl/issue-<番号>-<slug>`
- docs の直接コミットは `main` ブランチへ（requirements / design / task-split Worker）
- コード実装は必ずブランチを切り PR を作成（implement Worker）
