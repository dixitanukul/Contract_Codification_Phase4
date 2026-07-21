# Contract Intelligence Platform V4 — Complete Application Architecture

**Document type:** Technical Architecture Reference
**Audience:** Engineers, Product Managers, Executive Sponsors, Onboarding Team
**Last updated:** 2026-07-21
**Companion document:** `CONTRACT_INTELLIGENCE_V4_BUSINESS_OVERVIEW.md`

---

## Table of Contents

1. Executive Overview
2. System Architecture — The 5-Layer Stack
3. Complete File Map — Every File and Its Role
4. The Frontend — What Users Experience
5. The FastAPI Backend — The Application Gateway
6. What Happens When You Ask a Question (The Crown Jewel)
7. The AI Tools Arsenal — All 25+ Tools
8. The Trust and Validation Layer
9. The LLM Layer — How AI Models Are Used
10. The Data Pipeline — How Knowledge Was Built
11. Operational Infrastructure
12. The 6 Phase 10B REST API Tool Groups
13. Security Architecture
14. Deployment, Build, and Operations

---

## 1. Executive Overview

### What This Application Is

The Contract Intelligence Platform V4 is Blue Shield of California's AI-powered contract analysis system. It transforms 10,349 raw provider contract documents into an interactive intelligence layer that any contract analyst can query in plain English within seconds.

The platform answers questions like:

- "What are Sutter Health's current inpatient DRG rates, and how have they changed across amendments?"
- "Which providers in our Medi-Cal network have an IPA delegation clause?"
- "Show me the exact DOFR matrix for Adventist Health Bakersfield."
- "How many contracts expire in the next 90 days, and what is our financial exposure?"

The answer arrives in seconds, grounded in source documents, with citations to the exact page of the exact PDF.

### The Business Problem It Solves

Before this platform, contract intelligence at BSC meant:

- **Manual PDF review** — analysts searching through hundreds of pages per contract
- **No cross-contract comparison** — impossible to ask "which providers have X clause" across 306 network members
- **Amendment confusion** — no systematic way to track which rate version was current vs. superseded
- **Institutional knowledge risk** — contract knowledge lived in people, not systems
- **Compliance uncertainty** — no real-time view of regulatory compliance status across 7 regulations

This platform eliminates all five problems with a production AI system running on Databricks.

### What Was Built — In 5 Sentences

A FastAPI Python backend serves a React 18 + TypeScript frontend, both deployed as a single Databricks App. When a user asks a question, a Claude Sonnet 4.6-powered ReAct agent orchestrates up to 10 reasoning steps, dispatching from a library of 25+ specialized tools that query 31 structured Delta tables and a 727,876-chunk vector search index. Every answer passes through a 4-layer validation stack — hallucination guard, answer validator, grounding enforcer, and citation verifier — before being streamed token-by-token to the user's browser. The platform covers 306 providers, 10,349 documents, 459,902 rate facts, and supports 14 full-featured REST API groups for programmatic access. It has served over 2,229 recorded sessions in production and maintains 8 regression sentinels that run continuously to detect data drift.

---

## 2. System Architecture — The 5-Layer Stack

The application is organized into five distinct horizontal layers. Each layer has a single responsibility and communicates only with the layers immediately above and below it.

```
LAYER 5 — USER INTERFACE
  React 18 + TypeScript + TailwindCSS + Vite
  8 pages: Chat, Provider Explorer, Reports, Alerts, Deadlines,
           Compliance, Document Viewer, Status Dashboard
  Built to app/static/ — served as static files by FastAPI

           [HTTP / SSE]

LAYER 4 — APPLICATION GATEWAY (FastAPI)
  app/main.py — 20+ REST endpoints, CORS, rate limiting (120/min)
  PII redaction middleware, NaN-safe JSON, SSE streaming
  Lifespan startup: SQL probe -> LLM probe -> VS probe -> runtime init

           [Python function calls]

LAYER 3 — PLATFORM RUNTIME (platform/v4/)

  NotebookRuntime  (runtime/notebook_entry.py)
    Wires all components: registry, controller, assembler

  AgentController  (core/agent_controller.py)
    ReAct loop: Reason -> Act -> Observe (max 10 steps, $15 budget)
    QuestionRouter -> QuestionDecomposer -> ToolRegistry dispatch

  ToolRegistry  (tools/registry.py) — 25+ registered tools
    sql_query, passage_search, rate_query, clause_existence
    genie_query, citation_verify, risk_alert, financial_analysis
    temporal_analysis, compliance_query, provenance, and more

  Validation Stack
    HallucinationGuard, AnswerValidator, GroundingEnforcer
    CitationValidator, AnswerGroundingEnforcer

           [SDK / REST calls]

LAYER 2 — DATA SERVICES
  SQL Warehouse       | Vector Search Service  | LLM Endpoints
  WarehouseSpark      | VectorSearchService    | Claude Sonnet 4.6
  31 Delta tables     | 727,876 chunks         | Claude Haiku 4.5
  459,902 rates       | idx_unified_v4 index   | (Model Serving)

  Genie Space (ID: 01f14d360bd51f8d801452065b2df600)
  Natural-language SQL over Unity Catalog via Genie API

           [Delta Lake]

LAYER 1 — DATABRICKS INFRASTRUCTURE
  Unity Catalog: dev_adb.raw
  SQL Warehouse ID: 2c99c6485f03ee73
  VS Endpoint: contract_intelligence_vs_endpoint
  Volumes: /Volumes/prod_adb/default/ext-data-volume-stmlz/...
  Databricks Apps: uvicorn app.main:app -- port 8000
```

### The Business Value of Each Layer

| Layer | What It Does | Why It Matters |
|---|---|---|
| Layer 5 (UI) | Presents AI answers, charts, and alerts in a browser | No technical skill needed to access contract intelligence |
| Layer 4 (API) | Routes requests, enforces security, streams responses | All access through one auditable gateway |
| Layer 3 (Runtime) | Runs the AI reasoning loop; selects and calls the right tools | This is the intelligence core |
| Layer 2 (Data Services) | Executes SQL, semantic search, and LLM calls | Milliseconds per query across millions of rows |
| Layer 1 (Infrastructure) | Stores all data, serves compute, auto-scales | Zero ops burden |

### Key Design Decisions

**Fail-open startup.** The frontend and all structured-data endpoints (providers, reports, alerts) become available immediately when the app starts. The AI chat runtime initializes in the background. Users never see a blank screen.

**Single unified vector index.** All 727,876 text chunks live in one index (`idx_unified_v4`). This eliminates multi-index fan-out latency and simplifies retrieval logic.

**Dependency injection throughout.** Every component (LLM client, VS service, SQL session) is injected at startup rather than instantiated inside tools. This makes every component independently testable and replaceable.

**Three independent circuit breakers.** SQL, Vector Search, and LLM each have their own circuit breaker. If one service degrades, the others continue operating. A VS outage does not disable SQL-based answers.

---

## 3. Complete File Map — Every File and Its Role

This section catalogs every code file in the project. Files are grouped by directory and listed with a description of their role and business purpose.

---

### Root-Level Files

| File | Role |
|---|---|
| `app.yaml` | Databricks App manifest. Declares the startup command (uvicorn), environment variable bindings, and 3 resource declarations: SQL warehouse, Claude Sonnet 4.6 endpoint, Claude Haiku 4.5 endpoint. This is the deployment contract with the Databricks Apps runtime. |
| `build_and_deploy.py` | Automated build + deploy script. Bootstraps npm from a tarball, runs `npm ci` + `npm run build` to produce `app/static/`, then calls `WorkspaceClient().apps.deploy()` with `AppDeploymentMode.SNAPSHOT`. Bypasses the Databricks CLI (blocked in USER_ISOLATION mode). |
| `requirements.txt` | Python dependency list. Key packages: fastapi, uvicorn, databricks-sdk, databricks-vectorsearch, pydantic, pandas, slowapi (rate limiting). |
| `conftest.py` | Pytest configuration for the test suite. Sets up path fixtures so all test files can import from the project root. |

---

### `app/` — FastAPI Application Entry Point

| File | Role |
|---|---|
| `app/main.py` | The heart of the backend. FastAPI application object, all 20+ REST endpoint definitions, lifespan context manager (startup/shutdown), CORS middleware, PII redaction middleware, NaN-safe JSON serialization class, rate limiting integration, input sanitization, SSE streaming logic, session persistence wiring, and telemetry wiring. This single file is the gateway that every user request passes through. |
| `app/sql_client.py` | `WarehouseSpark` class — wraps the Databricks Statement Execution API to provide a Spark-like `.sql().collect()` interface against the SQL warehouse. Used by every tool that executes SQL. Handles warehouse startup probing with 4-attempt retry logic. |
| `app/__init__.py` | Package marker. Empty. |

---

### `platform/v4/core/` — The Intelligence Core

| File | Role |
|---|---|
| `agent_controller.py` | The ReAct orchestration engine. Implements the bounded Reason-Act-Observe loop: calls the LLM to select an action, dispatches to ToolRegistry, feeds the observation back, and repeats until SYNTHESIZE or ABORT. Contains 16+ system prompt rules encoding business logic about provider data (DOFR rules, rate query rules, portfolio question patterns, disambiguation rules). This is the most critical file in the application. |
| `question_router.py` | Fast-path dispatcher that pattern-matches incoming questions to direct-dispatch routes, bypassing the full ReAct loop for known question types. Handles provider deep-dives, rate lookups, and portfolio-level queries with lower latency. |
| `question_decomposer.py` | Detects complex multi-hop questions (compare, versus, "both X and Y") and decomposes them into parallel sub-questions with a dependency graph. Uses a fast heuristic regex check first (zero LLM cost), falls back to LLM decomposition only for genuinely complex questions. |
| `hallucination_guard.py` | Post-generation validation. Runs 5 checks on every answer: numeric consistency (numbers in the answer must appear in cited sources), provider identity (provider names must be in session context), date plausibility, ungrounded assertion detection, confidence floor enforcement. Zero LLM cost. |
| `answer_validator.py` | The "Prove It" guardrail. Scans the LLM's answer for absence claims ("does not have", "no data exists"). When found, runs a cheap existence-check SQL against the correct structured table. If data exists, returns `needs_retry=True` with a correction instruction. Prevents false negatives caused by incorrect tool routing. |
| `answer_grounding.py` | (Moved to `services/`) Enforces minimum passage relevance threshold (0.5) before synthesis is allowed. Returns a clear refusal message rather than an ungrounded guess when evidence is insufficient. |
| `context_manager.py` | Session context management. Loads conversation history for multi-turn support. Stores the accumulated `working_data` (tool observations) within a single question's ReAct loop. |
| `cost_controller.py` | Tracks cumulative USD spend within a session. Each tool call has a cost estimate; the controller blocks tool calls when the session budget ($15.00 default) would be exceeded. Prevents runaway LLM loops. |
| `current_effective_filter.py` | Builds dynamic WHERE clauses for "current effective" rate queries — rates where `effective_date <= today AND (superseded_date > today OR superseded_date IS NULL)`. Used by the RetrievalAdapter to scope vector search results to active documents only. |
| `provenance.py` | `ProvenanceResolver` — resolves citation strings to their source documents by querying `tbl_contract_documents_master`. Enables the frontend's DocumentViewer deep-linking feature. |
| `response_assembler.py` | Packages the final answer into a `QueryResponse` object: answer text, citations list (with source filename and page number), confidence label + score, tool call count, and latency. Also surfaces inactive provider notes from `CurrentEffectiveFilter`. |

---

### `platform/v4/tools/` — The Tool Library

Each file is a self-contained tool registered in the `ToolRegistry`. The agent selects and calls these tools during the ReAct loop.

| File | Tool Name | What It Does |
|---|---|---|
| `registry.py` | ToolRegistry | Central registration and dispatch. Applies 6 policy checks before every tool call: enabled, feature flag, budget, call count, input validation, then executes. |
| `base.py` | BaseTool | Abstract base class all tools inherit from. Defines the `run(**kwargs) -> ToolResult` interface. |
| `sql_query.py` | `sql_query` | Natural language to SQL pipeline: builds schema context from `dim_table_schema`, calls LLM to generate SQL, validates against 13-table whitelist, blocks DDL/DML, executes via WarehouseSpark. |
| `passage_search.py` | `passage_search` | Semantic search over the 727,876-chunk unified vector index. Accepts provider_name filter and keyword list. Returns top-20 passages with cosine similarity scores. |
| `rate_query.py` | `rate_query` | Specialized rate lookup tool. Queries `tbl_contract_rates_all` with rate category, date, and amendment filters. Understands status='current' vs. status='superseded' and the amendment_order column. |
| `clause_existence_tool.py` | `clause_existence` | Checks whether a specific clause type (offset, DOFR, IPA delegation) exists for a provider. Queries `tbl_genie_provider_profile` status flags. High-cost tool ($5.50 estimate) due to multi-step LLM reasoning. |
| `genie_query.py` | `genie_query` | Sends natural language questions to the Databricks Genie Space via polling API. Provides an alternative structured-data path with auto-generated SQL. Uses exponential backoff polling (`polling.py`). |
| `temporal_analysis.py` | `temporal_analysis` | Time-series analysis over the amendment timeline table. Answers "how has X changed over time?" questions. Queries `tbl_genie_amendment_timeline`. |
| `multi_concept.py` | `multi_concept` | Evaluates multiple contract concepts (offset, DOFR, auto-renewal, etc.) for one or more providers in a single pass. Used for report generation and portfolio-level classification. High-cost tool ($12 estimate). |
| `comparison.py` | `comparison` | Side-by-side comparison of two providers across any dimension: rates, clauses, LOB, amendment history. |
| `provider_lookup.py` | `provider_lookup` | Resolves ambiguous provider names against `dim_provider_canonical` (531 canonical entries). Returns the canonical name and provider_id needed by all other tools. |
| `provider_deep_dive.py` | `provider_deep_dive` | Comprehensive single-provider profile: rates summary, clause coverage, document inventory, amendment history, financial exposure, risk tier. |
| `citation_verify.py` | `citation_verify` | Uses Claude Haiku 4.5 to verify that a citation text actually supports the claim being made. Runs as a post-synthesis step to catch confident but wrong citations. |
| `summarize.py` | `summarize` | LLM-powered summarization of retrieved passages. Used when a content question requires synthesizing multiple source documents. |
| `draft_text.py` | `draft_text` | Drafts contract language based on retrieved clause examples and user requirements. Uses LLM generation grounded in actual clause text. |
| `document_fetch.py` | `document_fetch` | Retrieves the full text content of a specific contract document from the vector index by source filename. |
| `field_extraction.py` | `field_extraction` | Extracts specific fields (dates, rates, names) from document passages using targeted LLM prompts. |
| `calculate.py` | `calculate` | Performs arithmetic calculations on retrieved numeric data: averages, totals, percentages, year-over-year changes. |
| `chart_generate.py` | `chart_generate` | Generates chart specifications (Vega-Lite JSON) from structured data for display in the frontend. |
| `export_data.py` | `export_data` | Exports structured data (rate tables, provider lists) as CSV, JSON, or Markdown for download. |
| `explanation.py` | `explanation` | Explains contract terms, legal concepts, and abbreviations (DRG, DOFR, capitation, etc.) in plain English. |
| `amendment_chain.py` | `amendment_chain` | Traces the full amendment history for a provider: base agreement through all amendments in sequential order, with effective dates and supersession relationships. |
| `risk_alert.py` | `risk_alert` | Queries risk scores, alert queue, and contract deadlines. Returns priority-sorted risk information from the Phase 10B risk tables. |
| `financial_analysis.py` | `financial_analysis` | Queries financial exposure, repricing scenarios, and rate escalators. Answers "what is our financial exposure to this provider?" questions. |
| `system_membership.py` | `system_membership` | Queries health system membership and geographic service areas. Answers "which health system does this provider belong to?" |
| `compliance_query.py` | `compliance_query` | Queries compliance tracking, quality performance, and contract notifications across 7 regulations. |
| `clause_text.py` | `clause_text` | Retrieves actual clause text from `tbl_contract_clauses` (7,863 rows, 21 categories). Supports 4 query types: clause_text, category_summary, clause_compare, network_norms. |
| `provenance.py` | `provenance` | Queries extraction provenance and supersession history. Answers "where did this data come from?" and "what superseded what?" |

---

### `platform/v4/services/` — Shared Service Layer

| File | Role |
|---|---|
| `vector_search_service.py` | Wraps the Databricks Vector Search SDK. Provides `search()`, `retrieve()`, and `search_legal()` methods over the unified index. Tracks degradation state. Supports provider_name, is_current, and chunk_type filters. |
| `retrieval_service.py` | Orchestrates multi-stage retrieval: vector search + current_effective filter + recall boost. Built via `build_retrieval_service()` factory function. |
| `answer_grounding.py` | `AnswerGroundingEnforcer` — evaluates whether retrieved passages meet the minimum relevance threshold (0.5) to support synthesis. Returns a refusal message when grounding cannot be established. |
| `provider_service.py` | `ProviderService` — loads and caches the canonical provider list from `dim_provider_canonical`. Used by QuestionRouter for provider name resolution. |
| `recall_boost.py` | Augments vector search results by injecting additional structured context (e.g., provider profile facts) when VS recall is low. |
| `telemetry_writer.py` | Fire-and-forget async telemetry. Writes query metadata (latency, cost, confidence, tool calls) to `app_query_telemetry` without blocking the response path. |
| `citation_validator.py` | Validates citation text against source document content. |
| `rate_query_templates.py` | Pre-built SQL template patterns for rate queries — reduces LLM SQL generation latency for common rate lookup patterns. |
| `sql_helpers.py` | `escape_like_pattern()` and `rewrite_superseded_by_join()` utilities used by the SQL generation pipeline to prevent SQL injection and handle supersession joins. |
| `text_utils.py` | `extract_keywords()` and `score_passage()` used by AnswerGroundingEnforcer to compute relevance scores. |
| `tier_service.py` | Risk tier classification logic used by RiskAlertTool. |
| `polling.py` | Exponential backoff polling loop used by GenieQueryTool to wait for Genie API responses. |
| `concepts.py` | Defines the `CONCEPT_REGISTRY` — the 6 contract concepts (offset, DOFR, IPA delegation, AB352, encounters, auto renewal) exposed by the `/api/v1/concepts` endpoint and the Report generation system. |

---

### `platform/v4/runtime/` — Runtime Initialization

| File | Role |
|---|---|
| `notebook_entry.py` | The runtime factory. Exports `init_runtime()`, `init_runtime_standalone()`, and `init_runtime_api()`. `NotebookRuntime` class wires all adapters, tools, the registry, and the AgentController together into a single `.ask()` and `.stream_ask()` interface. |
| `databricks_llm_client.py` | `DatabricksLLMClient` — wraps Databricks Model Serving REST API with SDK-native OAuth auth and automatic token rotation. Provides `chat()` (single-turn) and `chat_multi()` (multi-turn) methods. Used by the agent, SQL generator, decomposer, and citation verifier. |
| `async_bridge.py` | Bridges the synchronous tool execution world (tools use blocking SQL calls) with FastAPI's async event loop. Runs blocking operations in a `ThreadPoolExecutor` via `asyncio.to_thread`. |
| `notebook_stream_handler.py` | Handles the token-by-token SSE streaming from the ReAct loop to the FastAPI StreamingResponse. Converts agent stream events into NDJSON lines. |
| `api_entry.py` | Alternative entry point for the API context (vs. notebook context). |

---

### `platform/v4/data/` — Persistence Repositories

| File | Role |
|---|---|
| `session_repo.py` | `SessionRepo` — reads/writes conversation turns to `tbl_agent_sessions`. Enables session history sidebar and multi-turn context loading. |
| `feedback_repo.py` | `FeedbackRepo` — writes user thumbs-up/down ratings and corrections to `tbl_user_feedback`. Feeds the quality improvement loop. |
| `telemetry_repo.py` | `TelemetryRepo` — writes retrieval events to `fact_retrieval_telemetry`. Tracks which chunks were retrieved, their quality scores, and whether they were cited. |
| `query_cache_repo.py` | `QueryCacheRepo` — reads/writes cached answers to `tbl_query_cache`. Identical questions within the cache TTL return instantly without re-running the agent. |
| `schema_context_builder.py` | `SchemaContextBuilder` — reads column names, types, and descriptions from `dim_table_schema` to build the schema context injected into the SQL generation prompt. This is how the LLM knows what columns exist in each table. |

---

### `platform/v4/config/` — Configuration

| File | Role |
|---|---|
| `settings.py` | All runtime constants: LLM model names, table paths (31 fully-qualified table names), agent limits (max steps, budget, timeouts), tool cost estimates, feature flags. All values are env-var overridable for dev/staging/prod routing. |
| `table_whitelist.py` | `CONTRACT_TABLES_FOR_SQL` — the 13 tables the SQL generation tool is allowed to query. Any table not on this list is blocked at execution time. This is the primary SQL injection prevention layer. |
| `workspace_config.py` | `WorkspaceConfig` — resolves workspace URL, API token, VS endpoint, and warehouse ID from the Databricks SDK or environment variables. Handles both notebook context and Databricks Apps context. |
| `tool_policies.py` | Per-tool policy objects: enabled flag, required feature flag, max cost estimate, max call count per session. Called by ToolRegistry before every tool dispatch. |

---

### `platform/v4/adapters/` — Legacy Compatibility Layer

| File | Role |
|---|---|
| `legacy_branch_adapter.py` | Wraps the V2 dispatch function for backward-compatible access to legacy extraction capabilities. Marked deprecated — all new tools use native V4 paths. |
| `retrieval_adapter.py` | Abstracts retrieval: routes to either the native `RetrievalService` (V4 path) or legacy `retrieve_fn` (V2 path). Also applies `CurrentEffectiveFilter` on the legacy path. |
| `citation_adapter.py` | Wraps the legacy citation verification function for backward compatibility. |
| `provider_adapter.py` | Queries provider data from `dim_provider_canonical` and `tbl_genie_provider_profile`. Used by the provider deep-dive tool. |

---

### `platform/v4/presentation/` — Report and Export Layer

| File | Role |
|---|---|
| `report_builder.py` | Builds the classification matrix report: queries `tbl_genie_provider_profile` clause status columns, applies provider/LOB/active filters, returns a matrix of HAS/NO_MENTION per concept per provider. |
| `report_enrichment.py` | Enriches report rows with additional context (LOB, payment type, amendment count). |
| `html_renderer.py` | Renders report data to styled HTML for the ReportPage frontend display. |
| `excel_export.py` | Exports report data as an Excel workbook with formatted headers and color-coded status cells. Used by the `/api/v1/export` endpoint. |
| `report_validator.py` | Validates report inputs and catches unsupported concept keys before query execution. |

---

### `platform/v4/contracts/` — Type Definitions

| File | Role |
|---|---|
| `agent_types.py` | `AgentRequest`, `AgentResponse`, `AgentStep`, `StepAction` — the typed contracts between the API layer and the agent. |
| `tool_types.py` | `ToolResult`, `ToolStatus`, `Confidence` — every tool returns a `ToolResult`; the agent reads `.status` and `.data`. |
| `streaming_types.py` | `StreamEvent`, `StreamMessage`, `StreamCallback` — the typed contracts for SSE streaming. |
| `context_types.py` | `SessionContext` — the accumulating context object passed through the ReAct loop, containing working_data and retrieved passages. |
| `decomposition_types.py` | `DecompositionPlan`, `SubQuestion`, `ComplexityLevel`, `SubQuestionType` — types for the question decomposer. |

---

### `frontend/src/` — React Application

| File/Directory | Role |
|---|---|
| `main.tsx` | React app entry point. Mounts the root component with `ReactDOM.createRoot()`. |
| `App.tsx` | Root component. Defines the React Router routes mapping URLs to page components. |
| `pages/ChatPageV2.tsx` | The AI chat interface. Streaming question input, message history, citation cards, feedback controls, session sidebar, export/share controls. The primary user-facing feature. |
| `pages/ProviderExplorerV2.tsx` | Searchable provider catalog. Debounced search, filter chips (LOB, payment type, active), paginated table, slide-over detail panel. |
| `pages/ReportPageV2.tsx` | Contract concept classification report generator. Concept selector cards, provider/LOB/active filters, classification matrix display. |
| `pages/AlertsPageV2.tsx` | Live risk alert dashboard. Sorted by priority (CRITICAL first), with provider, alert type, and days-until-due columns. |
| `pages/DeadlineCalendarV2.tsx` | Calendar view of 713 contract deadline events. Color-coded by deadline type and urgency. |
| `pages/CompliancePageV2.tsx` | Compliance status dashboard across 7 regulations. Color-coded COMPLIANT / PARTIAL / UNKNOWN / EXEMPT breakdown. |
| `pages/DocumentViewerV2.tsx` | In-app PDF viewer with `#page=N` deep-linking, snippet context bar, and navigation controls. |
| `pages/StatusPageV2.tsx` | Operational status dashboard: system health banner, 4 KPI cards, sparkline trend charts, circuit breaker panel, data freshness table. Auto-refreshes every 30 seconds. |
| `services/api.ts` | All API calls in one place. Typed functions for every backend endpoint. Handles SSE streaming connection lifecycle. |
| `hooks/useStreamQueryV2.ts` | React hook that manages the full SSE streaming lifecycle: opens EventSource, accumulates tokens, handles `done` and `error` events, exposes `isStreaming` state. |
| `types/api.ts` | TypeScript type definitions mirroring the backend Pydantic models: `QueryResponse`, `Session`, `Message`, `StreamEvent`, `StatusMetrics`, etc. |
| `styles/enterprise.css` | Custom enterprise CSS extending TailwindCSS defaults. Blue Shield brand colors, confidence badge styles, citation card styles. |

---

### `pipeline/` — Data Ingestion Pipeline

| File | Role |
|---|---|
| `00_config.py` | Pipeline-wide configuration: catalog, schema, volume paths, OCR settings, extraction model names. |
| `01_execute_ddl.py` | Creates all 31 Delta tables via DDL execution. The schema definition for the entire data model. |
| `02_state_engine.py` | Manages document processing state: tracks which PDFs have been processed, enables incremental re-runs, handles retry logic for failed documents. |
| `03_reimbursement_migration.py` | Migrates legacy reimbursement data formats into the unified `tbl_contract_rates_all` schema. |
| `04_legal_chunking.py` | Splits contract PDFs into semantic chunks: raw OCR passages (Layer 1) and structured legal units (Layer 2: clauses, definitions, rate tables). Uses `src/chunking.py`. |
| `05_retrieval_engine.py` | Builds and populates `tbl_vs_unified_ready` (727,876 rows) and syncs to the vector search index. |
| `07_citation_verification.py` | Post-ingestion citation quality check. Verifies that extracted clause text is properly attributed to source documents. |
| `08_intelligence.py` | Runs intelligence extraction: populates `tbl_genie_provider_profile` status flags (offset_clause_status, dofr_status, ipa_delegation_status), computes risk scores, and runs amendment chain analysis. |
| `11_evaluation_harness.py` | Automated evaluation harness. Runs a question set against the application and scores answers using LLM-as-judge. |

---

### `src/` — Shared Utilities

| File | Role |
|---|---|
| `chunking.py` | Legal document chunking strategy. Splits PDFs at section boundaries, preserves header context, produces both raw passage chunks (avg ~200 tokens) and structured legal unit chunks (avg ~400 tokens). |
| `validation.py` | Field validation utilities for extracted contract data. Validates date formats, rate numeric ranges, LOB values, and provider name formats before writing to Delta tables. |
| `json_repair.py` | Repairs malformed JSON returned by LLM extraction calls. Handles common LLM formatting errors: trailing commas, unquoted keys, truncated objects. |

---

## 4. The Frontend — What Users Experience

The frontend is a React 18 + TypeScript single-page application built with Vite and styled with TailwindCSS. It compiles to static files in `app/static/` and is served directly by FastAPI — no separate frontend server exists in production.

### Technology Stack

| Technology | Role |
|---|---|
| React 18 | UI component framework with concurrent rendering |
| TypeScript | Type-safe development — API types mirror backend Pydantic models exactly |
| Vite | Build tool. `npm run build` compiles to `app/static/` in ~15 seconds |
| TailwindCSS | Utility-first CSS with enterprise overrides in `enterprise.css` |
| React Router | Client-side routing — 8 URL routes mapped to 8 page components |
| SSE / ReadableStream | Token-by-token streaming from the AI agent via `useStreamQueryV2` hook |

### Page 1: ChatPageV2 — The AI Chat Interface

The primary interface. A two-panel layout: left panel is a session history sidebar listing past conversations; right panel is the main chat area.

**Streaming:** The `useStreamQueryV2` hook sends `POST /api/v1/query` with `stream=true`. Each NDJSON line that arrives is either a `token` event (appended to the current message) or a `done` event (delivers confidence, citations, session_id, cost, latency and closes the stream).

**Features on every AI message:**
- Citation cards (clickable, navigates to DocumentViewerV2 with exact page and snippet)
- Confidence badge: HIGH (green) / MODERATE (yellow) / LOW (orange) / UNCERTAIN (red) + numeric score
- ThumbsUp / ThumbsDown feedback buttons — thumbs-down opens a correction text field
- Provider context field — pre-scopes all questions to a specific provider

**Export controls (top-right, visible when messages exist):** Copy as Markdown, Share Link (copies `/?session={id}` URL), Export dropdown (markdown/json/csv via `/api/v1/export`).

### Page 2: ProviderExplorerV2 — Provider Catalog

Searchable, filterable table of all 306 network providers. Debounced search (300ms), filter chips (LOB, payment type, active/inactive), pagination at 25/page.

Clicking a row opens a **slide-over detail panel** without page navigation, showing:
- Profile card (contract term, auto-renewal, payment type, risk tier)
- Clause coverage badges (HAS/NO_MENTION for offset, DOFR, IPA delegation)
- Top-30 rates table sorted by rate_numeric descending
- Chronological amendment timeline
- Document inventory with role badges (base / amendment / exhibit / addendum)
- "Ask about this provider" button — navigates to ChatPage with provider pre-filled

### Page 3: ReportPageV2 — Classification Matrix Generator

Generates a cross-provider classification matrix for up to 6 contract concepts. User selects concepts, optionally filters by provider/LOB/active, clicks Generate. Output: summary cards showing network-wide % coverage per concept, plus a scrollable matrix table with color-coded HAS/NO_MENTION cells per provider per concept. Exportable as Excel.

### Page 4: AlertsPageV2 — Risk Alert Dashboard

Displays the 351 alerts from `tbl_alert_queue`, sorted by priority. 229 are CRITICAL (red-highlighted). Columns: provider, alert type, priority, days until due, description. Replaces manual contract expiry monitoring entirely.

### Page 5: DeadlineCalendarV2 — Contract Deadline View

713 contract deadlines from `tbl_contract_deadlines` in a month-by-month calendar. Each deadline is a color-coded chip: renewal (blue), rate effective date (green), compliance reporting (orange), termination notice window (red). Clicking shows provider name, deadline type, and required action.

### Page 6: CompliancePageV2 — Regulatory Compliance Dashboard

Compliance status across 7 regulations (KNOX_KEENE, CMS_MA, DMHC_STANDARDS, AB_1455, SB_137, AB_352, AB_72) for 2,871 tracked rows. Stacked horizontal bars showing COMPLIANT (663) / PARTIAL (1,231) / UNKNOWN (974) / EXEMPT (3) per regulation. Expanding a regulation shows per-provider status.

### Page 7: DocumentViewerV2 — In-App PDF Viewer

Receives `?doc=<filename>&page=<N>&snippet=<text>` from citation card links. Calls `/api/v1/documents/{filename}/pdf` to stream the PDF, embeds it in an `<iframe>` with `#page=N` hash deep-linking, and shows the cited snippet in a context bar above. Contract analysts can verify every AI citation without needing filesystem access.

### Page 8: StatusPageV2 — Operational Dashboard

Live platform health view, auto-refreshing every 30 seconds:
- Status banner (green / amber / red with pulsing dot)
- 4 KPI cards: 7-day query count, average latency (ms), error rate (%), total cost (USD)
- SVG sparkline trend charts for latency, volume, errors, cost over 7 days
- Dependency status indicators (SQL warehouse, vector search, LLM, Genie Space)
- Circuit breaker panel (CLOSED / OPEN / HALF_OPEN state per breaker)
- Confidence distribution horizontal bar chart
- Data freshness table (last-modified timestamp + row count per major table)
- Most-queried providers list (top 10 by 7-day query volume)

### The Streaming Hook: `useStreamQueryV2`

1. Sends `POST /api/v1/query` with `stream=true` using the `fetch` API
2. Reads the response body as a `ReadableStream`, processing line by line
3. `{"type":"token","content":"..."}` → appended character by character to current message
4. `{"type":"done","answer":"...","citations":[...]}` → delivers final metadata, closes stream
5. `{"type":"error","message":"..."}` → surfaces error to user
6. Exposes React state: `isStreaming`, `currentAnswer`, `citations`, `confidence`, `latencyMs`

---

## 5. The FastAPI Backend — The Application Gateway

`app/main.py` is the complete application gateway — 700+ lines, handling every HTTP interaction between the frontend and the platform runtime.

### Startup Sequence (Lifespan)

When uvicorn starts, FastAPI's async lifespan context manager runs before any request is served:

```
Step 1: FastAPI yields IMMEDIATELY
        -> app/static/ (React frontend) is served right away
        -> /health responds to probes
        -> Background async task launched

Step 2: Background init (max 300s timeout)
  2a. Three circuit breakers created (SQL / VS / LLM)
  2b. WarehouseSpark initialized + 4-attempt SQL probe (SELECT 1)
  2c. DatabricksLLMClient initialized + LLM endpoint probe (3-token ping)
  2d. VectorSearchService initialized + 3-attempt VS search probe
  2e. platform.v4 module resolution fix applied
      (evicts stdlib 'platform' module, loads our platform/__init__.py)
  2f. GenieClient initialized (workspace URL + API token from SDK)
  2g. NotebookRuntime assembled:
        Settings -> CurrentEffectiveFilter -> RetrievalService
        TelemetryRepo -> ProvenanceResolver -> ProviderService
        -> init_runtime() called

Step 3: app.state.runtime_ready = True
        -> /api/v1/query (AI chat) becomes fully operational
```

**Fail-open behavior:** If VS fails to initialize, SQL-based answers still work. If the full runtime fails, all structured endpoints (providers, alerts, reports) still function. The startup_error is surfaced at `/api/v1/startup-status` and in the health response.

### The AI Chat Endpoint

`POST /api/v1/query` is the most important endpoint. It accepts a `QueryRequest` and returns either a streaming SSE response or a synchronous JSON response.

```
QueryRequest:
  question:      str   -- the user's natural language question
  provider_name: str   -- optional: pre-scopes all tools to this provider
  stream:        bool  -- true = SSE (default), false = synchronous JSON
  session_id:    str   -- conversation session ID for multi-turn context

QueryResponse:
  answer:           str
  confidence:       str   -- HIGH / MODERATE / LOW / UNCERTAIN
  confidence_score: float -- 0.0 to 1.0
  citations:        list[dict]  -- source_filename, page_number, snippet
  session_id:       str
  latency_ms:       int
  tool_calls:       int
  cost_usd:         float
```

For streaming, the endpoint returns a `StreamingResponse` that yields NDJSON lines until the agent finishes. Fire-and-forget background tasks then persist the turn and write telemetry.

### All Endpoints

**Health and Status**

| Endpoint | Returns |
|---|---|
| `GET /health` | SQL/VS/LLM status, circuit breaker states, version, uptime |
| `GET /api/v1/startup-status` | `runtime_ready`, `sql_only` flag, `startup_error` detail |
| `GET /api/v1/status/metrics` | 7-day telemetry aggregates, sparkline data, data freshness |

**Chat**

| Endpoint | Returns |
|---|---|
| `POST /api/v1/query` | AI answer (SSE stream or JSON). Rate-limited 120/min. |

**Session Management**

| Endpoint | Returns |
|---|---|
| `POST /api/v1/sessions` | Create new session, returns session_id |
| `GET /api/v1/sessions` | List recent sessions for current user |
| `GET /api/v1/sessions/{id}` | Full session history (all turns) |
| `PATCH /api/v1/sessions/{id}` | Update session title/notes |

**Feedback**

| Endpoint | Returns |
|---|---|
| `POST /api/v1/feedback` | Records thumbs-up/down + correction text to `tbl_user_feedback` |

**Provider Explorer**

| Endpoint | Returns |
|---|---|
| `GET /api/v1/providers` | Paginated provider list. Params: search, lob, payment_type, active_only |
| `GET /api/v1/providers/{name}` | Full detail: profile + top-30 rates + top-50 docs + amendments + clause coverage |

**Reports**

| Endpoint | Returns |
|---|---|
| `GET /api/v1/concepts` | 6 registered contract concepts with names and descriptions |
| `POST /api/v1/report` | Classification matrix (HAS/NO_MENTION per concept per provider) |

**Documents and Export**

| Endpoint | Returns |
|---|---|
| `GET /api/v1/documents/{filename}/pdf` | Streams PDF from UC Volume with path-traversal protection |
| `GET /api/v1/documents/lookup` | Resolves citation string to document metadata |
| `GET /api/v1/export/{session_id}` | Session export in `?format=markdown\|json\|csv` |

**Phase 10B Specialized Endpoints (20+ endpoints)**

| Group | Endpoints |
|---|---|
| Risk | `/risk/alerts`, `/risk/alerts/{provider}`, `/risk/scores`, `/risk/deadlines`, `/risk/deadlines/{provider}` |
| Financial | `/financial/exposure`, `/financial/exposure/{provider}`, `/financial/repricing/{provider}`, `/financial/rates/{provider}/history`, `/financial/rates/{provider}/escalators` |
| Network | `/network/systems`, `/network/systems/{name}`, `/network/{provider}/membership`, `/network/{provider}/service-area` |
| Compliance | `/compliance/summary`, `/compliance/{provider}`, `/quality/{provider}`, `/notifications/{provider}` |
| Clauses | `/clauses/{provider}`, `/clauses/{provider}/summary`, `/clauses/network/{category}`, `/clauses/{provider}/deviations` |
| Provenance | `/provenance/summary`, `/provenance/{provider}`, `/provenance/document/{filename}`, `/provenance/quality` |

### Security Middleware Applied to Every Request

1. **Rate limiting** — `slowapi` enforces 120 requests/minute per IP. HTTP 429 on excess.
2. **CORS** — configured for the Databricks Apps domain. Other origins rejected.
3. **Input sanitization** — `_sanitize_like_input()` truncates to 100 chars, strips non-alphanumeric characters, escapes SQL wildcards before LIKE injection.
4. **PII redaction** — `redact_pii()` strips SSN patterns, DOB patterns, and street address patterns from all text before LLM injection.
5. **Path traversal protection** — PDF serving validates the resolved path is within the allowed volume directory.
6. **DDL/DML block** — SQL tool rejects any query containing CREATE, INSERT, UPDATE, DELETE, DROP, ALTER, TRUNCATE, GRANT, REVOKE, or MERGE.
7. **Table whitelist** — SQL tool only executes against 13 approved tables.

### NaN-Safe JSON

`NaNSafeJSONResponse` wraps all API responses. It recursively converts `float('nan')` and `float('inf')` to `null` before serialization. Without this, any NULL numeric value that passed through `.toPandas()` would produce a `ValueError` and crash the response. This prevents silent 500 errors on rate queries.

---

## 6. What Happens When You Ask a Question

This section traces the complete lifecycle of a single user question — from the moment a key is pressed to the moment the answer appears on screen. This is the most important section for understanding how the system works.

**Example question:** *"What are the current inpatient DRG rates for Sutter Health?"*

---

### Step 1: The User Types and Submits

The user types their question in the input bar on `ChatPageV2`. They optionally type a provider name in the provider context field. They press Enter or click Send.

The React component calls `useStreamQueryV2.submit()` which fires:
```
POST /api/v1/query
Body: {
  "question": "What are the current inpatient DRG rates for Sutter Health?",
  "provider_name": "",
  "stream": true,
  "session_id": "sess-abc123"
}
```
The browser opens a `ReadableStream` on the response body and begins reading NDJSON lines. The UI immediately shows a typing indicator.

---

### Step 2: FastAPI Receives the Request

`app/main.py` receives the request at the `/api/v1/query` handler.

**Security wall (executed in this order):**
1. Rate limiter checks: is this IP under 120 requests/min? If not → HTTP 429, stop.
2. `_sanitize_like_input()` cleans the provider_name field (truncate, strip non-alphanumeric, escape wildcards).
3. `redact_pii()` scans the question text for SSN, DOB, and address patterns and replaces them.

**Runtime gate:** The handler checks `app.state.runtime_ready`. If False (runtime still initializing), it checks `app.state.sql_only`. In sql_only mode it returns a message: *"Chat is temporarily unavailable while the AI runtime initializes."*

If `runtime_ready = True`, execution continues.

**Streaming response opened:** A `StreamingResponse` with `media_type="text/event-stream"` is returned immediately. This opens the SSE channel to the browser. The actual answer generation runs inside the async generator that feeds this response.

---

### Step 3: Session Context Loaded

Before the agent runs, the session history is loaded:

1. If `session_id` is non-empty, `SessionRepo.get_turns(session_id)` queries `tbl_agent_sessions` for the last N turns of this conversation.
2. The previous Q&A pairs are formatted as `[{"role":"user","content":...}, {"role":"assistant","content":...}]` pairs.
3. These become the `history` field in the `AgentRequest` object passed to the runtime.

This is how multi-turn context works — the agent "remembers" previous turns by reading them from the database at the start of each new question.

---

### Step 4: Query Cache Check

Before running the agent, `QueryCacheRepo.lookup(question_hash)` checks `tbl_query_cache`.

If a cache hit is found (same question, same provider scope, within TTL):
- The cached `QueryResponse` is returned immediately, bypassing the entire agent loop
- Latency drops from ~5 seconds to ~200ms
- The stream sends the cached answer as a single `done` event

If no cache hit: continue to agent execution.

---

### Step 5: The Question Router — Fast Path or Full ReAct?

`AgentController.run()` first calls `QuestionRouter.route(question, provider_name)`.

The router applies pattern matching against the question text. If the question matches a known high-confidence pattern, it dispatches directly to the right tool without starting the full ReAct loop:

| Pattern detected | Direct dispatch |
|---|---|
| Rate-related question + specific provider | `rate_query` tool directly |
| Provider name explicit + "deep dive" intent | `provider_deep_dive` tool directly |
| Portfolio-level enumeration pattern | `sql_query` tool with network-wide scope |
| Show rates / list rates pattern | `rate_query` with show_entries=True |

**For our Sutter Health DRG example:** The router detects a rate-related question with a specific provider name. It triggers direct dispatch to `rate_query`. The full ReAct loop may be abbreviated or skipped.

If the question does NOT match any fast-path pattern, execution falls through to the full ReAct loop.

---

### Step 6: Question Decomposition Check

`AgentController._maybe_decompose()` calls `QuestionDecomposer.analyze(question)`.

**Heuristic check first (zero LLM cost):** The decomposer scans for conjunction patterns (`both X and Y`, `compare X versus Y`, `calculate the difference`), multi-provider mentions, and calculation keywords.

- **Simple question** (like our DRG example) → `plan.is_simple = True` → skip decomposition, proceed with single ReAct loop.
- **Complex question** (e.g., *"Compare Sutter Health's DRG rates to Sharp Healthcare AND show me which one has more amendments"*) → LLM called to produce a `DecompositionPlan` with two sub-questions in separate execution groups → each sub-question runs its own ReAct loop → results synthesized together.

---

### Step 7: The ReAct Loop Begins

The `AgentController` initializes the loop state:
- `step_count = 0`, `cumulative_cost = 0.0`
- `working_data = {}` — accumulates all tool observations
- `retrieved_passages = []` — accumulates all semantic search results

The system prompt is assembled:
```
[16 pages of business rules encoding provider data logic]
+ [Tool catalogue: name, description, and parameters for every registered tool]
+ [Session history: previous turns from tbl_agent_sessions]
+ [Current working_data: observations from this question's tool calls so far]
+ [Provider context: if provider_name was specified]
```

The full system prompt is sent to Claude Sonnet 4.6 with the user's question.

---

### Step 8: LLM Reasoning Step — Selecting the First Action

The LLM receives the system prompt + question. It must return valid JSON:

```json
{
  "thought": "The user is asking about current DRG rates for Sutter Health. I should use rate_query to retrieve the current rates from tbl_contract_rates_all.",
  "action": "CALL_TOOL",
  "tool_name": "rate_query",
  "tool_input": {
    "provider_name": "Sutter Health",
    "rate_category": "drg",
    "status_filter": "current"
  }
}
```

Four actions are available:
- `THINK` — additional reasoning step, no tool call fired, costs one reasoning step
- `CALL_TOOL` — dispatches to the named tool with the provided parameters
- `SYNTHESIZE` — generates the final answer using all accumulated evidence
- `ABORT` — terminates with a "cannot answer" message (used when question is out of scope)

If the LLM returns malformed JSON: `src/json_repair.py` attempts to fix it. If repair fails, the step is retried once.

---

### Step 9: ToolRegistry Dispatch — Policy Checks

Before any tool executes, `ToolRegistry.call("rate_query", ...)` applies 6 policy checks in order:

1. **Is the tool registered?** If not → `ToolResult.error("not registered")`
2. **Is the tool enabled?** `tool_policies.py` has per-tool `enabled` flag. Disabled tools return `ToolResult.skipped()`
3. **Is the required feature flag active?** e.g., `GENIE_ENABLED` must be True for `genie_query`
4. **Does the session have enough budget?** `remaining_budget >= tool.max_cost_usd`. Blocks if not.
5. **Has the per-session call limit been reached?** e.g., `passage_search` is limited to 5 calls per session
6. **Does the tool's own input validation pass?** Each tool validates its own required parameters

All 6 checks pass → tool executes.

---

### Step 10: The Tool Executes

**For `rate_query`:**

1. Looks up the canonical provider name via `dim_provider_canonical` (handles fuzzy matching: "Sutter Health" → "Sutter Health Sacramento Sierra Region")
2. Queries `tbl_contract_rates_all`:
   ```sql
   SELECT rate_category, rate_numeric, rate_unit, rate_type,
          effective_date, amendment_order, status
   FROM dev_adb.raw.tbl_contract_rates_all
   WHERE LOWER(provider_name) LIKE '%sutter%'
     AND rate_category LIKE '%drg%'
     AND status = 'current'
   ORDER BY rate_numeric DESC
   LIMIT 50
   ```
3. Returns a `ToolResult` with `status=SUCCESS`, `data=[list of rate rows]`, `confidence=HIGH`

**For `sql_query` (used when `rate_query` is not dispatched):**

1. `SchemaContextBuilder` reads column names and descriptions from `dim_table_schema` for the relevant tables
2. Builds a SQL generation prompt: schema context + business rules + the user's question
3. Calls Claude Sonnet 4.6 with `max_tokens=800` to generate SQL
4. Extracts the SQL block (strips markdown fences)
5. Validates: all table references must be in `CONTRACT_TABLES_FOR_SQL` (13 tables); no DDL/DML keywords
6. Executes via `WarehouseSpark.sql(generated_sql).collect()`
7. Returns rows as `ToolResult.data`

**For `passage_search` (when semantic retrieval is needed):**

1. Calls `VectorSearchService.search(keywords, provider_name=..., top_k=20)`
2. The VS SDK sends the query to `idx_unified_v4` (727,876 chunks with GTE-1024d embeddings)
3. Returns top-20 passages ranked by cosine similarity score
4. `CurrentEffectiveFilter` may be applied to prefer passages from active documents
5. Passages are stored in `retrieved_passages` for grounding checks

---

### Step 11: Observation Fed Back — Loop Continues

The `ToolResult` is converted to an observation string and appended to `working_data`:

```
working_data["rate_query_1"] = {
  "rows": [{"rate_category": "drg", "rate_numeric": 12500.0,
             "rate_type": "base_rate", "status": "current",
             "amendment_order": 4}, ...],
  "row_count": 23,
  "confidence": "HIGH"
}
```

The updated `working_data` is appended to the system prompt context and the LLM is called again for the next reasoning step.

**The loop continues** until one of these exit conditions is met:
- LLM action = `SYNTHESIZE` — agent has enough evidence
- LLM action = `ABORT` — question is out of scope
- `step_count >= AGENT_MAX_STEPS` (10) — budget of steps exhausted
- `cumulative_cost >= AGENT_DEFAULT_BUDGET_USD` ($15.00) — spend cap reached
- `AGENT_TOTAL_TIMEOUT_S` (600 seconds) elapsed

For our Sutter Health DRG question, the agent typically completes in 2–3 steps: one tool call (rate_query) + one synthesis step.

---

### Step 12: LLM Synthesizes the Final Answer

When the LLM returns `action: "SYNTHESIZE"`, it generates the final answer in `final_answer`:

```json
{
  "thought": "I have 23 current DRG rates for Sutter Health across amendment 4.
               I can now provide a complete answer with the rates.",
  "action": "SYNTHESIZE",
  "final_answer": "Sutter Health currently has 23 active DRG-based rates...
                   [detailed answer with specific rate values and citations]"
}
```

System prompt rules enforced during synthesis:
- **Never fabricate** contract terms, rates, or dates
- **Never state specific numbers from memory** — must cite from tool observations
- **Never deflect to the PDF** — all content must be presented in the answer itself
- **List all data rows** returned by tools — never summarize as just a count
- **Cite source documents** with filename and page number when available

---

### Step 13: The 4-Layer Validation Stack

Before the answer is returned, it passes through four independent validation layers:

#### Layer 1: HallucinationGuard

`HallucinationGuard.check(answer, working_data, retrieved_passages)` runs 5 checks at zero LLM cost:

1. **Numeric consistency:** Every number in the answer (rates, counts, percentages) must appear in `working_data`. Ungrounded numbers → `GuardFinding(severity="WARN")`.
2. **Provider identity:** Provider names mentioned in the answer must match providers resolved in the session context. Unknown names → warning.
3. **Date plausibility:** Dates must fall within a reasonable range (not future decades, not before 1990). Implausible dates → warning.
4. **Ungrounded assertion detection:** Sentences with high specificity (numbers, proper nouns, legal terms) but zero citation support are flagged.
5. **Confidence floor:** If `retrieved_passages` is empty AND `working_data` has no SQL results, confidence is capped at MODERATE regardless of LLM's claim.

Findings with `severity="BLOCK"` prevent the answer from being returned (rare — only for clearly fabricated critical claims).
Findings with `severity="WARN"` are logged but do not block the answer.

#### Layer 2: AnswerValidator ("Prove It" Guardrail)

`AnswerValidator.validate(answer, question, spark)` scans for absence claims:

Absence patterns include: *"does not have"*, *"no data exists"*, *"not available"*, *"no traditional DOFR"*, *"zero rows"*, *"unable to find"*.

If an absence claim is detected:
1. Determine the correct table for the domain (e.g., DOFR claims → check `tbl_dofr_matrix_extracted`)
2. Run a cheap existence check: `SELECT 1 FROM tbl WHERE ... LIMIT 1`
3. If data EXISTS → `ValidationResult(needs_retry=True, correction="Check tbl_X for this data")`
4. The agent retries with the correction injected into the system prompt
5. If data does NOT exist → absence claim is valid, pass through

This layer prevents the most common class of wrong answers: the agent says "no data found" when data is in a different table it didn't check.

#### Layer 3: AnswerGroundingEnforcer

`AnswerGroundingEnforcer.evaluate(tool_results, passages, query)` checks whether the answer has sufficient evidentiary support:

1. `score_passage(passage, query_keywords)` computes a relevance score (0.0–1.0) for each retrieved passage
2. If the maximum passage relevance score across all retrieved passages is below 0.5:
   - AND the tool_results contain no structured data rows
   - → return `REFUSAL_MESSAGE`: *"I could not find verified evidence for this question in the contract corpus..."*
3. Otherwise → pass through

This prevents the agent from guessing when evidence is genuinely absent.

#### Layer 4: CitationValidator

For each citation in the answer, `citation_validator.verify(citation_text, source_passage)` uses Claude Haiku 4.5 (the fast, cheap model) to confirm that the cited text actually supports the claim being made.

Citations that fail verification are:
- Removed from the citations list
- The answer's confidence score is reduced accordingly

---

### Step 14: ResponseAssembler Packages the Answer

`ResponseAssembler.build(agent_response, session_context)` creates the final `QueryResponse`:

1. **Confidence label + score:** Maps the validated confidence enum to a display string and numeric score:
   - HIGH → 0.90, MODERATE → 0.65, LOW → 0.35, UNCERTAIN → 0.15
2. **Citations list:** Each citation gets: `source_filename`, `page_number_est`, `snippet` (first 200 chars of the cited passage)
3. **Inactive provider note:** If the queried provider has `is_active=FALSE` in `tbl_genie_provider_profile`, a note is appended: *"Note: This provider's contract document has expired. Rates shown are from their last active document."
4. **Metadata:** `latency_ms`, `tool_calls`, `cost_usd` (sum of cost estimates for all tool calls)

---

### Step 15: Streaming Tokens to the Browser

The `NotebookStreamHandler` converts the assembled response into NDJSON events:

```
{"type": "token", "content": "Sutter Health "}
{"type": "token", "content": "currently has "}
{"type": "token", "content": "23 active DRG..."}
... [one event per ~5-10 tokens]
{"type": "done",
  "answer": "Sutter Health currently has 23 active DRG...",
  "confidence": "HIGH",
  "confidence_score": 0.90,
  "citations": [{"source_filename": "Sutter_2024_Amend4.pdf",
                  "page_number_est": 12,
                  "snippet": "DRG Base Rate: $12,500..."}],
  "session_id": "sess-abc123",
  "latency_ms": 4823,
  "tool_calls": 2,
  "cost_usd": 0.0043}
```

The browser's `useStreamQueryV2` hook reads each line, appends `token` content to the displayed message, then on `done` renders the citations, confidence badge, and metadata.

---

### Step 16: Fire-and-Forget Background Tasks

After the stream closes, two background tasks run without blocking any further response:

**Session persistence (`_persist_turn()`):**
- Writes to `tbl_agent_sessions`: question, answer, confidence, citations, latency, tool_calls, cost, session_id, timestamp
- The session sidebar on the next page load will show this conversation

**Telemetry writing (`TelemetryWriter.write()`):**
- Writes to `app_query_telemetry`: all metadata fields
- Used by the StatusPage's 7-day trend charts and KPI cards
- Also writes retrieval events to `fact_retrieval_telemetry` (which chunks were used, their quality scores)

---

### Complete Timeline (Typical Question)

```
0ms      User presses Enter -> POST /api/v1/query
5ms      Rate limit check passes -> SSE stream opens -> UI shows typing indicator
10ms     Session history loaded from tbl_agent_sessions
15ms     Cache miss confirmed
20ms     QuestionRouter: fast-path match found (rate question + provider name)
25ms     ReAct loop begins -> LLM called (Claude Sonnet 4.6)
800ms    LLM returns {thought, action:CALL_TOOL, tool_name:rate_query}
810ms    ToolRegistry: 6 policy checks pass -> rate_query executes
900ms    SQL executes via WarehouseSpark: 23 current DRG rows returned
910ms    Observation stored in working_data
915ms    LLM called again with updated context
1800ms   LLM returns {action:SYNTHESIZE, final_answer:"Sutter Health has..."}
1810ms   HallucinationGuard: all 5 checks PASS
1815ms   AnswerValidator: no absence claims detected -> PASS
1820ms   AnswerGroundingEnforcer: structured data present -> PASS
1825ms   CitationValidator: citations verified -> PASS
1830ms   ResponseAssembler packages QueryResponse
1835ms   Stream begins delivering token events to browser
4800ms   Stream closes -> 'done' event delivered
4810ms   _persist_turn() fires (background)
4820ms   TelemetryWriter.write() fires (background)
4830ms   User sees complete answer with confidence badge and clickable citations
```

For complex multi-hop questions (compare two providers, temporal analysis), the timeline extends to 8–15 seconds as the agent runs 4–8 tool calls.

---

## 7. The AI Tools Arsenal — All 25+ Tools

Every tool is a self-contained Python class inheriting `BaseTool`. Each implements a `run(**kwargs) -> ToolResult` method. Tools never know they are inside a ReAct loop — they simply receive parameters, execute, and return a `ToolResult`.

### Tool Categories

#### Category A: SQL and Structured Data Tools

**`sql_query`**
- **What it does:** Natural language to SQL. Builds schema context from `dim_table_schema`, calls Claude Sonnet 4.6 to generate SQL, validates against the 13-table whitelist, blocks DDL/DML, executes via WarehouseSpark.
- **Best for:** Cross-provider aggregations, status lookups, count queries, filtering by LOB or payment type.
- **Cost estimate:** $0.001 per call
- **Example questions it handles:** *"How many active Medi-Cal providers do we have?"*, *"Which providers have auto_renewal=True?"*

**`rate_query`**
- **What it does:** Specialized rate lookup. Queries `tbl_contract_rates_all` (459,902 rows) with intelligent defaults: status='current', correct date-range logic for point-in-time queries, amendment_order awareness.
- **Best for:** *"What are the current rates for Provider X?"*, *"Show me the DRG rates from amendment 3."*
- **Cost estimate:** $0.001 per call

**`clause_existence`**
- **What it does:** Checks whether a specific clause type (offset, DOFR, IPA delegation) exists for a provider. Reads pre-computed flags from `tbl_genie_provider_profile`: `offset_clause_status`, `dofr_status`, `ipa_delegation_status` (each = 'HAS' or 'NO_MENTION').
- **Best for:** *"Does Adventist Health have a DOFR clause?"*
- **Business rule:** Never adds `AND status='HAS'` to the WHERE clause — that would return 0 rows for NO_MENTION providers. Instead: get the row, read the status column, translate to Yes/No.
- **Cost estimate:** $5.50 (high because it may trigger multi-step LLM reasoning)

**`temporal_analysis`**
- **What it does:** Time-series analysis over `tbl_genie_amendment_timeline` (6,609 rows, 302 providers). Answers "how has X changed over time?"
- **Best for:** Amendment history, rate escalation over time, contract term evolution.
- **Cost estimate:** $0.001 per call

**`genie_query`**
- **What it does:** Sends the question to Databricks Genie Space (ID: 01f14d360bd51f8d801452065b2df600) via the Genie REST API. Genie auto-generates SQL and returns results. Uses exponential backoff polling.
- **Best for:** Complex natural language queries where the SQL generation benefit of Genie's larger context window is useful.
- **Cost estimate:** $0.005 per call (network round-trip)
- **Feature flag:** `GENIE_ENABLED = True`

**`amendment_chain`**
- **What it does:** Traces the full amendment lineage for a provider: base agreement → amendment 1 → amendment 2 → ... with effective dates and supersession relationships from `tbl_contract_documents_master`.
- **Best for:** *"Show me the complete amendment history for Kaiser."*, *"What version of the contract is currently in force?"*

#### Category B: Semantic Search Tools

**`passage_search`**
- **What it does:** Semantic similarity search over the unified vector index (727,876 chunks). Accepts `keywords` list and optional `provider_name` filter. Returns top-20 passages with cosine similarity scores.
- **Best for:** Clause text retrieval, *"What does the offset clause say?"*, *"Find language about termination notice."*
- **Cost estimate:** $0.010 per call
- **Critical rule:** Always pass `provider_name` for content questions. Zero provider-filtered results = no indexed content for that provider on that topic. Never retry without the filter.

**`document_fetch`**
- **What it does:** Retrieves the full text of a specific document from the vector index by `source_filename`. Returns all chunks associated with that file.
- **Best for:** Full document review, *"Show me everything from the Scripps 2023 amendment."*

**`field_extraction`**
- **What it does:** Extracts specific fields (dates, rates, named parties) from retrieved passages using targeted LLM prompts. Used when structured extraction from unstructured text is needed.

#### Category C: LLM Generation Tools

**`summarize`**
- **What it does:** Uses Claude Sonnet 4.6 to synthesize multiple retrieved passages into a coherent summary. Used when a content question spans multiple documents.
- **Cost estimate:** $0.150 per call (expensive — uses large token counts)

**`draft_text`**
- **What it does:** Drafts contract language based on retrieved clause examples and user specifications. All generation is grounded in actual clause text from the vector index.
- **Cost estimate:** $0.100 per call

**`explanation`**
- **What it does:** Explains contract terms, legal abbreviations (DRG, DOFR, capitation, stop-loss, carve-out, IPA delegation), and regulatory concepts in plain English. Uses a curated knowledge base.

#### Category D: Analytical and Comparison Tools

**`comparison`**
- **What it does:** Side-by-side analysis of two providers across any dimension: rates, clauses, LOB, amendment depth. Calls the appropriate underlying tools for each provider and presents results as a structured comparison.
- **Best for:** *"Compare Sutter Health and Sharp Healthcare's DRG rates."*

**`multi_concept`**
- **What it does:** Evaluates multiple contract concepts (offset, DOFR, auto-renewal, IPA delegation, AB352, encounters) for one or more providers in a single LLM pass. Used by the Report Generator.
- **Cost estimate:** $12.00 per call (most expensive tool — large context)

**`provider_deep_dive`**
- **What it does:** Comprehensive single-provider profile aggregating: rates summary (min/max/avg), clause coverage, document inventory, amendment count, financial exposure, risk tier. Calls multiple underlying services and formats them together.

**`calculate`**
- **What it does:** Arithmetic calculations on retrieved numeric data: averages, totals, percentages, year-over-year changes, delta between amendment versions.

**`chart_generate`**
- **What it does:** Generates Vega-Lite JSON chart specifications from structured data. Used for rate trend visualizations and comparison charts embedded in the chat response.

**`export_data`**
- **What it does:** Exports structured data (rate tables, provider lists, clause matrices) as CSV, JSON, or Markdown. Called when a user asks to download or export results. Max 10,000 rows.

#### Category E: Phase 10B Specialized Tools

**`risk_alert`** — Queries `tbl_contract_risk_scores` (272 rows), `tbl_alert_queue` (351 rows, 229 CRITICAL), `tbl_contract_deadlines` (713 rows). Returns priority-sorted risk information.

**`financial_analysis`** — Queries `tbl_financial_exposure` (172 rows), `tbl_repricing_scenarios` (7,600 rows), `tbl_rate_escalators`, `fact_supersession`. Answers financial exposure and repricing scenario questions.

**`system_membership`** — Queries `dim_health_system` (20 systems), `tbl_provider_system_membership` (112 rows), `tbl_service_areas` (306 rows). Answers *"Which health system is Providence part of?"*

**`compliance_query`** — Queries `tbl_compliance_tracking` (2,871 rows, 7 regulations), `tbl_quality_performance` (894 rows), `tbl_contract_notifications` (1,934 rows). Uses 4 query types: compliance_status, quality_metrics, notification_history, summary.

**`clause_text`** — Queries `tbl_contract_clauses` (7,863 rows, 21 categories) and `tbl_clause_deviations` (7,777 rows). Returns actual extracted clause text and deviation analysis. 4 query types: clause_text, category_summary, clause_compare, network_norms.

**`provenance`** — Queries `tbl_extraction_provenance` (15,368 rows) and `fact_supersession` (2,380 rows). Answers *"Where did this data come from?"* and *"What replaced what?"*

#### Category F: Utility Tools

**`provider_lookup`** — Resolves ambiguous provider name strings to canonical entries in `dim_provider_canonical` (531 entries). Used by all other tools that need a canonical provider_id.

**`citation_verify`** — Uses Claude Haiku 4.5 to verify each citation against its source passage. Removes citations that do not actually support the stated claim.

### Tool Cost Budget Management

| Tool | Cost Estimate | Notes |
|---|---|---|
| `sql_query` | $0.001 | Cheap: 1 LLM SQL gen + 1 warehouse query |
| `passage_search` | $0.010 | Moderate: VS API call |
| `rate_query` | $0.001 | Very cheap: direct SQL |
| `clause_existence` | $5.50 | Expensive: multi-step reasoning |
| `multi_concept` | $12.00 | Most expensive: large LLM context |
| `summarize` | $0.150 | LLM call with large context |
| `citation_verify` | $0.020 | Haiku: cheap verification call |
| `genie_query` | $0.005 | Network round-trip |

Default session budget: $15.00. The CostController blocks tool calls when `remaining_budget < tool.max_cost_usd`. This prevents a session from spending more than $15 in LLM and API costs.

---

## 8. The Trust and Validation Layer

The platform's credibility depends on its ability to distinguish what it knows from what it doesn't. The trust layer is a defense-in-depth stack of 4 independent validators, applied sequentially after every answer synthesis.

### Why This Matters Commercially

A contract intelligence system that occasionally states wrong rates or fabricates clause existence creates legal risk. Every answer shown to a BSC analyst must be:
1. **Grounded** — supported by evidence from the source data
2. **Consistent** — numbers in the answer appear in cited sources
3. **Complete** — absence claims verified against all relevant tables
4. **Cited** — every factual claim traceable to a source document

The 4-layer validation stack enforces all four requirements.

### Layer 1: HallucinationGuard

**File:** `platform/v4/core/hallucination_guard.py`
**Cost:** Zero (no LLM calls)
**When it runs:** After every SYNTHESIZE action

Five independent checks:

| Check | What It Detects | Severity |
|---|---|---|
| Numeric consistency | Numbers in answer not found in any tool observation | WARN |
| Provider identity | Provider names not in session context | WARN |
| Date plausibility | Dates outside 1990–2035 range | WARN |
| Ungrounded assertions | High-specificity sentences with zero citation support | WARN |
| Confidence floor | Confidence > MODERATE when no evidence retrieved | BLOCK |

Block findings prevent the answer from being returned. Warn findings are logged and attached to the response metadata. The confidence floor is the most commonly triggered: if the LLM claims HIGH confidence but no tool results or passages were retrieved, confidence is automatically downgraded.

### Layer 2: AnswerValidator ("Prove It" Guardrail)

**File:** `platform/v4/core/answer_validator.py`
**Cost:** 1 cheap SQL query when triggered
**When it runs:** After HallucinationGuard

This validator prevents the most common failure mode in RAG systems: the agent claims something doesn't exist when it does (in a different table).

**Absence claim detection:** 12 regex patterns match phrases like:
- *"does not have"*, *"no data exists"*, *"not available"*, *"no traditional DOFR"*
- *"zero rows"*, *"unable to find"*, *"no relevant records"*

When detected:
1. The question's keywords determine the expected table (`dofr` → `tbl_dofr_matrix_extracted`)
2. `SELECT 1 FROM tbl WHERE ... LIMIT 1` runs in ~100ms
3. If rows exist: `ValidationResult(needs_retry=True, correction="Data exists in tbl_X")`
4. Agent retries with the correction — typically resolves on first retry

**Domain mismatch detection:** The validator also checks if the answer is from the wrong table (e.g., returning alert rows for a question about expired contracts).

### Layer 3: AnswerGroundingEnforcer

**File:** `platform/v4/services/answer_grounding.py`
**Cost:** Zero (keyword scoring, no LLM)
**When it runs:** After AnswerValidator, before synthesis when max steps reached

Computes a relevance score for each retrieved passage against the query keywords. The standard refusal message is returned when:
- Maximum passage relevance score < 0.5 across all retrieved passages
- AND no structured SQL results exist in working_data

The refusal message: *"I could not find verified evidence for this question in the contract corpus. The available documents do not contain sufficient information..."*

This replaces the old behavior of returning a WARNING-prefixed guess. An honest refusal is better than an ungrounded answer.

### Layer 4: CitationValidator

**File:** `platform/v4/services/citation_validator.py`
**Model:** Claude Haiku 4.5 (fast, low cost)
**When it runs:** Post-synthesis, per citation

For each citation in the answer:
1. Retrieves the source passage text from the vector index
2. Sends: *"Does this passage support this specific claim? YES or NO."*
3. Haiku returns YES/NO in ~300ms
4. Citations returning NO are removed; confidence score is reduced

### The System Prompt as a Validation Layer

Beyond the 4 runtime validators, the agent's system prompt itself encodes 16+ explicit business rules that act as a fifth validation layer at generation time:

- **Anti-hedging rules:** Never say *"I was unable to determine"* when evidence IS present in working_data
- **Anti-deflection rules:** Never say *"refer to the PDF"* or *"consult the original document"* — the application IS the access point
- **DOFR rules:** 6 specific rules governing when to use `sql_query` vs. `passage_search` for DOFR questions; when to give the TYPE B (FFS) response; never fabricate matrix rows
- **Provider filter enforcement:** Always pass `provider_name` to `passage_search` for content questions; zero provider-filtered results = no content for that provider
- **Rate query rules:** Use `status='current'` and `amendment_order` correctly; never mix current and superseded rates
- **Anti-ranking rules:** Never assert the "same provider" tops two independent ranked lists without SQL evidence

### Confidence Score Mapping

| Label | Score | When Used |
|---|---|---|
| HIGH | 0.90 | SQL data retrieved + citations verified + no guard warnings |
| MODERATE | 0.65 | Partial evidence; some unverified citations; one guard warning |
| LOW | 0.35 | Weak evidence; multiple guard warnings; no SQL results |
| UNCERTAIN | 0.15 | Minimal evidence; grounding enforcer nearly triggered |

---

## 9. The LLM Layer — How AI Models Are Used

Two LLM models are used, chosen for different cost/performance tradeoffs. Both run on Databricks Model Serving.

### The Two Models

| Model | Endpoint Name | Used For | Max Tokens |
|---|---|---|---|
| **Claude Sonnet 4.6** | `databricks-claude-sonnet-4-6` | Agent reasoning, SQL generation, answer synthesis, question decomposition | 6,000 (reasoning) |
| **Claude Haiku 4.5** | `databricks-claude-haiku-4-5` | Citation verification, entity extraction, fast lookups | 1,200 |

**Sonnet 4.6** is the primary model. Its broader context window and stronger reasoning capability are essential for the 16-page system prompt + accumulated tool observations that the ReAct loop requires.

**Haiku 4.5** is used wherever speed and cost matter more than depth. Citation verification calls are typically a YES/NO decision that Haiku handles accurately at a fraction of the cost.

### DatabricksLLMClient

**File:** `platform/v4/runtime/databricks_llm_client.py`

The client wraps the Databricks Model Serving REST API:

- **Authentication:** SDK-native OAuth via `WorkspaceClient`. The app's service principal token is automatically rotated by the Databricks Apps runtime. No hardcoded credentials anywhere.
- **Startup probe:** At startup, a 3-token ping (`"ping"` → max_tokens=3) verifies the endpoint is reachable before the runtime is marked ready.
- **Methods:**
  - `chat(model, system, user, max_tokens)` — single-turn call
  - `chat_multi(model, messages, max_tokens)` — multi-turn call (used for session-aware reasoning)

### Where the LLM Is Used

| Component | Model | Purpose | Typical tokens |
|---|---|---|---|
| AgentController — each reasoning step | Sonnet 4.6 | Select next action (THINK/CALL_TOOL/SYNTHESIZE/ABORT) | ~3,000 input, ~300 output |
| AgentController — synthesis | Sonnet 4.6 | Generate final answer with citations | ~4,000 input, ~800 output |
| SqlQueryTool — SQL generation | Sonnet 4.6 | NL → SQL from schema context | ~1,500 input, ~200 output |
| QuestionDecomposer | Sonnet 4.6 | Split complex questions into sub-questions | ~500 input, ~300 output |
| CitationValidator — per citation | Haiku 4.5 | Verify citation supports claim | ~500 input, ~10 output |
| SummarizeTool | Sonnet 4.6 | Synthesize multiple source passages | ~3,000 input, ~500 output |
| DraftTextTool | Sonnet 4.6 | Draft contract language from examples | ~2,000 input, ~600 output |

### The System Prompt Architecture

The system prompt sent to the LLM at each reasoning step is assembled dynamically from 5 components:

1. **Static rules block** (~2,000 tokens) — 16 sections of business logic:
   - Provider status rules (is_active semantics, flag interpretation)
   - LOB normalization (use `lob_normalized`, not `line_of_business`)
   - Rate version tracking (amendment_order, status='current' vs 'superseded')
   - DOFR handling (6 sub-rules for the most complex query type)
   - Anti-hallucination rules
   - Anti-deflection rules (never say "refer to the PDF")
   - Provider name disambiguation
   - Vague intent handling ("any provider", "an example")
   - Portfolio-level question patterns

2. **Tool catalogue** (~1,000 tokens) — name, description, and parameter schema for every registered tool

3. **Session history** (~500 tokens) — previous Q&A turns from `tbl_agent_sessions`

4. **Working data** (variable, up to ~2,000 tokens) — accumulated tool observations from this question's steps so far

5. **Provider context** (~100 tokens if set) — pre-scopes all tool calls to a specific provider

### Genie Space Integration

The Genie Space (ID: `01f14d360bd51f8d801452065b2df600`) provides an alternative structured-data path. When `genie_query` is called:

1. The question is sent to the Genie REST API (`/api/2.0/genie/spaces/{id}/start-conversation`)
2. The Genie system returns a `conversation_id`
3. `polling.py` polls for the response with exponential backoff (1s, 2s, 4s, 8s, up to `GENIE_QUERY_TIMEOUT_S = 120s`)
4. Genie auto-generates SQL against the Unity Catalog tables and returns results
5. Results are returned as `ToolResult.data`

Genie is used as an alternative when the agent suspects the native `sql_query` tool may generate suboptimal SQL for a particular question structure.

### Cost Tracking

Every LLM call contributes to the session's cumulative cost via `CostController`:

- Each reasoning step: $0.005
- `passage_search`: $0.010
- `clause_existence`: $5.50
- `multi_concept`: $12.00
- `summarize`: $0.150

The cost controller blocks tool calls when `cumulative_cost + tool.max_cost >= session_budget` ($15.00 default). This cap prevents runaway sessions from becoming expensive while still leaving enough budget for any single question.

---

## 10. The Data Pipeline — How Knowledge Was Built

The 31 Delta tables and 727,876 vector chunks didn't appear by magic. They were built by a numbered pipeline of Python scripts (`pipeline/00` through `pipeline/11`) running in Databricks notebooks.

### The 8 Pipeline Stages

#### Stage 0: Configuration (`00_config.py`)

Sets all pipeline parameters: Unity Catalog target (`dev_adb.raw`), volume path for PDFs, OCR engine settings, LLM extraction model names, batch sizes, and retry limits. All downstream scripts import from this config.

#### Stage 1: DDL Execution (`01_execute_ddl.py`)

Creates all 31 Delta tables with full schema definitions, including:
- Column types (BOOLEAN vs STRING vs TIMESTAMP vs DOUBLE)
- NOT NULL constraints
- Delta table properties (optimized writes, autocompaction)
- Initial indexing recommendations

Running this script on a fresh catalog creates the complete data model in ~5 minutes.

#### Stage 2: State Engine (`02_state_engine.py`)

Manages incremental processing. Maintains a state table tracking which PDFs have been processed, which failed, and which need reprocessing. Enables:
- **Incremental runs:** Only new/changed PDFs are processed on each run
- **Retry logic:** Failed documents are retried up to 3 times with backoff
- **Progress monitoring:** A status view shows % complete per document batch

#### Stage 3: Reimbursement Migration (`03_reimbursement_migration.py`)

Migrates legacy reimbursement data (from an older format) into the unified `tbl_contract_rates_all` schema. Handles:
- Rate unit normalization (PER_DAY, PER_CASE, PERCENT_OF_BILLED, DRG, etc.)
- Amendment order assignment
- Status labeling ('current' vs 'superseded') using the temporal re-labeling logic

#### Stage 4: Legal Chunking (`04_legal_chunking.py`)

The most complex pipeline stage. Processes each PDF to produce two types of chunks:

**Layer 1 — Raw OCR Passages:** (`chunk_type=PASSAGE`)
- Average ~200 tokens each
- Sliding window with 10% overlap to preserve context across chunk boundaries
- Preserves section headers and page numbers
- 670,728 passages total

**Layer 2 — Structured Legal Units:** (`chunk_type=CLAUSE|DEFINITION|RATE_TABLE|EXHIBIT|SCHEDULE`)
- Average ~400 tokens each
- Section-boundary-aware splitting (splits at numbered sections, not mid-sentence)
- Preserves the full section header chain (e.g., "Section 4.2.1: Capitation Rates")
- Extracts unit type from section headings
- 57,148 structured units total

`src/chunking.py` provides the core chunking logic used by this stage.

#### Stage 5: Retrieval Engine (`05_retrieval_engine.py`)

Builds `tbl_vs_unified_ready` (727,876 rows = Layer 1 + Layer 2) and syncs it to the vector search index:

1. Computes GTE-1024d embeddings for each chunk (via Databricks Model Serving)
2. Writes chunk metadata + embeddings to `tbl_vs_unified_ready`
3. Syncs to `idx_unified_v4` via the Databricks Vector Search Delta Sync API
4. Sets `is_current=TRUE` for chunks from documents with `contract_status=ACTIVE` or `ACTIVE_NO_EXPIRY`

#### Stage 7: Citation Verification (`07_citation_verification.py`)

Post-ingestion quality check. For a sample of extracted clauses, verifies that the clause text:
1. Is actually found in the source document (not hallucinated during extraction)
2. Is attributed to the correct source filename
3. Has a valid `page_number_est` that is within the document's page count

Populates `tbl_extraction_provenance` with verification status.

#### Stage 8: Intelligence Extraction (`08_intelligence.py`)

Runs the intelligence layer that produces the pre-computed flags used by the agent:

- **Provider profile flags:** For each provider, scans their chunks for offset clauses, DOFR language, and IPA delegation language. Sets `offset_clause_status`, `dofr_status`, `ipa_delegation_status` to 'HAS' or 'NO_MENTION' in `tbl_genie_provider_profile`.
- **Risk scoring:** Computes risk scores based on contract term length, amendment frequency, rate trend, and compliance status. Writes to `tbl_contract_risk_scores`.
- **Amendment chain analysis:** Builds the amendment timeline in `tbl_genie_amendment_timeline` with `amendment_order` and `sequential_order` columns.
- **DOFR matrix extraction:** For IPA/capitation providers, extracts the service-category responsibility matrix into `tbl_dofr_matrix_extracted` (1,821 rows).

#### Stage 11: Evaluation Harness (`11_evaluation_harness.py`)

Runs the automated test suite against the live application. Feeds 99+ test questions, scores answers using Claude Sonnet 4.6 as judge, and produces the pass/fail scorecard used to validate each pipeline run.

### Data Quality Infrastructure

The pipeline produces 8 regression sentinels that run as SQL views:

| Sentinel | Metric | Pass Threshold |
|---|---|---|
| S-1 | Rate traceability | >= 98% |
| S-2 | Clause verification | = 100% |
| S-3 | Notice-days accuracy | = 100% |
| S-4 | Compliance evidence coverage | >= 95% |
| S-5 | Active provider coverage | = 100% |
| S-6 | Status/date consistency | = 100% |
| S-7 | Delegation alignment mismatches | = 0 |
| S-8 | Notification types present | >= 6 |

All 8 sentinels are checked via `vw_phase5_regression_sentinels` before any production deployment.

---

## 11. Operational Infrastructure

The operational layer makes the platform observable, reliable, and continuously improving.

### Three Circuit Breakers

**File:** `platform/v4/middleware/circuit_breaker.py`

Three independent circuit breakers protect against cascading failures:

| Breaker | Protects | Failure threshold | Recovery |
|---|---|---|---|
| `sql_breaker` | SQL warehouse calls | 3 consecutive failures | 30-second half-open probe |
| `vs_breaker` | Vector search calls | 3 consecutive failures | 30-second half-open probe |
| `llm_breaker` | LLM API calls | 3 consecutive failures | 60-second half-open probe |

**States:** CLOSED (normal) → OPEN (all calls blocked, fast-fail) → HALF_OPEN (one probe call allowed) → CLOSED (if probe succeeds)

**Fail-open by design:** If VS is OPEN, the agent continues using SQL-only answers. If LLM is OPEN, the query endpoint returns a clear service-unavailable message rather than hanging. The SQL warehouse OPEN state is the only one that fully disables the chat endpoint.

Circuit breaker states are reported at `/health` and displayed on the StatusPage dashboard.

### Session Persistence

**File:** `platform/v4/data/session_repo.py`
**Table:** `tbl_agent_sessions` (>= 2,229 rows as of 2026-07-21)

Every completed query turn is automatically persisted via `_persist_turn()` — a fire-and-forget background task that runs after the SSE stream closes:

```
Written fields: session_id, turn_id, question, answer,
  confidence, confidence_score, citations (JSON),
  latency_ms, tool_calls, cost_usd, created_at, provider_name
```

The session sidebar in ChatPage calls `GET /api/v1/sessions` to list recent conversations. Multi-turn context is restored by loading the last N turns from this table at the start of each new question.

### Query Cache

**File:** `platform/v4/data/query_cache_repo.py`
**Table:** `tbl_query_cache`

Identical questions (same text, same provider scope) within the cache TTL return instantly without re-running the agent. Cache key = SHA-256 hash of `(question, provider_name)`. Hit rate is low for unique analytical questions but high for repeated status checks and common lookups.

### User Feedback Loop

**File:** `platform/v4/data/feedback_repo.py`
**Table:** `tbl_user_feedback`

Every AI message has ThumbsUp/ThumbsDown controls. Thumbs-down opens a correction field. All feedback is stored with the full answer context for human review and future fine-tuning. This table is the primary source for identifying recurring answer quality issues.

### Telemetry Stack

**Files:** `platform/v4/services/telemetry_writer.py`, `platform/v4/data/telemetry_repo.py`
**Tables:** `app_query_telemetry`, `fact_retrieval_telemetry`

`app_query_telemetry` captures per-query metrics:
- `latency_ms`, `cost_usd`, `confidence`, `tool_calls`, `cache_hit`
- `provider_name` (for "most queried providers" ranking)
- `timestamp` (for 7-day trend charts on StatusPage)

`fact_retrieval_telemetry` captures per-retrieval events:
- Which chunks were retrieved, their `quality_score`, `cosine_similarity`
- Whether each chunk was ultimately cited in the answer
- Used by the Phase 5 evaluation harness for retrieval quality scoring

Both writes are fire-and-forget (`asyncio.create_task()`) — telemetry failures never affect user-facing responses.

### Schema Context Builder

**File:** `platform/v4/data/schema_context_builder.py`
**Table:** `dim_table_schema`

The LLM can't know what columns exist in each table without being told. `SchemaContextBuilder.build(table_names)` queries `dim_table_schema` for column names, types, and business-facing descriptions, then formats them as a compact schema block injected into the SQL generation prompt.

This is how the LLM knows that `tbl_compliance_tracking` has a `regulation` column with valid values KNOX_KEENE, CMS_MA, DMHC_STANDARDS, etc. — and that `tbl_quality_performance` uses `program_name`, not `program_type`.

### Monitoring Views

The pipeline creates 4 monitoring SQL views in `dev_adb.raw`:

| View | Purpose |
|---|---|
| `vw_phase5_regression_sentinels` | All 8 regression sentinel checks in one query |
| `vw_phase5_table_drift_detection` | Detects unexpected row count changes in all major tables |
| `vw_phase5_expiration_alerts` | Surfaces contracts nearing expiration |
| `vw_phase5_external_data_triggers` | Monitors for external data updates that should trigger pipeline rerun |

---

## 12. The 6 Phase 10B REST API Tool Groups

Phase 10B added 6 specialized tool groups to the agent and exposed each as a REST API endpoint group. These enable programmatic access beyond the chat interface.

### Group 1: Risk and Alert Intelligence

**Tool:** `risk_alert` | **Endpoints:** `/api/v1/risk/...`

Answers questions about contract risk and upcoming deadlines. Sources:
- `tbl_contract_risk_scores` (272 rows) — risk_score, risk_tier, risk_factors per provider
- `tbl_alert_queue` (351 rows, 229 CRITICAL) — alert_type, priority, days_until_due
- `tbl_contract_deadlines` (713 rows) — deadline_type, due_date, required_action

**Example questions:** *"Which contracts are at critical risk right now?"*, *"What deadlines are due in the next 30 days?"*, *"Show me all CRITICAL alerts for Sutter Health."*

### Group 2: Financial Intelligence

**Tool:** `financial_analysis` | **Endpoints:** `/api/v1/financial/...`

Answers financial exposure and repricing questions. Sources:
- `tbl_financial_exposure` (172 rows) — exposure_usd, exposure_pct, risk_category per provider_id
- `tbl_repricing_scenarios` (7,600 rows) — scenario_name, projected_rate, impact_usd
- `tbl_rate_escalators` — escalator clauses with projected rates by year
- `fact_supersession` (2,380 rows) — which rate versions replaced which

**Note:** `tbl_financial_exposure` has no `provider_name` column. Queries JOIN to `tbl_genie_provider_profile` ON `provider_id`.

**Example questions:** *"What is our total financial exposure to Dignity Health?"*, *"Show me repricing scenarios if we renegotiate Kaiser's rates by 5%."*

### Group 3: Network and Geography Intelligence

**Tool:** `system_membership` | **Endpoints:** `/api/v1/network/...`

Answers health system membership and geographic coverage questions. Sources:
- `dim_health_system` (20 systems) — system_id, system_name, system_type
- `tbl_provider_system_membership` (112 rows, 106 providers, all OWNED relationship)
- `tbl_service_areas` (306 rows, 8 regions)

**Example questions:** *"Which health system does Mercy Medical Center belong to?"*, *"What geographic regions does Sutter Health serve?"*, *"List all providers in the CommonSpirit Health system."*

### Group 4: Compliance and Quality Intelligence

**Tool:** `compliance_query` | **Endpoints:** `/api/v1/compliance/...`, `/api/v1/quality/...`, `/api/v1/notifications/...`

Answers regulatory compliance and quality performance questions. Sources:
- `tbl_compliance_tracking` (2,871 rows, 7 regulations)
- `tbl_quality_performance` (894 rows, 291 providers, 5 program types: HEDIS/WITHHOLD/VALUE_BASED/P4P/STAR_RATING)
- `tbl_contract_notifications` (1,934 rows, 7 notification types)

**Important facts:** `compliance_status` values are COMPLIANT/PARTIAL/UNKNOWN/EXEMPT — NON_COMPLIANT has zero rows in current data. `tbl_quality_performance` uses `program_name` column, not `program_type`. Average notice days by type: TERMINATION=136d, AUDIT=145d, RATE_CHANGE=151d.

### Group 5: Clause Intelligence

**Tool:** `clause_text` | **Endpoints:** `/api/v1/clauses/...`

Answers questions about extracted clause text and network deviations. Sources:
- `tbl_contract_clauses` (7,863 rows, 300 providers, 21 categories, `clause_status=HAS|FALSE_POSITIVE`)
- `tbl_clause_deviations` (7,777 rows, `deviation_type=CONFORMANT|DEVIATION`, `severity=NONE|LOW|MEDIUM|HIGH`)

**4 query types:** `clause_text` (retrieve actual text), `category_summary` (all clause categories for a provider), `clause_compare` (two providers side-by-side), `network_norms` (baseline clause language across the network).

### Group 6: Provenance and Audit

**Tool:** `provenance` | **Endpoints:** `/api/v1/provenance/...`

Answers questions about data lineage and extraction quality. Sources:
- `tbl_extraction_provenance` (15,368 rows, `extraction_method=PATTERN_MATCH` only, all `human_verified=FALSE`)
- `fact_supersession` (2,380 rows, `doc_type=full_document`, `supersession_type=temporal_chain`, `confidence=0.85`)

**Example questions:** *"How was this rate extracted?"*, *"What superseded the 2022 Cedars-Sinai agreement?"*, *"Show me the extraction quality summary for Adventist Health."*

---

## 13. Security Architecture

### Defense-in-Depth Layers

Security is implemented at 7 independent layers so that a breach of any single layer does not expose contract data:

**Layer 1: Databricks Apps Authentication**
The application runs under a Databricks service principal. Users must be authenticated Databricks workspace members to access the app URL. Unauthenticated requests never reach `app/main.py`.

**Layer 2: Rate Limiting**
`slowapi` enforces 120 requests/minute per IP address. This prevents both denial-of-service attacks and runaway automated query loops. HTTP 429 responses include a `Retry-After` header.

**Layer 3: CORS Policy**
Only the Databricks Apps domain is in the allowed origins list. Cross-origin requests from unknown domains are rejected at the HTTP level.

**Layer 4: Input Sanitization**
`_sanitize_like_input()` applies to all user-controlled strings before SQL injection:
- Truncate to 100 characters
- Strip characters outside `[a-zA-Z0-9 .\-_&'/()]`
- Escape SQL wildcards: `%` removed, `_` escaped as `\_`
- Single quotes doubled: `'` → `''`

**Layer 5: PII Redaction**
`redact_pii()` scans all question text before LLM injection:
- SSN: `\d{3}-\d{2}-\d{4}` → `[SSN REDACTED]`
- DOB: `DOB:` or `Date of Birth:` followed by date → `[DOB REDACTED]`
- Street addresses: `\d+ Street/Ave/Blvd...` → `[ADDRESS REDACTED]`

This is defense-in-depth on top of corpus-level PII removal in the pipeline.

**Layer 6: SQL Injection Prevention (Table Whitelist + DDL Block)**
Two independent mechanisms:
1. `CONTRACT_TABLES_FOR_SQL`: 13 allowed tables. Any table reference not on this list fails validation before execution.
2. DDL/DML pattern block: Any generated SQL containing CREATE, INSERT, UPDATE, DELETE, DROP, ALTER, TRUNCATE, GRANT, REVOKE, or MERGE is rejected.

**Layer 7: Path Traversal Protection**
The PDF serving endpoint (`/api/v1/documents/{filename}/pdf`) validates that the resolved absolute path starts with the allowed volume directory prefix before opening the file. This prevents `../../etc/passwd` style attacks.

### OAuth Token Rotation

`DatabricksLLMClient` uses `WorkspaceClient()` from the Databricks Python SDK. In the Databricks Apps runtime, the SDK automatically uses the app's service principal OAuth token, which is rotated by the platform every 15 minutes. No secrets are stored in the application code or environment variables.

### Audit Trail

Every query is logged to `app_query_telemetry` with timestamp, user context, question text, and answer metadata. This provides a complete audit trail of all contract data access.

---

## 14. Deployment, Build, and Operations

### The `app.yaml` Manifest

`app.yaml` is the contract with the Databricks Apps runtime. It declares everything the app needs to run:

```yaml
command: ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000",
          "--log-level", "info"]

env:
  - APP_CATALOG: "dev_adb"
  - APP_SCHEMA: "raw"
  - DATABRICKS_WAREHOUSE_ID: "2c99c6485f03ee73"
  - LLM_ENDPOINT: (resolved from resource binding -> endpoint name)
  - LLM_FAST_ENDPOINT: (resolved from resource binding -> endpoint name)
  - VS_ENDPOINT_NAME: "contract_intelligence_vs_endpoint"

resources:
  - SQL warehouse (CAN_USE permission)
  - Claude Sonnet 4.6 serving endpoint (CAN_QUERY permission)
  - Claude Haiku 4.5 serving endpoint (CAN_QUERY permission)
```

The `resources` block is how the app gets permission to use the warehouse and LLM endpoints without any hardcoded credentials. The Databricks Apps runtime injects the endpoint names as environment variables at startup.

### Frontend Build Workflow

The frontend is pre-built and committed. To rebuild after frontend changes:

1. **Bootstrap npm:** Download from npm registry tarball to `/tmp/npm_bootstrap/` on the cluster
2. **Cache node_modules:** Key = MD5 hash of `package-lock.json`. If unchanged, skip `npm ci` (~3 min saved)
3. **Build:** `cd frontend && node /path/to/npm-cli.js run build` → outputs to `app/static/`
4. **Deploy:** `WorkspaceClient().apps.deploy(app_name, AppDeploymentMode.SNAPSHOT)` — Databricks CLI is blocked in USER_ISOLATION mode; SDK is the only path
5. **Cleanup:** Delete `frontend/node_modules/` (excluded by `.databricksignore`)

The complete workflow is in `build_and_deploy.py`. Typical runtime: ~10 seconds if node_modules cache is warm, ~3–4 minutes on a fresh cluster.

### The `.databricksignore` File

Prevents large directories from being uploaded to the Databricks workspace during deployment:
```
frontend/node_modules/
__pycache__/
.pytest_cache/
*.pyc
.git/
```

### SLO Targets

| Metric | Target | Current Status |
|---|---|---|
| Availability | 99.5% uptime | Maintained via fail-open startup + 3 circuit breakers |
| Chat p95 latency | < 5 seconds | Typical: 3–5s for simple questions |
| Provider Explorer p95 latency | < 3 seconds | Typical: 1–2s |
| Complex multi-hop latency | < 15 seconds | Typical: 8–12s |
| Answer confidence | >= 65% at MODERATE or above | Phase 5: 80%+ MODERATE/HIGH |
| Hallucination rate | < 10% | Phase 5: 5.63% |

### Development vs. Production Routing

All table references in `settings.py` are env-var driven:
```python
CATALOG = os.environ.get("APP_CATALOG", "dev_adb")
SCHEMA  = os.environ.get("APP_SCHEMA",  "raw")
```

To route to a production catalog, change `APP_CATALOG` and `APP_SCHEMA` in `app.yaml` without any code changes. All 31 table paths are assembled from these two variables.

### The Comprehensive Test Suite

The `Contract_Intelligence_Comprehensive_Test_Suite` notebook is the primary regression harness:

- **121 cells** covering 13 data model validation checks and 99 application question tests
- **Sections:** Provider Profile, Rates, Clause Existence, Clause Text, DOFR, Delegation, Compliance/Quality, Risk/Finance, Network, Temporal, Provenance, Identifiers, Edge Cases, Multi-turn sessions
- **Scoring:** Claude Sonnet 4.6 as LLM judge scores each answer against known facts grounded in live DB queries
- **Data model checks:** Each check validates exact row counts against verified facts (e.g., 306 total providers, 194 active, 47 with offset HAS)
- **Run frequency:** Before every production deployment. Results must show >= 95% judge PASS rate.

---

*This document covers the complete Contract Intelligence Platform V4 application as of 2026-07-21. For data model details, see `CONTRACT_INTELLIGENCE_V4_BUSINESS_OVERVIEW.md`. For live data counts and SLO tracking, see the StatusPage at `/status` or query `vw_phase5_regression_sentinels` directly.*
