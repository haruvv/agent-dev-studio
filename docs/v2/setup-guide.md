# セットアップ手順

openclaw-gateway と openclaw-dev を本番稼働させるための手順。

---

## Step 1: ZhipuAI API キー取得

1. [open.bigmodel.cn](https://open.bigmodel.cn) にアクセス・ログイン
2. 「API Keys」からキーを発行
3. メモしておく（`ZAI_API_KEY`）

---

## Step 2: Telegram Bot 作成

1. Telegram で `@BotFather` に `/newbot` を送る
2. Bot 名・ユーザー名を決める
3. 発行されたトークンをメモ（`TELEGRAM_BOT_TOKEN`）

---

## Step 3: GitHub PAT 発行

1. GitHub → Settings → Developer settings → Personal access tokens → Fine-grained tokens
2. 対象リポジトリ: `haruvv/openclaw-dev`
3. 権限: **Issues（Read & Write）**
4. 発行してメモ（`GITHUB_PERSONAL_ACCESS_TOKEN`）

---

## Step 4: Cloudflare デプロイ

```bash
cd ~/Develop/openclaw-gateway

# Workers Paid プランが必要（Cloudflare ダッシュボードで有効化）

# R2 バケット作成
npx wrangler r2 bucket create openclaw-gateway-data

# Secrets 設定
npx wrangler secret put ZAI_API_KEY
npx wrangler secret put TELEGRAM_BOT_TOKEN
npx wrangler secret put GITHUB_PERSONAL_ACCESS_TOKEN
npx wrangler secret put MOLTBOT_GATEWAY_TOKEN   # openssl rand -hex 32 で生成

# デプロイ
npm run deploy
```

---

## Step 5: Telegram Chat ID を取得

デプロイ後、Bot に話しかけてから：

```bash
curl "https://api.telegram.org/bot<TELEGRAM_BOT_TOKEN>/getUpdates"
```

レスポンスの `message.chat.id` が `TELEGRAM_CHAT_ID`。

```bash
npx wrangler secret put TELEGRAM_CHAT_ID
```

---

## Step 6: openclaw-dev の Secrets 設定

```bash
gh secret set CLAUDE_CODE_OAUTH_TOKEN --repo haruvv/openclaw-dev
gh secret set TELEGRAM_BOT_TOKEN --repo haruvv/openclaw-dev
gh secret set TELEGRAM_CHAT_ID --repo haruvv/openclaw-dev
```

---

## Step 7: 動作確認

1. Telegram で Bot に話しかける → 応答が返れば OK
2. `openclaw-dev` に Issue を作成して `ai-dev` ラベルを付ける → Actions が起動し、Telegram に通知が来れば完了
