# Data Model: PDF 請求書照合デモ

**Date**: 2026-03-27 | **Feature**: 001-invoice-matching-demo

## エンティティ

### InvoiceDocument (PDF 請求書)

Content Understanding が PDF から抽出する構造化データ。OneLake に JSON として格納し、AI Search にインデクシングする。

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| id | string | Yes | 一意識別子 (ファイル名ベース) |
| invoiceNumber | string | Yes | 請求書番号 |
| supplierName | string | Yes | 仕入先名 |
| invoiceDate | string (ISO 8601) | Yes | 請求日 |
| poNumber | string | Yes | 発注番号 (照合主キー) |
| lineItems | LineItem[] | Yes | 明細行 |
| totalAmount | number | Yes | 合計金額 |
| currency | string | Yes | 通貨 (JPY / USD) |
| confidenceScores | ConfidenceScores | Yes | 各フィールドの信頼度スコア |
| sourceFile | string | Yes | 元 PDF ファイル名 |
| extractedAt | string (ISO 8601) | Yes | 抽出日時 |

### LineItem (明細行)

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| productName | string | Yes | 商品名 |
| quantity | number | Yes | 数量 |
| unitPrice | number | Yes | 単価 |
| amount | number | Yes | 金額 (quantity × unitPrice) |

### ConfidenceScores (信頼度スコア)

| Field | Type | Description |
|-------|------|-------------|
| invoiceNumber | number (0-1) | 請求書番号の抽出信頼度 |
| supplierName | number (0-1) | 仕入先名の抽出信頼度 |
| invoiceDate | number (0-1) | 請求日の抽出信頼度 |
| poNumber | number (0-1) | 発注番号の抽出信頼度 |
| totalAmount | number (0-1) | 合計金額の抽出信頼度 |
| lineItems | number (0-1) | 明細行全体の抽出信頼度 |

### PurchaseOrder (発注データ)

Fabric Lakehouse に格納された WWI ベースの業務データ。Fabric Data Agent 経由でクエリされる。

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| PurchaseOrderID | int | Yes | 発注番号 (照合主キー、poNumber と対応) |
| SupplierName | string | Yes | 仕入先名 |
| OrderDate | date | Yes | 発注日 |
| ExpectedDeliveryDate | date | No | 納品予定日 |
| LineItems | OrderLineItem[] | Yes | 発注明細 |
| TotalAmount | decimal | Yes | 発注合計金額 |
| Currency | string | Yes | 通貨 |

### OrderLineItem (発注明細)

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| StockItemName | string | Yes | 商品名 |
| OrderedQuantity | int | Yes | 発注数量 |
| UnitPrice | decimal | Yes | 単価 |
| Amount | decimal | Yes | 金額 |

### MatchingResult (照合結果)

Agent が返す照合結果の構造。API レスポンスおよび UI 表示に使用。

| Field | Type | Description |
|-------|------|-------------|
| status | enum: "matched" \| "mismatched" \| "not_found" | 照合判定 |
| invoiceNumber | string | 照合対象の請求書番号 |
| poNumber | string | 照合対象の発注番号 |
| differences | Difference[] | 差異の詳細 (不一致時) |
| references | Reference[] | 参照したデータソースの引用 |
| summary | string | Agent の回答テキスト |

### Difference (差異)

| Field | Type | Description |
|-------|------|-------------|
| field | string | 差異のあるフィールド名 |
| invoiceValue | string | 請求書側の値 |
| orderValue | string | 発注側の値 |

### Reference (参照元)

| Field | Type | Description |
|-------|------|-------------|
| source | enum: "fabric_data_agent" \| "ai_search" \| "onelake" | データソース種別 |
| detail | string | 参照内容の詳細 |

## リレーション

```text
InvoiceDocument.poNumber ──(照合主キー)──> PurchaseOrder.PurchaseOrderID
InvoiceDocument.lineItems ──(明細比較)──> PurchaseOrder.LineItems
InvoiceDocument.totalAmount ──(金額検証)──> PurchaseOrder.TotalAmount
```

## デモデータバリエーション

| パターン | poNumber | 説明 |
|----------|----------|------|
| 正常一致 | PO-001〜PO-003 | 全フィールドが発注データと一致 |
| 金額不一致 | PO-004 | totalAmount が発注データと異なる |
| 仕入先不在 | PO-999 | 対応する発注データが存在しない |

## AI Search インデックススキーマ

インデックス名: `invoice-index`

| Field | Type | Searchable | Filterable | Sortable | Key |
|-------|------|------------|------------|----------|-----|
| id | Edm.String | No | No | No | Yes |
| invoiceNumber | Edm.String | Yes | Yes | No | No |
| supplierName | Edm.String | Yes | Yes | No | No |
| invoiceDate | Edm.DateTimeOffset | No | Yes | Yes | No |
| poNumber | Edm.String | Yes | Yes | No | No |
| totalAmount | Edm.Double | No | Yes | Yes | No |
| currency | Edm.String | No | Yes | No | No |
| lineItems | Collection(Edm.ComplexType) | Yes | No | No | No |
| sourceFile | Edm.String | No | Yes | No | No |
| content | Edm.String | Yes | No | No | No |
