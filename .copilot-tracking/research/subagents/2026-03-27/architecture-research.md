# Architecture Research: Invoice Matching Demo

## Research Topics

1. Azure Content Understanding Custom Analyzer for invoice field extraction
2. OneLake to AI Search Indexer configuration
3. Microsoft Agent Framework SDK - Fabric Data Agent Tool
4. Microsoft Agent Framework SDK - AI Search Tool
5. Container Apps deployment with azd
6. PDF Generation from WWI data (Japanese invoices)

---

## Topic 1: Azure Content Understanding Custom Analyzer

### Key Discoveries

**SDK Availability**: The `azure-ai-contentunderstanding` Python SDK is GA (v1.0.1), targeting API version `2025-11-01`.

- Package: `pip install azure-ai-contentunderstanding`
- Source: `Azure/azure-sdk-for-python/sdk/contentunderstanding/azure-ai-contentunderstanding`
- Requires Python 3.9+

**Prebuilt vs Custom Analyzer**:

- `prebuilt-invoice` analyzer extracts standard invoice fields (InvoiceId, VendorName, InvoiceDate, LineItems, InvoiceTotal, etc.) with confidence scores out of the box.
- Custom analyzers can be created by defining a `fieldSchema` with custom fields, using a `baseAnalyzerId` of `prebuilt-document`.

**Custom Analyzer Schema Format** (JSON):

```json
{
  "description": "Custom invoice analyzer",
  "baseAnalyzerId": "prebuilt-document",
  "models": {
    "completion": "gpt-4.1",
    "embedding": "text-embedding-ada-002"
  },
  "config": {
    "returnDetails": true,
    "estimateFieldSourceAndConfidence": true,
    "tableFormat": "html"
  },
  "fieldSchema": {
    "fields": {
      "VendorName": {
        "type": "string",
        "method": "extract",
        "description": "Vendor issuing the invoice"
      },
      "Items": {
        "type": "array",
        "method": "extract",
        "items": {
          "type": "object",
          "properties": {
            "Description": { "type": "string", "method": "extract" },
            "Amount": { "type": "number", "method": "extract" }
          }
        }
      }
    }
  }
}
```

**Field extraction methods**: `extract` (direct from content), `classify` (from categories), `generate` (AI-generated from content).

**Confidence Scores**: Enabled via `estimateFieldSourceAndConfidence: true`. Each field returns a `confidence` value (0-1) and a `source` with bounding box coordinates.

**API Pattern**:

1. PUT to create analyzer: `{endpoint}/contentunderstanding/analyzers/{analyzerId}?api-version=2025-11-01`
2. POST to analyze: `{endpoint}/contentunderstanding/analyzers/{analyzerId}:analyze?api-version=2025-11-01`
3. GET result: `{endpoint}/contentunderstanding/analyzerResults/{resultId}?api-version=2025-11-01`

**Python SDK Pattern**:

```python
from azure.ai.contentunderstanding import ContentUnderstandingClient
from azure.ai.contentunderstanding.models import AnalysisInput, AnalysisResult
from azure.identity import DefaultAzureCredential

client = ContentUnderstandingClient(endpoint=endpoint, credential=DefaultAzureCredential())
poller = client.begin_analyze(analyzer_id="prebuilt-invoice", inputs=[AnalysisInput(url=file_url)])
result: AnalysisResult = poller.result()
```

**Required Model Deployments**: `gpt-4.1`, `gpt-4.1-mini`, `text-embedding-3-large` must be deployed in Foundry resource and configured as defaults.

### Decision

- **Decision**: Use `prebuilt-invoice` for the demo, as it covers standard fields. Define a custom analyzer only if WWI-specific fields (e.g., PO Number) aren't in the prebuilt schema.
- **Rationale**: The prebuilt-invoice already extracts InvoiceId, VendorName, InvoiceDate, LineItems (Description, Quantity, UnitPrice, TotalAmount), InvoiceTotal, and DueDate with confidence scores. This matches all spec requirements. A thin custom wrapper may be needed only to add PurchaseOrderNumber extraction.
- **Alternatives**: (1) Fully custom analyzer with all fields explicitly defined - more control but more setup. (2) Use `prebuilt-document` as base and define all fields - useful if prebuilt-invoice doesn't match Japanese invoice format.

---

## Topic 2: OneLake to AI Search Indexer

### Key Discoveries

**Data Source Type**: `"type": "onelake"` — a dedicated indexer type exists for OneLake files.

**Documentation**: https://learn.microsoft.com/en-us/azure/search/search-how-to-index-onelake-files

**Authentication**: Token-based with managed identity (system or user-assigned). The AI Search service's managed identity must be granted **Contributor** role in the Fabric workspace.

**Data Source Configuration**:

```json
{
  "name": "onelake-datasource",
  "type": "onelake",
  "credentials": {
    "connectionString": "ResourceId={FabricWorkspaceGuid}"
  },
  "container": {
    "name": "{LakehouseGuid}",
    "query": "{optionalSubfolder}"
  }
}
```

- `credentials.connectionString`: Uses `ResourceId={FabricWorkspaceGuid}` (workspace GUID from Power BI URL).
- `container.name`: The Lakehouse GUID.
- `container.query`: Optional subfolder path to scope indexing.

**Supported Formats**: JSON, CSV, PDF, Markdown, HTML, Office documents, plain text, XML, and more. JSON parsing modes (arrays, lines) are supported.

**JSON Parsing**: Use `parsingMode: "jsonArray"` or `parsingMode: "jsonLines"` to index individual JSON objects as separate search documents.

**Key Limitations**:

- **Parquet/Delta Parquet NOT supported** — only Files section, not Tables.
- OneLake Table location content is NOT supported — only Files.
- Requires the AI Search service to be in the same tenant as the Fabric workspace.
- Must allow access to OneLake data from applications outside Fabric (`Settings > OneLake > Apps can access OneLake data`).

**Deletion Detection**: Via custom metadata soft-delete pattern (add `IsDeleted` metadata property to files).

**Indexer Configuration**:

```json
{
  "name": "onelake-indexer",
  "dataSourceName": "onelake-datasource",
  "targetIndexName": "invoice-index",
  "parameters": {
    "configuration": {
      "indexedFileNameExtensions": ".json",
      "parsingMode": "json",
      "dataToExtract": "contentAndMetadata"
    }
  },
  "schedule": { "interval": "PT5M" }
}
```

### Decision

- **Decision**: Store Content Understanding extraction results as JSON files in OneLake Lakehouse Files section, and configure an AI Search indexer with `type: "onelake"` and JSON parsing mode.
- **Rationale**: This is the officially supported path. The `onelake` indexer type with JSON parsing mode allows each extracted invoice JSON to become a searchable document. AI Search managed identity authenticates to OneLake via Fabric workspace Contributor role.
- **Alternatives**: (1) Push data directly to AI Search via SDK instead of indexer — more control but requires custom sync logic. (2) Use Azure Blob Storage as intermediate — adds complexity. (3) Use AI Search skillsets with integrated vectorization for chunking/embedding.

---

## Topic 3: Microsoft Agent Framework SDK - Fabric Data Agent Tool

### Key Discoveries

**Documentation**: https://learn.microsoft.com/en-us/azure/foundry/agents/how-to/tools/fabric

**Status**: Preview (public preview, not GA).

**Concept**: A Fabric Data Agent is created and published in Microsoft Fabric. It turns enterprise data (Lakehouse tables, Warehouses, Power BI semantic models, KQL databases) into a conversational Q&A experience using NL2SQL.

**Connection Setup**:

1. Create and publish a Fabric Data Agent in Fabric (under AI Skills).
2. Copy `workspace_id` and `artifact_id` from the Fabric URL: `.../groups/<workspace_id>/aiskills/<artifact_id>`.
3. In Foundry portal, create a connection of type "Microsoft Fabric" with these GUIDs.
4. Copy the connection ID.

**Python SDK**:

```python
from azure.ai.projects import AIProjectClient
from azure.ai.projects.models import (
    PromptAgentDefinition,
    MicrosoftFabricPreviewTool,
    FabricDataAgentToolParameters,
    ToolProjectConnection,
)

project = AIProjectClient(endpoint=PROJECT_ENDPOINT, credential=DefaultAzureCredential())
fabric_connection = project.connections.get(FABRIC_CONNECTION_NAME)

agent = project.agents.create_version(
    agent_name="MyAgent",
    definition=PromptAgentDefinition(
        model="gpt-4.1-mini",
        instructions="You are a helpful assistant.",
        tools=[
            MicrosoftFabricPreviewTool(
                fabric_dataagent_preview=FabricDataAgentToolParameters(
                    project_connections=[
                        ToolProjectConnection(project_connection_id=fabric_connection.id)
                    ]
                )
            )
        ],
    ),
)
```

**Key Constraints**:

- Uses **identity passthrough (On-Behalf-Of)** — end users must have access to the Fabric data agent and underlying data sources.
- Service principal authentication is NOT supported.
- Fabric data agent and Foundry project must be in the same tenant.
- The orchestration model (e.g., gpt-4.1-mini) is separate from the NL2SQL model used by Fabric.
- Does NOT work when agent is published to Microsoft Teams.

**Permissions Required**:

- Lakehouse: Read on the lakehouse item (and table access if enforced).
- Warehouse: Read (SELECT on relevant tables).
- Users need `Azure AI User` RBAC role.

### Decision

- **Decision**: Use `MicrosoftFabricPreviewTool` to connect the Foundry Agent to a Fabric Data Agent that queries WWI purchase order data in Lakehouse tables. This is Scenario A.
- **Rationale**: This provides natural language querying of structured data in Fabric Lakehouse (NL2SQL). The API is straightforward and well-documented despite being Preview. Identity passthrough is acceptable for demo scenarios.
- **Alternatives**: (1) Direct SQL queries via custom function tool — more control but loses the NL2SQL capability. (2) Use Fabric REST API to query data — more complex integration. (3) Pre-extract data and load into AI Search — simpler but loses the live query capability.

---

## Topic 4: Microsoft Agent Framework SDK - AI Search Tool

### Key Discoveries

**Two Approaches Exist**:

1. **AzureAISearchTool** (direct index tool) — GA. Connects agent directly to a specific AI Search index.
2. **MCPTool with Foundry IQ Knowledge Base** — Preview. Uses MCP protocol with AI Search knowledge bases for agentic retrieval with query planning, decomposition, and semantic reranking.

**Approach 1: AzureAISearchTool (GA)**

Documentation: https://learn.microsoft.com/en-us/azure/foundry/agents/how-to/tools/ai-search

```python
from azure.ai.projects.models import (
    AzureAISearchTool, PromptAgentDefinition,
    AzureAISearchToolResource, AISearchIndexResource, AzureAISearchQueryType,
)

azs_connection = project.connections.get(SEARCH_CONNECTION_NAME)
agent = project.agents.create_version(
    agent_name="MyAgent",
    definition=PromptAgentDefinition(
        model="gpt-4.1-mini",
        instructions="...",
        tools=[
            AzureAISearchTool(
                azure_ai_search=AzureAISearchToolResource(
                    indexes=[AISearchIndexResource(
                        project_connection_id=azs_connection.id,
                        index_name=SEARCH_INDEX_NAME,
                        query_type=AzureAISearchQueryType.SIMPLE,
                    )]
                )
            )
        ],
    ),
)
```

- **Supports**: simple, vector, semantic, vector_simple_hybrid, vector_semantic_hybrid query types.
- **Limitation**: Can only target ONE index per tool. Use connected agents for multiple indexes.
- Returns inline citations formatted as `[message_idx:search_idx†source]`.
- Requires vector index (Edm.String searchable + Collection(Edm.Single) vector fields).

**Approach 2: MCPTool with Foundry IQ Knowledge Base (Preview)**

Documentation: https://learn.microsoft.com/en-us/azure/foundry/agents/how-to/foundry-iq-connect

```python
from azure.ai.projects.models import MCPTool

mcp_kb_tool = MCPTool(
    server_label="knowledge-base",
    server_url=f"{search_endpoint}/knowledgebases/{kb_name}/mcp?api-version=2025-11-01-preview",
    require_approval="never",
    allowed_tools=["knowledge_base_retrieve"],
    project_connection_id=project_connection_name,
)
```

- Provides **agentic retrieval**: query planning, decomposition, multi-source retrieval, semantic reranking.
- Knowledge bases can include multiple knowledge sources (AI Search indexes, OneLake files, SharePoint).
- Uses MCP protocol for communication.
- More sophisticated retrieval but requires knowledge base setup in AI Search.

### Decision

- **Decision**: Use both approaches. Scenario A uses `AzureAISearchTool` (direct, GA) alongside `MicrosoftFabricPreviewTool`. Scenario B uses `MCPTool` with Foundry IQ Knowledge Base (connects OneLake + AI Search as knowledge sources).
- **Rationale**: Scenario A demonstrates tool composition (Fabric Data Agent + AI Search direct tool). Scenario B demonstrates the unified knowledge base approach with agentic retrieval. Both are valid patterns and showing both adds demo value.
- **Alternatives**: (1) Use only AzureAISearchTool for both scenarios — simpler but misses the Foundry IQ narrative. (2) Use only MCPTool — powerful but Preview and requires more setup.

---

## Topic 5: Container Apps Deployment with azd + Bicep

### Key Discoveries

**azd Template Structure**:

```
/
├── azure.yaml          # Service definitions mapping to Azure resources
├── infra/
│   ├── main.bicep      # Main Bicep orchestration
│   ├── main.parameters.json
│   └── app/
│       └── containerapp.bicep
├── src/
│   ├── agent/          # FastAPI backend
│   └── frontend/       # React SPA
└── Dockerfile
```

**azure.yaml Pattern for Single Container (FastAPI + React)**:

```yaml
name: invoice-matching-demo
services:
  web:
    project: ./src
    language: python
    host: containerapp
    docker:
      path: ./Dockerfile
```

**Dockerfile Pattern (FastAPI serving React static files)**:

```dockerfile
# Stage 1: Build React
FROM node:20-alpine AS frontend-build
WORKDIR /app/frontend
COPY src/frontend/package*.json ./
RUN npm ci
COPY src/frontend/ ./
RUN npm run build

# Stage 2: Python API + static files
FROM python:3.12-slim
WORKDIR /app
COPY src/agent/requirements.txt ./
RUN pip install --no-cache-dir -r requirements.txt
COPY src/agent/ ./
COPY --from=frontend-build /app/frontend/dist ./static

EXPOSE 8000
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

**FastAPI Static File Serving**:

```python
from fastapi import FastAPI
from fastapi.staticfiles import StaticFiles

app = FastAPI()
# API routes first
app.include_router(api_router, prefix="/api")
# React SPA static files
app.mount("/", StaticFiles(directory="static", html=True), name="static")
```

**Bicep for Container Apps**: Use Azure Verified Modules when available. Key resources: Container Apps Environment, Container App, Container Registry, Log Analytics Workspace.

**azd Deployment Flow**: `azd up` = `azd provision` (Bicep) + `azd deploy` (build & push container).

### Decision

- **Decision**: Use a multi-stage Dockerfile with React build + FastAPI, deployed to Container Apps via azd + Bicep. Single container serves both API and SPA.
- **Rationale**: Matches FR-008 requirement ("同一 Container Apps コンテナで FastAPI が静的ファイルとして配信"). Simplifies deployment to a single container. azd provides unified `azd up` for the entire stack.
- **Alternatives**: (1) Separate containers for frontend and backend — more scalable but unnecessary for demo. (2) Azure Static Web Apps for frontend + Container Apps for API — better separation but adds complexity. (3) App Service instead of Container Apps — simpler but less cloud-native.

---

## Topic 6: PDF Generation from WWI Data (Japanese Invoices)

### Key Discoveries

**ReportLab**:

- Package: `pip install reportlab` (v4.4.10, latest as of 2026-02).
- Open source Python library for generating PDFs and graphics.
- **CJK/Japanese support**: ReportLab has built-in support for CJK fonts via `reportlab.pdfbase.cidfonts` module with `UniJIS-UCS2-H` encoding and standard CJK fonts (HeiseiMin-W3, HeiseiKakuGo-W5).
- Can also register custom TrueType fonts (e.g., Noto Sans JP, IPAex Gothic) via `reportlab.pdfbase.ttfonts.TTFont`.
- Supports tables via `reportlab.lib.tables.Table`, paragraph formatting via `reportlab.platypus.Paragraph`.

**Japanese Font Pattern with ReportLab**:

```python
from reportlab.pdfbase import pdfmetrics
from reportlab.pdfbase.ttfonts import TTFont
from reportlab.pdfbase.cidfonts import UnicodeCIDFont

# Option A: Use built-in CID font
pdfmetrics.registerFont(UnicodeCIDFont('HeiseiMin-W3'))

# Option B: Use custom TTF (better rendering)
pdfmetrics.registerFont(TTFont('NotoSansJP', 'NotoSansJP-Regular.ttf'))
```

**Alternative Libraries**:

- **fpdf2** (`pip install fpdf2`): Lightweight, good Japanese support via `add_font()`, simpler API than ReportLab. MIT license.
- **WeasyPrint**: HTML/CSS to PDF. Good for complex layouts but heavier dependency (requires system packages).
- **borb**: Pure Python PDF library, less mature CJK support.

**WWI Data Source**: The spec mentions using WWI (Wide World Importers) PurchaseOrder/Invoice data. This can be provided as CSV/Parquet files in `sample-data/structured/`.

**PDF Content Requirements** (from spec):

- Invoice number, vendor name, invoice date
- Line items (product name, quantity, unit price, amount)
- Total amount
- At least 3 patterns: normal match, amount mismatch, vendor not found
- Must be extractable by Content Understanding (text-based PDF, not image-only)

### Decision

- **Decision**: Use ReportLab with a custom Japanese TrueType font (Noto Sans JP or IPAex Gothic) bundled in the project. Generate A4 invoices with standard Japanese invoice layout.
- **Rationale**: ReportLab is the most mature Python PDF library with proven CJK support. Custom TTF fonts provide better rendering quality than built-in CID fonts. Noto Sans JP is free (SIL OFL license) and widely used.
- **Alternatives**: (1) fpdf2 — lighter weight, simpler API, good Japanese support, worth considering if ReportLab feels too heavy. (2) WeasyPrint — HTML template approach is more designer-friendly but adds system dependencies. (3) Jinja2 + HTML → WeasyPrint pipeline — flexible template system but complex setup.

---

## Follow-on Questions (Directly Relevant)

1. Does the `prebuilt-invoice` analyzer handle Japanese invoice formats with PO Number (発注番号) fields, or is a custom analyzer required?
2. For OneLake indexer with JSON parsing, what is the optimal JSON structure for invoice extraction results to map cleanly to AI Search index fields?
3. What is the exact Bicep module path for Container Apps in Azure Verified Modules?
4. For Scenario B (Foundry IQ), what knowledge sources can be added to a knowledge base besides AI Search indexes? Can OneLake files be added directly?

## Clarifying Questions (Cannot Answer Through Research)

1. Which Fabric capacity SKU (F2, F4, etc.) is available for the demo environment? This affects Fabric Data Agent availability.
2. Is there an existing Fabric workspace and Lakehouse provisioned, or does the demo need to create them via Bicep/azd?
3. Should the Noto Sans JP font be bundled in the repo, or downloaded at build time?

## References

- Content Understanding overview: https://learn.microsoft.com/en-us/azure/ai-services/content-understanding/overview
- Content Understanding Python SDK: https://github.com/Azure/azure-sdk-for-python/tree/main/sdk/contentunderstanding/azure-ai-contentunderstanding
- Custom analyzer tutorial: https://learn.microsoft.com/en-us/azure/ai-services/content-understanding/tutorial/create-custom-analyzer
- REST API quickstart: https://learn.microsoft.com/en-us/azure/ai-services/content-understanding/quickstart/use-rest-api
- OneLake indexer: https://learn.microsoft.com/en-us/azure/search/search-how-to-index-onelake-files
- Fabric Data Agent tool: https://learn.microsoft.com/en-us/azure/foundry/agents/how-to/tools/fabric
- AI Search tool (new): https://learn.microsoft.com/en-us/azure/foundry/agents/how-to/tools/ai-search
- Foundry IQ Knowledge Base: https://learn.microsoft.com/en-us/azure/foundry/agents/how-to/foundry-iq-connect
- azd templates: https://learn.microsoft.com/en-us/azure/developer/azure-developer-cli/azd-templates
- ReportLab: https://pypi.org/project/reportlab/
