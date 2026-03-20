# customer-support-n8n

> カスタマーサポート自動化デモ — **パターン4：トリアージ＋ルーティング（n8n版）**

問い合わせをAIが自動分類・優先度付けし、担当者への通知と返信下書きを生成。  
担当者が承認してから送信する **Human-in-the-Loop（HIL）** 設計です。

```
[Webhook受信] → [AI分類・優先度付け] → [カテゴリルーティング]
      ↓
[返信下書き生成] → [Slack通知] → [担当者承認（HIL）] → [完了]
```

---

## トリガーバリエーション

| フォルダ | トリガー | 説明 |
|---|---|---|
| [`webhook/`](./webhook/) | Webhook POST | サンプルデータで即テスト可能 |
| [`gmail/`](./gmail/) | Gmail 新着メール | 実メールで自動起動 |
| [`typeform/`](./typeform/) | Typeform フォーム送信 | 問い合わせフォーム連携 |

---

## セットアップ

### 1. 必要なもの

- Docker / Docker Compose
- Anthropic API キー
- Slack Incoming Webhook URL

### 2. リポジトリのクローン

```bash
git clone https://github.com/yourname/customer-support-n8n.git
cd customer-support-n8n
```

### 3. 環境変数の設定

```bash
cp .env.example .env
# .env を編集して APIキーを設定
```

### 4. n8n を起動

```bash
docker-compose up -d
```

ブラウザで `http://localhost:5678` を開く

### 5. ワークフローをインポート

1. n8n の画面右上メニュー → **Import from File**
2. `webhook/workflow.json` を選択
3. `ANTHROPIC_API_KEY` と `SLACK_WEBHOOK_URL` が環境変数に設定されていることを確認
4. ワークフローを **Activate**

---

## テスト実行

```bash
curl -X POST http://localhost:5678/webhook/support-triage \
  -H 'Content-Type: application/json' \
  -d '{
    "from": "tanaka@example.com",
    "subject": "先月の請求が二重になっています",
    "body": "お世話になっております。先月分の請求が2回引き落とされているようです。確認いただけますか？"
  }'
```

### 承認（HIL）のテスト

Slack通知を受け取った後、手動で承認する場合：

```bash
# 承認
curl -X POST http://localhost:5678/webhook/approval \
  -H 'Content-Type: application/json' \
  -d '{"action": "approve"}'

# スキップ
curl -X POST http://localhost:5678/webhook/approval \
  -H 'Content-Type: application/json' \
  -d '{"action": "skip"}'
```

---

## 関連リポジトリ

| リポジトリ | 内容 |
|---|---|
| [customer-support-langgraph](https://github.com/yourname/customer-support-langgraph) | 同じユースケースをPython/LangGraphで実装 |
| [customer-support-dify](https://github.com/yourname/customer-support-dify) | FAQ・RAG型をDifyで実装 |

---

## ライセンス

MIT License
