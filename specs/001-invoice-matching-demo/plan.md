# Implementation Plan: PDF 請求書照合デモ (Content Understanding + Foundry Agent)

**Branch**: `001-invoice-matching-demo` | **Date**: 2026-03-27 | **Spec**: [spec.md](spec.md)
**Input**: Feature specification from `/specs/001-invoice-matching-demo/spec.md`

## Summary

WWI サンプルデータから PDF 請求書を自動生成し、Azure Content Understanding のカスタムアナライザーで構造化抽出、OneLake に JSON 格納、AI Search にインデクサー同期する。Microsoft Agent Framework SDK (Python) で構築した Foundry Agent が、Fabric Data Agent (発注データ) × AI Search (請求書データ) の横断照合 (シナリオ A) および OneLake + AI Search ナレッジソース構成 (シナリオ B) で自然言語照合を実行する。FastAPI + React SPA を Container Apps にデプロイし、azd + Bicep で一括プロビジョニングする。

## Technical Context

**Language/Version**: Python 3.11+, TypeScript 5.x (React)
**Primary Dependencies**: FastAPI, Microsoft Agent Framework SDK, Azure Content Understanding SDK, Azure AI Search SDK, React 18, Fluent UI v2, ReportLab (PDF 生成)
**Storage**: OneLake (JSON), AI Search (インデックス), Fabric Lakehouse (発注データ)
**Testing**: pytest (バックエンド), Vitest (フロントエンド)
**Target Platform**: Azure Container Apps (Linux), デスクトップブラウザ
**Project Type**: web-service (デモアプリケーション)
**Performance Goals**: PDF→構造化 30秒以内 (SC-001), Agent 照合応答 15秒以内 (SC-003)
**Constraints**: デモ/PoC 用途、VNet 統合不要、モバイル非対応
**Scale/Scope**: 5-10件のサンプル PDF、1名のデモ発表者が操作

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

Constitution が未定義 (テンプレート状態) のため、ゲート制約なし。基本方針として YAGNI・シンプルさを優先する。

## Project Structure

### Documentation (this feature)

```text
specs/001-invoice-matching-demo/
├── plan.md              # This file
├── research.md          # Phase 0 output
├── data-model.md        # Phase 1 output
├── quickstart.md        # Phase 1 output
├── contracts/           # Phase 1 output
└── tasks.md             # Phase 2 output (/speckit.tasks command)
```

### Source Code (repository root)

```text
src/
├── agent/                    # Microsoft Agent Framework SDK ベースの Agent コード
│   ├── agent.py              # Agent 定義 (ツール接続、プロンプト)
│   ├── tools/                # Agent ツール (AI Search、Fabric Data Agent 接続)
│   └── config.py             # 設定
├── content-processing/       # Content Understanding 処理
│   ├── analyzer.py           # カスタムアナライザー定義・実行
│   ├── uploader.py           # OneLake への JSON 格納
│   └── config.py             # 設定
└── webapp/                   # FastAPI + React SPA
    ├── backend/
    │   ├── main.py            # FastAPI エントリポイント
    │   ├── routers/           # API エンドポイント
    │   └── services/          # ビジネスロジック
    ├── frontend/
    │   ├── src/
    │   │   ├── components/    # React コンポーネント
    │   │   ├── pages/         # ページ (Upload, Results, Chat)
    │   │   └── services/      # API クライアント
    │   └── package.json
    └── Dockerfile             # FastAPI + 静的ファイル配信

scripts/
├── generate-pdfs.py           # WWI データから PDF 請求書生成
├── setup-analyzer.py          # Content Understanding カスタムアナライザー定義
└── setup-search-index.py      # AI Search インデックス作成・設定

infra/
├── main.bicep                 # azd メインテンプレート
├── modules/
│   ├── container-apps.bicep
│   ├── ai-search.bicep
│   ├── ai-services.bicep
│   ├── ai-foundry.bicep
│   └── container-registry.bicep
└── main.parameters.json

sample-data/
├── pdf/                       # 生成された PDF 請求書
└── structured/                # WWI 元データ (CSV/Parquet)

tests/
├── unit/
├── integration/
└── e2e/
```

**Structure Decision**: Web アプリケーション構成を採用。`src/` 配下に Agent・Content Processing・Web アプリの3モジュールを配置。既存の `src/agent/` と `src/content-processing/` ディレクトリを活用する。

## Complexity Tracking

> Constitution 未定義のため該当なし。
