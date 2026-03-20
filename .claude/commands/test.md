# /test — customer-support-n8n テスト手順

以下のステップを上から順に実行し、各確認項目をチェックしてください。

---

## STEP 1【必須】共通：環境構築・インポート

### 1-1. 環境変数ファイルの準備

```bash
# .env.example をコピー（初回のみ）
cp .env.example .env
```

`.env` を開いて以下3箇所を実際の値に書き換えてください：

```
ANTHROPIC_API_KEY=sk-ant-xxxx...        # Anthropic APIキー
SLACK_WEBHOOK_URL=https://hooks.slack.com/services/...  # Slack Webhook URL
N8N_BASIC_AUTH_PASSWORD=your-secure-password            # 任意のパスワード
```

### 1-2. コンテナ起動

```bash
docker-compose up -d
```

### 1-3. 起動確認

```bash
# コンテナが Running 状態か確認
docker ps --filter "name=customer-support-n8n"

# ログにエラーがないか確認
docker logs customer-support-n8n --tail 30
```

**確認ポイント：**
- [ ] コンテナの STATUS が `Up` になっている
- [ ] ログに `Editor is now accessible via: http://localhost:5678` が表示される
- [ ] http://localhost:5678 にブラウザでアクセスしてログイン画面が表示される

### 1-4. ワークフローのインポート

1. http://localhost:5678 にアクセス（ID: `admin` / PW: `.env` で設定したパスワード）
2. 左メニュー → **Workflows** → **Add workflow** → **Import from file**
3. `webhook/workflow.json` を選択してインポート
4. ワークフロー右上のトグルを **Active** に切り替え

**確認ポイント：**
- [ ] ワークフロー名「顧客サポート自動トリアージ＆ルーティング」が表示される
- [ ] ステータスが `Active` になっている
- [ ] 15ノードが正しく表示される

---

## STEP 2【高】パターン4固有：AI分類の4カテゴリ検証

Webhookエンドポイントに4種類の問い合わせを送信し、分類結果を確認します。

### 2-1. 返金・請求（priority: HIGH 期待）

```bash
curl -s -X POST http://localhost:5678/webhook/support-triage \
  -H 'Content-Type: application/json' \
  -d '{
    "from": "tanaka@example.com",
    "subject": "先月の請求が二重になっています",
    "body": "お世話になっております。田中と申します。先月分の請求が2回引き落とされており、至急ご確認をお願いしたいです。"
  }'
```

**確認ポイント：**
- [ ] `category: "返金・請求"` に分類される
- [ ] `priority: "HIGH"` が付与される
- [ ] Slackに通知が届く

### 2-2. 技術サポート（priority: MEDIUM 期待）

```bash
curl -s -X POST http://localhost:5678/webhook/support-triage \
  -H 'Content-Type: application/json' \
  -d '{
    "from": "sato@example.com",
    "subject": "アプリにログインできません",
    "body": "昨日からアプリにログインできない状態が続いています。パスワードリセットも試しましたが改善しません。"
  }'
```

**確認ポイント：**
- [ ] `category: "技術サポート"` に分類される
- [ ] `priority: "MEDIUM"` が付与される

### 2-3. 契約・解約

```bash
curl -s -X POST http://localhost:5678/webhook/support-triage \
  -H 'Content-Type: application/json' \
  -d '{
    "from": "yamada@example.com",
    "subject": "サービスを解約したい",
    "body": "今月末でサービスを解約したいと考えています。手続き方法を教えてください。"
  }'
```

**確認ポイント：**
- [ ] `category: "契約・解約"` に分類される

### 2-4. その他（分類不能なケース）

```bash
curl -s -X POST http://localhost:5678/webhook/support-triage \
  -H 'Content-Type: application/json' \
  -d '{
    "from": "guest@example.com",
    "subject": "ご質問",
    "body": "こんにちは。少し聞きたいことがあるのですが、どうすればいいですか。"
  }'
```

**確認ポイント：**
- [ ] `category: "その他"` にフォールバックされる
- [ ] ワークフローがエラーにならず最後まで完了する

### 2-5. 分類精度の全体確認（n8n UI）

1. n8n UI → **Workflows** → ワークフローを開く
2. 右上 **Executions** タブで実行履歴を確認

**確認ポイント：**
- [ ] 4件すべてが `Success` になっている
- [ ] 各実行の「🔍 分類結果パース」ノードでJSONが正しくパースされている
- [ ] 返信下書きが日本語・適切な文体で生成されている

---

## STEP 3【高】パターン4固有：HIL承認フロー検証

### 3-1. 承認フローのトリガー

まずSTEP 2の任意のリクエストを再送信してワークフローをWait状態にします：

```bash
curl -s -X POST http://localhost:5678/webhook/support-triage \
  -H 'Content-Type: application/json' \
  -d '{
    "from": "hil-test@example.com",
    "subject": "HILテスト：承認フロー確認用",
    "body": "承認フローのテスト用問い合わせです。"
  }'
```

**確認ポイント：**
- [ ] n8n UI の Executions で実行が `Waiting` 状態になっている
- [ ] Slackに「✅ 承認して送信」「❌ スキップ」ボタン付き通知が届いている

### 3-2. 承認（approve）

```bash
curl -s -X POST http://localhost:5678/webhook/approval \
  -H 'Content-Type: application/json' \
  -d '{"action": "approve"}'
```

**確認ポイント：**
- [ ] レスポンスに承認済みメッセージが返る
- [ ] n8n UI でワークフローが `Success` に変わる
- [ ] 「✅ 承認済みレスポンス」ノードが実行されている

### 3-3. スキップ（skip）

新たにリクエストを送信して再度Wait状態にしてから：

```bash
curl -s -X POST http://localhost:5678/webhook/support-triage \
  -H 'Content-Type: application/json' \
  -d '{
    "from": "skip-test@example.com",
    "subject": "スキップテスト用",
    "body": "スキップフローの確認用です。"
  }'
```

```bash
curl -s -X POST http://localhost:5678/webhook/approval \
  -H 'Content-Type: application/json' \
  -d '{"action": "skip"}'
```

**確認ポイント：**
- [ ] レスポンスにスキップメッセージが返る
- [ ] 「❌ スキップレスポンス」ノードが実行されている（承認ノードとは別分岐）

---

## STEP 4【中】パターン4固有：Slack通知確認

STEP 2・3 で届いたSlack通知を目視確認します。

**確認ポイント：**
- [ ] メッセージがBlock Kit形式（見出し・区切り線・フィールド）で表示される
- [ ] 以下の情報がすべて含まれている：
  - 問い合わせ元メールアドレス
  - 件名・本文（抜粋）
  - カテゴリ（日本語表記）
  - 優先度（HIGH / MEDIUM / LOW）
  - 担当チーム名
  - 分類理由（1文）
  - 返信下書きプレビュー（最初の500文字）
- [ ] 「✅ 承認して送信」「❌ スキップ」ボタンが表示される
- [ ] 4カテゴリで通知フォーマットが崩れていない

---

## STEP 5【中】異常系

### 5-1. 不正なJSONでPOST

```bash
curl -s -X POST http://localhost:5678/webhook/support-triage \
  -H 'Content-Type: application/json' \
  -d 'this is not json'
```

**確認ポイント：**
- [ ] n8n がエラーレスポンスを返す（500 or 400）
- [ ] コンテナがクラッシュしない（`docker ps` で確認）

### 5-2. 必須フィールド欠落

```bash
curl -s -X POST http://localhost:5678/webhook/support-triage \
  -H 'Content-Type: application/json' \
  -d '{"from": "test@example.com"}'
```

**確認ポイント：**
- [ ] ワークフローがエラーになるか、空文字として処理されるか確認
- [ ] n8n UI の Executions にエラーログが記録される

### 5-3. Claude APIキー無効時の挙動

`.env` の `ANTHROPIC_API_KEY` を無効な値に変更してコンテナを再起動：

```bash
# 一時的にAPIキーを無効化
docker-compose down
# .env の ANTHROPIC_API_KEY=invalid-key に変更後
docker-compose up -d

curl -s -X POST http://localhost:5678/webhook/support-triage \
  -H 'Content-Type: application/json' \
  -d '{"from":"err@example.com","subject":"APIエラーテスト","body":"テスト"}'
```

**確認ポイント：**
- [ ] 「🤖 AI分類・優先度付け」ノードでエラーになる
- [ ] n8n UI のExecuitonsにエラー詳細が記録される
- [ ] テスト後は正しいAPIキーに戻してコンテナを再起動

```bash
# APIキーを元に戻す
docker-compose down && docker-compose up -d
```

### 5-4. コンテナ再起動後のデータ永続化確認

```bash
docker-compose down && docker-compose up -d

# 再起動後にワークフローが残っているか確認
docker logs customer-support-n8n --tail 10
```

**確認ポイント：**
- [ ] http://localhost:5678 にアクセスしてワークフローが残っている
- [ ] `n8n_data/` ボリュームにデータが保持されている

---

## テスト完了チェックリスト

| フェーズ | 項目 | 結果 |
|---------|------|------|
| STEP 1 | コンテナ起動・ワークフローインポート | ☐ OK / ☐ NG |
| STEP 2 | 返金・請求の分類 | ☐ OK / ☐ NG |
| STEP 2 | 技術サポートの分類 | ☐ OK / ☐ NG |
| STEP 2 | 契約・解約の分類 | ☐ OK / ☐ NG |
| STEP 2 | その他へのフォールバック | ☐ OK / ☐ NG |
| STEP 3 | 承認フロー（approve） | ☐ OK / ☐ NG |
| STEP 3 | スキップフロー（skip） | ☐ OK / ☐ NG |
| STEP 4 | Slack通知のフォーマット | ☐ OK / ☐ NG |
| STEP 5 | 異常系（不正JSON・欠落フィールド） | ☐ OK / ☐ NG |
| STEP 5 | データ永続化 | ☐ OK / ☐ NG |
