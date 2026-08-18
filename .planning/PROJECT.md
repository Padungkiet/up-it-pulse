# UP IT Pulse

## What This Is

A web application for CITCOMS (ศูนย์บริการเทคโนโลยีสารสนเทศและการสื่อสาร) at University of Phayao that lets students, staff, and IT officers report IT incidents in Thai or English, uses AI to classify and extract structured ticket data, detects duplicate/similar tickets and emerging major incidents, and drafts (but never auto-sends) responses and announcements. Hackathon MVP — v1.0.0.

## Core Value

AI suggests, rules explain, humans decide, and every action is auditable — the system must never let AI close a ticket, confirm a major incident, publish an announcement, or perform any production/account action on its own.

## Requirements

### Validated

(None yet — ship to validate)

### Active

**Ticket intake & auth**
- [ ] Reporter can submit a ticket via web form with title (5–150 chars), description (10–5,000 chars), optional category/location/started_at/impact_scope/contact info/attachments (PNG/JPG/PDF, max 5 files, 10 MB each)
- [ ] System generates ticket ID as `UPIT-YYYY-NNNNN`, sets status `NEW`, shows the number to the reporter immediately
- [ ] `POST /tickets` supports `Idempotency-Key` to prevent duplicate submits
- [ ] Demo login supports three roles: `REPORTER`, `OFFICER`, `ADMIN`, each with a fixed demo account
- [ ] Every API enforces role-based authorization; Reporter cannot view other reporters' tickets
- [ ] Ticket status workflow enforced: `NEW → AI_ANALYZED → TRIAGED → IN_PROGRESS → WAITING_USER → RESOLVED → CLOSED`; AI may only advance a ticket to `AI_ANALYZED`

**PII masking**
- [ ] Before any text reaches the LLM or embedding service, mask email → `[EMAIL_n]`, phone → `[PHONE_n]`, student/staff ID patterns → `[PERSON_ID_n]`, IPv4/IPv6 → `[IP_n]`, secrets/tokens/passwords → `[SECRET_n]`
- [ ] Original and masked text stored separately; viewing original text is access-limited and audit-logged
- [ ] Secrets are never stored in plain text once detected

**AI ticket analysis**
- [ ] AI returns structured JSON per the fixed schema (summary_th, category, subcategory, affected_service, location, started_at, impact_scope, priority_suggestion, team_suggestion, missing_information, keywords, confidence, rationale_th) and backend validates it against schema before saving
- [ ] Unknown values are `null`/`UNKNOWN`, never guessed; `confidence < 0.70` shows a "Needs manual triage" badge
- [ ] Officer can edit/override every AI-suggested field; overrides require a reason and are recorded
- [ ] Model name, prompt version, latency, and call time are stored per analysis
- [ ] If AI fails (timeout, rate-limit, invalid JSON, service unavailable): ticket is still created, stays in `NEW`, shows "AI analysis unavailable", and routes to a manual triage queue; retried with exponential backoff before falling back

**Rule engine (priority & routing)**
- [ ] Rule engine runs after AI analysis and suggests Priority (`P1`–`P4`) and Team based on configurable category→team mapping and impact_scope/incident-status rules
- [ ] Rules are versioned, changes are audit-logged, and rule engine can never close a ticket or confirm an incident
- [ ] Admin can edit category→team mapping and priority rules from an Admin UI

**Similarity search & incident detection**
- [ ] System embeds masked `summary_th + category + affected_service + location` and finds similar tickets (default lookback: last 24 hours), showing Top 5 with similarity score, time, category, location, status
- [ ] Officer can mark a similar ticket as `Related`, `Duplicate`, or `Not related`; marking `Duplicate` never auto-closes the ticket
- [ ] Ranking formula: `final_score = semantic_similarity*0.65 + category_match*0.15 + location_match*0.10 + recency_score*0.10`; duplicate threshold configurable, default `0.82`
- [ ] System raises a `SUSPECTED` incident alert when: ≥5 tickets, from ≥3 distinct reporters, within a 15-minute sliding window, matching category or affected service, average similarity ≥0.82, and not already linked to another active incident
- [ ] Alert shows ticket/reporter counts, detection window, category/service/location, evidence tickets, confidence/reasoning, and `Confirm incident` / `Dismiss` / `Snooze 15 min` actions
- [ ] System never labels an alert a confirmed outage until an Admin confirms it

**Human triage & incident management**
- [ ] Officer can accept/override category, priority, team; assign to team/user; request more info; link ticket to incident; view AI, rule, and similarity results separately
- [ ] Admin can create an incident from an alert or manually, add/remove linked tickets, set severity/owner/start time/affected service, log timeline and internal notes, change incident status (`SUSPECTED → CONFIRMED → MONITORING → RESOLVED → CLOSED`, or `→ DISMISSED`), and record root cause/resolution
- [ ] Confirming an incident records confirmer name, timestamp, and evidence tickets
- [ ] Only an authorized human (Officer for `RESOLVED`, Admin for `CLOSED`) can close a ticket or incident

**AI drafts**
- [ ] AI can draft: request-for-info message, acknowledgement, investigating notice, incident announcement (fixed structure: service, symptoms, affected group/area, estimated start time, current status, temporary guidance, last updated), and service-restored message
- [ ] Every draft is labeled "AI-generated draft", editable before use, and MVP only supports "Copy to clipboard" — nothing is sent or published automatically

**Dashboard, audit, search & export**
- [ ] Dashboard shows today's tickets by status, tickets by category/priority, tickets awaiting manual triage, active suspected/confirmed incidents, potential-duplicate count, average time-to-triage, and AI acceptance/override rate, filterable by date range/category/priority/team/status
- [ ] Audit log is append-only at the application level, records login/sensitive-data access, ticket CRUD/assign/status changes, AI analysis + prompt version, AI-suggestion accept/override, incident confirm/dismiss/status change, announcement approval, and rule/threshold config changes; timestamps stored UTC, displayed in `Asia/Bangkok`
- [ ] Search by ticket ID/title/description (scoped to caller's access); filter by category/priority/team/status/date; export filtered tickets/incidents to CSV excluding PII by default

### Out of Scope

- Real integration with Helpdesk, Email, LINE, Facebook, or phone systems — MVP simulates intake via web form only
- Real UP Account/SSO integration — MVP uses demo accounts per role
- Reading real Syslog/SNMP/SIEM/monitoring feeds — no live infrastructure signal ingestion in MVP
- Automated remediation: restarting servers, blocking IPs, resetting passwords, changing configuration — these are human-only actions per the AI guardrails
- Automatic publishing of announcements via Email/SMS/LINE — drafts are copy-only in MVP
- Automatic ticket closing by AI — closing is always a human, audited action
- Automated root-cause analysis that confirms causation instead of a human — AI may suggest, never confirm
- Mobile native application — responsive web only
- Voice transcription — text-only intake
- OCR from screenshots — attachments can be stored but MVP analyzes text only
- Full SLA management — MVP tracks time-to-triage as a metric only, not a formal SLA

## Context

- Product for CITCOMS (ศูนย์บริการเทคโนโลยีสารสนเทศและการสื่อสาร), University of Phayao — hackathon MVP, not a production SLA commitment
- Full spec lives at `docs/SPEC.md` (Thai/English), version 1.0.0, status "Draft for implementation"
- Target quality bars for the trial dataset (not formal SLAs): ≥85% category classification accuracy, ≥80% of known-duplicate tickets appear in Top 5 similar results, ≥95% of email/phone PII masked, 0% AI-initiated ticket closures or auto-published announcements
- 10 seeded service categories: `ACCOUNT`, `EMAIL`, `NETWORK_WIFI`, `LMS`, `INFO_SYSTEM`, `SERVER_VM`, `FIREWALL_ACCESS`, `SOFTWARE`, `AV_CLASSROOM`, `OTHER` — editable from Admin UI; team names must be configurable, not hardcoded in source
- Seed data requirement: ≥60 sample tickets across categories (incomplete-info tickets, Thai/English mixed, near-duplicates worded differently, PII-bearing samples, an incident-cluster set, deliberately ambiguous samples) — no real names/phones/emails/student IDs/IPs
- Known open questions surfaced during spec review (see `docs/SPEC.md` review notes in conversation history): retry-count conflict between §10.3 (2 retries) and Edge Case 5 (1 retry) needs resolving during AI-analysis phase planning; rule-engine P1 auto-suggestion vs. human-only P1 approval (§10.2) needs an explicit "suggest only" contract; demo-login role binding must be fixed server-side per credential, not client-selectable, to avoid privilege escalation

## Constraints

- **Tech stack**: Next.js + TypeScript + Tailwind + shadcn/ui (frontend), FastAPI + Pydantic + SQLAlchemy/Alembic (backend), PostgreSQL + pgvector (data), optional Redis — per spec §14, chosen for Thai/English structured-output + vector search needs
- **AI provider independence**: LLM and embedding provider must be swappable via adapter + environment variables; business logic must not bind to one provider — protects the hackathon build from single-vendor lock-in
- **Security/privacy**: PII must be masked before any AI/embedding call; PII fields encrypted at rest; HTTPS in production; demo passwords hashed; no real user data anywhere in the repository
- **Human-in-the-loop**: AI is decision support only — confirming major incidents, setting `P1`, publishing announcements, closing tickets/incidents, and any production/account/config change must be human-only actions, enforced at the application layer not just by prompt instructions
- **Deployment**: must run via Docker Compose with a single command (`web`, `api`, `postgres`, optional `redis`)
- **Timezone**: all timestamps stored in UTC, displayed in `Asia/Bangkok`

## Key Decisions

| Decision | Rationale | Outcome |
|----------|-----------|---------|
| Auto-initialized from existing `docs/SPEC.md` via `/gsd:new-project --auto` | Spec was already reviewed and pushed to repo; avoids re-deriving requirements from scratch | — Pending |
| Granularity: Standard (5–8 phases) | Balances MVP scope (14 functional requirement areas) against manageable phase size | — Pending |
| Model profile: Quality (Opus for research/roadmap) | Domain has real ambiguity (Thai/English NLP, incident-detection thresholds) worth deeper research | — Pending |
| Parallel plan execution enabled | No stated timeline constraint against it; speeds up MVP delivery | — Pending |

## Evolution

This document evolves at phase transitions and milestone boundaries.

**After each phase transition** (via `/gsd-transition`):
1. Requirements invalidated? → Move to Out of Scope with reason
2. Requirements validated? → Move to Validated with phase reference
3. New requirements emerged? → Add to Active
4. Decisions to log? → Add to Key Decisions
5. "What This Is" still accurate? → Update if drifted

**After each milestone** (via `/gsd:complete-milestone`):
1. Full review of all sections
2. Core Value check — still the right priority?
3. Audit Out of Scope — reasons still valid?
4. Update Context with current state

---
*Last updated: 2026-08-18 after initialization*
