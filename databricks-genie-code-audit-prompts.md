# Contract Intelligence — Databricks Genie Code Audit Prompts

Run the following prompts in one Genie Code conversation, one batch at a time. Do not allow code modifications until a later implementation phase. The goal is a trustworthy Contract Intelligence application in one Databricks development environment for a controlled user trial, with deep data-model validation, measurable answer quality, evidence, confidence and a prioritized plan.

## Batch 0 — Establish the working rules

```text
You are reviewing and improving a Contract Intelligence application built on Databricks.

The objective is to create the best possible version in one Databricks development environment for a controlled group of users to use, understand, evaluate and decide whether it provides business value. This is not a production-readiness exercise. Do not focus on enterprise production hardening unless it directly prevents trustworthy answers or safe use by evaluation users.

The repository contains Databricks App backend and front-end code, SQL Warehouse and Model Serving integration, Unity Catalog metadata, architecture and data-model documents, prompts, SQL generation/validation, structured contract query logic, answer generation, citations/evidence, tests, possible obsolete/dead/duplicated code, and upstream OCR/ingestion code.

Client data remains in the client Databricks environment and must not be exported. Use repository code, documentation, the previous audit, permitted workspace schemas/metadata, controlled aggregates/approved samples, and synthetic data where needed. OCR, PDF parsing, document extraction and upstream ingestion are out of scope. Assume required structured contract data already exists.

The goals are: make development use successful; improve correctness, completeness, consistency and usefulness; deeply validate the model and its application use; identify active/dead/duplicated code; make the app understandable; provide evidence; add statistical/data-science validation; create an issue register and roadmap; separate confirmed defects from hypotheses; and avoid unnecessary production complexity.

Do not make unsupported assumptions. Cite exact files, classes, functions and line numbers. Distinguish active code, documented versus implemented architecture, source-evident defects and data-validation risks. Do not modify code. Maintain a progress ledger and preserve prior findings. Do not treat successful SQL or citations as proof of correctness; do not treat LLM self-confidence as statistical confidence.

Read the previous audit completely. Treat it as input rather than truth: verify each finding, retain what affects correctness/trust/UX/stability/evaluation, and downgrade production-only concerns.

Return only: objective; in-scope areas; out-of-scope areas; documents/repository areas to inspect; progress-ledger format; and blocking ambiguity. Do not begin detailed analysis yet.
```

## Batch 1 — Repository inventory and active code

```text
Continue the Contract Intelligence development-environment audit. Do not modify code.

Inspect the complete repository and determine what actually matters at runtime. It may contain old/duplicate controller methods, alternate prompts, unused versions, experiments, obsolete configuration, notebooks, stale static builds, OCR-related code, apparently active but inactive files, and dynamic/configuration-driven imports.

Classify every important directory/file/component as: confirmed active runtime code; likely active runtime code; front-end source; deployed/generated front-end artifact; development-only; test/evaluation; deployment/configuration; documentation; OCR/ingestion out of scope; legacy but referenced; duplicated; likely dead; obsolete configuration; or unknown requiring confirmation.

Use entry points, app.yaml, Python imports, routes, object construction, calls, dynamic imports, dependency injection, Databricks Apps/Asset Bundle configuration, variables, notebooks, build scripts, static asset serving, frontend builds, tests, documentation, duplicate symbols, shadowing, unreachable branches and unused configuration as evidence.

Focus on the app entry, API/chat routes, initialization, controller/router, SQL generation/execution, schema metadata, prompts, model clients, answer synthesis, citations/confidence, sessions, streaming, frontend deployment, tests and evaluation assets. For each likely dead/duplicated item provide path, symbol, evidence, references, confidence, removal risk and retain/quarantine/merge/archive/investigate/remove-after-tests action. Do not delete anything.

Produce repository map; confirmed entry points; active dependency graph; frontend source/build relationship; duplicate/dead register; OCR inventory; unresolved references; updated ledger. Do not deeply analyse the model yet.
```

## Batch 2 — Actual question-to-answer flow

```text
Continue using the repository map. Do not modify code.

Trace the active flow: app open; frontend initialization; conversation creation/loading; question submit; request payload; backend receipt; session/history; classification/rewrite/decomposition; model/schema context selection; SQL/retrieval/tool selection; SQL generation/validation; Warehouse execution; result transformation; evidence assembly; Model Serving invocation; answer generation; confidence/citations; response streaming/return; frontend render; feedback; trace persistence.

For every step identify file, class, method, caller, downstream dependency, inputs/outputs, configuration, fallback, error handling, fallback visibility and answer-quality impact.

Identify direct, generic-agent, decomposed, rate, clause, comparison, temporal, Genie, fallback/degraded and callable legacy paths. Compare prompts, endpoints, schema context, SQL/citation/confidence rules, output formatting and error behaviour. Explain where similar user questions can be routed differently.

Produce architecture diagram, lifecycle table, path comparison, stage failure points, user-visible failures, hidden fallbacks, documentation differences and updated ledger. First prove active behaviour; do not recommend broad refactoring.
```

## Batch 3 — Deep data-model validation

```text
Continue the audit. Do not modify code.

Determine whether the model supports correct Contract Intelligence answers and whether the app understands it. Read the full data-model/ER/schema documents, metadata, business definitions, SQL prompt context, whitelists, join descriptions, Unity Catalog comments, available samples/synthetic data, data-quality notes and previous findings. Do not assume documentation is correct; compare it with schemas and application use.

For every application-used table document qualified name, business purpose, grain, natural and technical keys, entity, foreign keys/cardinality, effective/termination/current/version/amendment fields, provider/contract/rate/clause IDs, status/source fields, duplicate/null/data-quality risks, use cases and active references.

Validate contract hierarchy; provider hierarchy; rate grain and rate types, periods, schedules, carve-outs and overlaps; temporal/currentness/version/supersession rules; and clause identity/text/version/source/effective periods/exclusions/negation/cross-references/amendment precedence. Classify relationships and identify row-multiplying joins.

Determine whether active semantic context supplies approved/prohibited joins, join keys/cardinality/grain, required filters, historical/current rules, distinct/aggregation rules, null semantics, units/percentages/formulas, precedence, hierarchies, synonyms and definitions.

Use permitted read-only aggregate profiling: counts, distinct keys, null/duplicate rates, dates, overlaps, orphans, join expansion, status/currentness/version distributions and conflicting rates. Do not export sensitive records. For each query give purpose, SQL, interpretation, threshold and automation suitability.

Produce table inventory; entity/grain matrix; approved/dangerous joins; temporal/currentness, rate and clause rulebooks; profiling plan; model and documentation defects; semantic-context gaps; unanswerable questions; updated ledger. Label every finding confirmed code/document defect, confirmed schema issue, probable data issue, semantic ambiguity or requiring development-data validation.
```

## Batch 4 — SQL semantic correctness

```text
Continue from the validated semantic model. Do not modify code.

Audit all active SQL paths: table/schema/column/join selection, provider/contract/clause/current/historical filters, amendments, aggregation and distinct counts, grouping/ordering/limits, nulls, percentages, formulas, money, duplicates, validation/repair, execution and handling of empty/partial/large results. Compare each rule to Batch 3. Identify SQL that executes but yields a businessly incorrect answer.

Create at least 30 test questions covering provider/contract lookup; current/historical rates; comparisons; clauses; amendments; provider-system aggregation; counts/distinct counts; percentages/formulas; exclusions; no-result; ambiguity; multi-part and follow-up questions. For each define meaning, entities, tables, joins, filters, grain, aggregation, temporal logic, expected SQL, common incorrect SQL, answer/evidence shape and clarification need.

Determine whether validation checks syntax/safety only or table eligibility, approved joins/cardinality/grain, required dates, hierarchy rules, aggregation/currentness, result limits and semantic consistency. Design a deterministic layer using parsed SQL AST, structured query plan, approved join graph, semantic intent, required-filter/grain/temporal/aggregation policies and result-shape checks. Prefer incremental improvements.

Produce SQL architecture, risk register, valid-SQL/wrong-answer examples, golden SQL benchmark, semantic-validator design, result rules, immediate and medium-term fixes, tests and updated ledger.
```

## Batch 5 — Grounding, evidence, citations and trust

```text
Continue the audit. Do not modify code.

Determine whether every final answer is traceable to executed data evidence. Inspect answer synthesis/summarization, citation generation/validation, evidence IDs, source table and contract/provider/clause/rate/record IDs, SQL trace, currentness, hallucination/refusal guard, confidence, persistence/history, frontend citations, export and feedback.

Determine whether citations are deterministic or model-generated, sourced from records or passages, validated/optional, preserved across history/export, visible and reproducible. Define minimum evidence for numeric values, rates, percentages, counts, provider relationships, status/dates, clause existence/absence, comparisons, historical/current claims, interpretation, recommendation and no-answer claims.

Define claim-evidence payload with claim ID/text/type, source type/ID/field/value, query/retrieval ID, currentness, calculation method, evidence completeness, contradiction status and confidence components. Do not use one opaque LLM confidence score.

Identify unsupported, partial, stale or contradictory claims; non-supporting citations; evidence lost in synthesis/UI/persistence/export; and hidden empty/partial results. Design a mandatory validator: every factual claim evidenced; numeric claims reproduce; counts preserve grain; current/historical evidence correct; comparisons comparable; clause negation/exclusions preserved; citations identify sources; contradictions/truncation disclosed; unsupported claims removed/labelled; no-result distinguished from system error.

Produce evidence/citation architecture, claim requirements, grounding defects, validator design, citation tests, answer-trace payload, frontend evidence requirements and updated ledger.
```

## Batch 6 — Statistical confidence and data-science-style validation

```text
Continue the audit. Do not modify code.

Stakeholders explicitly require statistics, data-science-style tests, measurable value and confidence indicators. Design a statistically defensible answer-confidence and quality-validation framework. Never represent LLM self-reported confidence as statistical confidence or invent one arbitrary percentage without measurable, documented and calibrated components.

Separate data confidence, query confidence, evidence confidence, answer confidence, model consistency, coverage, business-rule compliance and overall trust category.

Define metrics for data quality (keys, nulls, duplicates, orphans, status conflicts, temporal overlaps/staleness, unknown codes, source/amendment/provider/rate/clause completeness); query quality (intent/entity/table/column, join, grain/filter/temporal/aggregation, execution/results); evidence quality (claim coverage, citation precision/recall, currentness, consistency, directness, completeness, contradictions, reproducibility); answer quality (correctness, completeness, groundedness, numeric/temporal/business correctness, no-answer/clarification, unsupported claims, usefulness/clarity); and stability (repeated run, paraphrase, route, model version, SQL, answer and citation consistency).

For every metric give name, purpose, formula, numerator, denominator, question types, inputs, warning/failure thresholds, when available, ground-truth need, determinism and LLM dependency.

Create component scores, reasons for deductions, hard-failure rules, user trust bands and technical diagnostics. Define exact measurable criteria for High/Moderate/Low confidence, Insufficient/Conflicting evidence and Unable to validate. Hard failures override averages: missing date filter, unapproved join, unresolved provider, conflicting current record, uncited numeric claim, undisclosed truncation, zero rows treated as absence, unsupported model facts. Use question-type-specific scoring.

Design accuracy/groundedness confidence intervals, citation precision/recall, unsupported-claim rate, semantic SQL accuracy, repeatability/paraphrase consistency, category errors, bootstrap/Wilson intervals, inter-rater agreement, calibration/reliability plots, Brier/ECE where valid, severity weighting, coverage-accuracy/abstention, threshold selection and version significance testing. Explain sample-size validity.

Design benchmark review: sampling/stratification/sample size, experts, two reviewers/resolution, gold answers/SQL/evidence, severity/versioning, blinded comparisons and leakage prevention. Design a business trust panel (band, reasons, coverage/currentness, records, direct/inferred calculations, agreement, limitations, citations/evidence) and a separate technical panel.

Produce confidence framework, metric catalogue/formulas, hard failures, question-type scores, statistical/calibration/human-review plans, user and technical panels, minimum viable/advanced implementation, risks of misleading confidence and updated ledger.
```

## Batch 7 — User trial and usability

```text
Continue the audit. Do not modify code.

The app will be given to a controlled development user group. Users must understand its purpose/data coverage, ask effective questions, see why answers were produced, judge trust, give structured feedback and assess business value.

Review landing/onboarding/conversation pages, input/examples, loading/progress, answer layout/tables/citations/confidence/evidence/limitations, error/no-result/clarification states, history, feedback/export, long answers and responsive behaviour. Design purpose, supported/unsupported categories, coverage statement, examples, asking guidance, clarification, answer evidence/confidence, expert SQL/trace, limitations, feedback, issue reporting, questionnaire and usage analytics.

Design feedback beyond thumbs up/down: correct/partial/incorrect/unable to judge; missing data; wrong provider/contract/date/aggregation/rate/clause; citation issue; unclear explanation; confidence appropriate/too high/too low; expected answer; comment. Design scorecards for task success, correctness, trust, confidence/citation usefulness, time saved, ease, question difficulty, future use and value, separately for business users, contract experts, data owners and technical evaluators. Create a guided evaluation task script.

Produce current frontend gaps, recommended experience, onboarding, supported-question guide, feedback, questionnaire, trial tasks, metrics, frontend change list and updated ledger.
```

## Batch 8 — Databricks development-environment limitations

```text
Continue the audit. Do not modify code.

Review Databricks architecture only for a useful/reliable controlled development trial. Do not create a production-hardening checklist. Ignore multi-region DR, enterprise deployment automation, full SLO/incident governance and organizational-scale access unless directly trial-blocking.

Evaluate Apps, SQL Warehouse, Model Serving, Unity Catalog, active Vector Search, workspace/app resources, dev logs, variables/secrets, static assets, session storage, limited concurrency, timeouts/cold starts/routing/model parameters/result limits, app runtime/memory and workspace-specific hard coding.

Classify each finding as actual Databricks limitation, code limitation, configuration limitation, development operational limitation, not trial-relevant or requiring workspace validation. Determine whether app work should move to Warehouse/Serving; cold starts/timeouts make it appear broken; streaming is real; endpoint limits/routing truncate answers; queries are oversized; schema/resource/static builds are synchronized; small concurrency works; sessions isolate users; requests are traceable; dependencies reproduce; resource names are centrally configurable.

Produce platform assessment, relevant limitations, implementation/configuration limitations, trial blockers, non-blockers, workspace checklist, recommended dev configuration and updated ledger.
```

## Batch 9 — Benchmark and test suite

```text
Continue the audit. Do not modify code.

Design an end-to-end evaluation/regression suite using synthetic structured data, approved development data, deterministic SQL assertions, expected query plans/facts/citations, expert review, statistical analysis, repeat runs, paraphrases and controlled user evaluation.

Cover model/entity/table/join/grain/aggregation/temporal/version/amendment/provider/rate/clause logic, execution/results, correctness/groundedness/citations/confidence, clarification/no-answer/follow-up/multi-part, UI/session/feedback, latency and limited concurrency.

Define at least 100 stratified benchmarks. First state category, quantity, complexity, data scenario, gold labels, metric and acceptance threshold, then examples. Use fields: test ID, category/subcategory, question, paraphrase group, difficulty, entities, intent, tables/joins, temporal rule/filters/aggregation/query plan, expected/prohibited facts and citations, clarification/no-answer/confidence expectation, notes, data version and app version.

Set acceptance criteria for SQL semantics, answer correctness, groundedness, citations, unsupported claims, currentness, clarification/no-answer, calibration/high-confidence errors, repeatability/paraphrases and suitable latency. Define separate developer, expert, user-trial and continued-investment gates.

Produce test architecture, taxonomy, record schema, sample cases, statistical design, acceptance gates, regression strategy, execution plan, dashboard and updated ledger.
```

## Batch 10 — Consolidated issue register

```text
Continue the audit. Do not modify code.

Consolidate all findings. Remove duplicates, inactive-code findings, production-only hardening, unsupported speculation and generic advice. Retain issues affecting answer/model/SQL/temporal/version/provider/rate/clause correctness, evidence/citations/confidence, user understanding, UI reliability, development stability/evaluation, useful observability, dead-code confusion and Databricks development limits.

For each issue provide ID, title/category/description, exact evidence, file/class/function/line, active path, defect/weakness/ambiguity/documentation drift/hypothesis/workspace requirement, severity/likelihood, user/quality/model/trust/trial impact, fix/files/dependencies/complexity/regression risk, validation/acceptance and implementation phase.

Use Critical/High/Medium/Low and Confirmed/Highly likely/Plausible/Requires data validation/Requires workspace validation. Rank confidently wrong answers, wrong contract/provider/current/historical/rate/count answers, unsupported/misleading confidence, missing evidence, inability to evaluate/reproduce and competing active implementations first.

Produce master register; top 10 blockers and quality improvements; top model/trust/UX issues; dead-code cleanup list; deferred production issues; updated ledger.
```

## Batch 11 — Final implementation roadmap

```text
Continue the audit. Do not modify code.

Create the final incremental, testable roadmap for a controlled development user trial from the validated register. Do not propose one large rewrite.

Use: Phase 0 trusted baseline (runtime confirmation, characterization tests, deployed build, app/prompt/model version IDs, synthetic data, benchmark, trace, highest-risk duplicate); Phase 1 semantic foundations (grain, joins, temporal/amendments, hierarchies/rates/clauses, metadata/context); Phase 2 SQL correctness (structured intent and semantic/result validation); Phase 3 grounding/trust (claims, validation, deterministic citations, currentness, calculations, contradictions, abstention, confidence); Phase 4 UX; Phase 5 statistical evaluation; Phase 6 development reliability (truthful streaming, routing/timeouts/concurrency/sessions/traces/configuration/build consistency); Phase 7 dead-code quarantine; Phase 8 controlled user trial.

For each item provide roadmap ID, objective, linked issues, files/components, prerequisites/tasks, answer/trust/trial impact, complexity/risk/owner skill, test/acceptance, sequence and data/workspace requirement.

Also provide minimum viable trusted application, best-development-version plan, changes not to attempt yet, readiness checklist, go/no-go framework, implementation prompt order and final ledger. Prioritize correct entity selection, joins/grain, temporal logic, rates/counts, grounded answers, citations, non-misleading confidence, reproducible traces and understandable UI over visual polish.
```

## Final consolidation prompt

```text
Using all completed batches, create one final document titled:

CONTRACT INTELLIGENCE DEVELOPMENT APPLICATION
QUALITY, TRUST AND USER-EVALUATION PLAN

Do not modify code. Consolidate only validated findings. Include executive summary; development objective; current architecture and active map; dead/duplicate summary; question flow; model/semantic/join/temporal/rate/clause validation; SQL; grounding/citations; confidence/statistics; trust/UI/onboarding; development Databricks limitations; benchmark/testing; issue register; roadmap; user trial; acceptance criteria; go/no-go; open questions; and an appendix of read-only validation queries.

For every major finding include evidence, confirmation, importance, correction, test and acceptance criterion. Do not include generic production hardening except a small deferred appendix.

Before finalizing verify Critical/High items reference active code; each confidence metric has a formula; LLM self-confidence is never called statistical confidence; data-model, temporal, joins/grain, evidence, user trial and statistical testing are included; dead code is separate; and recommendations are sequenced and testable.
```

## Most important direction for Genie

```text
Do not continue with generic recommendations. Reinspect the active code,
data-model documentation and previous findings, and support every conclusion
with exact repository evidence, a concrete failure example and a validation
test.
```
