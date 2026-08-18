# Feature Research

**Domain:** AI-assisted IT service desk / incident triage & early-warning (internal university IT service centre, bilingual Thai/English, hackathon MVP)
**Researched:** 2026-08-18
**Confidence:** MEDIUM-HIGH (feature parity claims verified against vendor docs; benchmark numbers and ecosystem norms from secondary sources)

---

## Executive Framing

The market splits into three product shapes, and UP IT Pulse sits deliberately in the middle one:

1. **Deflection-first AI** (Zendesk AI agents, JSM Virtual Service Agent, Moveworks) — chatbot answers the user, target is ticket volume reduction. JSM advertises ~30% deflection. **Not our shape.**
2. **Triage-assist / copilot** (Zendesk Intelligent Triage, Freshservice Freddy Copilot, JSM agent-facing AI) — AI classifies, scores confidence, suggests, human decides. **This is our shape.**
3. **Major-incident / AIOps correlation** (ServiceNow Major Incident Workbench, incident.io, AlertOps) — clustering, promotion to major incident, status communication. **This is our differentiating half.**

The single most important market signal: **every credible vendor implements "AI suggests, human commits."** Freshservice's Similar Ticket Suggester auto-analyses but "no automatic actions occur without agent direction." Zendesk stamps predictions with High/Medium/Low confidence and lets agents overwrite the field. ServiceNow requires a human manager to click *Promote to Major Incident* on a *major incident candidate*. incident.io explicitly chose human-approved AI summaries over auto-publishing because "when AI does something unhelpful, users either ignore all AI suggestions or disable the feature entirely."

That means UP IT Pulse's core principle (`AI suggests. Rules explain. Humans decide.`) is **not a hackathon limitation to apologise for — it is the industry-standard architecture**, and the project should market it as such.

---

## Feature Landscape

### Table Stakes (Users Expect These)

Missing any of these and the product reads as a demo toy rather than a service-desk tool.

| Feature | Why Expected | Complexity | Notes |
|---------|--------------|------------|-------|
| Structured intake form with field validation + immediate ticket number | Every helpdesk on earth gives the reporter a reference number. Without it the reporter has no handle on their own issue. | LOW | Spec FR-002. `UPIT-YYYY-NNNNN`. Show it on a success page; also needed for the demo narrative. |
| Idempotent submit (`Idempotency-Key`) | Reporters double-click. Duplicate tickets from one click pollute the incident detector's unique-reporter count. | LOW | Direct dependency of incident detection quality, not just hygiene. |
| Reporter "my tickets" view with status + reopen | Reporters expect to check progress without emailing. Reopen matters because Edge Case 20 (closed but still broken) is the classic trust-killer. | LOW | Reporter must not see others' tickets — IDOR is the #1 helpdesk auth bug. |
| Server-side role enforcement on every endpoint (REPORTER/OFFICER/ADMIN) | Standard in all ITSM. Also the mechanism that makes "human-only actions" real rather than prompt-based. | MEDIUM | **Critical:** demo-login role must be bound server-side per credential, never chosen by the client. Already flagged as an open question in PROJECT.md. |
| Officer queue: table view, search, filter, priority/category/status badges | This is the officer's home screen. It is the product for 80% of a shift. | MEDIUM | Add badge for AI confidence and a "Needs manual triage" highlight — direct Zendesk parity (they flag low-confidence tickets for manual review). |
| Enforced status workflow / state machine | ITSM baseline. Also where the AI guardrail lives: the transition table simply has no edge allowing AI beyond `AI_ANALYZED`. | LOW | Cheap to build, disproportionately high credibility. Unit-test it. |
| AI classification → category, subcategory, affected service, location, impact scope | The defining feature of the category. Zendesk (Topic/Intent), Freshservice (Freddy), JSM all do this. | HIGH | Must be schema-validated server-side before persisting. Pydantic model = the contract. |
| Schema-validated structured JSON output with `null`/`UNKNOWN` for unknowns | Unvalidated LLM output is a data-integrity bug generator. Guessing a building name is worse than admitting ignorance. | MEDIUM | Validate → retry → manual queue. Never persist a partially-parsed analysis. |
| Confidence signal + low-confidence "needs manual triage" flag | Zendesk ships exactly this (High/Med/Low confidence per predicted field, route low-confidence to humans). Users now expect AI to say when it's unsure. | LOW | **See PITFALL note below — the 0.70 numeric threshold needs calibration, and displaying coarse bands may be safer than a raw number.** |
| Officer override of *every* AI-suggested field, with reason captured | Zendesk: "agents can update the field values if necessary." An AI panel you cannot correct is an AI panel officers stop reading. | LOW-MEDIUM | Reason capture is what turns overrides into the evaluation dataset. |
| Similar / related tickets panel on ticket detail | Freshservice Similar Ticket Suggester, JSM "identifying similar issues", ServiceNow. Now a baseline expectation, not a wow feature. | HIGH | Requires embeddings + pgvector. The *panel* is table stakes; the *early-warning clustering built on it* is the differentiator. |
| Assign to team / user, request more info, add internal notes | Basic queue mechanics. Triage without assignment is not triage. | LOW-MEDIUM | Team names must come from a `teams` table, not an enum in code (spec §6). |
| AI draft reply for officer to edit and send | Universal: Zendesk suggested replies, Freshservice, JSM response suggestions, "copilot" everywhere. | MEDIUM | MVP: Copy-to-clipboard only. This is a legitimate MVP boundary, not a gap. |
| "AI-generated draft" labelling on all generated text | Now both a norm and increasingly a legal expectation. EU AI Act Art.50 transparency rules take effect Aug 2026 (exempted where human editorial review occurs — which is exactly our flow, but labelling is cheap insurance and good UX). | LOW | Also required for the demo: judges must see AI is not speaking as CITCOMS. |
| PII masking before any external AI/embedding call | For a Thai public university this is **compliance, not polish.** Thailand PDPA restricts cross-border transfer to countries with "adequate" protection, and the PDPC has not published a whitelist. Sending raw student IDs/emails to a foreign LLM endpoint is the single largest legal risk in the build. | MEDIUM-HIGH | Data-minimisation is explicitly named in Thai AI-compliance guidance. Store original and masked separately; audit-log access to originals. |
| Graceful AI-failure path: ticket survives, lands in manual triage queue | Availability requirement §17. A helpdesk that refuses tickets when a third-party API is down is unusable. | MEDIUM | Demo Scenario D depends on it. Retry with backoff, then fall through — resolve the 2-retry (§10.3) vs 1-retry (Edge Case 5) conflict before building. |
| Dashboard with counts by status / category / priority | Managers expect one screen. | MEDIUM | Recharts is sufficient; avoid building a BI tool. |
| Append-only audit log of sensitive actions | ITSM baseline + PDPA "detailed access logs" expectation + the project's own stated Core Value. | MEDIUM | Application-level append-only is honest for an MVP; say so explicitly rather than claiming immutability. |
| Search + filter + CSV export excluding PII by default | Every service desk exports. PII-excluded-by-default is the correct default under PDPA. | LOW | Cheap. Do it. |

### Differentiators (Competitive Advantage)

Where UP IT Pulse can beat a generic Freshservice deployment.

| Feature | Value Proposition | Complexity | Notes |
|---------|-------------------|------------|-------|
| **Cross-reporter early-warning cluster alert (SUSPECTED incident)** | The headline. Commercial tools mostly show *"similar unresolved tickets in the last 7 days"* (Freshservice) or require a human to nominate a *major incident candidate* (ServiceNow). A **15-minute sliding window + ≥5 tickets + ≥3 distinct reporters + similarity ≥0.82** detector is a genuinely sharper early-warning primitive, and it is exactly what a campus IT centre needs at 09:00 on an exam day. | HIGH | The ≥3-distinct-reporters condition is the clever part: it is what separates "one frustrated student spamming" from "the AP in ICT building died." Edge Case 8 already handles the spam case correctly. |
| **Evidence-first alert card (counts, window, evidence tickets, reasoning) with Confirm / Dismiss / Snooze 15m** | Most AIOps correlation is a black box, which produces alert fatigue and disabled features. Showing the evidence tickets makes the alert auditable and dismissible with confidence. `Snooze` is the underrated one — it acknowledges the operator saw it without forcing a premature verdict. | MEDIUM | Dismiss-with-reason feeds threshold tuning (Edge Case 19). Never render the alert as "system is down" pre-confirmation. |
| **Mask-first pipeline with "this is what the AI saw" viewer** | Turns a compliance chore into a trust feature. Officers can see the exact masked text sent to the model, side by side with the original (access-controlled + logged). Competitors treat redaction as an agent-side cleanup step (Zendesk ADPP highlights PII *after* the fact for the agent to click); mask-before-inference is architecturally stronger. | HIGH | Adapter boundary means masking is enforced in one place. Judges will find this compelling and it is defensible under PDPA. |
| **Thai/English code-switched semantic matching** | "Wi-Fi ตึก ICT ต่อได้แต่เข้าเน็ตไม่ได้" and "connected to wifi but no internet at ICT building" must cluster together. Generic keyword-based duplicate detection fails outright on Thai (no whitespace word boundaries); English-only embeddings degrade badly. | MEDIUM (buy, don't build — multilingual embedding model) | Store `embedding_model` per row: changing model = full re-index (Edge Case 14). This is the highest-risk assumption in the build — validate embedding quality on the Thai seed set **early**. |
| **Explainable hybrid ranking with visible score components** | `semantic*0.65 + category*0.15 + location*0.10 + recency*0.10`, with the components shown. Research on cloud ticket aggregation (iPACK, Azure production data) found pure text similarity is insufficient because users describe the same failure very differently; fusing extra signals lifted F1 by 12–31%. The location weight also directly kills the "same symptom, different building" false-positive (Edge Case 9). | MEDIUM | Showing *why* two tickets matched is what makes officers accept the panel. Admin-tunable weights/threshold from UI. |
| **Four-way provenance display: reporter said / AI suggested / rule suggested / human confirmed** | Spec §13.3. No mainstream tool visually separates these four sources. It is the UI embodiment of the product principle, it makes disagreement visible, and it makes the audit trail legible without opening the audit log. | MEDIUM | Pure front-end work over data you already store. Highest value-per-hour feature in the whole spec. |
| **Rule engine that emits human-readable rationale, versioned + audited** | "Rules explain." Priority derived from a named, versioned rule (`many-users-high`) beats an opaque LLM priority guess for defensibility and for tuning. Hybrid AI-extraction + deterministic-rules is the right architecture and is uncommon in small products. | MEDIUM | **Resolve the contract:** §10.2 says P1 is human-only, but the `confirmed-major-critical` rule sets P1. Make the rule engine strictly *suggest-only* for P1 and require human commit. |
| **Structured incident announcement drafting (7-field template)** | ITIL practice targets **time-to-first-communication under 15 minutes** for a major incident, and best practice says publish on a fixed cadence, not only when there is news. A pre-structured, AI-drafted announcement with a `last updated` field is a direct hit on the hardest human bottleneck in a real outage. | MEDIUM | The fixed template is what makes it useful — freeform AI announcements are worse than a form. Draft in Thai first. |
| **Missing-information assistant + draft follow-up question** | Solves the spec's actual root problem ("ข้อมูลไม่ครบ"). The emerging pattern is AI generating a targeted checklist of missing fields (error message, building, time, device) rather than a generic "please provide more info." | LOW-MEDIUM | Nearly free once AI analysis returns `missing_information[]`. Very high perceived intelligence per line of code. |
| **AI acceptance / override rate on the dashboard** | Almost no small product exposes this. It converts "we used AI" into "here is whether our AI is actually any good", which is precisely what a hackathon jury and a real CITCOMS manager want to know. | LOW | **Caveat worth stating in the UI:** a *low* override rate is not automatically good — it can mean staff are rubber-stamping. Present it as a signal, not a score. |
| **Admin config UI for categories, category→team map, thresholds, window, min-tickets** | Detection thresholds *will* be wrong on first contact with real traffic. Shipping the tuning surface (not just an env var) is what makes this deployable rather than a prototype. | MEDIUM | Every change audit-logged. Also enables live threshold-tuning during the demo. |
| **Demo / scenario mode: inject cluster, simulate AI outage** | Hackathon-critical (Scenarios C and D are undemonstrable without it) and doubles as a genuine resilience test harness. | LOW | Must be gated to a non-production flag. Do not let it write real-looking incidents in prod mode. |
| **App-layer guardrail enforcement (not prompt-based)** | The constraint says enforce at the application layer. A state machine + role policy that makes AI-initiated closure *structurally impossible* is a stronger claim than "we told the model not to." | MEDIUM | Test it: assert `0%` AI-initiated closures as an integration test, mapping straight to MVP success criterion 4. |

### Anti-Features (Commonly Requested, Often Problematic)

| Feature | Why Requested | Why Problematic | Alternative |
|---------|---------------|-----------------|-------------|
| **AI auto-closes / auto-resolves tickets** | Best-looking metric in any demo. | Directly violates the Core Value and MVP success criterion (0% AI closures). Industry evidence: 70% of consumers would consider switching after one frustrating AI service experience; premature automation erodes trust faster than slow service. Edge Case 20 exists because wrongly-closed tickets are the most damaging failure mode. | Officer sets `RESOLVED`, Admin sets `CLOSED`. AI may *suggest* resolution text. Reporter can reopen. |
| **Auto-publish announcements to email/LINE/Facebook** | "Sub-15-minute comms!" | An AI-hallucinated outage announcement to the whole university is an unrecoverable reputational event. incident.io deliberately rejected auto-publishing AI summaries for weaker reasons than ours. | Draft + human edit + copy to clipboard. Publish integrations are a v2 item *after* trust is earned on a per-message-type basis. |
| **Auto-merge or auto-close tickets above the duplicate threshold** | Threshold is right there; merging is tidy. | Similarity ≥0.82 is not proof of same cause (Edge Case 10). Merging is destructive and hard to undo; if the match was wrong, a real reporter's issue silently vanishes. Freshservice keeps merge as an explicit agent action for exactly this reason. | Non-destructive `ticket_relations` rows: `POTENTIAL` / `RELATED` / `DUPLICATE` / `NOT_RELATED`, human-confirmed, ticket stays open. |
| **Chatbot / virtual agent front-end for deflection** | The loudest trend in the category (JSM claims ~30% deflection). | Different product shape entirely. Needs a knowledge base we do not have, is the biggest prompt-injection and hallucination surface, and would consume the entire MVP budget. Also directly conflicts with the "structured, complete intake" goal. | Structured form + missing-information assistant. Revisit after a CITCOMS knowledge base / RAG corpus exists (already spec §24). |
| **Automated remediation (restart service, block IP, reset password, config change)** | "Self-healing IT" sells. | Explicitly human-only per §10.2. A prompt-injected ticket that triggers an infrastructure action is a remote-code-execution-shaped vulnerability, not a feature. | Nothing in MVP. If ever built: read-only health checks first (§24), separate approval workflow, never LLM-initiated. |
| **AI-confirmed root cause** | Reads as sophisticated. | Correlation ≠ causation; an LLM asserting a cause creates a documented, auditable, wrong RCA that a human then has to retract. | AI may draft candidate hypotheses in an internal note; `incidents.root_cause` is a human-written field only. |
| **Real integrations: email/LINE/helpdesk intake, UP Account SSO, Syslog/SNMP/SIEM** | "It should read our real signals." | Each is a multi-week integration with its own auth, rate limits, and failure modes. Note the honest tradeoff: the iPACK research shows incident-side signals materially improve duplicate detection — so monitoring ingestion is genuinely valuable, just not affordable now. | Web form + demo accounts + seed data. Design the intake layer as an adapter so an email/LINE source can be added without touching triage logic. |
| **Full SLA engine (targets, clocks, pause/resume, breach escalation)** | Managers ask for it reflexively. | CITCOMS has no formal SLA policy yet (§3 explicitly says these are prototype bars, not SLAs). Building SLA machinery against a non-existent policy bakes in fiction and adds pause/resume/business-calendar complexity. | Track **time-to-triage** as a single metric. SLA/OLA config deferred to when policy exists (§24). |
| **Sentiment analysis / emotion detection** | Zendesk and JSM both ship it, so it feels like parity. | For an internal university IT desk, sentiment does not change the routing decision — impact scope and affected service do. Adds an LLM field to validate, display, and defend for near-zero triage value, and misreads Thai politeness registers. | Skip. `impact_scope` + `priority` carry the urgency signal. |
| **Numeric AI confidence treated as a probability and used to auto-route** | The 0.70 threshold looks like a clean automation lever. | **Well-documented calibration failure:** LLM verbalised confidence clusters in the 80–100% band and is systematically overconfident (e.g. 88% stated vs 79% actual). A fixed `< 0.70` gate may almost never fire, silently disabling the manual-triage safety net. Confidence is a *ranking* signal, not a probability, absent recalibration. | Keep the threshold, but (a) calibrate it against `evaluation-set.json` before trusting it, (b) consider displaying coarse High/Medium/Low bands as Zendesk does, (c) add non-confidence triggers for manual triage: `category = OTHER`, `multiple_issues = true`, many `missing_information` entries, schema-retry occurred. |
| **Auto fine-tuning / online learning from officer overrides** | "It learns!" | Feedback loops without held-out evaluation silently drift and can be poisoned. Spec §FR-009 already draws this line correctly. | Store overrides as labelled feedback; run offline prompt-version comparison against the evaluation set. Manual promotion of prompt versions only. |
| **Free-text "ask AI anything about our tickets" chat panel** | Cheap to bolt on, demos beautifully. | Bypasses the masking boundary and role scoping in one move — a broad retrieval surface over the ticket corpus is a PII-exfiltration and prompt-injection path that undoes the entire FR-003 investment. | Constrained, purpose-built AI calls only (analyse, draft-response, draft-announcement), each with a fixed schema and masked input. |
| **OCR of screenshots / voice transcription** | Users really do paste screenshots of error dialogs. | Genuinely useful and genuinely out of budget; also multiplies the PII surface (screenshots contain names, emails, IDs that a text masker never sees). | Store attachments, analyse text only. Flag in UI that attachments are not read by AI so officers know to look. Malware-quarantine before preview (Edge Case 16). |
| **Native mobile app** | Students are on phones. | Duplicates the whole client for zero new capability. | Responsive web, mobile-first on the reporter submit form specifically. |
| **Multi-tenant per faculty / department** | "Other faculties will want this." | Tenancy is an architectural commitment (row-level scoping everywhere) that is very expensive to retrofit but ruinous to build speculatively. | Single tenant. Keep `team_id` scoping clean so tenancy *could* be layered later. |

---

## Feature Dependencies

```
[Structured intake form]
    └──requires──> [Role-based auth + demo login]
    └──requires──> [Ticket ID generation + status state machine]

[PII masking]                          <── HARD GATE, must precede both AI paths
    ├──gates──> [AI ticket analysis]
    └──gates──> [Embedding generation]

[AI ticket analysis]
    ├──requires──> [JSON schema validation (Pydantic)]
    ├──requires──> [AI provider adapter + env config]
    ├──requires──> [Background job + retry/fallback path]
    ├──enables──> [Rule engine (consumes category/impact_scope)]
    ├──enables──> [Missing-information assistant]
    ├──enables──> [Confidence / needs-manual-triage flag]
    └──enables──> [AI draft response]

[Embedding generation]
    └──enables──> [Similar ticket search (pgvector + hybrid ranking)]
                       └──enables──> [Suspected incident detection]
                                          └──enables──> [Incident management + confirm/dismiss]
                                                             └──enables──> [Draft announcement]

[Human triage / override capture]
    ├──requires──> [AI ticket analysis]  (nothing to override otherwise)
    └──enables──> [AI acceptance/override rate metric]
                       └──enables──> [Evaluation set scoring / prompt comparison]

[Audit log]  ──cross-cuts──> every state-changing action
    (build the writer EARLY; retrofitting audit calls across N endpoints is the classic late-phase tax)

[Admin config UI]
    ├──enhances──> [Rule engine]           (editable category→team map, priority rules)
    └──enhances──> [Incident detection]    (window, min tickets, min reporters, threshold)

[Seed data ≥60 tickets]
    └──BLOCKS──> [Similar ticket search demo]      (nothing to be similar to)
    └──BLOCKS──> [Incident detection demo]         (needs a planted cluster)
    └──BLOCKS──> [Dashboard]                       (empty charts)

[Demo/scenario mode] ──enables──> [Scenario C early warning] + [Scenario D AI outage]

[Numeric confidence auto-gating] ──conflicts──> [Reliable manual-triage safety net]
[Auto-merge duplicates]          ──conflicts──> [Non-destructive ticket_relations model]
[Chatbot deflection]             ──conflicts──> [Structured complete intake goal]
```

### Dependency Notes

- **PII masking gates both AI paths, so it must ship before either.** The spec's implementation order (§23) puts masking at step 4, ahead of AI analysis at step 5 — correct. Resist the temptation to "add masking later"; it is not a filter you can insert once inference code exists in three places. Enforce it inside the AI adapter so no caller can bypass it.
- **Suspected-incident detection sits three layers deep** (masking → embedding → similarity → detection). This makes it the highest-schedule-risk feature despite being the headline differentiator. Mitigation: build a deterministic detector unit-testable on synthetic similarity scores, so detection logic can be validated before embedding quality is good.
- **Seed data is a hard blocker, not a nice-to-have.** Similar-search, incident detection, and dashboard are all undemonstrable on an empty database. Move the planted near-duplicate cluster and the incident cluster into seed data early, and treat `evaluation-set.json` as a deliverable, not documentation.
- **Audit log cross-cuts everything.** Build the append-only writer plus a decorator/dependency in the same phase as ticket CRUD. Adding audit calls to 18 endpoints in a late "audit phase" is where hackathon projects lose a day.
- **Human triage requires AI analysis to exist**, but the *override-with-reason* capture is what produces the acceptance/override metric and the evaluation feedback. If schedule slips, keep override capture and drop the dashboard chart, not the reverse.
- **Rule engine depends on AI analysis output fields** (`category`, `impact_scope`), so an AI failure must leave the rule engine in a defined no-suggestion state rather than crashing the pipeline.
- **Confidence auto-gating conflicts with the safety net it is supposed to provide** — see anti-features. Add non-confidence manual-triage triggers so the net does not depend on one uncalibrated number.

---

## MVP Definition

### Launch With (v1) — the demo must land all ten MVP success criteria

- [ ] **Demo auth, 3 roles, server-side role binding** — every guardrail claim rests on this
- [ ] **Ticket intake + validation + `UPIT-YYYY-NNNNN` + idempotency** — no product without it
- [ ] **Status state machine with AI capped at `AI_ANALYZED`** — cheap, and it *is* the Core Value in code
- [ ] **PII masking before all AI/embedding calls, dual storage** — PDPA-driven, gates everything downstream
- [ ] **AI analysis returning schema-validated JSON, with model/prompt-version/latency recorded** — criteria 3 & 4
- [ ] **AI-failure fallback into a manual triage queue** — Scenario D, and basic credibility
- [ ] **Rule engine, suggest-only, with human-readable rationale + version** — "rules explain"
- [ ] **Officer queue + ticket detail with four-way provenance panels** — the officer's whole job
- [ ] **Override every AI field, reason required, recorded** — criterion 7
- [ ] **Similar tickets Top 5 with visible score components + Related/Duplicate/Not-related marking** — criterion 5
- [ ] **Suspected incident detection (5 tickets / 3 reporters / 15 min / ≥0.82) + evidence alert card** — criterion 6, the headline
- [ ] **Incident create-from-alert, link/unlink tickets, human Confirm/Dismiss with reason** — criteria 6 & 8
- [ ] **AI draft response + draft announcement (7-field template), labelled, copy-only** — criterion 9
- [ ] **Dashboard: status/category/priority counts, manual-triage queue size, active incidents, time-to-triage, AI acceptance rate** — criterion 10
- [ ] **Append-only audit log + admin viewer** — criterion 10 and the Core Value's second half
- [ ] **Search, filter, CSV export without PII** — cheap, expected
- [ ] **Seed data ≥60 tickets incl. planted near-duplicates + incident cluster + PII samples** — blocks three other features
- [ ] **Demo/scenario mode: inject cluster, simulate AI outage** — Scenarios C and D
- [ ] **Admin config: categories, category→team map, thresholds, window** — thresholds will be wrong on day one

### Add After Validation (v1.x)

- [ ] **Confidence recalibration + coarse High/Med/Low bands** — trigger: evaluation set shows the 0.70 gate firing on <5% or >40% of tickets
- [ ] **Non-confidence manual-triage triggers** (`OTHER`, `multiple_issues`, sparse extraction, schema retry) — trigger: any low-confidence tickets slipping through untriaged
- [ ] **Split-ticket action for `multiple_issues = true`** (Edge Case 2) — trigger: officers hitting it more than a few times
- [ ] **Reporter reopen window + "still happening / fixed" confirmation** (Edge Case 20) — trigger: first wrongly-closed complaint
- [ ] **Prompt-version A/B comparison against the evaluation set** — trigger: second prompt revision
- [ ] **Threshold tuning driven by dismiss-with-reason data** (Edge Case 19) — trigger: >20% of alerts dismissed
- [ ] **Attachment malware scanning + quarantine before preview** — trigger: before any non-demo user touches it
- [ ] **Embedding re-index tooling** (Edge Case 14) — trigger: first embedding-model change
- [ ] **Incident timeline / update cadence reminders** (ITIL fixed-cadence practice) — trigger: first real multi-hour incident

### Future Consideration (v2+)

- [ ] **UP Account SSO (OIDC/SAML)** — defer: demo accounts prove the role model; SSO is an infra dependency, not a product idea
- [ ] **Email / LINE / existing-helpdesk intake adapters** — defer: each is a separate integration project; keep intake pluggable so the triage core is reused unchanged
- [ ] **Monitoring / Syslog / SIEM ingestion** — defer, but flag as the *highest-value* future item: research shows fusing incident-side signals with ticket text lifts duplicate-detection F1 by 12–31%, and it would upgrade "early warning" from reactive to genuinely predictive
- [ ] **Public status page + approved publishing workflow** — defer: needs an approval workflow and a comms policy owner, not just a publish button
- [ ] **Knowledge base / RAG + resolution suggestions from past incidents** — defer: needs a curated CITCOMS corpus first; without it, hallucination risk is unmanaged
- [ ] **OCR of screenshots** — defer: real user value, but expands the PII surface beyond what the text masker covers
- [ ] **Read-only automated service health checks** — defer: the only safe form of "automated remediation"; needs its own guardrail design
- [ ] **Formal SLA/OLA configuration** — defer until CITCOMS has a written policy to encode
- [ ] **Multi-tenant per faculty** — defer: architectural commitment, ruinous to build speculatively

---

## Feature Prioritization Matrix

| Feature | User Value | Implementation Cost | Priority |
|---------|------------|---------------------|----------|
| Role-based auth + server-side role binding | HIGH | LOW | P1 |
| Ticket intake + ID + idempotency | HIGH | LOW | P1 |
| Status state machine (AI capped at `AI_ANALYZED`) | HIGH | LOW | P1 |
| PII masking before AI/embedding | HIGH | MEDIUM | P1 |
| AI analysis + schema validation | HIGH | HIGH | P1 |
| AI-failure fallback / manual triage queue | HIGH | MEDIUM | P1 |
| Officer queue + ticket detail | HIGH | MEDIUM | P1 |
| Four-way provenance display | HIGH | LOW-MEDIUM | P1 |
| Override any AI field with reason | HIGH | LOW | P1 |
| Rule engine (suggest-only) + rationale | MEDIUM-HIGH | MEDIUM | P1 |
| Similar tickets Top 5 + hybrid score display | HIGH | HIGH | P1 |
| Suspected incident detection + evidence card | HIGH | HIGH | P1 |
| Incident management + human confirm/dismiss | HIGH | MEDIUM | P1 |
| Draft response + structured announcement (copy-only) | HIGH | MEDIUM | P1 |
| Audit log (append-only) + viewer | HIGH | MEDIUM | P1 |
| Dashboard core counts | MEDIUM-HIGH | MEDIUM | P1 |
| Seed data + evaluation set | HIGH (enabling) | MEDIUM | P1 |
| Demo/scenario mode | HIGH (demo) | LOW | P1 |
| Missing-information assistant | MEDIUM-HIGH | LOW | P1 |
| Admin config (thresholds, category→team) | MEDIUM-HIGH | MEDIUM | P1 |
| AI acceptance/override rate metric | MEDIUM | LOW | P1 |
| Search + filter + PII-free CSV export | MEDIUM | LOW | P1 |
| Reporter "my tickets" view | MEDIUM | LOW | P1 |
| Confidence recalibration / coarse bands | MEDIUM | LOW-MEDIUM | P2 |
| Non-confidence manual-triage triggers | MEDIUM-HIGH | LOW | P2 |
| Split ticket on `multiple_issues` | MEDIUM | MEDIUM | P2 |
| Reporter reopen window | MEDIUM | LOW | P2 |
| Attachment malware scan / quarantine | MEDIUM (HIGH in prod) | MEDIUM | P2 |
| Prompt-version A/B evaluation | MEDIUM | MEDIUM | P2 |
| Embedding re-index tooling | LOW (until needed) | LOW | P2 |
| SSO / real intake channels / monitoring ingestion | HIGH (long-run) | HIGH | P3 |
| Status page publishing workflow | MEDIUM | HIGH | P3 |
| Knowledge base / RAG resolution suggestions | HIGH (long-run) | HIGH | P3 |
| OCR / voice | LOW-MEDIUM | HIGH | P3 |
| SLA engine, multi-tenant, chatbot deflection | LOW (now) | HIGH | P3 |

**Priority key:** P1 = must have for launch · P2 = should have, add when possible · P3 = future consideration

---

## Competitor Feature Analysis

| Feature | ServiceNow ITSM | Freshservice (Freddy AI) | Zendesk Intelligent Triage | Jira Service Management | incident.io | **UP IT Pulse approach** |
|---------|-----------------|--------------------------|----------------------------|-------------------------|-------------|--------------------------|
| Auto classification | Yes, ML-based | Yes, Freddy categorisation | Yes — Topic/Intent, Sentiment, Language, Entities stamped on ticket creation | Yes, AI categorisation | N/A (incident-centric) | LLM structured extraction, 13-field schema incl. `missing_information` and `rationale_th` |
| Confidence exposure | Limited | Limited | **High/Medium/Low band per predicted field**; low-confidence can be routed to manual review | Limited | N/A | Numeric `confidence` + `Needs manual triage` badge at <0.70 — **calibrate, consider bands** |
| Agent override | Yes | Yes | Yes, agents edit predicted fields; `triage_override` tag to exempt tickets | Yes | Suggestions accepted/rejected | Yes, **every field, reason required, recorded as feedback** |
| Similar tickets | Yes | **Similar Ticket Suggester** — unresolved (7-day window) + resolved buckets; no similarity score shown; agent chooses parent/child, problem link, reassign, merge | Related-ticket surfacing | "Identifying similar issues" | N/A | **Top 5 with visible hybrid score breakdown**, 24h default lookback, non-destructive relation marking |
| Duplicate handling | Merge/parent-child | Agent-initiated merge; no auto-action | Agent-initiated | Agent-initiated | N/A | Non-destructive `ticket_relations`; `DUPLICATE` never auto-closes |
| Major-incident detection | **Major incident candidate** → human manager clicks *Promote to Major Incident* or *Reject*; Major Incident Workbench single-pane view | Flags clustering incidents, suggests creating a problem record and links incidents | Not a focus | Not a focus | Human-declared incidents + AI assistance | **Automatic SUSPECTED alert from a 15-min sliding window, ≥5 tickets, ≥3 distinct reporters, ≥0.82 avg similarity** — sharper trigger than any of these, still human-confirmed |
| Human confirmation of major incident | Required (promote/reject) | Human creates problem record | N/A | Human | Human declares | **Admin-only Confirm/Dismiss/Snooze, with confirmer + timestamp + evidence tickets recorded** |
| AI draft reply | Yes | Yes | Suggested replies; drafts as internal notes agents edit | Response suggestions | AI incident summaries | Yes, labelled draft, **copy-only in MVP** |
| Comms / announcements | Communication tooling opens on promotion | Announcements | Not a focus | Not a focus | **Status pages, auto customer updates — but AI summaries deliberately require human accept/reject** | 7-field structured Thai announcement draft, human edit, copy-only |
| PII redaction | DLP add-ons | Add-ons | **ADPP add-on**: trigger-based auto-redaction, AI redaction *suggestions* highlighted for agents, role-based masking, access logs | Add-ons / marketplace | N/A | **Mask before inference** (not post-hoc agent cleanup) + "what the AI saw" viewer + audit-logged original access |
| Explainability of routing | Rules + ML, limited surfacing | Limited | Limited | Limited | N/A | **Versioned named rules with human-readable rationale, separated from AI suggestion in UI** |
| AI quality metrics | Reporting suite | Reporting | Zendesk AI metrics/attributes | Virtual-agent dashboard with knowledge gaps | N/A | **Acceptance/override rate on the main dashboard** + offline evaluation-set scoring |
| Bilingual Thai/English | Localisation, generic ML | Generic | Language detection | Generic | English-centric | **Thai-primary UI, Thai/English code-switched semantic matching, Thai summaries and drafts** |
| Deflection chatbot | Virtual Agent | Freddy Self-Service | AI agents (vendor claims high autonomous resolution) | Virtual Service Agent, ~30% deflection | N/A | **Deliberately excluded** — different product shape, no KB, largest injection surface |

**Where UP IT Pulse genuinely wins:** (1) a sharper, multi-reporter, short-window early-warning trigger than the 7-day "similar unresolved" lists or manual candidate nomination the incumbents offer; (2) mask-before-inference rather than post-hoc agent redaction, which is the right architecture under Thailand PDPA cross-border constraints; (3) explicit four-way provenance so officers can see reporter vs AI vs rule vs human disagreement; (4) Thai-first bilingual semantic matching. **Where it cannot compete and should not try:** integration breadth, knowledge-base-grounded deflection, SLA machinery, scale.

---

## Confidence Assessment

| Claim area | Confidence | Basis |
|------------|------------|-------|
| "Human-in-the-loop is the industry-standard architecture, not a limitation" | **HIGH** | Verified in vendor documentation for ServiceNow (major incident candidate → promote/reject), Freshservice (no automatic actions without agent direction), Zendesk (agents update predicted fields), plus incident.io's published design rationale |
| Zendesk confidence bands + low-confidence manual review | **HIGH** | Zendesk help documentation |
| Freshservice similar-ticket behaviour (7-day unresolved window, agent-only actions, no score shown) | **HIGH** | Freshservice support documentation, fetched directly |
| ServiceNow major incident candidate / workbench flow | **MEDIUM-HIGH** | ServiceNow docs page + community threads; workbench UI element details from official docs |
| LLM verbalised confidence is systematically overconfident and clusters 80–100% | **MEDIUM-HIGH** | Multiple recent arXiv/journal calibration studies agree; exact figures vary by task, and structured multiple-choice tasks calibrate better than open extraction — so our schema task may sit between the extremes. **Must be validated on the project's own evaluation set.** |
| Pure text similarity underperforms for duplicate ticket aggregation; multi-signal fusion helps materially | **MEDIUM-HIGH** | iPACK paper, Azure production data, F1 0.871–0.935 and +12.4–31.2% over baselines. Cloud-provider domain, not university helpdesk — directionally transferable, magnitude not |
| Thailand PDPA cross-border constraint justifies masking as compliance | **MEDIUM** | Multiple legal-advisory sources agree the PDPC has published no adequacy whitelist and that data minimisation applies to AI inputs. **Not legal advice — CITCOMS should confirm with the university DPO before any production deployment.** |
| ITIL time-to-first-communication <15 min for major incidents | **MEDIUM** | Practitioner and vendor ITSM sources, not the ITIL 4 standard text itself. Treat as best-practice guidance, not a citation |
| EU AI Act Art.50 labelling relevance | **MEDIUM** | Official EU sources on Art.50 and Aug 2026 timing; **Thailand is not in scope of the EU AI Act**, so this is a norm/insurance argument, not a compliance requirement here |
| "Low override rate can mean rubber-stamping" | **MEDIUM** | Single credible industry-analysis source; conceptually sound (automation bias is well documented) but not independently verified |
| Deflection rates (~30% JSM) and accuracy benchmarks (85%+ routing, 90%+ tagging) | **LOW-MEDIUM** | Vendor marketing and secondary blog aggregation. Useful as a sanity check that the spec's 85% classification target is in a normal range; **do not cite as authority** |

---

## Gaps / Open Questions for Requirements Definition

1. **Confidence threshold calibration.** No source supports 0.70 as a meaningful cut point for an LLM's self-reported confidence on Thai extraction. Requirements should mandate calibrating it against `evaluation-set.json` and add non-confidence manual-triage triggers as a backstop.
2. **Retry-count contradiction.** §10.3 says up to 2 retries; Edge Case 5 says retry once. Must be resolved as a single explicit requirement (already flagged in PROJECT.md).
3. **P1 authority contradiction.** §10.2 makes P1 human-only, while the `confirmed-major-critical` rule sets P1. Requirements must state the rule engine is suggest-only and that P1 requires a human commit action.
4. **No source found for the specific 5-tickets / 3-reporters / 15-minute parameter set.** These appear to be spec-authored heuristics rather than industry-derived. That is defensible for an MVP, but requirements should mark them as tunable-by-design and demand the admin UI, not just env vars. A campus of ~20k users may need very different numbers per category (Wi-Fi noise vs SERVER_VM silence).
5. **Thai embedding quality is unvalidated.** No benchmark found for multilingual embeddings on Thai/English code-switched short IT-support text specifically. This is the highest technical risk in the differentiating feature and should be a spike with a pass/fail gate (Recall@5 ≥0.80 on planted duplicates) before the incident detector is built on top of it.
6. **Per-category thresholds not addressed anywhere.** One global 0.82 threshold across ACCOUNT and AV_CLASSROOM is likely wrong. Worth raising as a possible v1.x requirement.
7. **Announcement approval trail.** FR-013 audit-logs "draft announcement approval", but no requirement defines what "approval" means when the only action is copy-to-clipboard. Needs an explicit approval action to log.

---

## Sources

**Vendor / official documentation (highest weight)**
- Freshservice — Identify similar tickets using Similar Ticket Suggester: https://support.freshservice.com/support/solutions/articles/50000009528-identify-similar-tickets-using-similar-ticket-suggester
- Freshservice — Freddy AI Copilot overview: https://support.freshservice.com/support/solutions/articles/50000009429-freddy-ai-copilot-overview
- Zendesk — Automatically classifying tickets with intelligent triage: https://support.zendesk.com/hc/en-us/articles/4550640560538-Automatically-classifying-customer-intent-sentiment-and-language
- Zendesk — Viewing and managing intelligent triage predictions: https://support.zendesk.com/hc/en-us/articles/6298065502874-Viewing-and-managing-intelligent-triage-predictions
- Zendesk — Making sense of unexpected intelligent triage predictions: https://support.zendesk.com/hc/en-us/articles/5608698604698-Making-sense-of-unexpected-intelligent-triage-predictions
- Zendesk — Redacting identified PII (ADPP add-on): https://support.zendesk.com/hc/en-us/articles/10474374743450-Redacting-identified-PII-ADPP-add-on
- Zendesk — Automatically redacting sensitive information using triggers: https://support.zendesk.com/hc/en-us/articles/9248330321050-Automatically-redacting-sensitive-information-in-tickets-using-triggers
- ServiceNow — Major incident workbench UI elements: https://www.servicenow.com/docs/r/washingtondc/it-service-management/incident-management/mi-workbench-ui-elements.html
- ServiceNow Community — Promote to major incident / major incident candidate: https://www.servicenow.com/community/developer-forum/major-incident-candidate/m-p/2819438
- Atlassian — Jira Service Management AI feature guide: https://www.atlassian.com/software/jira/service-management/product-guide/tips-and-tricks/artificial-intelligence
- Atlassian — Agentic AI: the next chapter for Jira Service Management: https://www.atlassian.com/blog/announcements/jira-service-management-agentic-ai
- incident.io — Statuspage integration docs: https://docs.incident.io/integrations/statuspage
- EU — Article 50 transparency obligations: https://artificialintelligenceact.eu/article/50/
- EU — Quick facts, transparency rules for AI systems: https://digital-strategy.ec.europa.eu/en/factpages/quick-facts-transparency-rules-ai-systems

**Research literature**
- Incident-aware Duplicate Ticket Aggregation for Cloud Systems (iPACK), arXiv 2302.09520: https://arxiv.org/abs/2302.09520
- Calibration of Self-Reported Confidence and Accuracy of LLMs: https://link.springer.com/article/10.1007/s10916-026-02430-0
- Confidence Calibration in Large Language Models, arXiv 2605.23909: https://arxiv.org/html/2605.23909v1
- Uncertainty Decomposition for Clarification Seeking in LLM Agents, arXiv 2606.19559: https://arxiv.org/pdf/2606.19559
- WangchanBERTa: Pre-trained Thai Language Model: https://airesearch.in.th/releases/wangchanberta-pre-trained-thai-language-model/

**Legal / compliance (advisory, not authoritative)**
- Securiti — Thailand cross-border personal data transfer overview: https://securiti.ai/thailand-cross-border-personal-data-transfer-overview/
- Formichella & Sritawat — Cross-border customer data under Thailand's PDPA: https://fosrlaw.com/2026/thailand-pdpa-cross-border-data-transfers/
- Pertama Partners — Thailand AI regulations compliance guide: https://www.pertamapartners.com/insights/thailand-ai-regulations-2026

**Industry analysis (lowest weight, used for norms and cautions)**
- incident.io AI incident summary generator case study (human-approval design rationale): https://www.zenml.io/llmops-database/building-and-deploying-an-ai-powered-incident-summary-generator
- CMSWire — decision-quality metrics vs acceptance rate / rubber-stamping risk: https://www.cmswire.com/customer-experience/what-decision-quality-metrics-reveal-about-autonomous-agents/
- IrisAgent — AI ticket automation triage metrics and benchmarks: https://irisagent.com/ai-ticket-automation/
- Kustomer — why help desk is the wrong starting point for AI customer service: https://www.kustomer.com/resources/blog/why-help-desk-wrong-starting-point-ai-customer-service/
- TechRadar Pro — the trust recession: why customers don't trust AI: https://www.techradar.com/pro/the-trust-recession-why-customers-dont-trust-ai-and-how-to-fix-it
- eesel AI — AI response suggestions / helpdesk copilot patterns (copilot-first, autopilot-later): https://www.eesel.ai/blog/ai-response-suggestions
- InvGate — Major incident management process, roles and runbook: https://blog.invgate.com/major-incident-management
- ManageEngine — ITIL major incident management process and roles: https://www.manageengine.com/products/service-desk/it-incident-management/major-incident-management.html
- Inference Systems — setting up intelligent alert correlation and noise reduction: https://inferensys.com/guides/ai-first-it-operations-aiops-and-self-healing-it/setting-up-intelligent-alert-correlation-and-noise-reduction
- People Use AI — identifying missing information and next-best questions: https://peopleuse.ai/use-cases/customer-support/identifying-missing-information-and-next-best-questions

**Project documents**
- `/home/pdk/spec_workshop/.planning/PROJECT.md`
- `/home/pdk/spec_workshop/docs/SPEC.md` (v1.0.0, §3 MVP Success Criteria, §4 Scope, §9 Functional Requirements, §10 AI Guardrails, §18 Edge Cases, §24 Future Enhancements)

---
*Feature research for: AI-assisted IT incident triage & early warning (UP IT Pulse / CITCOMS, University of Phayao)*
*Researched: 2026-08-18*
