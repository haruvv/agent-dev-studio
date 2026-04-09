# Discord → PR通知 フロー図

```mermaid
sequenceDiagram
    actor User as ユーザー
    participant Discord
    participant Handler as handler Lambda
    participant SQS1 as job_queue (SQS)
    participant S2E as sqs-to-ecs Lambda
    participant LG as LangGraph ECS
    participant DA as dev-agent ECS
    participant DDB as DynamoDB
    participant GH as GitHub
    participant Orch as orchestrator Lambda
    participant CI as GitHub Actions (CI)

    %% ─── Discord メッセージ受信 ───
    User->>Discord: /chat メッセージ送信
    Discord->>Handler: Interaction イベント (type:2)
    Handler-->>Discord: type:5 ACK（「考え中...」表示）
    Handler->>SQS1: job_type=langgraph メッセージ投入

    %% ─── LangGraph ルーティング ───
    SQS1->>S2E: トリガー
    S2E->>LG: ECS タスク起動 (JOB_PAYLOAD)
    LG->>LG: GLM-4.5-flash で intent 分類

    alt self_resolve（質問・雑談）
        LG->>Discord: edit_deferred_response（「考え中...」を回答で置換）
    else development（開発依頼）
        LG->>DDB: put SESSION#{user_id}
        LG->>SQS1: job_type=dev-agent メッセージ転送
        LG->>Discord: edit_deferred_response（受け付けメッセージ）

        %% ─── dev-agent 会話ループ ───
        SQS1->>S2E: トリガー
        S2E->>DA: ECS タスク起動 (JOB_PAYLOAD)

        loop 要件ヒアリング（複数ターン）
            DA->>DA: Claude Code CLI で会話
            DA->>Discord: edit_deferred_response / send_message（質問・確認）
            User->>Discord: 返答
            Discord->>Handler: 次のメッセージ
            Handler->>SQS1: job_type=dev-agent（SESSION により直送）
            SQS1->>S2E: トリガー
            S2E->>DA: ECS タスク起動
        end

        alt approve（Issue 起票承認）
            DA->>GH: Issue 作成（create_structured_issue）
            DA->>DDB: save DISCORD#{issue_number}（channel_id + user_id）
            DA->>DDB: clear SESSION#{user_id}
            DA->>Discord: send_message（「Issue #N を起票しました」）
        else cancel（キャンセル）
            DA->>DDB: clear SESSION#{user_id}
            DA->>Discord: send_message（キャンセルメッセージ）
        end
    end

    %% ─── パイプライン（Workers）───
    Note over GH: Issue に「要件定義作成」ラベル付与

    GH->>GH: requirements Worker 起動
    GH->>GH: Issue コメントに <!-- requirements-doc --> 投稿
    GH->>GH: 「詳細設計作成」ラベル付与

    GH->>GH: design Worker 起動
    GH->>GH: Issue コメントに <!-- design-doc --> 投稿
    GH->>GH: 「タスク分割」ラベル付与

    GH->>GH: task-split Worker 起動
    GH->>GH: 子 Issue 作成（parent_issue: N）
    Note over GH: 子1→ ready-for-impl<br/>子2以降→ queued-for-impl

    %% ─── 実装（直列） ───
    loop 子 Issue を順次処理
        GH->>GH: implement Worker 起動（ready-for-impl トリガー）
        GH->>GH: ブランチ impl/issue-N-slug 作成
        GH->>GH: PR 作成（ai-generated ラベル）

        %% ─── PR → Orchestrator ───
        GH->>Orch: webhook: pull_request opened
        Orch->>GH: _github_get_issue（子 Issue body 取得）
        Orch->>Orch: _resolve_discord_issue_number（parent_issue 解決）
        Orch->>DDB: get DISCORD#{parent_issue_number}
        Orch->>Discord: send_message（「🔧 PR が作成されました」）

        GH->>CI: CI ワークフロー実行
        CI-->>GH: conclusion: success / failure

        alt CI success
            GH->>Orch: webhook: workflow_run success
            Orch->>GH: PR に ready-for-human-review ラベル付与
            Orch->>Discord: send_message（「✅ 開発完了！レビューをお願いします」）
        else CI failure（repair ループ）
            GH->>Orch: webhook: workflow_run failure
            Orch->>SQS1: repair ジョブ投入（最大3回）
            GH->>GH: repair Worker → push → CI 再実行
        end

        User->>GH: PR マージ
        GH->>Orch: webhook: pull_request merged
        Orch->>GH: 次の queued-for-impl → ready-for-impl に切り替え
    end

    %% ─── 全完了通知 ───
    Orch->>DDB: get DISCORD#{parent_issue_number}
    Orch->>Discord: send_message（「🎉 Issue #N の全実装が完了しました！」）
```

## 主要コンポーネント対応表

| コンポーネント | 実体 |
|---|---|
| handler Lambda | `agent-intake` の Discord Interaction ハンドラー |
| LangGraph ECS | `src/langgraph/` — GLM-4.5-flash で intent 分類・ルーティング |
| dev-agent ECS | `src/dev-agent/` — Claude Code CLI で要件ヒアリング |
| sqs-to-ecs Lambda | `src/sqs_to_ecs/` — SQS → ECS RunTask |
| orchestrator Lambda | `src/orchestrator/` — GitHub Webhook 受信・Discord 通知 |
| Workers | `dev-agents` リポジトリの `.github/workflows/` + `agent-intake/src/worker/` |
| DynamoDB | STATE_TABLE — SESSION / THREAD / PENDING_ISSUE / DISCORD# |
