# Research: PDF 請求書照合デモ

**Date**: 2026-03-27 | **Feature**: 001-invoice-matching-demo

## 1. Azure Content Understanding カスタムアナライザー

- **Decision**: `prebuilt-document` をベースにカスタムフィールドスキーマ (`fieldSchema`) を定義するカスタムアナライザーを使用する
- **Rationale**: `prebuilt-invoice` は汎用的だが、発注番号 (PO Number) など業務固有フィールドの抽出にはカスタムフィールド定義が必要。`prebuilt-document` ベースのカスタムアナライザーで精度と柔軟性を両立できる
- **Alternatives considered**:
  - `prebuilt-invoice` そのまま使用 → PO Number フィールドが標準スキーマに含まれない場合がある
  - Document Intelligence 単体 → Content Understanding の信頼度スコア (`estimateFieldSourceAndConfidence: true`) が使えない
- **SDK**: `azure-ai-contentunderstanding` v1.0.1 (GA)
- **前提条件**: gpt-4.1 + text-embedding-3-large モデルのデプロイが必要

## 2. OneLake → AI Search インデクサー同期

- **Decision**: AI Search の `onelake` タイプデータソースを使用し、Lakehouse Files セクションに格納した JSON を自動インデクシングする
- **Rationale**: Fabric SKU を保有しているため OneLake が追加コストなし。シナリオ B のナレッジソースとも共通化でき、アーキテクチャがシンプルになる
- **Alternatives considered**:
  - Blob Storage → Fabric エコシステム外で管理ポイントが増える
  - Push API で直接投入 → インデクサーの自動同期が使えず、再実行時の手間が増える
- **制約**: Parquet / テーブルは AI Search インデクサー非対応。Files セクションの JSON のみサポート
- **認証**: マネージド ID (Fabric ワークスペースの Contributor ロール)

## 3. Foundry Agent - Fabric Data Agent ツール

- **Decision**: `MicrosoftFabricPreviewTool` を使用し、Fabric Data Agent 経由で Lakehouse テーブルをクエリする
- **Rationale**: Foundry Agent から Fabric Lakehouse の発注データに自然言語でアクセスする最も直接的な方法
- **Alternatives considered**:
  - SQL エンドポイント直接接続 → Agent Framework のツール統合を活用できない
  - OneLake API でファイル読み取り → 構造化クエリが困難
- **制約**:
  - Preview 機能 (GA 前)
  - サービスプリンシパル非対応 (OBO / ID パススルーのみ)
  - エンドユーザーに Lakehouse の Read 権限が必要
  - workspace_id + artifact_id で接続設定

## 4. Foundry Agent - AI Search ツール

- **Decision**: シナリオ A は `AzureAISearchTool` (GA、直接インデックス検索)、シナリオ B は Foundry IQ Knowledge Base + `MCPTool` (Agentic Retrieval) を使用
- **Rationale**: 2つの構成の違いを明確にデモできる。シナリオ A はシンプルな直接検索、シナリオ B はクエリ分解・リランキングを含む高度な検索
- **Alternatives considered**:
  - 両シナリオ同じツールタイプ → 構成の差異がデモで見えにくい
- **シナリオ A**: AzureAISearchTool → ハイブリッド検索 (キーワード + ベクトル)
- **シナリオ B**: MCPTool + Foundry IQ → Agentic Retrieval (クエリプランニング + 分解 + リランキング)

## 5. Container Apps デプロイ (azd + Bicep)

- **Decision**: マルチステージ Dockerfile (Node.js ビルド → Python ランタイム) で FastAPI が React 静的ファイルを配信する単一コンテナ構成。`azd up` で一括デプロイ
- **Rationale**: デモ用途で構成をシンプルに保つ。CORS 不要、Dockerfile 1つ、Container Apps 1つで完結
- **Alternatives considered**:
  - フロントエンド別コンテナ → デモ用途ではオーバースペック
  - Azure Static Web Apps → 構成が増える
- **Dockerfile パターン**: Stage 1 で `npm run build`、Stage 2 で Python + FastAPI + `dist/` を `StaticFiles` マウント
- **azure.yaml**: `host: containerapp` でサービスをマッピング

## 6. PDF 生成 (WWI データ)

- **Decision**: ReportLab + Noto Sans JP フォントで日本語 PDF 請求書を生成
- **Rationale**: 日本語 CJK サポートが成熟しており、カスタム TTF フォントで品質の高い請求書 PDF を生成できる
- **Alternatives considered**:
  - fpdf2 → 軽量で API がシンプルだが、複雑なレイアウトの表現力が劣る
  - WeasyPrint → HTML→PDF 変換だが依存が重い
- **フォント**: Noto Sans JP (Google Fonts) をリポジトリにバンドル

## 未解決事項

| 項目 | 影響 | 対応方針 |
|------|------|----------|
| Fabric ワークスペース/Lakehouse の事前作成が必要か | infra/ の Bicep 範囲 | Fabric リソースは手動作成 (Bicep 非対応)、手順をドキュメント化 |
| `prebuilt-invoice` の日本語請求書対応状況 | アナライザー選択 | Phase 1 で検証、不十分な場合は `prebuilt-document` ベースに切り替え |
| Noto Sans JP フォントのバンドル方式 | Docker イメージサイズ | リポジトリに TTF を含める (デモ用途では許容範囲) |
