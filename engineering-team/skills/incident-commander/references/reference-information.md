# Incident Commander — Reference Information

Supplementary reference material for the Incident Commander skill: severity/priority matrices, escalation criteria, communication cadences, and postmortem structure. Use this alongside `incident_classifier.py`, `timeline_reconstructor.py`, and `pir_generator.py`.

## 1. Severity & Priority Matrix

Severity should be derived from **impact** (how bad) crossed with **urgency** (how fast it's getting worse or how time-sensitive the exposure is), not from gut feel alone.

| Impact ↓ / Urgency → | Immediate (spreading, revenue-critical) | High (stable but severe) | Low (isolated, contained) |
|---|---|---|---|
| **Total outage / data loss / security exposure** | SEV1 | SEV1 | SEV2 |
| **Major degradation (>25% users, core feature down)** | SEV1 | SEV2 | SEV2 |
| **Minor degradation (<25% users, workaround exists)** | SEV2 | SEV3 | SEV3 |
| **Cosmetic / no user impact** | SEV3 | SEV4 | SEV4 |

### Quick severity triage questions
1. Is the service completely unavailable, or is data being lost/corrupted/exposed? → if yes, **SEV1**.
2. Is more than ~25% of users or a revenue-critical path affected? → if yes, **SEV2 or higher**.
3. Is there a workaround and is impact confined to a single feature? → **SEV3**.
4. Cosmetic, internal-only, or dev/test environment? → **SEV4**.

### Severity Definitions (canonical)

| Level | Definition | Typical Response SLA |
|---|---|---|
| SEV1 | Complete service failure or critical business function down; data loss/corruption; customer data exposure; revenue system down | IC assigned ≤5 min, exec notified ≤15 min, status page ≤15 min |
| SEV2 | Significant degradation affecting a subset of users (>25%) or non-critical but important functions | IC assigned ≤30 min, status page ≤30 min, stakeholders ≤1 hr |
| SEV3 | Limited impact, workaround available, <25% of users | Response ≤2 hrs business hours, next-business-day otherwise |
| SEV4 | Cosmetic, documentation, monitoring gaps, no meaningful user impact | 1–2 business days, standard ticketing |

## 2. Escalation Criteria

Escalate **up a severity level** (e.g., SEV3 → SEV2) when any of the following occur, regardless of the initial classification:

- Impact scope grows beyond the original estimate (e.g., a single-region issue becomes multi-region).
- A workaround that was previously available stops working.
- The incident crosses a data-integrity boundary (reads become writes-at-risk, backups implicated).
- Time-in-incident exceeds 2x the expected resolution time for that severity with no clear ETA.
- A second, independent system starts failing (cascading failure).
- Customer support ticket volume spikes sharply or a customer/partner escalates directly to an executive.
- Legal, compliance, or regulatory exposure becomes plausible (PII, financial data, contractual SLAs).

**Escalate to Incident Commander immediately** (bypass normal triage) when:
- Any SEV1 signal is present.
- A security dimension is suspected (unexpected auth failures, unusual data egress) — hand off to `incident-response` for security triage in parallel.
- An engineer is unsure whether to declare — default to declaring and downgrading later; it is cheaper than a late declare.

**Escalate to Executive/Leadership** when:
- SEV1 for any duration.
- SEV2 exceeding 1 hour without resolution or clear mitigation path.
- Any customer-facing communication or status page update is required.
- Emergency spend, vendor engagement, or cross-org resource pulls are needed.

## 3. Communication Cadence Reference

| Stakeholder | SEV1 | SEV2 | SEV3 | SEV4 |
|---|---|---|---|---|
| Engineering Leadership | Real-time | Every 30 min | Every 4 hrs | Daily digest |
| Executive Team | Every 15 min | Every 1 hr | End of day | Weekly digest |
| Customer Support | Real-time | Every 30 min | Every 2 hrs | As needed |
| Customers | Every 15 min | Every 1 hr | Optional | None |
| Partners | Every 30 min | Every 2 hrs | Optional | None |

Rule of thumb: **never let a stakeholder's cadence window lapse silently** — if there's nothing new to report, send "no change, still investigating, next update at X" rather than going quiet.

## 4. Roles & RACI Summary

| Role | Responsibility during incident |
|---|---|
| Incident Commander (IC) | Owns the response, makes final calls, controls comms cadence, does not personally debug |
| Technical Lead | Owns the technical investigation and mitigation strategy, reports findings/options to IC |
| Scribe | Logs timeline of actions/decisions in real time (feeds `timeline_reconstructor.py`) |
| Communications Lead (SEV1/2) | Drafts and sends stakeholder/customer updates, keeps status page current |
| Subject Matter Experts (SMEs) | Pulled in as needed for specific systems; report to Technical Lead |

The IC should never be the sole hands-on-keyboard responder for anything but the smallest incidents — command and execution are separate jobs.

## 5. Root Cause Analysis (RCA) Frameworks

### 5 Whys
Ask "why" iteratively (typically 4–6 times) starting from the customer-visible symptom until you reach a systemic, actionable cause (process/design gap), not a person.

Example chain:
1. Why did users see 500 errors? → The API couldn't get a database connection.
2. Why couldn't it get a connection? → The connection pool was exhausted.
3. Why was the pool exhausted? → Queries were taking far longer than normal, holding connections open.
4. Why were queries slow? → A newly deployed query was doing a full table scan.
5. Why did it do a full table scan? → The required index was missing on the new query path.
6. Why was the index missing? → Schema migration review didn't include a query-plan check for new endpoints.

### Fishbone (Ishikawa)
Group contributing factors into categories to avoid tunnel vision on a single cause:
- **People:** on-call familiarity, staffing at time of incident, training gaps
- **Process:** review/approval steps skipped or insufficient, change management gaps
- **Technology:** code defects, missing indexes/capacity limits, monitoring blind spots
- **Environment:** load/traffic conditions, dependency/vendor state, infrastructure config drift

### Timeline-based Analysis
Overlay the reconstructed timeline (from `timeline_reconstructor.py`) with:
- **Contributing events** — every change/deploy/config edit in the preceding 24–72 hours
- **Detection gap** — time between the causal event and the first alert/human notice
- **Response gap** — time between detection and effective mitigation action
This highlights whether the bottleneck was prevention, detection, or response.

## 6. Postmortem (PIR) Structure

A complete Post-Incident Review, in order:

1. **Header** — Incident ID, title, severity, status, dates/times, authors, reviewers.
2. **Executive Summary** — 2–4 sentences: what happened, who was affected, how it was resolved.
3. **Impact** — users/percentage affected, duration, revenue/SLA impact, support ticket volume.
4. **Key Metrics** — Time to Detect (TTD), Time to Acknowledge (TTA), Time to Mitigate (TTM), Time to Resolve (TTR).
5. **Timeline** — chronological, timestamped, sourced from `timeline_reconstructor.py` output.
6. **Root Cause Analysis** — using one or more of the frameworks above; distinguish *root cause* from *contributing factors* and *trigger*.
7. **What Went Well** — detection speed, effective mitigations, good communication — reinforce these.
8. **What Went Wrong / Gaps** — blameless, system-focused; process, tooling, or design gaps.
9. **Action Items** — specific, owned, dated, prioritized (P0–P3); tracked to closure, not just listed.
10. **Lessons Learned / Follow-up** — broader patterns, links to related incidents, runbook updates made.

### Action Item Quality Bar
Every action item must have: an **owner** (individual, not a team), a **due date**, a **priority**, and be **independently verifiable** as done (not "investigate further" but "add index on orders.customer_id and verify via EXPLAIN ANALYZE").

## 7. Common Pitfalls Checklist

- Declaring too late because the on-call engineer wanted to "confirm" before paging — bias toward declaring early and downgrading.
- IC also acting as sole hands-on-keyboard debugger, losing situational awareness.
- Status page updates lagging internal knowledge by more than the stakeholder cadence window.
- Fix-forward under pressure when a known-good rollback was available.
- PIR scheduled but never actually held, or held without the IC/Technical Lead present.
- Action items with no owner or due date, which silently die in a backlog.
- Treating the *trigger* (e.g., a traffic spike) as the *root cause* instead of the underlying fragility it exposed (e.g., missing index, no pool ceiling alerting).
