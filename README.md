# Foundry Content Understanding Demo

非構造化データの構造化から AI エージェントによる業務照合まで

## シナリオ

顧客から受領した PDF 請求書を、社内の発注データ (Fabric Lakehouse) と自動照合する業務を AI エージェントで効率化するデモ。

### デモフロー

1. **Content Understanding** で PDF 請求書から構造化データを抽出
2. 抽出データを **AI Search** にインデックス
3. **Foundry Agent** が Fabric Data Agent (構造化) + AI Search (非構造化) を横断して照合
4. 差異があれば Agent が報告

## アーキテクチャ

```
[PDF 請求書] → [Blob Storage / OneLake]
                    ↓
        [Content Understanding]
        (OCR + フィールド抽出)
                    ↓
            [AI Search Index]
                    ↓
        [Fabric Data Agent] ←── [Fabric Lakehouse (WWI データ)]
                    ↓
          [Foundry Agent]
          (照合・差異検出・回答)
```

## サンプルデータ

| データ | 説明 | 格納先 |
|--------|------|--------|
| Wide World Importers | 卸売業の発注・請求・顧客データ | `sample-data/structured/` |
| PDF 請求書 | WWI データから生成したフェイク請求書 | `sample-data/pdf/` |

## ディレクトリ構成

```
foundry-content-understanding-demo/
├── README.md                  # このファイル
├── docs/                      # アーキテクチャ図・設計ドキュメント
├── scripts/                   # PDF 生成スクリプト、データ準備スクリプト
├── sample-data/
│   ├── pdf/                   # 生成した PDF 請求書
│   └── structured/            # WWI 構造化データ (CSV/Parquet)
├── infra/                     # Bicep / Terraform (インフラ定義)
└── src/
    ├── content-processing/    # Content Understanding 処理コード
    └── agent/                 # Foundry Agent 定義・コード
```

## 前提条件

- Azure サブスクリプション
- Microsoft Fabric ライセンス (F2 以上)
- Azure AI Foundry プロジェクト
- Python 3.11+

## セットアップ手順

> TODO: 各ステップの詳細を追記

1. WWI サンプルデータを Fabric Lakehouse に取り込み
2. `scripts/generate_invoices.py` で PDF 請求書を生成
3. Content Understanding でアナライザーを設定
4. AI Search インデックスを作成
5. Foundry Agent を構成
