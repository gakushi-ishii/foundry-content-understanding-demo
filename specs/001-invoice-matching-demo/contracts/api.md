# API Contracts: Web アプリケーション

**Date**: 2026-03-27 | **Feature**: 001-invoice-matching-demo

## Base URL

```text
http://<container-apps-host>/api
```

## Endpoints

### POST /api/upload

PDF 請求書をアップロードし、Content Understanding で構造化抽出を実行する。

**Request**:

```text
Content-Type: multipart/form-data
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| file | File (PDF) | Yes | 請求書 PDF ファイル |

**Response (200)**:

```json
{
  "id": "invoice-001",
  "invoiceNumber": "INV-2026-001",
  "supplierName": "Fabrikam Ltd",
  "invoiceDate": "2026-03-15",
  "poNumber": "PO-001",
  "lineItems": [
    {
      "productName": "Widget A",
      "quantity": 100,
      "unitPrice": 500,
      "amount": 50000
    }
  ],
  "totalAmount": 50000,
  "currency": "JPY",
  "confidenceScores": {
    "invoiceNumber": 0.98,
    "supplierName": 0.95,
    "invoiceDate": 0.99,
    "poNumber": 0.97,
    "totalAmount": 0.99,
    "lineItems": 0.92
  },
  "sourceFile": "invoice-001.pdf",
  "extractedAt": "2026-03-27T10:30:00Z"
}
```

**Error (400)**:

```json
{
  "error": "invalid_file",
  "message": "PDF ファイルを選択してください"
}
```

**Error (422)**:

```json
{
  "error": "extraction_failed",
  "message": "PDF からデータを抽出できませんでした"
}
```

### POST /api/match

Agent に照合質問を送信し、結果を取得する。

**Request**:

```json
{
  "question": "PO-001 の請求書は発注内容と一致していますか？",
  "scenario": "A"
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| question | string | Yes | 自然言語の照合質問 |
| scenario | string ("A" \| "B") | No | 使用するシナリオ (デフォルト: "A") |

**Response (200)**:

```json
{
  "status": "matched",
  "invoiceNumber": "INV-2026-001",
  "poNumber": "PO-001",
  "differences": [],
  "references": [
    {
      "source": "fabric_data_agent",
      "detail": "PurchaseOrder PO-001: Fabrikam Ltd, ¥50,000"
    },
    {
      "source": "ai_search",
      "detail": "Invoice INV-2026-001: Fabrikam Ltd, ¥50,000"
    }
  ],
  "summary": "一致しています。請求書 INV-2026-001 と発注 PO-001 の内容は全フィールドで一致しています。"
}
```

### GET /api/invoices

アップロード済みの請求書一覧を取得する。

**Response (200)**:

```json
{
  "invoices": [
    {
      "id": "invoice-001",
      "invoiceNumber": "INV-2026-001",
      "supplierName": "Fabrikam Ltd",
      "poNumber": "PO-001",
      "totalAmount": 50000,
      "extractedAt": "2026-03-27T10:30:00Z"
    }
  ]
}
```

### GET /api/health

ヘルスチェック。

**Response (200)**:

```json
{
  "status": "healthy",
  "version": "0.1.0"
}
```
