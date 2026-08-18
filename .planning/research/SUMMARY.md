# Project Research Summary

**Project:** UP IT Pulse
**Domain:** AI-assisted IT service-desk ticket triage & early-warning incident detection (bilingual Thai/English)
**Researched:** 2026-08-18
**Confidence:** MEDIUM-HIGH

## Executive Summary

UP IT Pulse sits in a well-understood product category — AI-assisted ITSM triage with human-in-the-loop approval — and the spec's core architecture (mask → AI-analyze → rule-suggest → embed/similarity-search → cluster-detect → human-confirm) matches how ServiceNow, Freshservice, and Zendesk structure the same problem. The chosen stack (Next.js/FastAPI/Postgres+pgvector) needs no changes; a handful of adjacent library choices should shift (`pwdlib[argon2]` not `passlib`, `PyJWT` not `python-jose`, skip `next-intl`).

The real risk is not "does this pattern exist" — it does — but three domain-specific traps that are invisible until measured: **(1)** the spec's similarity ranking formula (`semantic*0.65 + category*0.15 + location*0.10 + recency*0.10` vs. threshold `0.82`) is arithmetically broken — it caps any ticket with `location=null, category=OTHER` at a 0.75 ceiling, making the lowest-confidence tickets (exactly the ones needing duplicate/incident detection most) structurally unreachable, while research-verified negative-pair similarity for Thai text (0.74–0.81 on multilingual-e5) means unrelated same-building tickets can exceed threshold; **(2)** PostgreSQL has no Thai text search parser and pgvector filters *after* the index scan, so naive FTS and HNSW-with-filter both silently degrade on this exact dataset; **(3)** human-only approval gates (never confirm incidents, never auto-close, never auto-publish) are currently enforceable only by system prompt and UI convention — OWASP LLM06 is explicit that this must be a deterministic downstream constraint, and the `tickets` table is missing `resolved_by_user_id`/`closed_by_user_id` columns needed to even audit it.

Mitigation is cheap relative to the risk: recalibrate the ranking formula and threshold against the project's own seed/evaluation data before freezing it, skip vector indexing entirely at this scale (exact cosine scan gives 100% recall under 10k rows, well within the 1-second NFR), use `pg_trgm` instead of `to_tsvector` for search, and add a typed-actor state-machine with an exhaustive transition-matrix test plus the two missing audit columns in Phase 1 rather than retrofitting after 10 phases have already written to `tickets`.

## Key Findings

### Recommended Stack

Next.js 16.3 + FastAPI + PostgreSQL 18 + pgvector 0.8.6 is correct and current for 2026; no stack replacement needed. Three adjacent-dependency corrections apply, and Thai-language handling requires specific choices research validated against real benchmarks (SEA-BED, MIRACL) rather than assumption.

**Core technologies:**
- `pwdlib[argon2]` for demo password hashing — `passlib` is unmaintained since 2020 and breaks on modern bcrypt
- `PyJWT` for session tokens — `python-jose` is the deprecated alternative
- `BAAI/bge-m3` (self-hosted via TEI) as default embedding model, 1024-dim — MIRACL Thai nDCG@10 = 83.7, best measured Thai performance among viable self-hosted options; `gemini-embedding-001` @768-dim (with manual L2 normalization) as hosted fallback
- Exact cosine similarity scan (no HNSW index) at MVP scale — pgvector applies `WHERE` filters *after* the HNSW index scan, so the mandatory 24-hour lookback filter collapses recall; exact scan on ~10k×1024-dim rows runs in tens of milliseconds
- `pg_trgm` GIN index for FR-014 search, not `to_tsvector`/FTS — Postgres ships no Thai text-search configuration, since Thai has no inter-word spaces
- JSON-schema-constrained structured output (`response_format: json_schema, strict: true`) as the primary LLM contract, falling back through strict tool-call → `json_object`+repair → manual queue — but verify the actual provider supports it (llama.cpp and Ollama have shipped versions that silently ignore this parameter)
- Regex-based PII masking (not Presidio/NER) — Presidio has no production-quality Thai NER pipeline, and all five spec-mandated PII classes (email, phone, person ID, IP, secret) are pattern-detectable; must handle Thai digits (`๐-๙`), unreliable `\b` word boundaries in Thai script, and person-name masking is a real gap (not in FR-003's list, undermines FR-014's "PII-free export" claim)
- Deferrable entirely for MVP: Redis, Celery/arq, HNSW, `halfvec`, next-intl — every extra compose service is a live-demo failure point

### Expected Features

The spec's scope already matches the industry-standard feature set for this category; research confirms rather than expands it, with a few sharp differentiators and anti-features worth calling out explicitly.

**Must have (table stakes) — already in spec:**
- Similar-ticket panel with score/time/category/location — now standard across ServiceNow, Freshservice, JSM
- Human-edit-every-AI-field — Zendesk's pattern; non-negotiable for this category
- Audit log spanning login, ticket changes, AI calls, overrides, incident actions
- AI-generated draft labeling with human copy/edit before use

**Should have (competitive) — spec gets these right, they're genuinely sharper than incumbents:**
- 15-minute sliding window + ≥5 tickets + **≥3 distinct reporters** trigger — incumbents use 7-day "similar unresolved" lists or manual nomination; the distinct-reporter condition specifically distinguishes a spamming single user from a real outage
- Mask-before-inference (vs. Zendesk's post-hoc redaction-for-agent-view) — stronger privacy posture, worth stating as a design decision
- Four-way field provenance (user-reported / AI-suggested / rule-suggested / human-confirmed) shown separately in UI

**Defer / reconsider (anti-features not worth building, three not already in spec's Out of Scope):**
- Auto-merge above the duplicate threshold — destructive, silently loses a real reporter's distinct issue
- Sentiment analysis — zero triage value for this use case despite Zendesk/JSM parity pressure
- Free-text "ask AI about our tickets" chat panel — undoes the entire PII-masking investment in one feature

### Architecture Approach

The system is a fast synchronous intake path (ticket create, <2s, no AI wait) feeding an async staged enrichment pipeline (mask → analyze → rule-suggest → embed → cluster-detect) whose outputs land in provenance-tagged side tables — never directly mutating the authoritative `tickets` row, which only a guarded human-actor state-machine transition may write. This is what makes the spec's "show AI vs rule vs human separately" requirement and "AI acceptance/override rate" metric possible without a bolt-on event pipeline.

**Major components:**
1. **Ticket lifecycle + state machine** — typed-actor guarded transitions (AI can reach `AI_ANALYZED` only), audit-log writer wired in from the start since every later phase depends on it
2. **Job harness** (`analysis_jobs` table + `FOR UPDATE SKIP LOCKED`, `BackgroundTasks` as trigger only) — provides the retry/durability/queryability that FastAPI `BackgroundTasks` alone cannot, needed before any AI call exists
3. **Masking boundary** — `LLMPort`/`EmbeddingPort` accept only a typed `MaskedText` value, making "forgot to mask" a compile-time/type error rather than a runtime hope
4. **AI analysis + rule engine** — LLM adapter behind a provider-agnostic interface; rule engine outputs a distinct `RuleSuggestion` type, never writes `tickets.priority` directly (this is what resolves the P1-authority conflict architecturally)
5. **Embedding + similarity + incident detection** — one k-NN query per new ticket within the window (not O(n²) pairwise), incidents entity and human-confirm gate must exist *before* the detector is built, since the detector's only job is inserting into it

### Critical Pitfalls

1. **Broken ranking formula** — `sem*0.65+cat*0.15+loc*0.10+rec*0.10` vs. threshold `0.82` makes low-confidence tickets (`location=null`) mathematically unreachable (ceiling 0.75) while letting unrelated-but-context-matching tickets exceed it. Must recalibrate against real seed/evaluation data before freezing, and make the threshold reachable for the exact ticket profile Edge Case 6 describes.
2. **Human-only gates enforced only by prompt/UI, not data model** — `tickets` lacks `resolved_by_user_id`/`closed_by_user_id`; add both in Phase 1, enforce via typed-actor + exhaustive transition-matrix test, not prose.
3. **AI-outage fallback disables early-warning exactly when volume spikes** — embedding input (`summary_th`, etc.) is itself an AI output, so an outage that spikes ticket volume also silently kills incident detection at the worst possible moment. Needs an explicit design response, not just graceful per-ticket degradation.
4. **Thai-specific masking failures invisible to Latin-script tests** — Thai digits, unreliable `\b` boundaries, combining-character normalization, and missing person-name coverage all defeat naive regex; must test against Thai-specific cases, not just English PII samples.
5. **Confidence threshold `<0.70` likely never fires** — LLM verbalized confidence is systematically overconfident (research shows 88% stated vs. 79% actual); the manual-triage safety net FR-004 depends on may be silently disabled. Add non-confidence manual-triage triggers (category=OTHER, multiple_issues=true, schema-retry-occurred) alongside the raw threshold.

## Implications for Roadmap

### Phase 1: Foundation & Ticket Lifecycle
**Rationale:** Every later phase writes tickets and needs audit logging; retrofitting the state-machine guard and audit columns after 9 more phases have written to `tickets` is the single most expensive mistake to make late.
**Delivers:** Project scaffold, Postgres schema, demo auth (3 fixed-role accounts, no client-selectable role), ticket CRUD, guarded status state machine with exhaustive transition-matrix test, `resolved_by_user_id`/`closed_by_user_id` columns added to `tickets`, audit-log writer wired into every write path from day one.
**Addresses:** FR-001, FR-002 (partial), FR-013 foundation
**Avoids:** Pitfall #2 (unenforceable human gates), Architecture correction (retrofit cost)

### Phase 2: Job Harness & Idempotency
**Rationale:** AI analysis, embedding, and incident detection all need durable, retryable, queryable background execution; must exist before any AI call is made.
**Delivers:** `analysis_jobs` table with `FOR UPDATE SKIP LOCKED`, `Idempotency-Key` support on `POST /tickets`, retry/backoff scaffold.
**Uses:** FastAPI BackgroundTasks as trigger only; Postgres as the durable queue (Redis/Celery deferred)
**Implements:** Job harness component from Architecture research

### Phase 3: PII Masking
**Rationale:** Must precede both AI analysis and embedding calls; also must precede realistic seed data, which deliberately contains PII test cases.
**Delivers:** Regex-based masking for email/phone/person-ID/IP/secret with Thai-digit normalization, `MaskedText` typed boundary enforced at the LLM/embedding adapter interface, original-vs-masked storage separation, access-limited original-view with audit logging.
**Addresses:** FR-003
**Avoids:** Pitfall #4 (Thai-specific masking gaps)

### Phase 4: AI Ticket Analysis
**Rationale:** Now that masking and job harness exist, the AI call path can be built safely.
**Delivers:** Provider-agnostic LLM adapter, JSON-schema-constrained structured output with fallback ladder (strict mode → tool-call → repair-retry → manual queue), confidence handling, model/prompt-version/latency logging, resolved retry policy (1 transport retry for timeout/429, 1 schema-repair retry for invalid JSON — reconciling the §10.3-vs-Edge-Case-5 conflict as two distinct failure classes).
**Addresses:** FR-004, FR-005
**Avoids:** Pitfall #5 (confidence threshold miscalibration) — add non-confidence manual-triage triggers alongside the raw `<0.70` gate

### Phase 5: Rule Engine
**Rationale:** Runs after AI analysis per spec; independent of embedding/similarity, so can proceed in parallel with Phase 6 once Phase 4 lands.
**Delivers:** Versioned category→team and priority rules, `RuleSuggestion` as a type distinct from `tickets.priority`, Admin-only API gate on any `P1` write, admin UI for category→team mapping.
**Addresses:** FR-006
**Avoids:** P1-authority conflict (resolved architecturally: rule engine never writes to the guarded field)

### Phase 6: Embedding & Similarity Search
**Rationale:** Needs masked text (Phase 3) and stored analyses (Phase 4) as input; must be calibrated before the incident detector (Phase 7) is built on top of it.
**Delivers:** Embedding adapter (bge-m3 default), exact cosine similarity scan (no HNSW at this scale), **recalibrated ranking formula and threshold** validated against the seed/evaluation dataset with an explicit test proving a `location=null, category=OTHER` ticket can still cross threshold, `pg_trgm`-based search for FR-014.
**Addresses:** FR-007, FR-014 (search portion)
**Avoids:** Pitfall #1 (broken ranking formula) — this is the phase where it must be fixed, not merely implemented as specified

### Phase 7: Suspected Incident Detection
**Rationale:** Detector logic depends on a calibrated similarity score (Phase 6) and needs the incidents entity + human-confirm gate to exist first, since detection's only output is an insert into that gate.
**Delivers:** Incidents entity and `CONFIRMED`/`DISMISSED` human-approval workflow built first; sliding-window cluster detector as a pure, unit-testable function operating on similarity scores; demo-mode seeding that guarantees ≥3 distinct reporter accounts so Scenario C actually fires on stage.
**Addresses:** FR-008, FR-010 (partial)
**Avoids:** Pitfall #3 (AI-outage disables detection silently) — document/handle this failure mode explicitly here

### Phase 8: Human Triage, Incident Management & AI Drafts
**Rationale:** All upstream data (AI results, rule suggestions, similarity, incidents) now exists; this phase is pure UI/workflow assembly.
**Delivers:** Officer ticket-detail 3-panel layout with override+reason capture, incident management (severity/owner/timeline/root-cause), AI draft generation for responses/announcements with copy-to-clipboard only (no send path).
**Addresses:** FR-009, FR-010 (remainder), FR-011

### Phase 9: Dashboard, Search, Export & Audit Viewer
**Rationale:** Depends on all prior data being generated; pure read/aggregation layer.
**Delivers:** Dashboard metrics and filters, ticket/incident search using `pg_trgm`, CSV export excluding PII by default, Admin audit-log viewer.
**Addresses:** FR-012, FR-013 (viewer), FR-014 (remainder)

### Phase 10: Evaluation Harness & Demo Mode
**Rationale:** Needs the full pipeline working to generate meaningful evaluation data; closes the loop on the MVP's quality targets.
**Delivers:** `evaluation-set.json` (100 labeled samples), classification accuracy/F1, PII precision/recall, Recall@5, JSON-schema success rate, human-acceptance/override-rate reporting; finalized 60-ticket seed set; AI-outage simulation mode.
**Addresses:** §21 Test Requirements, §19 Seed Data, §20 Demo Scenarios

### Phase Ordering Rationale

- Phases 1–2 exist so nothing downstream needs retrofitting for audit, state-machine guards, or durable job execution — the single most-cited "expensive to fix late" pattern across all four research docs
- Phase 3 (masking) precedes Phase 4 (AI) and the eventual seed data because both need masked text and PII test cases respectively
- Phase 6 (embedding/similarity) must be calibrated with real data before Phase 7 (detector) builds on its output — this is the ordering fix that prevents shipping the mathematically broken formula unexamined
- Phase 7 builds the human-approval entity before the automated detector that feeds it, so the human gate is never bypassed even transiently during development
- Phases 5 and 6 have no dependency on each other and could run in parallel if `config.json`'s parallelization setting is used

### Research Flags

Phases likely needing deeper research during planning:
- **Phase 3 (PII masking):** Thai student/staff ID and mobile-number pattern formats are not specified anywhere in the spec or PROJECT.md — needs real-format research or a CITCOMS data request before regex patterns can be written
- **Phase 4 (AI analysis):** whether the actual chosen provider supports native JSON-schema-constrained output determines whether the repair-retry loop is even needed — provider-specific verification required
- **Phase 6 (Embedding/similarity):** embedding model final choice and ranking-formula/threshold calibration are both empirical questions that can only be answered against the project's own seed and evaluation data, not from published benchmarks alone
- **Phase 7 (Incident detection):** FR-008's "average final similarity score" aggregation method (pairwise mean vs. centroid vs. vs-newest) is undefined in the spec and must be decided before implementation

Phases with standard, well-documented patterns (skip research-phase):
- **Phase 1 (Foundation):** standard FastAPI/Postgres/state-machine patterns
- **Phase 2 (Job harness):** well-established Postgres-as-queue pattern (`FOR UPDATE SKIP LOCKED`)
- **Phase 5 (Rule engine):** build-don't-import is the clear conclusion; no complex integration
- **Phase 8 (Human triage UI):** standard CRUD/form patterns
- **Phase 9 (Dashboard/export):** standard reporting patterns

## Confidence Assessment

| Area | Confidence | Notes |
|------|------------|-------|
| Stack | HIGH | Versions and pgvector behavior verified via Context7/official docs; frontier LLM model naming is MEDIUM only |
| Features | MEDIUM-HIGH | Verified against ServiceNow/Freshservice/Zendesk/JSM vendor docs and incident.io design rationale |
| Architecture | MEDIUM-HIGH | Component boundaries and data-flow patterns well-supported; specific enforcement techniques (typed-actor, AST-guard) are engineering judgment, marked as such in source doc |
| Pitfalls | MEDIUM-HIGH | Embedding/similarity math and Thai-language issues are HIGH (peer-reviewed sources with hard numbers); alert-fatigue magnitudes are LOW (vendor marketing figures, order-of-magnitude only) |

**Overall confidence:** MEDIUM-HIGH

### Gaps to Address

- Thai PII pattern formats (student ID, staff ID, mobile number) — not in any source document; needs a real-format request to CITCOMS or a Phase 3 spike before regex patterns can be finalized
- Ranking formula and similarity threshold — spec's numbers (`0.65/0.15/0.10/0.10`, `0.82`) are plausible defaults, not measured; must be calibrated against the 60-ticket seed set in Phase 6 with a specific test proving low-confidence tickets remain reachable
- FR-008's "average final similarity score" aggregation method — genuinely undefined in the spec, needs a product/engineering decision during Phase 7 planning
- Person-name PII masking — not in FR-003's scope at all, but affects the accuracy of FR-014's "PII-free export" claim; needs an explicit accept-the-risk decision or a Phase 3 addition
- Secret-storage contradiction in FR-003 ("never store secrets in plaintext" vs. "original text stored separately") — needs a product decision, likely: irreversibly redact secret spans even from the "original" record

## Sources

### Primary (HIGH confidence)
- Presidio (Context7) — PII anonymization patterns, Thai NER gap confirmed
- pgvector / pgvector-python (Context7) — index behavior, filter-after-scan issue, dimension limits
- OpenAI Structured Outputs docs — strict-mode JSON schema behavior and limitations
- OWASP LLM06 Excessive Agency / Prompt Injection Prevention cheat sheets — authorization placement requirements
- W3C Thai Gap Analysis, Unicode 16 ch.16 — Thai script tokenization and normalization issues
- SEA-BED (arXiv 2508.12243) — Thai embedding benchmark across 17 models
- BGE-M3 paper (arXiv 2402.03216) — MIRACL Thai performance numbers

### Secondary (MEDIUM confidence)
- ServiceNow, Freshservice, Zendesk, JSM, incident.io vendor documentation — human-approval-gate design patterns
- llama.cpp / Ollama GitHub issues — structured-output provider-compatibility gaps
- Cross-lingual anisotropy paper (arXiv 2306.00458) — negative-pair similarity inflation in multilingual embeddings

### Tertiary (LOW confidence)
- AIOps vendor blogs (Rootly, AiOps School) — alert-fatigue magnitude figures, order-of-magnitude only, not verified against primary data

---
*Research completed: 2026-08-18*
*Ready for roadmap: yes*
