# Claude Code 作業指針（パイプライン開発）

このリポジトリは `ai-agent-pipeline` の開発・設計を行うワークスペース。
Claude Code CLI としての私がこのドキュメントを読む。

## リポジトリ構成

| リポジトリ | 役割 |
|-----------|------|
| `haruvv/CodingAgentv2` | このワークスペース。設計・開発の作業場 |
| `haruvv/ai-agent-pipeline` | Reusable Workflow の定義・スクリプト管理 |
| `haruvv/ai-agent-sandbox` | pipeline の動作確認環境 |

## 作業方針

- 変更は基本的に `ai-agent-pipeline` に対して行う
- 変更後は `ai-agent-sandbox` で動作確認する
- pipeline の Reusable Workflow を変更したら sandbox でテストしてから完了とする

## pipeline の構成

```
ai-agent-pipeline/
├── .github/workflows/
│   ├── claude-impl-reusable.yml  # Claude実装ワークフローの本体
│   ├── ci-reusable.yml           # CIの本体
│   └── ci.yml                    # pipeline自身のPR用CI
└── scripts/
    └── setup-repo.sh             # 新リポジトリのセットアップ
```

## 新しいプロダクトリポジトリを作る手順

```bash
gh repo create <repo-name> --public --template haruvv/ai-agent-pipeline
bash /path/to/ai-agent-pipeline/scripts/setup-repo.sh haruvv/<repo-name>
gh secret set CLAUDE_CODE_OAUTH_TOKEN --repo haruvv/<repo-name>
```

## 注意事項

- `ai-agent-pipeline` の Reusable Workflow を変更する場合、後方互換性に注意する
- sandbox で確認せずに pipeline の main を変更しない
- 秘密情報（APIキー・トークン等）をコードやコミットに含めない
