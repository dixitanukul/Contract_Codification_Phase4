# BSC Provider Contract Intelligence Platform (PCIP)
## Master System Documentation — Complete Technical Reference

**Version:** 3.0 | **Last Updated:** June 2026 | **Author:** Architecture Review (Reverse-Engineered from Live Code)  
**Scope:** End-to-end documentation of the BSC Provider Contract Intelligence Platform  
**Classification:** Internal — Engineering & Architecture

---

## TABLE OF CONTENTS

1. Executive Summary
2. Business Vision
3. End-to-End Architecture
4. Technical Deep Dive
5. Data Flow
6. LLM Pipeline Analysis
7. Contract Lifecycle Handling
8. Table-by-Table Documentation
9. Folder-by-Folder Documentation
10. Notebook-by-Notebook Documentation
11. Pipeline Stage Breakdown
12. Risks and Gaps
13. Scalability Analysis
14. Operational Readiness
15. Production Readiness
16. Multi-BlueShield Expansion Strategy
17. Recommended Improvements
18. Demo Strategy
19. KT Guide
20. FAQ — Leadership Questions

---

## 1. EXECUTIVE SUMMARY

### What This System Is

The Provider Contract Intelligence Platform (PCIP) is an AI-powered system that converts 10,372 provider contract PDFs (27 GB) into a queryable, structured intelligence layer. It transforms unstructured legal documents into actionable negotiation intelligence for Blue Shield of California's (BSC) provider contracting team.

### Key Metrics (Live Production State)

| Metric | Value |
|--------|-------|
| Total PDFs in corpus | 10,372 |
| PDFs extracted (JSON) | 10,350 (99.8%) |
| Providers represented | 302 canonical facilities |
| Rate rules extracted | 39,906 |
| Legal units indexed | 75,306 |
| Amendment chains tracked | 3,297 amendments across 3,312 contracts |
| Serving tables | 20 Genie-registered tables |
| Column comments (Genie) | 100% coverage |
| Schema | dev_adb.raw |
| LLM Model | Claude Sonnet 4.5 (200K context) |
| Orchestration | 24-task Lakeflow Job DAG |

### Business Impact

1. **Rate Benchmarking**: Every provider's rates positioned against portfolio-wide percentiles (p10-p90) enabling data-driven negotiation
2. **Amendment Intelligence**: Full supersession chains (up to 34 amendments deep) with scope-aware current-effective rate resolution
3. **Clause Discovery**: Natural-language queries against 75K legal units via hybrid retrieval (vector semantic + keyword SQL)
4. **Renewal Prioritization**: 2,219 active contracts scored and ranked by urgency, impact, and savings opportunity
5. **Self-Service Analytics**: Genie Space enables non-technical users to query contract data in plain English

### Technology Stack

| Layer | Technology |
|-------|------------|
| Compute | Databricks (Azure), Shared Cluster |
| Storage | Delta Lake (Unity Catalog) |
| LLM | Claude Sonnet 4.5 via Databricks Model Serving |
| Vector Search | Databricks Vector Search (delta sync index) |
| Embedding | databricks-gte-large-en |
| OCR | pypdf (Tier 1) + ai_parse_document (Tier 2) |
| Orchestration | Lakeflow Jobs (24-task DAG) |
| Serving | Genie Space + AI/BI Dashboard |
| Language | Python (pipeline), SQL (analytics), REST API (LLM) |

---

## 2. BUSINESS VISION

### Problem Statement

Blue Shield of California manages 3,312 provider contracts across 302 healthcare facilities. These contracts:
- Span 30+ years of amendments (oldest chain: 1990-2013, 34 amendments deep)
- Contain complex rate structures: per diem, case rates, capitation (PMPM), percentage-of-charges, DRG, fee schedules
- Include network-specific provisions (Commercial, Medicare, Medi-Cal, CalPERS, EPN, Tandem)
- Have critical termination, dispute resolution, and compliance clauses buried in dense legal text
- Are modified by amendments that may fully replace, partially update, or merely extend prior terms

**Before PCIP:** Contract analysts manually searched PDFs, maintained spreadsheets, and had no systematic way to compare rates across providers or identify upcoming renewals.

**After PCIP:** A structured intelligence layer enables:
- Instant rate comparison across all 302 providers
- Automated identification of providers with rates above the 75th percentile
- Natural-language queries: "Which providers don't have offset clauses?"
- Proactive renewal management with risk-scored prioritization
- Amendment chain tracking showing exactly which rates are current-effective

### Stakeholders

| Role | Use Case |
|------|----------|
| VP Provider Contracting | Portfolio-level rate benchmarking, renewal calendar |
| Contract Negotiators | Per-provider rate cards, competitive positioning |
| Legal/Compliance | Clause existence queries, regulatory term verification |
| Finance | Stop-loss thresholds, capitation PMPM analysis |
| Network Operations | Provider coverage, service line gaps |
| Executive Leadership | Savings opportunity quantification, risk exposure |

### Value Propositions

1. **Negotiation Leverage**: Benchmark rates at p10-p90 percentiles — identify $316K+ savings per expensive outlier
2. **Compliance Risk Reduction**: Automated clause existence verification across portfolio
3. **Operational Efficiency**: Transform weeks of manual research into seconds of natural-language queries
4. **Amendment Certainty**: Deterministic supersession logic ensures only current-effective rates are surfaced
5. **Proactive Management**: Sigmoid-scored renewal urgency with auto-renewal dampener

---

## 3. END-TO-END ARCHITECTURE

### Architectural Layers (Bottom to Top)

**Layer 1 — Source**: 10,372 PDFs in UC Volume (prod_adb/default/ext-data-volume-stmlz/Health_Plan_Ops_Transformation/Provider_Contracts). 597 provider folders. File naming: providerID_providerName_contractID_docType_version.pdf

**Layer 2 — Extraction**: 2-tier OCR + 4-prompt LLM extraction + validation + gap filling. Outputs: json_extract/ (one JSON per PDF, V7 schema, 10,825 chars schema definition).

**Layer 3 — Base Tables**: Direct JSON-to-Delta materialization. rates_all (317K), terms (69K), documents_master, financial_protections (18K), regional_factors (81K).

**Layer 4 — State Engine**: Contract registry (3,312), amendment registry (3,297), rate rules (39,906), service line taxonomy. Deterministic scope classification: FULL_REPLACEMENT, RATE_UPDATE, TERM_EXTENSION.

**Layer 5 — Semantic Layer**: 75,306 legal units chunked from contracts. Vector Search index (delta sync). Clause classification (25 types in 10 categories). Supersession graph (2,380 edges).

**Layer 6 — Intelligence Layer**: Benchmarks (percentile rates), renewal priority (sigmoid scoring), provider MDM (302 canonical, 906 aliases, 13 health systems), provider percentiles.

**Layer 7 — Serving Layer**: 4 Genie tables (pre-joined, deduplicated, 100% column comments) + 12 normalized serving tables (rate cards, stop-loss, capitation, outpatient, case rate, services coverage).

**Layer 8 — Presentation**: Genie Space (20 tables, NL queries), AI/BI Dashboard (portfolio overview), Contract_Intelligence_V2 notebook (8-branch multi-modal QA engine).

### Job DAG (24 Tasks)

The production job 'Contract Codification End-to-End' (ID: 659634711491833) runs weekly (Mon 6AM PT, currently PAUSED). Tasks form 4 parallel branches after validation:

- **Extraction Branch** (10 tasks): file_discovery → ocr → metadata → prompts → llm → validation → build_rates → build_terms → build_documents → enrich_tables → build_genie → build_serving
- **Platform Branch** (4 tasks): platform_reimbursement_migration → platform_state_engine / platform_intelligence → platform_legal_chunking
- **Semantic Branch** (4 tasks): gap_filler → clause_classification → semantic_materialization_refresh → (joins quality branch)
- **Quality Branch** (4 tasks): gap_filler → slo_checks → quality_refresh → benchmark_rescore → retrieval_eval

Convergence point: quality_audit depends on build_serving + platform_legal_chunking + platform_intelligence.

### Design Principles

1. **Checkpoint-Resumable**: Every stage writes to Parquet/Delta checkpoints; re-runs skip completed work
2. **Idempotent**: All table writes use CREATE OR REPLACE or INSERT OVERWRITE (atomic, no partial states)
3. **Zero-Dependency Reads**: Rate/term builders read directly from JSON files, not intermediate tables
4. **Additive-Only Serving**: Raw/base tables are never modified; serving tables are regenerated
5. **Version-Aware**: Extraction deduplicates by (provider_id, contract_id, doc_type, effective_date, version)
6. **Scope-Aware Supersession**: Amendments classified as FULL_REPLACEMENT, RATE_UPDATE, or TERM_EXTENSION

---

## 4. TECHNICAL DEEP DIVE

### 4.1 OCR Strategy (2-Tier)

**Tier 1: pypdf (ProcessPoolExecutor)** — 14 parallel OS processes (bypasses GIL), 30s per-page timeout. Handles ~82% of files (well-formed text-layer PDFs). Quality gate: MIN_GOOD_CHARS=500, MIN_CHARS_PER_PAGE=100, MAX_EMPTY_PAGE_RATIO=15%.

**Tier 2: ai_parse_document (Spark SQL)** — Paid API for scanned/complex PDFs. Batch size 25, concurrency 20. Processes remaining ~18%. Superior to pytesseract (removed: 150+ hours for 1,246 files vs minutes with ai_parse).

**Fallback Logic**: If Tier 2 produces fewer characters than Tier 1 partial result, keeps Tier 1 text.

### 4.2 LLM Extraction Engine

**Architecture**: Direct REST API calls to Claude Sonnet 4.5 endpoint (NOT Spark SQL ai_query — previous approach caused 10+ hour stalls).
- 15 concurrent ThreadPoolExecutor workers with per-thread HTTP sessions
- Per-file timeout (1800s) prevents cascade failures
- Circuit breaker: pauses 45s after 10 consecutive connection errors
- Session retry: 3 retries with exponential backoff on 500/502/503/504
- Manual 429 (rate limit) handling

**Token Budget**: Context window 200K tokens. max_tokens output 65,536. Effective input budget ~134K tokens. Char-to-token ratio ~2.9 chars/token.

**Chunking Strategy**: Files > 150K chars split into 80K-char segments with 2K overlap. Results deep-merged across chunks. JSON repair handles truncation at max_tokens boundary.

### 4.3 Four Document-Type Prompts

| Prompt | Target Docs | Key Extractions |
|--------|------------|------------------|
| EXTRACTION_PROMPT | Base Agreements (3,304) | Full rates, terms, parties, services, compliance |
| AMENDMENT_PROMPT | Amendments (6,754) | amendments_impact, supersession tracking, rate deltas |
| COVER_MEMO_PROMPT | Cover Memos | Exhibits replaced, routing metadata, summary of changes |
| SETTLEMENT_PROMPT | Settlements (93) | Settlement amounts, release language, confidentiality |

**Routing Logic** via _get_prompt_category(doc_type): Classifies by filename suffix/keywords.

### 4.4 V7 JSON Schema (10,825 chars)

The extraction schema covers 18 top-level sections with Delta-ready requirements (rate_numeric on every rate row, effective_date, program, network). Key additions in V7: program_carve_outs, audit_provisions, assignment_provisions, limitation_of_liability, expanded termination (12 sub-fields), expanded parties (medicaid_participation, recoupment).

### 4.5 Validation Engine

**Required Sections** (all documents): contract_overview, parties, financial_terms, covered_services, termination, compliance_and_regulatory, dispute_resolution, quality_and_performance, confidentiality_and_records.

**Type-Specific**: Base requires inpatient_rates + outpatient_rates + stop_loss. Settlements require settlement_terms. Amendments require amendments_impact.

### 4.6 JSON Repair System (src/json_repair.py)

Handles: markdown code fence stripping, trailing comma removal, unclosed bracket repair, incomplete string literals, unicode issues. Unit tested in tests/test_json_repair.py.

### 4.7 Gap Filler (Notebook 15)

Targeted re-extraction for known gaps: reads existing JSON + audit results → builds targeted prompt for ONLY missing fields → deep-merges into existing JSON → backup + validation comparison. Prioritized by worst quality score first.

---

## 5. DATA FLOW

### Source to JSON

PDF Volume → [01] File Discovery (10,372 PDFs discovered, file_registry_v2.parquet) → [02] OCR (step1_parsed_v2.parquet: filename, full_text, num_pages, extraction_method) → [03] Metadata (step2_with_metadata_v2.parquet: + provider_id, contract_id, doc_type parsed from filename) → [04] Prompts loaded → [05] LLM Extraction (10,350 JSONs in json_extract/) → [06] Validation (extraction_scores.parquet)

### JSON to Delta Tables

[08] Build Rates: json_extract/ → tbl_contract_rates_all (317K), tbl_contract_regional_factors (81K), tbl_contract_financial_protections (18K). Includes version deduplication, rate status supersession, program normalization (148 variants → 8 canonical).

[09] Build Terms: json_extract/ → tbl_contract_terms_vs_ready (69K structured rows: clauses, definitions, sections, parties, medical codes).

[10] Build Documents: json_extract/ → tbl_contract_documents_master (one row per document with parsed metadata, amendment_order, doc_category). Also builds dim_provider_canonical (facility-level entity resolution).

[11] Enrich Tables: Adds amendment_depth, scope_classification, doc_role_normalized, typed columns to documents_master and amendment_timeline.

[12] Build Genie: Pre-joins through dim_provider_canonical → 4 Genie-optimized tables with 67+ column comments.

[13] Build Serving: 12 normalized tables for dashboards (rate_card, stop_loss, capitation, case_rate, outpatient, services_coverage, dim_service_category, dim_network, etc.).

### Platform Branch (Parallel after Validation)

Reimbursement Migration: Transforms V1 rates into V2 rule format (tbl_reimb_rate_rules: 39,906 rules with valid_from/valid_to temporal bounds, formula_type, formula_json).

State Engine: Populates tbl_v2_contract_registry (3,312 contracts) and tbl_v2_amendment_registry (3,297 amendments) with scope classification.

Legal Chunking: Creates 75,306 legal units from tbl_contract_terms_vs_ready + rates. Registers VS delta sync index.

Intelligence: Computes benchmarks (percentile rates per service line) and renewal priority scores.

---

## 6. LLM PIPELINE ANALYSIS

### Model Selection Rationale

Claude Sonnet 4.5 was chosen for:
- 200K token context window (handles 90% of PDFs in a single call)
- Superior structured output (JSON) compliance
- Legal domain comprehension (complex healthcare contract language)
- Cost efficiency vs GPT-4 for batch workloads

### Prompt Engineering Strategy

1. **Role priming**: "You are an expert legal contract analyst specializing in healthcare provider agreements"
2. **Format enforcement**: "Return ONLY valid JSON — no markdown, no explanation, no code fences"
3. **Delta-ready requirements**: Explicit instructions for rate_numeric, effective_date, program, network on EVERY rate row
4. **Full clause text**: For termination, compliance, dispute resolution — "copy COMPLETE contract language verbatim"
5. **Normalization rules**: Date format (YYYY-MM-DD), dollar amounts ($ prefix + _numeric field), percentage formatting
6. **Schema-first**: Full JSON schema provided as template with field descriptions

### Extraction Quality Results (10,350 files)

- 99.8% extraction success rate (22 quarantined)
- rate_numeric population: 63% (improved from 45% via gap filler)
- full_clause_text presence: 37%
- amendments_impact presence: 98%
- Medical codes extracted: 2,545 via regex
- Unverified dates (possible hallucinations): 36

### Cost Model

- Input: $3.00 per million tokens
- Output: $15.00 per million tokens
- Average per file: ~12K input + ~8K output tokens
- Estimated total corpus cost: ~$1,500 (full 10,372 files)
- Cost per file: ~$0.15

### Retry and Recovery

1. **3a (initial)**: First pass extraction
2. **3a-retry**: Retry files that returned empty/invalid JSON
3. **3b**: Re-extract files with low extraction_score (< threshold)
4. **3c**: Chunked extraction for oversized files (> 270K chars)
5. **Gap Filler**: Post-audit targeted re-extraction for specific missing fields

---

## 7. CONTRACT LIFECYCLE HANDLING

### Document Types

| Type | Count | Handling |
|------|-------|----------|
| Base Agreements | 3,304 | Full extraction, anchor for amendment chains |
| Amendments | 6,754 | Supersession tracking, rate deltas, scope classification |
| Cover Memos | ~200 | Metadata extraction, exhibit version tracking |
| Settlements | 93 | Financial terms + release language |
| Other | 221 | Best-effort extraction with base prompt |

### Amendment Scope Classification

The state engine classifies each amendment into one of three scopes:

1. **FULL_REPLACEMENT**: Supersedes all prior rates/terms (new rate schedule entirely). All prior rates from older documents marked 'superseded'.
2. **RATE_UPDATE**: Updates specific rate lines without replacing entire schedule. Only affected rate categories superseded.
3. **TERM_EXTENSION**: Extends contract duration without modifying rates. All rates remain 'current'.

### Supersession Logic (Rate Status Resolution)

```
For each rate row:
  IF explicit status provided in extraction → use it
  ELSE IF a FULL_REPLACEMENT amendment exists with higher amendment_order → 'superseded'
  ELSE IF this doc IS a TERM_EXTENSION → 'current'
  ELSE IF this doc has max amendment_order for this provider → 'current'
  ELSE → 'superseded'
```

Safety net: Every provider gets at least 1 'current' rate (zero-current safety net prevents total supersession).

### Version Deduplication

Multiple extraction versions per file (from re-runs, gap fills) are deduplicated: keep highest _v(N) version per (provider_id, contract_id, doc_type, effective_date).

### Contract Registry Entity Model

- **Contract** = unique (provider_id × base_agreement_doc). One provider may have multiple contracts.
- **Status**: ACTIVE (2,219), EXPIRED (1,047), SUPERSEDED (40), TERMINATED (6)
- **Lineage**: base_agreement_doc → total_amendments → latest_amendment_date → latest_amendment_doc
- **Deterministic ID**: sha256(provider_id|base_agreement_doc)

### Renewal Priority Scoring

Composite score (0-1 normalized) combining:
- Urgency: Sigmoid function of days_to_expiry (steeper as expiry approaches)
- Impact: Log-scaled estimated_spend
- Rate position: Percentage of rates above median (savings opportunity)
- Auto-renewal dampener: Reduces urgency for auto-renewing contracts

Tiers: HIGH (28), MEDIUM (1,076), LOW (950), ROUTINE (165)

---

## 8. TABLE-BY-TABLE DOCUMENTATION

### Base Tables (Built from JSON)

| Table | Rows | Purpose | Key Columns |
|-------|------|---------|-------------|
| tbl_contract_rates_all | 317,609 | All rates across all documents | provider_id, rate_category, rate_numeric, status, program_normalized, is_valid_rate |
| tbl_contract_regional_factors | 81,150 | County-level geographic adjustments | provider_id, county, factor_type, factor_value |
| tbl_contract_financial_protections | 18,462 | Stop-loss, outlier, risk corridor | provider_id, protection_type, threshold_numeric, reimbursement_pct_normalized |
| tbl_contract_terms_vs_ready | 69,000+ | Clauses, definitions, sections for VS | provider_id, content_type, topic, title, content_text |
| tbl_contract_documents_master | ~10,350 | One row per document | provider_id, source_filename, doc_category, amendment_order, effective_date_parsed |
| dim_provider_canonical | ~600 | Provider entity resolution | provider_id, canonical_name, primary_provider_id, is_primary_id, ids_in_facility |

### State Engine Tables

| Table | Rows | Purpose |
|-------|------|--------|
| tbl_v2_contract_registry | 3,312 | Contract-level entity (one per agreement) |
| tbl_v2_amendment_registry | 3,297 | Amendment-to-contract linkage with scope |
| tbl_reimb_rate_rules | 39,906 | Temporal rate rules with valid_from/valid_to |
| tbl_reimb_stop_loss | 741 | Stop-loss provisions (V2 format) |
| tbl_reimb_lesser_of | 20 | Lesser-of provisions |
| dim_service_line_taxonomy | 50 | Service line classification |
| dim_service_line_patterns | 45 | Regex patterns for service line matching |

### Genie Tables (4 — Pre-joined, Deduplicated)

| Table | Rows | Purpose |
|-------|------|--------|
| tbl_genie_provider_profile | 302 | Executive provider summary (sentinel-safe, no NULLs in key fields) |
| tbl_genie_rates_current | 34,811 | Current rates only, amendment-aware dedup |
| vw_genie_contract_terms | 339,929 | VIEW — clauses + codes + parties with is_from_latest_doc flag |
| tbl_genie_amendment_timeline | 6,609 | Amendment chain with doc_category, parsed dates, rate_count |

### Serving Tables (12 — Normalized for Dashboards)

| Table | Purpose |
|-------|--------|
| dim_service_category | 26 inpatient service lines |
| tbl_serving_rate_card | Inpatient per diem by service line |
| tbl_serving_stop_loss | One stop-loss per facility |
| tbl_serving_amendment_history | Parsed dates, computed order |
| tbl_serving_parties | Deduplicated, artifacts removed |
| tbl_serving_regional_factors | Type-separated, county-level |
| tbl_serving_provider_summary | Executive profile (302 facilities) |
| dim_network | 185 raw → 12 canonical network mappings |
| tbl_serving_capitation_card | PMPM rates by age/gender band |
| tbl_serving_case_rate_card | Transplant, cardiac, complex |
| tbl_serving_outpatient_card | Surgical, therapy, infusion |
| tbl_serving_services_coverage | DOFR + quality, deduped |

### Intelligence Tables

| Table | Rows | Purpose |
|-------|------|--------|
| tbl_intel_benchmark_results | 1,802 | Percentile rates by service line (p10-p90) |
| tbl_intel_renewal_priority | 2,219 | Contract renewal scores (urgency × impact × savings) |
| tbl_intel_provider_percentiles | ~5,000 | Per-provider rate position vs benchmarks |
| dim_provider_master | 302 | MDM: canonical providers + health system mapping |
| bridge_provider_alias | 906 | Alias-to-canonical resolution |
| fact_supersession | 2,380 | Amendment supersession graph (edges) |
| dim_clause_type_taxonomy | 25 | Clause type ontology (10 categories) |

### Semantic/Retrieval Tables

| Table | Rows | Purpose |
|-------|------|--------|
| tbl_retrieval_legal_units | 75,306 | Legal semantic units for VS retrieval |
| idx_legal_units_v2 | — | Vector Search delta sync index |

### Quality/Observability Tables

| Table | Purpose |
|-------|--------|
| tbl_confidence_scores | Per-file, per-signal confidence (0-1) |
| tbl_review_queue | Tiered routing with SLA tracking |
| tbl_obs_extraction_runs | Per-run extraction stats |
| tbl_obs_quality_metrics | Periodic quality measurements |
| tbl_obs_slo_definitions | SLO target definitions |
| tbl_obs_alerts | Alert history with resolution |
| tbl_obs_cost_tracking | Daily cost estimates |
| tbl_eval_golden_queries | 20 golden queries for retrieval eval |

---

## 9. FOLDER-BY-FOLDER DOCUMENTATION

### extraction/ (16 notebooks)

The extraction pipeline — PDF to structured Delta tables. Each notebook is self-contained with %run ./_config dependency. Designed for both interactive development and Lakeflow Job orchestration.

- `_config` — Shared constants (paths, widgets, logger, quality thresholds)
- `01_file_discovery` — Scans Volume, builds file_registry, prefetches to NVMe SSD
- `02_ocr_extraction` — 2-tier OCR (pypdf multiprocess + ai_parse_document)
- `02a_reprocess_missing_pages` — Targeted reprocessing for specific files
- `03_metadata_parsing` — Extracts provider_id, contract_id, doc_type from filenames
- `04_llm_prompts` — V7 JSON schema + 4 document-type prompt templates
- `05_llm_extraction` — REST API extraction engine (15 workers, adaptive concurrency)
- `06_validation` — Schema validation, quality scoring, quarantine
- `08_build_rates` — JSON → 3 rate tables + post-processing (supersession, normalization)
- `09_build_terms` — JSON → tbl_contract_terms_vs_ready
- `10_build_documents` — JSON → tbl_contract_documents_master + dim_provider_canonical
- `11_enrich_tables` — Amendment depth, scope classification, typed columns
- `12_build_genie` — 4 Genie tables with 67+ column comments
- `13_build_serving` — 12 normalized serving tables
- `14_quality_audit` — Extraction quality assessment, gap identification
- `15_gap_filler` — Targeted re-extraction for missing data

### platform/ (20+ modules)

The intelligence platform runtime — state engine, retrieval, analytics, quality.

**Python files (legacy .py format, importable modules):**
- `00_config.py` — Constants, table names, feature flags, model endpoints
- `01_execute_ddl.py` — DDL execution (reads from sql/)
- `02_state_engine.py` — Contract + amendment registry population
- `03_reimbursement_migration.py` — V1→V2 rate rules transformation
- `04_legal_chunking.py` — Bootstrap 75K legal units + VS index registration
- `05_retrieval_engine.py` — 4-channel hybrid retrieval + RRF
- `07_citation_verification.py` — 6-check verification engine + telemetry
- `08_intelligence.py` — Benchmarks + renewal priority (real data only, fabricated removed)
- `11_evaluation_harness.py` — 20 golden queries + monitoring view

**Notebooks (Wave 3-8 implementations):**
- `12_gold_annotation` — Human review workflow: review_file(), submit_annotation(), compute_f1()
- `13_temporal_validity` — Bitemporal rate rule validation
- `14_observability` — SLO monitoring, alerting, cost tracking
- `15_governance` — Data governance and access controls
- `16_confidence` — Per-file confidence scoring (8 signals), tiered review queue
- `17_testing` — Automated testing framework
- `18_retrieval_quality` — Retrieval evaluation metrics
- `19_semantic_intelligence` — Clause classification, supersession graph, MDM, benchmarking
- `20_clause_classification` — LLM-based clause type assignment (25 types)
- `run_platform_modules_orchestrator` — Sequential module execution

### sql/ (3 DDL files)

- `state_engine_ddl.sql` — 5 tables: contract_registry, amendment_registry, rate_facts_bitemporal, document_lineage, state_computation_log
- `reimbursement_ddl.sql` — 8 tables: rate_rules, formula_components, stop_loss, carve_outs, escalators, conditions, lesser_of, + 2 dims
- `taxonomy_seed.sql` — Service line taxonomy and pattern seeds

### data/ (3 subdirectories)

- `json_extract/` — 10,350 JSON files (one per PDF, V7 schema)
- `json_consolidated/` — 456 consolidated files (legacy, now unused)
- `checkpoints/` — Parquet checkpoints for pipeline resumability

### src/ (4 Python modules)

- `__init__.py` — Package marker
- `json_repair.py` — LLM output JSON repair utilities
- `validation.py` — V6 schema validation (ValidationResult dataclass)
- `chunking.py` — Text chunking utilities for oversized documents

### tests/ (4 files)

- `__init__.py` — Package marker
- `test_json_repair.py` — Unit tests for JSON repair
- `test_validation.py` — Unit tests for validation
- `test_chunking.py` — Unit tests for chunking

### analysis_reports/ (8 Excel files)

Generated analytical reports from the V2 QA engine:
- Rate analysis, provider comparison, clause analysis, temporal analysis, dispute resolution, provider deep dive, multi-concept offset, structured extraction reports

---

## 10. NOTEBOOK-BY-NOTEBOOK DOCUMENTATION

### Extraction Notebooks (execution order)

| # | Notebook | Duration | Key Logic |
|---|----------|----------|----------|
| 01 | file_discovery | ~30s | Volume scan, cache comparison, provider-folder sampling, NVMe prefetch (32 threads) |
| 02 | ocr_extraction | ~45min | Tier 1 pypdf (14 procs), Tier 2 ai_parse_document, post-validation |
| 03 | metadata_parsing | ~2min | Filename regex: providerID_name_contractID_docType_version.pdf |
| 04 | llm_prompts | instant | JSON_SCHEMA (10,825 chars), 4 prompt templates, LLM_MODEL constant |
| 05 | llm_extraction | ~90min | REST API to Claude, 15 workers, chunking, retry, circuit breaker |
| 06 | validation | ~5min | Required sections check, type-specific validation, scoring |
| 08 | build_rates | ~6min | JSON → 3 tables, version dedup, supersession fix, program normalization |
| 09 | build_terms | ~4min | JSON → terms table (clauses, definitions, parties, codes) |
| 10 | build_documents | ~3min | JSON → documents_master + dim_provider_canonical |
| 11 | enrich_tables | ~5min | Amendment depth, scope, doc_role_normalized, typed columns |
| 12 | build_genie | ~2min | 4 Genie tables with SQL CTEs, 67 column comments |
| 13 | build_serving | ~3min | 12 normalized tables with ranking and deduplication |
| 14 | quality_audit | ~10min | Gap identification, quality scoring |
| 15 | gap_filler | ~30min | Targeted LLM re-extraction for missing fields |

### Platform Notebooks

| Module | Purpose | Key Output |
|--------|---------|------------|
| 02_state_engine | Contract/amendment registry | 3,312 + 3,297 rows |
| 03_reimbursement | Rate rules transformation | 39,906 temporal rules |
| 04_legal_chunking | Legal units + VS index | 75,306 units |
| 08_intelligence | Benchmarks + renewal | 1,802 + 2,219 rows |
| 11_evaluation | Golden query testing | 20 queries evaluated |
| 14_observability | SLO checks + alerting | Metrics + alerts |
| 16_confidence | Confidence scoring | Per-file scores + queue |
| 19_semantic | Clause ontology + MDM + supersession | Full semantic layer |
| 20_clause_class | LLM clause typing | 75K units classified |

### Intelligence Notebooks

**Contract_Intelligence_V2** — Multi-modal question answering engine with 8 branches:
- Branch A: Clause Existence (7.5-step pipeline with evidence gating)
- Branch B: Single Provider Deep Dive
- Branch C: Provider Comparison (side-by-side)
- Branch D: Rate/Data Query
- Branch H: Structured Field Extraction (28 registered fields)
- Plus: portfolio analytics, temporal queries, benchmarking

**Contract_Intelligence_Demo** — Simplified demonstration notebook.

---

## 11. PIPELINE STAGE BREAKDOWN

### Stage 1: Ingestion (Notebooks 01-02)
**Duration**: ~45 minutes for full corpus
**Bottleneck**: Tier 2 OCR (scanned PDFs)
**Failure mode**: Volume mount timeout, ai_parse rate limits
**Recovery**: Checkpoint-based; re-run skips completed files

### Stage 2: Metadata + Prompts (Notebooks 03-04)
**Duration**: ~2 minutes
**Bottleneck**: None (CPU-only parsing)
**Failure mode**: Unparseable filename format
**Recovery**: Files with unparseable metadata get default values

### Stage 3: LLM Extraction (Notebook 05)
**Duration**: ~90 minutes (15 workers, REST API)
**Bottleneck**: Claude API throughput, token costs
**Failure mode**: 429 rate limits, 5xx errors, max_tokens truncation
**Recovery**: 3 retries per file, circuit breaker, JSON repair

### Stage 4: Validation (Notebook 06)
**Duration**: ~5 minutes
**Bottleneck**: None
**Failure mode**: Schema drift in new document types
**Recovery**: Quarantine invalid files, continue pipeline

### Stage 5: Table Building (Notebooks 08-13)
**Duration**: ~20 minutes total
**Bottleneck**: Spark SQL (large CTE queries)
**Failure mode**: Schema changes in extraction format
**Recovery**: Full table regeneration (CREATE OR REPLACE)

### Stage 6: Intelligence (Platform modules)
**Duration**: ~15 minutes
**Bottleneck**: Legal chunking (75K unit creation)
**Failure mode**: Missing upstream tables
**Recovery**: DDL creates IF NOT EXISTS, INSERT OVERWRITE

### Stage 7: Quality + Semantic (Platform 14-20)
**Duration**: ~30 minutes
**Bottleneck**: Clause classification (LLM calls per unit)
**Failure mode**: Stale dependency tables
**Recovery**: Freshness gate raises RuntimeError if dependencies are old

---

## 12. RISKS AND GAPS

### Critical Risks

1. **Single-developer dependency**: Entire system built and maintained by one person. No code review, no pair programming, no secondary knowledge holder.

2. **No claims integration**: Feature flag CLAIMS_AVAILABLE=False. Provider spend, savings opportunity, and leverage scores are placeholder/estimated without actual claims data. The platform cannot quantify real dollar impact.

3. **dev_adb schema**: All production tables in development catalog. No promotion path to prod. No environment separation.

4. **Shared cluster dependency**: Job uses existing_cluster_id (not job clusters). Cluster unavailability blocks entire pipeline.

5. **LLM hallucination risk**: 36 unverified dates detected. No systematic hallucination detection beyond date range checks.

6. **Rate confidentiality**: Contract rates stored in plain Delta tables without row-level security. Exposure risk if catalog permissions misconfigured.

### Data Quality Gaps

1. **33% NULL per diem**: 99/302 providers have NULL avg_inpatient_per_diem (providers without standard per diem contracts)
2. **15% empty content_text**: vw_genie_contract_terms rows with < 10 chars
3. **37% full_clause_text**: Only 37% of extractions include verbatim clause text
4. **No reranker**: Feature flag RERANKER_DEPLOYED=False. Retrieval uses keyword + vector only (no cross-encoder)
5. **No full-text index**: Vector search covers legal_units only (75K), not raw 339K terms

### Technical Debt

1. **Mixed module formats**: platform/ has both .py files and notebooks. Inconsistent execution model.
2. **No unit test CI**: tests/ exists but no CI pipeline runs them.
3. **Hardcoded paths**: All paths reference specific user workspace.
4. **No parameterization**: Schema (dev_adb.raw) hardcoded throughout, not configurable per environment.
5. **JSON files on workspace**: 10,350 JSON files stored on workspace filesystem (not in Volume or Delta). Performance and governance risk.
6. **Legacy json_consolidated/**: 456 files from old consolidation step that no longer runs.

### Operational Risks

1. **No alerting integration**: SLO breaches write to Delta table but don't trigger PagerDuty/email.
2. **No data freshness SLA**: Pipeline schedule is PAUSED; runs are manual/ad-hoc.
3. **No disaster recovery**: No backup of json_extract/ outside workspace.
4. **Cluster cost**: Shared cluster runs 24/7 regardless of pipeline schedule.

---

## 13. SCALABILITY ANALYSIS

### What Scales Well

1. **OCR Tier 1**: ProcessPoolExecutor scales linearly with CPU cores (tested at 14 workers)
2. **LLM Extraction**: Thread-based concurrency scales with API rate limits (tested at 15 workers)
3. **Delta Tables**: All tables use Delta with optimizeWrite and changeDataFeed — designed for append/overwrite at scale
4. **Checkpoint Design**: Resume from any failure point without reprocessing
5. **Version Deduplication**: Handles multiple extraction versions gracefully

### What Does Not Scale

1. **JSON on Workspace Filesystem**: 10,350 files in a single directory. os.listdir() becomes slow at 50K+ files. No partitioning strategy.
2. **Single-threaded JSON loading**: build_rates reads all JSONs sequentially into memory (~2GB dictionary). Will fail at 50K+ files.
3. **dim_provider_canonical**: Entity resolution logic is O(n²) on provider name similarity. Will degrade with 1000+ providers.
4. **Clause Classification**: LLM call per legal unit (75K units × $0.01 per call = $750 per full refresh).
5. **Genie Table Materialization**: Full table rebuild on every run (no incremental). At 1M+ rows, will take hours.

### Scaling Recommendations

1. Move JSON files to UC Volume with provider_id partitioning
2. Use Spark to read JSONs in parallel (not Python os.listdir)
3. Implement incremental/CDC processing for downstream tables
4. Batch clause classification (multiple units per LLM call)
5. Partition tbl_contract_rates_all by provider_id for faster queries

---

## 14. OPERATIONAL READINESS

### Current State: SEMI-PRODUCTION

The system is operationally functional but not enterprise-grade:

| Dimension | Status | Gap |
|-----------|--------|-----|
| Data Pipeline | Working (99.8% extraction) | No incremental processing |
| Orchestration | 24-task DAG defined | Schedule PAUSED, manual runs |
| Monitoring | SLO framework built | No external alerting |
| Error Handling | Checkpoint + retry | No dead-letter queue |
| Logging | Structured (pipe_log) | No centralized log aggregation |
| Testing | Unit tests exist | No CI/CD, no integration tests |
| Documentation | This document + README | No runbook or playbook |
| Access Control | UC catalog-level | No row-level security |
| Backup | JSON backup (gap filler) | No systematic DR |
| Cost Tracking | Delta table | No budget alerts |

### What Works Today

1. Full pipeline runs end-to-end in ~3 hours (file_discovery through retrieval_eval)
2. Genie Space answers natural-language queries immediately
3. Dashboard shows portfolio overview
4. V2 QA engine handles 8 question types with citation verification
5. Renewal priority scoring actively identifies upcoming expirations

### What Needs Attention

1. Enable weekly schedule (currently PAUSED)
2. Add email/Slack notification on pipeline failure
3. Implement data quality checks as pipeline gates (not just reporting)
4. Add cluster auto-start/auto-terminate to reduce idle cost
5. Create runbook for common failure scenarios

---

## 15. PRODUCTION READINESS

### Production Readiness Score: 6/10

| Category | Score | Notes |
|----------|-------|-------|
| Functionality | 9/10 | Core pipeline works, all major features implemented |
| Reliability | 6/10 | Checkpointing works but no automated recovery |
| Performance | 7/10 | 3-hour end-to-end acceptable for weekly batch |
| Security | 5/10 | UC catalog-level only, no fine-grained access |
| Observability | 6/10 | SLO framework exists but no external alerting |
| Maintainability | 5/10 | Single developer, hardcoded paths, mixed formats |
| Scalability | 5/10 | JSON filesystem bottleneck at scale |
| Governance | 6/10 | Column comments 100%, but no data classification |
| Testing | 4/10 | Unit tests exist but no CI, no integration tests |
| DR/Backup | 3/10 | No systematic backup or recovery plan |

### Path to Production (Recommended)

**Phase 1 (2 weeks)**: Parameterize schema, add email alerting, enable schedule, create runbook
**Phase 2 (4 weeks)**: Move JSON to Volume, add integration tests, implement incremental processing
**Phase 3 (6 weeks)**: Promote to prod catalog, add row-level security, deploy reranker
**Phase 4 (8 weeks)**: Claims integration, full DR plan, CI/CD pipeline

---

## 16. MULTI-BLUESHIELD EXPANSION STRATEGY

### Architecture for Multi-Plan Deployment

The platform is designed for Blue Shield of California but can expand to other BSC/BCBS plans:

**What's Plan-Specific:**
- Source PDF Volume path
- Provider master data (names, IDs)
- Network names (EPN, Tandem are BSC-specific)
- Service line taxonomy (California-specific: Medi-Cal)
- Rate benchmarks (geographic market)

**What's Reusable (80%+):**
- OCR pipeline (universal PDF handling)
- LLM extraction engine (prompts are generic healthcare)
- JSON schema (V7 covers standard contract structures)
- Validation logic
- State engine (contract/amendment lifecycle)
- Legal chunking + retrieval
- Confidence scoring
- Genie Space framework
- Dashboard templates

### Multi-Tenant Architecture Recommendation

1. **Catalog-per-plan**: `bsc_ca.raw`, `bcbs_il.raw`, `bcbs_tx.raw`
2. **Shared extraction code**: Parameterized by catalog/schema
3. **Plan-specific taxonomy**: dim_service_line per plan with regional variations
4. **Cross-plan benchmarking**: Federated views across catalogs for national benchmarks
5. **Shared model serving**: Single Claude endpoint, routing by plan-specific prompts

### Expansion Estimate

- Per-plan setup: 2-3 weeks (mostly data loading + taxonomy mapping)
- Cross-plan analytics: Additional 4 weeks
- Total for 5-plan deployment: 4-5 months with dedicated team

---

## 17. RECOMMENDED IMPROVEMENTS

### Priority 1: Trust & Reliability (Immediate)

1. **Enable pipeline schedule** with failure alerting (Slack/email)
2. **Promote to prod catalog** with separate dev/staging/prod schemas
3. **Add dead-letter queue** for failed extractions (don't silently skip)
4. **Implement DR**: Backup json_extract/ to UC Volume nightly
5. **Add integration tests**: End-to-end test with 10 known-good files

### Priority 2: Data Quality (Next 30 days)

1. **Deploy cross-encoder reranker** (RERANKER_DEPLOYED=True)
2. **Connect claims feed** (CLAIMS_AVAILABLE=True) — enables real spend/savings
3. **Implement incremental processing** (process only new/changed PDFs)
4. **Add data contracts**: Schema enforcement at ingestion boundary
5. **Hallucination detection**: LLM-as-judge for extracted dates and amounts

### Priority 3: Performance & Scale (Next 60 days)

1. **Move JSON to UC Volume** with provider_id subdirectories
2. **Parallelize JSON loading** via Spark DataFrame (not Python dict)
3. **Implement liquid clustering** on large tables (rates_all, legal_units)
4. **Batch clause classification** (5-10 units per LLM call)
5. **Materialize expensive views** on schedule (vw_genie_contract_terms)

### Priority 4: Governance & Security (Next 90 days)

1. **Row-level security**: Restrict provider-specific data to authorized users
2. **Data classification**: Tag PII/PHI columns in Unity Catalog
3. **Audit logging**: Track who queries which contract data
4. **Rate confidentiality controls**: Ensure competitive rate data is access-controlled
5. **Retention policy**: Define lifecycle for superseded extractions

---

## 18. DEMO STRATEGY

### Demo Audiences & Approaches

**Executive Demo (15 min)**:
1. Open Genie Space → ask "Which providers have the highest inpatient per diem rates?"
2. Show portfolio dashboard (rate distribution, renewal calendar)
3. Ask "Which providers are up for renewal in the next 90 days with rates above median?"
4. Show savings opportunity table (one slide: total addressable savings)

**Contracting Team Demo (30 min)**:
1. Provider deep dive: "Show me everything about [Provider X]"
2. Rate comparison: "Compare inpatient rates between [Provider A] and [Provider B]"
3. Clause query: "Which providers don't have termination for convenience clauses?"
4. Amendment timeline: Show how rates evolved over 10+ amendments
5. Rate card export to Excel

**Technical Demo (45 min)**:
1. Run pipeline end-to-end on 5-provider sample
2. Show JSON extraction output (before/after gap fill)
3. Walk through supersession logic with real amendment chain
4. Demonstrate vector search retrieval with citation verification
5. Show observability dashboard (SLOs, cost tracking)

### Demo Assets

- Genie Space: BSC Provider Contract Intelligence (20 tables)
- Dashboard: BSC Provider Contract Intelligence - Portfolio Overview
- V2 QA Engine: Contract_Intelligence_V2 notebook (8 branches)
- Analysis Reports: 8 Excel exports in analysis_reports/

---

## 19. KT GUIDE (Knowledge Transfer)

### For New Engineers (Week 1)

1. Read this document (sections 1-5 for context)
2. Run _config notebook to understand paths and constants
3. Run 01_file_discovery in sample mode (5 providers)
4. Examine a JSON extraction file (json_extract/*.json)
5. Query tbl_genie_provider_profile in SQL editor
6. Ask a question in the Genie Space

### For New Engineers (Week 2)

1. Read 04_llm_prompts to understand V7 schema
2. Run 08_build_rates to see JSON → Delta transformation
3. Read 02_state_engine.py to understand contract lifecycle
4. Run 14_observability to see SLO status
5. Try Contract_Intelligence_V2 with different question types

### Key Concepts to Understand

1. **Amendment supersession**: How amendments create chains and supersede prior rates
2. **Scope classification**: FULL_REPLACEMENT vs RATE_UPDATE vs TERM_EXTENSION
3. **Provider canonical resolution**: Multiple provider_ids map to one facility
4. **Program normalization**: 148 raw variants → 8 canonical (Commercial, Medicare, etc.)
5. **is_from_latest_doc**: Flag indicating current-effective document version
6. **doc_role_normalized**: Authoritative document role for filtering

### Critical Files to Know

1. `extraction/_config` — All shared constants
2. `extraction/04_llm_prompts` — JSON schema + prompts (controls extraction quality)
3. `extraction/08_build_rates` — Rate table construction (most complex transformation)
4. `platform/00_config.py` — Platform constants and feature flags
5. `sql/state_engine_ddl.sql` — State engine schema definition

### Troubleshooting Guide

| Symptom | Likely Cause | Fix |
|---------|-------------|-----|
| Pipeline fails at ocr_extraction | Cluster terminated | Restart cluster, re-run |
| LLM extraction stalls | Rate limiting (429) | Reduce REST_CONCURRENCY to 5 |
| Empty rates table | JSON schema changed | Check extraction JSONs match expected paths |
| Genie returns wrong rates | Supersession not applied | Re-run 11a-fix (rate status) |
| Provider missing from profile | Not in dim_provider_canonical | Check 10_build_documents |
| Quality audit shows gaps | Normal behavior | Run 15_gap_filler on affected files |

---

## 20. FAQ — LEADERSHIP QUESTIONS

### Q: How accurate is the extraction?
**A:** 99.8% of PDFs are successfully extracted. Rate_numeric population is 63%. We validate against 9 required sections with type-specific checks. The confidence scoring system routes uncertain extractions to human review queues. 36 dates flagged as potentially hallucinated out of millions extracted.

### Q: What's the total cost to run this?
**A:** Estimated $1,500 for full corpus LLM extraction (one-time). Ongoing cost: ~$50/week for incremental processing + cluster compute. Total platform cost including compute: ~$500/month.

### Q: How long until we can show this to the contracting team?
**A:** The Genie Space and Dashboard are functional today. They can answer questions about all 302 providers immediately. The V2 QA engine handles 8 question types with citation-backed answers.

### Q: Can we trust the rates it shows?
**A:** Rates are extracted directly from PDFs and validated. The supersession engine ensures only current-effective rates are surfaced in Genie tables. We track amendment chains up to 34 deep. However, without claims data integration, we cannot verify rates match actual payment amounts.

### Q: What happens when new contracts arrive?
**A:** New PDFs added to the Volume are detected automatically on next pipeline run. The checkpoint system ensures only new files are processed. End-to-end processing for a new batch: ~90 minutes.

### Q: How does this compare to commercial tools (Icertis, Agiloft)?
**A:** PCIP is purpose-built for healthcare provider contract intelligence — a narrow domain commercial tools don't specialize in. Key advantages: (1) Rate-specific extraction with program/network granularity, (2) Amendment supersession logic, (3) Integrated with existing Databricks ecosystem (no data movement), (4) No per-contract licensing fees. Disadvantage: Higher maintenance burden (custom code vs SaaS).

### Q: What's the risk of deploying this to production?
**A:** Primary risks: (1) Rate confidentiality — need row-level security before broader access, (2) Single developer dependency — need knowledge transfer and second engineer, (3) No claims validation — rates are extracted-as-stated, not verified against payments, (4) dev catalog — needs promotion to prod with proper governance.

### Q: Can this work for other Blue Shield/BCBS plans?
**A:** Yes. 80%+ of the platform is reusable (OCR, LLM, schema, state engine, retrieval). Per-plan customization needed: taxonomy mapping, network names, source data. Estimated 2-3 weeks per new plan. Cross-plan national benchmarking possible with federated views.

### Q: What's the maintenance burden?
**A:** Weekly pipeline runs are automated (when schedule is enabled). Primary maintenance: (1) Address SLO breaches from quality monitoring, (2) Update prompts if extraction quality drifts, (3) Refresh benchmarks after claims integration, (4) Handle edge cases in new document formats. Estimated: 4-8 hours/week of engineering time.

### Q: What would you build differently if starting over?
**A:** (1) Store JSON outputs in UC Volume from day one (not workspace filesystem), (2) Use Spark Declarative Pipelines for the table building layer, (3) Implement incremental CDC processing from the start, (4) Deploy to prod catalog with proper dev/staging/prod separation, (5) Build CI/CD pipeline for code changes, (6) Use structured extraction with constrained decoding instead of free-form JSON.

---

## APPENDIX A: HEALTH SYSTEM MAPPING

| Health System | Providers | Total Contracts | Total Rates |
|--------------|-----------|----------------|-------------|
| Independent | 259 | 5,056 | 293,787 |
| Adventist Health | 16 | 367 | 18,685 |
| Providence | 10 | 641 | 47,384 |
| Sharp HealthCare | 3 | 125 | 3,940 |
| Sutter Health | 2 | 112 | 11,027 |
| Stanford Health Care | 2 | 31 | 2,551 |
| Scripps Health | 2 | 61 | 2,204 |
| CommonSpirit Health | 1 | 44 | 3,758 |
| UCLA Health | 1 | 21 | 1,590 |
| UCSF Health | 1 | 36 | 3,163 |
| Cedars-Sinai | 1 | 24 | 1,098 |

## APPENDIX B: CLAUSE TYPE ONTOLOGY (Top 10 by Volume)

| Category | Clause Type | Legal Units | Significance |
|----------|------------|------------|-------------|
| financial | reimbursement_methodology | 64,710 | HIGH |
| scope | covered_services | 4,290 | HIGH |
| termination | termination_for_convenience | 1,390 | HIGH |
| dispute | dispute_resolution | 912 | HIGH |
| financial | stop_loss | 488 | HIGH |
| compliance | hipaa | 457 | HIGH |
| renewal | auto_renewal | 420 | HIGH |
| financial | lesser_of | 357 | MEDIUM |
| termination | termination_for_cause | 288 | HIGH |
| financial | timely_filing | 252 | MEDIUM |

## APPENDIX C: LONGEST AMENDMENT CHAINS

| Provider | Chain Depth | Span (Years) | Start | End |
|----------|------------|-------------|-------|-----|
| 129362433 | 34 | 23 | 1990 | 2013 |
| 129228693 | 29 | 5 | 2009 | 2014 |
| 129349705 | 21 | 16 | 1997 | 2013 |
| 129364224 | 20 | 23 | 1990 | 2013 |
| 240328764 | 17 | 7 | 2011 | 2018 |
| 129356192 | 17 | 23 | 1991 | 2014 |

---

*Document generated by architecture review, June 2026. Source of truth: live code in Contract_Codification_Pipeline/.*