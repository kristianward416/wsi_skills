# Backend Security Practices

Reference for the Security Hardening Workflow in `senior-backend`. Consulted after `scripts/api_scaffolder.py` generates routes and before load-testing attack patterns with `scripts/api_load_tester.py --expect-rate-limit` / `--expect-status 400`. This skill owns *implementation* of these patterns; for formal threat modeling, fork to `engineering-team/skills/senior-security/` or `adversarial-reviewer` per the composition map in SKILL.md — don't try to do a full threat model from this doc alone.

---

## 1. OWASP API Security Top 10 — Mapped to Backend Concerns

| # | Risk | Backend concern | Primary mitigation |
|---|---|---|---|
| API1 | Broken Object Level Authorization (BOLA) | Endpoint checks auth but not "does *this* user own *this* object ID" | Authorize on every object access by ownership/tenant, not just endpoint-level RBAC — see §2 |
| API2 | Broken Authentication | Weak token handling, no MFA, predictable session tokens | OAuth2/OIDC + short-lived tokens — see §3 |
| API3 | Broken Object Property Level Authorization | Mass-assignment: client sets fields it shouldn't (`role: admin` in a signup payload) | Explicit allow-lists in validation schema — see §4 |
| API4 | Unrestricted Resource Consumption | No limits on pagination size, batch size, request body size, or concurrent requests | Rate limiting + pagination caps + body-size limits — see §5 |
| API5 | Broken Function Level Authorization | Admin-only endpoint reachable by a regular authenticated user | Deny-by-default RBAC/ABAC checked in middleware, not scattered in handlers |
| API6 | Unrestricted Access to Sensitive Business Flows | No abuse control on flows like "create account," "apply coupon," "request password reset" | Rate limit by business action, not just by endpoint — see §5 |
| API7 | Server-Side Request Forgery (SSRF) | Backend fetches a URL supplied by the client (webhooks, image-from-URL, PDF renderers) | Allow-list destination schemes/hosts, block link-local/metadata IP ranges (`169.254.169.254`), disable redirects to internal ranges |
| API8 | Security Misconfiguration | Verbose errors, default creds, permissive CORS, missing security headers | `helmet`-style headers (SKILL.md Step 5), least-privilege config, no debug mode in prod |
| API9 | Improper Inventory Management | Undocumented/legacy API versions still reachable, no ownership tracking | Keep OpenAPI spec generation in the loop (`api_scaffolder.py --generate-spec`) so shadow endpoints get caught |
| API10 | Unsafe Consumption of APIs | Trusting a third-party API's response without validation | Validate and schema-check inbound responses from upstream APIs just like inbound client requests |

---

## 2. Authorization: BOLA and Beyond

Authentication answers "who are you?"; authorization answers "can you touch *this* object?" — BOLA (API1) is the single most common real-world API vulnerability because teams check the first and skip the second.

**Wrong (endpoint-level only):**
```typescript
app.get('/invoices/:id', requireAuth, async (req, res) => {
  const invoice = await db.invoices.findById(req.params.id);
  res.json(invoice);  // any authenticated user can read any invoice by guessing IDs
});
```

**Right (object-level check on every access):**
```typescript
app.get('/invoices/:id', requireAuth, async (req, res) => {
  const invoice = await db.invoices.findById(req.params.id);
  if (!invoice || invoice.tenantId !== req.user.tenantId) {
    return res.status(404).json({ error: { code: 'NOT_FOUND', message: 'Invoice not found' } });
    // 404, not 403 — don't confirm the object exists to a caller who shouldn't see it
  }
  res.json(invoice);
});
```
- Scope the query itself by tenant/owner where possible (`WHERE id = $1 AND tenant_id = $2`) rather than fetching first and checking after — this also closes a timing side-channel and reduces DB load from unauthorized lookups.
- For multi-tenant systems (Forcing Question 2's tenancy model), enforce this at a shared middleware/repository layer, not per-handler — a single missed check in one handler is the entire vulnerability class.
- Use non-sequential IDs (UUIDv4/v7, not auto-increment integers) for any resource where object enumeration itself is a risk — sequential IDs make BOLA exploitation trivial to script even where authorization checks exist but are inconsistent.

---

## 3. Authentication: OAuth2/OIDC Flows and JWT Pitfalls

### Which OAuth2 flow
| Flow | Use for | Avoid for |
|---|---|---|
| Authorization Code + PKCE | Any user-facing login — web SPA, mobile, native app | Never use Implicit flow anymore; PKCE replaces it even for public clients |
| Client Credentials | Service-to-service, machine-to-machine, no user context | User-facing anything |
| Refresh Token Rotation | Keeping users logged in without long-lived access tokens | Storing a static long-lived refresh token client-side without rotation/revocation |
| Device Code | CLI tools, smart TVs, no browser on the input device | Anything with a normal browser available |

Never implement Resource Owner Password Credentials (ROPC) for anything new — it hands the client app the user's raw password.

### JWT pitfalls (this is where most home-grown auth breaks)
1. **`alg: none` / algorithm confusion.** Always pin the expected algorithm when verifying — never trust the `alg` header in the token itself:
```typescript
// vulnerable: verifier trusts whatever alg the token claims
jwt.verify(token, secretOrKey);

// correct: verifier pins the algorithm explicitly
jwt.verify(token, publicKey, { algorithms: ['RS256'] });
```
   This closes the classic attack where a token signed with `HS256` using the *public key* as the HMAC secret is accepted by a verifier that blindly trusts an `alg: HS256` header against an RS256 key.
2. **Prefer asymmetric (RS256/ES256) over symmetric (HS256)** for any token verified by more than one service — this matches SKILL.md's Step 1 guidance (`algorithm: 'RS256'`). With HS256, every verifier needs the same secret used to sign, meaning every verifying service can also *forge* tokens; with RS256, verifiers only hold the public key.
3. **Short expiry, always.** `expiresIn: '1h'` or shorter for access tokens (per SKILL.md), paired with a longer-lived, rotatable refresh token stored more securely (httpOnly cookie, not localStorage).
4. **JWTs are not sessions — you cannot revoke one before expiry** without extra infrastructure (a denylist keyed by `jti`, or a "token valid after" timestamp per user checked on every request). If your product needs "log out everywhere" or "force logout on password change," you need that revocation mechanism — a bare JWT scheme cannot do it.
5. **Never put sensitive data in the payload.** JWT payloads are base64-encoded, not encrypted — anyone with the token can read the claims. Roles/permissions are fine; PII, secrets, or anything sensitive is not.
6. **Validate `iss`, `aud`, and `exp`** explicitly, not just signature — a validly-signed token issued for a different audience/service should still be rejected.
7. **Store secrets/keys per §6, never hardcoded** — this is the exact line SKILL.md's Step 1 draws (`process.env.JWT_SECRET`).

### OIDC on top of OAuth2
OIDC adds the `id_token` (a JWT asserting identity claims) on top of OAuth2's `access_token` (which authorizes API calls). Don't use the `access_token` to authenticate a user's identity on your own backend and don't use the `id_token` as an API bearer token — they serve different purposes even though both are JWTs.

---

## 4. Rate Limiting for Abuse Prevention

Distinct from capacity-driven rate limiting in `api_design_patterns.md` §7 — here the adversary is intentionally trying to defeat the limit, so the design must assume evasion attempts.

- **Key by identity, not just IP.** For login/auth endpoints, apply two independent limiters: one per-IP (blunts distributed slow brute force from many accounts) and one per-account-identifier (blunts credential stuffing concentrated on one account from rotating IPs/botnets). Neither alone is sufficient.
- **Sensitive business flows need their own limits**, independent of general API throughput limits (OWASP API6): password reset requests, coupon/promo code redemption, signup, OTP verification. These are attacked directly regardless of overall API traffic shape.
- **Escalating response, not just reject:** consider CAPTCHA or step-up verification after N failed attempts rather than a flat lockout, to avoid trivial account-lockout DoS against a known username.
- **Don't leak signal in the rejection.** A `429` for rate limiting and a `401` for bad credentials should look identical in timing and payload shape to an attacker probing for valid usernames (avoid "this account doesn't exist" vs "wrong password" distinctions in the response).
- Apply the SKILL.md Step 4 verification discipline literally — after implementing a limiter, load test it with the attack shape it's meant to stop: `python scripts/api_load_tester.py https://api.example.com/login --concurrency 200 --duration 10 --expect-rate-limit`.

---

## 5. Input Validation and Mass Assignment

Validate at the boundary, with an explicit allow-list schema — never a deny-list, and never trust client-supplied field sets.

```typescript
import { z } from 'zod';

// Allow-list: only fields a client is permitted to set on signup.
// role/isAdmin/tenantId are deliberately absent — server sets them.
const CreateUserSchema = z.object({
  email: z.string().email().max(255),
  name: z.string().min(1).max(100),
  age: z.number().int().positive().optional(),
}).strict();  // .strict() rejects unknown keys outright — this is what stops mass assignment (API3)

const data = CreateUserSchema.parse(req.body);
const user = await userService.create({ ...data, role: 'member', tenantId: req.user.tenantId });
```
- `.strict()` (Zod) or equivalent (`forbidNonWhitelisted: true` in class-validator) is the difference between an allow-list and a filter that silently drops unknown fields but doesn't flag the anomaly — prefer the hard rejection so a client trying to smuggle `role: admin` gets a `400`, which also gives you a signal worth alerting on.
- Validate **size**, not just shape: max string lengths (as above), max array lengths, max request body size (`express.json({ limit: '100kb' })`) — unbounded input size is how API4 (Unrestricted Resource Consumption) turns into a resource-exhaustion DoS.
- Validate on the server always, even if the client also validates — client-side validation is UX, not security.
- SQL/NoSQL injection: use parameterized queries/prepared statements exclusively (`db.query('... WHERE id = $1', [id])`); never string-concatenate user input into a query. ORMs prevent this by default — the risk reappears the moment someone drops to raw SQL for a "complex" query and forgets to parameterize.
- Output encoding matters as much as input validation: escape/encode on output (HTML-escape for any value rendered into HTML, even if it "came from the database" — it was still user input once) to prevent stored XSS.

---

## 6. Secrets Management

- **Never hardcode secrets** — the exact rule in SKILL.md's Step 1 (`process.env.JWT_SECRET`). Extend it: env vars are the *minimum* bar, not the ceiling. For anything beyond a single-instance toy deployment, use a managed secrets store (AWS Secrets Manager, GCP Secret Manager, HashiCorp Vault) with rotation, not a static `.env` file checked into a deploy artifact.
- **Rotate on a schedule and on personnel change.** Any secret that hasn't been rotated in > 90 days, or that was known to a person who has left the team, should be treated as compromised.
- **Never log secrets.** Redact `Authorization` headers, tokens, and password fields in logging middleware before they reach any log sink — this is a common accidental leak vector (request/response logging middleware added for debugging, left on in production).
- **Distinguish secret classes and store accordingly:** DB credentials and API keys → secrets manager with IAM-scoped access; TLS private keys → key management service or short-lived cert issuance (mTLS with automatic rotation, e.g. via SPIFFE/SPIRE for service-to-service); JWT signing keys → asymmetric key pair with the private key never leaving the issuing service (see §3).
- **Principle of least privilege on secret access**, not just secret storage — a secrets manager doesn't help if every service has blanket read access to every secret. Scope IAM policies per-service to the specific secrets that service needs.

---

## 7. Data Sensitivity Tiers and Compliance Floor

Directly from Forcing Question 4 in SKILL.md — the data sensitivity tier (public / internal / PII / PHI / PCI) sets your compliance floor and should gate every decision below, not be layered on after the fact:

| Tier | Backend obligations |
|---|---|
| Public | Standard hardening (this doc), no special handling |
| Internal | Access control + audit logging on read, not just write |
| PII | Encryption at rest and in transit, data minimization (don't collect/store what you don't need), right-to-deletion support, breach notification plan |
| PHI | All PII obligations + HIPAA-specific: BAAs with subprocessors, audit logs immutable and retained per policy — fork to `ra-qm-team/` for compliance evidence per SKILL.md's composition map |
| PCI | Never store raw card numbers/CVV — tokenize via a PCI-compliant processor (Stripe, Braintree); if you touch raw card data at all, you're in PCI DSS SAQ D territory, not SAQ A — this is a decision to make explicitly, not stumble into |

If the sensitivity tier is unknown, stop and resolve it before hardening further — SKILL.md explicitly treats this as a blocking assumption, not a nice-to-have.

---

## 8. Security Headers and Transport

Baseline for every API, matching SKILL.md's Step 5:
```typescript
import helmet from 'helmet';

app.use(helmet({
  contentSecurityPolicy: true,
  crossOriginEmbedderPolicy: true,
  crossOriginOpenerPolicy: true,
  crossOriginResourcePolicy: true,
  hsts: { maxAge: 31536000, includeSubDomains: true },
}));
```
- `Strict-Transport-Security` with a 1-year `max-age` plus `includeSubDomains` — and only add `preload` once you're certain every subdomain is HTTPS-only, since HSTS preload is effectively irreversible for months.
- Set CORS explicitly with an allow-list of origins — never `Access-Control-Allow-Origin: *` on any endpoint that reads authenticated/cookie-based state; reflecting the request `Origin` header without validating it against an allow-list is equivalent to `*` for credentialed requests.
- Terminate TLS 1.2 minimum, prefer 1.3; disable legacy cipher suites at the load balancer/ingress, not the application.

---

## 9. Verification Discipline

Per SKILL.md's Karpathy-discipline section, no security recommendation from this doc is complete without stating what "hardened" means in verifiable terms for the system at hand: latency impact of the added checks (p50/p95/p99), the SLO it must not breach, and RPO/RTO for the auth/secrets stores themselves (a secrets manager outage is an incident, not a footnote). Load-test the hardened endpoint the same way SKILL.md's Step 4 does — validate both the attack case (`--expect-rate-limit`, `--expect-status 400`) and the legitimate-traffic case, to confirm the mitigation doesn't silently reject normal users at the p99 tail.
