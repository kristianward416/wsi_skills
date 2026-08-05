# Responsible Disclosure Guide

Full disclosure workflow referenced from the main SKILL.md's "Responsible Disclosure Workflow" section and Anti-Pattern #3 ("Skipping responsible disclosure"). Applies to any vulnerability discovered during authorized testing, whether under a formal engagement, a bug bounty program, or independent research.

---

## 1. Core Principles

1. **Authorization first.** Only test systems you have written authorization to test (signed scope-of-work / rules of engagement, or a published bug bounty scope). Testing outside that authorization is unauthorized access under the CFAA (US) and equivalent computer-misuse laws elsewhere, even with good intentions.
2. **Minimize harm.** Stop at proof of concept. Do not exfiltrate real user data beyond what's needed to prove impact (a single record, redacted, is usually enough — not a full table dump). Do not pivot beyond what's needed to establish severity.
3. **Confidentiality.** Do not disclose the vulnerability publicly, to third parties, or on social media before the agreed disclosure date. Encrypt all communications containing vulnerability details (PGP for email, or the vendor's secure reporting channel).
4. **Document everything with timestamps.** Every request/response, every action taken, and every communication should be logged with a timestamp — this protects both you and the vendor if there's ever a dispute about scope or timeline.
5. **Give the vendor a fair chance to fix it before it's public.** The entire point of coordinated disclosure is balancing user safety (informing them promptly) against giving the vendor time to remediate before attackers can weaponize public details.

---

## 2. Disclosure Timelines

### Standard Timeline — 90 Days (Industry Norm)

This is the de facto industry standard, popularized by Google Project Zero and adopted broadly (ISO/IEC 29147, CERT/CC guidance, most vendor PSIRTs).

| Day | Action |
|-----|--------|
| Day 0 | Vulnerability discovered/confirmed. Report submitted to vendor via their published security contact or bug bounty platform. |
| Day 1 | Vendor acknowledgment expected. If none received, resend and escalate to a secondary contact (`security@`, `security.txt` if published). |
| Day 7 | First follow-up if no substantive response. Ask for a triage status and expected timeline. |
| Day 30 | Status check-in. Vendor should have confirmed the issue and be working on a fix, or provided a reason for delay. |
| Day 60 | Second status check-in. Confirm patch is in development/testing, or discuss timeline slippage. |
| Day 90 | Public disclosure date (unless extended by mutual agreement — see below). Publish after confirming the fix is deployed, or disclose as "unpatched" if the vendor has gone unresponsive. |

### Accelerated Timeline — 30 Days (Actively Exploited or Trivial-to-Weaponize)

Used when the vulnerability is already being exploited in the wild, is trivially discoverable by others (e.g., visible via a simple Shodan/Google dork), or poses immediate severe risk (e.g., unauthenticated RCE on internet-facing infrastructure).

| Day | Action |
|-----|--------|
| Day 0 | Report submitted, flagged explicitly as "actively exploited" or "critical/trivial to rediscover" with evidence. |
| Day 1–3 | Expect expedited vendor acknowledgment; escalate immediately if silent. |
| Day 14 | Status check — patch should be in progress or already shipped for actively-exploited issues. |
| Day 30 | Public disclosure, or earlier by mutual agreement once a patch ships and is broadly deployed. |

### Extended Timeline — 120 Days (Complex Remediation)

Used when the fix requires architectural changes, affects a widely deployed embedded/firmware component, or needs coordination across multiple downstream vendors (supply-chain issue).

| Day | Action |
|-----|--------|
| Day 0 | Report submitted, noting expected complexity. |
| Day 30 | Status check — vendor should have a remediation plan and rough timeline. |
| Day 60 | Progress check — request evidence of active work (beta patch, internal ticket reference). |
| Day 90 | Negotiate extension if genuinely needed — extensions should be justified by the vendor, not just requested to delay. |
| Day 120 | Disclosure date, or coordinated multi-party disclosure date if multiple vendors are affected (e.g., CERT/CC-style multi-vendor coordination). |

**Extension rule of thumb:** Grant extensions in good faith when there's demonstrable active remediation work, not indefinitely. If a vendor goes silent for 30+ days after initial acknowledgment, proceed toward the original disclosure date.

---

## 3. Safe Harbor Language

Safe harbor terms protect a good-faith researcher from legal action by the vendor, provided testing stayed within scope. Always confirm safe harbor terms exist **before** testing — a bug bounty program's public policy page, or an explicit clause in a pen-test SOW.

**What good safe harbor language looks like (adapted from common bug bounty policies, e.g., the disclose.io framework):**

> "We will not pursue civil action or initiate a complaint to law enforcement for accidental, good-faith violations of this policy. We consider activities conducted consistent with this policy to constitute 'authorized' conduct under the Computer Fraud and Abuse Act and similar state and federal laws, and to be exempt from restrictions in our Terms of Service that would otherwise prohibit security research. We will not bring a DMCA claim against you for circumventing technological measures we've implemented to protect the applications in scope, when circumvention was solely for the purpose of good-faith security research within the scope described above."

**Key elements a safe harbor clause should include:**
- Explicit statement that good-faith research within scope will not trigger legal action (CFAA, DMCA, state computer-crime statutes)
- Definition of "good faith" (no data exfiltration beyond PoC, no service disruption, no privacy violations, immediate reporting)
- Scope boundaries — what's in scope vs. explicitly out of scope
- Prohibition on social engineering, physical security testing, or DoS unless explicitly authorized
- A named point of contact for questions before testing begins

**If no safe harbor exists:** Do not test. Contact the organization first and ask for written permission or point to a bug bounty program before touching anything.

---

## 4. What a Good Vulnerability Report Contains

A report that gets triaged quickly and taken seriously includes, in this order:

1. **Title** — Specific and descriptive: "SQL Injection in `/api/login` via `email` parameter allows authentication bypass" not "Security issue found."
2. **Summary** — One or two sentences: what the vulnerability is and its impact.
3. **Severity/CVSS score** — Self-assessed score with the full vector string (see CVSS section below). Vendors re-score, but a well-reasoned initial score speeds triage.
4. **Affected asset(s)** — Exact URL(s), endpoint(s), parameter(s), app version, and environment (production vs. staging) where confirmed.
5. **Steps to reproduce** — Numbered, exact, reproducible by someone with no prior context. Include full request (method, URL, headers, body) and the exact payload used.
6. **Proof of concept** — Screenshot, `curl` command, or short script — the minimum needed to demonstrate impact. Redact any real user data captured incidentally.
7. **Impact** — What an attacker can actually do with this: data exposed, privilege gained, systems reachable, business consequence. Avoid both underselling and hype — state it plainly.
8. **Suggested remediation** — Concrete fix guidance (parameterized queries, specific header to add, config change), not just "fix the vulnerability."
9. **Discovery timestamp and researcher contact info** — When you found it, and how to reach you securely (PGP key if available).
10. **Disclosure timeline you're proposing** — State the standard/accelerated/extended timeline you intend to follow, so there's no ambiguity later.

**Example minimal report skeleton:**

```
Title: IDOR in /api/orders/{id} allows access to any customer's order (PII exposure)
Severity: High (CVSS 3.1: 8.1 — AV:N/AC:L/PR:L/UI:N/S:U/C:H/I:N/A:N)
Affected: https://target.com/api/orders/{id}  (production, confirmed 2026-08-01)

Steps to Reproduce:
1. Authenticate as any user, capture the bearer token.
2. Request GET /api/orders/1043 with that token (own order is 1042).
3. Response returns another customer's full order, including name, address, and phone number.

PoC:
curl -H "Authorization: Bearer <redacted>" https://target.com/api/orders/1043

Impact: Any authenticated user can enumerate and read every other customer's order
records (est. 40,000+ records based on ID range), exposing PII at scale.

Remediation: Add server-side ownership check comparing order.user_id to the
authenticated user's ID before returning data; return 403/404 on mismatch.

Discovered: 2026-08-01T14:32Z UTC
Researcher contact: [name/handle], PGP fingerprint: [xxxx]
Proposed timeline: Standard 90-day coordinated disclosure.
```

---

## 5. CVSS Scoring Basics (v3.1)

CVSS scores communicate severity in a standardized, defensible way. Compute using the official vector string format so vendors can independently verify the score.

**Base metric groups:**

- **Attack Vector (AV):** Network (N), Adjacent (A), Local (L), Physical (P) — how the attacker reaches the vulnerable component.
- **Attack Complexity (AC):** Low (L) — no special conditions; High (H) — requires specific conditions outside attacker's control.
- **Privileges Required (PR):** None (N), Low (L), High (H) — what access the attacker needs before the attack.
- **User Interaction (UI):** None (N) — no victim action needed; Required (R) — e.g., victim must click a link (relevant for XSS/CSRF).
- **Scope (S):** Unchanged (U) — impact confined to the vulnerable component; Changed (C) — impact extends to resources beyond the vulnerable component's security scope (e.g., SQLi reaching the OS).
- **Impact metrics — Confidentiality (C), Integrity (I), Availability (A):** each None (N), Low (L), High (H).

**Worked examples:**

| Finding | Vector | Score | Severity |
|---|---|---|---|
| Unauthenticated SQLi with full DB read/write | `AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:H` | 9.8 | Critical |
| IDOR exposing one user's PII, requires valid low-priv session | `AV:N/AC:L/PR:L/UI:N/S:U/C:H/I:N/A:N` | 6.5–8.1 | High |
| Reflected XSS requiring victim to click a crafted link | `AV:N/AC:L/PR:N/UI:R/S:U/C:L/I:L/A:N` | ~5.4 | Medium |
| Missing HSTS header alone (no active exploitation shown) | `AV:N/AC:H/PR:N/UI:N/S:U/C:L/I:N/A:N` | ~3.7 | Low |
| SSRF reaching cloud metadata, stealing IAM creds with broad permissions | `AV:N/AC:L/PR:N/UI:N/S:C/C:H/I:H/A:H` | 10.0 | Critical |

**Practical guidance:**
- Score the vulnerability as actually demonstrated, not the theoretical worst case across all possible deployments — but do account for realistic escalation you've proven is reachable (e.g., the SQLi-to-RCE chain in `attack_patterns.md` justifies scoring for the RCE outcome once demonstrated, not just the initial injection).
- Use **Scope: Changed** whenever impact crosses a security authority boundary — classic case: app-layer SQLi that reaches OS-level file write.
- Always publish the full vector string alongside the numeric score in a report; the number alone isn't independently verifiable.
- Severity bands (informational, standard across most vendor programs): 0.1–3.9 Low, 4.0–6.9 Medium, 7.0–8.9 High, 9.0–10.0 Critical.

---

## 6. Bug Bounty Program Integration

- **Check the published scope before testing** — in-scope assets, out-of-scope assets (often includes third-party integrations, physical offices, social engineering), and prohibited techniques (DoS, spam, automated scanning without rate limits).
- **Submit through the platform, not email**, when a program uses HackerOne, Bugcrowd, or an internal portal — this preserves the timeline record and triage SLA that platform terms guarantee.
- **Respect duplicate/N/A handling gracefully** — if marked duplicate, request the original report's disclosure timeline so you know when you're free to discuss it; don't rush independent disclosure out of frustration.
- **Do not negotiate payout by threatening early disclosure** — this is treated as extortion by every major program and can result in legal action and permanent bans, regardless of the underlying finding's validity.

---

## 7. CVE Request Process

For vulnerabilities in software/products (not one-off custom web apps), request a CVE ID to give the fix a durable public identifier once disclosure is agreed:

1. **Identify a CVE Numbering Authority (CNA)** — many vendors are their own CNA (check the vendor's security page); otherwise use MITRE's CNA list or request directly from MITRE at cveform.mitre.org.
2. **Submit required fields:** affected product/version, vulnerability type (CWE ID if known — e.g., CWE-89 for SQLi), description, and a reference (advisory URL, commit fixing it) once available.
3. **Coordinate timing** — CVE reservation can happen before public disclosure; the ID is typically embargoed until the agreed disclosure date, then the CNA publishes the full record.
4. **Cross-reference in your report** — once assigned, include the CVE ID in your public writeup and any advisory, and link back to the CWE category for classification consistency (ties back to the OWASP category mapping in `owasp_top_10_checklist.md`, since most CWEs map cleanly to an OWASP Top 10 bucket).

---

## 8. Legal Considerations Checklist

- [ ] Written authorization exists (SOW, ROE, or published bug bounty scope with safe harbor) before any testing begins.
- [ ] Testing stayed within the documented scope — no testing of explicitly out-of-scope assets or techniques.
- [ ] No real user data was accessed beyond the minimum needed for proof of concept; anything incidentally captured is redacted in reports and deleted after the engagement per the SOW's data-handling terms.
- [ ] No service disruption occurred (or, if authorized load/DoS testing was in scope, it was scheduled and coordinated in advance).
- [ ] All communications with the vendor about the vulnerability are encrypted and access-restricted.
- [ ] The disclosure timeline (standard/accelerated/extended) was communicated to the vendor at time of initial report, not sprung on them later.
- [ ] Public disclosure, if it happens, occurs only after the agreed date or explicit vendor sign-off — and is limited to technical detail necessary for defenders, not a step-by-step weaponized exploit for a still-unpatched issue.
