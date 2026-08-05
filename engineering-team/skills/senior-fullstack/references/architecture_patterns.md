# Architecture Patterns

Structural decisions for how a codebase is decomposed, how components talk to each other, and how data flows. Read this after Workflow 1 (Start New Project) or Workflow 2 (Audit Existing Codebase) once a stack is chosen — this document is about the shape of the system, not the framework names.

Every pattern below is tied back to the four customization profiles (`saas-startup`, `enterprise-scale`, `internal-tool`, `marketing-site`) and to Sam Newman's "MonolithFirst" canon this skill cites in its assumptions section.

---

## 1. Monolith vs. modular monolith vs. microservices

This is the single highest-leverage decision this skill's Decision Engine makes, driven by the `team-size-12mo` input. Get this wrong and every other architectural decision inherits the mistake.

### Decision criteria

| Signal | Monolith | Modular monolith | Microservices |
|---|---|---|---|
| Team size (12mo) | < 8 eng | 8-30 eng | 30+ eng, multiple autonomous teams |
| Deploy cadence | Any | Daily+ | Per-team independent cadence required |
| Domain clarity | Unclear/evolving | Clear-ish, bounded contexts identifiable | Well-understood, stable bounded contexts |
| Ops maturity | Low — no dedicated platform team | Low-medium | Requires a platform/infra team |
| Scaling need | Uniform load | Uniform load, maybe one hot module | Genuinely divergent scaling needs per domain (e.g., search vs. billing) |
| **Kill criterion** | — | If you can't name your bounded contexts, you're not ready to modularize — you'll draw the wrong boundaries and pay to redraw them | If you can't staff independent on-call rotations per service, you've built a distributed monolith with microservices' ops tax and none of its benefits |

**The default for `saas-startup` (< 10 eng) is a modular monolith on Next.js + Postgres.** Not a plain monolith, and not microservices. The reasoning:

- A plain monolith with no internal module boundaries becomes a "big ball of mud" the moment 3+ engineers touch it concurrently — you get merge conflicts on shared files and no clear ownership.
- Microservices at this team size means every feature requires touching 2-3 repos, standing up service-to-service auth, and debugging distributed traces for what should be a single function call. Sam Newman's own "MonolithFirst" argument: you don't understand your domain boundaries well enough yet to draw service lines correctly, so you'll draw them wrong and pay a painful re-split later.

### What a modular monolith actually looks like (text diagram)

```
apps/web/                       ← single deployable
├── modules/
│   ├── billing/
│   │   ├── domain/             ← entities, business rules, no framework imports
│   │   ├── application/        ← use-cases / services, orchestrates domain
│   │   ├── infra/              ← Postgres repository impls, Stripe client
│   │   └── api/                ← route handlers / tRPC router for this module
│   ├── users/
│   │   ├── domain/
│   │   ├── application/
│   │   ├── infra/
│   │   └── api/
│   └── notifications/
│       └── ...
├── shared/
│   ├── db/                     ← single Prisma/Drizzle client, shared migrations
│   └── ui/                     ← shared component library
└── package.json                ← ONE package, one deploy artifact
```

**The rule that makes this work:** `modules/billing` may not import from `modules/users/infra` directly — cross-module calls go through the other module's `application` layer (its public interface), never its internals. Enforce this with import boundaries (`eslint-plugin-boundaries` or a Nx/Turborepo module graph rule — see `development_workflows.md` §3). This is what makes the eventual "split billing into its own service" surgery a boundary-cut instead of a rewrite: the module already only talks to `users` through a narrow interface.

### When to actually split into services

Split a module into its own service only when one of these is concretely true, not hypothetically true:

1. **Divergent scaling profile**: e.g., a video-transcoding module needs GPU instances while the rest of the app runs on commodity web servers.
2. **Divergent deploy cadence with organizational boundary**: a separate team owns `billing` and needs to deploy it without coordinating with the `web` team's release train.
3. **Divergent tech requirement**: a fraud-detection module needs Python/ML tooling the rest of the TS codebase doesn't use.
4. **Compliance isolation**: PCI scope reduction by isolating payment processing into a narrowly-scoped service with its own network boundary.

If none of these are true, "microservices for scalability" is very often a `saas-startup`-profile anti-pattern this skill's Decision Engine is designed to flag.

### `enterprise-scale` profile: domain-bounded services

At 50+ engineers, the modular monolith's own coordination cost (one deploy pipeline, one repo, everyone rebasing against `main`) becomes the bottleneck, not the solution. The pattern shifts to domain-bounded services, each owned by one team, each independently deployable:

```
                     ┌─────────────┐
                     │   BFF /     │
                     │   Gateway   │
                     └──────┬──────┘
            ┌───────────────┼───────────────┐
      ┌─────▼─────┐   ┌─────▼─────┐   ┌─────▼─────┐
      │  Billing   │   │   Users   │   │  Catalog  │
      │  service   │   │  service  │   │  service  │
      │ (own DB)   │   │ (own DB)  │   │ (own DB)  │
      └────────────┘   └───────────┘   └───────────┘
```

Each service owns its own database — no shared schema, no cross-service joins. This is the line that actually defines a microservice vs. just a "distributed monolith": if two services share a database, they are not independently deployable (a schema migration in one can break the other), and you've paid the network-hop tax without the isolation benefit.

---

## 2. The Backend-for-Frontend (BFF) pattern

**Problem it solves:** a mobile app, a web app, and a partner API all want data shaped differently from the same domain services. Without a BFF, either every client does N calls and stitches data together client-side (chatty, slow on mobile networks), or your domain services grow client-specific fields to satisfy everyone (leaky abstraction).

```
┌──────────┐   ┌──────────┐   ┌──────────┐
│  Web app  │   │ Mobile app│   │ Partner  │
└─────┬────┘   └─────┬────┘   └────┬─────┘
      │              │              │
┌─────▼────┐   ┌─────▼────┐   ┌────▼─────┐
│  Web BFF  │   │Mobile BFF │   │Partner API│
│(aggregate,│   │(aggregate,│   │  (REST,   │
│ shape for │   │ shape for │   │ versioned,│
│  web UI)  │   │small payload)  │  stable)  │
└─────┬────┘   └─────┬────┘   └────┬─────┘
      └──────────────┼──────────────┘
              ┌───────▼───────┐
              │ Domain services│
              │ (Billing, Users,│
              │  Catalog...)   │
              └────────────────┘
```

**Concrete rule:** a BFF is allowed to call multiple domain services and aggregate/reshape their responses. A BFF is *not* allowed to contain business logic (that belongs in the domain service) — its job is orchestration and shaping, nothing else. Violating this turns the BFF into an undocumented second copy of your business rules that drifts from the source of truth.

**When to skip the BFF entirely:** `saas-startup` profile with a single web client and no mobile app or partner API — a BFF layer here is solving a problem you don't have yet. Route those calls directly from the frontend to the modular monolith's API (tRPC or REST). Add a BFF the day a second client type (mobile, partner) shows up with genuinely different data-shape needs.

---

## 3. Event-driven vs. request-response

| | Request-response | Event-driven |
|---|---|---|
| **Model** | Caller waits for a synchronous (or short-lived async) reply | Producer emits an event, doesn't know or care who consumes it; consumers react independently |
| **Pick when** | Caller needs the result to proceed (e.g., "is this discount code valid") | Producer doesn't need to know who reacts, or there are multiple independent reactions to one fact (e.g., "order placed" → charge card, send email, update inventory, notify warehouse) | 
| **Coupling** | Tight — caller knows the callee's contract and is blocked on it | Loose — producer and consumers only agree on an event schema, not on each other's existence |
| **Failure mode** | Caller gets an immediate error; easy to reason about, easy to retry at the call site | Harder to trace — a failed consumer doesn't block the producer, so you need dead-letter queues + monitoring to know something silently failed |
| **Consistency** | Can be strongly consistent (one transaction) | Eventually consistent by default — the "order placed" event and "inventory decremented" happen at different times |
| **Infra needed** | None beyond your normal API layer | Message broker (SQS, RabbitMQ, Kafka, or Postgres-as-queue via `pg-boss`/BullMQ+Redis for simpler needs) |

**Rule of thumb for this skill's profiles:**
- `saas-startup`: request-response for almost everything; introduce a lightweight job queue (BullMQ on Redis) only for things that are genuinely async from the user's perspective (sending emails, processing uploaded files, webhook delivery with retries). Don't reach for Kafka at this team size — that's ops overhead with no matching payoff.
- `enterprise-scale`: event-driven becomes necessary once services are domain-bounded (§1) — "order placed" needs to fan out to Billing, Inventory, and Notifications without the Orders service knowing about all three. This is also where Kafka/SNS+SQS-style brokers earn their keep, along with the discipline of event schema versioning and dead-letter-queue monitoring as an actual on-call responsibility.

**Concrete example — order placement, event-driven:**

```
Orders service                Message broker              Consumers
     │  publish                    │
     │  "OrderPlaced"               │
     │  {orderId, userId, items}    │
     ├─────────────────────────────►│
     │                              ├──► Billing service (charge card)
     │                              ├──► Inventory service (decrement stock)
     │                              └──► Notifications service (send email)
```

The Orders service returns "order placed" to the user the instant its own write succeeds — it does not wait for Billing, Inventory, or Notifications. If the card charge fails, that's a separate compensating event (`PaymentFailed`) that Orders subscribes back to, not a synchronous block on order creation. This is the saga pattern in miniature; full saga orchestration (with a dedicated orchestrator vs. choreography) is a `enterprise-scale`-tier concern — don't build one for a `saas-startup` profile's checkout flow.

---

## 4. Backend patterns: Clean Architecture and Repository Pattern

**Clean Architecture's core rule, restated practically:** dependencies point inward. Your domain/business logic must not import your framework, your database driver, or your HTTP layer. Your framework code imports your domain code — never the reverse.

```
   ┌─────────────────────────────────────┐
   │  Infra (Postgres, Stripe, S3, HTTP)  │  ← depends on
   │  ┌─────────────────────────────┐    │     Application
   │  │  Application (use-cases)     │    │  ← depends on
   │  │  ┌───────────────────────┐  │    │     Domain
   │  │  │  Domain (entities,     │  │    │
   │  │  │  business rules)       │  │    │
   │  │  └───────────────────────┘  │    │
   │  └─────────────────────────────┘    │
   └─────────────────────────────────────┘
```

**Why it matters practically, not academically:** if `domain/Order.ts` imports `PrismaClient`, you cannot unit-test order-total calculation without a database connection. The Repository Pattern is the seam that fixes this — `application` code depends on a `OrderRepository` *interface*, and `infra` provides the Prisma-backed implementation:

```typescript
// domain/order.ts — no framework imports
export class Order {
  constructor(private items: LineItem[]) {}
  total(): Money { /* pure business logic */ }
}

// application/orderRepository.ts — the interface, owned by application layer
export interface OrderRepository {
  findById(id: string): Promise<Order | null>;
  save(order: Order): Promise<void>;
}

// infra/prismaOrderRepository.ts — the implementation, owned by infra
export class PrismaOrderRepository implements OrderRepository {
  constructor(private prisma: PrismaClient) {}
  async findById(id: string) {
    const row = await this.prisma.order.findUnique({ where: { id } });
    return row ? toDomainOrder(row) : null;
  }
  async save(order: Order) { /* ... */ }
}
```

**Pragmatic call for `saas-startup` profile:** full Clean Architecture with four strict layers is often over-engineering for a 3-person team's MVP. The 80/20 version: keep domain logic in plain functions/classes with zero framework imports, and keep a thin repository interface around your ORM calls so you *can* unit test business logic without a database. Skip the ceremony of separate `application`/`infra` folders until the module graph in §1 actually needs the boundary enforced.

---

## 5. Caching strategies

| Pattern | How it works | Pick when | Risk |
|---|---|---|---|
| **Cache-aside** | App checks cache first; on miss, reads from DB, writes result to cache, returns it | Default for read-heavy data with acceptable staleness (product catalog, user profile) | Cache stampede on cold cache / expiry — mitigate with jittered TTLs or a request-coalescing lock |
| **Write-through** | Every write goes to cache and DB in the same operation | Data must never be stale in cache (pricing, inventory counts) | Slower writes (double write), cache becomes a single point of failure if not made resilient |
| **HTTP cache headers (`Cache-Control`, `ETag`)** | CDN/browser caches responses per HTTP semantics | Public, cacheable GET responses — marketing pages, public API reads | Cache invalidation on mutation requires either short TTLs or explicit purge calls to the CDN |

```
Cache-aside read path:
  Request → [Redis: HIT?] ──yes──► return cached value
                 │no
                 ▼
           [Postgres query]
                 │
                 ▼
        [Write result to Redis, TTL=60s]
                 │
                 ▼
              return value
```

**Rule for this skill's profiles:** `marketing-site` profile leans almost entirely on HTTP/CDN caching (static-first — there's often no origin cache-aside layer needed at all). `saas-startup` and `enterprise-scale` profiles use Redis cache-aside for expensive, read-heavy queries (dashboard aggregates, computed permissions) — but cache-aside is a performance optimization applied *after* you've confirmed the query is actually slow, not a default added preemptively.

---

## 6. Authentication architecture: JWT + refresh tokens vs. sessions

| | Session-based | JWT + refresh token |
|---|---|---|
| **State** | Server-side session store (Redis) | Stateless access token; refresh token may be tracked server-side for revocation |
| **Revocation** | Immediate — delete the session | Access token can't be revoked before expiry unless you maintain a blocklist (which reintroduces server state) |
| **Scale** | Requires a shared session store across instances | Access token needs no shared store — verifiable by signature alone |
| **Pick when** | Single app, want instant logout/revocation, simplicity over statelessness | Multiple services need to verify identity without a shared session store (fits the domain-bounded services diagram in §1) |

**Concrete flow for JWT + refresh:**

```
1. Login → server issues:
     - access token  (JWT, 15min TTL, signed, contains userId+roles)
     - refresh token (opaque random string, 30-day TTL, stored hashed in DB)
2. Client stores access token in memory, refresh token in httpOnly+secure cookie
3. Client calls API with access token in Authorization header
4. Access token expires → client calls /auth/refresh with refresh token cookie
5. Server validates refresh token against DB (allows revocation check here),
   issues a NEW access token + rotates the refresh token
6. Logout / revoke-all-sessions → delete refresh token row(s) from DB
```

The access token being short-lived (15 min) is what bounds the "can't revoke a JWT before expiry" problem — you accept up to 15 minutes of exposure on a compromised access token in exchange for not needing a shared blocklist store on every request. The refresh token rotation on every use (step 5) is what limits the blast radius of a stolen refresh token — a legitimate client's next refresh call will fail because its old refresh token was already invalidated, revealing the theft.

**Rule for this skill's profiles:** `saas-startup` and `internal-tool` profiles default to session-based auth via Auth.js/Clerk (simpler, revocation is trivial, no token-rotation logic to get wrong). Reach for custom JWT + refresh only when the `enterprise-scale` profile's domain-bounded services (§1) need to verify identity independently without a shared Redis session store — i.e., when statelessness is solving a real architectural problem, not a hypothetical one.
