---
name: ship
description: dev-agents の現在のブランチに対して Codex レビューを実行し、問題がなければ PR を作成する（実装 → Codex review → PR作成）
---

# Ship スキル（実装 → Codex レビュー → PR作成）

以下の手順を順番に実行してください。

## 1. 前提確認

`~/Develop/dev-agents` にて以下を確認する：
- 現在のブランチ名（`git -C ~/Develop/dev-agents branch --show-current`）
- main ブランチでないこと
- コミット済みの変更があること（`git -C ~/Develop/dev-agents log main..HEAD --oneline`）

main ブランチの場合や未コミット変更がある場合は中断してユーザーに伝える。

## 2. Codex レビュー実行

Bash ツールの `run_in_background: true` を使い、以下をバックグラウンド実行する：

```
cd ~/Develop/dev-agents && codex review --base main
```

完了通知を受け取ったら出力ファイルの末尾（`tail -c 6000`）を読んでレビュー結果を確認する。

## 3. レビュー結果の判定

Codex の出力を確認する：

**指摘なし / 軽微な指摘のみの場合：**
→ ステップ4（PR作成）へ進む

**P1 / P2 の指摘がある場合：**
→ 指摘内容をユーザーに示し、「修正してから再実行しますか？」と確認する。
→ ユーザーが修正を希望する場合は修正を実施し、コミット後にステップ2へ戻る。
→ ユーザーがそのまま進めたい場合はステップ4へ進む。

## 4. PR 作成

`~/Develop/dev-agents` で `gh pr create` を実行する。

- `--head` : 現在のブランチ名
- `--base` : `main`
- `--title` : `git log` から1件目のコミットメッセージを元に生成
- `--body` : 以下のテンプレートを使用
- `--label` : `ai-generated`, `ready-for-review`

```
## 変更概要

Closes haruvv/dev-agents#<issue番号>

## 実施内容

<変更内容の要約>

## テスト実施内容

<テスト・動作確認内容>

## 懸念点 / 未対応事項

<懸念点があれば記載、なければ「なし」>

## レビュー観点

<レビュアーに特に見てほしいポイント>
```

Issue番号はブランチ名（例: `impl/issue-6-xxx`）から推定する。

## 5. 完了報告

PR の URL をユーザーに伝える。
