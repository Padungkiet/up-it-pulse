# Requirements: UP IT Pulse

**Defined:** 2026-08-18
**Core Value:** AI suggests, rules explain, humans decide, and every action is auditable — the system must never let AI close a ticket, confirm a major incident, publish an announcement, or perform any production/account action on its own.

## v1 Requirements

Requirements for initial release. Each maps to roadmap phases.

### Authentication & Authorization

- [ ] **AUTH-01**: User can log in with a fixed-role demo account (`REPORTER`, `OFFICER`, or `ADMIN`) — role is bound server-side to the credential, never client-selectable
- [ ] **AUTH-02**: Every API endpoint that reads or writes data enforces role-based authorization
- [ ] **AUTH-03**: Reporter cannot view tickets submitted by other reporters
- [ ] **AUTH-04**: Officer and Admin see tickets according to their configured visibility scope
- [ ] **AUTH-05**: Every security-relevant action (login, sensitive-data access, ticket change, AI override, incident action, config change) is recorded in the audit log

### Ticket Lifecycle

- [ ] **TICK-01**: Reporter can submit a ticket with title (5–150 chars), description (10–5,000 chars), and optional category/location/started_at/impact_scope/contact info/attachments (PNG/JPG/PDF, max 5 files, 10 MB each)
- [ ] **TICK-02**: On successful submit, system generates a ticket ID formatted `UPIT-YYYY-NNNNN`, sets status `NEW`, and shows the number to the reporter immediately (within 2 seconds, without waiting on AI)
- [ ] **TICK-03**: `POST /tickets` accepts an `Idempotency-Key` header to prevent duplicate ticket creation on repeated submits
- [ ] **TICK-04**: Ticket status transitions are enforced by a guarded state machine: `NEW → AI_ANALYZED → TRIAGED → IN_PROGRESS → WAITING_USER → RESOLVED → CLOSED`; AI-driven transitions may only reach `AI_ANALYZED`
- [ ] **TICK-05**: Only an Officer can transition a ticket to `RESOLVED`; only an Admin can transition a ticket to `CLOSED`; both actions record the acting user
- [ ] **TICK-06**: Reporter can request to reopen a `CLOSED` ticket within a configured reopen window

### PII Masking

- [ ] **MASK-01**: Before any text reaches the LLM or embedding service, the system masks email → `[EMAIL_n]`, phone numbers → `[PHONE_n]`, student/staff ID patterns → `[PERSON_ID_n]`, IPv4/IPv6 → `[IP_n]`, and secrets/tokens/passwords → `[SECRET_n]`
- [ ] **MASK-02**: Original ticket text and masked ticket text are stored separately
- [ ] **MASK-03**: Officer/Admin UI indicates which text version (original or masked) the AI actually saw
- [ ] **MASK-04**: Viewing original (unmasked) ticket text requires elevated permission and is recorded in the audit log
- [ ] **MASK-05**: Detected secrets are never persisted in plain text

### AI Ticket Analysis

- [ ] **AIAN-01**: AI returns ticket analysis as JSON matching the fixed schema (summary_th, category, subcategory, affected_service, location, started_at, impact_scope, priority_suggestion, team_suggestion, missing_information, keywords, confidence, rationale_th); backend validates against the schema before saving
- [ ] **AIAN-02**: Fields the AI cannot determine are returned as `null`/`UNKNOWN`, never guessed
- [ ] **AIAN-03**: Tickets with `confidence < 0.70` (or another manual-triage trigger such as `category=OTHER`, `multiple_issues=true`, or a schema-retry having occurred) are flagged "Needs manual triage"
- [ ] **AIAN-04**: Officer can edit or override every AI-suggested field; overriding requires a reason, which is recorded
- [ ] **AIAN-05**: Each AI analysis call records model name, prompt version, latency, and call timestamp
- [ ] **AIAN-06**: If AI analysis fails (timeout, rate-limit, invalid JSON, service unavailable), the ticket is still created, remains in `NEW`, displays "AI analysis unavailable," and routes to a manual triage queue after a defined retry policy (transport retry with backoff, plus a schema-repair retry for invalid JSON)
- [ ] **AIAN-07**: System surfaces AI-identified missing information and can generate an editable draft follow-up question requesting it

### Rule Engine

- [ ] **RULE-01**: A rule engine runs after AI analysis and suggests priority (`P1`–`P4`) and responsible team based on configurable category→team mapping and impact-scope/incident-status conditions
- [ ] **RULE-02**: Rule engine output is a suggestion only — it is never written directly to a ticket's confirmed priority/team, and it cannot close a ticket or confirm an incident
- [ ] **RULE-03**: Setting a ticket's confirmed priority to `P1` requires explicit human (Officer/Admin) action, even when the rule engine suggests it
- [ ] **RULE-04**: Admin can edit category→team mapping and priority rules from an Admin UI; rule changes are versioned and audit-logged

### Similarity Search

- [ ] **SIM-01**: System generates an embedding from masked `summary_th + category + affected_service + location` for each analyzed ticket
- [ ] **SIM-02**: System finds and displays the Top 5 most similar tickets (default lookback: last 24 hours) with similarity score, time, category, location, and status
- [ ] **SIM-03**: Similarity ranking and duplicate-detection threshold are calibrated against real seed/evaluation data so that low-confidence tickets (e.g. `location=null`, `category=OTHER`) remain reachable as potential duplicates, not structurally excluded by the scoring formula
- [ ] **SIM-04**: Officer can mark a similar ticket as `Related`, `Duplicate`, or `Not related`; marking `Duplicate` never auto-closes either ticket
- [ ] **SIM-05**: Similarity threshold is configurable (default `0.82`) without a code change

### Suspected Incident Detection

- [ ] **INCD-01**: System raises a `SUSPECTED` incident alert when a ticket cluster meets all configured conditions (minimum ticket count, minimum distinct reporters, sliding time window, matching category/affected service, minimum aggregate similarity) and is not already linked to another active incident
- [ ] **INCD-02**: Alert displays ticket/reporter counts, detection window, category/service/location, evidence tickets, confidence/reasoning, and `Confirm incident` / `Dismiss` / `Snooze` actions
- [ ] **INCD-03**: An alert is never described as a confirmed outage until an Admin explicitly confirms it
- [ ] **INCD-04**: Admin can create an incident from a `SUSPECTED` alert or manually from any ticket; can add/remove linked tickets, set severity/owner/start time/affected service, log timeline and internal notes, and change incident status (`SUSPECTED → CONFIRMED → MONITORING → RESOLVED → CLOSED`, or `→ DISMISSED`)
- [ ] **INCD-05**: Confirming an incident records the confirming user, timestamp, and evidence tickets
- [ ] **INCD-06**: Admin can record root cause and resolution once an incident is resolved

### Human Triage

- [ ] **TRIAGE-01**: Officer can accept or override AI-suggested category, priority, and team, and assign a ticket to a team/user
- [ ] **TRIAGE-02**: Officer can request additional information from the reporter
- [ ] **TRIAGE-03**: Officer can link or unlink a ticket to/from an incident
- [ ] **TRIAGE-04**: Ticket detail view shows AI results, rule-engine results, and similarity results as distinct, separately labeled sections
- [ ] **TRIAGE-05**: Every AI-suggestion override is captured as feedback data for future model evaluation

### AI Drafts

- [ ] **DRAFT-01**: AI can generate drafts for: request-for-more-information, acknowledgement, investigating notice, incident announcement (fixed structure: service, symptoms, affected group/area, estimated start time, current status, temporary guidance, last updated), and service-restored message
- [ ] **DRAFT-02**: Every AI-generated draft is labeled "AI-generated draft" and is editable before use
- [ ] **DRAFT-03**: MVP draft delivery is "Copy to clipboard" only — no draft is ever sent or published automatically

### Dashboard

- [ ] **DASH-01**: Dashboard shows today's ticket count by status, tickets by category and priority, tickets awaiting manual triage, active suspected/confirmed incidents, potential-duplicate count, average time-to-triage, and AI acceptance/override rate
- [ ] **DASH-02**: Dashboard supports filtering by date range, category, priority, team, and ticket/incident status

### Audit Log

- [ ] **AUDIT-01**: Audit log is append-only at the application level and covers login/sensitive-data access, ticket create/update/assign/status-change, AI analysis with prompt version, AI-suggestion accept/override, incident confirm/dismiss/status-change, draft-announcement approval, and rule/threshold configuration changes
- [ ] **AUDIT-02**: Audit log timestamps are stored in UTC and displayed in `Asia/Bangkok`
- [ ] **AUDIT-03**: Admin can view and filter the audit log

### Search & Export

- [ ] **SRCH-01**: User can search tickets by ID, title, and description, scoped to tickets they are authorized to access — search must return usable results for Thai-language text, not just English
- [ ] **SRCH-02**: User can filter tickets/incidents by category, priority, team, status, and date
- [ ] **SRCH-03**: User can export filtered tickets/incidents to CSV; export excludes PII by default

### Evaluation & Demo Readiness

- [ ] **EVAL-01**: Project includes an `evaluation-set.json` with at least 100 human-labeled samples (expected category, expected priority range, expected missing fields, duplicate group ID, PII spans)
- [ ] **EVAL-02**: Project reports classification accuracy/F1, PII masking precision/recall, duplicate-retrieval Recall@5, JSON schema success rate, and human acceptance/override rate against the evaluation set
- [ ] **EVAL-03**: Seed data includes at least 60 tickets across all service categories, covering normal tickets, incomplete-information tickets, mixed Thai/English tickets, worded-differently duplicates, PII-bearing samples for masking tests, an incident-cluster set (with enough distinct reporters to trigger `SUSPECTED` detection), and deliberately ambiguous samples — using no real names, phone numbers, emails, student IDs, or IPs
- [ ] **EVAL-04**: A "Simulate AI outage" demo mode lets a reporter still create a ticket and see it routed to the manual triage queue

## v2 Requirements

Deferred to future release. Tracked but not in current roadmap.

### Integrations

- **INTG-01**: Real UP Account/SSO integration via OIDC/SAML
- **INTG-02**: Ticket intake from Email, LINE, Facebook, or phone systems
- **INTG-03**: Ingest real Syslog/SNMP/SIEM/monitoring signals
- **INTG-04**: Automated status-page publishing

### AI Enhancements

- **AIEN-01**: OCR extraction from attached screenshots
- **AIEN-02**: Voice transcription for ticket intake
- **AIEN-03**: Knowledge-base/RAG-backed resolution suggestions from CITCOMS documentation
- **AIEN-04**: Feedback-driven model evaluation and prompt-version comparison using override data collected in v1

### Platform

- **PLAT-01**: Multi-tenant support for multiple faculties/departments
- **PLAT-02**: Formal SLA/OLA configuration and enforcement

## Out of Scope

Explicitly excluded. Documented to prevent scope creep.

| Feature | Reason |
|---------|--------|
| Real Helpdesk/Email/LINE/Facebook/phone integration | MVP simulates intake via web form only |
| Real UP Account/SSO integration | MVP uses fixed demo accounts per role |
| Reading real Syslog/SNMP/SIEM/monitoring feeds | No live infrastructure signal ingestion in MVP |
| Automated remediation (restart server, block IP, reset password, change config) | Human-only actions per AI guardrails — a core product principle, not a phase-1 gap |
| Automatic announcement publishing (Email/SMS/LINE) | Drafts are copy-to-clipboard only in MVP |
| Automatic ticket/incident closing by AI | Closing is always a human, audited action |
| Automated root-cause confirmation by AI | AI may suggest a rationale, never confirm causation |
| Mobile native application | Responsive web only for MVP |
| Voice transcription | Text-only intake for MVP |
| OCR from screenshot attachments | Attachments stored; MVP analyzes ticket text only |
| Full SLA management | MVP tracks time-to-triage as a metric only, not an enforced SLA |
| Auto-merge tickets above the duplicate similarity threshold | Destructive — risks silently discarding a real reporter's distinct issue (research finding) |
| Sentiment analysis on ticket text | No triage value for this use case; adds complexity without addressing MVP success criteria (research finding) |
| Free-text "ask AI about our tickets" chat panel | Would let AI answer from unmasked ticket content, undermining the PII-masking guarantee (research finding) |

## Traceability

Which phases cover which requirements. Updated during roadmap creation.

| Requirement | Phase | Status |
|-------------|-------|--------|
| AUTH-01 | Phase 1 | Pending |
| AUTH-02 | Phase 1 | Pending |
| AUTH-03 | Phase 1 | Pending |
| AUTH-04 | Phase 1 | Pending |
| AUTH-05 | Phase 1 | Pending |
| TICK-01 | Phase 1 | Pending |
| TICK-02 | Phase 1 | Pending |
| TICK-03 | Phase 2 | Pending |
| TICK-04 | Phase 1 | Pending |
| TICK-05 | Phase 1 | Pending |
| TICK-06 | Phase 1 | Pending |
| MASK-01 | Phase 3 | Pending |
| MASK-02 | Phase 3 | Pending |
| MASK-03 | Phase 3 | Pending |
| MASK-04 | Phase 3 | Pending |
| MASK-05 | Phase 3 | Pending |
| AIAN-01 | Phase 4 | Pending |
| AIAN-02 | Phase 4 | Pending |
| AIAN-03 | Phase 4 | Pending |
| AIAN-04 | Phase 4 | Pending |
| AIAN-05 | Phase 4 | Pending |
| AIAN-06 | Phase 4 | Pending |
| AIAN-07 | Phase 4 | Pending |
| RULE-01 | Phase 5 | Pending |
| RULE-02 | Phase 5 | Pending |
| RULE-03 | Phase 5 | Pending |
| RULE-04 | Phase 5 | Pending |
| SIM-01 | Phase 6 | Pending |
| SIM-02 | Phase 6 | Pending |
| SIM-03 | Phase 6 | Pending |
| SIM-04 | Phase 6 | Pending |
| SIM-05 | Phase 6 | Pending |
| INCD-01 | Phase 7 | Pending |
| INCD-02 | Phase 7 | Pending |
| INCD-03 | Phase 7 | Pending |
| INCD-04 | Phase 7 | Pending |
| INCD-05 | Phase 7 | Pending |
| INCD-06 | Phase 7 | Pending |
| TRIAGE-01 | Phase 8 | Pending |
| TRIAGE-02 | Phase 8 | Pending |
| TRIAGE-03 | Phase 8 | Pending |
| TRIAGE-04 | Phase 8 | Pending |
| TRIAGE-05 | Phase 8 | Pending |
| DRAFT-01 | Phase 8 | Pending |
| DRAFT-02 | Phase 8 | Pending |
| DRAFT-03 | Phase 8 | Pending |
| DASH-01 | Phase 9 | Pending |
| DASH-02 | Phase 9 | Pending |
| AUDIT-01 | Phase 1 | Pending |
| AUDIT-02 | Phase 1 | Pending |
| AUDIT-03 | Phase 9 | Pending |
| SRCH-01 | Phase 9 | Pending |
| SRCH-02 | Phase 9 | Pending |
| SRCH-03 | Phase 9 | Pending |
| EVAL-01 | Phase 10 | Pending |
| EVAL-02 | Phase 10 | Pending |
| EVAL-03 | Phase 10 | Pending |
| EVAL-04 | Phase 10 | Pending |

**Coverage:**
- v1 requirements: 54 total
- Mapped to phases: 54
- Unmapped: 0 ✓

---
*Requirements defined: 2026-08-18*
*Last updated: 2026-08-18 after initial definition*
