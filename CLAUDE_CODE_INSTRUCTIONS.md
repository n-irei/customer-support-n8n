# Claude Code 実行指示書
# customer-support-n8n リポジトリのセットアップ＆GitHub push

## 実行手順

以下をClaude Codeのターミナルで順番に実行してください。

---

### STEP 1: プロジェクトディレクトリに移動・初期化

```bash
# このリポジトリのファイルを配置したディレクトリに移動
cd ~/path/to/customer-support-n8n

# Gitを初期化
git init

# 最初のコミット
git add .
git commit -m "feat: initial commit - pattern4 triage routing (n8n webhook)"
```

---

### STEP 2: GitHubにリポジトリ作成＆push（GitHub CLI使用）

```bash
# GitHub CLIでリポジトリ作成＋push（1コマンドで完了）
gh repo create customer-support-n8n \
  --public \
  --description "カスタマーサポート自動化デモ - パターン4 トリアージ＋ルーティング（n8n版）" \
  --source=. \
  --push
```

---

### STEP 3: gh コマンドがない場合のフォールバック

```bash
# GitHub APIでリポジトリ作成
curl -X POST https://api.github.com/user/repos \
  -H "Authorization: token YOUR_GITHUB_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "customer-support-n8n",
    "description": "カスタマーサポート自動化デモ - パターン4 トリアージ＋ルーティング（n8n版）",
    "public": true
  }'

# リモート追加＆push
git remote add origin https://github.com/YOUR_USERNAME/customer-support-n8n.git
git branch -M main
git push -u origin main
```

---

### ファイル構成（確認用）

```
customer-support-n8n/
├── README.md
├── .env.example
├── .gitignore
├── docker-compose.yml
├── webhook/
│   └── workflow.json     ← Webhookトリガー版
├── gmail/                ← 今後追加
└── typeform/             ← 今後追加
```

---

### 完了確認

pushが成功したら以下のURLでリポジトリを確認:
https://github.com/YOUR_USERNAME/customer-support-n8n
