# Security Standards

Reference for `senior-secops`: secure coding standards, authentication requirements, API security controls, and the OWASP Top 10 as it applies to day-to-day engineering (not pen-testing engagements — see `security-pen-testing` for offensive-security depth on the same list).

## OWASP Top 10 (2021) — Engineering Checklist

| # | Category | What to check in code review / CI |
|---|----------|-----------------------------------|
| A01 | Broken Access Control | Every route/handler enforces authorization server-side, not just UI hiding. No IDOR (object references checked against the caller's ownership, not just existence). |
| A02 | Cryptographic Failures | TLS everywhere in transit; sensitive fields encrypted at rest; no home-rolled crypto; secrets never in source or logs. |
| A03 | Injection | Parameterized queries only; no string-concatenated SQL/NoSQL/LDAP/OS commands; output-encode for XSS contexts. |
| A04 | Insecure Design | Threat model exists for the feature (route to `senior-security` for STRIDE); abuse cases considered, not just happy path. |
| A05 | Security Misconfiguration | No default credentials; unused features/ports disabled; verbose errors suppressed in production; security headers set (see below). |
| A06 | Vulnerable and Outdated Components | Dependency manifest scanned (`vulnerability_assessor.py`); no components past EOL. |
| A07 | Identification and Authentication Failures | MFA available for sensitive accounts; session tokens rotated on privilege change; no credential stuffing exposure (rate limiting, lockout). |
| A08 | Software and Data Integrity Failures | CI/CD pipeline verifies artifact signatures; no unsigned/untrusted plugins auto-updated; deserialization of untrusted data avoided or strictly typed. |
| A09 | Security Logging and Monitoring Failures | Auth failures, access-control failures, and input-validation failures are logged with enough context to investigate; logs shipped somewhere the app compromise can't erase. |
| A10 | Server-Side Request Forgery (SSRF) | Outbound requests from user-controlled URLs are allow-listed or routed through a proxy that blocks internal address ranges. |

## Secure Coding Standards

**Input validation.** Validate on the server, not just the client. Prefer allow-lists (accept known-good patterns) over deny-lists (block known-bad patterns) — deny-lists always miss a variant.

**Output encoding.** Encode for the context you're writing into (HTML entity encoding for HTML bodies, JS string escaping for inline scripts, URL encoding for query params). One encoding function does not cover every sink.

**Error handling.** Never return stack traces, internal file paths, or database error text to the client. Log the detail internally; return a generic error with a correlation ID the user/support can reference.

**Dependency hygiene.** Pin versions in lockfiles. Treat a new CVE against a pinned dependency as a ticket the moment `vulnerability_assessor.py` reports it — see the CVE Triage SLA table in the main SKILL.md.

## Authentication Requirements

- Passwords hashed with a slow, salted algorithm (bcrypt/argon2/scrypt) — never plain MD5/SHA1/SHA256 alone.
- Session tokens are high-entropy, transmitted only over TLS, marked `HttpOnly` and `Secure`, and rotated on login and on privilege escalation.
- Failed-login handling uses progressive delay or lockout with a safe unlock path (not permanent lockout from a single actor's typo).
- MFA is available for any account with elevated privileges; SSO/OIDC preferred over bespoke auth for anything customer-facing at scale.

## API Security Controls

- Every endpoint declares required scopes/roles explicitly; nothing is "authenticated by omission."
- Rate limiting is per-identity (API key/user/IP tuple), not global, so one abusive caller can't exhaust the budget for everyone.
- Versioned APIs deprecate old versions on a published timeline rather than running unlimited parallel surface area.
- Webhook receivers verify a signature (HMAC over the raw body) before trusting payload contents.

## Security Headers (baseline for any web-facing service)

```
Content-Security-Policy: default-src 'self'; ...   (tightened per app, no wildcard `*` in production)
Strict-Transport-Security: max-age=63072000; includeSubDomains; preload
X-Content-Type-Options: nosniff
X-Frame-Options: DENY   (or CSP frame-ancestors)
Referrer-Policy: strict-origin-when-cross-origin
Permissions-Policy: geolocation=(), camera=(), microphone=()
```

Run `security_scanner.py --headers <url>` to check a live endpoint against this baseline; exit code 1 means at least one required header is missing or misconfigured.
