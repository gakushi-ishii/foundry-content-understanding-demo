# Quickstart: PDF 請求書照合デモ

**Date**: 2026-03-27 | **Feature**: 001-invoice-matching-demo

## 前提条件

- Azure サブスクリプション (Contributor ロール)
- Microsoft Fabric ライセンス (F2 以上) と Fabric ワークスペース
- Azure Developer CLI (`azd`) v1.9+
- Docker Desktop
- Python 3.11+
- Node.js 20+
- Azure CLI (`az`) v2.60+

## 手順

### 1. リポジトリのクローンと設定

```bash
git clone <repo-url>
cd 001-invoice-matching-demo
```

### 2. Fabric Lakehouse の準備 (手動)

1. Fabric ポータルで Lakehouse を作成
2. WWI サンプルデータ (`sample-data/structured/`) を Lakehouse テーブルにインポート
3. Fabric Data Agent を作成し、Lakehouse を接続
4. workspace_id と artifact_id をメモ

### 3. Azure リソースのデプロイ

```bash
azd auth login
azd init
azd up
```

プロビジョニングされるリソース:

- Azure Container Apps Environment + Container App
- Azure AI Search
- Azure Content Understanding (AI Services)
- Azure Container Registry
- Azure AI Foundry プロジェクト

### 4. サンプル PDF の生成

```bash
python scripts/generate-pdfs.py
```

`sample-data/pdf/` に請求書 PDF が生成される。

### 5. Content Understanding アナライザーの設定

```bash
python scripts/setup-analyzer.py
```

### 6. AI Search インデックスの設定

```bash
python scripts/setup-search-index.py
```

### 7. デモの実行

```bash
# ローカル開発
cd src/webapp
pip install -r requirements.txt
cd frontend && npm install && npm run build && cd ..
uvicorn backend.main:app --reload
```

ブラウザで `http://localhost:8000` にアクセス。

### 8. デモフロー

1. PDF アップロード画面で請求書 PDF を選択・アップロード
2. 構造化抽出結果 (フィールド + 信頼度スコア) を確認
3. チャット欄で「PO-001 の請求書は発注内容と一致していますか？」と質問
4. Agent の照合結果 (一致/不一致、差異、参照元) を確認
5. シナリオ B に切り替えて同じ質問を実行し、構成の違いを説明

## トラブルシューティング

| 症状 | 原因 | 対処 |
|------|------|------|
| PDF アップロードでエラー | Content Understanding の接続設定 | 環境変数の確認 |
| Agent が応答しない | Foundry Agent の接続設定 | workspace_id / artifact_id の確認 |
| AI Search に結果がない | インデクサーの同期遅延 | インデクサーの手動実行 |
