# Feature Specification: PDF 請求書照合デモ (Content Understanding + Foundry Agent)

**Feature Branch**: `001-invoice-matching-demo`  
**Created**: 2026-03-26  
**Status**: Draft  
**Input**: User description: "WWI サンプルデータから PDF 請求書を自動生成し、Content Understanding で構造化、AI Search にインデックス、Foundry Agent で Fabric Data Agent × AI Search の横断照合を行うデモ。OneLake + AI Search をナレッジソースとする別シナリオも用意。"

## User Scenarios & Testing *(mandatory)*

### User Story 1 - PDF 請求書の構造化抽出 (Priority: P1)

デモ発表者が、顧客から受領した PDF 請求書をシステムにアップロードすると、請求書番号・仕入先名・請求日・明細行 (商品名・数量・単価・金額)・合計金額が自動的に抽出され、構造化データとして保存される。

**Why this priority**: デモの起点であり、非構造化データ → 構造化の価値を最も直接的に示す。この機能単体でも「手作業の削減」の価値を訴求できる。

**Independent Test**: PDF 請求書をアップロードし、抽出された構造化データ (JSON) が正しいフィールドと値を含むことを確認する。

**Acceptance Scenarios**:

1. **Given** WWI の発注データから生成された PDF 請求書がある, **When** ユーザーが PDF をアップロードする, **Then** 請求書番号・仕入先名・請求日・明細行・合計金額が JSON 形式で抽出される
2. **Given** 抽出処理が完了した, **When** 抽出結果を確認する, **Then** 各フィールドに信頼度スコアが付与されている
3. **Given** 手書き注釈や印影がある PDF をアップロードした, **When** 抽出処理が実行される, **Then** 印刷テキスト部分は正しく抽出され、手書き部分は可能な範囲で抽出される

---

### User Story 2 - Foundry Agent による請求書照合 (シナリオ A: Fabric Data Agent × AI Search) (Priority: P1)

デモ発表者が Foundry Agent に「この請求書の内容は社内の発注データと一致していますか？」と自然言語で質問すると、Agent が Fabric Lakehouse の発注データ (構造化) と AI Search の請求書データ (非構造化から抽出済み) を横断的に検索し、一致・不一致の結果を根拠付きで回答する。

**Why this priority**: デモの核心部分。AI Agent が複数のデータソースを横断して業務判断を支援する価値を直接示す。

**Independent Test**: Agent に照合質問を投げ、回答に (1) 一致/不一致の判定、(2) 具体的な差異の指摘、(3) 参照元データへの引用 が含まれることを確認する。

**Acceptance Scenarios**:

1. **Given** 発注データと一致する正常な請求書が登録されている, **When** Agent に「PO-12345 の請求書は発注内容と一致していますか？」と質問する, **Then** Agent は「一致しています」と回答し、照合した発注番号・金額・日付を引用付きで提示する
2. **Given** 金額が発注データと異なる請求書が登録されている, **When** Agent に照合を依頼する, **Then** Agent は「不一致があります」と回答し、差異の詳細 (例: 請求書 ¥105,000 vs 発注 ¥100,000) を提示する
3. **Given** 社内に対応する発注データが存在しない請求書が登録されている, **When** Agent に照合を依頼する, **Then** Agent は「対応する発注データが見つかりません」と回答する

---

### User Story 3 - Foundry Agent による請求書照合 (シナリオ B: OneLake + AI Search ナレッジソース) (Priority: P2)

デモ発表者が、Foundry IQ の Knowledge Base に OneLake (業務データファイル) と AI Search (抽出済み請求書) をナレッジソースとして接続した別構成のAgent を使い、同様の照合質問を行う。シナリオ A との構成の違いと、それぞれの利点をデモで説明できるようにする。

**Why this priority**: アーキテクチャの選択肢を複数見せることで、顧客環境に応じた提案力を示す。シナリオ A が動いていれば構成の差分で構築可能。

**Independent Test**: OneLake + AI Search を Knowledge Base に接続した Agent に照合質問を投げ、回答が得られることを確認する。

**Acceptance Scenarios**:

1. **Given** OneLake に WWI の業務データファイルが格納され、AI Search に請求書データがインデックスされている, **When** Agent に照合質問をする, **Then** Agent は Foundry IQ の Agentic Retrieval を通じて両ソースから情報を取得し、照合結果を回答する
2. **Given** シナリオ A とシナリオ B の両方が動作している, **When** 同じ照合質問を両 Agent に投げる, **Then** 回答の内容は同等であり、利用したデータソースの違いが応答から確認できる

---

### User Story 4 - デモ用サンプルデータの準備 (Priority: P1)

デモ準備者が、スクリプトを実行すると、WWI サンプルデータの発注・請求テーブルから PDF 請求書が自動生成される。生成される PDF には正常データ (発注と一致)、金額不一致データ、存在しない仕入先データなど、デモシナリオに必要なバリエーションが含まれる。

**Why this priority**: データがなければデモが成立しない。他の全ユーザーストーリーの前提条件。

**Independent Test**: スクリプトを実行し、指定フォルダに複数パターンの PDF 請求書が生成されることを確認する。

**Acceptance Scenarios**:

1. **Given** WWI サンプルデータの CSV/Parquet が用意されている, **When** PDF 生成スクリプトを実行する, **Then** sample-data/pdf/ フォルダに少なくとも 5 件の PDF 請求書が生成される
2. **Given** スクリプトが実行された, **When** 生成された PDF を確認する, **Then** 正常一致・金額不一致・仕入先不在の 3 パターン以上が含まれる
3. **Given** 生成された PDF を Content Understanding で処理する, **When** 抽出結果を確認する, **Then** 請求書番号・仕入先名・金額などのフィールドが正しく抽出できる形式になっている

---

### User Story 5 - デモ用 Web アプリケーション (Priority: P2)

デモ発表者が Web ブラウザからアクセスできるアプリケーションを通じて、PDF のアップロード → 構造化結果の表示 → Agent への照合質問 → 回答表示 という一連のフローを操作する。

**Why this priority**: デモの見栄えと説得力を大きく左右する。ただし Agent の動作確認は CLI やノートブックでも可能なため、必須ではない。

**Independent Test**: ブラウザからアプリにアクセスし、PDF アップロードから照合結果表示までの操作を完了できることを確認する。

**Acceptance Scenarios**:

1. **Given** アプリケーションが起動している, **When** ブラウザからアクセスする, **Then** PDF アップロード画面が表示される
2. **Given** PDF をアップロードした, **When** 処理が完了する, **Then** 抽出された構造化データが画面上に表示される
3. **Given** 構造化データが表示されている, **When** 「照合する」ボタンを押す、またはチャット欄で質問する, **Then** Foundry Agent からの照合結果が表示される

---

### Edge Cases

- PDF が破損している、またはパスワード保護されている場合、適切なエラーメッセージが表示される
- PDF のページ数が極端に多い (50 ページ超) 場合、処理が適切にタイムアウトまたは分割処理される
- Agent に照合対象外の質問 (例: 天気) をした場合、「請求書照合に関する質問をしてください」と案内する
- WWI データに同一の発注番号が複数存在する場合、全候補を提示して確認を促す
- PDF 内のテキストが日本語と英語の混在である場合、両言語を正しく抽出する

## Clarifications

### Session 2026-03-27

- Q: Content Understanding のアナライザー構成は？ → A: カスタムアナライザー (フィールドを明示定義)
- Q: 抽出データの格納先と AI Search への同期方法は？ → A: OneLake に JSON 格納 → AI Search インデクサーで同期 (シナリオ B と共通化)
- Q: Web アプリの技術スタックと Agent フレームワークは？ → A: Python (FastAPI) + Container Apps / React SPA (Fluent UI v2) / Microsoft Agent Framework SDK (Python)
- Q: 発注データと請求書の照合キーは？ → A: 発注番号 (PO Number) を主キーとし、金額・日付を検証項目として照合
- Q: インフラのプロビジョニング方式は？ → A: azd (Azure Developer CLI) + Bicep テンプレート (`azd up` で一括デプロイ)

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: システムは PDF 請求書を受け取り、Content Understanding のカスタムアナライザー (抽出対象フィールドを明示定義) を使用して請求書番号・仕入先名・請求日・明細行 (商品名/数量/単価/金額)・合計金額を抽出できなければならない
- **FR-002**: システムは抽出した構造化データを OneLake に JSON 形式で格納し、AI Search インデクサーで自動同期して自然言語検索可能にしなければならない
- **FR-003**: システムは WWI サンプルデータ (PurchaseOrder / Invoice) を元に、デモ用 PDF 請求書を自動生成するスクリプトを提供しなければならない
- **FR-004**: 生成される PDF には、正常一致パターン、金額不一致パターン、仕入先不在パターンの最低 3 種類のバリエーションが含まれなければならない
- **FR-005**: AI Agent は自然言語の照合質問に対して、発注番号 (PO Number) を主キーとして構造化データソース (業務データ) と検索インデックス (請求書データ) を横断して回答できなければならない。金額・日付は検証項目として使用する
- **FR-006**: AI Agent の回答には、一致/不一致の判定、差異の具体的内容、参照したデータソースの引用が含まれなければならない
- **FR-007**: シナリオ A (Fabric Data Agent × AI Search) とシナリオ B (OneLake + AI Search ナレッジソース) の 2 つの構成で照合デモを実行できなければならない
- **FR-008**: Web アプリケーション (Python FastAPI バックエンド + React SPA フロントエンド、同一 Container Apps コンテナで FastAPI が静的ファイルとして配信) から PDF アップロード → 構造化結果表示 → Agent への照合質問 → 回答表示の一連の操作が可能でなければならない
- **FR-009**: 抽出された各フィールドには信頼度スコアが付与され、結果画面で確認できなければならない
- **FR-010**: デモ環境のセットアップは azd (Azure Developer CLI) + Bicep テンプレートで自動化され、`azd up` で一括デプロイ可能で、手順がドキュメント化され、再現可能でなければならない

### Key Entities

- **PDF 請求書 (Invoice Document)**: 顧客から受領する非構造化ドキュメント。請求書番号・仕入先・日付・明細・合計金額を含む
- **発注データ (Purchase Order)**: Fabric Lakehouse に格納された社内の発注記録。発注番号・仕入先・発注日・明細・金額を含む。WWI サンプルデータから取得
- **抽出結果 (Extraction Result)**: Content Understanding が PDF から抽出した構造化データ。各フィールドの値と信頼度スコアを持つ
- **照合結果 (Matching Result)**: Agent が発注データと請求書データを比較した結果。一致/不一致の判定と差異の詳細を含む

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: PDF 請求書をアップロードしてから構造化データが表示されるまで 30 秒以内で完了する
- **SC-002**: デモ用 PDF 請求書の主要フィールド (請求書番号・仕入先名・合計金額) の抽出精度が 95% 以上である
- **SC-003**: Agent への照合質問から回答表示まで 15 秒以内で完了する
- **SC-004**: Agent が正常一致・金額不一致・仕入先不在の 3 パターンすべてで正しい判定を返す
- **SC-005**: シナリオ A とシナリオ B の両構成でデモが正常に動作する
- **SC-006**: デモ環境のセットアップがドキュメントに従って 2 時間以内に完了できる
- **SC-007**: デモ発表者がスクリプト 1 つでサンプル PDF を再生成でき、PDF の追加・変更が 5 分以内に反映される

## Assumptions

- デモ対象者は Azure / Fabric の基本概念を理解している商社の IT 担当者および意思決定者である
- 商社は Microsoft Fabric を導入済みであり、Snowflake からのミラーリングで業務データを保持している
- デモ環境として Azure サブスクリプションと Fabric ライセンス (F2 以上) が利用可能である
- AI Agent は Microsoft Agent Framework SDK (Python) で構築し、Foundry Agent Service と統合する
- デモ用サンプルデータには WWI (Wide World Importers) を使用し、本番の顧客データは使用しない
- PDF 請求書は A4 サイズ、日本語または英語のテキストベースを想定し、手書きのみの文書は対象外とする
- デモは社内ネットワーク接続がある環境で実施し、オフライン動作は不要である
- Web アプリケーションはデスクトップブラウザでの利用を想定し、モバイル最適化は対象外とする
- 本デモは PoC / デモ目的であり、本番環境のセキュリティ要件 (VNet 統合、Private Endpoint 等) は対象外とする
