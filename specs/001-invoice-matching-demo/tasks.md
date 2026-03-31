# Tasks: PDF 請求書照合デモ (Content Understanding + Foundry Agent)

**Input**: Design documents from `/specs/001-invoice-matching-demo/`
**Prerequisites**: plan.md (required), spec.md (required), research.md, data-model.md, contracts/

**Organization**: Tasks are grouped by user story to enable independent implementation and testing of each story.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story this task belongs to (e.g., US1, US2, US3)
- Include exact file paths in descriptions

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: Project initialization, azd 設定、共通設定ファイル

- [ ] T001 Create project structure per implementation plan (src/webapp/backend/, src/webapp/frontend/, scripts/, infra/modules/, tests/)
- [ ] T002 Initialize Python project with pyproject.toml and dependencies (FastAPI, azure-ai-contentunderstanding, azure-search-documents, azure-identity, reportlab, microsoft-agent-framework) in project root
- [ ] T003 [P] Initialize React project with package.json, Vite, Fluent UI v2, TypeScript in src/webapp/frontend/
- [ ] T004 [P] Create azd configuration (azure.yaml) with service mapping to Container Apps in project root
- [ ] T005 [P] Create shared environment configuration (.env.sample, src/webapp/backend/config.py) with Content Understanding, AI Search, OneLake, Foundry endpoints
- [ ] T006 [P] Create multi-stage Dockerfile (Node.js build + Python runtime) for webapp in src/webapp/Dockerfile

**Checkpoint**: プロジェクト構造が確立され、依存関係がインストール可能

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Bicep インフラ、共通モデル、Content Understanding アナライザー設定 — 全ユーザーストーリーの前提

**⚠️ CRITICAL**: No user story work can begin until this phase is complete

- [ ] T007 Create main Bicep template with resource group and parameter definitions in infra/main.bicep
- [ ] T008 [P] Create Container Apps Environment + Container App Bicep module in infra/modules/container-apps.bicep
- [ ] T009 [P] Create AI Search Bicep module with invoice-index definition in infra/modules/ai-search.bicep
- [ ] T010 [P] Create AI Services (Content Understanding) Bicep module in infra/modules/ai-services.bicep
- [ ] T011 [P] Create Container Registry Bicep module in infra/modules/container-registry.bicep
- [ ] T012 [P] Create AI Foundry project Bicep module in infra/modules/ai-foundry.bicep
- [ ] T013 Create Bicep parameter file with default values in infra/main.parameters.json
- [ ] T014 [P] Create InvoiceDocument Pydantic model per data-model.md in src/webapp/backend/models/invoice.py
- [ ] T015 [P] Create MatchingResult Pydantic model per data-model.md in src/webapp/backend/models/matching.py
- [ ] T016 Create Content Understanding カスタムアナライザー定義スクリプト (fieldSchema for invoiceNumber, supplierName, invoiceDate, poNumber, lineItems, totalAmount) in scripts/setup-analyzer.py
- [ ] T017 Create AI Search インデックス・インデクサー・データソース (OneLake) 作成スクリプト per data-model.md index schema in scripts/setup-search-index.py

**Checkpoint**: Azure リソースがデプロイ可能、共通モデルとアナライザー設定準備完了

---

## Phase 3: User Story 4 - デモ用サンプルデータの準備 (Priority: P1) 🎯 MVP

**Goal**: WWI サンプルデータから PDF 請求書を自動生成し、3 パターン以上のバリエーションを含む

**Independent Test**: `python scripts/generate-pdfs.py` を実行し、sample-data/pdf/ に 5 件以上の PDF が生成されることを確認

### Implementation for User Story 4

- [ ] T018 [US4] Create WWI サンプル発注データ (CSV) with PurchaseOrderID, SupplierName, OrderDate, LineItems, TotalAmount in sample-data/structured/purchase-orders.csv
- [ ] T019 [US4] Create PDF 請求書生成スクリプト (ReportLab + Noto Sans JP) with 正常一致・金額不一致・仕入先不在パターン in scripts/generate-pdfs.py
- [ ] T020 [US4] Add Noto Sans JP font file for Japanese PDF rendering in scripts/fonts/NotoSansJP-Regular.ttf
- [ ] T021 [US4] Generate initial sample PDFs (5+ files, including 日英混在パターン) to sample-data/pdf/ by running scripts/generate-pdfs.py

**Checkpoint**: sample-data/pdf/ に正常一致 (PO-001〜PO-003)、金額不一致 (PO-004)、仕入先不在 (PO-999) の PDF が存在する

---

## Phase 4: User Story 1 - PDF 請求書の構造化抽出 (Priority: P1) 🎯 MVP

**Goal**: PDF をアップロードすると Content Understanding で構造化抽出され、信頼度スコア付きの JSON が OneLake に保存される

**Independent Test**: PDF をアップロードし、抽出された JSON に invoiceNumber, supplierName, poNumber, totalAmount, confidenceScores が含まれることを確認

### Implementation for User Story 1

- [ ] T022 [US1] Implement Content Understanding analyzer client (カスタムアナライザー実行、結果パース、信頼度スコア取得) in src/content-processing/analyzer.py
- [ ] T023 [US1] Implement OneLake JSON uploader (抽出結果を Lakehouse Files に格納) in src/content-processing/uploader.py
- [ ] T024 [US1] Create Content Processing config (endpoint, analyzer ID, OneLake path) in src/content-processing/config.py
- [ ] T025 [US1] Implement upload service (PDF 受信 → Content Understanding 抽出 → OneLake 格納 → レスポンス構築) in src/webapp/backend/services/upload_service.py
- [ ] T026 [US1] Implement POST /api/upload endpoint per contracts/api.md in src/webapp/backend/routers/upload.py
- [ ] T027 [US1] Implement GET /api/invoices endpoint per contracts/api.md in src/webapp/backend/routers/invoices.py
- [ ] T028 [US1] Create FastAPI main app with routers, health endpoint, CORS, static files mount in src/webapp/backend/main.py

**Checkpoint**: POST /api/upload に PDF を送信すると、信頼度スコア付きの InvoiceDocument JSON が返り、OneLake に保存される

---

## Phase 5: User Story 2 - Foundry Agent による請求書照合 シナリオ A (Priority: P1) 🎯 MVP

**Goal**: Foundry Agent が Fabric Data Agent (発注データ) × AI Search (請求書データ) を横断し、PO Number ベースで照合結果を回答する

**Independent Test**: Agent に「PO-001 の請求書は一致していますか？」と質問し、一致/不一致の判定・差異・参照元引用が返ることを確認

### Implementation for User Story 2

- [ ] T029 [US2] Create Agent config (Foundry project, workspace_id, artifact_id, AI Search endpoint) in src/agent/config.py
- [ ] T030 [US2] Implement AI Search tool (AzureAISearchTool でインデックス検索、PO Number フィルタ) in src/agent/tools/search_tool.py
- [ ] T031 [US2] Implement Fabric Data Agent tool (MicrosoftFabricPreviewTool で発注データクエリ) in src/agent/tools/fabric_tool.py
- [ ] T032 [US2] Implement Agent 定義 (System Prompt に照合ルール: PO Number 主キー、金額・日付検証、回答フォーマット指示、重複 PO Number 時の全候補提示ルール) in src/agent/agent.py
- [ ] T033 [US2] Implement match service (Agent への質問送信、レスポンス解析、MatchingResult 構築) in src/webapp/backend/services/match_service.py
- [ ] T034 [US2] Implement POST /api/match endpoint (scenario="A") per contracts/api.md in src/webapp/backend/routers/match.py

**Checkpoint**: POST /api/match に照合質問を送信すると、Agent がシナリオ A 構成で回答を返す (matched / mismatched / not_found)

---

## Phase 6: User Story 3 - Foundry Agent による請求書照合 シナリオ B (Priority: P2)

**Goal**: Foundry IQ Knowledge Base (OneLake + AI Search) をナレッジソースとした別構成の Agent で同様の照合を実行する

**Independent Test**: scenario="B" で照合質問を送信し、Agentic Retrieval 経由で回答が得られることを確認

### Implementation for User Story 3

- [ ] T035 [US3] Implement シナリオ B Agent 定義 (MCPTool + Foundry IQ Knowledge Base 接続) in src/agent/agent_scenario_b.py
- [ ] T036 [US3] Update match service to support scenario="B" routing in src/webapp/backend/services/match_service.py
- [ ] T037 [US3] Create Knowledge Base セットアップ手順ドキュメント (OneLake + AI Search ナレッジソース接続) in docs/setup-scenario-b.md

**Checkpoint**: POST /api/match に scenario="B" を指定すると、シナリオ B の Agent が Agentic Retrieval で回答を返す

---

## Phase 7: User Story 5 - デモ用 Web アプリケーション (Priority: P2)

**Goal**: ブラウザから PDF アップロード → 構造化結果表示 → 照合質問 → 回答表示の一連のデモフローを操作できる

**Independent Test**: ブラウザで http://localhost:8000 にアクセスし、PDF アップロードから照合結果表示まで完了する

### Implementation for User Story 5

- [ ] T038 [P] [US5] Create API client service (upload, match, invoices endpoints) in src/webapp/frontend/src/services/api.ts
- [ ] T039 [P] [US5] Create PDF upload component (drag & drop, file select, progress) in src/webapp/frontend/src/components/PdfUploader.tsx
- [ ] T040 [P] [US5] Create extraction result component (フィールド表示 + 信頼度スコアバッジ) in src/webapp/frontend/src/components/ExtractionResult.tsx
- [ ] T041 [P] [US5] Create matching result component (一致/不一致ステータス、差異テーブル、参照元引用) in src/webapp/frontend/src/components/MatchingResult.tsx
- [ ] T042 [P] [US5] Create chat input component (自然言語質問入力 + シナリオ A/B 切替) in src/webapp/frontend/src/components/ChatInput.tsx
- [ ] T043 [US5] Create upload page (PdfUploader + ExtractionResult 統合) in src/webapp/frontend/src/pages/UploadPage.tsx
- [ ] T044 [US5] Create match page (ChatInput + MatchingResult 統合、請求書一覧サイドバー) in src/webapp/frontend/src/pages/MatchPage.tsx
- [ ] T045 [US5] Create App layout with navigation (Upload / Match ページ切替、Fluent UI v2 テーマ) in src/webapp/frontend/src/App.tsx
- [ ] T046 [US5] Configure Vite build output to dist/ and verify FastAPI static files mount in src/webapp/frontend/vite.config.ts

**Checkpoint**: ブラウザから一連のデモフロー (アップロード → 抽出結果 → 照合 → 結果表示) が操作可能

---

## Phase 8: Polish & Cross-Cutting Concerns

**Purpose**: ドキュメント、エラーハンドリング、デモ手順の最終整備

- [ ] T047 [P] Create デモ環境セットアップガイド (azd up 手順、Fabric Lakehouse 手動設定、データインポート手順、マネージド ID の RBAC 設定 (Fabric ワークスペース Contributor)、環境変数設定) in docs/setup-guide.md
- [ ] T048 [P] Create デモ実行手順書 (デモフロー、シナリオ A/B の説明ポイント、想定 Q&A) in docs/demo-script.md
- [ ] T049 [P] Add edge case handling: PDF validation (破損チェック、サイズ制限、パスワード保護検出、50ページ超のページ数制限) in src/webapp/backend/services/upload_service.py
- [ ] T050 [P] Add edge case handling: Agent のスコープ外質問に対するガードレール in src/agent/agent.py
- [ ] T051 Update README.md with project overview, architecture diagram, quickstart reference in README.md
- [ ] T052 Measure and verify performance criteria: SC-001 (PDF→構造化 30秒以内) and SC-003 (Agent照合応答 15秒以内) with sample PDFs
- [ ] T053 Run quickstart.md validation (全手順を実行し、デモが正常動作することを確認)

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: No dependencies — can start immediately
- **Foundational (Phase 2)**: Depends on Setup completion — BLOCKS all user stories
- **US4: サンプルデータ (Phase 3)**: Depends on Foundational — 他の US の前提データを生成
- **US1: 構造化抽出 (Phase 4)**: Depends on Foundational + US4 (テスト用 PDF が必要)
- **US2: シナリオ A 照合 (Phase 5)**: Depends on US1 (AI Search にデータが必要)
- **US3: シナリオ B 照合 (Phase 6)**: Depends on US2 (Agent の差分構築)
- **US5: Web アプリ (Phase 7)**: Depends on US1 の API (フロントエンドが API を呼び出す)、US2 と並行可能
- **Polish (Phase 8)**: Depends on all desired user stories being complete

### User Story Dependencies

- **US4 (P1)**: Foundational 完了後すぐ開始可能 — 他の US の前提
- **US1 (P1)**: US4 完了後に開始 (テスト用 PDF が必要)
- **US2 (P1)**: US1 完了後に開始 (AI Search にインデックスされたデータが必要)
- **US3 (P2)**: US2 完了後に開始 (シナリオ A の差分で構築)
- **US5 (P2)**: US1 の API 完了後に開始可能。US2 と並行して進められる

### Within Each User Story

- Models before services
- Services before endpoints
- Core implementation before integration

### Parallel Opportunities

- Setup: T003, T004, T005, T006 は並行実行可能
- Foundational: T008〜T012 (Bicep modules) は並行実行可能、T014〜T015 (models) は並行実行可能
- US5: T038〜T042 (コンポーネント) は並行実行可能

---

## Parallel Examples

### Phase 2 (Foundational) Parallel Batch

```bash
# Bicep modules — all independent
Task: T008 "Create Container Apps Bicep module in infra/modules/container-apps.bicep"
Task: T009 "Create AI Search Bicep module in infra/modules/ai-search.bicep"
Task: T010 "Create AI Services Bicep module in infra/modules/ai-services.bicep"
Task: T011 "Create Container Registry Bicep module in infra/modules/container-registry.bicep"
Task: T012 "Create AI Foundry project Bicep module in infra/modules/ai-foundry.bicep"

# Models — all independent
Task: T014 "Create InvoiceDocument Pydantic model in src/webapp/backend/models/invoice.py"
Task: T015 "Create MatchingResult Pydantic model in src/webapp/backend/models/matching.py"
```

### Phase 7 (US5 Web App) Parallel Batch

```bash
# React components — all independent
Task: T038 "Create API client service in src/webapp/frontend/src/services/api.ts"
Task: T039 "Create PDF upload component in src/webapp/frontend/src/components/PdfUploader.tsx"
Task: T040 "Create extraction result component in src/webapp/frontend/src/components/ExtractionResult.tsx"
Task: T041 "Create matching result component in src/webapp/frontend/src/components/MatchingResult.tsx"
Task: T042 "Create chat input component in src/webapp/frontend/src/components/ChatInput.tsx"
```

---

## Implementation Strategy

### MVP First (US4 + US1 + US2)

1. Complete Phase 1: Setup
2. Complete Phase 2: Foundational (CRITICAL — blocks all stories)
3. Complete Phase 3: US4 サンプルデータ生成
4. Complete Phase 4: US1 構造化抽出
5. Complete Phase 5: US2 シナリオ A 照合
6. **STOP and VALIDATE**: デモコアフロー (PDF → 抽出 → 照合) が動作することを確認
7. Deploy/demo if ready — CLI / API レベルでデモ可能

### Incremental Delivery

1. Setup + Foundational → 基盤完成
2. US4 サンプルデータ → PDF 生成確認
3. US1 構造化抽出 → API レベルで抽出確認 (MVP-1)
4. US2 シナリオ A → API レベルで照合確認 (MVP-2)
5. US5 Web アプリ → ブラウザでデモ可能 (MVP-3)
6. US3 シナリオ B → 2 構成比較デモ完成 (Full Demo)
7. Polish → ドキュメント整備、デモリハーサル
