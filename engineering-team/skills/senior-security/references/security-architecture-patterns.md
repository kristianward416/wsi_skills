# Security Architecture Patterns

This reference covers the architecture-level patterns referenced by the senior-security skill's routing table: Zero Trust, defense-in-depth, secure-by-design patterns (least privilege, segmentation), authentication patterns, and API gateway security. These are the design-time controls that STRIDE mitigations map onto — when a threat model surfaces "Elevation of Privilege → least privilege," this document is where "least privilege" gets a concrete implementation shape.

## 1. Zero Trust Architecture

Zero Trust's core assumption: **no network location is trusted by default, including the internal network.** This replaces the older perimeter/castle-and-moat model where anything inside the firewall was implicitly trusted. NIST SP 800-207 formalizes this as a set of tenets; the practical ones:

- Every access request is authenticated and authorized per-request, not per-session-at-the-perimeter. A request from inside the VPC gets the same scrutiny as one from the public internet.
- Access is granted per-resource, based on identity + device posture + context (time, location, behavior), not by network segment membership alone.
- All traffic is encrypted regardless of network location — internal service-to-service traffic uses TLS too, not just north-south traffic at the edge.

**Implementation building blocks:**

- **Mutual TLS (mTLS)** between services — each service has an identity (certificate), and both sides verify each other. A service mesh (Istio, Linkerd) typically automates mTLS issuance/rotation so individual services don't hand-roll certificate management.
- **Policy enforcement point per resource**, not just at the network edge — every service validates the caller's identity and authorization on every call, e.g., via a sidecar proxy or an in-process authz library (OPA/Open Policy Agent is a common choice: policies as code, evaluated per request).
- **Device/workload identity**, not just user identity — SPIFFE/SPIRE is the common open framework for issuing cryptographic workload identities (SVIDs) so a service knows *which* service is calling it, not just that a valid cert was presented.
- **Continuous verification** — session validity is re-checked against current context (has the device posture changed? has the token been revoked?) rather than trusted for the session's full lifetime.

**Zero Trust vs. perimeter model — concrete contrast:**

| | Perimeter model | Zero Trust |
|---|---|---|
| Trust basis | Network location (inside firewall = trusted) | Verified identity + context, every request |
| Internal traffic | Often plaintext, unauthenticated | mTLS + per-request authz |
| Lateral movement after breach | Easy — one foothold reaches everything internal | Constrained — each hop re-authenticates |
| Segmentation | Coarse (DMZ vs internal) | Fine-grained, per-resource policy |

**When it's the right investment:** multi-tenant systems, systems handling regulated data, any environment where "an attacker got a foothold on one internal host" is a realistic threat you're modeling (which, per a STRIDE pass, it almost always is for any networked system). It's a heavier operational lift (cert lifecycle, policy management) — for a small internal tool with no sensitive data and a tiny attack surface, defense-in-depth without full Zero Trust tooling may be proportionate instead.

## 2. Defense in Depth

Zero Trust is one layer; defense-in-depth is the principle that no single control should be a single point of failure — layer independent controls so that one bypass doesn't equal full compromise.

Layered model for a typical web-facing service:

```
Internet
  │
  ▼
[1] Edge/CDN — DDoS mitigation, WAF (OWASP CRS rules), TLS termination
  │
  ▼
[2] API Gateway — authn (token validation), coarse authz, rate limiting, input schema validation
  │
  ▼
[3] Network segmentation — service can only reach the specific downstream services it needs (enforced by network policy / security groups, not just documentation)
  │
  ▼
[4] Application layer — per-request authz checks, input validation/sanitization at every trust boundary, output encoding
  │
  ▼
[5] Data layer — encryption at rest, least-privilege DB credentials (this service's DB user can SELECT/INSERT on its tables only, not DROP or cross-schema access), parameterized queries
  │
  ▼
[6] Audit/detection — logging of security-relevant events, anomaly detection, alerting (this is the layer that turns "we got breached" into "we detected it in minutes")
```

The test for whether you actually have defense-in-depth, versus just calling one control by six names: **pick any single layer, assume it fails completely, and ask whether the system still contains the blast radius.** If a WAF bypass alone leads directly to full database access, layers 3–5 aren't actually independent — the network segmentation, app-layer authz, and DB credential scoping should each independently stop that particular attack path.

Concrete illustration: an SQL injection payload that gets past the WAF (layer 1 fails) should still be defeated by parameterized queries (layer 4), and even if the query layer also has a flaw, a compromised app should only be able to reach the tables its DB credential is scoped to (layer 5) — not the entire database, other tenants' schemas, or the `users.password_hash` column if that service doesn't need it.

## 3. Secure-by-Design: Least Privilege

Least privilege means every principal — a user, a service, a process, a DB credential — holds the *minimum* set of permissions needed for its current task, for the minimum necessary duration, and nothing else "just in case."

**Applying it across principal types:**

- **Human users:** role-based access control (RBAC) scoped to job function, not "give everyone admin because it's easier." Time-bound elevation for admin actions (just-in-time access / break-glass procedures with audit logging) instead of standing admin privileges.
- **Service accounts / workload identities:** a service's credential should be scoped to exactly the resources it calls — a service that only reads from a queue and writes to one table should have a DB user that can `SELECT` from zero tables and `INSERT` into exactly one, not a shared "app_user" role with blanket read/write across the schema.
- **IAM roles in cloud environments:** avoid `Action: "*"` / `Resource: "*"` policies. Scope IAM policies to specific actions and resource ARNs. Use permission boundaries and SCPs (Service Control Policies in AWS Organizations) as an outer bound that individual role policies can't exceed even if misconfigured.
- **API tokens:** scope tokens to specific operations (read-only vs read-write, specific endpoints) rather than issuing all-powerful API keys. Short-lived tokens (OAuth access tokens with short TTLs + refresh tokens) beat long-lived static API keys for anything crossing a trust boundary.

**Anti-pattern to name explicitly:** the "temporary" broad grant that never gets revoked — a debugging session gets `admin` on a database "just for today" and the revocation never happens. Least privilege requires a *process*, not just a policy: access reviews on a cadence, automatic expiry on elevated grants, and alerting when a grant broader than a role's baseline has been active longer than its stated justification.

## 4. Secure-by-Design: Segmentation

Segmentation limits lateral movement by ensuring that compromising one component doesn't grant reachability to everything else.

**Network segmentation:**
- Separate network zones/VPCs for tiers with different trust levels (public-facing edge, application tier, data tier) with explicit allow-lists between them (security groups / NACLs / firewall rules) rather than a flat network where every host can reach every other host.
- Microsegmentation goes further: per-service network policies (e.g., Kubernetes `NetworkPolicy`) so service A can only reach service B on the specific port it needs, not the entire pod network.

**Data segmentation:**
- Multi-tenant systems: enforce tenant isolation at the data layer, not just the application layer — row-level security (Postgres RLS) or separate schemas/databases per tenant for higher-sensitivity data, so an application-layer authz bug doesn't automatically mean cross-tenant data leakage.
- Sensitive data classes (credentials, PII, payment data) live in separate stores from general application data, with separate, narrower access paths — this is also what makes a PCI-DSS or similar compliance scope tractable, since it shrinks the boundary of "systems that touch card data" rather than making the entire application in-scope.

**Segmentation and the STRIDE/DREAD workflow:** when a threat-modeling pass turns up a high-DREAD Elevation of Privilege or Information Disclosure threat whose mitigation is "limit blast radius," segmentation is usually the concrete mechanism — the mitigation owner should be able to point at a specific network policy, IAM boundary, or schema separation, not just write "improve isolation" as a mitigation note.

## 5. API Gateway Security Patterns

The API gateway is the layer-2 control in the defense-in-depth stack above, and it's worth its own detail because it's the most common place organizations under-invest.

**Core responsibilities a gateway should own:**

1. **Authentication enforcement** — validate bearer tokens (JWT signature + expiry + issuer/audience claims) or API keys before any request reaches an upstream service. Upstream services should not have to re-implement token validation from scratch — but per Zero Trust, they *should* still independently verify authorization for the specific action, not just trust "the gateway let it through."
2. **Rate limiting and quota enforcement** — per-client, per-endpoint limits (token bucket or sliding window) to blunt both abusive clients and credential-stuffing/brute-force attempts. Layer this per-IP *and* per-authenticated-identity — an attacker rotating source IPs but reusing one stolen token should still get throttled.
3. **Input validation at the edge** — schema validation (OpenAPI/JSON Schema) rejecting malformed requests before they reach application code, reducing the attack surface app-layer parsers are exposed to.
4. **TLS termination and re-encryption** — terminate client TLS at the edge, but re-encrypt (mTLS) to upstream services rather than dropping to plaintext internally, consistent with Zero Trust's "no trusted network" tenet.
5. **Request/response logging for audit** — the R (Repudiation) mitigation from STRIDE lives here as much as in application code: every request through the gateway logged with identity, timestamp, and outcome, feeding the audit layer of the defense-in-depth stack.

**Common gateway misconfigurations to check for:**
- Gateway validates token *signature* but not *audience/issuer* claims — a token legitimately issued for a different service/tenant gets accepted here too (a classic confused-deputy setup).
- Rate limits keyed only by IP, trivially bypassed by rotating source addresses or via a shared NAT'd corporate network causing false positives for legitimate users.
- Gateway-level authn treated as sufficient authz — an authenticated-but-unauthorized request (valid token, wrong permissions) passed straight through because the gateway only checks "is this token valid," not "is this token allowed to call this specific route." Route-level, and ideally resource-level, authorization needs to happen either at the gateway with real policy data or at the service — never assumed away.
- CORS configured with a wildcard origin (`Access-Control-Allow-Origin: *`) alongside credentialed requests — this is invalid per spec for credentialed requests in most implementations, but permissive reflection of the `Origin` header (echoing back whatever origin sent the request) achieves the same broken effect and should be flagged as an Information Disclosure risk.
- Backend services reachable directly, bypassing the gateway entirely, because network segmentation (section 4) wasn't actually enforced — the gateway's controls are meaningless if there's a path around it.

**Gateway pattern for microservices — BFF vs. shared gateway:** a single shared API gateway in front of many backend services centralizes the controls above but becomes a large blast-radius target itself and a potential bottleneck for policy changes across teams. A Backend-for-Frontend (BFF) pattern — a thinner gateway layer per client type (web BFF, mobile BFF) — narrows each gateway's scope and blast radius at the cost of some duplicated cross-cutting logic. Choose based on team topology: one team owning all client integration favors a shared gateway; multiple independent frontend teams favor BFFs with shared underlying policy libraries to avoid drift.

## 6. Tying architecture patterns back to the threat model

This document is not a standalone checklist — its patterns are what "mitigation" cells resolve to once a STRIDE/DREAD pass identifies a threat:

| STRIDE category | Architecture pattern from this doc |
|---|---|
| Spoofing | mTLS/workload identity (Zero Trust §1), gateway token validation (§5) |
| Tampering | Segmentation limiting write-reachability (§4), input validation at gateway (§5) |
| Repudiation | Audit logging layer in defense-in-depth (§2), gateway request logging (§5) |
| Information Disclosure | Data segmentation (§4), least-privilege data access (§3), encryption (see cryptography-implementation.md) |
| Denial of Service | Rate limiting at gateway (§5), redundancy across defense-in-depth layers (§2) |
| Elevation of Privilege | Least privilege (§3), per-request authz under Zero Trust (§1), route-level gateway authz (§5) |

When a DREAD score ≥7 threat lands on your desk, the mitigation owner should be able to point to a specific pattern here (which network policy, which IAM boundary, which gateway rule) — not a vague "we'll harden this" note.
