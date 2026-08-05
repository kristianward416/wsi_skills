# Threat Modeling Guide

This guide is the reference companion to the senior-security skill's own STRIDE/DREAD workflow (see `scripts/threat_modeler.py`). It goes one level deeper than the workflow steps in SKILL.md: how to actually run STRIDE per DFD element, how to build an attack tree when a single threat needs decomposition, how DREAD scores are computed and defended, and how PASTA complements STRIDE when you need to justify a threat model to business stakeholders rather than just engineers.

## 1. Data Flow Diagrams: the prerequisite

Every threat model starts from a Data Flow Diagram (DFD), not from a threat list. A DFD has exactly four element types:

- **External Entity** — actors outside your trust boundary (users, third-party APIs, other services you don't control).
- **Process** — code that transforms data (a service, a function, a lambda).
- **Data Store** — anything that persists data at rest (database, cache, file system, message queue's retained messages).
- **Data Flow** — the arrows: data in motion between the above.

**Trust boundaries** are the lines you draw wherever privilege or ownership changes — a network perimeter, a container boundary, a process boundary between services owned by different teams, the boundary between "authenticated" and "unauthenticated" request handling. Threats cluster at trust boundary crossings; if your DFD has no trust boundaries drawn, you haven't scoped it yet.

Minimum viable DFD for threat modeling:
1. List every external entity that talks to the system.
2. List every process that touches data crossing a trust boundary.
3. List every data store, tagged with sensitivity (credentials, PII, financial, public).
4. Draw every data flow, labeled with protocol and whether it crosses a trust boundary.

This is exactly the input `threat_modeler.py --component "User Authentication" --assets "credentials,sessions"` expects — the DFD is what tells you what `--component` and `--assets` values to enumerate, one invocation per element.

## 2. STRIDE methodology

STRIDE maps six threat categories to security properties. Each letter is a *property being violated*, not a vulnerability class:

| Letter | Threat | Property violated | Typical mitigation family |
|---|---|---|---|
| S | Spoofing | Authentication | MFA, mutual TLS, signed tokens |
| T | Tampering | Integrity | HMAC/signatures, checksums, immutable logs |
| R | Repudiation | Non-repudiation | Audit logging, signed transactions, timestamps |
| I | Information Disclosure | Confidentiality | Encryption at rest/in transit, access control |
| D | Denial of Service | Availability | Rate limiting, autoscaling, redundancy, quotas |
| E | Elevation of Privilege | Authorization | Least privilege, input validation, sandboxing |

### Applying STRIDE per element type

Not every letter applies to every DFD element — this is the single most common mistake in shallow threat models (applying all six everywhere and drowning in noise). Use this matrix:

| DFD Element | S | T | R | I | D | E |
|---|---|---|---|---|---|---|
| External Entity | X | | X | | | |
| Process | X | X | X | X | X | X |
| Data Store | | X | X | X | X | |
| Data Flow | | X | | X | X | |

Rationale for the gaps: an External Entity can't be "tampered with" — you don't own it, so tampering is out of scope, but it can be spoofed (impersonated) and can repudiate actions. A Data Store can't be "spoofed" — spoofing applies to identities/actors, not passive storage — but it's a prime target for tampering, disclosure, and DoS (fill the disk, corrupt an index).

### Worked example: "User Authentication" process

Scope: a login endpoint that accepts username/password, checks credentials against a data store, and issues a session token. Assets: `credentials`, `sessions`.

Walk the process row (all six letters apply):

- **S — Spoofing:** An attacker submits stolen or guessed credentials and is authenticated as the victim. *Mitigation:* MFA, credential-stuffing rate limits, breached-password checks against a known-compromised list.
- **T — Tampering:** An attacker on a shared network modifies the login request in transit to inject a different username (parameter tampering) or modifies the "remember me" flag to extend session lifetime. *Mitigation:* TLS 1.2+, server-side re-validation of all fields, ignore/re-derive any client-supplied trust flags.
- **R — Repudiation:** A user denies having initiated a login, or an admin denies having reset another user's password. *Mitigation:* append-only audit log of authentication events (who, when, source IP, outcome), signed or hash-chained so it can't be quietly edited.
- **I — Information Disclosure:** The login error path reveals whether an account exists ("invalid password" vs "no such user"), enabling account enumeration; or the credential data store is exfiltrated in plaintext. *Mitigation:* generic error messages, salted+peppered password hashing (Argon2id — see `cryptography-implementation.md`), encryption at rest for the credential store.
- **D — Denial of Service:** An attacker floods the login endpoint, exhausting compute or triggering account lockouts for legitimate users (a lockout policy can itself become a DoS vector). *Mitigation:* per-IP and per-account rate limiting with exponential backoff, CAPTCHA after N failures, lockout thresholds tuned to avoid weaponization.
- **E — Elevation of Privilege:** A logic flaw lets a standard user's session token be replayed against an admin-only endpoint, or a JWT `alg: none` downgrade lets an attacker forge claims. *Mitigation:* server-side authorization checks independent of token claims, reject `alg: none`, pin expected signing algorithm.

Each row above is one row `threat_modeler.py` would emit as a `{component, category, dread: {damage, reproducibility, exploitability, affected_users, discoverability}, mitigation}` record. Six threats in, six mitigations out, before any DREAD triage happens.

## 3. Attack trees: decomposing a single threat

STRIDE tells you *what kind* of threat exists; an attack tree tells you *how* it could actually be realized, which matters when a single STRIDE row (e.g., "Spoofing the login process") has multiple independent attack paths that need different mitigations.

Structure: a root goal, decomposed into sub-goals connected by AND/OR:

```
GOAL: Authenticate as victim without their credentials
│
├── OR: Credential compromise
│   ├── OR: Phishing (harvest password directly)
│   ├── OR: Credential stuffing (reuse leaked password from another breach)
│   └── OR: Keylogger/malware on victim device
│
├── OR: Session hijacking
│   ├── AND: Steal session token (via XSS or network sniffing)
│   │   AND: Token not bound to device/IP → replay succeeds
│   └── OR: Session fixation (attacker sets victim's session ID before login)
│
└── OR: Authentication bypass
    ├── OR: SQL injection in login query
    ├── OR: JWT algorithm confusion (alg:none or RS256→HS256 key confusion)
    └── OR: Password reset flow flaw (predictable reset token)
```

Reading rules:
- **OR nodes**: any one child achieves the parent goal — the parent is only as secure as its *weakest* child.
- **AND nodes**: all children must succeed — the attacker needs every step, so breaking any one link defeats that path.

Use attack trees when: a DREAD score seems too coarse to act on, a pen-test or bug bounty report needs root-causing into a specific control gap, or you need to show stakeholders *why* a single mitigation ("add MFA") doesn't close the whole goal (it kills the credential-compromise branch but does nothing for session hijacking or injection bypass — those need separate fixes).

Each leaf of an attack tree is itself a good candidate to re-score with DREAD, since leaves are concrete enough to estimate exploitability and discoverability accurately, whereas the root goal usually isn't.

## 4. DREAD scoring

DREAD scores five dimensions, each 1–10, then averages (or sums, depending on convention — this skill uses the average, matching `threat_modeler.py`'s output):

| Dimension | Question | 1 (low) | 10 (high) |
|---|---|---|---|
| **D**amage | How bad is impact if exploited? | Minor cosmetic issue | Full data breach / RCE |
| **R**eproducibility | How reliably can it be triggered? | Race condition, rarely works | Works every time, scriptable |
| **E**xploitability | How much skill/access is needed? | Requires physical access + insider | Public exploit, no auth needed |
| **A**ffected users | What fraction of users/systems? | Single test account | All users / all tenants |
| **D**iscoverability | How easy to find? | Requires source access + deep analysis | Visible in browser dev tools |

Worked scoring for the login-endpoint account-enumeration threat above (STRIDE: Information Disclosure):

- Damage: 4 — reveals account existence, not credentials directly, but feeds targeted phishing/stuffing.
- Reproducibility: 10 — deterministic, every request behaves identically.
- Exploitability: 9 — no auth needed, trivial to script.
- Affected users: 8 — every registered account is enumerable.
- Discoverability: 9 — visible by comparing two error responses in a browser.

Average = (4+10+9+8+9)/5 = **8.0** → per the SKILL.md verification gate, anything ≥7 average requires a named mitigation owner before ship. This one clears the bar decisively — generic error messages ship before release, owner assigned to the auth team lead.

Compare to the JWT `alg:none` bypass leaf from the attack tree: Damage 9 (full auth bypass), Reproducibility 10, Exploitability 7 (needs to know the library accepts `none`), Affected users 10, Discoverability 5 (needs to inspect JWT handling code or fuzz it) → average 8.2. Both clear the ≥7 gate, but the enumeration fix is a one-line error-message change while the JWT fix requires a library/config change and regression tests — DREAD tells you *severity*, not *effort*; don't conflate the two when assigning owners and deadlines.

**Common DREAD pitfalls:**
- Scoring in a vacuum without the attack tree leaf detail — coarse "spoofing is bad, 8/10 everything" scoring defeats the purpose of having five dimensions.
- Letting Damage dominate — a catastrophic-but-undiscoverable-and-unreproducible threat (e.g., a flaw requiring physical hardware access to a data-center rack) should score low overall even with Damage=10, because R/E/D pull the average down. If your team keeps overriding the average toward the worst dimension, you're implicitly doing MAX not AVG — be explicit about which convention you use and stay consistent, since flipping conventions mid-project makes historical scores incomparable.

## 5. PASTA: when STRIDE isn't enough for stakeholders

STRIDE is a *system-centric* methodology: you enumerate elements, apply the matrix, get threats. That's fast and good for engineering-level threat modeling, but it doesn't tie threats to business impact or attacker motivation, which is often what security leadership and risk-acceptance signoffs need.

PASTA (Process for Attack Simulation and Threat Analysis) is a seven-stage, risk-centric methodology that wraps around STRIDE-level detail with business context:

1. **Define objectives** — business goals, compliance requirements (e.g., PCI-DSS scope), risk tolerance.
2. **Define technical scope** — the same DFD work as STRIDE, plus infrastructure/technology inventory.
3. **Application decomposition** — identify use cases, entry points, trust levels, assets — this is where STRIDE's element enumeration plugs in directly.
4. **Threat analysis** — pull from threat intelligence (CVE feeds, ATT&CK, industry incident reports) to identify *who* would target this system and *why*, not just *what could go wrong structurally*.
5. **Vulnerability & weakness analysis** — map known CVEs, misconfigurations, and code-level weaknesses (CWE) onto the decomposition from stage 3.
6. **Attack modeling** — build attack trees (section 3) or attack graphs simulating how identified threat actors would chain vulnerabilities to reach the objectives from stage 1.
7. **Risk & impact analysis** — quantify business impact (financial, regulatory, reputational) and residual risk after mitigations, feeding a formal risk-acceptance or remediation decision.

**When to reach for PASTA instead of (or on top of) plain STRIDE:**
- The audience for the threat model includes non-engineers (execs, auditors, compliance officers) who need "why does this matter to the business" framed explicitly, not just a threats table.
- The system is in a regulated scope (PCI-DSS, HIPAA, SOC 2) where the threat model needs to demonstrate alignment with a named risk methodology for an auditor.
- Threat intelligence should drive prioritization — e.g., you're specifically worried about a threat actor class (ransomware crews, insider threat, nation-state) rather than generic structural weaknesses.

**When plain STRIDE/DREAD is sufficient:** day-to-day engineering threat modeling of a new feature or component, where the audience is the engineering team and the output feeds directly into a sprint's mitigation backlog — this is the common case this skill's workflow targets, and running full PASTA for every feature PR is overkill. Reserve PASTA for system-level or compliance-driven threat models, not per-component STRIDE passes.

## 6. Putting it together: the end-to-end flow

1. Build the DFD, draw trust boundaries.
2. Run the STRIDE matrix per element type — for Processes, all six letters; for others, use the reduced matrix in section 2 to avoid noise.
3. For any STRIDE row with multiple plausible realizations or that a reviewer disputes, decompose it into an attack tree and re-score the leaves individually.
4. Score every threat with DREAD (or attack-tree leaf) using a consistent averaging convention.
5. Sort by DREAD descending. Everything ≥7 gets a named mitigation owner — no exceptions, no "we'll track it in the backlog" without an owner and a date.
6. If the model needs to justify itself to auditors, compliance, or leadership, wrap the STRIDE/DREAD output in a PASTA stages 1, 4, and 7 (objectives, threat-actor context, business impact) — you don't need to redo stages 2–3 and 6, since STRIDE and the attack tree already cover that ground.
7. Re-run the threat model (and the secret scanner per SKILL.md step 4) after mitigations land. The re-run passing is the completion signal — not the document being written.
