# Contract Intelligence Platform — Strategic Response & Productionalization Roadmap

**Prepared for:** Meeting — Contract Codification Tool: Next Steps
**Prepared by:** Ankit Dixit, Data Engineering
**Date:** 2026-07-21
**Attendees:** Michael Simpson's Team, Network Strategy Team, Krista (Provider Contract Codification SME)

> **Purpose of this document:** This document responds directly to the questions raised in the meeting email, provides an honest assessment of where the Contract Intelligence Platform stands today, makes a clear architecture recommendation for enterprise scale, and defines a realistic productionalization roadmap — for BSC internal use and, longer term, for commercial expansion.

---

## A Note Before We Begin

Everything described in this document — the application, the data model, the AI pipeline, the extraction infrastructure — was built by a single person, without formal business requirements, without a dedicated team, and without budget for external tooling or vendors. It was built in production on the Databricks platform using real BSC contract data.

This context is not shared to ask for recognition. It is shared because it is directly relevant to the roadmap: the platform is further along than any externally-procured solution would be at this stage, and the gaps that exist are well-understood, well-documented, and fixable with the right resources.

What follows is an honest picture — what works, what doesn't, and what it realistically takes to make this enterprise-grade.

---

## Section 1 — What Was Built

The Contract Intelligence Platform V4 is a purpose-built, AI-powered contract knowledge system that transforms raw provider contract PDFs into structured, queryable, and actionable intelligence.

### The Scale

These are verified, live figures from the production database (`dev_adb.raw`) as of 2026-07-21:

| What | Count |
|---|---|
| Provider contracts processed | **10,349 PDF documents** |
| Providers covered | **306 providers** (194 active, 112 inactive) |
| Major California health systems mapped | **20** (Kaiser, Dignity, Sutter, UCSF, Stanford, Cedars, and more) |
| Reimbursement rate rows | **459,902** |
| Current / active rates | **171,763** |
| Historical / superseded rates | **288,139** (full amendment history) |
| Structured clause extractions | **7,863** across 21 legal categories |
| AI-searchable text chunks | **727,876** (every word of every contract) |
| Compliance regulations tracked | **7** (KNOX-KEENE, CMS-MA, DMHC, AB-1455, SB-137, AB-352, AB-72) |
| Risk scores computed | **272** providers scored across 19 risk dimensions |
| Critical alerts currently open | **229** requiring immediate attention |
| Contract deadline events tracked | **713** |
| Repricing scenarios pre-computed | **7,600** (four what-if scenarios per provider) |
| Production Delta tables | **31** structured tables |
| Total rows across all tables | **~1.3 million** |
| Data quality sentinels | **10 automated checks** running continuously |

### What a User Can Do Today

A user with access to the application can:

- Ask any natural-language question about any provider contract and receive a structured, sourced answer in 3–5 seconds — *"What are the current inpatient DRG rates for Sutter Health?"* or *"Does Adventist Health have an offset clause?"*
- Browse all 306 providers with full contract metadata, rate tables, amendment timelines, and clause coverage
- Generate a classification report across any set of providers showing which contracts contain specific clauses (offset, DOFR, IPA delegation, AB-352, auto-renewal)
- Review 351 open contract alerts — 229 of which are CRITICAL — with deadline tracking
- Track compliance status across 7 regulations for every active provider
- Access the source PDF for any AI-generated answer with a single click, landing on the exact page the answer came from
- Export any conversation, report, or data slice to Excel, JSON, or Markdown

### What Was Built Is Not a Prototype

This is a working system with real data, a production data model, automated quality monitoring, a full React frontend, a FastAPI backend with 30+ endpoints, a ReAct AI agent with 25+ specialized tools, and a comprehensive test suite with 99 validated questions.

The question is not whether it works. The question is how to make it enterprise-grade, and on what infrastructure.

---

## Section 2 — Answering the Email: Short-Term Access Provisioning

> *"Short term, we would like to know what is required for you to host the application and provision access to specific users to explore and use the tool."*

### The Application Is Already Running

The application is currently live and accessible as a Databricks App — a hosted web application running within the BSC Databricks workspace. There is no separate server to stand up, no deployment to execute. The infrastructure is already there.

### What "Provisioning Access" Requires Today

Access operates at two layers:

**Layer 1: Databricks Workspace Access**
Every user needs a Databricks workspace account. This is a standard IT provisioning request. Once a user has workspace access, they can be granted read-only permissions to the application's data catalog (`dev_adb.raw`) via Unity Catalog.

**Layer 2: Application URL Access**
The application runs at a Databricks Apps URL tied to the workspace. Any authenticated workspace member can navigate to the URL and use the full application.

### What a Provisioned User Can See and Do

Once provisioned, a user gets full access to:
- The AI chat interface (all 25+ AI tools, full contract corpus)
- Provider Explorer (all 306 providers)
- Reports (classification matrix generator)
- Alerts and Deadlines dashboard
- Compliance tracker
- Document viewer (with PDF source verification)

**What they cannot do today:**
- They cannot modify data — the application is read-only by design
- They do not need any Databricks knowledge — the interface is a standard web application
- There is no team-level access differentiation today (all provisioned users see the same application)

### Honest Limitation: No In-App Role Separation

Today, all provisioned users have identical access. Michael Simpson's team, the Network team, and Provider Relations would all see the same application with the same features and the same data.

If the business requires that different teams see different data, or that certain sensitive fields (e.g., specific financial terms) are hidden from some users, **that requires building an application-level role-based access control (RBAC) layer** — which does not exist yet. This is on the roadmap (Section 8) but is not available for the initial pilot.

### Recommended Pilot Approach

**What:** Provision 10–15 users across two teams for a structured 3-week User Acceptance Testing (UAT) session
**Who:** 4–5 from Michael Simpson's team (offset clause and metadata use case), 4–5 from Network team (global contract intelligence exploration), 2–3 from Provider Relations
**When:** Can begin within 2 weeks of receiving the user list and workspace access requests being processed
**Format:** Structured UAT sessions with a test script, plus open exploration time
**What we learn:** Which questions the AI gets right, which it gets wrong, which features matter most, and what the attribute taxonomy should look like (Krista's input is critical here)

### Timeline and Effort

| Step | Owner | Effort | Calendar Time |
|---|---|---|---|
| Compile user list | Business teams | — | Day 1 |
| IT: provision Databricks workspace accounts | IT | Low | 3–5 business days |
| Grant catalog read permissions | Data Engineering | < 1 hour | Day 1 (can be done same day as accounts) |
| UAT session scheduling and test script | Data Engineering + Business | 3–4 days | Week 1–2 |
| UAT execution | Business users | 2–3 hours per session | Weeks 2–4 |

**Bottom line: the first users can be using the application within 2 weeks. No new build is required.**



---

## Section 3 — Answering the Email: Long-Term Bulk Extraction Automation

> *"Long term, I am curious what it would take to automate the extraction process in bulk, by running a set of search terms through it, capturing the results, and storing those somewhere (for example, Databricks)."*

### Important Reframe: The Storage Part Is Already Done

This question assumes results still need to be extracted and stored. They don't — not for the contracts already in the system. **~1.3 million rows of structured contract knowledge are already extracted and stored in 31 Delta tables in Databricks.** Any downstream system can query this data today via SQL, REST API, or direct Delta table access.

What "automate in bulk" means in this architecture breaks into three capabilities:

### Capability 1: New Document Ingestion Automation

**Current state:** The extraction pipeline exists and runs successfully on all 10,349 documents, but requires a manual notebook trigger. No automation exists.

**What needs to be built:** A Databricks Job with a file-arrival trigger on the contract PDF volume. When a new PDF is added, the job runs OCR, chunking, structuring, and vector index sync automatically. Results land in the existing 31 tables. A notification is sent on completion or failure.

**Estimated effort:** 2-3 weeks. Standard Databricks Jobs configuration, not a new build.

### Capability 2: Bulk Concept Extraction on Demand

**Current state:** The application's Report Generator already does this for 6 predefined concepts on demand. A scheduled or API-triggered batch job for arbitrary concept lists does not yet exist.

**What the workflow looks like:**
1. Business provides a concept list (offset clause, DOFR language, quality withhold %, auto-renewal terms)
2. The batch job runs each concept against every provider's contract corpus
3. Results are written to the existing `tbl_contract_clauses` table
4. Every result includes source document, page number, and extracted text for auditability

This is the global contract intelligence vision in operational terms: extract once, store once, reuse everywhere.

**Estimated effort:** 4-6 weeks for a robust batch extraction API with scheduling.

### Capability 3: Downstream System Integration

**Current state:** Data is already in Delta tables. Any BSC system that can connect to Databricks (JDBC, ODBC, Statement Execution API, or direct Delta access) can read it today. The REST API (30+ endpoints) provides an additional path requiring no Databricks knowledge on the consuming side.

**What needs to be built per integration:** A formal data contract (agreed schema, valid values, refresh SLA), and possibly a dedicated export table shaped for a specific team's schema.

**Estimated effort:** 2-4 weeks per downstream system.

### Summary

| Capability | Current State | What Is Missing | Effort |
|---|---|---|---|
| New document ingestion | Manual pipeline run | Automated trigger and job | 2-3 weeks |
| Bulk concept extraction | On-demand via app UI | Scheduled batch API | 4-6 weeks |
| Downstream system access | Data live in Delta tables | Data contracts and integration | 2-4 weeks per system |

**The storage question is answered: Databricks is already the store. The remaining work is automation, scheduling, and integration plumbing.**

---

## Section 4 — The "Global Contract Intelligence" Vision: Network Team Alignment

> *"They want to create a standardized set of contract attributes that can be digitized once and then reused across Utilization Management, Care Management, Provider Relations, Analytics and other downstream functions."*

This is not a new vision. This is a precise description of what was built.

### What We Have That Directly Delivers This Vision

**Digitized once:** Every attribute was extracted from source PDFs exactly once. It lives in Delta tables. Any downstream function reads from the same source. No duplication, no team-specific versions, no manual re-extraction.

**Standardized attributes:** 21 standardized clause categories, 7 compliance regulations, 4 risk scoring dimensions, and a canonical provider identity key that every table joins on. These are governed schema columns with defined valid values.

**The canonical provider key** (`dim_provider_canonical`, 531 entries including aliases) is the cornerstone. UM's reference to "Cedars-Sinai," Care Management's "Cedars-Sinai Medical Center," and Provider Relations' "CSMC" all resolve to the same provider record. One identity. No ambiguity across teams.

### How "Reused Across Functions" Works Technically

| Function | What They Access | Access Method |
|---|---|---|
| Provider Relations | Clause coverage, offset terms, IPA delegation, amendment history | Application UI or REST API |
| Utilization Management | DOFR responsibility matrix, IPA delegation scope, pre-auth clause language | SQL or REST API `/api/v1/network/{provider}/membership` |
| Care Management | Delegation terms, capitation rates, IPA scope | SQL on `tbl_delegation_matrix` and `tbl_contract_rates_all` |
| Finance | Current rates, rate history, financial exposure, repricing scenarios | REST API `/api/v1/financial/exposure/{name}` or SQL |
| Analytics | All 31 tables | Direct Delta access via Databricks SQL, Power BI, or Spark |
| Compliance | Regulation tracking per provider | REST API `/api/v1/compliance/{name}` |

All of the above are powered by one extraction run. One source of truth.

### What Is Still Missing for the Full Vision

**Gap 1: The attribute taxonomy has not been formally agreed with the business.**
Attributes extracted to date were determined by what was technically extractable and obviously valuable from a data engineering perspective, not from a structured business requirements process. The Network team's "standardized set" may include things not yet extracted. The most important output of this meeting is a first draft of that attribute list, led by Krista.

**Gap 2: Some attributes needed by UM and Care Management are not yet structured.**
Likely gaps based on common UM and Care Management use cases:
- Value-based care and shared savings terms
- Specific pre-authorization clause language
- Dispute resolution and appeal timelines
- Provider-specific performance guarantees
- Credentialing and recredentialing requirements

These may exist in the 727,876 searchable text chunks but have not been extracted into discrete structured columns. Structuring them requires a targeted pipeline run, not a full re-extraction.

**Gap 3: No formal data contracts for downstream consumers.**
Before UM or Care Management build workflows on top of this data, they need agreed column names, valid values, refresh schedule, and a versioning policy. These data contracts do not exist yet.

### The Strategic Ask from This Section

The most productive use of Krista's time in this meeting is to review the 21 current clause categories and answer: *What is here that we need? What is missing that we need? What is here that we do not need?*

That conversation produces the official attribute taxonomy and drives all remaining extraction work. Without it, we risk building toward a definition of "contract intelligence" that does not match what the business actually means.

---

## Section 5 — Honest Assessment: Where the Project Stands Today

The project has received positive feedback from the business. That feedback is earned — the scale and depth of what was built is real. But the honest picture also includes what is conditional, what has known gaps, and what needs to happen before this can be called a production system that business teams depend on daily.

### What Is Production-Quality Today

The following components are stable, tested, and working with real production data:

**The data model:**
- 31 Delta tables with verified row counts and continuous quality monitoring
- 10 of 10 automated regression sentinels passing
- Full referential integrity across all critical foreign key relationships
- Temporal labeling applied — 0 stale CURRENT rates, full amendment history preserved

**The AI application:**
- 80%+ of answers grade MODERATE or HIGH confidence (tested against 99 validated questions)
- Hallucination rate: 5.63% (below the 10% production threshold)
- 8-page React frontend fully functional
- 30+ REST API endpoints all live and tested
- 25+ specialized AI tools covering every contract question category
- Source PDF verification: every AI answer links to the exact source page

**The infrastructure:**
- Fail-open startup with 3 independent circuit breakers (SQL, Vector Search, LLM)
- Fire-and-forget telemetry for every query
- Session persistence across conversations
- Rate limiting, PII redaction, SQL injection prevention, and path traversal protection

### What Is Conditional / Needs Work Before Enterprise Rollout

The Phase 5 production hardening audit produced an honest grade: **17 of 21 metrics passing (80.95%) — CONDITIONAL.** This means the system is operationally monitorable and regression-safe, but four issues need resolution before it can be called fully production-ready:

| Issue | Current State | Target | Impact |
|---|---|---|---|
| Broken amendment chains | 62.37% of providers have broken chains | Less than 5% | Temporal analysis and rate history may be incomplete for these providers |
| Rate outliers | 16,548 rows with implausible values | Less than 200 | Specific rate queries may return erroneous data for affected rows |
| DOFR confirmation rate | 70% (heuristic match) | 75% minimum | DOFR responsibility answers for borderline cases may be unreliable |
| Consistency score | 75% | 90% | Cross-provider comparison answers may have minor inconsistencies |

**These are known, scoped, and fixable.** They are not surprises. They are the result of a thorough audit that documented every issue with its root cause. The broken chain issue, for example, stems from providers having multiple parallel amendment chains under the same document category — a structural issue in the source documents that requires a chain-ID backfill in the extraction pipeline.

### Additional Known Data Gaps

Beyond the four metrics, three external data gaps remain open:

- **232 contract documents are missing expiration dates.** The dates exist in the PDF text but were not captured during OCR. A targeted re-extraction pass is required. This gap affects deadline tracking accuracy for those contracts.
- **12 providers are missing their base contract PDFs.** Nine of these have amendments on file but the base document was never added to the volume. Rate and clause data for these providers is derived from amendments only.
- **External benchmark data is incomplete.** The rate benchmark table (`tbl_rate_benchmarks`) has 10,434 rows but still needs Medicare IPPS/OPPS rates, Medi-Cal FFS schedule, and commercial survey data to support meaningful rate comparison analysis.

### UAT Has Not Yet Happened

The application has been demonstrated but has not been through structured User Acceptance Testing with actual business users doing actual business tasks. This is the most important remaining validation step before production rollout. The demo validates that the system can answer questions. UAT validates that it answers the questions business users actually need to answer, in the way they need them answered.

### The Honest Bottom Line

The system is impressive as a proof of capability and further along than any comparable procurement would be. For real business use at scale, it needs:
1. The four failing metrics resolved
2. The 232 expiry date gaps patched
3. Formal UAT completed with Michael Simpson's team and the Network team
4. Enterprise authentication (not just Databricks workspace accounts)

That is approximately 8-12 weeks of focused work. The work is well-understood. The architecture is sound. The data is real.

---

## Section 6 — The Architecture Decision: Databricks Apps vs. Enterprise Hosting

This is the most consequential technical decision facing the project. The answer determines whether the platform can scale to enterprise use, support non-Databricks users, and ultimately be positioned for commercial expansion.

### Current Architecture: What It Is

The application runs as a **Databricks App** — a hosted web container within the BSC Databricks workspace. The FastAPI backend and React frontend run on a single container. The data layer (Delta tables, Vector Search index, LLM endpoints, SQL Warehouse) also runs within Databricks. Everything lives in one place.

This was the right choice for development. It is not the right long-term choice for enterprise scale.

### Where Databricks Apps Falls Short at Enterprise Scale

| Limitation | What It Means in Practice |
|---|---|
| Single container, no auto-scaling | As concurrent users grow beyond ~50, response times degrade. No way to add capacity without manual intervention. |
| Authentication tied to Databricks workspace | Every user needs a Databricks workspace account. External partners, non-technical business users, and users from other health plans cannot be onboarded without Databricks accounts. |
| No fine-grained in-app RBAC | All users see the same application. There is no way to show Provider Relations a different view than Finance without building auth from scratch. |
| No custom domain | The app URL contains Databricks workspace identifiers. Not suitable for a product presented to business users or external clients. |
| No Web Application Firewall | Enterprise security requirements typically mandate WAF coverage. Databricks Apps does not provide this. |
| No CDN for frontend | The React frontend is served directly from the Databricks container, not from a globally distributed CDN. Performance degrades for users far from the Azure region. |
| Not designed for external clients | If the goal is to sell this product to other health plans, the application must run independently of any client's Databricks workspace. Databricks Apps makes this structurally impossible. |

### The Recommendation: A Hybrid Architecture

The answer is not "move everything off Databricks." Moving the data off Databricks would be the wrong decision — Databricks is exactly the right platform for the data and AI layer.

The answer is: **keep the data and AI on Databricks, move the application to Azure-hosted containers.**

```
STAYS IN DATABRICKS (permanently — this is the correct long-term architecture):

  Delta tables           31 tables, ~1.3M rows, governed by Unity Catalog
  Vector Search index    727,876 chunks, semantic retrieval, GTE-1024d embeddings
  LLM endpoints          Claude Sonnet 4.6 and Haiku 4.5 via Databricks Model Serving
  SQL Warehouse          Rate queries, structured lookups, sub-second response
  Genie Space            Natural language SQL alternative path
  Unity Catalog          Data governance, lineage, access control, permissions
  Extraction pipeline    OCR, chunking, structuring, quality monitoring
  Data quality views     10 regression sentinels, 4 monitoring views

MOVES TO AZURE (application tier only):

  FastAPI backend        Azure Container Apps (auto-scaling, 0-to-N instances)
  React frontend         Azure Static Web Apps (CDN, global edge, fast everywhere)
  Authentication         Azure Active Directory / Entra ID (SSO, MFA, zero new accounts)
  Application RBAC       Built into the application layer (team-level feature access)
  WAF and routing        Azure Front Door (DDoS protection, WAF, custom domain)
  Monitoring             Azure Monitor + Application Insights
```

This is a standard enterprise architecture pattern: the intelligence platform runs on Databricks, the user-facing application runs on Azure, and the two talk through well-defined APIs.

### Why This Works Without Code Changes

The FastAPI backend already calls Databricks entirely through REST APIs:
- SQL queries: Databricks Statement Execution API
- Vector search: Databricks Vector Search SDK (REST-based)
- LLM calls: Model Serving REST endpoints
- Genie: Genie Conversation REST API

Moving the FastAPI container from Databricks Apps to Azure Container Apps requires: (a) packaging as a Docker container, (b) configuring Databricks OAuth service principal credentials as Azure Key Vault secrets, and (c) deploying to Azure Container Apps. **The application code does not change.** It does not know or care where it is running — it only knows the Databricks endpoint URLs.

### Why This Is Critical for Commercial Expansion

If the goal is to sell this product to other health plans, the Databricks Apps model makes this structurally impossible — each client would need to run the application inside their own Databricks workspace, and BSC would have no visibility into or control over the deployment.

The Azure Container Apps model makes commercial expansion straightforward: one application deployment serves all clients. Each client gets their own Unity Catalog schema for data isolation (`client_x.raw.*`), but the application infrastructure is shared. This is standard SaaS multi-tenancy.

### Phased Migration Plan

Migrating immediately is not recommended. The right approach is phased:

| Phase | Compute for Application | Rationale | User Count |
|---|---|---|---|
| Now (pilot) | Databricks Apps | Sufficient for 10-30 concurrent users; no migration risk during UAT | 10-30 |
| Month 3-6 | Azure Container Apps | When user base grows beyond pilot, SSO becomes a hard requirement | 30-200 |
| Month 9+ | Azure Container Apps + multi-tenant | When commercial expansion begins | 200+ |

**The data layer does not change at any phase.** Databricks remains the data platform permanently.

---

## Section 7 — What We Are Missing for Enterprise Grade

A gap analysis across 10 dimensions. Each gap has a severity, a realistic effort estimate, and a specific description of what "fixed" looks like.

| # | Gap | Severity | Effort | What "Fixed" Looks Like |
|---|---|---|---|---|
| 1 | Enterprise SSO / Azure AD integration | Critical | 3-4 weeks | Users authenticate with their BSC Active Directory credentials. No Databricks account needed. MFA is inherited from corporate policy. |
| 2 | In-app role-based access control | High | 4-6 weeks | Provider Relations, Finance, and Compliance see different feature sets and potentially different data subsets. Roles are assigned by team. |
| 3 | Four failing quality metrics | High | 6-8 weeks | Broken amendment chains below 5%, rate outliers below 200, DOFR confirmation above 75%, consistency score above 90%. |
| 4 | Formal UAT with end users | High | 4 weeks | Structured testing sessions with Michael Simpson's team and Network team produce a documented pass/fail scorecard and a prioritized issue list. |
| 5 | Pipeline automation | High | 2-3 weeks | New contracts trigger automatic ingestion. No manual notebook runs. Email notification on success or failure. |
| 6 | SLA governance and incident response | High | 2-3 weeks | Formal SLAs documented. On-call process defined. Alert routing configured (PagerDuty or equivalent). |
| 7 | API versioning and data contracts | Medium | 3-4 weeks | Every REST endpoint has a version prefix. Schema changes follow a deprecation policy. Downstream consumers are notified of breaking changes. |
| 8 | Security: penetration test and SOC 2 readiness | Medium | 8-12 weeks | External pen test completed. Security findings remediated. SOC 2 Type II audit scoped and initiated. Required for any enterprise customer contract. |
| 9 | UX research and UI refinement | Medium | 6-8 weeks | User research sessions with actual users produce a prioritized UX improvement backlog. The interface moves from demo-quality to enterprise-quality. |
| 10 | Multi-tenancy for commercial expansion | Lower (later) | 4-6 months | Schema-per-client data isolation. Tenant routing in the application. Billing and metering. Self-service onboarding. Client admin portal. |

### What This Means for Timeline

Items 1-6 are prerequisites for production rollout. They need to be complete before the platform is used by more than a small pilot group or relied on for business decisions. Items 7-9 should be completed in parallel with the pilot. Item 10 is a separate workstream that begins after BSC internal production is stable.

---

## Section 8 — Productionalization Roadmap: BSC Internal

Three phases to move from "impressive demo" to "BSC production system."

### Phase 1 — Validate (Weeks 1-6)

**Goal:** Confirm the platform answers the questions business users actually need to answer. Agree on the attribute taxonomy. Identify the highest-priority gaps.

| Activity | Owner | Output |
|---|---|---|
| Provision pilot user accounts (10-15 users) | IT + Data Engineering | Users can access the application |
| Structured UAT sessions (3 rounds) | Business users + Data Engineering | Pass/fail scorecard, issue list, question gaps |
| Attribute taxonomy workshop with Krista | Network team + Data Engineering | Official list of target contract attributes |
| Triage 4 failing metrics | Data Engineering | Decision: fix now vs. accept risk for pilot |
| Document question types that fail | UAT participants | Prioritized backlog for Phase 2 |

**Success criteria:** UAT scorecard shows 85%+ pass rate. Attribute taxonomy is documented and agreed.

### Phase 2 — Solidify (Weeks 7-16)

**Goal:** Close the non-negotiable quality and infrastructure gaps. Make the platform safe for broader business use.

| Activity | Owner | Output |
|---|---|---|
| Fix broken amendment chains | Data Engineering | Broken chain rate below 5% |
| Fix rate outliers | Data Engineering | Outlier count below 200 |
| Patch 232 missing expiry dates | Data Engineering | Expiry date coverage above 98% |
| Build pipeline automation | Data Engineering | File-arrival trigger on contract volume |
| Implement Azure AD / Entra ID SSO | Data Engineering + IT | No Databricks accounts required |
| Build monitoring and alerting | Data Engineering | PagerDuty or equivalent, alert routing |
| Extract missing attributes from taxonomy | Data Engineering | New concepts structured into Delta tables |
| Formal load testing | Data Engineering | 50 concurrent users tested and validated |
| API versioning and data contracts | Data Engineering | v1 contracts documented and frozen |

**Success criteria:** All 10 quality metrics passing. SSO live. Pipeline automated. Load test passed.

### Phase 3 — Productionize (Weeks 17-28)

**Goal:** Move the application to enterprise-grade infrastructure. Enable cross-team use at scale. Complete first downstream integration.

| Activity | Owner | Output |
|---|---|---|
| Migrate application to Azure Container Apps | Data Engineering | Auto-scaling, custom domain, CDN |
| Implement full RBAC | Data Engineering | Team-level feature and data access control |
| Formal SLAs documented and signed | Data Engineering + Leadership | SLA document with uptime, latency, support commitments |
| User training and onboarding materials | Data Engineering + Business | Self-service onboarding guide, FAQ |
| First downstream system integration | Data Engineering + UM or Provider Relations | One BSC system reading from the API |
| External pen test | Security | Security findings report and remediation plan |
| Phase 3 go-live | All | Platform available to all authorized BSC users |

**Success criteria:** Zero manual access provisioning required. At least one downstream system consuming data via API. Formal SLAs in effect.

---

## Section 9 — The Path to Commercial Expansion (Selling to Other Health Plans)

The business has indicated interest in eventually offering this platform to other organizations. This section describes what that requires, how it differs from BSC internal production, and what a realistic timeline looks like.

### What Changes for Multi-Tenancy

The data layer architecture is already correct for multi-tenancy. Unity Catalog's schema-level isolation means each client gets their own schema (`client_x.raw.*`) with their own contract data, while sharing the same Databricks platform. One extraction pipeline, parameterized by client. One application deployment, routing by tenant ID.

What needs to be built:

| Component | Description | Effort |
|---|---|---|
| Tenant provisioning automation | When a new client signs up, automatically create their Unity Catalog schema, provision their storage volume, and configure their application environment | 4-6 weeks |
| Application-level tenant routing | Every API request includes a tenant identifier. The application routes to the correct schema automatically. | 3-4 weeks |
| Billing and metering | Track API calls, LLM usage, and storage per client. Generate monthly usage reports for billing. | 4-6 weeks |
| Self-service onboarding | Client uploads their contract PDFs through a portal. The extraction pipeline runs automatically and produces their private knowledge base. | 6-8 weeks |
| Client admin portal | Clients can manage their own user access, view their usage, and configure extraction parameters. | 8-10 weeks |
| Enterprise security posture | SOC 2 Type II audit, BAA (Business Associate Agreement) for HIPAA-adjacent data, enterprise support SLA. | 12-16 weeks (parallel) |

**Total new build for multi-tenancy:** approximately 4-6 months of engineering work after BSC internal production is stable.

### Cost Model Considerations

Each client's contract corpus requires its own Databricks compute for ingestion, its own Vector Search index, and its own storage. LLM API calls are per-query costs that scale with usage. The pricing model for the commercial product needs to reflect these per-client infrastructure costs.

A rough cost floor per client per month (based on current BSC infrastructure costs extrapolated): approximately $2,000-5,000/month for storage, compute, and LLM API calls at moderate usage. This defines the minimum viable pricing for commercial viability.

### Realistic Timeline

| Milestone | Timeline from Today |
|---|---|
| BSC internal production (Phase 3 complete) | Month 7 |
| Multi-tenancy architecture design | Month 7-8 |
| First external pilot client | Month 12 |
| Commercial launch (general availability) | Month 18 |

The commercial timeline is realistic only if BSC internal production is treated as the first priority. Building BSC internal and commercial in parallel is too risky — bugs and architectural changes during BSC production will continuously disrupt the commercial build.

---

## Section 10 — What We Need from the Business

This project has reached the point where further progress requires inputs from the business that only the business can provide. The following are concrete asks — without them, the roadmap stalls.

### Ask 1: The Attribute Taxonomy (Krista)

**What:** A structured list of contract attributes the business considers essential to "global contract intelligence." For each attribute: its name, where it typically appears in a contract, what valid values look like, and which teams need it.

**Why it matters:** This list determines what the extraction pipeline targets next. Without it, we continue extracting attributes that seem important from a technical perspective but may not match actual business priorities.

**Format:** A spreadsheet or document is fine. Even a rough first draft is more valuable than waiting for a perfect list.

**Owner:** Krista, with input from Network team, UM, and Care Management

**Needed by:** Before Phase 2 extraction work begins (by end of Week 6)

### Ask 2: UAT Participants and Time Commitment

**What:** Named participants from Michael Simpson's team and the Network team who will commit to 3 structured UAT sessions of 2-3 hours each over 3 weeks.

**Why it matters:** UAT is the only way to validate that the platform answers the actual questions business users need to answer. Without real users doing real tasks, we cannot know what is production-ready and what is not.

**Owner:** Michael Simpson and Network team lead

**Needed by:** As soon as possible — UAT can begin within 2 weeks of receiving the participant list

### Ask 3: Databricks Workspace Accounts (IT)

**What:** A standard IT request to provision Databricks workspace accounts for the UAT participants. Accounts need read-only access to the `dev_adb.raw` catalog.

**Owner:** IT, with the user list from Ask 2

**Needed by:** Week 1

### Ask 4: Scope Decision — BSC Internal First vs. Parallel Commercial

**What:** A clear decision on whether BSC internal production is the sole near-term focus, or whether commercial expansion should be pursued in parallel.

**Why it matters:** Building both simultaneously requires at minimum 3-4 engineers. Building BSC internal first requires 1-2 engineers and produces a better commercial product as a byproduct. The two paths have meaningfully different resource and timeline implications.

**Owner:** Leadership

**Needed by:** Before Phase 2 resourcing decisions are made

### Ask 5: Dedicated Engineering Resources

**What:** A commitment to assign at least 2 dedicated engineers (data + application) to the productionalization roadmap. The platform was built by one person. Productionalization cannot be done solo without extending all timelines by 2-3x.

**Why it matters:** The current state was built solo as a proof of concept. Moving to enterprise-grade requires parallel workstreams (data quality fixes, SSO integration, pipeline automation, UX work) that cannot be executed sequentially by one person within any reasonable timeframe.

**Owner:** Leadership

**Needed by:** Before Phase 2 begins

---

## Section 11 — Investment Summary and Timeline

### Summary Timeline

| Milestone | Effort | Calendar Time from Today |
|---|---|---|
| Pilot user access (10-15 users) | 1-2 weeks | Weeks 1-2 |
| UAT completion | 4 weeks | Weeks 3-6 |
| Attribute taxonomy agreed | 2 weeks | Week 6 |
| Pipeline automation | 2-3 weeks | Month 2 |
| Quality gaps closed (4 metrics) | 6-8 weeks | Month 3 |
| Enterprise SSO (Azure AD) | 3-4 weeks | Month 3-4 |
| Azure Container Apps migration | 4-6 weeks | Month 4-5 |
| Full BSC internal production | — | Month 6-7 |
| First downstream system integration | 4-6 weeks | Month 7-8 |
| Multi-tenancy / commercial architecture | 4-6 months | Month 12+ |
| Commercial launch | — | Month 18 |

### Recommended Next Steps from This Meeting

1. **Agree on UAT scope and participants.** Name the people from Michael Simpson's team and the Network team who will participate in UAT. Set a start date.

2. **Begin IT access provisioning.** Submit the Databricks workspace account request for UAT participants this week.

3. **Schedule the attribute taxonomy workshop.** Set a 90-minute session with Krista to review the 21 current clause categories and produce a first draft of the target attribute list.

4. **Make the scope decision.** BSC internal first vs. commercial in parallel. This decision gates all resource planning.

5. **Identify engineering resources.** Confirm who will support the Phase 2 productionalization work and when they are available.

---

*This document was prepared by Ankit Dixit, Data Engineering, in response to the Contract Codification Tool meeting request of 2026-07-21. For technical questions about the platform, the data model, or the architecture, contact the author directly. For the supporting technical documentation, see `CONTRACT_INTELLIGENCE_V4_APPLICATION_ARCHITECTURE.md` and `CONTRACT_INTELLIGENCE_V4_BUSINESS_OVERVIEW.md` in the `docs/` directory.*

---

## Section 12 — Cost Analysis: What This Costs to Build, Run, and Scale

This section covers every cost dimension the business should understand before committing to production: what was spent to build this, what it costs to run today, what each AI answer costs, what production scale costs, and how it compares to buying a vendor solution.

---

### 12.1 — Investment Already Made

The platform was built by one person over approximately four months without vendor contracts, consulting engagements, or licensed tooling. The only hard costs incurred were standard Databricks compute and LLM API tokens.

| Cost Component | Estimated Spend |
|---|---|
| Engineering time (1 senior data engineer, ~4 months) | $120,000 – $200,000 (at $150–250/hour fully loaded) |
| Databricks compute during development | $2,000 – $5,000 |
| LLM API costs during development and testing | $500 – $2,000 |
| External tooling, licenses, consulting | $0 |
| **Total investment to current state** | **~$125,000 – $210,000** |

**For reference:** A comparable engagement with a consulting firm (Accenture, Deloitte, or a health-tech boutique) to produce a working prototype of this scope would typically cost $800K – $2M. A commercial contract intelligence vendor (Icertis, Conga, Ironclad) at enterprise scale typically runs $250K – $750K per year in licensing fees alone — before implementation costs.

---

### 12.2 — Current Monthly Operating Costs (Today, Low Usage)

The system is running today at minimal usage (development and demo traffic only). The infrastructure costs reflect this low baseline.

| Component | What It Is | Estimated Monthly Cost |
|---|---|---|
| Vector Search endpoint (`contract_intelligence_vs_endpoint`) | Hosts the 727,876-chunk semantic index. Runs 24/7. | $150 – $400 |
| SQL Warehouse (`DATABRICKS_WAREHOUSE_ID: 2c99c6485f03ee73`) | Executes all structured contract queries. Auto-suspends when idle. | $100 – $300 (at current low usage) |
| Databricks Apps compute | Hosts the FastAPI backend and React frontend. | $100 – $250 |
| LLM API tokens (Claude Sonnet 4.6 + Haiku 4.5) | Per-token costs for all AI reasoning, synthesis, and citation verification. | $50 – $200 (minimal demo traffic) |
| Delta table storage (dev_adb.raw, ~31 tables, ~1.3M rows) | Approximately 15–25 GB of structured contract data. | $5 – $20 |
| PDF volume storage (~10,349 contract documents) | Approximately 50–80 GB of source PDFs. | $5 – $15 |
| **Total current monthly cost** | | **~$410 – $1,185/month** |

Note: These estimates use standard Azure Databricks list pricing. BSC likely has an enterprise discount agreement that reduces actual DBU costs by 20–40%.

---

### 12.3 — Per-Query AI Cost: What Each Answer Costs

This is the most important unit economics metric. The AI agent uses a tool cost budget system — every tool call has an assigned cost that counts against a $15.00 per-session budget. These costs reflect actual LLM token consumption plus compute.

| Question Type | Tools Typically Called | Estimated Cost per Query |
|---|---|---|
| Simple factual lookup ("What is Sutter's DRG base rate?") | `rate_query` or `sql_query` | $0.01 – $0.05 |
| Standard chat question (1–3 tools + LLM reasoning + synthesis) | `passage_search` + `sql_query` + synthesis | $0.05 – $0.30 |
| Complex analytical question (multi-tool + amendment history) | `temporal_analysis` + `passage_search` + `sql_query` | $0.30 – $1.00 |
| Clause existence check with verification | `clause_existence` + `citation_verify` | $0.50 – $1.50 |
| Provider deep dive (full contract profile) | `provider_deep_dive` + multiple lookups | $1.00 – $3.00 |
| Report generation across all 306 providers | `multi_concept` (runs per provider) | $5.00 – $12.00 |
| Maximum session budget (hard cap) | Any combination | $15.00 |

**Average realistic cost per user question: $0.10 – $0.40.**

At $0.25 average per query, 1,000 questions/month = $250 in LLM costs. This is the marginal cost of answering questions that previously required a contract analyst to manually search PDFs — typically 30–90 minutes per question.

---

### 12.4 — Projected Monthly Cost at Production Scale

Three usage scenarios based on the anticipated user rollout.

#### Scenario A: Internal Pilot (15–30 users, ~200 queries/month)

| Component | Monthly Cost |
|---|---|
| Infrastructure (VS endpoint, SQL Warehouse, Apps compute, storage) | $400 – $900 |
| LLM API costs (200 queries × $0.25 avg) | $50 |
| **Total** | **~$450 – $950/month** |

#### Scenario B: BSC Production (50–100 users, ~2,000 queries/month)

| Component | Monthly Cost |
|---|---|
| Vector Search endpoint (same, always-on) | $200 – $400 |
| SQL Warehouse (higher utilization, less idle time) | $400 – $800 |
| Databricks Apps or Azure Container Apps compute | $300 – $600 |
| LLM API costs (2,000 queries × $0.25 avg) | $500 |
| Azure CDN, Front Door, monitoring (if migrated) | $200 – $400 |
| **Total** | **~$1,600 – $2,700/month** |

#### Scenario C: Multi-Team Enterprise (200+ users, ~10,000 queries/month)

| Component | Monthly Cost |
|---|---|
| Vector Search endpoint (may need scale-up) | $400 – $800 |
| SQL Warehouse (multiple concurrent clusters) | $800 – $1,600 |
| Azure Container Apps (auto-scaling, 3–10 instances) | $600 – $1,200 |
| LLM API costs (10,000 queries × $0.25 avg) | $2,500 |
| Azure Front Door, monitoring, Key Vault | $300 – $600 |
| **Total** | **~$4,600 – $6,700/month** |

---

### 12.5 — One-Time Productionalization Investment

The cost to close the gaps documented in Sections 7 and 8 and reach enterprise-grade production.

| Work Item | Engineering Effort | Estimated Cost (2 engineers, $175/hr loaded) |
|---|---|---|
| UAT facilitation and issue remediation | 3–4 weeks | $42,000 – $56,000 |
| Fix 4 failing quality metrics (amendment chains, rate outliers) | 6–8 weeks | $84,000 – $112,000 |
| Enterprise SSO / Azure AD integration | 3–4 weeks | $42,000 – $56,000 |
| In-app RBAC (role-based access control) | 4–6 weeks | $56,000 – $84,000 |
| Pipeline automation (file-arrival trigger) | 2–3 weeks | $28,000 – $42,000 |
| Azure Container Apps migration | 4–6 weeks | $56,000 – $84,000 |
| Monitoring, alerting, and SLA governance | 2–3 weeks | $28,000 – $42,000 |
| API versioning and data contracts | 3–4 weeks | $42,000 – $56,000 |
| Security pen test (external vendor) | External | $20,000 – $50,000 |
| User training and onboarding materials | 2 weeks | $14,000 – $28,000 |
| **Total productionalization investment** | **~28–42 weeks** | **~$412,000 – $610,000** |

This is the cost of doing it properly with a dedicated 2-person team. The current single-person approach can execute the same work but on a timeline that is 2–3× longer.

---

### 12.6 — Annual Maintenance Cost (Post-Production)

Once the platform is in production, ongoing maintenance requires sustained engineering investment.

| Activity | Effort | Annual Cost Estimate |
|---|---|---|
| Infrastructure maintenance and upgrades | 0.25 FTE | $35,000 – $55,000 |
| Data quality monitoring and issue resolution | 0.25 FTE | $35,000 – $55,000 |
| New contract ingestion (as contracts renew or change) | 0.25 FTE | $35,000 – $55,000 |
| Feature development and UX improvements | 0.50 FTE | $70,000 – $110,000 |
| Security patches, dependency updates, DBR upgrades | 0.10 FTE | $14,000 – $22,000 |
| **Total annual maintenance** | **~1.35 FTE** | **~$189,000 – $297,000/year** |

Plus infrastructure run costs (Scenario B above): **~$19,200 – $32,400/year.**

**Total annual cost of ownership (production):** approximately **$210,000 – $330,000/year.**

---

### 12.7 — Build vs. Buy: Vendor Comparison

The business should understand what alternative solutions cost, to contextualize the investment required.

| Solution | Typical Annual Cost | Notes |
|---|---|---|
| **Contract Intelligence V4 (this platform)** | **$210K – $330K/year (maintenance + infra)** | Built to BSC-specific data model, AI-native, fully integrated with Databricks |
| Icertis Contract Intelligence | $300K – $800K+/year | Enterprise contract management + AI. Requires 6–18 month implementation. Limited to structured metadata — not full-text AI reasoning across PDFs. |
| Conga (Apttus) Contract Management | $150K – $400K/year | Strong CLM workflow, weaker analytics. Does not answer ad hoc questions. |
| Ironclad | $80K – $250K/year | Smaller-scale. Modern UI but limited health-plan domain model. |
| Evisort (Workday) | $150K – $500K+/year | AI contract extraction. Good for standard contracts. Not optimized for California health plan regulation complexity. |
| Custom build with consulting firm | $2M – $5M (one-time) | Would produce comparable functionality in 18–24 months. No pre-built BSC data model. |
| Manual contract analysis (status quo) | $300K – $600K/year (analyst time) | Approximate cost of 3–5 FTE contract analysts doing what the AI currently answers in seconds. Does not scale. |

**The platform as built delivers functionality comparable to a $300K–$800K/year vendor solution, built for a fraction of the cost, on BSC's own infrastructure, with full data ownership and no vendor lock-in.**

---

### 12.8 — Commercial SaaS Unit Economics (If Sold to Other Health Plans)

If BSC decides to license this platform commercially, the unit economics per client are as follows.

**Per-client infrastructure cost floor:**

| Component | Monthly Cost per Client |
|---|---|
| Separate Unity Catalog schema (data isolation) | Minimal (storage only) |
| Per-client Vector Search index (their contract corpus) | $150 – $400 |
| Per-client SQL Warehouse (dedicated or shared) | $100 – $300 |
| LLM costs proportional to their query volume | $100 – $500 (at moderate usage) |
| Shared application infrastructure (Container Apps) | $50 – $100 (amortized) |
| **Total per-client monthly infrastructure cost** | **~$400 – $1,300/month** |

**Minimum viable commercial price:** To cover infrastructure, maintain a 60%+ gross margin, and fund ongoing development, a minimum price of **$3,000 – $8,000/month per client** is required. At market rates for health plan contract intelligence ($15K–$50K/month from enterprise vendors), the platform has significant pricing headroom.

**Revenue potential at 10 clients:** $30,000 – $80,000 MRR ($360K – $960K ARR). This does not require BSC to become a software company — it can be structured as a data services agreement or a joint-venture product with an existing health-tech partner.

---

### 12.9 — Cost Summary for the Meeting

| Question | Answer |
|---|---|
| What did it cost to build? | ~$125K–$210K (1 engineer, 4 months, zero vendor spend) |
| What does it cost to run today? | ~$410–$1,185/month at current low usage |
| What does each AI answer cost? | $0.10 – $0.40 per typical question |
| What does BSC production cost? | ~$1,600 – $2,700/month (50–100 users) |
| What does it cost to productionalize properly? | ~$410K – $610K one-time (2-engineer team, 28–42 weeks) |
| What does it cost to maintain annually? | ~$210K – $330K/year (infrastructure + 1.35 FTE) |
| What would a vendor alternative cost? | $300K – $800K/year in licensing alone, plus $500K–$2M implementation |
| What is the break-even vs. manual analysis? | The platform pays for itself if it saves 1,500–2,500 analyst hours per year |
| What can we sell it for (if commercialized)? | $3,000 – $8,000/month per client; 10 clients = $360K–$960K ARR |

