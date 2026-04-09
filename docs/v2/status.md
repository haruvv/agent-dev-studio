# openclaw-gateway 現状メモ

## 動いていること

- Telegram bot → 返信あり
- cron keepalive → 毎分コンテナ死活確認（デプロイ後最大1分で自動復帰）
- CF Access → `/_admin/` 保護済み（haruki.ito0044@gmail.com のみ）
- dmPolicy=open → 誰でも即返答

## 未解決: R2 バックアップ

### 症状
- Admin UI の「Last backup」が常に Never のまま
- Sync now を押しても変わらない

### 調査済み事項

| 確認内容 | 結果 |
|---------|------|
| R2 バケット `openclaw-gateway-data` の存在 | ✅ 存在する |
| wrangler CLI でのバケット読み書き | ✅ 可能 |
| Sandbox SDK バックアップアップロードログ | "Backup uploaded to R2, Success: true" と出るが R2 に存在しない |
| Worker の `backup-handle.json` 書き込み | ログ上は成功するが R2 に存在しない |
| wrangler CLI で手動書き込みした `backup-handle.json` | Admin UI に反映されない |

### 仮説
Worker の `BACKUP_BUCKET` binding が参照している R2 名前空間と、wrangler CLI が参照しているものがズレている。

### 現状の影響
バックアップなしのため、コンテナ再起動（デプロイ時）で会話履歴がリセットされる。Bot の応答自体は問題なし。

## 設定済み secrets

| Secret | 用途 |
|--------|------|
| TELEGRAM_BOT_TOKEN | Telegram bot |
| TELEGRAM_DM_POLICY | open |
| MOLTBOT_GATEWAY_TOKEN | Gateway 認証 |
| ZAI_API_KEY | AI モデル（zai/glm-5.1） |
| CF_ACCESS_TEAM_DOMAIN | CF Access 認証 |
| CF_ACCESS_AUD | CF Access 認証 |
| R2_ACCESS_KEY_ID | R2 API トークン |
| R2_SECRET_ACCESS_KEY | R2 API トークン |
| CLOUDFLARE_ACCOUNT_ID | bbcaddbc9938b5f98b25475a69589baf |
| BACKUP_BUCKET_NAME | openclaw-gateway-data |
| GITHUB_PERSONAL_ACCESS_TOKEN | GitHub MCP サーバー |
