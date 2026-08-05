# API Design Patterns

Reference for the API Design Workflow in `senior-backend`. Consulted when scaffolding new endpoints with `scripts/api_scaffolder.py`, defining OpenAPI contracts, or choosing between REST/GraphQL/gRPC before locking a profile in `backend_decision_engine.py`. Aligns with the response/error formats and status-code table in `SKILL.md`'s Common Patterns Quick Reference.

---

## 1. Choosing a Protocol: REST vs GraphQL vs gRPC

Don't pick by fashion — pick by access pattern, client diversity, and the read/write ratio + QPS forecast from Forcing Question 1.

| Dimension | REST | GraphQL | gRPC |
|---|---|---|---|
| Client shape | Many heterogeneous clients (web, mobile, partners) | Many clients with divergent data needs (product surfaces, BFFs) | Small number of known internal services |
| Payload efficiency | Over/under-fetching common | Client specifies exact fields | Binary protobuf, smallest wire size |
| Caching | HTTP caching (ETags, CDN) works natively | Cache-unfriendly by default; needs persisted queries | Not cache-friendly; RPC semantics |
| Streaming | Awkward (SSE/long-poll bolt-ons) | Subscriptions (usually over WS) | First-class (unary, server-stream, client-stream, bidi) |
| Tooling maturity | Universal | Strong but needs a gateway | Best for polyglot microservices, needs codegen pipeline |
| Latency floor | Good | Adds resolver overhead — expect +10-30ms per nested resolver fan-out | Lowest (HTTP/2 + binary framing) |
| Versioning | Well-understood conventions | Schema evolves via deprecation, not versioning | Protobuf field numbers give backward compat "for free" |

**Decision tree:**
1. Is this internal service-to-service traffic with a known set of callers and QPS ≥ 1000 (the `go-or-rust-microservice` profile threshold in SKILL.md)? → gRPC.
2. Do you have ≥ 3 client surfaces with materially different data shapes off the same resources (e.g., mobile app needs a thin payload, admin dashboard needs deep nesting)? → GraphQL, but only if you can staff a gateway/persisted-queries layer. Otherwise the resolver N+1 problem (see `database_optimization_guide.md` §2) will dominate your latency budget.
3. Everything else, including anything under the `node-express`/`fastapi-python`/`django-monolith` profiles → REST. It's the default because it composes with HTTP caching, is debuggable with curl, and every profile's 600-800ms p99 floor in SKILL.md assumes REST-shaped request/response, not resolver graphs.

Mixed estates are normal: public REST API + internal gRPC between services + an optional GraphQL BFF in front of REST for a single high-value client (e.g., a mobile app) is a legitimate end-state, not a compromise.

---

## 2. Resource Modeling and URL Design

- Nouns, not verbs: `/orders`, not `/getOrders`.
- Nesting depth ≤ 2: `/users/{id}/orders` is fine; `/users/{id}/orders/{id}/line-items/{id}/discounts` is not — promote deep resources to top-level with a filter: `/discounts?line_item_id=123`.
- Plural collections always: `/users`, `/users/{id}`.
- Actions that don't map to CRUD get a verb sub-resource, not a query param hack: `POST /orders/{id}/cancel`, `POST /invoices/{id}/void`. This keeps the action auditable and idempotency-keyable (see §5).
- Filtering/sorting/field-selection via query params, not new endpoints: `GET /orders?status=paid&sort=-created_at&fields=id,total`.

---

## 3. Versioning Strategies

| Strategy | Example | When to use | Cost |
|---|---|---|---|
| URI versioning | `/v1/users`, `/v2/users` | Public APIs, partner integrations | Simple, cacheable, but encourages "fork and abandon v1" sprawl |
| Header versioning | `Accept: application/vnd.acme.v2+json` | When URL stability matters (bookmarks, webhooks) | Harder to test with curl/browser, but keeps URLs canonical |
| Query param versioning | `/users?version=2` | Rare — mostly for internal quick toggles | Pollutes cache keys, generally discouraged |
| No versioning, additive-only evolution | — | Internal services under your own release control | Requires strict "never remove/rename a field" discipline; cheapest long-term if you can enforce it |

**Additive-only rules** (use regardless of strategy, to delay a version bump as long as possible):
- Only add optional fields; never remove or rename fields, never change a field's type or semantics.
- New enum values are additive; consumers must treat unknown enum values as "unknown," not crash.
- Deprecate via `Deprecation` and `Sunset` HTTP headers (RFC 8594) for at least one full deprecation window (90 days minimum for anything with external consumers) before removal.
- Track breaking-change risk formally — this skill forks that concern to `engineering/skills/api-design-reviewer/` per the composition map in SKILL.md; don't freelance a breaking change through this reference alone.

---

## 4. Pagination Patterns

| Pattern | Mechanism | Pros | Cons | Use when |
|---|---|---|---|---|
| Offset/limit | `?limit=20&offset=40` | Simple, supports "jump to page N" | O(offset) scan cost in the DB; drifts under concurrent writes (skipped/duplicated rows) | Small, mostly-static tables; admin UIs |
| Cursor (keyset) | `?limit=20&cursor=eyJpZCI6MTIzfQ==` (opaque, base64-encoded last-seen sort key) | O(1) per page via index seek; stable under writes | No "jump to page N"; cursor must encode full sort key, not just id, if sorting by non-unique column | Feeds, activity logs, any high-write table |
| Seek with `(created_at, id)` tuple | `WHERE (created_at, id) < ($1, $2) ORDER BY created_at DESC, id DESC LIMIT 20` | Handles ties in `created_at` correctly | Requires composite index matching the tuple order | Time-ordered lists where timestamps collide |

**Rule of thumb:** default to cursor pagination for any collection expected to exceed ~10k rows or to receive concurrent writes during paging. Offset pagination beyond `offset=10000` on Postgres routinely costs 50-200ms+ just walking and discarding rows — this is the same class of problem covered in `database_optimization_guide.md` §1 (index-only scans vs seq scans).

Response envelope (consistent with SKILL.md's `meta` convention):
```json
{
  "data": [ { "id": 101, "name": "..." } ],
  "meta": {
    "requestId": "abc-123",
    "nextCursor": "eyJpZCI6MTIwfQ==",
    "hasMore": true
  }
}
```

---

## 5. Idempotency

Required for any `POST` that creates a resource with a side effect that must not double-fire on retry (payments, order creation, email sends). Retries happen constantly — client timeouts, load balancer failover, mobile network flaps.

**Idempotency-Key pattern (Stripe-style):**
```typescript
app.post('/payments', async (req, res) => {
  const idempotencyKey = req.header('Idempotency-Key');
  if (!idempotencyKey) {
    return res.status(400).json({
      error: { code: 'MISSING_IDEMPOTENCY_KEY', message: 'Idempotency-Key header is required' }
    });
  }

  const existing = await idempotencyStore.get(idempotencyKey);
  if (existing) {
    // Replay the original response verbatim — same status, same body
    return res.status(existing.status).json(existing.body);
  }

  const lock = await idempotencyStore.acquireLock(idempotencyKey, { ttlMs: 30_000 });
  if (!lock) {
    return res.status(409).json({ error: { code: 'REQUEST_IN_FLIGHT', message: 'Duplicate request already processing' } });
  }

  try {
    const payment = await paymentService.charge(req.body);
    await idempotencyStore.save(idempotencyKey, { status: 201, body: payment }, { ttlHours: 24 });
    return res.status(201).json(payment);
  } finally {
    await idempotencyStore.releaseLock(idempotencyKey);
  }
});
```

Design notes:
- Key scope: idempotency key + a hash of the request body should both be checked — same key with a different body is a client bug, return `422`.
- TTL: store idempotency records for at least as long as your client's retry window plus margin (24h is a common default for payment-adjacent APIs).
- `PUT` is naturally idempotent by HTTP semantics (full replace) — you don't need a key for `PUT`, only for `POST` that creates.
- `DELETE` should be idempotent too: deleting an already-deleted resource returns `204`, not `404` (some teams prefer `404`; pick one and document it — this is one of the "conventions to lock" the API-design-reviewer fork checks for).

---

## 6. Error Response Conventions

Use the structured error envelope from SKILL.md consistently — never return bare strings or HTML error pages from an API:

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Invalid email format",
    "details": [{ "field": "email", "message": "must be valid email" }]
  },
  "meta": { "requestId": "abc-123" }
}
```

- `code` is a stable, machine-readable, UPPER_SNAKE_CASE string clients can switch on — never change an existing code's meaning, only add new ones (same additive-evolution discipline as §3).
- `message` is human-readable and safe to log/display; never leak stack traces, SQL, or internal file paths (see `backend_security_practices.md` §5 on information disclosure).
- `details` is an array for multi-field validation errors so the client can highlight every bad field in one round trip, not one-at-a-time.
- Always include `requestId` in `meta` — it's the join key between client bug reports and server logs/traces.
- Map errors to status codes consistently with the table in SKILL.md; the two codes teams most often misuse:
  - `401` = "who are you?" (missing/invalid credentials). `403` = "I know who you are, you can't do this." Conflating them leaks information about resource existence to unauthenticated callers.
  - `409` for conflicting state (duplicate unique key, version mismatch on optimistic locking), not `400`.

---

## 7. Rate Limiting Algorithms

Rate limiting serves two different goals — capacity protection (SLO-driven) and abuse prevention (security-driven, see `backend_security_practices.md` §4). The algorithm choice matters.

### Token Bucket
State: `tokens` (current count), `capacity` (max burst), `refillRate` (tokens/sec).
```
on request:
  now = current_time()
  elapsed = now - last_refill_time
  tokens = min(capacity, tokens + elapsed * refillRate)
  last_refill_time = now
  if tokens >= 1:
    tokens -= 1
    allow()
  else:
    reject()  # 429
```
- Allows bursts up to `capacity`, then throttles to `refillRate` steady-state — good for APIs where occasional bursts (a user paging through results quickly) are legitimate.
- Redis implementation: store `tokens` and `last_refill_time` in a hash, update atomically via `EVAL` (Lua script) or `MULTI`/`EXEC` to avoid race conditions across concurrent requests from the same key.

### Sliding Window Counter
Approximates a true sliding window using two fixed windows, cheaper than storing every timestamp:
```
current_window_count = requests in [now_bucket]
previous_window_count = requests in [now_bucket - 1]
weight = (window_size - elapsed_in_current_window) / window_size
estimated_count = current_window_count + previous_window_count * weight
if estimated_count < limit:
  allow(); increment(current_window_count)
else:
  reject()
```
- More accurate than fixed-window (avoids the "2x burst at window boundary" flaw: fixed window lets `limit` requests at 23:59:59 and another `limit` at 00:00:01).
- Standard choice for API gateways (this is what `express-rate-limit`'s default store approximates, and what Cloudflare/Kong use).

### Fixed Window
Simplest, cheapest (one counter + TTL per window), but has the boundary-burst flaw above. Acceptable for coarse, generous limits (e.g., 10k req/day) where a 2x burst at the boundary doesn't threaten capacity.

### Leaky Bucket
Same math as token bucket but framed as a queue with constant drain rate — used when you want to *smooth* traffic into a downstream system (e.g., queueing webhook deliveries) rather than just reject bursts.

**Applying to SKILL.md's example:**
```typescript
import rateLimit from 'express-rate-limit';

const apiLimiter = rateLimit({
  windowMs: 15 * 60 * 1000,  // sliding-window-counter under the hood with the default store
  max: 100,
  standardHeaders: true,     // emits RateLimit-* headers per IETF draft
  legacyHeaders: false,
});
app.use('/api/', apiLimiter);
```
Key the limiter by authenticated user ID when available, falling back to IP — IP-only limiting punishes NAT'd/corporate users and is trivially evaded by rotating IPs. For login/auth endpoints specifically, rate-limit by both IP *and* account identifier to blunt both credential stuffing (many accounts, one IP) and distributed brute force (one account, many IPs) — see `backend_security_practices.md` §4.

Set limits from measured capacity, not guesses: `max ≈ (target p99 latency budget) × (safe concurrent connections per instance) × (instance count)`, then load-test the number with `scripts/api_load_tester.py --concurrency N --expect-rate-limit` before shipping it.

---

## 8. Bulk Operations and Batching

- Prefer a dedicated batch endpoint (`POST /orders/batch`) over N sequential calls — reduces round trips and lets the server use a single transaction/bulk insert.
- Cap batch size explicitly (e.g., 100 items) and return `207 Multi-Status` with a per-item result array when partial failure is possible — don't force all-or-nothing semantics on a batch unless the use case is genuinely transactional.
- For GraphQL, this is the same problem DataLoader solves for reads — batch and dedupe within a single tick (see `database_optimization_guide.md` §2).

---

## 9. HATEOAS and Hypermedia (When It's Worth It)

Full HATEOAS (embedding `_links` for every possible next action) is rarely worth the complexity for internal or single-client APIs — it mainly pays off for APIs with many independent third-party consumers who need to discover workflow transitions without hardcoding URLs (e.g., payment provider APIs where a payment can be `_links.capture`, `_links.refund`, `_links.void` depending on state). If you're not shipping an SDK to unaffiliated third parties, skip it and document state transitions in the OpenAPI spec instead.
