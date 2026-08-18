# Roadmap: UP IT Pulse

## Overview

UP IT Pulse is built from the trust boundary outward. Phase 1 lays the authoritative spine — demo auth with server-bound roles, the ticket entity, a guarded status state machine that only a typed human actor can advance past `AI_ANALYZED`, durable background-job execution, and an audit writer wired into every write path — because every later phase writes to `tickets` and retrofitting those guards is the single most expensive late mistake. Phase 2 installs the PII masking boundary before any text can reach a model. Phases 3–5 build the suggestion layers on top of that boundary: AI analysis plus rule-engine priority/team suggestions (never written to confirmed fields), then embeddings and a *calibrated* similarity ranking, then sliding-window cluster detection feeding a human-only incident confirmation gate. Phase 6 assembles the officer's working surface — three-panel provenance, overrides with reasons, and copy-only AI drafts. Phase 7 adds the read layer (dashboard, Thai-capable search, PII-free CSV export, audit viewer). Phase 8 proves it: a labeled evaluation set, measured accuracy/masking/recall numbers, the 60-ticket seed corpus, and an AI-outage demo mode. Throughout, the invariant is the core value: AI suggests, rules explain, humans decide, everything is auditable.

## Phases

**Phase Numbering:**
- Integer phases (1, 2, 3): Planned milestone work
- Decimal phases (2.1, 2.2): Urgent insertions (marked with INSERTED)

Decimal phases appear between their surrounding integers in numeric order.

- [ ] **Phase 1: Foundation, Auth & Ticket Lifecycle** - Docker-Compose stack, server-bound demo roles, ticket intake with idempotency, guarded state machine, audit spine
- [ ] **Phase 2: PII Masking Boundary** - Regex masking for Thai/English PII enforced as a typed precondition on every model call
- [ ] **Phase 3: AI Analysis & Rule Suggestions** - Schema-constrained AI analysis with fallback ladder, plus versioned priority/team rules as suggestions only
- [ ] **Phase 4: Embedding & Similarity Search** - Embeddings over masked text and a recalibrated ranking formula proven against seed data
- [ ] **Phase 5: Incident Detection & Management** - Sliding-window cluster alerts feeding an Admin-only incident confirmation and lifecycle workflow
- [ ] **Phase 6: Human Triage & AI Drafts** - Officer three-panel triage with override capture, plus labeled copy-only AI drafts
- [ ] **Phase 7: Dashboard, Search, Export & Audit Viewer** - Metrics, Thai-capable search, PII-free CSV export, and the Admin audit log viewer
- [ ] **Phase 8: Evaluation Harness & Demo Readiness** - Labeled evaluation set, measured quality numbers, 60-ticket seed corpus, AI-outage demo mode

## Phase Details

### Phase 1: Foundation, Auth & Ticket Lifecycle
**Goal**: A reporter can log in and submit a ticket that immediately returns a tracking number, while every read and write is role-checked, state-machine-guarded, and audit-logged
**Mode:** mvp
**Depends on**: Nothing (first phase)
**Requirements**: AUTH-01, AUTH-02, AUTH-03, AUTH-04, AUTH-05, TICK-01, TICK-02, TICK-03, TICK-04, TICK-05, TICK-06, AUDIT-01, AUDIT-02
**Success Criteria** (what must be TRUE):
  1. The stack (`web`, `api`, `postgres`) starts with a single `docker compose up`, and each of the three demo credentials logs in to its own fixed role — sending a different role in the request body or token payload does not change the effective role
  2. A reporter submits a valid ticket (title 5–150, description 10–5,000, optional category/location/started_at/impact_scope/contact/attachments PNG-JPG-PDF ≤5 files ≤10 MB) and sees a `UPIT-YYYY-NNNNN` number with status `NEW` within 2 seconds without waiting on AI; replaying the same request with the same `Idempotency-Key` returns that same ticket instead of creating a second one
  3. A reporter requesting another reporter's ticket is denied, while Officer and Admin see tickets per their configured visibility scope
  4. Illegal status transitions are refused by the API (e.g. `NEW → CLOSED`, Officer attempting `CLOSED`, or any AI-actor transition past `AI_ANALYZED`); `RESOLVED` records the acting Officer, `CLOSED` records the acting Admin, and a reporter can request reopen of a `CLOSED` ticket only inside the configured reopen window
  5. For every one of the above actions an append-only audit row exists with actor, action, target and a UTC timestamp that renders as `Asia/Bangkok`
**Plans**: TBD
**UI hint**: yes

### Phase 2: PII Masking Boundary
**Goal**: No unmasked ticket text can reach an LLM or embedding provider, and revealing original text is a privileged, audited act
**Mode:** mvp
**Depends on**: Phase 1
**Requirements**: MASK-01, MASK-02, MASK-03, MASK-04, MASK-05
**Success Criteria** (what must be TRUE):
  1. A ticket containing email addresses, phone numbers in both Latin and Thai digits (`๐-๙`), student/staff ID patterns, IPv4 and IPv6 addresses, and an API token/password produces masked text where each occurrence is replaced by `[EMAIL_n]`, `[PHONE_n]`, `[PERSON_ID_n]`, `[IP_n]`, `[SECRET_n]` — verified against a Thai-script test set, not only Latin-script samples
  2. Original and masked text are stored as separate fields, and the Officer/Admin ticket view labels exactly which version the AI was given
  3. An Officer without elevated permission cannot reveal the original text; when an authorized user does reveal it, an audit row names the viewer, the ticket, and the time
  4. Inspecting stored rows for a secret-bearing ticket shows no plaintext token or password in either the masked *or* the original record
  5. Any attempt to call the LLM/embedding port with text that has not passed through masking is rejected at the type/contract level rather than silently proceeding
**Plans**: TBD
**UI hint**: yes

### Phase 3: AI Analysis & Rule Suggestions
**Goal**: Every ticket receives schema-valid AI analysis and an explainable priority/team suggestion, while confirmed fields stay under human authority and AI failure degrades into a manual queue
**Mode:** mvp
**Depends on**: Phase 2
**Requirements**: AIAN-01, AIAN-02, AIAN-03, AIAN-04, AIAN-05, AIAN-06, RULE-01, RULE-02, RULE-03, RULE-04
**Success Criteria** (what must be TRUE):
  1. Submitting a ticket produces a stored analysis validated against the fixed schema (summary_th, category, subcategory, affected_service, location, started_at, impact_scope, priority_suggestion, team_suggestion, missing_information, keywords, confidence, rationale_th) with undeterminable fields as `null`/`UNKNOWN`, and the ticket advances to `AI_ANALYZED` and no further
  2. A "Needs manual triage" flag appears for tickets triggered by low confidence (`<0.70`) *or* a non-confidence trigger (`category=OTHER`, multiple issues, schema-repair retry occurred), and an Officer can override any AI-suggested field only by supplying a reason that is recorded
  3. The ticket view shows the rule engine's `P1`–`P4` and team output as a labeled *suggestion*; the ticket's confirmed priority/team remain unset until a human accepts, and writing a confirmed `P1` is refused for any non-Officer/Admin actor
  4. An Admin edits category→team mapping and priority rules in the Admin UI; the change is versioned and audit-logged, and subsequent tickets receive suggestions from the new version
  5. With the provider forced to fail (timeout, 429, invalid JSON, unavailable), the ticket is still created, stays `NEW`, shows "AI analysis unavailable", and lands in the manual triage queue after the defined retry policy; every successful analysis records model name, prompt version, latency, and call timestamp
**Plans**: TBD
**UI hint**: yes

### Phase 4: Embedding & Similarity Search
**Goal**: An Officer sees trustworthy similar-ticket candidates whose threshold is proven reachable for exactly the low-confidence tickets that need duplicate detection most
**Mode:** mvp
**Depends on**: Phase 3
**Requirements**: SIM-01, SIM-02, SIM-03, SIM-04, SIM-05
**Success Criteria** (what must be TRUE):
  1. Each analyzed ticket has an embedding derived from its masked `summary_th + category + affected_service + location`
  2. The ticket detail view lists the Top 5 most similar tickets from the default 24-hour lookback with similarity score, time, category, location, and status, and differently-worded near-duplicate seed tickets actually appear in that list
  3. A test against the seed/evaluation data proves a genuine duplicate with `location=null` and `category=OTHER` still crosses the duplicate threshold — i.e. the ranking formula and threshold are calibrated from measured data, not adopted unexamined
  4. An Officer marks a candidate `Related`, `Duplicate`, or `Not related`; marking `Duplicate` leaves both tickets open and records who marked it and when
  5. Changing the similarity threshold through configuration (no code change) visibly changes duplicate flagging on re-query
**Plans**: TBD
**UI hint**: yes

### Phase 5: Incident Detection & Management
**Goal**: Ticket clusters raise a SUSPECTED alert that only an Admin can turn into a confirmed incident, and an Admin can run that incident through its full lifecycle
**Mode:** mvp
**Depends on**: Phase 4
**Requirements**: INCD-01, INCD-02, INCD-03, INCD-04, INCD-05, INCD-06
**Success Criteria** (what must be TRUE):
  1. A seeded cluster meeting all configured conditions (minimum ticket count, minimum distinct reporters, sliding window, matching category/affected service, minimum aggregate similarity) raises exactly one `SUSPECTED` alert, and tickets already linked to an active incident do not spawn a duplicate alert
  2. The alert shows ticket and distinct-reporter counts, detection window, category/service/location, evidence tickets, and its confidence/reasoning, with `Confirm incident` / `Dismiss` / `Snooze` actions — and nowhere describes itself as a confirmed outage
  3. An Admin confirming an alert moves it to `CONFIRMED` with confirming user, timestamp, and evidence tickets recorded; an Officer attempting to confirm is refused
  4. An Admin creates an incident manually from a ticket, adds and removes linked tickets, sets severity/owner/start time/affected service, logs timeline entries and internal notes, and walks the status through `SUSPECTED → CONFIRMED → MONITORING → RESOLVED → CLOSED` (or `→ DISMISSED`), with each change audit-logged
  5. An Admin records root cause and resolution on a resolved incident and it is visible on the incident view
**Plans**: TBD
**UI hint**: yes

### Phase 6: Human Triage & AI Drafts
**Goal**: An Officer works a ticket end-to-end from one screen where AI, rule, and human provenance are visibly separate, and every outbound message is a labeled draft the human copies
**Mode:** mvp
**Depends on**: Phase 5
**Requirements**: TRIAGE-01, TRIAGE-02, TRIAGE-03, TRIAGE-04, TRIAGE-05, AIAN-07, DRAFT-01, DRAFT-02, DRAFT-03
**Success Criteria** (what must be TRUE):
  1. The ticket detail view presents AI results, rule-engine results, and similarity results as three distinct, separately labeled sections, with human-confirmed values shown apart from all three
  2. An Officer accepts or overrides category, priority, and team and assigns the ticket to a team/user; each override stores its reason plus before/after values as queryable feedback data
  3. Requesting more information surfaces the AI-identified `missing_information` and generates an editable draft follow-up question, and the ticket moves to `WAITING_USER`
  4. An Officer links and unlinks a ticket to/from an incident, and the change is reflected on the incident's linked-ticket list
  5. An Officer generates each draft type (request-for-info, acknowledgement, investigating notice, incident announcement with its fixed structure of service / symptoms / affected group-area / estimated start time / current status / temporary guidance / last updated, and service-restored), sees the "AI-generated draft" label, edits the text, and copies it to the clipboard — no send or publish control exists anywhere in the UI or API
**Plans**: TBD
**UI hint**: yes

### Phase 7: Dashboard, Search, Export & Audit Viewer
**Goal**: CITCOMS staff can see the whole operational picture, find any ticket in Thai or English, export it without PII, and inspect who did what
**Mode:** mvp
**Depends on**: Phase 6
**Requirements**: DASH-01, DASH-02, SRCH-01, SRCH-02, SRCH-03, AUDIT-03
**Success Criteria** (what must be TRUE):
  1. The dashboard shows today's tickets by status, tickets by category and priority, tickets awaiting manual triage, active suspected and confirmed incidents, potential-duplicate count, average time-to-triage, and AI acceptance/override rate
  2. Applying date-range, category, priority, team, and status filters changes every dashboard panel and list consistently
  3. Searching by ticket ID, by Thai-language title/description text, and by English text all return the matching tickets, scoped to what the caller is authorized to see
  4. Exporting a filtered ticket or incident list produces a CSV that omits PII columns by default
  5. An Admin opens the audit log viewer and filters by actor, action type, and date range, with timestamps displayed in `Asia/Bangkok`
**Plans**: TBD
**UI hint**: yes

### Phase 8: Evaluation Harness & Demo Readiness
**Goal**: The MVP's quality claims are measured rather than asserted, and the hackathon demo runs reliably including the AI-outage scenario
**Mode:** mvp
**Depends on**: Phase 7
**Requirements**: EVAL-01, EVAL-02, EVAL-03, EVAL-04
**Success Criteria** (what must be TRUE):
  1. `evaluation-set.json` exists in the repo with at least 100 human-labeled samples carrying expected category, expected priority range, expected missing fields, duplicate group ID, and PII spans
  2. A single command produces a report of classification accuracy/F1, PII masking precision/recall, duplicate-retrieval Recall@5, JSON-schema success rate, and human acceptance/override rate against that set
  3. Loading seed data creates at least 60 tickets spanning all 10 service categories and covering normal, incomplete-information, mixed Thai/English, differently-worded duplicate, PII-bearing, and deliberately ambiguous samples plus an incident cluster with enough distinct reporter accounts to actually fire `SUSPECTED` detection — with no real names, phone numbers, emails, student IDs, or IPs anywhere
  4. With "Simulate AI outage" enabled, a reporter still successfully creates a ticket and that ticket appears in the manual triage queue with "AI analysis unavailable"
**Plans**: TBD

## Progress

**Execution Order:**
Phases execute in numeric order: 1 → 2 → 3 → 4 → 5 → 6 → 7 → 8

| Phase | Plans Complete | Status | Completed |
|-------|----------------|--------|-----------|
| 1. Foundation, Auth & Ticket Lifecycle | 0/TBD | Not started | - |
| 2. PII Masking Boundary | 0/TBD | Not started | - |
| 3. AI Analysis & Rule Suggestions | 0/TBD | Not started | - |
| 4. Embedding & Similarity Search | 0/TBD | Not started | - |
| 5. Incident Detection & Management | 0/TBD | Not started | - |
| 6. Human Triage & AI Drafts | 0/TBD | Not started | - |
| 7. Dashboard, Search, Export & Audit Viewer | 0/TBD | Not started | - |
| 8. Evaluation Harness & Demo Readiness | 0/TBD | Not started | - |

## Notes

**Requirement coverage:** 58/58 v1 requirements mapped, each to exactly one phase.

**Divergence from the 10-phase sketch in research/SUMMARY.md** (granularity is `standard` = 5–8 phases):
- Research Phase 2 (Job Harness & Idempotency) merged into **Phase 1**. Its only user-visible requirement is TICK-03 (`Idempotency-Key`); the `analysis_jobs` table with `FOR UPDATE SKIP LOCKED` is infrastructure with no observable user behavior of its own, so it belongs to the foundation phase that already owns the write path.
- Research Phase 5 (Rule Engine) merged into **Phase 3** with AI analysis. Both produce provenance-tagged *suggestions* that must never touch confirmed fields — that is one architectural boundary, and splitting it invites the P1-authority conflict to be resolved twice. Parallelism is preserved at the plan level (`parallelization: true`), so rule-engine plans can still run alongside AI-analysis plans.
- Research Phases 7 and 8 split incident work across two phases (entity + detector, then management UI). Consolidated into **Phase 5** so the human-confirm gate, the detector, and the incident lifecycle ship as one verifiable capability — the research's own ordering constraint ("build the approval entity before the detector") is satisfied inside the phase.
- AIAN-07 (draft follow-up question for missing information) is mapped to **Phase 6**, not Phase 3, because its observable behavior is draft generation alongside TRIAGE-02 and DRAFT-01. Phase 3 still stores the `missing_information` field via AIAN-01.
- `pg_trgm` search (research Phase 6) is mapped to **Phase 7** with the other SRCH requirements, so Thai-language search is verified as one user-facing capability rather than half-built early.

**Requirement count correction:** REQUIREMENTS.md previously stated 54 v1 requirements; the actual traceability table lists 58. Corrected in that file.

**Research flags carried into planning** (from research/SUMMARY.md):
- Phase 2: Thai student/staff ID and mobile-number formats are unspecified — needs a format spike or CITCOMS data request before regex is written. Person-name masking is not in scope of MASK-01 but affects SRCH-03's PII-free claim; needs an explicit accept-the-risk decision.
- Phase 3: whether the chosen provider truly honors `response_format: json_schema, strict: true` determines whether the repair-retry rung is needed.
- Phase 4: embedding model choice and ranking-formula/threshold calibration are empirical — answerable only against this project's own seed/evaluation data.
- Phase 5: FR-008's "average similarity" aggregation method (pairwise mean vs. centroid vs. vs-newest) is undefined in the spec and must be decided during planning. Also document the AI-outage failure mode that silently disables detection when volume spikes.
- Phase 2 note: the "never store secrets in plaintext" vs. "original text stored separately" contradiction needs a product decision (likely irreversible redaction of secret spans even in the original record).

---
*Roadmap created: 2026-08-18*
